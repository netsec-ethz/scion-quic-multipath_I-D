---
title: "Guidelines for QUIC Multipath over Path Aware Networks"

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
 -  ins: J. van Bommel
    name: Jelte van Bommel
    org: ETH Zurich
    email: "jelte.vanbommel@inf.ethz.ch"
 -  ins: F. Wirz
    name: Francois Wirz
    org: ETH Zurich
    email: "wirzf@inf.ethz.ch"
 -  ins: T. Zaeschke
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
  SCION-CP: I-D.draft-dekater-scion-controlplane
  SCION-DP: I-D.draft-dekater-scion-dataplane
  OLIA:
    title: "MPTCP is not pareto-optimal: performance issues and
a possible solution"
    date: "2012"
    seriesinfo: "Proceedings of the 8th international conference on
Emerging networking experiments and technologies, ACM"
    author:
    - ins: R. Khalili
    - ins: N. Gast
    - ins: M. Popovic
    - ins: U. Upadhyay
    - ins: J.-Y. Le Boudec
  UMCC:
    title: "UMCC: Uncoupling Multipath Congestion Control through
Shared Bottleneck Detection in Path-Aware Networks"
    date: "2024"
    seriesinfo: "Proceedings of the IEEE 49th Conference on Local
Computer Networks (LCN)"
    author:
    - ins: M. Gartner
    - ins: D. Hausheer

--- abstract

This document provides guidelines for using the Multipath Extension
for QUIC {{QUIC-MP}} with path aware networks (PAN).
PANs may provide hundreds of paths between endpoint, including
detailed path metadata that facilitates an informed selection.
This offers opportunities for new or improved algorithms for
multipath networking.

This document discusses mostly algorithms for path selection.
However, it also comments on congestion control and load distribution
(scheduling), as well as general considerations for
API design and applications that use multipath QUIC over SCION.

**TODO**

- Section on QUIC-specific advantages?

--- middle

# Introduction

Path aware networks (PAN) can provide many detailed metadata about
available paths, such as latency, bandwidth, MTU, geographic location,
or many other properties.

Even when just one path is used, this allows selecting the
best path for each use case while providing a suitable backup
paths if the first paths fails or becomes unatractive.

In the case of concurrent multipathing, detailed metadata provides
information about any links where different paths overlap and about the
properties of these links. This allows, for example, choosing paths
such that they don't share low bandwidth links.

This is useful for developing or improving network related algorithms,
for example for path selection, or more informed algorithms for
congestion control.

This document first identifies and categorizes multipath usage
scenarios ({{categories}}), then discusses guidelines for path selection
algorithms and suggests how these may be applicable to congestion
control algorithms, without discussing the later in detail
({{algorithms}}).
Finally, in order to facilitate these algorithms, this documents
contains suggestions for API design and general use in
applications ({{api}}).

As a practical example of a PAN and how path metadata
can be made available and path selection and routing can be
implemented, we refer to the SCION {{SCION-CP}}, {{SCION-DP}}.


## SCION

One example of a PAN is SCION {{SCION-CP}}, {{SCION-DP}}.
SCION is an interdomain routing protocol that provides path metada
on the level of individual router interfaces. This provide many
opportunities for improving congestion control and other algorithms.
They also avoid some problems that gave rise to current algorithms.

The SCION protocol makes detailed path information available to
endpoints. Besides the 4-tuple of address/IP at each endpoint, the
information includes a list of all traversed ASes and
respective links between ASes, as well as metadata about ASes and links,
including MTU, bandwidth, latency, AS internal hopcount, or
geolocation information.


## Terminology {#terms}

**Autonomous System (AS)**: An autonomous system is a network under
a common administrative control.  For example, the network of an
Internet service provider or organization can constitute an AS.

**Endpoint**: An endpoint is the start or the end of a path,
as defined in {{PATH-VOCABULARY}}.

**Inter AS link**: A direct link between two external interfaces of two
ASes.

**Intra AS link**: A direct link between twi internal interfaces of
a single AS. A direct link may contains several internal hops.

**Link**: General term that refers to "inter-AS links" and "intra-AS
links".

**Path**: Consists of a 4-tuple of address/IP at each endpoint, a list
of all traversed ASes, and links inside and between ASes, including
interface IDs on border routers of each AS.

**Path metadata**: Path metadata is additional data that is available to
clients when they request a selection of paths to a destination.
Path metadata is authenticated wrt to owner of each link, but
otherwise not verified.
Path metadata includes data about ASes and links, such as MTU,
bandwidth, latency, AS internal hopcount, or geolocation information.
Path metadata is stable, e.i. it is updated infrequently, probably at
most once per hour. Properties such as bandwidth and latency
therefore represent hardware properties rather than live traffic
information.

