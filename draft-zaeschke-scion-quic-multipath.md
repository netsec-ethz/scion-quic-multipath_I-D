---
title: "Guidelines for QUIC Multipath over Path Aware Networks"

abbrev: "SCION-QUIC-MP"

category: info

docname: draft-zaeschke-scion-quic-multipath-latest

submissiontype: IRTF  # also: "independent", "editorial", "IAB", "IRTF"

date: 2025-06-03

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
  CC-ALGORITHMS: RFC9743

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

This document provides informational guidance for using the
Multipath Extension for QUIC {{QUIC-MP}} with path aware networks (PAN).
PANs may provide hundreds of paths between endpoints, each path including
detailed path metadata that facilitates informed path selection.

Using the QUIC Multipath Extension over PAN creates pitfalls as well as
opportunities for associated algorithms, such as for path selection,
congestion control, and scheduling.
This document also gives general considerations for API design and
applications that use multipath QUIC over SCION.

This memo looks specifically at PANs for inter-domain routing.

--- middle

# Introduction

Path aware networks (PAN) may provide a large selection of
available path, each path with detailed path metadata. Path metadata
may contain path properties such as latency, bandwidth, MTU,
or geographic location, potentially with granularity down to individual
links and router interfaces.

When only one path is used, metadata and path selection
capabilities allow identifying the best path for each use case while
providing suitable backup paths if the first path degrades or other
paths become comparatively better.

When data is transferred over multiple paths in parallel, detailed
metadata provides information about any links where different paths
overlap and about the properties of these links.
This allows, for example, avoiding bottlenecks by choosing paths
such that they don't share links that have low bandwidth capacity.
This is useful for developing or improving algorithms, for example,
for path selection, or more informed algorithms for congestion control.

The recommendations in this document are categorized into
recommendations for API design ({{apicon}}), library implementations
({{impcon}}) and algorithm design ({{algcon}}).

As a practical example of an inter-domain PAN with multipathing
capability, we refer to the SCION ({{SCION-CP}}, {{SCION-DP}}).


## SCION {#scion}

One example of a PAN is SCION {{SCION-CP}}, {{SCION-DP}}.
SCION is an inter-domain routing protocol that provides path metadata
on the level of links and router interfaces. This metadata
provides many opportunities for improving congestion control and other
algorithms. It also avoids some problems that gave rise to current
algorithms.

The SCION protocol makes detailed path information available to
endpoints. Besides the 4-tuple of address/IP at each endpoint, the
information includes a list of all traversed ASes and
respective links between ASes, as well as metadata about ASes and links,
including MTU, bandwidth, latency, AS internal hopcount, or
geolocation information.



# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology {#terms}

We assume that the reader is familiar with the terminology used in
{{QUIC-TRANSPORT}} and {{QUIC-MP}}. We also draw on the terminology
of SCION ({{SCION-CP}} and {{SCION-DP}}).
For ease of reference, we have included some definitions here, but
refer the reader to the references above for complete specifications
of the relevant terminology.

**Autonomous System (AS)**: An autonomous system is a network under
common administrative control.  For example, the network of an
Internet service provider or organization can constitute an AS.

**Endpoint**: An endpoint is the start or end of a path,
as defined in {{PATH-VOCABULARY}}.

**Inter-AS Link**: A direct link between two external interfaces of two
ASes.

**Intra-AS Link**: A direct link between two internal interfaces of
a single AS. A direct link may contain several internal hops.

**Link**: General term that refers to "inter-AS links" and "intra-AS
links".

**Network Address**: In IP networks, the network address is the IP and
port of an endpoint. In PANs, the network address may hold additional information, such as the AS number of the endpoint.

**Network Path**: This term exists only in PANs. The network path
consists of the network address at each endpoint, a list of all
traversed ASes, and links inside and between ASes, including interface
IDs on border routers of each AS.

**QUIC-MP Path**: Consists of the network address at each
endpoint and a Path ID (see {{QUIC-MP}}). The Path ID allows having
multiple logical paths for the same set of network addresses.

