---
layout: post
title: "A Whirlwind tour of TCP"
date:   2018-07-24 14:58:33 +0800
categories: networking
---

TCP
---

TCP [RFC793][1] is a transport protocol for reliable, in-sequence delivery.
It provides a bidirectional byte stream transmission between endpoints across Internet.
The original RFC provides an implementation guideline with message format and connection states defined.
[RFC1122][2] provides more formal regulation and bugfix based on the observation of the deployment.
[RFC7414][3] provides an historical overview of the TCP development.

### header format

     0               8               16              24            31
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           Source Port         |        Destination Port       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                        Sequence Number                        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                    Acknowledgement Number                     |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |offset | rsv |  control bits   |           Window Size         |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Checksum          |         Urgent Pointer        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

-   Source Port / Destination Port
-   __Sequence number__:
-   __Acknowledgement number__:
-   __Data offset__:
-   __Reserved__: reserved bits for new control bits in the future. Set to 0 for now.
-   __Control bits__:
--    NS:
--    CWR:
--    ECE:
--    URG: set to `1` if urgent point is meaningful
--    ACK: set to `1` if acknowledgement number is meaningful
--    PSH: set to `1` to push the buffer to receiving application
--    RST: set to `1` to reset the connection
--    SYN: set to `1` to synchronoze the initial sequence number
--    FIN: set to `1` to close outgoing connection, no more data can be sent.
-   __Window Size__:
-   __Checksume__:
-   __Urgent Pointer__:
sequence number
acknoledgement number
data offset
control bits
urgent data pointer
optional

### connection management

TCB
state transition
connection setup
connection shutdown


### in-sequence delivery

The sequence number represents the relative offset of the first octet of the data carried by this TCP segment.
Upon receiving new data, an acknoledgement should be generated with the sequence number of next expected octet.
A delayed ACK mechanism is used to reduced the number of ACKs sent and piggyback with the response data, see RFC1122 section 4.2.3.2.
TCP optionally support selective acknoledgement (SACK, [RFC2018][5]) for better detecting multiple packet lost.

Since the data is delivered inorderly, receiver can only consume a byte until all the bytes before it are received.
(Imaging the server is trying to send a warning of nuclear bomb explosion while you are downloading a 4GB movie from it.)
Urgent data is designed to transport out-of-band message, which allow high-priority control message to be delivered
during the transmission of regular data.
However the current best practice is against the use of TCP urgent mechanism [RFC6093][4] due to various design and implementation issues.
One naive way is create additional TCP connection for the event notification but it consumes additional server resources.
The other way is to create an application protocol that multiplex multiple streams onto single TCP connection, e.g. HTTP2.

### flow control

TCP endpoint needs to store the received but not-yet-consumed data in a buffer called sliding window, which represents a slice
of the byte stream. There is no point for a sender to transmit more data if the buffer in receiver is full.
The `Window` field in combined with the `Acknoledgement Number` represents the last index of octet that receiver will accept.
Receiver updates the `Window` field to trigger sender sending more data. However, advancing window boundary with small increment
will cause small segment of data to be sent, which decrease the network efficiency (packet overhead, more ACK).
In combined with the delayed ACK mechanism, a SWS avoidance algorithm is introduced in RFC1122 section 4.2.3.3. The idea is to
update the window boundary after the size of unadvertised buffer space is less than MSS or the low watermark of receiving buffer.

The window update packet might be lost in the network. The sender should perform window probing after `rwnd` reaches zero for sometime.
A packet with one byte of data is sent. Receiver will consume this one-byte data if window is avaialble and ACK normally. Receiver must
send an ACK even if the current window is zero, this ACK should contains the next expected sequence number with `0` window.
This probing should start after the retransmission timeout after `rwnd` down to 0, RFC1122 suggest that the timeout value should increase
exponentially for consecutive probing.

### congestion control

The middleboxes (router, proxy, firewall, etc) along the routing path also have buffering issues and the buffer is
shared among all the packet flows go pass the node. Sending packets in a burst is more likely to fill the buffer of the middlebox
and causes packet drop/retrasmission. This is bad for throughput by increasing delay and unnecessary traffic for error handling.

TCP maintains a congestion window `cwnd` as a sweet spot for sending as many data as possible without jamming the network.
The congestion window is dynamcially adjusted during the lifetime of the TCP connection.
Two phases is used:
1.  __slow start__:
    Starting with the inital window size ( 1 segment in RFC2001 ), grow the `cwnd` exponentially by increasing 1 everytime receiving
    an ACK (doubled each round trip time). While detecting packet drop (i.e. retransmission timeout), set the high watermark `ssthresh`
    to half of current `cwnd`, reset `cwnd` to initial value and redo slow start until `cwnd` reach the `ssthresh`, switch to
    congestion avoidance phase.

