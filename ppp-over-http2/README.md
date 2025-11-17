# PPP-over-HTTP/2: having fun with dumbproxy and pppd

I run few instances of [dumbproxy](https://github.com/SenseUnit/dumbproxy) (simple but quite versatile forward proxy server) for my personal needs. Not so long ago I implemented a new operation mode for it, allowing dumbproxy to be run as a subprocess and communicate with parent process via stdin/stdout instead of listening port. It is very useful to use it as a `ProxyCommand` for OpenSSH client. But what's important is that it made me realize that I'm just one small feature away from something I hadn't gotten around to doing. Send PPP tunnel through a HTTP proxy!

We already have TLS security around tunnel, authentication, (optional) active probing resistance, good firewall bypassing capabilities, including state VPN censorship. It would be nice to bring all of it to some well-known VPN protocols in order to enable direct IP forwarding. Sure, OpenVPN already has limited support for proxies, we could just point it to a local dumbproxy instance and make it forward further to remote ("parent" in Squid terms) TLS-enabled proxy. But with my aforementioned idea itching, I decided to pay a tribute to one of oldest tunneling protocols, PPP. Dial-up era tunnel running over modern HTTP/2, how cool is that?!

## Starting point

The instances I already run have fairly basic configuration as described [here](https://github.com/SenseUnit/dumbproxy/wiki/Quick-deployment) with few additions:

* Certificate cache is stored in shared Redis instance in order to make instances completely stateless.
* Some domains are filtered by access filter JS script.
* .onion domains redirected to Tor instance.

All in all, it's a plain forward proxy setup with automatic certs from LetsEncrypt and local password database in file.

By the way, [there is a cloud-init spec available](https://github.com/SenseUnit/dumbproxy/wiki/Cloud%E2%80%90init-configuration) to go through setup for you.

## Server setup

Let's take a look into the redirection script (`-js-proxy-router` option).

/etc/dumbproxy-route.js:

```js
function getProxy(req, dst, username) {
	if (dst.originalHost.replace(/\.$/, "").toLowerCase().endsWith(".onion")) {
		return "socks5://127.0.0.1:9050"
	}
	return ""
}
```

There is already one redirection rule, which is irrelevant for now. Let's add another one to send some special destination address into a `pppd file /etc/ppp/options.vpn` subprocess.

/etc/dumbproxy-route.js:

```js
function getProxy(req, dst, username) {
	if (dst.originalHost.replace(/\.$/, "").toLowerCase().endsWith(".onion")) {
		return "socks5://127.0.0.1:9050"
	}
	if (dst.originalHost.toLowerCase() == "pppd") {
		return "cmd://?cmd=pppd&arg=file&arg=/etc/ppp/options.vpn"
	}
	return ""
}
```

Make sure pppd is installed, it's in `ppp` package:

```
apt install ppp
```

pppd options will be

/etc/ppp/options.vpn:

```
nodetach
notty
noauth
172.22.255.1:172.22.255.2
ms-dns 1.1.1.1
ms-dns 8.8.8.8
```

That's already enough to establish a tunnel. But we also need few bits to make system actually forward traffic.

Enable IP forwarding:

```sh
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf && sysctl -p
```


Masquerade traffic leaving through default gateway interface:

```sh
iptables -t nat -D POSTROUTING -o $(ip route show default | head -1 | grep -Po '(?<=dev\s)\s*\S+') -j MASQUERADE
iptables -t mangle -A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

Assuming you're using `iptables-persistent` package to manage iptables, we can make previous change persistent across reboots like this:

```sh
/etc/init.d/netfilter-persistent save
```

That's it, we're done with the server configuration.

## Client setup

Let's create peer configuration for pppd,

/etc/ppp/peers/vpn:

```
nodetach
noauth
nodeflate
nobsdcomp
novj
novjccomp
ipparam vpn
usepeerdns
pty "/usr/local/bin/dumbproxy -proxy h2://LOGIN:PASSWORD@vps.example.org -mode stdio pppd 0"
```

where `h2://LOGIN:PASSWORD@vps.example.org` is a specification of remote proxy we just configured.

Here we use dumbproxy command as a pty command for pppd in order to funnel connection into it. It connects through upstream proxy to a fake address `pppd:0` which is just a symbolic marker for router script on the server side to direct connection straight into a pppd subprocess.

Installing dumbproxy binary (assuming Linux and amd64 architecture, for other architectures see [latest release assets](https://github.com/SenseUnit/dumbproxy/releases/latest)):

```sh
curl -Lo /usr/local/bin/dumbproxy \
	'https://github.com/SenseUnit/dumbproxy/releases/latest/download/dumbproxy.linux-amd64' \
	&& chmod +x /usr/local/bin/dumbproxy
```

The tunnel configuration is done, but we need also sprinkle a little script on top of it in order to configure routing after PPP connection established.


/etc/ppp/ip-up.d/vpn:

```
#!/bin/bash

INTERFACE="$1"
DEVICE="$2"
SPEED="$3"
LOCALIP="$4"
REMOTEIP="$5"
IPPARAM="$6"

if [[ "$IPPARAM" != "vpn" ]] ; then
	# not our config
	exit 0
fi

PROTECT=("vps.example.org") # preserve route for these addresses

default_route4=$(ip -4 route show default | head -1 | cut -d\  -f2-)
default_route6=$(ip -6 route show default | head -1 | cut -d\  -f2-)

for protect_address in "${PROTECT[@]}"; do
	>&2 echo "Protecting $protect_address..."

	if [[ "$default_route4" ]]; then
		for ip in $(getent ahostsv4 "$protect_address" | cut -f1 -d\  | sort | uniq); do
			ip -4 route replace "$ip" $default_route4
		done
	fi
	if [[ "$default_route6" ]]; then
		for ip in $(getent ahostsv6 "$protect_address" | cut -f1 -d\  | sort | uniq); do
			ip -6 route replace "$ip" $default_route6
		done
	fi
done

ip -4 route replace 0.0.0.0/1   dev "$INTERFACE"
ip -4 route replace 128.0.0.0/1 dev "$INTERFACE"
# prevent ipv6 leaks
ip -6 route replace unreachable 2000::/3 

# workaround bug https://lists.opensuse.org/archives/list/bugs@lists.opensuse.org/thread/ZHDF667RJDGAEWJCJB7HGWNARKLAIPGK/
#if [[ "$DNS1" ]]; then
#	resolvconf="/var/run/ppp/resolv.conf.$INTERFACE"
#	chattr -i "$resolvconf"
#	echo "nameserver $DNS1" > "$resolvconf"
#	if [[ "$DNS2" ]]; then
#		echo "nameserver $DNS2" >> "$resolvconf"
#	fi
#	chmod 0644 "$resolvconf"
#	chattr +i "$resolvconf"
#	mount --bind --onlyonce "$resolvconf" /etc/resolv.conf
#fi
```

That script installs direct route to upstream proxy hosts to make sure already encapsulated traffic will not be looping back into the tunnel. Also it installs default route in a way preserving original route after PPP session shutdown.

Don't forget to replace `vps.example.org` with an actual domain name and make script executable.

It's all done, let's try it out!

```
user@ws:~> sudo pppd call vpn
Using interface ppp0
Connect: ppp0 <--> /dev/pts/4
MAIN    : 2025/11/18 03:54:20 main.go:656: INFO     Starting proxy server...
MAIN    : 2025/11/18 03:54:20 main.go:812: INFO     Proxy server started.
local  LL address fe80::b940:dde6:f755:0427
remote LL address fe80::e5da:861e:b382:4e83
Script /etc/ppp/ipv6-up finished (pid 47510), status = 0x0
Script /etc/ppp/ip-pre-up finished (pid 47515), status = 0x0
local  IP address 172.22.255.2
remote IP address 172.22.255.1
primary   DNS address 1.1.1.1
secondary DNS address 8.8.8.8
Script /etc/ppp/ip-up finished (pid 47520), status = 0x0
```

## Checking out

Let's confirm we have achieved both datagram forwarding and traffic goes through remote server. We can go and chat directly with DNS echo server and see which IP address we've used to reach it:

```sh
dig +trace TXT whoami.ds.akahelp.net | grep -P 'IN\s+TXT'
```

It should output IP address of the machine on the remote end of tunnel.

Now, the speed. Here's my result:

![speedtest](speedtest.png)

Not bad, as for TCP-carried tunnel.

## Bonus

But we can make it even weirder! Initially PPP was used for communication over serial line, often carried over phone line with the help of mode. Typically modem was connected to serial port of computer (tty for pppd) and some program had to prepare it for actual data transfer: send AT commands to modem, dial some number, maybe even send login-password over the line before PPP session can be started. We can do it like that too.

We can skip use of dumbproxy on the client and go for just `openssl` command line utility in conjunction with the `chat` program used for setting up modem and expecting responses from it.

pppd peer config becomes

/etc/ppp/peers/vpn-lite:

```
nodetach
noauth
nodeflate
nobsdcomp
novj
novjccomp
ipparam vpn
usepeerdns
connect /usr/local/bin/dialer.sh
pty "openssl s_client -brief -verify_return_error -ign_eof vps.example.org:443"
```

Instead of single `pty` option we use connect script and just openssl s_client utility. It's basically `netcat` but for SSL/TLS.

And the `connect` script will be

/usr/local/bin/dialer.sh:

```sh
#!/bin/bash

USERNAME="username"
PASSWORD="password"

AUTH="$(echo -n "$USERNAME:$PASSWORD" | base64)"

exec /usr/sbin/chat -v -T "$AUTH" \
	TIMEOUT 5 \
	ABORT 'HTTP/1.1 3' \
	ABORT 'HTTP/1.1 4' \
	ABORT 'HTTP/1.1 5' \
	"" "CONNECT pppd:0 HTTP/1.1\r\nHost: pppd:0\r\nProxy-Authorization: Basic \T\r\n\r\n\c" \
	"HTTP/1.1 200" ""
```

It's just an invocation of `chat` program with encoded login-password pair passed as a phone number. Similarly, connection can be started with `sudo pppd call vpn-lite` command.

Of course, it will use HTTP/1.1 instead, but maybe it's for the better - there will be no overhead for HTTP/2 frame encoding/decoding. Speed looks a bit better, but likely within error margin:

![speedtest](speedtest2.png)