**Path Metadata**: Path metadata is additional data that is available to
endpoints when they request a selection of paths to a destination.
Path metadata is authenticated by the owner of each link, but is
otherwise not verified.
Path metadata includes data about ASes and links, such as MTU,
hardware bandwidth, latency, AS internal hop count, or geolocation information.


# Multipath Features {#mpfeatures}

In this document, we focus on two features that PANs provide for
multipath applications: metadata and paths selection.

## Path Metadata {#metadata}

We assume that path metadata reflects hardware properties rather
than live traffic information (especially for bandwidth and latency).
One should not expect path metadata to be updated more than once
every hour.

### Path Metadata Granularity

We assume a protocol for inter-AS routing that provides path
information per AS, more specifically per border router of an AS
and per any link between any border routers (link may be internal
or external to an AS).

If we compare multiple paths, we can see where they
overlap and what the hardware characteristics of the overlapping
parts are.

### Path Metadata Dimensions

We assume a protocol that provides information on links and border
routers, such as MTU, bandwidth, latency, geo-location, as well
as identities (interface ID, port and IP) of border routers.
We assume that bandwidth and latency reflect hardware characteristics
rather than current or recent load.

### Path Metadata Liveliness

We assume that path metadata is updated at most every few hours.
This is appropriate because all values reflect hardware properties
rather than current traffic load.
Path metadata is disseminated together with paths, so its
freshness also depends on the path lifetime, which can be several hours.

### Path Metadata Reliability

We assume that all path metadata is truthful. The metadata is
cryptographically protected and signed by the data originator,
which is the relevant AS owner. However, the data correctness is not
verified, instead we rely on the AS owner to be honest.

It is possible for an AS to offer deceptive metadata, see wormhole
attack in {{Section 7.3.4 of SCION-CP}}.


## Path Selection

We assume that a PAN protocol allows explicitly selecting paths, based,
for example, on path metadata, such as latency, bandwidth, AS numbers,
or geolocation of routers.
Paths can be added and removed from a connection.
Paths may have an expiration date, after which they are invalid.

See {{patsel}} for details.



# Multipath Categorization {#categories}

This document distinguishes the following usage categories:

* High bandwidth (BW): Optimizing bandwidth by parallel transfer on
  multiple paths.
* Minimum latency (LAT): Optimizing latency for low latency.
  This can be achieved by regularly checking multiple paths and using
  the one with the lowest latency or by parallel transmission over
  multiple paths.
* Fault tolerance (FT): Optimizing fault tolerance through
  parallel transfers over multiple paths.
* Evasion (EVA): Avoid certain links or ASes, for example, based on
  MTU, geolocation or AS number. Evasiona can also imply frequent
  path switching in order to avoid tracking or detection by
  adversaries.

The discussions of these categories are written with multiple paths
per interface in mind (i.e. multiple paths per 4-tuple).  However, they
can be generalized to multipathing over multiple interfaces.

These categories can be combined. For example, LAT and FT may often be
combined and EVA can be useful in combination with any other category.



# Notable Differences and Benefits

Using QUIC or QUIC-MP over a PAN changes some of the underlying
assumptions. This provides certain benefits, such as additional
information and control over paths, but also some pitfalls.


## Endpoint Identity {#endpoint-identity}

In QUIC, endpoints are identified by their IP and port number. This
works because IP addresses are unique (globally, or in whatever
context they are used).

PANs may use a network address beyond the 4-tuple of
local/remote IP/port. For example, SCION ({{scion}}), allows IPs
from the private IP ranges (e.g. 192.168.1.1/32). To uniquely identify
endpoints globally, the AS (autonomous system) identifier is added
to fully qualify a network address.
This has consequences for security mechanisms. Implementations need
to be careful to consider the full network address, for example, when
triggering path validation (see {{attack-path-injection}} and {{token}}).


## Path Identity {#path-identity}

The identification of "paths" varies between QUIC, QUIC-MP and PANs.

- {{QUIC-TRANSPORT}} uses a 4-uple of local/remote IP/port to
distinguish paths.
- {{QUIC-MP}} extends the 4-tuple with a path IDs to distinguish
  logical paths (connections).
