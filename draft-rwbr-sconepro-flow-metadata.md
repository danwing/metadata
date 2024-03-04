---
title: "Flow Metadata for Collaborative Host/Network Signaling"
abbrev: "Flow Metadata"
category: std


docname: draft-rwbr-sconepro-flow-metadata-latest
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
  latest: "https://danwing.github.io/metadata/draft-rwbr-flow-metadata.md.html"

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
 -
    fullname: Mohamed Boucadair
    organization: Orange
    country: France
    email: mohamed.boucadair@orange.com
 -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    country: India
    email: kondtir@gmail.com


informative:
  QUIC: RFC9000
  LOSSY-QUIC: RFC9221
  RTP: RFC3550
  RELIABLE-RTP: RFC4588
  SCONEPRO:
    title: SCONEPRO Working Group Charter
    target: https://datatracker.ietf.org/group/sconepro/about/
    date: 2024-02-02


--- abstract

This document defines per-flow and per-packet metadata for both
network-to-host and host-to-network signaling in Concise Data Definition Language (CDDL) which
expresses both CBOR and JSON.  The common metadata definition allows interworking between
signaling protocols with high fidelity. The metadata is also self-
describing to improve interpretation by network elements and
endpoints while reducing the need for version negotiation.


--- middle

# Introduction

Historically, metadata is defined within each protocol. While this can
be very efficient on the wire (e.g., DSCP consumes only 6 bits) it
suffers from inability to authorize or authenticate the metadata
signaling. But the more signifcant problem is inability to interwork
between signaling protocols because each have different definitions.
Such interworking is often needed when the metadata signaling protocol
for packets leaving a network differs from the metadata signaling
protocol entering a different network. For example, important packets
leaving a server and its network might be marked with DSCP (as the
sending host is known and trusted) but the receiving network doesn't
trust the DSCP bits in received packets because there is no
authorization or authentication for differented treatment.

By using the same metadata, both networks can communicate how packets
should be treated and use their own signaling mechanism with their
network elements (e.g., routers, {{?MASQUE=I-D.ietf-masque-quic-proxy}} proxies).

Both the above use cases are improved by metadata described in this document. This
document is a companion to host-to-network signaling the metadata itself, such as:

* UDP Options (e.g., {{?I-D.kaippallimalil-tsvwg-media-hdr-wireless}}, {{?I-D.reddy-tsvwg-explcit-signal}}),
* IPv6 Hop-by-Hop Options ({{Section 4.3 of ?RFC8200}}),
* SCONE Protocol ({{SCONEPRO}}), or
* QUIC CID mapping ({{?I-D.wing-cidfi}}).

{{?I-D.herbert-host2netsig}} provides an analysis of most of those metadata signaling mechanisms.

This document does not assume nor preclude any companion signaling protocol.
Also, the document does not preclude API-based approaches to
control flows, packets, applications, etc. that are bound a given metadata and which
will benefit from the differentiated behavior. As such, **the metadata in this document is defined to be independent of the
signaling protocol** ({{sec-meta}}). In doing so, we ensure that consistent
metadata definitions are used by the various signaling protocols. Also,
this approach allows to factorize key considerations such as security and operational
considerations. This approach also ease passing policies between controllers of domains involved in packet delivery (e.g., RAN, Core, and Transport domains).

The metadata is described using Concise Data Definition Language (CDDL) {{!CDDL=RFC8610}} which can be expressed
in both {{?JSON=RFC8259}} and binary using {{?CBOR=RFC8949}}.  Both
the JSON and CBOR encodings are self-describing.  It is out of scope
of this document to define how the proposed encoding will be mapped to
a specific signaling protocol.

<!--
Some applications use heuristics to determine rate-limiting policy. This document
proposes an explicit approach that is meant to share more granular information
so that these application adjusts their behavior in a timely manner (e.g., anticipate congestion).

The application metadata defined in this document primary target signals
that are meant to soften implications of reactive policies. Also, these
metadata provide hints to guide the enforcement of those policies on **packets within a flow, not between
distinct flows or applications**.
-->