**SCMP**: A signaling protocol analogous to the Internet Control
Message Protocol (ICMP).  This is described in {{SCION-CP}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# API Recommendations {#apirec}

- Expose Path ID
- Expose API for custom congestion control algorithms
- Expose callback for QUIC-MP path abandon (REFERENCE!!!)
- Expose API for name resolution (only of library does that), not
  useful if API works with IP/port only.


# QUIC implementation Considerations {#imprec}

- Allow to detect path changes while 4-tuple stays the same
  - Port mangling?
  - Change PATH-ID (SCION would need to know about QUIC...!!)
  - Is PATH ID encrypted? Then attacker cannot know it (but use, it,
    see replay, which is not possible due to sequence numbers??)
    ... so what?
  - SCION could just drop packets where 4-tuple is the same but
    remote AS has changed.


## Padding
From Section 8.1 of {{QUIC-TRANSPORT}}: "Clients MUST ensure that
UDP datagrams containing Initial packets have UDP payloads of at
least 1200 bytes, adding PADDING frames as necessary."

PAN packets may be bigger than traditional packets because they may
carry additional routing information.

**TODO** Measure SCION path header!


# Algorithm Recommendations {#algrec}

What has changed:
- Much better MPU
- Knowledge about shared links/routers between paths
- indication of latency+bandwidth limits.
- (Potentially live traffic info: Avoid "pulsing" -> Simon Scherer?)


## General
{{QUIC-TRANSPORT}} requires that there is no connection migration
during the initial handshake, and that there are no other packets
send (including probing packets) during the initial handshake, see
{{QUIC-TRANSPORT}} Section 9, paragraphs 2 and 3.

We need to ensure on some level that no path change or probing occurs.

**TODO** Read {{QUIC-MP}} on 4-tuple change / path validation.
-> Sexction 9 in RFC 9000

## Algorithms

- MPU detection algorithm can be removed/replaced with metadata query
- Congestion control algorithms
- Path selections algorithms




# Multipath Features {#mpfeatures}

This document discusses multipath features that are available in
SCION {{SCION-CP}}, {{SCION-DP}}. However, the discussion is kept
general and relevant to all PAN that support these features.


## Path Metadata

### Path Metadata Granularity

We assume a protocol for inter-AS routing that provides path
information per AS, more specifically per border router of an AS
and per any link between any border routers (link may be internal
or external to an AS).

That means, if we compare multiple paths, we can see where they
overlap and what the hardware characteristics of the overlapping
parts are.

### Path Metadata Dimensions

We assume a protocol a provides information on links and border
routers, such as MTU, bandwidth, latency, geo-location, as well
as identities (interface ID, port and IP) of border routers.
We assume the data is static, wich means that bandwidth and latency
reflect hardware characteristics rather than current or recent load.

### Path Metadata Liveliness

We assume that path metadata is updated at most every few hours.
This should be more than sufficient since all values reflect
hardware properties rather than current traffic load.

### Path Metadata Reliability

We assume that all values are correct. The metadata is
cryptographically protected. It is signed by the data originator,
which is the relevant AS owner. However, the data correctness is not
verified, instead we rely on the AS onwer to be honest.

## Path Selection

We assume that a PAN protocol allows selecting paths explicitly, based,
for example, on path metadata. Paths can be added and removed from a
connection. Paths may have a best-before data and may expire, after
which they are invalid.





# OLD PART BELOW - IGNORE

# Multipath Categorization {#categories}

This document distinguishes the following usage categories:

* High bandwidth (BW): Optimizing bandwidth by parallel transfer on
  multiple paths.
* Minimum latency (LAT): Optimizing latency for low latency.
  This can be achieved by regularly checking multiple paths and using
  the one with the lowest latency or by parallel transmission over
  multiple path.
* Failure tolerance (FT): Optimizing for failure tolerance by
  parallel transfer on multiple paths.
* Evasion (EVA): Avoid certain links or ASes, for example based on MTU,
  geolocation or AS number.

The discussions of these categories are written with multiple paths
per interface in mind (i.e. multiple paths per 4-tuple).  However, they
can usually be generalized to multipathing over multiple interfaces.

These categories can be combined, for example LAT and FT may often be
combined and EVA can be can be useful in combination with any other
category.


## Disjunctness

For FT, paths are only interesting if they are disjunct.
For BW, paths should mostly be disjunct, but overlap is
acceptable if the overlapping links have high BW available (see
{{bottleneck}}).

For LAT and EVA, path disjunctness is mostly irrelevant.

**TODO** Discuss link level, router level and AS level path disjunctness.

## Path Metadata

SCION paths have associated metadata with latency and bandwidth
information. The values represent best case values and tend to be
updated rarely, for example once a day.

Path metadata may also be incomplete, ASes are not required to provide
or regularly update the data.

Users of path metadata must keep in mind that the data is mostly not
verifiable but depends on the diligence and trustworthiness of the
link owners.
However, once disseminated by a link owner, the path metadata is
authenticated an cannot be changed by other parties.

Due to the inherent unreliability, users should implement sanity
checks as to whether a link holds up to the promised capabilities.


# Algorithms {#algorithmsold}

## Path Selection {#patsel}

### Dynamic Approach

A dynamic approach could start with using low latency paths. If the
connection appears to be long lasting (e.g. at least 1 second duration
and 1MB of traffic) it could start adding additional paths and see
whether the traffic increases. Additional paths can be chosen
following the guidelines discussed in {{datra}}.

### Bottleneck Detection {#bottleneck}

If no live traffic information is available, bottleneck detection
can help to identify linkks that should be avoided. In PAN this can
be done using approaches such as {{UMCC}}.

One alternative is to use two SCMP `traceroute` commands that measure
latency between two consecutive AS border routers. The measured
latency can be compared to earlier measurements or to the latency
given in the path metadata. Discrepancies can be an indication for
high traffic and queueing problem on the measured link.

**TODO** Should we discourage this? It creates unnecessary traffic...


## MTU

SCION provides MTU information for every AS-level link on a path.


## Detecting Byzantine ASes and Links

By comparing performance (latency, bandwidth, packet drop rate, link
errors, MTU) of multiple paths, we can (with some accuracy) detect
unreliable links and ASes.
These can be blacklisted and excluded from further path selection,
or possibly kept as backup paths for emergencies.

**TODO** Add reference to research.

**TODO** Move to Path Metadata section?


## DMTP

One example of an application / algorithm is discussed in {{DMTP}}.


## Congestion Control {#concon}

General recommendations for congestion control are defined in
Congestion Control Principles {{CC-PRINCIPLES}}.
Congestion control for QUIC is discussed in
QUIC Loss Detection and Congestion Control {{QUIC-RECOVERY}}.
More generally, congestion control for UDP is discussed in the
UDP Usage Guidelines {{UDP-GUIDELINES}}. UDP MTU discovery is further
developed in {{MTU-DISCOVERY}}.

Multipath congestion control is also discussed for TCP in
{{CC-MULTIPATH-TCP}}. They state that "One of the prominent
   problems is that running existing algorithms such as standard TCP
   independently on each path would give the multipath flow more than
   its fair share at a bottleneck link traversed by more than one of its
   subflows.".
This can be avoided in SCION through link-level analysis of flows
and selecting paths that do not create or share a bottleneck link.
This avoids the need for Coupled Congestion Control.

The second point in {{CC-MULTIPATH-TCP}} is "Further, it is desirable
   that a source with multiple paths
   available will transfer more traffic using the least congested of the
   paths, achieving a property called "resource pooling" where a bundle
   of links effectively behaves like one shared link with bigger
   capacity.".
This can be achieved with a simple load distribution algorithm.

**TODO** Differentiate "load distribution" from "path selection".


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

## Load Distribution (Scheduling) {#loaddist}

Load distribution algorithms are mainly useful for high bandwidth
(HBW) scenarios. Latency may still be relevant though, for example
for high definition video streams. Scheduling halps distributing
the transfer load efficiently over multiple path.

However, sending data stream over multiple paths in parallel will
usually result in packets arriving out of order at the receiver.
This should be avoided because:

* Packet reordering requires larger buffers on the receiver side which
  are used to put packets back in order. Latency, jitter and
  drop rate on different paths directly affect the required buffer
  size.
* Head of line blocking (HOLB): the latency of the slowest packet
  determines the effective latency of the stream. This may be
  ignored for data transfers where latency is irrelevant.

These problems can be mitigated, but it is difficult to do so for both
problems at the same time.
A simple way to mitigate these problem is to select multiple paths
with similar latency.

Another way to mitigate these problems is to schedule packets on
different paths such that they are likely to arrive in (more or less)
the correct order. Care should be taken to avoid requiring an
equally large buffer on the sender side.

**TODO** Check which existing algorithms do that.

**TODO** Can we facilitate QUIC streams for this?


# Applications {#apps}

## Data Transfer {#datra}

The aim of a data transfer application is to maximize throughput,
regardless of latency, jitter or packet loss.

The solution here is to identify multiple paths that are either
disjunct, or where the non-disjunct links allow higher throughput than
other links on the same paths (i.e. high enough to prevent the
link from being a bottleneck).

## Low Latency {#lola}

There are multiple approaches to transfer traffic with the lowest
possible latency.  For example:

  1) With separate latency measurements. Latency measurements run in
  parallel to data traffic. This allows performing measurements at a
  different frequency and over many more paths than payload traffic.
  This can be useful if payload packets are large and sending them
  redundantly over multiple links is to costly or considered to invasive
  with respect to other network users.
  Low frequency measurements may be combined with traffic prediction
  algorithms in order to identify best paths between measurements.
  Due to low bandwidth overhead, this may be the most cost efficient
  approach.
  It may be possible to do this with SCION SCMP packets, either with
  ECHO packets or possibly with traceroute if the AS internal latency
  of the destination AS is negligible (because it may be small or it may
  be almost the same for all border routers that are on interesting
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


Network analysis can be used to identify, and subsequently avoid, links
with high or unreliable latency.

## High Availability / Redundancy {#redu}

An approach to high availability is to send data on multiple paths in
parallel. A tradeoff here is that sending on all available paths is
probably infeasible. Depending on cost factors, and to avoid
overloading the network, any algorithms should keep redundant
sending to a minimum.

Path analysis can be used to identify multiple paths that are mostly
or completely (using multiple interfaces) disjunct, but that still
satisfy latency, bandwidth, and other constraints.

Additional polling with SCMP or with additional QUIC stream may be
used to regularly measure packet drop rates or latency variations.

## Multipathing for Anonymity {#anon}

Multipathing could also be used for anonymity, e.g. by switching
paths at random intervals.
With a continuous data stream, care should be taken that new
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


# API Design consideration {#api}

## Multipathing for Applications

Applications will have very different requirements on a multipath API.
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
application.  This may also depend on API designer to provide such
transparent  multipathing with additional code on the application level.


# Security Considerations

This document has no security considerations.

May sending data on multiple paths in parallel, or at regular
intervals, expose information about data streams? E.g. can streams
be associated with users such that a change in connection ID does
not help anonymity?

## Latency Polling

If a user sends latency measurements on 10 paths in parallel
every 5 seconds, then these 10 paths can (with some probability)
be attributed to the same user.
Solution: do not send all probes at the same time.
To prevent patterns (1. after 0.1s, 2. after 0.3s, 3. after 0.35s, ...)
the interval between packets on each path should also vary.
Additionally, the number of polled paths should vary.

## Path Validation

Following {{QUIC-MP}} (Section 5.1) and {{QUIC-TRANSPORT}} (Section 9),
endpoints MUST drop a connection or perform path validation when the
4-tuple changes:

```
"Not all changes of peer address are intentional, or active,
migrations. The peer could experience NAT rebinding: a change of
address due to a middlebox, usually a NAT, allocating a new outgoing
port or even a new outgoing IP address for a flow. An endpoint MUST
perform path validation (Section 8.2) if it detects any change to a
peer's address, unless it has previously validated that address."
```

With SCION, endpoints may use private IPs, say 192.168.0.1. These IPs
are obviously not unique. Two paths with an identical 4-tuple may
therefor connect to two different machines if the machines are in
different ASes but use the same IP/port.

In short, an attacker can impersonate a client by using an identical
IP/port to connect a server. The server would probably just reverse
the path and answer to the client without triggering a path validation.

**TODO** What is the implication of this?

* Can an attacker, without having being able to encrypt/decrypt data,
cause any harm? Can they redurect traffic to themselves? To prevent
this, the server SCION stack should accept a new path only if the
QUIC stack also accepts it! This means we need to trigger a path
validation and somehow learn whether it worked. Or we use our own
validation process. Or we don't allow the path to change and require
a new QUIC connection (with a new SCION connection)?
SOLUTION: The SCION layer MUST NOT cache paths locally, instead paths
must be accepted by the QUIC layer before being used.

Examples:
* Java over DatagramChannel:
  * Comparing ResponsePath objects MUST return `false` if the
paths differ.
  * The QUIC layer MUST use only those addresses for sending data that
    have previously been accepted. If an attacker sends a packet, it
    should be identified as malicious, be rejected, and the path should
    not be used.
    Why would an attacker packet be accepted? Replay should be
    impossible (there are apcket sequence IDs?), and any other
    requests should be cryptographically protected.
    Can this attack be successful with spoofed IPs???
* Java over DatagramSocket: Problematic, it cahces the paths...
* C + Rust: How exactly do we map PathID to paths? How are paths
  updated?

To restore the behavior intended by QUIC, we must therefore extend
the 4-tuple to a 6-tuple of ISD-AS + IP + port on both endpoints.

**TODO**
This does not require any change in the QUIC layer but can probably
be handled in the SCION ayer (the QUIC layer does not know about the
ISD/AS code).

It seems with SCION we can insert arbitrary IP addresses into the
stack before handing incoming packets up to the QUIC layer...?




## More ?

See {{anon}}.

**TODO**

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
