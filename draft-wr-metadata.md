---
title: "Flow Metadata"
abbrev: "Flow Metadata"
category: std


docname: draft-wr-metadata-latest
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
    fullname: Dan Wing
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    email: ["danwing@gmail.com"]
 -
    fullname: Sridharan Rajagopalan
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    email: sridharan.rajagopalan@cloud.com
 

normative:


informative:


--- abstract

Metadata about packet importance is useful to signal to network components.

This document describes how to exchange metadata in JSON which can be
mapped to a Connection Identifier (CID), and how that same metadata can
be carried in each packet as a UDP option.

--- middle

# Introduction

blah blah

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
    +<== End-to-End Secure Connection User Data with CID ==>+
    |                             |                         |
~~~~~
{: #design-approaches artwork-align="center" title="Candidate Design Approaches"}


{{fig-arch}} provides a sample network diagram of a CIDFI system showing two
bandwidth-constrained networks (or links) depicted by "B" and
CIDFI-aware devices immediately upstream of those links, and another
bandwidth-constrained link between a smartphone handset and its Radio
Access Network (RAN).  This diagram shows the same protocol and same mechanism
can operate with or without 5G, and can operate with different administrative
domains such as Wi-Fi, an ISP edge router, and a 5G RAN.

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
+------+   +------+ | +------+            |          |
|CIDFI-|   |CIDFI-| | |CIDFI-|            |          |
|aware |   |aware | | |aware |  +------+  |          |   +----------+
|client+-B-+Wi-Fi +-B-+edge  +--+router+------+      |  +---------+ |
+------+   |access| | |router|  +------+  |   |      | +--------+ | |
           |point | | +------+            |   |      | | CIDFI- | | |
           +------+ |                     | +-+----+ | | aware  | | |
                    |                     | |router+---+ QUIC or| | |
+---------+         | +------+            | +-+----+ | | DTLS   | |-+
| CIDFI-  |         | |CIDFI-|            |   |      | | server |-+
| aware   |         | |aware |  +------+  |   |      | +--------+
| client  +-----B-----+RAN   +--+router+------+      |
|(handset)|         | |router|  +------+  |          |
+---------+         | +------+            |          |
                    |                     |          |
                    |                     | Transit  |  Server
   User Network     |    ISP Network      | Network  |  Network
~~~~~
{: #fig-arch artwork-align="center" title="Network Diagram" :height=88}


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Metadata exchanged

1. Relative priority within the same stream.  Applications running over
UDP can indicate which packets within the same UDP 4-tuple are more
important than other packets.

2. blah blah

# Metadata Signaling

The following sections explain how metadata is exchanged between systems.

## Mapped to CID

As described in {{?I-D.wing-cidfi}} the QUIC or DTLS CID can be mapped
to packet metadata. This mapping is conveyed using over an HTTPS connection
between the CIDFI client and the CIDFI network elements.

JSON syntax:

  CID:
    priority: high/medium/low



## UDP Options

UDP options can carry metadata in each packet as described below.

16 bits for magic cookie and versioning.

2 bits for high/medium/low priority,
   0b00=low
   0b01=medium
   0b10=high
   0b11=reserved



# Security Considerations

To be completed.

# IANA Considerations

None.


# Acknowledgments
{:numbered="false"}

To be completed.