- PANs can typically distinguish paths through detailed
path metadata based on the physical network path. The metadata may
additionally include an expiration date that may or may not be used
to distinguish path instances.

When using a PAN, the path identity can and should be used to detect
changes even when the 4-tuple of local/remote IP/port (or equivalent)
stays the same.


**TODO reference RECOMMENDATIONS, but exclude expiration**.

Change detection can be useful to avoid unintended path changes or
to trigger actions, such as resetting congestion control or RTT
estimation algorithms.


### Interoperability of the QUIC-MP Path ID and the Network Paths

Unfortunately, there is not always a 1:1 mapping
between path IDs and networks paths as they are used in SCION.

- A PAN path may expire and should be replaceable with a new version
  without requiring a new Path ID (**TODO TBD**: We could also require
  path migration in this case)

**TODO** In future we may have more dynamic path availability with
live metadata. Do we really need path migration for every (small) route
change?

- Network address changes should trigger path validation
- Network path changes should trigger algorithm reset (CC, RTT
  estimate), see {{Section 5.1 of QUIC-MP}}.

**TODO** It seems path migration is only useful when the network
address changes?

In SCION, neither the AS nor the network path are validated and can be
forged, see {{security}} for a discussion. However, SCION provides
extensions for validating these properties, namely SPAO and EPIC.
(**TODO** remove or replace references to SPAO/EPIC?)


## Disjointness {#disjointness}

For FT, paths are more interesting if they are disjoint.
For BW, paths should mostly be disjunct, but overlap is
acceptable if the overlapping links have high BW available (see
{{bottleneck}}).

For LAT and EVA, path disjointness is less important.

**TODO** Discuss link level, router level and AS level path disjunctness.


## Path Metadata {#metadata-benefit}

SCION paths have associated metadata with latency and bandwidth
information. The values represent best case values and tend to be
updated rarely, for example every few hours (depending on the
implementation).

Path metadata may also be incomplete. ASes are not required to provide
or regularly update the data.

Users of path metadata must keep in mind that the data is mostly not
verifiable but depends on the diligence and trustworthiness of the
link owners.
However, once disseminated by a link owner, the path metadata is
authenticated and cannot be changed by other parties.

Due to the inherent unreliability, users should implement checks to
verify that a link holds up to the promised capabilities.


## Explicit Path Selection

Based on path metadata ({{metadata-benefit}}) and algorithmic analysis
({{disjointness}}), an endpoint can explicitly select paths or avoid
paths. This allows avoiding or abandoning paths for more paths with
more suitable properties.



# QUIC-specific advantages?

Describe why we are writing this document for QUIC and especially for
QUIC-MP. **TODO**



# API Considerations {#apicon}

Concrete API design depends on many factors, such as the programming
language, intended use or simply personal preference. We therefore
suggest only features, not concrete API designs.


## Initialization

When a QUIC(-MP) socket is created, it can be useful (depending on
the programming language) to allow injection of custom modules:

- Custom UDP underlay (for injecting a PAN, such as SCION).
- Custom DNS resolver (if the library resolves URLs)
- Custom congestion control and RTT estimation algorithms
- Custom Packet scheduling algorithm
- PAN exclusive: path selection algorithms. This differs from scheduling
  algorithms by determining which paths should be allowed at all and
  which should be held as backup paths.
  Path selection algorithms should have access to
