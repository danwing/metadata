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
    email: ["sridharan.girish@gmail.com"]
 -
    fullname: Dan Wing
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    email: ["danwing@gmail.com"]


normative:


informative:

  ctx-aware:
    title: "Study on Context Aware Service Delivery in RAN for LTE"
    target: https://www.etsi.org/deliver/etsi_tr/136900_136999/136933/14.00.00_60/tr_136933v140000p.pdf
    date: 2017-04

  wifi-ll:
    title: "Ending the Anomaly: Achieving Low Latency and Airtime Fairness in WiFi"
    target: https://www.usenix.org/system/files/conference/atc17/atc17-hoiland-jorgensen.pdf
    author:
      -
        name: Toke Høiland-Jørgensen
      -
        name: Michał Kazior
      -
        name: Dave Täht
      -
        name: Per Hurtig
      -
        name: Anna Brunstrom
    date: 2017-07-12

  docsis-ll:
    title: "Low Latency DOCSIS®"
    target: https://www.cablelabs.com/technologies/low-latency-docsis
    author:
      -
        name: "CableLabs"

--- abstract

As part of host-to-network signaling, an entire flow can share the same
metadata or certain packets can have certain metadata associated with those
packets.  This document describes the metadata exchangd in a
host-to-network signaling protocol in both binary and JSON.

As part of network-to-host signaling, network metadata can be communicated
to the host, allowing the host to modulate its requests to conform to
the available network resources.

--- middle

# Introduction

Host-to-network metadata signaling has historically been performed by the sender setting
DSCP bits ({{?RFC7657}})({{?RFC8837}})({{?RFC2475}}). While DSCP can express high priority (Expedited Forwarding {{?RFC3246}})
and low priority ({{?RFC3662}}), DSCP bits are frequently ignored at congestion
points or lost (stripped) during a packet's lifetime across the Internet.  Also, DSCP
attempts influences the packet's treatment compared to all other packets from other
hosts.  In contrast, the metadata described by this document influences the treatment
of all the packets in a flow (UDP 4-tuple) or individual packets within a single flow
(UDP 4-tuple), rather than influencing treatment between flows belonging to different
hosts.

Network-to-host metadata signaling has historically been performed by dropping packets or
setting the ECN bit on packets.  Both of these techniques work well to consume
the network's available bandwidth.  However, it is wasteful for the host to download
streaming video at the full available bandwidth.  If the user abandons the video
that downloaded data is wasted and it counts against the user's monthly data allotment.

In a network, if the routers can tell apart different types of traffic, like real-time (like video calls) and bulk (like file downloads), or reliable (TCP) and unreliable (UDP), it can help the network perform more efficiently.

For example, Reliable traffic, which needs all its data packets to reach their destination, might get more bandwidth. Unreliable traffic, which can afford to lose some packets, might get less. This way, the network can handle more traffic and provide a better user experience. Unreliable traffic can sometimes be associated with network attacks. Identifying this traffic can help in implementing security measures and spot potential security threats. Understanding the mix of reliable and unreliable traffic can aid in network troubleshooting and future network planning.

Real-time data packets must be delivered in a timely and consistent manner to ensure a smooth, uninterrupted user experience. Bulk traffic involves the transfer of large amounts of data, such as file downloads, where the focus is on throughput rather than speed. While it’s still important for data to be delivered reliably, it’s less critical for this to happen in real-time. Once traffic is classified, routers can use Quality of Service (QoS) mechanisms to prioritize certain types of traffic over others({{?RFC4594}}). Furthermore, routers can employ traffic shaping techniques ({{?RFC7640}}) to control the rate at which packets are sent. Differentiating and appropriately managing real-time and bulk traffic optimizes network performance, improve user experience and is particularly valuable in networks with diverse traffic patterns and varying performance requirements.

The advantages of both reliable {{?QUIC=RFC9000}} and unreliable QUIC {{?RFC9221}}
will bring QUIC to displace TCP and bespoke UDP applications. Network elements
have long been able to identify reliable flows between hosts as those are
usually TCP flows; unreliable flows have been carried over UDP (e.g., VoIP,
SIP signaling, NTP, DNS) and frequently use application-level retransmission to
achieve some amount of reliability.

There are several advatanges to an application using a single 5-tuple for
its communications with another host.  Multiplexing RTP and RTCP {{?RFC5761}}) was
found advantageous, but fortunately those protocols are both realtime and have
nearly-identical network scheduling requirements.  Another advantage is shared
congestion control state.  Applications wanting to use a single 5-tuple have
been constrained to signal packet metadata using DSCP bits which don't survive
from the sender across the Internet to the receiver's access network.  In the
event the DSCP bits do survive, the sender's intended interpretation may not
align with the receiving network's interpretation, or the receiving network
may not honor those bits because the sender is (perceived) to lack authorization
to influence the receiver's network treatment.

