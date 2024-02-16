---
title: "Flow Metadata for Collaborative Host/Network Signaling"
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
pi:
  tocdepth: 9
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  docmapping: yes
  tocdepth: 9
  toc: yes

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


informative:
  QUIC: RFC9000
  LOSSY-QUIC: RFC9221
  RTP: RFC3550
  RELIABLE-RTP: RFC4588


--- abstract

As part of host-to-network signaling, an entire flow can share the same set of
metadata or certain packets of a flow can have specific metadata associated with those
packets. Also, as part of network-to-host signaling, network metadata can be communicated
to a host, allowing the host to modulate its behavior (e.g., requested CODECs) to conform to
the available network resources.

This document describes the metadata exchanged in a companion
host-to-network/network-to-host signaling protocol.  These metadata
are intended to be applicable independent of the signaling protocol
used between a host and a network.

--- middle

# Introduction

Host-to-network metadata signaling has historically been performed by
the sender setting DSCP bits
(e.g., {{?RFC2475}}, {{?RFC7657}}, and {{?RFC8837}}). While DSCP can express
high priority (Expedited Forwarding (EF) {{?RFC3246}}) and low priority
(Lower-Effort Per-Hop Behavior (LE PHB) {{?RFC8622}}), DSCP bits are frequently ignored at
congestion points or lost (stripped) while forwarded across the Internet.
See {{Section 4 of ?RFC9435}} for a detailed overview of observed DSCP re-marking behaviors.
Also, DSCP attempts to influence the packet's treatment compared to
all other packets from other hosts.

Network-to-host metadata signaling has historically been performed by dropping packets or
setting the Explicit Congestion Notification (ECN) bit on packets {{?RFC3168}}.  Both of these techniques work well to consume
the network's available bandwidth when the sending rate exceeds the network's
available bandwidth, but are complicated for the receiver to determine the network's
bandwidth policy rate in a timely manner.

Both the above use cases are improved by metadata described in this document. This
document is a companion to host-to-network signaling the metadata itself, such as:

* UDP Options (e.g., {{?I-D.kaippallimalil-tsvwg-media-hdr-wireless}}, {{?I-D.reddy-tsvwg-explcit-signal}}),
* IPv6 Hop-by-Hop Options ({{Section 4.3 of ?RFC8200}}), or
* QUIC CID mapping ({{?I-D.wing-cidfi}}).

An analysis of most of those metadata signaling mechanisms is at {{?I-D.herbert-host2netsig}}.

The metadata defined in this document is independent of the actual
companion signaling protocol. In doing so, we ensure that consistent
metadata definitions are used by the various signaling protocols.  The
metadata is described using {{!CDDL=RFC8610}} which can be expressed
in both {{?JSON=RFC8259}} and binary using {{?CBOR=RFC8949}}.  It is
out of scope of this document to define how the proposed encoding will
be mapped to a specific signaling protocol.

Some applications use heuristics to determine rate-limiting policy. This document
proposes an explicit approach that is meant to share more granular information
so that these application adjusts their behavior in a timely manner (e.g., anticipate congestion).

For host-to-network metadata, individual packets within a flow can
contain metadata describing their drop preference or their
reliability. The network elements aware of this metadata can apply
preferential or deferential treatment to those packets during a
'reactive traffic policy' event. It is also assumed that such network
elements are provisioned with local policy that guides their behavior
jointly with a signaled metadata. Examples of metadata signaling
for video streaming and for remote desktop are provided in {{examples-h2n}}.

For network-to-host metadata, the host can be informed, e.g., of the network
bandwidth policy for the subscriber to receive streaming video. This
policy can be used by video streaming applications on that host to
choose video streams that fit within that policy.



# Conventions and Definitions

{::boilerplate bcp14-tagged}

Reactive policy:
: Treatment given to a flow when an exceptional event occurs, such as diminished throughput to the host caused by radio interference or weak radio signal, congestion on the network caused by other users or other applications on the same host.

Intentional policy:
: Configured bandwidth, pps, or similar throughput constraints applied to a flow, application, host, or subscriber.

# Host to Network Metadata

Three bits are introduced. A network element desiring the simplest interpretation can use the bits as a
3-bit 'importance' field, where higher values indicate high importance and lower values indicate less importance. A packet tagged as less
important means that such a packet can be discarded during a reactive policy event in favor, eventually, of a competing but more important packet.
Absent such discard preference indication, the network element will blindly drop packets during a reactive policy event.

For more comprehensive interpretation, metadata is characterized into two different nature:

* Network Metadata:
    This consists of metadata that specifies how the network element should treat that packet. The network metadata comprises of the importance field and is specified in the MSB and of size 1 bit. This field indicates if the packet is more important or less important.