`initial_max_path_id` ({{Section 2.1 of QUIC-MP}}) and MAX_PATH_ID
frames ({{Section 4.6 of QUIC-MP}} in order to know when, and how many,
paths can be created. Path selection must exclude paths that are too
long to guarantee 1200 bytes MTU for QUIC packets. The algorithm
also needs to know about PATH_AVAILABLE and PATH_BACKUP, see
{{Section 3.3 of QUIC-MP}}, as well as PATH_ABANDON {{Section 3.4 of
QUIC-MP}}. **TODO** Should we also consider probing frames here ({
{Section 3.1.2 of QUIC-MP}}).?

Many available implementations already allow injecting most of these
modules.


## Network Address and Network Path

Generally, it can be useful if any API of a QUIC-MP library uses
not only IP/port for addressing local/remote peers, but uses a network
address (IP + port + other identifiers, such as AS number) and a
network path.

Note that a "network path" is different from {{QUIC-MP}} path ID.
There is usually a 1:1 mapping from network path to path ID, but the
network path may change while the path ID remains the same.

For example:
- During path migration (**TODO** ref), a path ID may be (loosely)
associated with multiple network paths.
- A PAN library may have paths that can expire and that can be
  renewed automatically. However, this should usually result in an
  identical path (except for the expiration date).

### Recommendation {#recommendation-path-switching}

PAN libraries should, as default when used with QUIC, not
automatically switch paths, except when renewing expired paths with
otherwise identical paths.

The PAN library may have a configuration option to automatically
switch paths, potentially after probing, but this should happen
only when explicitly configured and when the overlay libraries
can detect or otherwise tolerate paths changes.

QUIC(-MP) libraries should consider the possibility that paths may
change. Typical algorithms for congestion control etc can tolerate
this. Potential implications of skipping path validation need to be
considered.

See also:
- resetting algorithms (**TODO** refs)
- path validation considerations (**TODO** refs)


## General API

The API of a QUIC-MP implementation that works with PAN should:

- Expose Path ID
- Expose API for custom congestion control algorithms
- Expose callback for QUIC-MP path abandon (REFERENCE!!!)
- Expose API for name resolution (only if the library does that), not
  useful if API works with IP/port only.
- Expose API for usage profile.  Applications will have very
  different requirements on a multipath API, see {{categories}}.
  A comprehensive API should therefore allow for mostly automatic
  selection of Path Selection and Congestion Control algorithms.



# QUIC Implementation Considerations {#impcon}

**TODO READ:**
See also "Implementation Considerations" in {{Section 5 of QUIC-MP}}.

Recommendations are summarized in {{recommendations}}


## Automatic Path Changes - Initial Handshake

{{QUIC-TRANSPORT}} requires that there is no connection migration
during the initial handshake, and that there are no other packets
sent (including probing packets) during the initial handshake, see
{{Section 9 of QUIC-TRANSPORT}}, paragraphs 2 and 3.

An implementation must ensure at some level that no path change or
probing occurs.

This may be covered by the recommendation that a PAN layer should
not automatically switch without explicit request by the QUIC(-MP)
layer. See also {{recommendation-path-switching}}

## Path Change Detection - Path Injection {#pathchange-injection}

This is a security problem and is discussed in {{attack-path-injection}}.



# Algorithm Considerations {#algcon}

The availability of a PAN layer provides additional information that
can be used by algorithms for congestion control, RTT estimation,
MTU estimation, ... and others. It also requires additional
algorithms, for example, for path selection.

The additional information includes:
- path metadata with route information, including MTU and hardware
limits on bandwidth and latency.
- comparability of multiple paths which gives knowledge about
shared links/routers between paths.


## MTU Detection

MPU detection algorithm can be removed/replaced with metadata query
See Path MTU Discovery in {{Section 14.3 of QUIC-TRANSPORT}} and
{{Section 5.8 of QUIC-MP}}.


## Congestion Control

Congestion control (CC) algorithms can benefit from exact knowledge of
a path:

- When using multiple paths, a CC algorithm can access information
as to if and where the paths overlap and some of the properties of the
overlapping sections.
- If implemented by the QUIC-MP library, a CC algorithm can be notified
  of every path change, allowing it to reset only when necessary.
  Without a PAN layer, a CC algorithm has to rely on 4-tuple changes
  to trigger a reset, with the drawback that changes may go unnoticed
  (in case of path change without 4-tuple change) or that a reset may
  be unnecessary (in case only the NAT mapping changed while leaving
  the rest of the route intact).

See also {{Section 5.3 of QUIC-MP}}.

General recommendations for congestion control are defined in
Congestion Control Principles {{CC-PRINCIPLES}}.
Congestion control for QUIC is discussed in
QUIC Loss Detection and Congestion Control {{QUIC-RECOVERY}}.
More generally, congestion control for UDP is discussed in the
UDP Usage Guidelines {{UDP-GUIDELINES}}. UDP MTU discovery is further
developed in {{MTU-DISCOVERY}}.


### Coubpled Congestion Control {#concon}

Multipath congestion control is also discussed for TCP in
{{CC-MULTIPATH-TCP}}. They state that:

> "One of the prominent
   problems is that running existing algorithms such as standard TCP
   independently on each path would give the multipath flow more than
   its fair share at a bottleneck link traversed by more than one of its
   subflows.".

This can be avoided in PANs through link-level analysis of flows
and selecting paths that do not create or share a bottleneck link.
This avoids the need for traditional Coupled Congestion Control.
Instead, this bottleneck knowledge may be used to effectively use
separate congestion control for each path, or at least let coupled
congestion control focus on known bottlenecks links.

Note that in the case of SCION, it uses different paths from IP.
While some of the links may carry both IP and SCION traffic, the
SCION traffic typically has its own bandwidth allocation.

There are several congestion control algorithms proposed in literature,
e.g. LIA, OLIA, BALIA and RSF.  These combine congestion control with
path selection algorithms.
For simplicity, we suggest separating concerns in terms of
congestion control and path selection. This allows us to better
tailor the solutions to the different usage scenarios.

The proposition is to use non-coupled congestion control per path,
tailored for each use case (max bandwidth / low latency) and
on top of that separately use path selection algorithms.


## RTT Estimation {#rtt}

- Must reset on path change (how?). See also from {{Section 5.1 of
  QUIC-MP}}:
  "If path validation process succeeds, the endpoints set the path's
  congestion controller and round-trip time estimator according to
  {{Section 9.4 of QUIC-TRANSPORT}}."
- {{Section 5.4 of QUIC-MP}} describes how data packets and
acknowldegment packets may be sent on different paths, making it
difficult to detemine the RTT.
  - With PANs, the path is known, so it is easy to tell whether
    data and acknowledgment were sent on the same path or not.
  - With PANs, we could recommend a policy (**TODO recommend?**)
    that ACKs should be sent on the return path of the data
    (**TODO** why are they sent on different paths? Besides path
    abandon?)
  - With PANS, explicit path probing is easier.
  - We can try to derive lower and upper limits from analyzing
    latency of non-disjoint paths.
- Should benefit from knowledge about minimum latency expected on
  a path, see {{metadata}}.
- This allso affects packet scheduling, see {{Section 5.5 of
  QUIC-MP}}.


**TODO** Has this the same purpose as RTT estimation?

- Latency polling
  - How bad is latency polling?
  - Traceroute can help to reduce polling (ideally, every link is
traversed only by one poll packet). Traceroute also allows identifying
  links with high variance or generally high latency.
  - An implementation should avoid sending too many polling packets
    at once, see {{Section 7.2 of QUIC-MP}}.


## Retransmission & PTO

See {{Section 5.6 of QUIC-MP}} and {{Section 5.7 of QUIC-MP}}.


## DMTP

One example of an application / algorithm is discussed in {{DMTP}}.


## Path Selection {#patsel}

### Dynamic Approach

A dynamic approach could start with using low latency paths. If the
connection appears to be long lasting (e.g. at least 1 second duration
and 1MB of traffic) it could start, and keep, adding additional paths
and as long as the traffic increases. Additional paths can be chosen
following the guidelines discussed in {{datra}}.


### Bottleneck Detection {#bottleneck}

If no live traffic information is available, bottleneck detection
can help to identify linkks that should be avoided. In PAN this can
be done using approaches such as {{UMCC}}.

One alternative is to use two SCMP `traceroute` commands that measure
latency between two consecutive AS border routers. The measured
latency can be compared to earlier measurements or to the latency
given in the path metadata. Discrepancies can be an indication of
high traffic and queueing problems on the measured link.

**TODO** Should we discourage this? It creates unnecessary traffic...


## Packet Scheduling {#scheduling}

Packet scheduling algorithms are mainly useful for high bandwidth
(HBW) scenarios. Latency may still be relevant though, for example,
for high definition video streams. Scheduling helps distributing
the transfer load efficiently over multiple paths.

However, sending data streams over multiple paths in parallel will
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
A simple way to mitigate these problems is to select multiple paths
with similar latency.

Another way to mitigate these problems is to schedule packets on
different paths such that they are likely to arrive in (more or less)
the correct order. Care should be taken to avoid requiring an
equally large buffer on the sender side.

**TODO** Check which existing algorithms do that.

**TODO** Can we facilitate QUIC streams for this?


# Applications {#apps}

See also {{categories}}. **TODO merge with Categories?**

## Data Transfer {#datra}

The aim of a data transfer application is to maximize throughput,
regardless of latency, jitter or packet loss.

The solution here is to identify multiple paths that are either
disjoint, or where the non-disjoint links allow higher throughput than
other links on the same paths (i.e. high enough to prevent the
link from being a bottleneck).

## Low Latency {#lola}

There are multiple approaches to identify low latency paths.
For example:

1) Rely only on metadata. This is the fastest approach and may
   give a path that is already "good enough".