If the companion signaling protocol supports host-to-network metadata,
individual packets within a flow can contain metadata describing their
drop preference or their reliability. The network elements aware of
this metadata can apply preferential or deferential treatment to those
packets during a 'reactive traffic policy' event. It is also assumed
that such network elements are provisioned with local policy that
guides their behavior jointly with a signaled metadata. Examples of
metadata signaling for video streaming and for remote desktop are
provided in {{examples-h2n}}.

For network-to-host metadata, a host can be informed of network
policy for nominal downlink bandwidth. Certain applications,
such as most especially video streaming applications, can use
that information to optimize their video streaming bandwidth to
fit within that policy.

To track metadata that are defined for host/network signalling,
a new IANA registry is defined: "Flow Metadata Registry" {{sec-fmr}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms:

Reactive policy:
: Treatment given to a flow when an exceptional event occurs, such as
diminished throughput to the host caused by radio interference or weak
radio signal, congestion on the network caused by other users or other
applications on the same host.

Intentional policy:
: Configured bandwidth, pps, or similar throughput constraints applied
to a flow, application, host, or subscriber.

# Metadata Structure {#sec-meta}

The metadata is described in CDDL {{!RFC8610}} format shown in {{meta-cddl}}.

~~~~~ cddl
; one or more metadata can be signaled.
metadata = {
  metadata-type: (0..1), ; 0 is Network Metadata
                         ; 1 is Application Metadata
  * $$metadata-extensions
}

; Application Metadata

$$metadata-extensions //= (
; true indicates packet of high importance
; false indicates packet of low importance
  importance: bool,
; Packets can be tagged as reliable (true) or unreliable (false)
  reliable: bool,
; Packets can be tagged as preference to keep (true) or discard (false)
  prefer-keep: bool
; Has a meaning only for packets marked as reliable
; True indicates realtime
; False indicates bulk (non-realtime)
  realtime: bool
)

; Network Metadata

; Provides information about the nominal downlink bitrate
; Returning a value set to 0 (or a very low value) should trigger
; the host to seek for better paths.

Bitrate =  {
  nominal: uint,  ; Mbps
  ? burst-d => burst-info
}

burst-info = {
  burst: uint,          ; Mbps
  burstDuration: uint   ; milliseconds
}

$$metadata-extensions //= (
   ? downlinkBitrate => Bitrate,
; Indicates whether a flow is to be offloaded to alternate
; available paths.
   pref-alt-path: bool
)