This document describes both per-flow and per-packet host-to-network signaling of
packet metadata.  The metadata is carried in a host-to-network protocol that
effectively creates a "pointer" (mapping) to the metadata such as {{?I-D.wing-cidfi}}
or carries the metadata directly in the packet itself
{{?I-D.reddy-tsvwg-explcit-signal}}.

By providing a signaling mechanism that survives across the Internet,
intermediaries can identify the nature of the packet without
decryption and without understanding the application protocol.

Data between two endpoints can be broadly classified into interactive
and non-interactive classes.  These are often in conflict, especially
when carried in the same 5-tuple.  A network with constrained
resources will have need to ECN-mark {{?RFC3168}} or discard packets
of that flow.  If the packet's metadata can be communicated to the
network, the network can make a more informed decision on which
packets to discard.  For example, an application might have a single
5-tuple connection that carries both realtime audio and a
bandwidth-consuming file transfer. By informing the network of the
metadata of those packets, the network can discard packets belonging
to the file transfer.  As the congestion control is affected by
those discards, the realtime traffic can continue being sent but
the file transfer can slow down.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Metadata Interpretation

[[

* Reliable subchannel packets that are loss-intolerant can be transmitted using proactive measures to avoid loss (eg. double retransmission, FEC).
* Loss-tolerant packets can be forwarded or sent duplicates based on the don't care flags – for key frames in graphics/Multimedia.
* These packets can contain the same CID but different sequence numbers (listed in 1 – either IP ID or custom sequence numbers) and can be used to update the QoS measurement for each multiple transmits.

]]

# Host to Network Metadata

## Reliable/Unreliable

Reliable packets are intended to be treated like TCP packets:  The host will re-transmit the packet until
it is successfully received or the sender gives up, typically many seconds or minutes later.  Thus the
network SHOULD make attempts to deliver this packet, otherwise the end-host will send it again, incurring
additional delay for the end user (harming end user experience) and causing more traffic on the network
overall.

* When signaled in binary, the Reliable bit is set (1).
* When signaled in JSON, it is encoded as name "reliable" and value "true".

Unreliable packets are expected to experience some loss, and are not re-transmitted by the application
because a re-transmission would take too long to be useful for the receiver.  For example if the receiver's
playout (jitter) buffer is 60ms but a retransmission would consume 100ms, the application will endeavour
to avoid retransmitting the packet and instead rely on the receiving application concealing the loss
somehow (with audio, some loss of an intermediate video update, etc.).

* When signaled in binary, the Reliable bit is cleared (0).
* When signaled in JSON, it is encoded as name "reliable" and value "false".

## Don't Care bit

Don't care bit (DC) indicates if the sender expects the packet to be delivered as reliably as possible or if the packet can be dropped without significant impact to the experience. Setting/Clearing don't care bit is not the same as tagging a traffic reliable/loss-tolerant. The application would function when an unreliable packet with don't care bit set is lost (will not require retransmission of the packet) but there would be a considerable impact to experience. On the other hand, when the nature of a packet is reliable, packet not being delivered will cause the application to fail (will require retransmission of the packet). An example for this distinction is discussed in {{IndustrialApplication}} section.

* When signaled in binary, the don't care bit is 0 or a 1.
* When signaled in JSON, it is encoded as name "DontCare" and is either "true" or "false".

## Realtime/Bulk Traffic

### Realtime Traffic

Realtime traffic requires immediate processing and transmission of packets. It’s characterized by the need for speed and accuracy, such as in user typing or online gaming.

* When signaled in binary, the Realtime bit is set (1).
* When signaled in JSON, it is encoded as name "realtime" and value "true".

### Bulk Traffic

Bulk traffic involves the transfer of large data volumes across a network, often scheduled and not time-sensitive. It’s like sending large files or printing jobs.

* When signaled in binary, the Realtime bit is cleared (0).
* When signaled in JSON, it is encoded as name "realtime" and value "false".


## Reliable Traffic

Reliable packets cannot afford to be lost. This refers to network traffic where the successful delivery of each data packet is crucial. Protocols used for this type of traffic prioritize accuracy and completeness over speed. They ensure that every packet reaches its destination, even if it requires retransmission due to packet loss or error. Examples include file transfers and email delivery.

