# Poor Man's Global Traffic Manager

Sometimes we need to add redundancy to some service or server which happen to be a public-facing entry point of our infrastructure. For example, imagine we want to add a high availability pair for a load balancer which sits on the edge of network and forwards traffic to alive backend servers.

```
                                             ┌─────────────┐
                                             │             │
                                      ┌─────►│  Backend 1  │
                                      │      │             │
                                      │      └─────────────┘
                                      │
                                      │
                                      │      ┌─────────────┐
                                      │      │             │
                    ┌────────────┐    ├─────►│  Backend 2  │
                    │            │    │      │             │
                    │            │    │      └─────────────┘
  Public traffic    │    Load    │    │
───────────────────►│            ├────┤
                    │  Balancer  │    │      ┌─────────────┐
                    │            │    │      │             │
                    │            │    ├─────►│     ...     │
                    └────────────┘    │      │             │
                                      │      └─────────────┘
                                      │
                                      │
                                      │      ┌─────────────┐
                                      │      │             │
                                      └─────►│  Backend N  │
                                             │             │
                                             └─────────────┘
```

We can't just add another load balancer in front of it because otherwise any kind of switch in front of our HA pair will become a single point of failure itself too. But we still need to switch traffic between load balancer instances:

```
                                             ┌─────────────┐
                                             │             │
                                      ┌─────►│  Backend 1  │
                    ┌────────────┐    │      │             │
                    │            │    │      └─────────────┘
                    │            │    │
  Public traffic    │    Load    │    │
───────────────────►│            ├────┤      ┌─────────────┐
         ▲          │ Balancer 1 │    │      │             │
         │          │            │    ├─────►│  Backend 2  │
         │          │            │    │      │             │
                    └────────────┘    │      └─────────────┘
     Switching?                       │
                                      │
         │          ┌────────────┐    │      ┌─────────────┐
         │          │            │    │      │             │
         ▼          │            │    ├─────►│     ...     │
  Public traffic    │    Load    │    │      │             │
─ ─ ─ ─ ─ ─ ─ ─ ─ ─►│            ├────┤      └─────────────┘
                    │ Balancer 2 │    │
                    │            │    │
                    │            │    │      ┌─────────────┐
                    └────────────┘    │      │             │
                                      └─────►│  Backend N  │
                                             │             │
                                             └─────────────┘
```

## ~~Rich~~ Wealthy Men's Solutions

There are various solutions for this problem exist for a long time. Basically, all of them mess with the network switching at some level in order to direct incoming public traffic to both or only one load balancer.

### VRRP, CARP, Virtual IP, Floating IP, ...

Essentially assigns one or few IP addresses to one or few active load balancer instances. IP addresses (re-)attached to operational instances on failure. Such methods ultimately use local network equipment to switch traffic to operational load balancers.

It is worth noting that network equipment in question does not necessarily has any redundancy. For example, there may be perfectly good VRRP pair of two load balancers, still connected to single Ethernet switch, acting as a single point of failure. Even redundant switches may be prone to simultaneous failures due to similar conditions and common broadcast domain.

This solution is suitable for local traffic management only, e. g. for load balancers within single datacenter.

### Anycast, BGP, other methods based on dynamic routing

Usually not used alone, but in conjunction with local traffic management within point of presence (datacenter, availability zone, whatever). Sends announces of single IP block from multiple locations, effectively making traffic to some IPs served by machines in multiple locations. These methods ultimately use network neighbors and neighbors of their neighbors as a switch in front of their infrastructure, announcing or not anouncing specific blocks from given location.

This particular method is available only to fairly large network operators, presumably even operating their own autonomous systems.

### DNS-based methods

Does traffic switching at DNS level, driving client to correct destination server. There are following options widely known:

* [Round-robin DNS](https://en.wikipedia.org/wiki/Round-robin_DNS). Actually, non-option, because it just exposes all potentially available instances, hoping client either will be lucky to connect to the right one or persistent enough to try until it finds working one.
* Dynamic DNS, tracking state of origin servers (AWS Route 53, Cloudflare DNS LB, PowerDNS dnsdist, ...). Keeps track on healthy destination servers and responds to address requests with **just one** address which belongs to currently healthy server.

Interesting fact about such cloud DNS load balancing services is that these are billed on per-request basis, but we basically have no way to control incoming flow of requests  or a way to check if these DNS requests actually happened.

### Wealthy Men's Problems

Some of these methods are hard to implement properly. Even for keepalived it is recommended to run VRRP protocol on separate link between servers. Otherwise maxed out bandwidth of single link will interfere with master election. Some of them are easy as plug and play (DNS GTMs), but may become quite costy.

Solutions above imply that traffic forwarding target (load balancer or other origin server) is either healthy or faulty, which is quite an assumption. It may be not always the case, especially for global traffic management solutions. For example, AWS Route 53 makes periodic healthcheck probes from few locations to ensure target server is available. But it may be not necessarily available from some remote locations while other origin servers are - connectivity on the Internet is not binary.

Poor Man's Global Traffic Manager doesn't make such assumptions, not limited to single datacenter only, doesn't have moving parts and costs you basically nothing. With it you can spin up global-scale fault-tolerant services quickly and dedicate more time to make living.

## Layout

Usually DNS resolving process works like this:

```
  ┌────────────┐ A? example.com ┌───────────────┐
  │            │      (1)       │               │
  │            ├───────────────►│ DNS recursive │
  │   Client   │                │               │
  │            │◄───────────────┤   resolver    │
┌─┤            │ A  example.com │               │
│ └────────────┘      (8)       └┬────┬────┬────┘
│                                │ ▲  │ ▲  │ ▲
│  ┌─────────────────────────────┘ │  │ │  │ │
│  │A? example.com (2)             │  │ │  │ │
│  │ ┌─────────────────────────────┘  │ │  │ │
│  ▼ │NS com (3)                      │ │  │ │
│ ┌──┴──────────┐    ┌────────────────┘ │  │ │
│ │             ├┐   │A? example.com (4)│  │ │A? example.com (6)
│ │    ROOT     ││   │NS example.com (5)│  │ │A  example.com (7)
│ │             ││   ▼ ┌────────────────┘  │ │
│ │ nameservers ││  ┌──┴──────────┐        │ │
│ │             ││  │             ├┐       │ │
│ └┬────────────┘│  │    .COM     ││       │ │
│  └─────────────┘  │             ││       ▼ │
│                   │ nameservers ││  ┌──────┴──────┐
│                   │             ││  │             ├┐
│                   └┬────────────┘│  │ example.com ││
│                    └─────────────┘  │             ││
│                                     │ nameservers ││
│                                     │             ││
│                                     └┬────────────┘│
│                                      └─────────────┘
│                  ┌─────────────┐
│                  │             ├┐
│Actual connection │ EXAMPLE.COM ││
└─────────────────►│             ││
         (9)       │   servers   ││
                   │             ││
                   └┬────────────┘│
                    └─────────────┘
```

Client wants to establish connection with some host specified by its domain name. Client asks DNS resolver (usually it's DNS servers provided by ISP, residing in the same network). DNS resolver, if has no record in cache, follows all hierarchy of authoritative DNS servers. On each step it either gets redirected to more specific nameserver, where requsted domain is delegated, or finally retrieves requested resource record.

Note that on each step usually there are multiple nameservers available for DNS recursor to make a requests. DNS has native fault tolerance mechanisms and if some nameserver is not available, it will request another nameserver in that resource record set. For example, right now there are 13 nameservers available to serve .COM zone:

```
a.gtld-servers.net.
b.gtld-servers.net.
...
m.gtld-servers.net.
```

Each of these nameservers can be requested for nameserver of example.com domain and DNS recursor will try to contact another one if it will receive no response on the first attempt. We can use this property to build DNS-based traffic switching between working servers. The idea is following: we can deploy two or more authoritative nameservers for our domain and make each of them return its own IP address. This way there will be a causal relationship between address of nameserver, which DNS recursor has reached, and IP address used to contact actual service.

Unlike dynamic DNS GTMs we do not try to figure out which server is operational, we do not make any active probes. We just let DNS recursor to figure out which NS server is reachable and it will direct client to the same machine which successfully provided DNS response. Effectively it shifts probing and switching to client's DNS recursor, allowing us to get away with two simple DNS server instances with static configuration. Diagram of interactions may look like this:

```
  ┌────────────┐ A? example.com ┌───────────────┐
  │            │      (1)       │               │
  │            ├───────────────►│ DNS recursive │
┌─┤   Client   │                │               │
│ │            │◄───────────────┤   resolver    │
│ │            │ A  example.com │               │
│ └────────────┘   (9) (=LB2)   └┬────┬────┬─┬──┘
│                                │ ▲  │ ▲  │ │ ▲
│  ┌─────────────────────────────┘ │  │ │  │ │ │
│  │A? example.com (2)             │  │ │  │ │ │
│  │ ┌─────────────────────────────┘  │ │  │ │ │
│  ▼ │NS com (3)                      │ │  │ │ │
│ ┌──┴──────────┐    ┌────────────────┘ │  │ │ │
│ │             ├┐   │A? example.com (4)│  │ │ │
│ │    ROOT     ││   │NS example.com (5)│  │ │ │ A example.com (8)
│ │             ││   ▼ ┌────────────────┘  │ │ │
│ │ nameservers ││  ┌──┴──────────┐        │ │ │ (=LB2)
│ │             ││  │             ├┐       │ │ │
│ └┬────────────┘│  │    .COM     ││       │ │ │
│  └─────────────┘  │             ││       │ │ │
│                   │ nameservers ││       │ │ │
│                   │             ││       │ │ │
│                   └┬────────────┘│       │ │ │
│                    └─────────────┘       │ │ │
│    A? example.com (6)                    │ │ │
│    ┌─xxxxxxxxxxxxxxxx────────────────────┘ │ │
│    │                     A? example.com (7)│ │
│    │ ┌──xxxxxxxxxxxxx     ┌────────────────┘ │
│    ▼ │                    ▼                  │
│   ┌──┴──────────────┐    ┌─────────────────┐ │
│   │                 │    │                 ├─┘
│   │   example.com   │    │   example.com   │
│   │                 │    │                 │
│   │  nameserver1 &  │    │  nameserver2 &  │
│   │                 │    │                 │
│   │  loadbalancer1  │    │  loadbalancer2  │
│   │                 │    │                 │
│   │    (FAULTY)     │    │    (HEALTHY)    │
│   │                 │    │                 │
│   └─────────────────┘    └─────────────────┘
│                                   ▲
│ Actual connection (10)            │
└───────────────────────────────────┘
```

As diagram indicates, DNS recursor descends hierarchy as usual. When it comes to resolving of actual address record of example.com resource, it tries to contact first[^1] nameserver, which is also the first loadbalancer providing example.com service. First server is faulty and doesn't provides response to DNS recursor. DNS recursor tries to contact another nameserver and succeeds. Second (name)server responds with its own IP address as always. It makes DNS recursor to return IP of alive nameserver which is also an IP address of alive loadbalancer of example.com service.

[^1]: For sake of clarity. Actual order is not guaranteed.

## Implementation

Let's consider a bit more practical example where we need to loadbalance some hostname, but we don't want to delegate entire zone to our own authoritative nameservers. We will take `example.com` domain and ensure high availability for hostname `api.example.com` which points to load balancers.

### Step 1. Prepare servers

Prepare two servers for incoming traffic. They can even reside in different datacenters and forward traffic to their local group of workers. We will assume we have two servers with IP addresses: `198.51.100.10` and `203.0.113.20`.

**Validation:** check if you're able to login to these servers and these are reachable on designated addresses.

### Step 2. Install payload on servers

Setup service or load balancers providing actual service on these IP addresses.

**Validation:** depends on payload.

### Step 3. Install catch-all authoritative DNS server

At this step we need to install authoritative DNS server on each machine and make it respond on any request with IP address of its machine. Almost any DNS server can do this job, even simple dnsmasq. But reasonably good option for this is CoreDNS.

Install CoreDNS on each server and apply following configuration:

#### First server

`/etc/coredns/db.lb`:

```
@	3600 IN	SOA lb1 adminemail.example.com. 2021121600 1200 180 1209600 30
	3600 IN NS lb1
	3600 IN NS lb2

*       30   IN A     198.51.100.10
```

`/etc/coredns/Corefile`:

```
example.com {
    file /etc/coredns/db.lb
}
```

#### Second server

`/etc/coredns/db.lb`:

```
@	3600 IN	SOA lb1 adminemail.example.com. 2021121600 1200 180 1209600 30
	3600 IN NS lb1
	3600 IN NS lb2

*       30   IN A     203.0.113.20
```

`/etc/coredns/Corefile`:

```
example.com {
    file /etc/coredns/db.lb
}
```

**Validation:** command `dig +short api.example.com @198.51.100.10` should return address `198.51.100.10`. Command `dig +short api.example.com @203.0.113.20` should return address `203.0.113.20`

### Step 4. Add A-records for servers in DNS

Create following DNS records in example.com zone:

```
lb1.example.com.	300	IN	A	198.51.100.10
lb2.example.com.	300	IN	A	203.0.113.20
```

DNS zone edit process depends where you're hosting it. Sometimes it's Godaddy control panel, sometimes Cloudflare. You should know better.

**Validation:** command `dig +short lb1.example.com` should return `198.51.100.10`. Command `dig +short lb2.example.com` should return `203.0.113.20`.

### Step 5. Finally delegate hostname to loadbalancers/nameservers

Remove all existing DNS records for name `api.example.com`. Add following ones:

```
api.example.com. 300 IN	NS	lb1.example.com.
api.example.com. 300 IN	NS	lb2.example.com.
```

Done! After few minutes you will be able to reach domain api.example.com via two load balancer we set up.

**Validation:** command `dig +trace api.example.com` should produce output indicating lb1 or lb2 were contacted and resolve name to one of their addresses.

## Maintenance

If you need to do maintenance on one of servers or server is misbehaving, just stop coredns on that server and wait TTL (30 seconds in our example).
