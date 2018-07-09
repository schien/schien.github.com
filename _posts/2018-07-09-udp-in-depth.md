---
layout: post
title: "UDP in Depth"
description: ""
category: Computer Neworking
tags: [udp, network, ietf]
---
{% include JB/setup %}

UDP is a simple message-based protocol defined in [RFC768][1], originally designed on top of IPv4.
[RFC2460][2] ([RFC8200][3] for latest version) section 8.1 provides necessary modification for UDP over IPv6.

### What is defined:
-   header format:
    each datagram will contains four fields in header: src port, dest port, data length, checksum.
-   number of ports can be used:
    The src/dest port in header is a 2-byte integer, which means there are 65536 ports can be used
    on a single IP address. This is not shared with TCP.
-   max length of a datagram:
    The length field in header is a 2-byte integer, which represents the total length of the
    datagram (including header). In this case, the maximum data can be send in one datagram
    is `65535-8` bytes.
-   chechsum algorithm:
    a pseudo header is added to protect misrouted datagrams: src addr, dest addr,
    UDP protocol number, UDP length. Datagram will be discard if checksum validation failed, except
    if `0x0000` is specified in checksum field. `0x1111` is used if the computed checksum is `0x0000`.
    However, IPv6 doesn't allow turning off checksum validation (defined by [RFC6935][4]).
    Datagrams with checksum `0x0000` will be discard. [RFC6936][5] provides the applicabity of enabling
    UDP zero checksum on IPv6.

### UDP Header

     0               8               16              24            31
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           Source Port         |        Destination Port       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Length            |           Checksum            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

### UDP Pseudo-header

For IPv4:

     0               8               16              24            31
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         Source Address                        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                      Destination Address                      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     zero      | Protocol (17) |           UDP Length          |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

For IPv6:

     0               8               16              24            31
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    +                                                               +
    |                                                               |
    +                         Source Address                        +
    |                                                               |
    +                                                               +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    +                                                               +
    |                                                               |
    +                      Destination Address                      +
    |                                                               |
    +                                                               +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                   Upper-Layer Packet Length                   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                      zero                     |Next Header(17)|
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

### What is not defined:
-   connection-less:
    Doesn't define connection establishment procedure.
    Endpoint can send datagrams to designated remote IP/port directly.
-   receiving order is not guaranteed:
    Doesn't define sequence number in datagram. Datagram can arrived at receiver in any order.
-   message delivery is not guaranteed:
    Doesn't define message acknowledge procedure. Both sender and receiver will not know if there
    is a datagram dropped on the routing path.  ICMP error might be received by sender but the
    information will not propagate to UDP layer.
-   no flow control:
    Doesn't define flow control / congestion control algorithm.
    Receiver and routers on the path will simply discard the datagram it cannot handle.

UDP-Lite
--------

UDP uses the entire datagram to calculate checksum and the datagram will be discard if validation failed.
For UDP tunneling usecase, the tunneling protocol might defined their own data correction mechanism.
Discarding the entire datagram will cancel the correction mechanism. Therefore, UDP-Lite [RFC3828][6] is
introduced to allow application-specified data range for computing checksum.

In POSIX Socket API, UDP-Lite socket can be create by specifying the protocol id [udplite][7]. The checksum data range
is controlled via socket option `UDPLITE_SEND_CSCOV` and `UDPLITE_RECV_CSCOV`.

    sockfd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDPLITE);


SCTP
----

DCCP
----

[1]: https://tools.ietf.org/html/rfc768
[2]: https://tools.ietf.org/html/rfc2460
[3]: https://tools.ietf.org/html/rfc8200
[4]: https://tools.ietf.org/html/rfc6935
[5]: https://tools.ietf.org/html/rfc6936
[6]: https://tools.ietf.org/html/rfc3828
[7]: http://man7.org/linux/man-pages/man7/udplite.7.html