* When signaled in binary, the Loss-Tolerant bit is cleared (0).
* When signaled in JSON, it is incoded as "Loss-Tolerant" and value "false".
* The QUIC metadata in network bits will look like:

| Loss-Tolerant | Real-Time | DC | CID |
|:-------------:|:---------:|:--:|:---:|
| 0 | W | X | Y Y ... Y |
{: #Reliable-network-bits title="Network-Bits"}

When there is loss detected between 2 nodes, using the above QoS measurement, the network will endeavor to deliver the packet reliably using any means the network decides to (some e.g., using FEC, stronger radio transmission, 3x transmits – up to the network to decide).

## Unreliable/Loss-Tolerant Traffic

Loss-Tolerant packets have the luxury of being dropped (that is, the application will not
re-transmit the packet). This refers to network traffic where the immediate delivery of data packets takes precedence over their reliable arrival. Protocols used for this type of traffic accept the risk of packet loss or error in favor of speed and real-time communication. Examples include user typing and online gaming.

* Key Loss-Tolerant packets:
   If the application decides that any loss tolerant packet needs to be delivered reliably for some reason, the application can choose to send that specific packet as reliable so that the network will know to deliver it reliably. This is the same as reliable scenario. The QUIC metadata in network bits will look same as reliable (shown above). The application can also choose to assign high-priority to the loss-tolerant packet as well. The QUIC metadata in network bits will look like

  * When signaled in binary, the Loss-Tolerant bit is set (1) and the DontCare bit is cleared (0).
  * When signaled in JSON, it is indicated as "Loss-Tolerant" and value "true" and "DontCare" vale set to "false".
  * The QUIC metadata in network bits will look like:

| Loss-Tolerant | Real-Time | DC | CID |
|:-------------:|:---------:|:--:|:---:|
| 1 | W | 0 | Y Y ... Y |
{: #key-loss-tolerant-network-bits title="Network-Bits"}

When the loss tolerant packet is marked with higher priority (implementation decided by the network node), the packet can be transmitted with corresponding loss avoidance mechanism multiple times to increase the probability of delivery (implementation decided by the network node).

* Loss-Tolerant packets:
   If the application decides the packet is loss-tolerant and not too important to be transmitted reliably, the packet will just be transmitted once irrespective of the loss between the 2 nodes. The QUIC metadata in network bits will look like

  * When signaled in binary, the Loss-Tolerant bit is set (1) and the DontCare bit is set (1).
  * When signaled in JSON, it is indicated as "Loss-Tolerant" and value "true" and "DontCare" vale set to "true".
  * The QUIC metadata in network bits will look like:

| Loss-Tolerant | Real-Time | DC | CID |
|:-------------:|:---------:|:--:|:---:|
| 1 | W | 1 | Y Y ... Y |
{: #Loss-Tolerant title="Network-Bits"}

For a real-time example of the above proposal, let's consider a desktop virtualization use case that comprises of a server (VDA) and a client connected over the internet, which has mixed traffic – interactive traffic (graphics updates, mouse and keyboard activity), bulk data transfer traffic (file copy and printing), and video streaming (interactive video – VR, gaming) and VoIP remoting (MS Teams) - interactive. The application can identify the type of traffic through various methods – type of virtual channel, use of hooks to identify operations, heuristics that detect keyboard and mouse activity etc. and based on the heuristics and nature of the channel, the application can send the packets through loss tolerant or reliable channels and determine the priority.

* Real-time traffic:
Most of the real-time traffic (graphics, VoIP) are loss-tolerant. The packets don't have much of a significance when they reach late. Real-Time bit will be set to 1. Priority of that packet will be set in the following 2 bits. These packets, based on the Metadata and the QoS, will be routed in the path with least latency. The video key frames will be routed just like a reliable transport - network decides how to avoid loss on these packets.

* Bulk transfer:
Most of the bulk transfer (file copy or printing) are reliable data with big sized packets. Real-Time bit will be set to 0. Priority of that packet will be set in the following 2 bits. The packets, based on the Metadata and the QoS, will be transmitted in the path with high BDP and MTU size.

Proactive transmission can be done at Layer 4 to avoid any possible loss (e.g., {{?FEC=RFC6363}}) or below as well {{?I-D.meng-tsvwg-wireless-collaboration}}, up to the network nodes to decide it. Local retransmissions in L2 are more efficient than L4 end to end retransmissions. Explicit signaling this way can help reduce end to end loss and also exposes application-level parameters to the network to make end to end packet delivery more efficient by making the intermediaries more intelligent and capable of handling losses and congestion more effectively, as suggested in {{?I-D.meng-tsvwg-wireless-collaboration}} as well. This provides means and data required for achieving that.

In today's network, packet prioritization and proactive loss management/avoidance is a challenge. Packet routing for reliable protocols (like TCP) has never been optimal for mixed data traffic. Retransmissions has been predominantly end-to-end with little to no room for intelligent routing, especially in reliable transports. UDP based protocols give more flexibility in having reliable and non-reliable traffic in the same connection - with different subchannels. But the existing protocols don't give an option for intermediaries to identify the nature of the packet without having to decrypt the packet, which is not a possibility, to determine the ideal route for the packet. There is no way for application to dictate which packet needs to go through which route and for the intermediaries to act upon them practically and efficiently.

Data sent between 2 endpoints can be broadly classified two types. Interactive (Real-time) data and non-interactive (bulk transfer) data. The requirements for these at times are conflicting. e.g., Interactive data needs to be time critical – any delay in delivering a packet will impact user interactivity negatively (e.g., Audio, video, graphics updates) whereas non-interactive data (e.g., File transfer, print job) can be accumulated and sent together for best throughput. One depends on how fast a data can be delivered and other depends on how effectively large data can be delivered. Based on above requirement dichotomy, performance can also be divided into two broad categories: Interactive performance, e.g., what is the click-to-photon time (the time between a user interaction such as a mouse click and the corresponding graphics update, how interactive the session is, how clear the video is and if there is a lag between action and reaction or buffering of video, and non-interactive performance, e.g., File copy speed from server to client.

When the amount of in-flight data in the network is high, the network throughput is high (sending more data per second). This may cause delay in delivering interactive packets leading to poor perceived user experience like jitter or inconsistent UI updates but will achieve better throughput for data-oriented streams. Interactivity might suffer but non-interactive performance improves. Conversely, when the amount of in-flight data in the network is low, interactivity improves while hindering non-interactive performance.

A lot of applications require both types of performances to be optimized, depending on the operation being carried out (e.g., Excel – typing and scrolling is interactive whereas loading a file or saving is non-interactive). The main purpose of a transport is to optimize performance to satisfy all types of applications. Due to the counter intuitive nature of interactive and non-interactive traffic, it is hard to get best of both the worlds. The ideal solution would be to get the best performance for the activity that is happening at that point in the session with minimal impact to the other activities happening in the background.

# Industrial Application of Flow Metadata {#IndustrialApplication}

Example packet metadata for Desktop Virtualization (like Citrix Virtual Apps and Desktops - CVAD) application.

| Traffic type    | Nature    | Loss-tolerant | DC | Comments |
|:---------------:|:---------:|:-------------:|:--:|:---------|
| User typing | real-time | Reliable | NA |          |
| Glyph critical | real-time | Loss-tolerant | 0 |  The frames that form the base for the image is more critical and needs to be transmitted as reliably as possible. Retransmits of these are harmful to the UX.**|
| Glyph smoothing | real-time | Loss-tolerant | 1 | The smoothing elements of the glyph can be lost and would still present a recognizable image, although with a lesser quality. Hence, these can be marked as loss tolerant as the user action is still completed with a small compromise to the UX. Moreover, with the reception of the next glyph critical frame would mitigate the loss in quality caused by lost glyph smoothing elements. |
| Video key frame | real-time | Loss-tolerant | 0 | Video key frames form the base frames of a video upon which the next 'n' timeframe of video updates is applied on. These frames, are hence, critical and without them, the video would not be coherent until the next critical frame is received. Retransmits of these are harmful to the UX.**|
| Video predictive frame | real-time | Loss-tolerant | 1 | Video predictive frames can be lost, which would result in minor glitch but not compromise the user activity and video would still be coherent and useful. The reception of subsequent video key frame would mitigate the loss in quality caused by lost predictive frames. |
| User moving/clicking mouse (client to VDA) | real-time | Loss-tolerant | 0 | ** |
| Mouse Pointer tracking | real-time | Loss-tolerant | 1 | When the pointer is moved from one point to another, the coordinates of the pointers between the two points can be lost without much of an impact to the UX as long as the start and endpoint reaches. This would ensure the user action is completed, even if the experience seems glitchy. |
| Pointer endpoint | real-time | loss-tolerant | 0 | The start and endpoint of the pointer movement is vital to ensure user action is completed correctly. So, the endpoints have to be reliably transmitted with real-time priority. **|
| Print job | bulk | Reliable | NA |   |
| File copy | bulk | Reliable | NA |   |
| VoIP-Audio | real-time | Loss-tolerant | 0 | ** |
| VoIP-Video | real-time | Loss-tolerant | 1 | |

** These are critical but considering implementation constraints, data from a specific source (a virtual channel like mouse, graphics etc in this case) is either transmitted reliably or as loss-tolerant. These packets lost will have user experience impact but still since most of the traffic from this use-case come under loss-tolerant and it is not critical that the application breaks if these are not received (unlike file transfer), these are listed as loss-tolerant while having the don't bit set to 0.

# Network to Host Metadata

((video streaming bandwidth. QoS maybe to help with application's congestion control and loss handling? What else??))



# Ease of adaptation and Industrial significance:

The current proposal can be extended to radio and WiFi protocols as well. The currently provided CID metadata can be used to obtain the necessary information about the nature of the packet for the radio and wireless protocols to make more intelligent and informed decisions. For example, references {{ctx-aware}} and {{wifi-ll}} can get all the required information for their respective optimizations from the CID metadata that is being proposed.

The explicit signaling through QUIC CID will be of immense benefits for various applications and corporations. These are a few which would benefit from the current proposal:

* Explicit signals can help in deciding how to handle packets – queuing, routing etc.
*It can also help reduce the queueing for streaming protocols – taking more proactive approach of delivering key frames and reliable packets, and not having to queue every packet being exchanged, thereby further reducing queueing delays, eliminating them in some cases (the explicit signal on whether the packet is loss tolerant or not makes a big impact on deciding whether to queue the packet or not). It will be useful in cases mentioned in reference {{docsis-ll}}. Non-queue building (NQB) mentioned in the overview can be signaled in the QUIC CID, as well (eg. if QUIC CID=12345, treat it same as DiffServ NQB PHB).
* The inclusion of vital metadata in QUIC CID and improving the performance through it is a step closer to {{?L4S=RFC9330}}.
* {{?L4S=RFC9330}} CC information can also be explicitly signaled in the metadata, possibly simplifying handling of the queues in the network nodes. The L4S indication {{?RFC9331}} can be added to the QUIC CID.

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




# Acknowledgments
{:numbered="false"}

To be completed.

--- back

# Example Metadata Encoding

This section illustrates how the QUIC CID Metadata would look like in
different encoding mechanisms. For purpose of illustration, network
bits and JSON format are selected as encoding mechanisms.

## User typing

* Network bits

~~~~~
+-------------+
| 0 1 0 1 2 3 4 5 |
+-------------+
~~~~~
{: #user-typing-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"Loss-Tolerant":false,"Real-Time":true,"DontCare":0},"CID":{"id":12345}}
~~~~~
{: #user-typing-json artwork-align="left" title="JSON"}

## Glyph critical

* Network bits

~~~~~
+-------------+
| 1 1 0 1 2 3 4 5 |
+-------------+
~~~~~
{: #glyph-critical-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"Loss-Tolerant":true,"Real-Time":true,"DontCare":0},"CID":{"id":12345}}
~~~~~
{: #glyph-critical-json artwork-align="left" title="JSON"}

## Glyph smoothing

* Network bits

~~~~~
+-------------+
| 1 1 1 1 2 3 4 5 |
+-------------+
~~~~~
{: #glyph-smoothing-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"Loss-Tolerant":true,"Real-Time":true,"DontCare":1},"CID":{"id":12345}}
~~~~~
{: #glyph-smoothing-json artwork-align="left" title="JSON"}

## File Transfer

* Network bits

~~~~~
+-------------+
| 0 0 0 1 2 3 4 5 |
+-------------+
~~~~~
{: #file-transfer-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"Loss-Tolerant":false,"Real-Time":false,"DontCare":0},"CID":{"id":12345}}
~~~~~
{: #file-transfer-json artwork-align="left" title="JSON"}

## VoIP - Audio

* Network bits

~~~~~
+-------------+
| 1 1 0 1 2 3 4 5 |
+-------------+
~~~~~
{: #VoIP-audio-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"Loss-Tolerant":true,"Real-Time":true,"DontCare":0},"CID":{"id":12345}}
~~~~~
{: #VoIP-audio-json artwork-align="left" title="JSON"}

## VoIP - Video

* Network bits

~~~~~
+-------------+
| 1 1 1 1 2 3 4 5 |
+-------------+
~~~~~
{: #VoIP-video-network-bits artwork-align="left" title="Network-Bits"}

* JSON syntax

~~~~~
{"metadata":{"Loss-Tolerant":true,"Real-Time":true,"DontCare":1},"CID":{"id":12345}}
~~~~~
{: #VoIP-video-json artwork-align="left" title="JSON"}


