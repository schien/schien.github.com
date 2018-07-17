---
layout: post
title: "UDP in Depth"
date:   2018-07-09 19:24:37 +0800
categories: networking
---

This document provides a detailed study of UDP protocol and a list of datagram protocols that can be used with or replaced UDP.

UDP
---

UDP is a simple message-based protocol defined in [RFC768][1], originally designed on top of IPv4.
[RFC2460][2] ([RFC8200][3] for latest version) section 8.1 provides necessary modification for UDP over IPv6.

### What is defined:
-   __header format__:
    each datagram will contains four fields in header: src port, dest port, data length, checksum.
-   __number of ports can be used__:
    The src/dest port in header is a 2-byte integer, which means there are 65536 ports can be used
    on a single IP address. This is not shared with TCP.
-   __max length of a datagram__:
    The length field in header is a 2-byte integer, which represents the total length of the
    datagram (including header). In this case, the maximum data can be send in one datagram
    is `65535-8` bytes.
-   __chechsum algorithm__:
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
-   __connection-less__:
    Doesn't define connection establishment procedure, thus no multiplexing.
    Endpoint can send datagrams to designated remote IP/port directly.
-   __state-less__:
    Doesn't define any state and event handling procedure like in TCP.
-   __receiving order is not guaranteed__:
    Doesn't define sequence number in datagram. Datagram can arrived at receiver in any order.
-   __message delivery is not guaranteed__:
    Doesn't define message acknowledge procedure. Both sender and receiver will not know if there
    is a datagram dropped on the routing path.  ICMP error might be received by sender but the
    information will not propagate to UDP layer.
-   __no flow/congestion control__:
    Doesn't define flow control / congestion control algorithm.
    Receiver and routers on the path will simply discard the datagram it cannot handle.
-   __no encryption__:
    Doesn't define data encryption mechanism.
    The message in the datagram is clear text.

UDP-Lite
--------

UDP uses the entire datagram to calculate checksum and the datagram will be discard if validation failed.
For UDP tunneling usecase, the tunneling protocol might defined their own data correction mechanism.
Discarding the entire datagram will cancel the correction mechanism. Therefore, UDP-Lite ([RFC3828][6]) is
introduced to allow application-specified data range for computing checksum.

In POSIX Socket API, UDP-Lite socket can be create by specifying the [UDPLite][7] protocol id.
The checksum data range is controlled via socket option `UDPLITE_SEND_CSCOV` and `UDPLITE_RECV_CSCOV`.

    sockfd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDPLITE);


SCTP
----
SCTP ([RFC2960][8]) is a message-oriented transport protocol which supports reliable, in-sequence delivery with congestion control like TCP. The design goal is to transmit signaling message reliablely.
SCTP also defined message framing format so that application protocols doesn't need to define a token of message boundary like while using TCP.
SCTP-UDP ([RFC6951][9]) defines a mechanism to run SCTP over UDP.

In current draft of WebRTC DataChannel ([RTCWEB-DATA][13]) it leverage SCTP in the protocol stack to provide configurable, reliable transport on top of DTLS. 

### features:
-   __reliable connection setup/teardown__:
    4-way handshake is used to createa an association between two endpoints (a logical connection).
    Application data can start to be sent with the third and fourth handshake messages.
-   __no head-of-line blocking__:
    Support multiple data stream in one association. Message order is guaranteed inside a stream.
    application data sent on a stream is divided into chucks, which allows interleaving chuncks on
    different stream during transmission.
-   __fregmentation in transport layer__:
    Message is divided into chucks that can fit with the smallest size of MTU of all paths, to prevent IP fregmentation.
-   __TCP-like flow control/congestion control__:
    Use `rwnd` (receiver window size) to do flow control, `cwnd` (congestion window size) and slow start to do congestion control.
-   __support Explicit Congestion Notification__:
    ECN ([RFC3168][21]) provides congestion notification by middlebox in IP layer. SCTP will use this information to adjust the congestion window as well.
-   __support multi-homing__:
    SCTP association can have multiple IP address.
    SCTP will use alternative path for transmitting while failed to use the primary path.
-   __support configurable reliability__:
    Partially Reliable Stream Control Transmission Protocol extension ([PR-SCTP][12]) provides a mechanism to limit the number of retransmission.

DCCP
----

DCCP ([RFC4340][10]) is another message-oriented transport protocol, which supports congestion control.
Unlike SCTP, delivery order is not guaranteed.
The design goal is to support streaming data, i.e. optimized for latency over reliability.
This protocol can also be used on top of UDP in order to provide general congestion contorl ([DCCP-UDP][11]) to application layer.

### features:
-   __reliable connection setup/teardown__:
    DCCP uses 3-way handshake like TCP, and application data can be appended to the handshake message.
-   __negociation for congestion control mechanism__:
    DCCP defines a TCP-like congestion control (send as much as possible) and a TCP-friendly rate control (provide smooth sending rat) for different usecase.
-   __support Explicit Congestion Notification__
-   __support partial checksum like UDP-Lite__:
    DCCP allows application-defined checksume coverage. However, checksume is disabled while using DCCP over UDP,
    since checksum for UDP datagram already protect the DCCP payload.

DTLS
----