2) Traceroute on a few selected paths. Traceroute gives latencies
   for each segment. By considering overlapping linnk in known paths,
   the paths for traceroutes can be selected such that a minimum of
   traceroutes provides latency onformation for a lorge number of
   links. These links can then be used to select the most promising
   path.

   Traceroute also has the advantage that it does not put load on
   the server.

   However, traceroute may not give information for the last mile
   to the remote endpoint.

3) Use RTT estimation, see {{rtt}}.

4) With implicit measurements through QUIC ACK frames.
  {{Section 13.2 of QUIC-TRANSPORT}} recommends sending an ACK frame
  after receiving at least two ack-eliciting packets or after a delay.
  If the application sends data on multiple paths in parallel, this may
  be sufficient for some low latency applications.
  {{QUIC-ACKFREQUENCY}} proposes an extension that allows more control
  over how often and when ACK-FRAMES are sent.
  This approach can be useful if {{QUIC-ACKFREQUENCY}} becomes
  available or if the ACK behavior of the standard QUIC server is
  sufficient.

5) Deadline aware multipathing is described in {{DMTP}}.


Generally, instead of probing many paths at once, an implementation
should probe only the most promising paths (following the metadata).
This should allow excluding many other paths if their metadata latency
(which should represent best-case latency) is larger than any latency
measured on the previously selected paths.

