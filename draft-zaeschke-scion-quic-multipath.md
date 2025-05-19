---
title: "QUIC Multipath over SCION"

abbrev: "SCION-QUIC-MP"

category: info

docname: draft-zaeschke-scion-quic-multipath-latest

submissiontype: IRTF  # also: "independent", "editorial", "IAB", "IRTF"

number:

date: 2025-05-07

consensus: true

v: 3

ipr: trust200902

area: IRTF

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

stand_alone: yes

pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. van Bommel
    name: Jelte van Bommel
    org: ETH Zurich
    email: "jelte.vanbommel@inf.ethz.ch"
 -
    ins: T. Zaeschke
    name: Tilmann Zaeschke
    role: editor
    org: ETH Zurich
    email: tilmann.zaeschke@inf.ethz.ch

normative:
  DCCP-UDPENCAP: rfc6773
  MPTCP-ARCHITECTURE: rfc6182
  QUIC-TRANSPORT: rfc9000
  QUIC-TLS: rfc9001
  QUIC-RECOVERY: rfc9002
  CC-PRINCIPLES: RFC2914
  UDP-GUIDELINES: RFC8085
  MTU-DISCOVERY: RFC8899

informative:
  DMTP: I-D.draft-tjohn-quic-multipath-dmtp
  QUIC-ACKFREQUENCY: I-D.draft-ietf-quic-ack-frequency
  QUIC-MP: I-D.draft-ietf-quic-multipath
  CC-MULTIPATH-TCP: RFC6356
  PATH-VOCABULARY: RFC9473
  SCION-OVERVIEW: I-D.draft-dekater-panrg-scion-overview
  SCION-CP: I-D.draft-dekater-scion-controlplane
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


--- abstract

This document gives general recommendations when using the Multipath
Extension for QUIC {{QUIC-MP}} with SCION
{{SCION-OVERVIEW}}.  The recommendations concern
algorithms for congestion control and path selection, as well as
general considerations for API design and applications that use
multipath QUIC over SCION.

This document discusses multipathing mainly in the sense of multiple
paths per 4-tuple (client IP/port + server IP/port). Multipathing over
multiple interfaces is mentioned but not discussed in detail.

--- middle

# Introduction

The SCION protocol makes detailed path information available to
endpoints. Besides the 4-tuple of address/IP at each endpoint, the
information includes a list of all traversed ASes and
respective links between ASes, as well as metadata about ASes and links,
such as MTU, bandwidth, latency, AS internal hopcount, or geolocation
information.

In the context of multipathing, this path information can be useful
for algorithms that select paths (see {{patsel}}) and perform
congestion control (see {{concon}}).

In order to facilitate these algorithms, this documents contains
suggestions for API design and general use in applications.

Multipath usage profiles are categorized into data transfer {{datra}},
low latency {{lola}} and high availability / redundancy {{redu}}.

One example of an application / algorithm is discussed in {{DMTP}}.


## Terminology {#terms}

**Autonomous System (AS)**: An autonomous system is a network under
a common administrative control.  For example, the network of an
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
as defined in {{PATH-VOCABULARY}}.

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

**Inter AS link**: A direct link between two external interfaces of two
ASes.

**Intra AS link**: A direct link between twi internal interfaces of
a single AS. A direct link may contains serveral internal hops.

**Link**: General term that refers to "inter AS links" and "intra AS
links".

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

**Path**: Besides the 4-tuple of address/IP at each endpoint, the
information includes a list of all traversed ASes and
respective links between ASes, as well as metadata about ASes and links,
such as MTU, bandwidth, latency, AS internal hopcount, or geolocation
information.

**Path metadata**: Path metadata is additional data that is available to
clients when they request a selection of paths to a destination.
Path metadata is authenticated wrt to owner of each link, but
otherwise not verified.