* Application Metadata:
    This consists of metadata that specifies how the application treats that packet. The appplication metadata comprises of two fields - Keep/Discard bit and Reliable/Unreliable bit.

## Importance

The "Importance" metadata signifies if the packet is of more important (true) or
less important (false) by the host, relative to other packets in the
same flow.  Importance belongs to Network Metadata.

### Application Treatment

An application would mark a packet as important when it needs the
network to treat the packet with greater preference compared to the
unmarked packets or to packets marked important=false (of the same
flow). This tagging does not provide more privileges to an application
with regards to resources usage compared to the absence of signal. An
example of this interpretation is specified in {{examples-h2n}}.

### CDDL Encoding

~~~~~
importance = true / false
~~~~~

## Reliable/Unreliable

The "Reliable" metadata indicates if a packet that carries that bit is reliably transmitted by the host. Packets are marked as 'unreliable' or 'reliable':

* Reliable packets are re-transmitted by the underlying transport
(e.g., TCP {{?RFC9293}} or {{QUIC}}) or re-transmitted by the appplication (e.g., {{RELIABLE-RTP}}, NTP).
* Unreliable packets are not re-transmitted by the transport
(e.g., UDP, {{RTP}}, {{LOSSY-QUIC}}) and also not re-transmitted by the application (e.g., {{RTP}}).

Packets marked reliable, if delayed excessively or dropped outright, will be re-transmitted (up to a maximum retries) by
the sender application, appearing on the network again. Thus, delaying or discarding such packets does not
reduce the amount of transmitted data in a network; it only defers when it appears on the network.

Reliable/Unreliable belongs to Application Metadata.

### Network Treatment

During a reactive policy event, dropping unreliable traffic is preferred over dropping reliable
traffic. The reliable traffic will be re-transmitted by the sender so dropping such traffic
only defers it until later, but this deferral can be useful.

### CDDL Encoding

~~~~~
reliable = true / false
~~~~~

## Packet Discard Preference

This metadata indicates discard preference for unreliable traffic and reliable traffic, as detailed below.

### Unreliable Traffic

Packets are marked as 'may-discard' or 'prefer-keep'.

Many flows contain a mix of important packets and less-important packets, but applications
seldom signal that difference themselves let alone signal the difference to the network.
Rather, applications send everything over a reliable transport (TCP or QUIC) and leave it
at that, as evidenced by video streaming using TCP.

With the advent of {{LOSSY-QUIC}}, applications can send both {{QUIC}} reliable traffic and
{{LOSSY-QUIC}} unreliable traffic {{LOSSY-QUIC}} on the same 5-tuple.  With
host-to-network metadata signaling, the network can become an active assistant in such
flows during a reactive policy event by endeavouring to send the more-important 'prefer-keep'
traffic at the expense of the less-important 'may-discard' traffic.

The reasoning why a packet, marked as 'may-discard', is transmitted by an application while
the application can avoid sending that packet is application-specific.

#### Network Treatment

During a reactive policy event, dropping 'may-discard' packets is preferred over dropping
'prefer-keep' packets.

#### CDDL Encoding

~~~~~
prefer-keep = true / false
~~~~~

### Reliable Traffic

For reliable traffic, this metadata indicates whether the packet belongs to bulk or real-time traffic.

#### Network Treatment

Realtime traffic prefers lower latency network paths and bulk traffic prefers high throughoupt paths.

#### CDDL Encoding

~~~~~
realtime = true / false
~~~~~

# Network to Host Metadata

## Downlink Bitrate

Monthly data quotas on cellular networks can be easily exceeded by video streaming, in particular, if the
client chooses excessively high quality or routinely abandons watching videos that were
downloaded. The network can assist the client by informing the client of the network's
bandwidth policy.

If the video is encoded with variable bit rate, the bitrate cannot exceed the indicated
bitrate.

The nominal bitrate is calculated over each second, whereas the burst
bitrate is calculated over the signaled interval (burst-duration).
For either measurement, packets can arrive at the start of a second,
as near as possible behind each other, and the remaining portion of
that second could have no packets transmitted.

### Units

Bit rate is expressed in Mbps and duration is in milliseconds.

### Host Treatment

The host chooses a video streaming bit rate at or below the signaled rate.

The host may also choose to signal the received bitrate to the remote peer. The remote
peer will adapt its transmission behavior to comply with the received bitrate.

### CDDL Encoding

~~~~~
downlinkBitrate = {
  nominal: uint,        ; Mbps
  ? burst-info
}

burst-info = {
  burst: uint,          ; Mbps
  burstDuration: uint   ; milliseconds
}
~~~~~

