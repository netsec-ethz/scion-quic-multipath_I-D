---
title: "QUIC Multipath over SCION"
abbrev: "SCION-QUIC-MP"
category: info

docname: draft-zaeschke-scion-quic-multipath-latest
submissiontype: IRTF  # also: "independent", "editorial", "IAB", "IRTF"
# number:
date: 2025-05-07
# consensus: true
# v: 3
ipr: trust200902
# area: AREA
workgroup: PANRG
keyword:
 - SCION
 - QUIC
 - multipath
venue:
  group: WG
  type: Working Group
  mail: panrg@irtf.org
  arch: https://datatracker.ietf.org/rg/panrg
  github: "netsec-ethz/scion-quic-multipath_I-D"
  latest: "https://netsec-ethz.github.io/scion-quic-multipath_I-D/draft-zaeschke-scion-quic-multipath.html"

author:
 -
    ins: J. van Bommel
    name: Jelte van Bommel
    org: ETH Zurich - NetSec Group
    email: "tilmann.zaeschke@inf.ethz.ch"
 -
    ins: T. Zaeschke
    name: Tilmann Zaeschke
    role: editor
    org: ETH Zurich - NetSec Group
    email: tilmann.zaeschke@inf.ethz.ch

normative:
  DCCP-UDPENCAP: rfc6773
  MPTCP-ARCHITECTURE: rfc6182
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
  RFC9473:
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
# area: Transport
# workgroup: QUIC Working Group
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

  I-D.rustignoli-scion-overview:
    title: SCION Overview
    date: 2025
    target: https://datatracker.ietf.org/doc/draft-dekater-panrg-scion-overview/
    author:
      -
        ins: C. de Kater
        name: Corine de Kater
        org: SCION Association
        email: c_de_kater@gmx.ch
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: SCION Association
        email: nic@scion.org
      -
        ins: A. Perrig
        name: Adrian Perrig
        org: ETH Zuerich
        email: adrian.perrig@inf.ethz.ch

  I-D.dekater-scion-controlplane:
    title: SCION Control Plane
    date: 2023
    target: https://datatracker.ietf.org/doc/draft-dekater-scion-controlplane/
    author:
    -
      ins: C. de Kater
      name: Corine de Kater
      org: SCION Association
    -
      ins: M. Frei
      name: Matthias Frei
      org: SCION Association
    -
      ins: N. Rustignoli
      name: Nicola Rustignoli
      org: SCION Association


--- abstract

This document gives general recommendations when using the Multipath
Extension for QUIC {{QUIC-MP}} with SCION
{{I-D.rustignoli-scion-overview}}.  The recommendations
concern algorithms for congestion control and path selection, as
well as general considerations for API design and applications that use
multipath QUIC over SCION.

--- middle

# Introduction

The SCION protocol makes detailed path information available to
endpoints. Besides the 4-tuple of address/IP at each endpoint, the
information includes a list of all traversed ASes and
respective links between ASes, as well as metadata about ASes and links,
such as bandwidth, latency, AS internal hopcount, or geolocation
information.

In the context of multipathing, this path information can be useful
for algorithms that select paths (see {{patsel}}) and perform
congestion control (see {{concon}}).

In order to facilitate these algorithms, this documents contains
suggestions for API design and general use in applications.

Multipath usage profiles are categorized into including data
transfer {{datra}}, low latency {{lola}} and high availability /
redundancy {{redu}}.

One example of an application / algorithm is discussed in {{DMTP}}.


## Terminology {#terms}

**Autonomous System (AS)**: An autonomous system is a network under
a  common administrative control.  For example, the network of an
Internet service provider or organization can constitute an AS.

**Core AS**: Each SCION isolation domain (ISD) is administered by a
set of distinguished autonomous systems (ASes) called core ASes,
which are responsible for initiating the path discovery and path
construction process (in SCION called "beaconing").

**Data Plane**: The data plane (sometimes also referred to as the
forwarding plane) is responsible for forwarding data packets that
endpoints have injected into the network.  After routing information
has been disseminated by the control plane, packets are forwarded
by the data plane in accordance with such information.

**Egress/Ingress**: refers to the direction of travel.  In SCION,
path construction with beaconing happens in one direction, while
actual traffic might follow the opposite direction.  This document
clarifies on a case-by-case basis whether 'egress' or 'ingress'
refers to the direction of travel of the SCION packet or to the
direction of beaconing.

**Endpoint**: An endpoint is the start or the end of a SCION path,
as defined in {{RFC9473}}.