downlinkBitrate = "downlinkBitrate"
burst-d = "burst-info"
~~~~~
{: #meta-cddl title="CDDL Structure of the Metadata"}

The structure shown in {{meta-cddl}} does not assume that the metadata
will be encoded as a single blob when mapped to a signaling protocol or
that all the metadata components will be mapped. Such matters
are specific to the individual signaling protocols and deployment contexts.

New metadata for collaborative host/network signaling MUST be registered
in the IANA registry, "Flow Metadata Registry" {{sec-fmr}}.

More details about each of these metadata are provided in {{sec-h2n}} and {{sec-n2h}}.
Both client and network intended behaviors are specified for each
metadata.

# Host-to-Network Metadata {#sec-h2n}

Metadata is characterized into two different nature:

Network Metadata:
: This consists of metadata that specifies how a network element should treat that packet. The network metadata comprises of the importance field and is specified in the MSB and of size 1 bit. This field indicates if the packet is more important or less important.

Application Metadata:
: This consists of metadata that specifies how the application treats that packet. The appplication metadata comprises of two components - Keep/Discard bit and Reliable/Unreliable bit.

## Packet Importance ('Importance')

The "Importance" metadata signifies if the packet is of more important (true) or
less important (false) by the host, relative to other packets in the
same flow.  Importance belongs to Network Metadata.

An application would mark a packet as important when it needs the
network to treat the packet with greater preference compared to the
unmarked packets or to packets marked important=false (of the same
flow). This tagging does not provide more privileges to an application
with regards to resources usage compared to the absence of signal. An
example of this interpretation is specified in {{examples-h2n}}.

### Network Treatment

During a reactive policy event, a network element is encouraged to
discard packets marked importance=false in favor of packets marked
importance=true, for the same flow.

## Packet Type - Reliable/Unreliable ('PacketType')

The "Reliable" metadata indicates if a packet is reliably transmitted by the host.

* Reliable packets are re-transmitted by the underlying transport
(e.g., TCP {{?RFC9293}} or {{QUIC}}) or re-transmitted by the appplication (e.g., {{RELIABLE-RTP}}, NTP).
* Unreliable packets are not re-transmitted by the transport
(e.g., UDP, {{RTP}}, {{LOSSY-QUIC}}) and also not re-transmitted by the application (e.g., {{RTP}}).

Packets marked reliable, if delayed excessively or dropped outright, will be re-transmitted (up to a maximum retries) by the sender application, appearing on the network again. Thus, delaying or discarding such packets does not reduce the amount of transmitted data in a network; it only defers when it appears on the network.

Reliable/Unreliable belongs to Application Metadata.

### Network Treatment

During a reactive policy event, dropping unreliable traffic is preferred over dropping reliable
traffic. The reliable traffic will be re-transmitted by the sender so dropping such traffic
only defers it until later, but this deferral can be useful.

## Packet Nature ('PacketNature')

This metadata indicates discard preference for unreliable traffic and reliable traffic, as detailed below.

### Unreliable Traffic

Packets are marked with 'prefer-keep' set to either true or false. When set to true, it indicates a preference to keep the packet. Conversely, when set to false, it signals that the packet may be subject to discard based on a reactive policy.

Many flows contain a mix of important packets and less-important packets, but applications
seldom signal that difference themselves let alone signal the difference to the network.
Rather, applications send everything over a reliable transport (TCP or QUIC) and leave it
at that, as evidenced by video streaming using TCP.

With the advent of {{LOSSY-QUIC}}, applications can send both {{QUIC}} reliable traffic and
{{LOSSY-QUIC}} unreliable traffic {{LOSSY-QUIC}} on the same 5-tuple.  With
host-to-network metadata signaling, the network can become an active assistant in such
flows during a reactive policy event by endeavouring to send the more-important 'prefer-keep'
traffic at the expense of the less-important 'may-discard' traffic.

The reason why an application transmits a packet marked as 'prefer-keep' set to false, when the
application has the capability to avoid sending that packet, is application-specific.

#### Network Treatment

During a reactive policy event, dropping packets with 'prefer-keep' set to false is preferred
over dropping 'prefer-keep' set to true packets.
Absent such discard preference indication, the network element will blindly drop packets during a reactive policy event.

### Reliable Traffic

For reliable traffic, "realtime" metadata indicates whether the packet belongs to bulk or real-time traffic.

An application such as a web browser might mark certain flows as realtime (e.g., the flow is
related to dynamically updating a search box and quick responses help the user experience)
and other flows as bulk (e.g., file download, file upload).

#### Network Treatment

Realtime traffic prefers lower latency network paths and bulk traffic prefers high throughoupt paths.


# Network to Host Metadata {#sec-n2h}

## Downlink Bitrate ('DownlinkBitrate') {#sec-dbr}

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


An example of the encoding is provided in {{examples-n2h}}.

## Prefer Alternate Path ('pref-alt-path')

There are also crisis cases where nominal network resources cannot be
used at maximum to handle packets. A network would thus seek to offload some of the
traffic during these events. Under such exceptional events, a network
element may signal to a host that it is preferrable to use alternate
paths, if available. An alternate path is typically an alternate network
attachment.  After the crisis has subsided, the network should signal
with pref-alt-path=false.

The 'pref-alt-path' metadata may be sent together with the bitrate metadata ({{sec-dbr}}) set to a very low value.

### Host Treatment

The host offloads its connections to alternate available paths.

# Guidance For Mapping Metadata to Specific Signaling Protocols

TBC.

# Implementation Impact of Metadata

## Reliable/Unreliable set by the respective transport level protocol

TCP {{?RFC9293}} is a reliable transport protocol, while UDP {{?RFC0768}} provides a minimal, unreliable, best-effort, message-passing transport to applications and other protocols (such as tunnels) that wish to operate over IP {{?RFC8085}}. Protocols built over UDP may implement reliability features at the "application" layer if such a transport feature is needed {{?RFC8304}}. For example, streams of reliable application data are sent using STREAM QUIC frames ({{Section 19.8 of ?RFC9000}}), while application data that do not require retransmission can be carried in DATAGRAM QUIC frames {{?RFC9221}}. Applications that are utilizing such a protocol, will have to choose the delivery service (reliable or loss-tolerant) based upon the nature of the packet being sent -- loss-tolerant packet cannot be carried in a reliable frame and vice-versa. Hence, based on the transport service being invoked, setting of the reliable/unreliable metadata entry can be offloaded to the underlying transport protocol, unless specifically overridden by the application.

## Offloading Loss-Avoidance to the network

Network nodes, upon learning of the nature of a packet (reliable/prefer-keep) can choose to implement loss avoidance algorithms between hops where there is packet loss detected (e.g., using out-of-band or in-band QoS measurement, which is out of the scope of this document). By doing so, end-to-end retransmissions can be reduced/avoided thereby minimizing the need for handling loss at the application layer using protocols such as {{?RFC7198}}, {{?RFC7197}}, or {{?RFC7104}}.

# Manageability Considerations

## Impact on Network Operation

TBC.

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

## Flow Metadata Registry {#sec-fmr}

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
| 2          | PacketType        | Indicates whether a packet is reliably or unreliably transmitted   | This-Document | 1.0     |
| 3          | PacketNature      | Indicates a discard preference         | This-Document | 1.0     |
| 4          | DownlinkBitrate   | Specifies the maximum downlink bitrate         | This-Document | 1.0     |
| 5          | PreferAltPath     | Sollicits the hosts to use an alternate path if available       | This-Document | 1.0     |
| 250-255    | Exp               | Reserved for private use       | This-Document | 1.0     |
{: #initial-reg title="Initial Values"}

New values in the 6-99 range can be assigned using "Standards Action" policy ({{Section 4.9 of !RFC8126}}).

Values in the 100-149 range can be assigned using "Expert Review" policy ({{Section 4.5 of !RFC8126}}).

Values in the 150-249 range can be assigned using "First Come First Served" ({{Section 4.4 of !RFC8126}}). This range can be, e.g., used by other SDOs to register metadata that are specific to their domain and which is not used outside that scope.


# Acknowledgments
{:numbered="false"}

To be completed.

--- back

# Examples of Host-to-Network Metadata Encoding {#examples-h2n}

## Video Streaming {#example-video-streaming}

Video Streaming Metadata:

The use case requirements and the table values below explained in detail in {{?I-D.rwbr-tsvwg-signaling-use-cases}}.

| Traffic type                             | Importance | PacketNature      | PacketType           |
|:----------------------------------------:|:----------:|:-----------------:|:--------------------:|
| video I-frame (key frame)                | low        | realtime          | reliable             |
| video delta P-frame                      | low        | discard           | unreliable           |
| video delta B-frame                      | low        | discard           | unreliable           |
| audio                                    | high       | realtime          | reliable             |
{: #table-video-streaming title="Example Values for Video Streaming Metadata"}

The encoding of the metadata in CDDL for the traffic will look like:
Video I-frame:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": false,
    "reliable": true,
    "realtime": true
  }
}
~~~~~

Audio:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": true,
    "reliable": true,
    "realtime": true
  }
}
~~~~~