An example of the encoding is in {{examples-n2h}}.

## Prefer Alternate Path ('pref-alt-path')

There are also crisis cases where nominal network resources cannot be
used at maximum to handle packets. A network would thus seek to offload some of the
traffic during these events. Under such exceptional events, a network
element may signal to a host that it is preferrable to use alternate
paths, if available. An alternate path is typically an alternate network
attachment.  After the crisis has subsided, the network should signal
with pref-alt-path=false.

The 'pref-alt-path' metadata may be sent together with the bitrate metadata set to a very low value.

### Host Treatment

The host offloads its connections to alternate available paths.

### CDDL Encoding

~~~~~
pref-alt-path = true / false
~~~~~

# Security Considerations

Metadata increases the information available to attackers to
distinguish important packets from less-important packets, which the
attacker might use to attack such packets (e.g., prevent their
delivery) or attempt to decrypt those packets. It is RECOMMENDED to
encrypt or obfuscate the metadata information so it is only available
to hosts and to authorized network elements.  The method of
encryption or obfuscation is not described in this document but
rather in other documents describing how this metadata is encoded
and exchanged amongst hosts and network elements.

# IANA Considerations

## Metadata for Collaborative Host/Network Signaling Registry Group

This document requests IANA to create a new registry group, entitled "Metadata for Collaborative Host/Network Signaling".

## Flow Metadata Registry

IANA is requested to create a new registry, entitled "Flow Metadata Registry", under the "Metadata for Collaborative Host/Network Signaling" registry group.
This registry is inspired by the "Performance Metrics Registry" created by {{?RFC8911}}. The structure of the registry is as follows:

Identifier:
: A numeric identifier for the registered metadata.
: The Identifier 0 is Reserved.
: The Identifier values from 250 to 255 are reserved for private or experimental use.

Name:
: Name of the registered metadata.

Description:
: Provides a description of the intended use of the registered metadata.

Reference:
: Lists the authoritative reference that specifies the registered metadata.

Version:
: Tracks the current version of the metadata.
: The initial version of a new registered metadata MUST be 1.0.
: IANA will bump the version when a new RFC that changes the format/semantic of a registered entry.

The initial values of the registry are listed in {{initial-reg}}.