Probing too many paths should also be avoided to avoid overloading individual links, and it may effectively be limited (except traceroute)
by the abvailable path IDs and connection IDs, see
{{Section 7.2 of QUIC-MP}}.


## High Availability / Redundancy {#redu}

An approach to high availability is to send data on multiple paths in
parallel. A tradeoff here is that sending on all available paths may
be infeasible because of the number of available paths (with SCION we
often see 100+ paths to a destination).
Depending on cost factors, and to avoid overloading the network, any
algorithms should keep redundant sending to a minimum. See also
{{Section 7.2 of QUIC-MP}} for DoC security considerations.

Path analysis can be used to identify multiple paths that are mostly
or completely (using multiple interfaces) disjoint, but that still
satisfy latency, bandwidth, and other constraints.

Additional polling with SCMP or with an additional QUIC stream may be
used to regularly measure packet drop rates or latency variations.

## Multipathing for Anonymity {#anon}

Multipathing could also be used for anonymity, e.g., by switching
paths at random intervals.
With a continuous data stream, care should be taken that new
paths are not just switched over from one to the next, otherwise
traffic characteristics may be used to identify paths.
For example, paths could be identified by packet frequency, packet
burst frequency, or general bandwidth.
For continuous streams, just moving one stream from one path to another
may expose stream identity.

As mitigation, a sender could start sending on a new path for a
while before stopping sending on the old path. The sender could send
redundant data or random data at a suitable bandwidth.

**TODO** Should we really recommend bandwidth usage.

**TODO** Is this SCION specific?
SCION allows for choosing paths based on trusted or untrusted ASes,
but this is not specific to multipathing...



# Summary of Recommendations {#recommendations}

**TODO** This memo is informational but we have some MUST and SHOULD
here.