Video delta P-frame:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": false,
    "reliable": false,
    "prefer-keep": false
  }
}
~~~~~

## Interactive Gaming or Audio/Video  {#example-interactive-av}

The use case requirements and the table values below explained in detail in {{?I-D.rwbr-tsvwg-signaling-use-cases}}.

Interactive A/V, downstream Metadata:

| Traffic type      | Importance | PacketNature      | PacketType           |
|:-----------------:|:----------:|:-----------------:|:--------------------:|
| video key frame   | low        | realtime          | reliable             |
| video delta frame | low        | discard           | unreliable           |
| audio             | high       | realtime          | reliable             |
{: #table-interactive-av-downstream title="Example Values for Interactive A/V, downstream"}

Encoding:

Video key frame:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": false,
    "reliable": true,
    "realtime": true
  }
}
~~~~~

Video delta frame:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": false,
    "reliable": false,
    "prefer-keep": false
  }
}
~~~~~

Audio:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": true,
    "reliable": true,
    "realtime": true
  }
}
~~~~~

## Remote Desktop Virtualization {#example-rdt}

Example packet metadata for Desktop Virtualization (like Citrix
Virtual Apps and Desktops - CVAD) application.

Remote Desktop Virtualization Metadata:

The use case requirements and the table values below explained in detail in {{?I-D.rwbr-tsvwg-signaling-use-cases}}.

