---
layout: default
---

# [](#header-1) Introduction

This document describes how to use the SRv6 C API on Linux. This API is compatible with the  <a href="https://tools.ietf.org/html/rfc3542#section-7">RFC 3542</a> API for routing header type 0 (Source Routing). Please Section 7 in RFC3542 for additional details.

You must have a SRv6 compatible Linux kernel installed. 
Please see [Kernel Setup Tutorial](centos7.md) for a tutorial on installing a SRv6 Mainline kernel. 

## [](#header-2) Step One - Install CMake 3.1 or higher

The build system requires CMake version 3.1 or higher. On CentOS 7, you must build and install the latest version of CMake. Please see <a href="https://cmake.org/install/">Installing CMake</a> for instructions.

## [](#header-2) Step Two - Get the latest version of the API and build

Using git, clone the latest source code:

```shell
git clone https://github.com/v6sr/v6sr_c_api v6sr_c_api
```

Confirm that you have CMake v3.1 or higher:

```shell
cmake --version
```

The output should indicate the version of cmake.

Build the library:
```shell
cd v6sr_c_api
cmake .
make
```

This should build a static library, libSegmentRoutingAPI.a in the sr_api/ directory. 

## [](#header-2) Step Three - Example Test

Configure your build system to link sr_api/libSegmentRoutingAPI.a and include sr_api/include/sr_api.h. 
For a standalone, example:
```shell
gcc -Wall -o srhtest srhtest.c -Isr_api/include -Lsr_api/ -lSegmentRoutingAPI
```

Please see the commented code snippet for instructions on building a SRv6 header. 

```c
/* some standard socket setup details and 
    error checking omitted for tutorial purposes */

/* include the sr api header */
#include "sr_api.h"
/* allocate structs for destination address and 2 segments */
struct in6_addr da;
struct in6_addr sid1, sid2;

/* create AF_INET6 socket and bind */
...

/* create addr structs for each sid */
inet_pton( AF_INET6, "2001:db8:0:1::1", &da) ;
inet_pton( AF_INET6, "2001:db8:0:2::1", &sid1 );
inet_pton( AF_INET6, "2001:db8:0:7::1", &sid2 );

/* number of segments in the SRH, including the DA */
sid_count = 3;

/* Allocate a new SR header with space for 3 segments */
socklen_t srh_len = inet6_rth_space_n( IPV6_RTHDR_TYPE_4, sid_count );
void *srh = malloc( srh_len );

/* Initialize the header and get a pointer */
inet6_rth_init_n( srh, srh_len, IPV6_RTHDR_TYPE_4, sid_count );
struct ip6_rthdr4 *rthdr = (struct ip6_rthdr4*) srh;

/*  add sids to the header. 
    sids should be reverse order 
    with the destination address first! */

/* You can leave the first sid field empty 
    and the kernel will write the DA. */
rthdr->ip6r4_addr[0] = da;
/* add sids 1 and 2 */
rthdr->ip6r4_addr[1] = sid1;
rthdr->ip6r4_addr[2] = sid2;

/* set the segsleft and lastentry fields */
rthdr->ip6r4_lastentry = sid_count - 1;
rthdr->ip6r4_segleft = sid_count - 1;

/* setup your socket and/or application logic */
...
/* sticky setsockopt on sockfd to send srh. 
    all packets sent on this sockfd with have srh */
rv = setsockopt( sockfd, IPPROTO_IPV6, IPV6_RTHDR, srh, srh_len );
/* check ret value */
...
/* application logic */
...
/* remove srh on socket. all packets sent after will not have srh */
rv = setsockopt( sockfd, IPPROTO_IPV6, IPV6_RTHDR, NULL, 0 );
/* check ret value */
...
```

At this point, you can send packets with a SRH from your application running on a Linux host. 

[Return to the Main page](../)

