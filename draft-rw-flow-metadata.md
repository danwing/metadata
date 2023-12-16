---
title: "Flow Metadata"
abbrev: "Flow Metadata"
category: std


docname: draft-rw-flow-metadata-latest
submissiontype: IETF
v: 3
area: "Network"
workgroup: "Network Working Group"
keyword:
 - user experience
 - bandwidth
 - priority
 - enriched feedback
 - media streaming
 - realtime media
 - QoS
 - 5G
 - Wi-Fi
 - WiFi
 - DTLS Connection Identifier
 - DTLS-SRTP
 - QUIC Connection Identifier
 - QUIC

venue:
  group: "TSV Area"
  type: "Working Group"
  mail: "tsvwg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tsvwg/"
  github: "danwing/cidfi"
  latest: "https://danwing.github.io/cidfi/draft-wing-cidfi.html"

stand_alone: yes
pi: [toc, sortrefs, symrefs, strict, comments, docmapping]

author:
 -
    fullname: Sridharan Rajagopalan
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    email: ["sridharan.rajagopalan@cloud.com"]
 -
    fullname: Dan Wing
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    email: ["danwing@gmail.com"]


normative:


informative:


--- abstract

Metadata about packet importance is useful to signal to network components.

Including metadata (MD) in a QUIC CID, along with some heuristics and QoS measurements at the router (in-band and out-of-band) can considerably improve user experience. Appropriate data paths can be chosen based on nature of the traffic and loss can be proactively handled between the intermediary nodes avoiding end-to-end retransmissions as much as possible. This document describes how to exchange metadata in JSON which can be
mapped to a Connection Identifier (CID), and how that same metadata can
be carried in each packet as a UDP option.

--- middle

# Introduction

In today's network, packet prioritization and proactive loss management is a challenge. Packet routing for reliable protocols (like TCP) has never been optimal for mixed data traffic. Retransmissions has been predominantly end-to-end with little to no room for intelligent routing, especially in reliable transports. UDP based protocols give more flexibility in having reliable and non-reliable traffic in the same connection - with different subchannels. But the existing protocols don't give an option for intermediaries to identify the nature of the packet without having to decrypt the packet, which is not a possibility, to determine the ideal route for the packet. There is no way for application to dictate which packet needs to go through which route and for the intermediaries to act upon them practically and efficiently.

