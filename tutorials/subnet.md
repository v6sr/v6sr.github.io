---
layout: default
---

## [](#intro) Introduction

This document describes how to configure Linux to allow an application to receive packets and establish
incoming connections on IP addresses that are NOT explicitly configured (via interfaces) on the host. This
technique does not use netfilter/iptables, instead it relies on the Linux routing table and rules. This 
capability has been available in Linux since v2.6.37 for IPv6.

This functionality has several use cases such as load balancers, SR application/service chaining, 
virtual hosting, and for servers delivering content identified by IPv6 addresses.

There are 2 methods for configuration. To allow incoming connections by explicitly configuring the subnet, 
use [Method 1](#method-1-configuring-by-subnet). To allow incoming connections for any packet received on an 
interface and leave the policy of whether to serve the traffic up to the application use 
[Method 2](#method-2-configuring-by-interface). This tutorial focuses on IPv6, but these methods should work 
with IPv4 as well (not tested by the author). 

## [](#method1)Method 1: Configuring by Subnet

To receive packets destined for a IPv4 or IPv6 subnet, we add an unicast prefix entry to the ***local*** 
routing table. 

For example, to receive packets for 2001:db8:0:1::/64 -

```console
# ip -6 route add local 2001:db8:0:1::/64 dev lo table local
```

To persist that route in CentOS create /etc/sysconfig/network-scripts/route6-eth0, replacing 
eth0 with your primary interface, and add the following -

```console
local 2001:db8:0:1::/64 dev lo table local
```

Continue onto [configuring the application](#configuring-the-application).

## [](#method2)Method 2: Configuring by Interface

The first is step is to create a new routing table. To create routing table 100, named ggv6 -

```console
echo 100 ggv6 >> /etc/iproute2/rt_tables
```

Next, To receive any packets received on an interface (e.g. eth0), create a new Linux policy routing rule -

```console
# ip -6 rule add from all iif eth0 lookup ggv6
```

To persist this rule in CentOS create /etc/sysconfig/network-scripts/rule6-eth0, replacing 
eth0 with your primary interface, and add the following -

```console
from all iif eth0 lookup ggv6
```

Next, add an entry to routing table ***ggv6*** - 

```console
# ip -6 route add local default dev lo table ggv6
```

To persist this route in CentOS create /etc/sysconfig/network-scripts/route6-eth0, replacing 
eth0 with your primary interface, and add the following -

```console
local default dev lo table ggv6
```
Continue onto [configuring the application](#configuring-the-application).

## [](#app)Configuring the Application

For TCP based applications, no modification is required to accept TCP connections and send/receive packets.
The application should be configured to listen on all IPs. 

For example, configure nginx to listen to all IPs as shown below:

```console
listen       [::]:80
```

For UDP based applications, incoming packets will arrive to any application that is bound to all IPs 
without any modifications. However, to respond with UDP packets with a source address of one of the 
addresses in the subnet, you must set the IP_TRANSPARENT flag via setsocketopt(). 

Continue on for optional reading on how this all works. 

***

## [](#details)(Optional) Implementation Details

Each Linux host, has at least 3 routing tables {***local***, ***main***, ***default***} created at boot as 
long as the CONFIG_IP_MULTIPLE_TABLES (for IPv4) and CONFIG_IPV6_MULTIPLE_TABLES (for IPv6) kernel 
configuration options are set. This seems to be the case on most modern distributions including CentOS 7. 

Linux will first lookup for a match in the ***local*** table, then the ***main*** table and finally, the 
***default*** table. The ***default*** table is usually empty and unused. The ***main*** routing table is 
manipulated by the iproute2 utilities when the table name is not explicitly specified. 

This lookup process is controlled by the Linux Policy Routing Database. Each rule in the database has a 
priority and the rules are examined sequentially. On a host, with the default rules -

```console
# ip -6 rule list

0:  from all lookup local
32766:  from all lookup main
```

The kernel will iterate over each rule until a packet to be routed matches. According to the default rules,
each packet will match rule 0. The kernel will perform a route lookup against the ***local*** table, if a 
matching route is found the kernel will use that route and continue. If a match is not found, then the kernel
will continue to the next rule. 

<!--For example, on a host with IPv6 address 2001:db8:0:8::2/64 and one interface (eth0), the following will list 
the entries in the ***main*** IPv6 table -

```console
# ip -6 route list

2001:db8:0:8::/64 dev eth0  proto kernel  metric 256
fe80::/64 dev eth0  proto kernel  metric 256
default via 2001:db8:0:8::1 dev eth0  metric 1
```
-->
On a host with IPv6 address 2001:db8:0:8::2/64 and one interface (eth0), the following will list the 
***local*** routing table -

```console
# ip -6 route list table local

local ::1 dev lo  proto kernel  metric 0
local 2001:db8:0:8::2 dev lo  proto kernel  metric 0
local fe80::a4af:89ff:fe5f:2311 dev lo  proto kernel  metric 0
ff00::/8 dev eth0  metric 256
```

This table consists of routes for local/broadcast addresses and is automatically maintained by the kernel. 
Based on the default policy rules, the ***local*** table is checked first as shown above. In this case, a 
packet with destination address 2001:db8:0:8::2 will match the local entry and be delivered locally. 

What happens if another interface (dummy0 with IPv6 address 2001:db8:0:7::2/64) is added to the host? 

```console
# ip link add dummy0 type dummy
# ip -6 addr add 2001:db8:0:7::2/64 dev dummy0 table local
# ip -6 route list table local

...
local 2001:db8:0:7::2 dev lo  proto kernel  metric 0
...
```

If another interface is added to the host, another entry is added by the kernel to the ***local*** routing 
table for that interface's address.

Since ***local*** is a standard Linux routing table, we can manually add entries to the table as well. 
For example, we add 2001:db8:0:8::/64 as a local route type to the table -

```console
# ip -6 route add local 2001:db8:0:8::/64 dev lo table local
# ip -6 route list table local

...
local 2001:db8:0:8::/64 dev lo  metric 1024
...
```

Now all packets destined for any address in 2001:db8:0:8::/64 will be matched in ***local*** and delivered 
locally. 

What if the subnet is not known? Going back to the policy database, add a rule to match all incoming packets
on eth0 and do a route lookup in the new routing table (100) -

```console
# ip -6 rule add from all iif eth0 lookup 100
# ip -6 rule list

0:  from all lookup local
32765:  from all iif eth0 lookup 100
32766:  from all lookup main
```

Now, the kernel will do a route lookup in table 100 for any packets inbound on eth0. Add a route to deliver
all packets locally -

```console
# ip -6 route add local default dev lo table 100
```

## [](#program)(Optional) Programmatically Maintaining the Routing Table

It might be useful to maintain these entries programmatically from an application without using command line
utilities. The rtnetlink API is used to manipulate the routing tables on Linux. 

For example, using the python [pyroute2](https://pypi.python.org/pypi/pyroute2) library, to add a local 
route for subnet 2001:db8:0:8::/64 on the ***local*** table (255) -

```python
idx = ip.link_lookup(ifname="lo")[0]
ip.route('add', dst="2001:db8:0:8::/64", table=255, proto="static", oif=idx, type="local")
```

More examples are available in the [pyroute2 documentation](http://docs.pyroute2.org/).

***

[Return to the Main page](../)