| Traffic type               | Importance | PacketNature    | PacketType          | Comments  |
|:--------------------------:|:----------:|:---------------:|:-------------------:|:---------:|
| Glyph critical             | high       | realtime        | reliable          | The frames that form the base for the image is more critical and needs to be transmitted as reliably as possible. Retransmits of these are harmful to the UX.**|
| Interactive (or streaming) audio   | high       | keep            | unreliable          |   |
| Haptic feedback            | high       | discard         | unreliable          | Virtualizing haptic feedback is real-time and high importance although the feedback being delivered late is of no use. So dropping the packet altogether and not retransmitting it makes more sense |
| Interactive (or streaming) video key frame            | low        | keep            | unreliable          | Video key frames form the base frames of a video upon which the next 'n' timeframe of video updates is applied on. These frames, are hence, critical and without them, the video would not be coherent until the next critical frame is received. Retransmits of these are harmful to the UX. ***|
| File copy                  | low        | bulk            | reliable            |   |
| Interactive (or streaming) video predictive frame     | low        | discard         | unreliable          | Video predictive frames can be lost, which would result in minor glitch but not compromise the user activity and video would still be coherent and useful. The reception of subsequent video key frame would mitigate the loss in quality caused by lost predictive frames. |
| Glyph smoothing            | low        | discard         | Unreliable          | The smoothing elements of the glyph can be lost and would still present a recognizable image, although with a lesser quality. Hence, these can be marked as loss tolerant as the user action is still completed with a small compromise to the UX. Moreover, with the reception of the next glyph critical frame would mitigate the loss in quality caused by lost glyph smoothing elements. |
{: #table-desktop-virtualization-s2c title="Example Values for Remote Desktop Virtualization Metadata, server to client"}


Encoding:

Glyph critical:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": true,
    "reliable": true,
    "realtime": true
  }
}
~~~~~

Glyph smoothing:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": false,
    "reliable": false,
    "prefer-keep": false
  }
}
~~~~~

Interactive Audio:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": true,
    "reliable": false,
    "prefer-keep": true
  }
}
~~~~~

Haptic feedback:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": true,
    "reliable": false,
    "prefer-keep": false
  }
}
~~~~~

File copy:

~~~~~ cddl
metadata = {
  "metadata-type": 1,
  "Application Metadata": {
    "importance": false,
    "reliable": true,
    "realtime": false
  }
}
~~~~~

# Example of Network-to-Host Metadata for Video Streaming {#examples-n2h}

A network element can signal the maximum bandwidth allowed for video streaming. Typically,
this policy limit exists in cellular networks.

The example shown in {{ex-video-bitrate}} indicates the burst bandwidth (2 Mbps), burst duration
(3 seconds), and nominal (non-burst) bandwidth (1 Mbps) for the requesting
user:

~~~~~
{
  "downlinkBitrate": {
    "nominal": 1024,
    "burst": 2048,
    "burstDuration": 3000
  }
}
~~~~~
{: #ex-video-bitrate title="Example of Network-to-Host Metadata for Video Streaming"}