Data sent between 2 endpoints can be broadly classified two types. Interactive (Real-time) data and non-interactive (bulk transfer) data. The requirements for these at times are conflicting. e.g., Interactive data needs to be time critical – any delay in delivering a packet will impact user interactivity negatively (e.g., Audio, video, graphics updates) whereas non-interactive data (e.g., File transfer, print job) can be accumulated and sent together for best throughput. One depends on how fast a data can be delivered and other depends on how effectively large data can be delivered. Based on above requirement dichotomy, performance can also be divided into two broad categories: Interactive performance, e.g., what is the click-to-photon time (the time between a user interaction such as a mouse click and the corresponding graphics update, how interactive the session is, how clear the video is and if there is a lag between action and reaction or buffering of video, and non-interactive performance, e.g., File copy speed from server to client.

When the amount of in-flight data in the network is high, the network throughput is high (sending more data per second). This may cause delay in delivering interactive packets leading to poor perceived user experience like jitter or inconsistent UI updates but will achieve better throughput for data-oriented streams. Interactivity might suffer but non-interactive performance improves. Conversely, when the amount of in-flight data in the network is low, interactivity improves while hindering non-interactive performance.

A lot of applications require both types of performances to be optimized, depending on the operation being carried out (e.g., Excel – typing and scrolling is interactive whereas loading a file or saving is non-interactive). The main purpose of a transport is to optimize performance to satisfy all types of applications. Due to the counter intuitive nature of interactive and non-interactive traffic, it is hard to get best of both the worlds. The ideal solution would be to get the best performance for the activity that is happening at that point in the session with minimal impact to the other activities happening in the background.

# Overview

This document briefs over items that can work together to improve network performance drastically. Although this need not necessarily be confined to QUIC, we want to discuss how QUIC has the capability to make it more efficient and effective. The broad items:

* Measuring QoS between two network nodes (at least one of it being "cid-aware"). QoS measurement at IP layer with minimal interface to transport layer, leaving an option to make it transport agnostic.
* QoS can be in-band, for "cid-aware" nodes, and out-of-band using loopback protocols if one of the nodes is not "cid-aware".
* Explicit signaling is vital since there is no way to know the nature of a packet today – if it's a key frame or if it's part of a lossy subchannel. The intermediaries more often can't make an informed, intelligent and a useful decision without the metadata, even after having all the QoS measurements and heuristics in place.
* QUIC CIDs lying outside of DTLS encryption is an added advantage - gives ease of access.
* Using the above information to successfully identify loss prone nodes in a network and come up with efficient heuristics to handle packet transmission between these nodes themselves, instead of propagating it end to end, to achieve better network performance with minimal impact on processing and bandwidth.
* The computation is simple and can be confined to the link cards and not required to go to the CPU.
* Not all the network elements will be intelligent (cid-aware) but even if we have a few of them, we could still achieve better performance by proactively avoiding end-to-end retransmissions between the "cid-aware" nodes.

~~~~~ aasvg
(1)  Proxied Connection
                       .--------------.                   +------+
                      |                |                +-+----+ |
+------+              |   Network(s)   |              +-+----+ +-+
|Client+--------------)----------------(--------------+Server+-+
+---+--+              |                |              +---+--+
    |                  '-------+------'                   |
    |                          |                          |
    +<===User Data+Metadata=========User Data+Metadata===>+
    |                          |                          |

(2)  Client-centric Metadata Sharing
                          .--------------.                  +------+
                         |                |               +-+----+ |
+------+                 |   Network(s)   |             +-+----+ +-+
|Client+-----------------)----------------(-------------+Server+-+
+---+--+                 |                |             +---+--+
    |                     '-------+------'                  |
    |                             |                         |
    +<--------- Metadata -------->+                         |
    |        Secure Connection    |                         |
    |                             |                         |
    +<== End-to-End Secure Connection User Data with     ==>+
    |                          Metadata                     |
    |                             |                         |
~~~~~
{: #design-approaches artwork-align="center" title="Candidate Design Approaches"}

{{fig-arch}} provides a sample network diagram of a Metadata system showing two bandwidth-constrained networks (or links) depicted by "B" and Metadata-aware devices immediately upstream of those links, and another bandwidth-constrained link between a smartphone handset and its Radio Access Network (RAN).  This diagram shows the same protocol and same mechanism can operate with or without 5G, and can operate with different administrative domains such as Wi-Fi, an ISP edge router, and a 5G RAN.

For the sake of illustration, {{fig-arch}} simplifies the representation
of the various involved network segments. It also assumes that multiple
server instances are enabled in the server network but the document
does not make any assumption about the internal structure of the service
nor how a flow is processed by or steered to a service instance. However,
CIDFI includes provisions to ensure that the service instance that is
selected to service a client request is the same instance that will
receive CIDFI metadata for that client.

~~~~~ aasvg
                       |                     |          |
+-------+   +--------+ | +--------+          |          |
|Meta-  |   |Meta-   | | |Metadata|          |          |
|data   |   |data    | | |        |          |          |
|aware  +-B-+aware   +-B-+aware   | +------+ |          |   +----------+
|client |   |Wi-Fi   | | |edge    +-+router+-----+      |  +---------+ |
+-------+   |access  | | |router  | +------+ |   |      | +--------+ | |
            |point   | | +--------+          |   |      | |Metadata| | |
            +--------+ |                     | +-+----+ | |aware   | | |
                       |                     | |router+---+QUIC or | | |
+-----------+          | +--------+          | +-+----+ | |DTLS    | |-+
| Metadata  |          | |Metadata|          |   |      | |server  |-+
| aware     |          | |aware   | +------+ |   |      | +--------+
| client    +-----B------+RAN     +-+router+-----+      |
|(handset)  |          | |router  | +------+ |          |
+-----------+          | +--------+          |          |
                       |                     |          |
                       |                     | Transit  | Server
     User Network      |      ISP Network    | Network  | Network
~~~~~
{: #fig-arch artwork-align="center" title="Network Diagram" :height=88}


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# QoS Measurement

QoS measurement between 2 (or more) nodes without relying on transport layer: (QoS – Latency, Loss, MTU, BDP etc)

* In-band: For the example, the QoS under measurement is loss. Consider 2 routers (A and B) in the network that are having loss between them. Both routers establish a handshake between them using any given communication protocol for OOB communication. They exchange their capability to support QoS measurement and the predefined algorithm they use to measure the QoS (loss in this case).  Router A receives packet of size considerably less than the MTU of the network. Router A appends a sequence number to the packets and sends them to router B. Router B, upon receiving the packets, updates its table with the tag and strips the sequence number added and forwards it to the next hop. Upon transmission of 'n' packets, router A sends a 'fin' sequence number and router B returns the sequence numbers that were missed and hence, how much loss is there between the nodes. We can also use methods discussed in [5] and [6].
* A similar approach can be used to measure QoS out of band as well, upon demand, using synthetically generated packets and the communication protocol that the routers use to pass control information. UDP echo can also be used for this purpose [4].
* For a case where one of the routers is 'intelligent' and other one is not, any of the loopback protocols - like a UDP echo service [4] and measure the by sending synthetically generated packets.
* We can also leverage using IP ID extn for passing sequence numbers and more metadata if needed [1].
* We can leverage both OOB and IB, based on the heuristics, to measure QoS between 2 nodes.
* One can even modify the TTL accordingly and decide between which nodes, the QoS needs to be measured. QoS measurement happens only for the nodes where the packet's TTL is 255. The 2nd node (in this case, B) can decide to reset the TTL or leave it as it is before sending it to the next node.

# Metadata Design

Using QUIC CID to pass metadata required to route the packets effectively.

* Metadata can have a handshake in the first packet with a short header [13]. It can be a 2-way or a 3-way handshake. The handshake will contain various information related to:
   * Encryption information for the Metadata (refer encryption below for more details).
   * Information related to utilizing Metadata (refer ISP section below for more details).
* A bit to indicate whether the metadata needs to be processed or ignored. This bit will decide whether a particular traffic needs to be optimized or not. If this bit is set to 0, the metadata is ignored, and the traffic is just treated as it is today.
* The packet information:
   * One bit to say if the packet belongs to loss-tolerant or reliable stream.
   * One bit to signify whether the traffic is real-time or bulk transfer.
   * 2 bits indicating the priority based on the nature of traffic. This helps in determining not only the ideal route but also determine type of loss avoidance mechanism and resources to be deployed.
* Rest of the bits are reserved for now. We can use them to determine heuristics or for determining versions etc. Other data that are good to have but not necessary data can be included in an optional metadata extension, indicated by a bit, for future extension.

# Metadata Interpretation

* Reliable subchannel packets that are loss-intolerant can be transmitted using proactive measures to avoid loss (eg. double retransmission, FEC) based on the priority bits.
* Loss-tolerant packets can be forwarded or sent duplicates based on the priority flags – for key frames in graphics/Multimedia.
* These packets can contain the same CID but different sequence numbers (listed in 1 – either IP ID or custom sequence numbers) and can be used to update the QoS measurement for each multiple transmits.

# Metadata exchanged

## Metadata Hanshake

Metadata handshake is done at the first QUIC packet that goes out with short header. The handshake metadata packet contains information on

1. Information for metadata support
   * Authorize/reject metadata optimization.
   * Support for optimization.
2. Information for encryption of metadata
   * Obfuscation/Encryption mechanism.
   * Rotation Mechanism.
   * Obfuscation/Encryption specific data.
3. ISP token exchanged between server/client and ISP that contains data such as:
   * Optimization - enabled/disabled.
   * Type of optimization.
   * Extent of optimization.

## Metadata structure for different types of traffic

The packets can be broadly classified as 2 types – loss tolerant (lossy) and reliable (lossless).

### Reliable

Lossless/Reliable packets cannot afford to be lost and hence, every packet needs to be treated important (that is, the application
will re-transmit the packet if it is lost). Based on the traffic, the Real-Time Tx bit is set. The QUIC metadata in network bits will look like

| MD Enable | Loss-Tolerant | Real-Time Tx | Priority | CID |
|:---------:|:-------------:|:------------:|:--------:|:---:|
| 1 | 0 | W | X X | Y Y ... Y |
{: #Reliable-network-bits title="Network-Bits"}
When there is loss detected between 2 nodes, using the above QoS measurement, the network will endeavor to deliver the packet reliably using any means the network decides to (some e.g., using FEC, stronger radio transmission, 3x transmits – up to the network to decide). If the traffic is real-time, then the PDB is taken into account and packet is handled accordingly.

### Loss-Tolerant

Loss-Tolerant packets have the luxury of being dropped (that is, the application will not
re-transmit the packet).

* Key Loss-Tolerant packets:
   If the application decides that any loss tolerant packet needs to be delivered reliably for some reason, the application can choose to send that specific packet as reliable so that the network will know to deliver it reliably. This is the same as reliable scenario. The QUIC metadata in network bits will look same as reliable (shown above). The application can also choose to assign high-priority to the loss-tolerant packet as well. The QUIC metadata in network bits will look like

| MD Enable | Loss-Tolerant | Real-Time Tx | Priority | CID |
|:---------:|:-------------:|:------------:|:--------:|:---:|
| 1 | 1 | W | 1 1 | Y Y ... Y |
{: #key-loss-tolerant-network-bits title="Network-Bits"}

When the loss tolerant packet is marked with higher priority (implementation decided by the network node), the packet can be transmitted with corresponding loss avoidance mechanism multiple times to increase the probability of delivery (implementation decided by the network node).

* Loss-Tolerant packets:
   If the application decides the packet is loss-tolerant and not too important to be transmitted reliably, the packet will just be transmitted once irrespective of the loss between the 2 nodes. The QUIC metadata in network bits will look like

| MD Enable | Loss-Tolerant | Real-Time Tx | Priority | CID |
|:---------:|:-------------:|:------------:|:--------:|:---:|
| 1 | 1 | W | 0 0 | Y Y ... Y |
{: #Loss-Tolerant title="Network-Bits"}

For a real-time example of the above proposal, let's consider a desktop virtualization use case that comprises of a server (VDA) and a client connected over the internet, which has mixed traffic – interactive traffic (graphics updates, mouse and keyboard activity), bulk data transfer traffic (file copy and printing), and video streaming (interactive video – VR, gaming) and VoIP remoting (MS Teams) - interactive. The application can identify the type of traffic through various methods – type of virtual channel, use of hooks to identify operations, heuristics that detect keyboard and mouse activity etc. and based on the heuristics and nature of the channel, the application can send the packets through loss tolerant or reliable channels and determine the priority.

* Real-time traffic:
Most of the real-time traffic (graphics, VoIP) are loss-tolerant. The packets don't have much of a significance when they reach late. Real-Time Tx bit will be set to 1. Priority of that packet will be set in the following 2 bits. These packets, based on the Metadata and the QoS, will be routed in the path with least latency. The video key frames will be routed just like a reliable transport - network decides how to avoid loss on these packets.

* Bulk transfer:
Most of the bulk transfer (file copy or printing) are reliable data with big sized packets. Real-Time Tx bit will be set to 0. Priority of that packet will be set in the following 2 bits. The packets, based on the Metadata and the QoS, will be transmitted in the path with high BDP and MTU size.

Proactive transmission can be done at Layer 4 to avoid any possible loss (e.g., FEC [7]) or below as well [14], up to the network nodes to decide it. Local retransmissions in L2 are more efficient than L4 end to end retransmissions. Explicit signaling this way can help reduce end to end loss and also exposes application-level parameters to the network to make end to end packet delivery more efficient by making the intermediaries more intelligent and capable of handling losses and congestion more effectively, as suggested in [14] as well. This provides means and data required for achieving that.

In today's network, packet prioritization and proactive loss management/avoidance is a challenge. Packet routing for reliable protocols (like TCP) has never been optimal for mixed data traffic. Retransmissions has been predominantly end-to-end with little to no room for intelligent routing, especially in reliable transports. UDP based protocols give more flexibility in having reliable and non-reliable traffic in the same connection - with different subchannels. But the existing protocols don't give an option for intermediaries to identify the nature of the packet without having to decrypt the packet, which is not a possibility, to determine the ideal route for the packet. There is no way for application to dictate which packet needs to go through which route and for the intermediaries to act upon them practically and efficiently.

Data sent between 2 endpoints can be broadly classified two types. Interactive (Real-time) data and non-interactive (bulk transfer) data. The requirements for these at times are conflicting. e.g., Interactive data needs to be time critical – any delay in delivering a packet will impact user interactivity negatively (e.g., Audio, video, graphics updates) whereas non-interactive data (e.g., File transfer, print job) can be accumulated and sent together for best throughput. One depends on how fast a data can be delivered and other depends on how effectively large data can be delivered. Based on above requirement dichotomy, performance can also be divided into two broad categories: Interactive performance, e.g., what is the click-to-photon time (the time between a user interaction such as a mouse click and the corresponding graphics update, how interactive the session is, how clear the video is and if there is a lag between action and reaction or buffering of video, and non-interactive performance, e.g., File copy speed from server to client.

When the amount of in-flight data in the network is high, the network throughput is high (sending more data per second). This may cause delay in delivering interactive packets leading to poor perceived user experience like jitter or inconsistent UI updates but will achieve better throughput for data-oriented streams. Interactivity might suffer but non-interactive performance improves. Conversely, when the amount of in-flight data in the network is low, interactivity improves while hindering non-interactive performance.

A lot of applications require both types of performances to be optimized, depending on the operation being carried out (e.g., Excel – typing and scrolling is interactive whereas loading a file or saving is non-interactive). The main purpose of a transport is to optimize performance to satisfy all types of applications. Due to the counter intuitive nature of interactive and non-interactive traffic, it is hard to get best of both the worlds. The ideal solution would be to get the best performance for the activity that is happening at that point in the session with minimal impact to the other activities happening in the background.

# Industrial Application of Flow Metadata

Example packet metadata for Desktop Virtualization (like Citrix Virtual Apps and Desktops - CVAD) application.

| Traffic type    | Nature    | Loss-tolerant | Priority | Comments |
|:---------------:|:---------:|:-------------:|:--------:|:---------|
| User typing | real-time | Reliable (retransmit harmful to UX) | High (3) |          |
| Glyph critical | real-time | Reliable (retransmit harmful to UX) | High (3) |  The frames that form the base for the image is more critical and needs to be transmitted with high priority. Retransmits of these are harmful to the UX and hence need to be high priority. |
| Glyph smoothing | real-time | Loss-tolerant | Medium (1) | The smoothing elements of the glyph can be lost and would still present a recognizable image, although with a lesser quality. Hence, these can be marked as loss tolerant as the user action is still completed with a small compromise to the UX. Moreover, with the reception of the next glyph critical frame would mitigate the loss in quality caused by lost glyph smoothing elements. |
| Video key frame | real-time | Reliable | High (3) | Video key frames form the base frames of a video upon which the next 'n' timeframe of video updates is applied on. These frames, are hence, critical and without them, the video would not be coherent until the next critical frame is received. |
| Video predictive frame | real-time | Loss-tolerant | Medium (1) | Video predictive frames can be lost, which would result in minor glitch but not compromise the user activity and video would still be coherent and useful. The reception of subsequent video key frame would mitigate the loss in quality caused by lost predictive frames. |
| User moving/clicking mouse (client to VDA) | real-time | Reliable (retransmit harmful to UX) | High (3) |    |
| Mouse Pointer tracking | real-time | Loss-tolerant (retransmit is ok for UX) | Low (0) | When the pointer is moved from one point to another, the coordinates of the pointers between the two points can be lost without much of an impact to the UX as long as the start and endpoint reaches. This would ensure the user action is completed, even if the experience seems glitchy. |
| Pointer endpoint | real-time | Reliable (retransmit harmful to UX) | High (3) | The start and endpoint of the pointer movement is vital to ensure user action is completed correctly. So, the endpoints have to be reliably transmitted with real-time priority. |
| Print job | bulk | Reliable | Medium (1) |   |
| File copy | bulk | Reliable | Low (0) |  |
| VoIP | real-time | Loss-tolerant (loss is harmful to UX) | Medium-High (2) |   |

# Metadata Encoding

This section illustrates how the QUIC CID Metadata would look like in different encoding mechanisms. For purpose of illustration, network bits and JSON format are selected as encoding mechanisms.

Priority scaling: Low is 0, Medium is 1, Medium-High is 2. High is 3.

## User typing

* Network bits

~~~~~
+---------------+
| 1 0 1 1 1 1 2 3 4 5 |
+---------------+
~~~~~
{: #user-typing-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"MD Enable":true,"Loss-Tolerant":false,"Real-Time":true,"Priority":3},"CID":{"id":12345}}
~~~~~
{: #user-typing-json artwork-align="left" title="JSON"}

## Glyph smoothing

* Network bits

~~~~~
+---------------+
| 1 1 1 0 1 1 2 3 4 5 |
+---------------+
~~~~~
{: #glyph-smoothing-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"MD Enable":true,"Loss-Tolerant":true,"Real-Time":true,"Priority":1},"CID":{"id":12345}}
~~~~~
{: #glyph-smoothing-json artwork-align="left" title="JSON"}

## Print job

* Network bits

~~~~~
+---------------+
| 1 0 0 0 1 1 2 3 4 5 |
+---------------+
~~~~~
{: #print-job-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"MD Enable":true,"Loss-Tolerant":false,"Real-Time":false,"Priority":1},"CID":{"id":12345}}
~~~~~
{: #print-job-json artwork-align="left" title="JSON"}

## File Transfer

* Network bits

~~~~~
+---------------+
| 1 0 0 0 0 1 2 3 4 5 |
+---------------+
~~~~~
{: #file-transfer-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"MD Enable":true,"Loss-Tolerant":false,"Real-Time":false,"Priority":0},"CID":{"id":12345}}
~~~~~
{: #file-transfer-json artwork-align="left" title="JSON"}

## VoIP

* Network bits

~~~~~
+---------------+
| 1 1 1 1 0 1 2 3 4 5 |
+---------------+
~~~~~
{: #VoIP-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"MD Enable":true,"Loss-Tolerant":true,"Real-Time":true,"Priority":2},"CID":{"id":12345}}
~~~~~
{: #VoIP-json artwork-align="left" title="JSON"}

# Ease of adaptation and Industrial significance:

The current proposal can be extended to radio and WiFi protocols as well. The currently provided CID metadata can be used to obtain the necessary information about the nature of the packet for the radio and wireless protocols to make more intelligent and informed decisions. For example, references [8] and [9] can get all the required information for their respective optimizations from the CID metadata that is being proposed.

The explicit signaling through QUIC CID will be of immense benefits for various applications and corporations. These are a few which would benefit from the current proposal:

* Explicit signals can help in deciding how to handle packets – queuing, routing etc.
*It can also help reduce the queueing for streaming protocols – taking more proactive approach of delivering key frames and reliable packets, and not having to queue every packet being exchanged, thereby further reducing queueing delays, eliminating them in some cases (the explicit signal on whether the packet is loss tolerant or not makes a big impact on deciding whether to queue the packet or not). It will be useful in cases mentioned in reference [10]. Non-queue building (NQB) mentioned in the overview can be signaled in the QUIC CID, as well (eg. if QUIC CID=12345, treat it same as DiffServ NQB PHB).
* The inclusion of vital metadata in QUIC CID and improving the performance through it is a step closer to L4S [11].
* L4S [11] CC information can also be explicitly signaled in the metadata, possibly simplifying handling of the queues in the network nodes. The L4S indication [12] can be added to the QUIC CID.

# ISP implementation of Metadata based optimization:

## Metadata Subscription
The ISP can provide the metadata optimization as an enhanced feature that the client can subscribe to, for a fee.

1. Client and ISP have token exchanges periodically. The token contains details on:
   * What time of traffic to be optimized.
   * Optimization is to be done for how much bandwidth.
2. This token is exchanged with the server as part of the Metadata handshake (explained above) and the server shares this token with the ISP. The ISP is aware of what is subscribed by both server and client through the token.
3. The client/server now, will tag each packet to indicate whether the packet needs to be optimized/processed via Metadata Enable bit. When the bit is set to 1, further information about the packet is populated in the CID metadata.
   * Application protocol can have more information exchanged between client and server that helps the server to decide which packets to prioritize.
   * For example, in application virtualization, the client can send information about what window/app is in the foreground (say video player) and what is in the background (say webpage with animated advertisements) server can prioritize only packets containing data from that particular application (video player in this case). This intelligence can be added to the application's protocol.
4. The option to pause optimization can be made available to server, ISP and client at any point. Users can also pause the optimization based on their need (e.g. a gamer can pause it when they are away from keyboard or not playing to conserve the allocated optimization bandwidth).

## Handling/Preventing misuse of metadata optimization:

1. The ISP, now aware of the nature of optimization, can now look at the Metadata and prioritize accordingly. If the ISP sees the server is misusing the metadata optimization (e.g. tagging every packet as important packet so all of them are optimized), the ISP can police or punish the client/server for misusing - can either charge extra or disable optimization for the server/client.
2. The client also has the option to decline optimization through the handshake or at any point if it detects any misuse.

# Security Considerations

## Metadata Encryption

Encryption/Obfuscation of metadata is vital to prevent attackers from knowing the nature of the packet, since CID lies outside the DTLS encryption.

1. Obfuscation:
   * Exchange the Meta data format in Handshake. Handshake can also include the signal that indicates rotation of obfuscation mechanism.
   * Obfuscation mechanism can also be exchanged through the metadata handshake.

2. Encryption:
   * Encryption key can be derived using a standard algorithm, from the DTLS key exchanged.
   * The standard algorithm used to derive the key can be exchanged during the metadata handshake.
   * The derived key can be communicated to the nodes through secure protocol (application protocol or any secure protocol between the nodes).
   * The derived key can also be sent to the nodes through the ISP token exchanged. Client/Server can send the key to the ISP through secure protocol (same or different from the protocol used to exchange tokens) and ISP can push the token to the intermediary nodes through internal command protocol.

# IANA Considerations

None.

# References

1. draft-templin-intarea-ipid-ext-12 - Identification Extension for the Internet Protocol (ietf.org)
2. CID Flow Indicator (CIDFI) (ietf.org)
3. draft-shi-quic-structured-connection-id-01 - Structured Connection ID Carrying Metadata (ietf.org)
4. RFC 862 - Echo Protocol (rfc-editor.org)
5. RFC 7456 - Loss and Delay Measurement in Transparent Interconnection of Lots of Links (TRILL) (ietf.org)
6. RFC 8321 - Alternate-Marking Method for Passive and Hybrid Performance Monitoring (ietf.org)
7. https://datatracker.ietf.org/doc/rfc6363
8. https://www.etsi.org/deliver/etsi_tr/136900_136999/136933/14.00.00_60/tr_136933v140000p.pdf
9. https://www.usenix.org/conference/atc17/technical-sessions/presentation/hoilan-jorgesen
10. Low Latency DOCSIS® - CableLabs
11. RFC 9330 - Low Latency, Low Loss, and Scalable Throughput (L4S) Internet Service - Architecture (ietf.org)
12. RFC 9331 - The Explicit Congestion Notification (ECN) Protocol for Low Latency, Low Loss, and Scalable Throughput (L4S) (ietf.org)
13. RFC 8999 - Version-Independent Properties of QUIC (rfc-editor.org)
14. draft-meng-tsvwg-wireless-collaboration-00 (ietf.org)

# Acknowledgments
{:numbered="false"}

To be completed.


