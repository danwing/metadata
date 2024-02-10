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
  group: "TSV"
  type: "Working Group"
  mail: "tsvwg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tsvwg/"
  github: "danwing/metadata"
  latest: "https://danwing.github.io/metadata/draft-rw-flow-metadata.md.html"

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
  QUIC: RFC9000
  LOSSY-QUIC: RFC9221
  RTP: RFC3550
  RELIABLE-RTP: RFC4588




--- abstract

As part of host-to-network signaling, an entire flow can share the same
metadata or certain packets can have certain metadata associated with those
packets.  This document describes the metadata exchanged in a
host-to-network signaling protocol in both binary and JSON.

As part of network-to-host signaling, network metadata can be communicated
to the host, allowing the host to modulate its requests to conform to
the available network resources.

--- middle

# Introduction

Host-to-network metadata signaling has historically been performed by
the sender setting DSCP bits
({{?RFC7657}})({{?RFC8837}})({{?RFC2475}}). While DSCP can express
high priority (Expedited Forwarding {{?RFC3246}}) and low priority
(Lower Effort PDB {{?RFC8622}}), DSCP bits are frequently ignored at
congestion points or lost (stripped) while routed across the Internet.
Also, DSCP attempts to influence the packet's treatment compared to
all other packets from other hosts.

Network-to-host metadata signaling has historically been performed by dropping packets or
setting the ECN bit on packets.  Both of these techniques work well to consume
the network's available bandwidth when the sending rate exceeds the network's
available bandwidth, but are complicated for the receiver to determine the network's
bandwidth policy rate.

Both the above use cases are improved by metadata described in this document. This
document is a companion to host-to-network signaling the metadata itself, such as

* UDP Options (e.g., {{?I-D.kaippallimalil-tsvwg-media-hdr-wireless}}, {{?I-D.reddy-tsvwg-explcit-signal}}),
* IPv6 Hop-by-Hop Options ({{Section 4.3 of ?RFC8200}}), or
* QUIC CID mapping ({{?I-D.wing-cidfi}}).

An analysis of most of those metadata signaling mechanisms is at {{?I-D.herbert-host2netsig}}.

For host-to-network metadata, individual packets within a flow can
contain metadata describing their drop preference and their
reliability. The network elements aware of this metadata can apply
preferential or deferential treatment to those packets during a
'reactive traffic policy' event.  Examples of metadata signaling
for video streaming and for remote desktop are in {{examples}}.

For network-to-host metadata, the host can be informed of the network
bandwidth policy for the subscriber to receive streaming video. This
policy can be used by video streaming applications on that client to
choose video streams that fit within that policy.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Reactive policy:
: Treatment given to a flow when an exceptional event occurs, such as diminished throughput to the host caused by radio interference or weak radio signal, congestion on the network caused by other users or other applications on the same host

Intentional policy:
: Configured bandwidth, pps, or similar throughput constraints applied to a flow, application, host, or subscriber.

# Host to Network Metadata

Three bits are introduced. A network element desiring the simplest interpretation can use the bits as a
3-bit 'importance' field, where higher values are more important and lower values are less important. Less
importance means those packets can be discarded during a reactive policy event.

For more comprehensive interpretation, metadata is characterized into 2 different nature:

* Network Metadata:
    This consists of metadata that specifies how the network element should treat that packet. The network metadata comprises of the importance field and is specified in the MSB and of size 1 bit. This field indicates if the packet is more-important or less-important.

* Application Metadata:
    This consists of metadata that specifies how the application treats that packet. The appplication metadata comprises of 2 fields - Keep/Discard bit and Reliable/Unreliable bit.

## Importance

Importance bit signifies if the packet is of more importance or less importance by the network element. This is specified in the MSB of metadata bits and is of 1 bit length. Importance belongs to Network Metadata.

### Application Treatment

Application would mark a packet important when it needs the network to treat the packet with greater preference compared to the unmarked packets. An example of this interpretation is specified in the {{examples}} section.

