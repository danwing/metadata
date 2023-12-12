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




## Loss-Tolerant

Indicates the packet is loss-tolerant (that is, the application will not
re-transmit the packet) or is not loss-tolerant (that is, the application
will re-transmit the packet if it is lost).

### JSON Syntax

### UDP Options




## Real-Time

Indicates real-time or bulk.

### JSON Syntax

### UDP Options



## Priority

1. Relative priority within the same stream.  Applications running over
UDP can indicate which packets within the same UDP 4-tuple are more
important than other packets.

### JSON syntax

Low is 1, Medium is 2, High is 3.  4 is undefined.

> or, do we want to use "Low", "Medium", "High" ???

~~~~~
{"metadata":{"Priority":2}}
~~~~~
{: #json-priority artwork-align="left" title="JSON for Priority"}

### UDP Options

UDP options can carry metadata in each packet as described below.

16 bits for magic cookie and versioning.

2 bits for high/medium/low priority,

~~~~~
   0b00=low
   0b01=medium
   0b10=high
   0b11=reserved
~~~~~

# Examples

## Per-Packet Prioritization

Per-packet prioritization is useful for a remote desktop application, as shown in {{example}}.

| traffic type    | nature    | loss-tolerant | priority | comments |
|:---------------:|:---------:|:-------------:|:--------:|:---------|
| glyph critical  | real-time |               |          |          |
| glyph smoothing | real-time |               |          |          |
| print job       | bulk      |               |          |          |
{: #example}

## Encoding as UDP Options

A high-priority realtime loss-tolerant packet would be encoded in a UDP Option as follows:

~~~~~
+---------------+
| 1 0 0 1 0 1 0 ... |
+---------------+
~~~~~

## Encoding as JSON

The same packet as above, encoded in JSON,

~~~~~
{"metadata":{"Priority":2,"Nature":"real-time","Loss-Tolerant":"yes"}}
~~~~~

# Security Considerations

To be completed.

# IANA Considerations

None.


# Acknowledgments
{:numbered="false"}

To be completed.