DTLS ([RFC4347][14] for 1.0, [RFC6347][15] for 1.2) provides encrypted communication for datagram protocol.
It is based on TLS ([RFC5246][18]) with necessary modification in order to run on unreliable transport channel (e.g. UDP).

It defines three layers: Record layer for fregmentation, encryption, and compression; Handshake layer for exchanging encryption parameters; Alert layer for error handling
Fragmentation and retrasmission timer is used solely for handshake message, no explicit flow control / congestion control on DTLS channel.

### differences to TLS:
-   __no stream cipher__:
    Stream cipher cannot decipher if data lost in the middle of cipher text, therefore these cipher algorithms are disallowed in DTLS.
-   __cookie exchange__:
    A stateless cookie exchange is added to prevent DoS attack (similar design as SCTP). The cookie is the HMAC of IP and parameters of received `ClientHello`, generated by server. Client need to send the same `ClientHello` again with cookie attached.
-   __explicit sequence number and epoch__:
    Unlike TLS, the sequence number of DTLS Record is not encrypted to detect reordered message. `epoch` is increased each time when cipher state is changed (i.e. `ChangeCipherSpec` is sent). Endpoint should discard messages with epoch less than current one. Sequence number is reset to 0 every time `epoch` is changed.
-   __fregment handshake message__:
    Large handshake message will be splited into multiple records that is smaller than PMTU. Sender will retransmit entire message if no response is received before timeout.

QUIC
----

QUIC ([quic-transport][16]) is a new transport protocol under development/standardization.
The goal is to provide a reliable, encrypted, in-sequence delivery, stream-based channel.

This protocol aggregates the benefit from several other protocols, such as HTTP2 ([RFC7540][17]), SCTP, and TLS.
QUIC runs on top of UDP to reduce the burden of deploying protocol in the middlebox (which is the lesson learned from the IPv6 deployment).
It is also implemented in userspace for easy use/update by application, without waiting for the upgrade from operating system provider (lesson learned from TCP new feature deployment).

[comparison-quic-sctp][22] provides a detailed comparison on the design and feautures between QUIC and SCTP.

### features:
-   __reliable connection setup/teardown__:
    The 1-RTT scenario for QUIC is similar to SCTP 4-way handshake, a token is created and sent back by server for verification. The 0-RTT scenario is achieved by preserving a token generated by server for future connection.
    QUIC supports both bidirectional and unidirectional stream for transmitting data.
-   __no head-of-line blocking__:
    Like HTTP2 and SCTP, A QUIC connection consists multiple streams that allows interleaving data delivery, thus avoid the head-of-line blocking of while using TCP.
-   __in-sequence delivery__:
    Byte-offset information is included in the application data frame to recontruct the byte stream.
-   __segmentation__:
    QUIC transport handles PMTU discovery, segments control messages and application data into appropriate packet size to prevent IP fragmentation.
-   __encrypted__:
    QUIC integrates the cryptographic parameter handshake procedure from TLS. The data delivered through QUIC is always encrypted.
-   __flow control/congestion control__:
    The flow control is done on both connection level and stream level. A TCP-like congestion control with modification is employed on QUIC ([quic-recovery][20]).
-   __support 0-RTT__:
    QUIC borrows the 0-RTT concept from TLS 1.3, therefore it can reduce one more RTT comparing to TLS over TCP.
-   __support Explicit Congestion Notification__
-   __support Connection Migration__:
    QUIC connection is identified by a connection ID instead of `(IP, port)` tuple.
    A path verification procedure is introduced to while detecting the change of `(IP, port)`.
    An extension for supporting multipath delivery [quic-multipath][19] is under development to provide bandwith aggregation and seamless handover for multihoming device (e.g. cellphone connected to 4G network and Wi-Fi).

[1]: https://tools.ietf.org/html/rfc768
[2]: https://tools.ietf.org/html/rfc2460
[3]: https://tools.ietf.org/html/rfc8200
[4]: https://tools.ietf.org/html/rfc6935
[5]: https://tools.ietf.org/html/rfc6936
[6]: https://tools.ietf.org/html/rfc3828
[7]: http://man7.org/linux/man-pages/man7/udplite.7.html
[8]: https://tools.ietf.org/html/rfc2960
[9]: https://tools.ietf.org/html/rfc6951
[10]: https://tools.ietf.org/html/rfc4340
[11]: https://tools.ietf.org/html/rfc6773
[12]: https://tools.ietf.org/html/draft-ietf-tsvwg-sctp-prpolicies-06
[13]: https://tools.ietf.org/html/draft-ietf-rtcweb-data-channel-13
[14]: https://tools.ietf.org/html/rfc4347
[15]: https://tools.ietf.org/html/rfc6347
[16]: https://tools.ietf.org/html/draft-ietf-quic-transport-13
[17]: https://tools.ietf.org/html/rfc7540
[18]: https://tools.ietf.org/html/rfc5246
[19]: https://tools.ietf.org/html/draft-deconinck-quic-multipath-00
[20]: https://tools.ietf.org/html/draft-ietf-quic-recovery-13
[21]: https://tools.ietf.org/html/rfc3168
[22]: https://tools.ietf.org/html/draft-joseph-quic-comparison-quic-sctp-00