- MUST: Enforce endpoint identity by requiring QUIC-MP
implementations to provide a destination network address instead of
just a IP/port.
Related: The SCION stack should not store/cache paths, especially not
on the server side.
- SHOULD: Enable QUIC-MP implementations to recognize network path
changes beyond 4-tuple changes
- MUST: SCION stack should by default not change the network paths,
possibly with the exception of refreshing expired paths. When a path
stops working (link errors, etc), it should instead report an error
to the QUIC(-MP) layer or time out silently.
- On a server, the PAN layer SHOULD return probing packets on the same
  network path on which they were received, this greatly simplifies RTT
  estimation, see {{rtt}}
- Generally, a server SHOULD respond on the same path on which the data
  was originally requested, unless the new path has been validated.
  This ensures that the path does not violate the path policy of the
  client.
  This also ensures that we are only using paths of packets that
  have been accepted by the QUIC(-MP) layer or above. This protects
  against several attacks, see {{attack-path-injection}}.
- {{jelte-attack}}: servers should/must authenticate paths. **TODO**

Possible workaround: port/address mangling?

Future: how to handle dynamic traffic data? This requires anyway a
tighter integration if CC algorithms.


## Address Validation Token {#token}

**TODO** See discussion in https://github.com/quicwg/multipath/issues/550

From {{Section 3.1.3 of QUIC-MP}}:
> As specified in {{Section 9.3 of QUIC-TRANSPORT}}, a server is
> expected to send a new address validation token to a client
> following the successful validation of a new client address.
> The client will receive several tokens. When considering using a
> token for subsequent connections, it may be difficult for the
> client to pick the "right" token among multiple tokens obtained in
> a previous connection.
> The client is likely to fall back to the strategy specified in
> Section 8.1.3 of [QUIC-TRANSPORT], i.e., pick the last received
> token. To avoid issues when clients make the "wrong" choice, a
> server SHOULD issue tokens that are capable of validating any of
> the previously validated addresses. Further guidance on token usage
> can be found in Section 8.1.3 of [QUIC-TRANSPORT].

Clients may not know their IP address (e.g. NAT) and their IP address
may change.

As discussed (**TODO** elsewhere: trigger path validation, reset CC
and RTT estimation algorithms), QUIC-MP implementations should
consider not only the 4-tuple, but also the AS codes and actual paths
when comparing network addresses.

One problem here is as follows:
If we adopt an implementation to use the full network address + path
for identity, and if we use this to generate tokens, then we may end
up generating many more or longer tokens.

This needs to be considered carefully.

See also {{Section 21.3 of QUIC-TRANSPORT}}.

**TODO** Move this section to _after_ discussing network addresses

**TODO** read the referenced sections and come up with recommendations.



# Security Considerations {#security}

THe aim is that {{QUIC-MP}} over PANs retains all security
properties of {{QUIC-MP}}. However, this requires some
implementation changes and additional consideration regarding:

- endhost identity: a 4-tuple is not sufficient to identify an endhost;

- netwotk path authenticity: paths may be spoofed;

- probing patterns which may expose user intentions or identity.



## Probe Fingerprinting

An endpoint may probe multiple paths in order to determine the best
path(s) for a given usecase. One example of probing packets are
packets that measure round trip time (RTT).

If sent en block, probing packets can be detected because they
may be sent in bulk, to the same destination, in regular intervals,
and all with slightly different paths attached.

This can be used to fingerprinting an endpoints or their intentions
(applications may have unique intervals definded).

This can be mitigated by varying and generally reducing the
number of probing packets, and by sending probing packets
not en block but time-shifted.


## Path Injection {#attack-path-injection}

There are several potential attacks that build on injecting paths
(valid or invalid) into server-side software stack.

These attacks can be prevented in several ways, we recommend the
following where possible:

1. PAN layers should avoid storing/caching paths and network addresses
   (beyond IP/port) internally.
   Instead, they should be given to the QUIC(-MP) layer or the application layer. That means that path information would only be accepyted and retained if the QUIC(-MP) or application layer.
2. PAN layers and QUIC(-MP) layers should interface  by using
   network addersses that include all information that identifies an  andpoint, including, for example, AS code. Any change to a
   network address (including the AS code) should trigger path
   validation.