### Encoding

More-Important:

* When signaled in binary, the Importance bit is set (1).
* When signaled in JSON, it is encoded as name "importance" and value "true".

Less-Important:

* When signaled in binary, the Importance bit is cleared (0).
* When signaled in JSON, it is encoded as name "importance" and value "false".

## Reliable/Unreliable

Packets are marked as 'unreliable' or 'reliable':

* Reliable packets are re-transmitted by the transport
(e.g., TCP or {{QUIC}}) or re-transmitted by the appplication (e.g., {{RELIABLE-RTP}}, NTP).
* Unreliable packets are not re-transmitted by the transport
(e.g., UDP, {{RTP}}, {{LOSSY-QUIC}}) and also not re-transmitted by the application (e.g., {{RTP}}).

Packets marked reliable, if delayed excessively or dropped outright, will be re-transmitted by
the sender application, appearing on the network again.  Thus, delaying or discarding such packets does not
reduce the amount of transmitted data, it only defers when it appears on the network.

This is specified in the MSB of metadata bits and is of 1 bit length. Reliable/Unreliable belongs to Application Metadata.

### Network Treatment

During a reactive policy event, dropping unreliable traffic is preferred over dropping reliable
traffic. The reliable traffic will be re-transmitted by the sender so dropping such traffic
only defers it until later, but this deferral can be useful.

### Encoding

Reliable:

* When signaled in binary, the Reliable bit is set (1).
* When signaled in JSON, it is encoded as name "reliable" and value "true".

Unreliable:

* When signaled in binary, the Reliable bit is cleared (0).
* When signaled in JSON, it is encoded as name "reliable" and value "false".


## Packet Nature/Preference

### Packet Nature for Unreliable Traffic

Packet Nature indicates discard preference for unreliable traffic.

Packets are marked as 'discard' or 'keep'.

Many flows contain a mix of important packets and less-important packets, but applications
seldom signal that difference themselves let alone signal the difference to the network.
Rather, applications send everything over a reliable transport (TCP or QUIC) and leave it
at that, as evidenced by video streaming using TCP.

With the advent of {{LOSSY-QUIC}}, applications can send both {{QUIC}} reliable traffic and
{{LOSSY-QUIC}} unreliable traffic {{LOSSY-QUIC}} on the same 5-tuple.  With
host-to-network metadata signaling, the network can become an active assistant in such
flows during a reactive policy event by endeavouring to send the more-important 'keep'
traffic at the expense of the less-important 'discard' traffic.

#### Network Treatment

During a reactive policy event, dropping 'discard' packets is preferred over dropping
'keep' packets.

#### Encoding

Discard:

* When signaled in binary, the PacketNature bit is set (1).
* When signaled in JSON, it is encoded as name "Discard" and value "true".

Keep:

* When signaled in binary, the PacketNature bit is cleared (0).
* When signaled in JSON, it is encoded as name "Discard" and value "false".

### Packet Nature for Reliable Traffic

Discard preference for reliable traffic doesn't make sense since no reliable traffic should be discarded. So the packet nature bits for reliable traffic indicates whether the packet belongs to bulk or real-time traffic.

#### Network Treatment

Realtime traffic prefers less latency network paths and bulk traffic prefers high throughoupt paths.

#### Encoding

Realtime:

* When signaled in binary, the PacketNature bit is set (1).
* When signaled in JSON, it is encoded as name "realtime" and value "true".

Bulk traffic:

* When signaled in binary, the PacketNature bit is cleared (0).
* When signaled in JSON, it is encoded as name "realtime" and value "false".

To adhere to both simple and comprehensive interpretation of the metadata, it is vital to order the bits in the same sequence as mentioned below.

MSB - Importance bit.
Bit 1 - Packet Nature bit.
LSB - Reliable/Unreliable bit.

More details on how simple and comprehensive interpretation of metadata would work for different types of traffic is listed in the {{examples}} section.

> Discussion: Come up with better name for PacketNature.