**SCMP**: A signaling protocol analogous to the Internet Control
Message Protocol (ICMP).  This is described in {{SCION-CP}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Multipath Scenarios

This document distinguishes the following usage scenarios:

* High bandwidth (HBW): Optimizing bandwidth by parallel transfer on
  multiple paths.
* Minimum latency (MinLat): Optimizing latency by regualrily checking
  multiple paths and using the one with the lowest latency.
* Minimum latency throuh redundancy (MinLatRed): Optimizing latency
  by parallel transmission on multiple path.
* Failure tolerance (FT): Optimizing for failure tolerance by
  parallel transfer on multiple paths.

The disciussions of these scenarios are written with multiple paths
per interface in mind (i.e. multiple paths per 4-tuple).  However, they
can usually be generalized to multipathing over multiple interfaces.

**TODO** "Usually" should be replaced with "Unless noted otherwise" or
possibly "cannot be generalized, unless noted otherwise". Let's see.


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

Users of path metadata must keep in mind that the data is mostly not
verifyable but depends on the diligence and trusworthyness of the
link owners.
However, once disseminated by a link owner, the patgh metadata is
authenticated an cannot be changed by other parties.

Due to the inherent unreliability, users should implement sanity
checks the verify that a link holds up to the promised capabilities.


# Algorithms

## MTU

SCION provides MTU information for every AS-level link on a path.


## Detecting Bad ASes and Links

By comparing performance (latency, bandwidth, packet droprate, link
errors, MTU) of multiple paths, we can (with some accuracy) detect
unreliable links and ASes.
These can be blacklisted and excluded from further path selection,
or possibly kept as backup paths for emergencies.

**TODO** Add reference to ressearch.

**TODO** Move to Path Metadata section?


## Congestion Control {#concon}

General recommendations for congestion control are defined in
Congestion Control Principles {{CC-PRINCIPLES}}.
Congestion control for QUIC is discussed in
QUIC Loss Detection and Congestion Control {{QUIC-RECOVERY}}.
More generally, congestion control for UDP is discussed in the
UDP Usage Guidelines {{UDP-GUIDELINES}}. UDP MTU discovery is further
developed in {{MTU-DISCOVERY}}.

Multipath congestion control is also discussed for TCP in
{{CC-MULTIPATH-TCP}}.

There are several congestion control algorithms proposed in literature,
e.g. LIA, OLIA, BALIA and RSF.  These combine congestion control with
path selection algorithms.
For simplicity, we suggest separating concerns in terms of
congestion control and path selection. This allows us to better
tailor the solutions to the different scenarios discussed in
(SCENARIOS).
The proposition is to use standard congestion control per path (LIST
HERE), tailored for each use case (max bandwidth / low latency) and
on top of that separately use path selection (load distribution)
algorithms.

**TODO** Terminology: path selection vs load distribution?


## Path Selection {#patsel}

### Hybrid approach
A hybrid approach could start with using low latency paths. If the
connection appears to be long lasting (e.g. at least 1 second duration
and 1MB of traffic) it could start adding additional paths and see
whether the traffic increases. Additional paths can be chosen
following the guidlines discussed in {{datra}}.



# Applications (#apps}

## Data Transfer {#datra}

The aim of a data transfer application is to maximize throughput,
regardless of latency, jitter or packet loss.

The solution here is to identify multiple paths that are either
disjunct, or where the non-disjunct links allow higher throughput than
other pinks on the same paths (i.e. high enough to prevent the
link from being a bottleneck).

## Low Latency {#lola}

There are multiple approaches to transfer traffic with the lowest
possible latency.  For example:

  1) With separate latency measurents. Latency measurments run in
  parallel to data traffic. This allows performing measurements at a
  different frequency and over many more paths than payload traffic.
  This can be useful if payload packets are large and sending them
  reduntantly over multiple links is to costly or considered to invasive
  with respect to other network users.
  Low frequency measurements may be combined with traffic prediction
  algorithms in order to identify best paths between measurements.
  Due to low bandwidth overhead, this may be the most cost efficient
  approach.
  It may be possible to do this with SCION SCMP packets, either with
  ECHO packets or possibly with traceroute if the AS internal latency
  of the destination AS is negligible (because it may be small of it may
  be almost the same for all border routersthat are on interesting
  paths).

  2) With implicit measurements through QUIC ACK frames.
  {{Section 13.2 of QUIC-TRANSPORT}} recommends sending an ACK frame
  after receiving at least two ack-eliciting packets or after a delay.
  If the application sends data on multiple paths in parallel, this may
  be sufficient for some low latency applications.
  {{QUIC-ACKFREQUENCY}} proposes an extension that allows more control
  over how often and when ACK-FRAMES are sent.
  This approach can be useful if {{QUIC-ACKFREQUENCY}} becomes
  available or if the ACK behavior of the standard QUIC server is
  sufficient.

  3) With implicit measurements through application specific ACKs.
  This is useful if the application can sensibly be adapted to have its
  own ACK protocol.


Network analysis can be used to identify, and subsequenlty avoid, links
with high or unreliable latency.

## High Availability / Redundancy {#redu}

A approach to high availability is to send data on multiple paths in
parallel. A tradeoff here is that sending on all available paths is
probably infeasibale. Depending on cost factors, and to avoid
overloading the network, any algorithms should keep redundant
sending to a minimum.

Path analysis can be used to identify multiple paths that are mostly
or completely (using multiple interfaces) disjunct, but that still
satisfy latency, bandwidth, and other constraints.

Additional polling with SCMP or with additikonal QUIC stream may be
used to regularly measure packet drop rates or latency variations.

## Multipathing for Anonymity {#anon}

Multipathing could also be used for anonymity, e.g. by switching
paths at random intervalls.
With a continuous datastream, care should be taken that new
paths are not just switched over from one to the next, otherwise
traffic characteristics may be used to identify paths.
For example, paths could be identified by packet frequency, packet
burst frequency or general bandwidth.
For continuous stream, just moving one stream from one path to another
may expose stream identity.

As mitigation, a sender could start sending on a new path for a
while before stopping sending on the old path. The sender could send
redundant data or random data at a suitable bandwidth.

**TODO** Should we really recommend bandwidth usage.

**TODO** Is this SCION specific?
SCION allows for choosing paths based on trusted or untrusted ASes,
but this is not specific to multipathing...


# API Design consideration

## Multipathing for Applications

Applications will have very different requirments on a multipath API.
A comprehensive API should therefore allow for mostly automatic
selection of {{patsel}} Path Selection and Congestion Control
algorithms {{concon}}.

At the same time it should give access to SCION paths and their metadata
to allow implementation of custom algorithms.

## Algorithm Integration

Several proposals in {{lola}} and {{redu}} suggest sending data
redundantly in parallel on multiple paths.
Similarly, some proposals suggest sending packets purely for latency
measurements.

Congestion control algorithms and path selection algorithms should
try to hide parallel transfer and measurement streams from the
application.  This may also depend on API designer to such
transparent  multipathing with additional code on the application level.


# Security Considerations

This document has no security considerations.

May sending data on multiple paths in parallel, or at reguklar
intervballs, expose information about data streams? E.g. can streams
be associated with users such that a change in connection ID does
not help anonymity?

## Latency Polling

If a user sends latency measurements on 10 paths in parallel
every 5 seconds, then these 10 paths can (with some probablility)
be attributed to the same user.
Solution: do not send all probes at the same time.
To prevent patterns (1. after 0.1s, 2. after 0.3s, 3. after 0.35s, ...)
the intervall between packets on each path should also vary.
Additionally, the number of polled paths should vary.

## More ?

See {{anon}}.

**TODO**

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