Alternatives:
1. If paths and network addresses must be stored in the PAN layer, an
   alternative solution would be to implement some kind of signalling
   which would indicate that a packet is (or would be) rejected/dropped
   by the QUIC(-MP) layer. These addresses and path from such packets
   should not be added to storage. However, to avoid connection
   drop, they should not be removed if they were previously used with
   a valid connection.

Examples of attacks include memory exhaustion attacks, traffic
redirection attacks, and traffic amplification attacks.



### Memory Exhaustion

An attacker may flood a server with packets that each have a
different source network address. If these are stored in the PAN layer,
they may cause memory exhaustion.

Mitigation: do not store state in the PAN layer, or implement
a way to clean up state without affecting valid connection.


### Traffic Redirection to Different AS

An attacker may craft a packet that appears to originate from the same
IP/port, but is located in a different AS than an existing connection.
If the server's PAN layer stores paths internally, and uses IP/port
as key to look them up, then the new paths may replace the exisitng one
and outgoing traffic is redirected to the new paths and destination.

Mitigation:

- The QUIC(-MP) layer MUST trigger path validation if the
  network address changes, and must consider every attribute of the address, not just IP and port.
- If a packet is rejected by the QUIC(-MP) layer, the PAN layer MUST
  NOT add it to any local state (including not replacing exisint state).
  This can be achieved trivially by not having state in the PAN layer.


### Traffic Redirection over Different Path

An attacker may craft a path with a network address that is identical
to an existing valid client, but with a different path.

The new route is invalid (contains loops or non-existent links)
or may be broken (contains links that are broken or have high latency or drop rate).

The new route may also work fine, but violate the client's path policy
or be used for traffic observation.


~~~~
     AS #100               AS #200                   AS #300
   +------------+        +----------------+        +----------------+
   | Server     |        | Attacker       |        | Victim         |
   | IP=1.2.3.4 | ------ | IP=192.168.0.1 | ------ | IP=192.168.0.1 |
   | port = 42  |        | port= 12345    |        | port= 12345    |
   +------------+        +----------------+        +----------------+

    **TODO** use IPs from documentation range
~~~~
{: #fig-example-non-unique-ip title="Example of non-unique IPs"}

This attack requires either spoofing of the client's IP address
(when the attacker is in the same AS as the client) or injection
of a path (which requires control over an AS that is en-route
between the client and server).

Mitigation: **TODO**


### Traffic Amplification

An attacker may establish a normal connection with a server,
request a large amount of data, and then inject a path that
redirects to a victim that has the same IP/port, but in a different AS.

If the server-side QUIC-MP does not trigger path validation
(because IP/port are the same), then it may implicitly accept
the new path and send the requested data to a victim.

This attack requires the attacker to have control over an AS that
is en-route between client (victim) and server.

Mitigation:

- A QUIC(-MP) library must consider all attributes
(not just the 4-tuple) when checking for a change in the network
address. This would then trigger path validation and the attack
can be averted.
- If a QUIC(-MP) library cannot compare additional attributes
  (e.g. legacy library), the PAN layer (server side) should have an
  option to perform port mangling or IP mangling: when the PAN layer
  detects a new network address that differs only in the AS number
  from a previously seen address (IP/port are the same), then it
  should perform IP/port mangling, i.e. reporting a modified IP or
  port to the QUIC(-MP) layer. This new IP/port would trigger a path validation or algorithm reset where required.

Caveats

- Offering a mangeled IP/port to the application may have
implications for application correctness, such as displaying an
unexpected IP/port.


~~~~
   Attacker                                                Server
   (Establish connection)
   Path=[#200, #100] -->
                                                <-- Path=[#100, #200]

   (Change Path)
   Path=[#300, #200, #100] -->
                                          <-- Path=[#100, #200, #300]

   Client: receives unwanted traffic
~~~~
{: #fig-example-new-path title="Example of traffic amplification
attack"}

## Jelte's Attack {#jelte-attack}

**TODO**


## More ?

- Use multipathing for anonymity, see EVA in {{categories}}.

- See other attacks in {{Section 7.2.4 of SCION-CP}}?

**TODO**

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