2.  __congestion avoidance__
    We are about to reach the maximum throughput. The increment of `cwnd` is increased at most one segment each round trip time.
    This will ensure a linear growth of `cwnd`. Once the packet drop starts again, set `ssthresh` to half of current `cwnd` and restart
    the procedure with `cwnd` set to the inital value.

Single duplicate ACK can represents packet drop or packet reorder. Multiple duplicate ACKs on the same sequence number is likely to
be a minor congestion incidence. The fast retransmission algorithm, described in RFC2001 section 3, will retransmit the desired segment
after more than three duplicate ACKs is received without waiting for retransmittion timeout.

After fast trasmission happened, minor congestion is detected. The fast recovery algorithm described in RFC2001 section 4 switch the phase
to congestion avoidance by setting `ssthresh` to the half of `cwnd` and change the `cwnd` to `ssthresh` after the retrasmitted segment is acked.
The latest algorithm in [RFC5681][6] further tweak the `ssthresh` and `cwnd` value by considering the number of data that is sent before
fast retrasmission happend.

ECN [RFC3168][7] is introduced to notify packet drop in the middlebox via IP layer. Two control bits `ECE` and `CWR` are introduced in the
TCP header. While reciever get Congestion Experienced (CE) notification from IP layer, `ECE` flag is on for every following outgoing packet
until `CWR` bit is observed in the incoming packet. This feedback mechanism help reduce the congestion window before packet drop since the
CE flag is set when middlebox detecting impending congestion.

Various congestion control algorithms are developed for different network environment and requirement, which can be found on [wikipedia][8].

### security consideration
checksum

The pseudo header format for calculating checksum is the same as the one for UDP.
The only difference is the protocol number (0x06 for TCP)

For IPv4:

     0               8               16              24            31
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         Source Address                        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                      Destination Address                      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     zero      |  Protocol (6) |           TCP Length          |
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
    |                      zero                     |Next Header (6)|
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

message authenticity
data integrity
data encryption - TLS

### performance consideration

With the increasing of available memory on network device and the network bandwith, the `Window` value in TCP header
becomes the performance bottle neck (65535 bytes at most in one RTT).
[RFC1323][9] introduce the window scaling extension, provide a `shift` value of the `Window` field. The maximum `shift`
value is 14, which increase the window size to 2^30 = 1 Gbytes.

packet utilization, The Nagle algorithm described in RFC1122 Section 4.2.3.4 provides a solution to send larger packet
by delay sending the fregment.

The original version of congestion control use one SMSS as the initial window size, which is too small as the network
bandwith increases. It was increased to 2 in [RFC2414][10] , to 4 in [RFC3390][11], and to 10 in [RFC6928][12].
This will decrease the number of RTT required to pass slow start phase.

Various researches try to resolve the slowness of slow start mechanism by probing/guessing/predicting a better initial `cwnd`
[Fast Start][] [Jump Start][] [Quick Start][]
However there is no one can generally replace the slow start based on the experiment result.

One common performance issue while using TCP is the initial delay for the three-way handshake procedure, which means 1.5 RTT is
required for server to receive the first byte of data. Therefore, creating a long-lived TCP connection seems a good approach
to avoid the latency. In order to determine the connectivity between two endpoints, a duplicate ACK to  with no data is sent
periodically while idle. RFC1122 section 4.2.3.6 regulates how keep-alive message should be implemented.

Another approach to reduce the delay is to carry the application data along with the handshake message. This experimental
mechanism is called TCP Fast Open ([TFO][13]). However there are still security issues and deployment issues requires to be
fixed before it can be widely used.

multipath
[RFC6824][14]

[1]: https://tools.ietf.org/html/rfc793
[2]: https://tools.ietf.org/html/rfc1122
[3]: https://tools.ietf.org/html/rfc7414
[4]: https://tools.ietf.org/html/rfc6093
[5]: https://tools.ietf.org/html/rfc2018
[6]: https://tools.ietf.org/html/rfc5681
[7]: https://tools.ietf.org/html/rfc3168
[8]: https://en.wikipedia.org/wiki/TCP_congestion_control
[9]: https://tools.ietf.org/html/rfc1323
[10]: https://tools.ietf.org/html/rfc2414
[11]: https://tools.ietf.org/html/rfc3390
[12]: https://tools.ietf.org/html/rfc6928
[13]: https://tools.ietf.org/html/rfc7413
[14]: https://tools.ietf.org/html/rfc6824