**Hop Field (HF)**: As they traverse the network, path segment
construction beacons (PCBs) accumulate cryptographically protected
AS-level path information in the form of Hop Fields.  In the data
plane, Hop Fields are used for packet forwarding: they contain the
incoming and outgoing interface IDs of the ASes on the forwarding path.

**Info Field (INF)**: Each path segment construction beacon (PCB)
contains a single Info field, which provides basic information
about the PCB.  Together with Hop Fields (HFs), these are used to
create forwarding paths.

**Interface Identifier (Interface ID)**: A 16-bit identifier that
designates a SCION interface at the end of a link connecting two
SCION ASes, with each interface belonging to one border router. Hop
fields describe the traversal of an AS by a pair of interface IDs
called `ConsIngress` and `ConsEgress`, as they refer to the ingress
and egress interfaces in the direction of path construction
(beaconing). The Interface ID MUST be unique within each AS.
Interface ID 0 is not a valid identifier as implementations MAY use
it as the "unspecified" value.

**Isolation Domain (ISD)**: In SCION, Autonomous Systems (ASes) are
organized into logical groups called Isolation Domains or ISDs.
Each ISD consists of ASes that span an area with a uniform trust
environment (e.g. a common jurisdiction).  A possible model is for
ISDs to be formed along national boundaries or federations of nations.

**Leaf AS**: An AS at the "edge" of an ISD, with no other
downstream ASes.

**MAC**: Message Authentication Code.  In the rest of this document,
"MAC" always refers to "Message Authentication Code" and never to
"Medium Access Control".  When "Medium Access Control address" is
implied, the phrase "Link Layer Address" is used.

**Path Authorization**: A requirement for the data plane is that
endpoints can only use paths that were constructed and authorized
by ASes in the control plane.  This property is called path
authorization.  The goal of path authorization is to prevent
endpoints from crafting Hop Fields (HFs) themselves, modifying HFs
in authorized path segments, or combining HFs of different path
segments.

**Path Control**: Path control is a property of a network
architecture that gives endpoints the ability to select how their
packets travel through the network.  Path control is stronger than
path transparency.

**Path Segment**: Path segments are derived from path segment
construction beacons (PCBs).  A path segment can be (1) an up
segment (i.e. a path between a non-core AS and a core AS in the
same ISD), (2) a down segment (i.e. the same as an up segment, but
in the opposite direction), or (3) a core segment (i.e., a path
between core ASes).  Up to three path segments can be used to create
a forwarding path.

**SCMP**: A signaling protocol analogous to the Internet Control
Message Protocol (ICMP).  This is described in
{{I-D.dekater-scion-controlplane}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Multipath Scenarios

This document distinguishes the following usage scenarios:

* High bandwidth (HBW): Optimizing bandwidth by parallel transfer on
  multiple paths.
* Minimum latency (MinLat): Optimizing latency by regualrily checking
  multiple paths and using the one with the lowest latency.
* Minimum latencythrouh redundancy (MinLatRed): Optimizing latency
  by parallel transmission on multiple path.
* Failure tolerance (FT): Optimizing for failure tolerance by
  parallel transfer on multiple paths.


## Disjunctness

For FT, paths are only interesting if they are disjuct.
For HWB, paths should mostly be disjunct, but overlap is
acceptable if the overlapping links have high BW available.

For MinLat and MinLatRed, path disjunctness is mostly irrelevant.

## Path Metadata

SCION paths have associated metadata with latency and bandwidth
information. The values represent best case values and tend to be
updated rarely, for example once a day.

Path metadata may also be incomplete, ASes are not required to provide
or regularly update the data.


# Algorithms

## Detecting Bad ASes and Links




## Congestion Control {#concon}

Congestion control is also discussed for TCP in {{RFC6356}}.

## Path Selection {#patsel}

### Hybrid approach
A hybrid approach could start with using low latency paths. If the
connection appears to be long lasting (e.g. at least 1 second duration
and 1MB of traffic) it could start adding additional paths and see
whether the traffic increases. Additional paths can be chosen
following the guidlines discussed in {{datra}}.



# Applications (#apps}

## Data Transfer {#datra}

## Low Latency {#lola}

## High Availability / Redundancy {#redu}

# API Design consideration

Applications will have very different requirments on a multipath API.
A comprehensive API should therefore allow for mostly automatic
selection of {{patsel}} Path Selection and Congestion Control
algorithms {{concon}}.

At the same time it should give access to SCION paths and their metadata
to allow implementation of custom algorithms.



# Security Considerations

This document has no security considerations.

TODO ?

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