# Network to Host Metadata

## Video Streaming Bandwidth

Monthly data quotas on cellular networks can be easily exceeded by video streaming if the
client chooses excessively high quality or routinely abandons watching videos that were
downloaded.  The network can assist the client by informing the client of the network's
bandwidth policy.

If the video is encoded with variable bit rate, the bitrate cannot exceed the indicated
bitrate.

The bitrate is calculated over each second. This means the packets can arrive at the
start of a second, as near as possible behind each other, and the remaining portion
of that second could have no packets transmitted.

> Discussion: should we also signal bursts for variable bit rates?


### Host Treatment

The host chooses a video streaming bit rate at or below the signaled rate.

### Encoding

When signaled in JSON, the bandwidth is encoded as namae "video-streaming-bandwidth" and
the value is in kilobits per second.


# Security Considerations

Metadata increases the information available to attackers to
distinguish important packets from less-important packets, which the
attacker might use to attack such packets (e.g., prevent their
delivery) or attempt to decrypt those packets.  It is useful to
encrypt or obfuscate the metadata information so it is only available
to end hosts and to participating network elements.  The method of
encryption or obfuscation is not described in this document but
rather in other documents describing how this metadata is encoded
and exchanged amongst hosts and network elements.

# IANA Considerations

TBD:  Will need a new registry.




# Acknowledgments
{:numbered="false"}

To be completed.

--- back

# Examples of Host-to-Network Metadata {#examples}

## Video Streaming {#example-video-streaming}

Streaming video contains the occasional key frame ("i-frame")
containing a full video frame.  These are necessary to rebuild
receiver state after loss of delta frames.  The key frames are
therefore more critical to deliver to the receiver than delta frames.

Streaming video also contains audio frames which can be encoded
separately and thus can be signaled separately.  Audio is more
critical than video for almost all applications, but its importance
is still an application decision.  In the example below, the audio
is more important than video (importance=high, KD=keep, RU=reliable), video key frames
have middle importance (importance=low, discard, reliable), and video delta
frames have least importance (importance=low, KD=discard, RU=unreliable).

Comprehensive Interpretation:

| Traffic type      | Importance | PacketNature      | RU                   |
|:-----------------:|:----------:|:-----------------:|:--------------------:|
| video key frame   | low        | realtime          | reliable             |
| video delta frame | low        | discard           | unreliable           |
| audio             | high       | realtime          | reliable             |
{: #table-video-streaming-ci title="Example Values for Video Streaming Metadata - Comprehensive Interpretation"}

Simple Interpretation:

| Traffic type      | Simple Interpretation |
|:-----------------:|:---------------------:|
| video key frame   | 011                   |
| video delta frame | 001                   |
| audio             | 111                   |
{: #table-video-streaming-si title="Example Values for Video Streaming Metadata - Simple Interpretation"}

Astute readers will notice this use case could be handled with a
ternary value represented by 2 bits (rather than 3 bits).  However,
other use cases such as {{example-rdt}} need 3 bits.

> Discussion: Do we need to mention the above statement about 2/3 bits?

> Discussion: The importance is a single bit - so change the medium to a high in the above table and made video key frame as reliable keep and low. That way, the importance bit would be the only differentiator.

> Discussion: Breaking KD bits to indicate Keep/Discard for Unreliable traffic and realtime/bulk for reliable traffic

## Interactive Audio/Video Streaming {#example-interactive-av}

Interactive audio/video, such as a video call, involves important traffic
in both directions -- thus is a slightly more complicated use-case than
the previous example.  Additionally, most Internet service providers
constrain upstream bandwidth so proper packet treatment is critical in
the upstream direction.

Comprehensive Interpretation:

| Traffic type      | Importance | PacketNature      | RU                   |
|:-----------------:|:----------:|:-----------------:|:--------------------:|
| video key frame   | low        | realtime          | reliable             |
| video delta frame | low        | discard           | unreliable           |
| audio             | high       | realtime          | reliable             |
{: #table-interactive-av-downstream-ci title="Example Values for Interactive A/V, downstream - Comprehensive Interpretation"}

| Traffic type      | Importance | PacketNature      | RU                   |
|:-----------------:|:----------:|:-----------------:|:--------------------:|
| video key frame   | low        | realtime          | reliable             |
| video delta frame | low        | discard           | unreliable           |
| audio             | high       | realtime          | reliable             |
{: #table-video-av-upstream-ci title="Example Values for Interactive A/V, upstream - Comprehensive Interpretation"}

Simple Interpretation:

| Traffic type      | Simple Interpretation |
|:-----------------:|:---------------------:|
| video key frame   | 011                   |
| video delta frame | 001                   |
| audio             | 111                   |
{: #table-interactive-av-downstream-si title="Example Values for Interactive A/V, downstream - Simple Interpretation"}

| Traffic type      | Simple Interpretation |
|:-----------------:|:---------------------:|
| video key frame   | 011                   |
| video delta frame | 000                   |
| audio             | 111                   |
{: #table-video-av-upstream-si title="Example Values for Interactive A/V, upstream - Simple Interpretation"}

Many interactive audio/video applications also support sharing the presenter's
screen, file, video, or pictures.  During this sharing the presenter's video
is less important but the screen or picture is more important.  This change
of imporance can be conveyed in metadata to the network, as in the table
below:

Comprehensive Interpretation:

| Traffic type      | Importance | PacketNature      | RU                   |
|:-----------------:|:----------:|:-----------------:|:--------------------:|
| video key frame   | low        | realtime          | reliable             |
| video delta frame | low        | discard           | unreliable           |
| audio             | high       | realtime          | reliable             |
| picture sharing   | high       | realtime          | reliable             |
{: #table-video-av-sharing-ci title="Example Values for Interactive A/V, upstream - Comprehensive Interpretation"}

Simple Interpretation:

| Traffic type      | Simple Interpretation |
|:-----------------:|:---------------------:|
| video key frame   | 011                   |
| video delta frame | 000                   |
| audio             | 111                   |
| picture sharing   | 111                   |
{: #table-video-av-sharing-si title="Example Values for Interactive A/V, upstream - Simple Interpretation"}


## Remote Desktop Virtualization {#example-rdt}

Example packet metadata for Desktop Virtualization (like Citrix
Virtual Apps and Desktops - CVAD) application.  This is shown in two
tables, client-to-server traffic ({{table-desktop-virtualization-c2s-ci}})({{table-desktop-virtualization-c2s-si}})
and server-to-client traffic ({{table-desktop-virtualization-s2c-ci}})({{table-desktop-virtualization-s2c-si}}).

Comprehensive Interpretation:

| Traffic type               | Importance | PacketNature    | Reliable/Unreliable | Comments  |
|:--------------------------:|:----------:|:---------------:|:-------------------:|:---------:|
| User typing                | high       | realtime        | reliable            | Client To Server Traffic         |
| Mouse click/End Position   | high       | realtime        | reliable            | The start and endpoint of the pointer movement is vital to ensure user action is completed correctly. So, the endpoints have to be reliably transmitted with real-time priority. **|
| Interactive audio          | high       | keep            | unreliable          |   |
| Authentication - Finger print, smart card | low | realtime | reliable | Comments |
| Interactive video key frame            | low        | keep            | unreliable          | Video key frames form the base frames of a video upon which the next 'n' timeframe of video updates is applied on. These frames, are hence, critical and without them, the video would not be coherent until the next critical frame is received. Retransmits of these are harmful to the UX. ***|
| Mouse position tracking    | low        | discard         | unreliable          | When the pointer is moved from one point to another, the coordinates of the pointers between the two points can be lost without much of an impact to the UX as long as the start and endpoint reaches. This would ensure the user action is completed, even if the experience seems glitchy. |
| Interactive video delta frame           | low        | discard            | unreliable          |   |
{: #table-desktop-virtualization-c2s-ci title="Example Values for Remote Desktop Virtualization Metadata, client to server - Comprehensive Interpretation"}

| Traffic type               | Importance | PacketNature    | Reliable/Unreliable | Comments  |
|:--------------------------:|:----------:|:---------------:|:-------------------:|:---------:|
| Glyph critical             | high       | keep            | reliable          | The frames that form the base for the image is more critical and needs to be transmitted as reliably as possible. Retransmits of these are harmful to the UX.**|
| Interactive (or streaming) audio   | high       | keep            | unreliable          |   |
| Haptic feedback            | high       | discard         | unreliable          | Virtualizing haptic feedback is real-time and high importance although the feedback being delivered late is of no use. So dropping the packet altogether and not retransmitting it makes more sense |
| Interactive (or streaming) video key frame            | low        | keep            | unreliable          | Video key frames form the base frames of a video upon which the next 'n' timeframe of video updates is applied on. These frames, are hence, critical and without them, the video would not be coherent until the next critical frame is received. Retransmits of these are harmful to the UX. ***|
| File copy                  | low        | bulk            | reliable            |   |
| Interactive (or streaming) video predictive frame     | low        | discard         | unreliable          | Video predictive frames can be lost, which would result in minor glitch but not compromise the user activity and video would still be coherent and useful. The reception of subsequent video key frame would mitigate the loss in quality caused by lost predictive frames. |
| Glyph smoothing            | low        | discard         | Unreliable          | The smoothing elements of the glyph can be lost and would still present a recognizable image, although with a lesser quality. Hence, these can be marked as loss tolerant as the user action is still completed with a small compromise to the UX. Moreover, with the reception of the next glyph critical frame would mitigate the loss in quality caused by lost glyph smoothing elements. |
{: #table-desktop-virtualization-s2c-ci title="Example Values for Remote Desktop Virtualization Metadata, server to client - Comprehensive Interpretation"}

Simple Interpretation:

| Traffic type               | Simple Interpretation |
|:--------------------------:|:---------------------:|
| User typing                | 111                   |
| Mouse click/End Position   | 111                   |
| Interactive audio          | 110                   |
| Authentication - Finger print, smart card | 011                   |
| Interactive video key frame            | 010                   |
| Mouse position tracking    | 000                   |
| Interactive video delta frame           | 000                   |
{: #table-desktop-virtualization-c2s-si title="Example Values for Remote Desktop Virtualization Metadata, client to server - Simple Interpretation"}

| Traffic type               | Simple Interpretation |
|:--------------------------:|:---------------------:|
| Glyph critical             | 111                   |
| Interactive (or streaming) audio   | 110                   |
| Haptic feedback            | 100                   |
| Interactive (or streaming) video key frame            | 010                   |
| File copy                  | 001                   |
| Interactive (or streaming) video predictive frame     | 000                   |
| Glyph smoothing            | 000                   |
{: #table-desktop-virtualization-s2c-si title="Example Values for Remote Desktop Virtualization Metadata, server to client - Simple Interpretation"}

*** There is a key difference between a video key frame in a streaming application compared to video played within a remote desktop session. The video streaming application's primary and only nature of traffic is multimedia while it is not the case for a remote desktop application. There are certain traffic that would require more importance over multimedia (like graphics updates on a word document while user is typing in one window and a video is playing in another). Hence, the values are different even for the same nature of traffic but a different application. This is one more reason to justify 3 bits since the priorities and variety of the traffic will vary based on the application.

> Discussion: Haptic inputs are going to be part of future of Virtualization and including them here makes sense?

<!--
** These are critical but considering implementation constraints, data from a specific source (a virtual channel like mouse, graphics etc in this case) is either transmitted reliably or as loss-tolerant. These packets lost will have user experience impact but still since most of the traffic from this use-case come under loss-tolerant and it is not critical that the application breaks if these are not received (unlike file transfer), these are listed as loss-tolerant while having the don't bit set to 0.
-->