| Identifier | Name              | Description      | Reference     | Version |
|:----------:|:-----------------:|:-----------------|:-------------:|:-------:|
| 0          |                   | Reserved         | This-Document |         |
| 1          | Importance        | Indicates the level of importance of a packet in a flow            | This-Document | 1.0     |
| 2          | PacketNature      | Indicates whether a packet is reliably or unreliably transmitted   | This-Document | 1.0     |
| 3          | DiscardPreference | Indicates a discard preference         | This-Document | 1.0     |
| 0          | DownlinkBitrate   | Specifies the maximum downlink bitrate         | This-Document | 1.0     |
{: #initial-reg title="Initial Values"}

New entries can be added to the registery using "Standards Action" policy ({{Section 4.9 of !RFC8126}}.

# Acknowledgments
{:numbered="false"}

To be completed.

--- back

# Examples of Host-to-Network Metadata {#examples-h2n}

## Video Streaming {#example-video-streaming}

Streaming video contains the occasional key frame ("i-frame")
containing a full video frame.  These are necessary to rebuild
receiver state after loss of delta frames.  The key frames are
therefore more critical to deliver to the receiver than delta frames.

Streaming video also contains audio frames which can be encoded
separately and thus can be signaled separately.  Audio is more
critical than video for almost all applications, but its importance
(relative to other packets in the flow) is still an application decision.  In the example below, the audio
is more important than video (importance=high, KD=keep, RU=reliable), video key frames
have middle importance (importance=low, discard, reliable), and both types
of video delta frames (P-frame and B-frame) have least importance (importance=low, KD=discard, RU=unreliable).

Comprehensive Interpretation:

| Traffic type                             | Importance | PacketNature      | RU                   |
|:----------------------------------------:|:----------:|:-----------------:|:--------------------:|
| video I-frame (key frame)                | low        | realtime          | reliable             |
| video delta P-frame                      | low        | discard           | unreliable           |
| video delta B-frame                      | low        | discard           | unreliable           |
| audio                                    | high       | realtime          | reliable             |
{: #table-video-streaming-ci title="Example Values for Video Streaming Metadata - Comprehensive Interpretation"}

Simple Interpretation:

| Traffic type      | Simple Interpretation |
|:-----------------:|:---------------------:|
| video I-frame     | 011                   |
| video delta frame | 001                   |
| audio             | 111                   |
{: #table-video-streaming-si title="Example Values for Video Streaming Metadata - Simple Interpretation"}


## Interactive Gaming or Audio/Video  {#example-interactive-av}

Both gaming (video in both directions, audio in both directions, input
devices from client to server) and interactive audio/video (VoIP,
video conference) involves important traffic in both directions --
thus is a slightly more complicated use-case than the previous
example.  Additionally, most Internet service providers constrain
upstream bandwidth so proper packet treatment is critical in the
upstream direction.

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

In many scenarios a game or VoIP application will want to signal different
metadata for the same type of packet in each direction.  For example, for
a game, video in the server-to-client direction might be more important
than audio, whereas input devices (e.g., keystrokes) might be more important
than audio.


## Remote Desktop Virtualization {#example-rdt}

Example packet metadata for Desktop Virtualization (like Citrix
Virtual Apps and Desktops - CVAD) application.  This is shown in two
tables, client-to-server traffic ({{table-desktop-virtualization-c2s-ci}})({{table-desktop-virtualization-c2s-si}})
and server-to-client traffic ({{table-desktop-virtualization-s2c-ci}})({{table-desktop-virtualization-s2c-si}}).

Comprehensive Interpretation:

| Traffic type               | Importance | PacketNature    | Reliable/Unreliable | Comments  |
|:--------------------------:|:----------:|:---------------:|:-------------------:|:---------:|
| User typing                | high       | realtime        | reliable            |           |
| Mouse click/End Position   | high       | realtime        | reliable            | The start and endpoint of the pointer movement is vital to ensure user action is completed correctly. So, the endpoints have to be reliably transmitted with real-time priority. **|
| Interactive audio          | high       | keep            | unreliable          |   |
| Authentication - Finger print, smart card | low | realtime | reliable |  |
| Interactive video key frame            | low        | keep            | unreliable          | Video key frames form the base frames of a video upon which the next 'n' timeframe of video updates is applied on. These frames, are hence, critical and without them, the video would not be coherent until the next critical frame is received. Retransmits of these are harmful to the UX. ***|
| Mouse position tracking    | low        | discard         | unreliable          | When the pointer is moved from one point to another, the coordinates of the pointers between the two points can be lost without much of an impact to the UX as long as the start and endpoint reaches. This would ensure the user action is completed, even if the experience seems glitchy. |
| Interactive video delta frame           | low        | discard            | unreliable          |   |
{: #table-desktop-virtualization-c2s-ci title="Example Values for Remote Desktop Virtualization Metadata, client to server - Comprehensive Interpretation"}

| Traffic type               | Importance | PacketNature    | Reliable/Unreliable | Comments  |
|:--------------------------:|:----------:|:---------------:|:-------------------:|:---------:|
| Glyph critical             | high       | realtime        | reliable          | The frames that form the base for the image is more critical and needs to be transmitted as reliably as possible. Retransmits of these are harmful to the UX.**|
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
| Authentication - Finger print, smart card | 011    |
| Interactive video key frame            | 010       |
| Mouse position tracking    | 000                   |
| Interactive video delta frame           | 000      |
{: #table-desktop-virtualization-c2s-si title="Example Values for Remote Desktop Virtualization Metadata, client to server - Simple Interpretation"}

| Traffic type               | Simple Interpretation |
|:--------------------------:|:---------------------:|
| Glyph critical             | 111                   |
| Interactive (or streaming) audio   | 110           |
| Haptic feedback            | 100                   |
| Interactive (or streaming) video key frame  | 010  |
| File copy                  | 001                   |
| Interactive (or streaming) video predictive frame | 000 |
| Glyph smoothing            | 000                   |
{: #table-desktop-virtualization-s2c-si title="Example Values for Remote Desktop Virtualization Metadata, server to client - Simple Interpretation"}

*** A video key frame should be handled differently by the network
depending on a streaming application versus a remote desktop
application.  The video streaming application's primary and only
nature of traffic is video and audio.  In contrast, a remote desktop
application might be playing a video and its associated audio while at
the same time the user is editing a document.  The user's keystrokes
and those glyphs need to be prioritized over the video lest the user
think their inputs are being ignored (and type the same characters
again). Hence, the values are different even for the same nature of
traffic but a different application.


# Example of Network-to-Host Metadata for Video Streaming {#examples-n2h}

A network element can signal the maximum bandwidth allowed for video streaming. Typically,
this policy limit exists in cellular networks.

The example below indicates the burst bandwidth (2Mbps), burst duration
(3 seconds), and nominal (non-burst) bandwidth (1Mbps) for the requesting
user:

~~~~~
{
  "videoStreamingBandwidth": {
    "burst": 2048,
    "burstDuration": 3000,
    "nominal": 1024
  }
}
~~~~~


