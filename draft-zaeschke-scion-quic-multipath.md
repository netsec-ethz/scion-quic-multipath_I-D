---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "QUIC Multipath over SCION"
abbrev: "SCION-QUIC-MP"
category: info

docname: draft-zaeschke-scion-quic-mp-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date: 2025-05-07
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - SCION
 - QUIC
 - multipath
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: netsec-ethz/scion-quic-multipath_I-D
  latest: https://example.com/LATEST

author:
 -
    ins: J. van Bommel
    name: Jelte van Bommel
    organization: ETH Zurich - NetSec Group
    email: your.email@example.com
 -
    ins: T. Zaeschke
    name: Tilmann Zaeschke
    role: editor
    org: ETH Zurich - NetSec Group
    email: tilmann.zaeschke@inf.ethz.ch

normative:
  DCCP-UDPENCAP: rfc6773 
  MPTCP-ARCHITECTURE: rfc6182
  MPTCP-CONGESTION: rfc6356
  QUIC-TRANSPORT: rfc9000
  QUIC-TLS: rfc9001
  QUIC-RECOVERY: rfc9002

informative:
  DMTP: 
    title: "Deadline Aware Streams in MP-QUIC"
    date: "2025-03-03"
    author: 
    - 
      ins: T. John
    -
      ins: T. Riechard 
  RFC6356:
  OLIA:
    title: "MPTCP is not pareto-optimal: performance issues and
a possible solution"
    date: "2012"
    seriesinfo: "Proceedings of the 8th international conference on
Emerging networking experiments and technologies, ACM"
    author:
    -
      ins: R. Khalili
    -
      ins: N. Gast
    -
      ins: M. Popovic
    -
      ins: U. Upadhyay
    -
      ins: J.-Y. Le Boudec
  QUIC-MP:
    title: Multipath Extension for QUIC
    date: 2025
    area: Transport
    workgroup: QUIC Working Group
    author:
    -
      fullname:
          :: "刘彦梅"
          ascii: "Yanmei Liu"
    -
       fullname:
         :: "马云飞"
         ascii: "Yunfei Ma"
    -
       ins: Q. De Coninck
       name: Quentin De Coninck
    -
       ins: O. Bonaventure
       name: Olivier Bonaventure
    -
       ins: C. Huitema
       name: Christian Huitema
    -
       ins: M. Kuehlewind
       name: Mirja Kuehlewind


--- abstract

Using Multipath Extension for QUIC {{QUIC-MP}} with SCION provides unique
opportunities for application but also for for congestion control, path
selection and related algorithms.
This document discusses some opportunities and general recommendations.

--- middle

# Introduction

We consider several application profiles, including data transfer {{datra}},
low latency {{lola}} and high availability / redundancy {{redu}}.

One example of an application / algorithm is discussed in {{DMTP}}.

The aim of this document is to provide guideliens for designing and 
implementing multipathing over SCION. Some key differences to traditional
(non-SCION) networks are the availability of the actual route (on the 
granularity of autonomous systems) and the availability of path metadata,
such as latency and banwidth, for path segments.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Algorithms

## Congestion Control {#concon}

## Path Selection {#patsel}

### Hybrid approach
A hybrid approach could start with using low latency paths. If the 
connection appears to be long lasting (e.g. at least 1 second duration
and 1MB of traffic) it could start adding additional paths and see whether
the traffic increases. Additional paths can be chosen following the 
guidlines discussed in {{datra}}.
  


# Applications (#apps}

## Data Transfer {#datra}

## Low Latency {#lola}

## High Availability / Redundancy {#redu}

# API Design consideration

Applications will have very different requirments on a multipath API.
A comprehensive API should therefore allow for mostly automatic selection
of {{patsel}} Path Selection and Congestion Control algorithms {{concon}}.

At the same time it should give access to SCION paths and their metadata
to allow implementation of custom algorithms.



# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
