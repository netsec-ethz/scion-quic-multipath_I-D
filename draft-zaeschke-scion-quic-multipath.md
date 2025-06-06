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

This document provides guidelines for using the Multipath Extension
for QUIC {{QUIC-MP}} with path aware networks (PAN).
PANs may provide hundreds of paths between endpoint, each path including
detailed path metadata that facilitates informed path selection.

Using the QUIC Multipath Extension over PAN creates pitfalls as well as
opportunities for associated algorithms, such as for path selection,
congestion control, and scheduling.
This document also gives general considerations for API design and
applications that use multipath QUIC over SCION.

This memo looks speciafically at PANs for inter domain routing.

--- middle

# Introduction

Path aware networks (PAN) may provide a large selection of
available path, each path with detailed path metadata. Path metadata
may contain path properties such as latency, bandwidth, MTU,
or geographic location, potentially with granularity down to individual
links and router interfaces.

When only one path is used, metadata and path selection
capabilities allow identifying the best path for each use case while
providing a suitable backup paths if the first paths degrades or others
paths become comparatively better.

When data is transferred over multiple paths in parallel, detailed
metadata provides information about any links where different paths
overlap and about the properties of these links.
This allows, for example, avoiding bottlenecks by choosing paths
such that they don't share links that have low bandwidth capacity.
This is useful for developing or improving algorithms, for example
for path selection, or more informed algorithms for congestion control.

The recommendations in this document are categorized into
recommendations for API design ({{apicon}}), library implementations
({{impcon}}) and algorithm design ({{algcon}}).

As a practical example of an inter-domain PAN with multipathing
capability, we refer to the SCION {{SCION-CP}}, {{SCION-DP}}.


## SCION

One example of a PAN is SCION {{SCION-CP}}, {{SCION-DP}}.
SCION is an inter-domain routing protocol that provides path metadata
on the level of individual router interfaces. This path metadata
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
{{QUIC-TRANSPORT}} and {{QUIC-MP}}. We also draws on the terminology
of SCION ({{SCION-CP}}, {{SCION-DP}}).
For ease of reference, we have included some definitions here, but
refer the reader to the references above for complete specifications
of the relevant terminiology:

**Autonomous System (AS)**: An autonomous system is a network under
a common administrative control.  For example, the network of an
Internet service provider or organization can constitute an AS.

**Endpoint**: An endpoint is the start or the end of a path,
as defined in {{PATH-VOCABULARY}}.

**Inter-AS Link**: A direct link between two external interfaces of two
ASes.

**Intra-AS Link**: A direct link between twi internal interfaces of
a single AS. A direct link may contains several internal hops.

**Link**: General term that refers to "inter-AS links" and "intra-AS
links".

**Network Path**: Consists of a 4-tuple of address/IP at each
endpoint, a list of all traversed ASes, and links inside and between
ASes, including interface IDs on border routers of each AS.

**Path ID**: The Path ID is defined in {{QUIC-MP}} . On the network
layer, it is defined by a 4-tuple of IP/port of the local and remote
endpoints.

**Path Metadata**: Path metadata is additional data that is available to
clients when they request a selection of paths to a destination.
Path metadata is authenticated wrt to owner of each link, but
otherwise not verified.
Path metadata includes data about ASes and links, such as MTU,
bandwidth, latency, AS internal hopcount, or geolocation information.
Path metadata is updated infrequently, probably at most once per
hour. Properties such as bandwidth and latency represent hardware
properties rather than live traffic information.


# Multipath Features {#mpfeatures}

This document discusses multipath features that are available in
SCION {{SCION-CP}}, {{SCION-DP}}. However, the discussion is kept
general and relevant to all PAN that support these features.


## Path Metadata {#metadata}

We assume that path metadata is reflects hardware properties rather
than live traffic information (especially for bandwidth and latency).
One should not expect path metadata is updated to be more than once
every hour. Path metadata is disseminated together with paths, so its
freshness depends on the path livetime, wgich can be several hours.

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
We assume the data is static, which means that bandwidth and latency
reflect hardware characteristics rather than current or recent load.

### Path Metadata Liveliness

We assume that path metadata is updated at most every few hours.
This should be more than sufficient since all values reflect
hardware properties rather than current traffic load.

### Path Metadata Reliability

We assume that all values are correct. The metadata is
cryptographically protected. It is signed by the data originator,
which is the relevant AS owner. However, the data correctness is not
verified, instead we rely on the AS owner to be honest.

## Path Selection

We assume that a PAN protocol allows selecting paths explicitly, based,
for example, on path metadata. Paths can be added and removed from a
connection. Paths may have a best-before data and may expire, after
which they are invalid.


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


# Potential Problems -- WIP

## 4-tuple changes {#four-tuple-changes}

If the 4-tuple (IP/port of local/remote endpoint) changes, {{QUIC-MP}}
and {{QUIC-TRANSPORT}} require several actions, including resetting
congestion control and RTT estimation algorithms, and initiating
path validation.

Using path aware networks affects this in two ways:

1. PANs can provide detailed path information that can be used to
detected path changes with high granularity.
This can affect algorithm design because they need to be less resilient
against undetected path changes (diffserv, BGP route changes, ...)
and can rely more on the path being consistent, at least to the
granualrity offered by the PAN layer.

2. PANs, such as SCION, extend the network address with path, AS code
and ISD code. ASes may use private IP ranges with IPs that are
globally not unique; This would allow a man-in-the-middle attacker to
construct a scenarion where the IP/port of an attacked machine
can be duplicated in a different AS. The attacker can then change the
destination (i.e. routing to a different AS) of a connection without
changing the destination port/IP.
This could be used to avoid path validation when coaxing a machine to
send traffic to a new destination. This attack is still not easy to
execute because it requires the attacker to have control over an AS
that lies en-route between server and client.

**TODO** Illustration.

### Mitigation

- 4-tuple is defined in {{QUIC-TRANSPORT}} as IP/port of
  local/remote endpoint. However, {{QUIC-MP}} does not clearly
  define it.
  If we assume it to mean network-address/port of local remote/endpoint
  (replacing IP with network-address) we can define it to include
  the AS number or any other relevant identifier. This would ensure
  that path validation is triggered whenever necessary.
- Send a signal to the QUIC implementation that the path has changed:
  - Use port mangling / IP mangling to emulate a 4-tuple change? ->
    That is what Anapaya does. **TODO**
  - We could trigger a "double" path validation by changing
    the port/IP to a made up value and back to the original value.
    This would trigger two path validations. At least eventually,
    the QUIC layer will know the correct remote IP/port.


# API Considerations {#apicon}

## Initialization

When a QUIC(-MP) socket is created, it can be useful (depending on
the programming language) to allow injection of a custom UDP underlay.

- Custom DNS resolver (if the library resolves URLs)
- Custom Congestion control and RTT estimation algorithms
- Custom Packet scheduling algorithm
- PAN exclusive: path selection algorithms


## Network Address and Path

Generally, it can be useful if any API of a QUIC-MP library uses
not only IP/port for addressing local/remote peers, but uses a network
address (IP + port + other identifiers, such as AS number) and a
network path.

Note that network path is different from QUIC-MP path ID. There is
usually a 1:1 mapping from network pagth to path ID, but the
network path may change while the path ID stays the same.
For example, a PAN library may have paths that can expire and
it may have a configuration option that expiring paths should be
renewed automatically. However, this works only for paths that are
otherwise (except for the expiration date) identical.



## General API...?

The API of a QUIC-MP implementation that works with PAN should
probably provide interfaces for

The API of a QUIC-MP implementation that works with PAN should:

- Expose Path ID
- Expose API for custom congestion control algorithms
- Expose callback for QUIC-MP path abandon (REFERENCE!!!)
- Expose API for name resolution (only of library does that), not
  useful if API works with IP/port only.
- Expose API for usage profile.  Applications will have very
  different requirements on a multipath API, see {{categories}}.
  A comprehensive API should therefore allow for mostly automatic
  selection of Path Selection and Congestion Control algorithms.


# QUIC implementation Considerations {#impcon}

**TODO READ:**
See also "Implementation Considerations" in {{Section 5 of QUIC-MP}}.


## Path Change Detection - Path Injection {#pathchange-injection}

### Problem

**TODO check SCION IRTF draft for this attack:**

This requires the attacker to spoof IP adresses in an AS or fully
control an AS between the attacked endpoints.

An attacker can send a spoofed packet to a server. The packet contains
a new path (for example broken or with high latency).
The server accepts the packet and stores the path for being used
for all future communication with the client. The next packet sent to
the client will take the injected malicious path and will fail (or
at least be redirected).
This happens even if the packet is rejectd by the QUIC(-MP) layer
because the SCION layer will never learn about the packet being
rejected.

In order to execute the attack, the attacker must send the forged path
from somewhere between the two attacked endoints. Usually, a border
routers that forwards traffic to a neighbour AS will validate the paths
and check that traffic with a transit path originates from another
border router.
**TODO** Verify SCION spec.
To avoid this check, the attacker must either spoof the IP address to
look like another boirder router, or it must control the border router
such that it does not perform the check.

A variant of the attack happens when the attacker is located in the
same AS a the attacke client endpoint. In that case it must only be
able to spoof the IP address to look like the attacked client's IP.
**TODO** Do border routers even check the underlay IP against the SCION
header IP? They may not do that, see NAT...


### Mitigation
- Authenticate packets on SCION level....
  There could still be a replay:
  - Client uses path P1 (P1 could be injected with wormhole attack).
  - Attacker captures packet with P1 path
  - Attacker breaks P1 and forces migration to P2
  - Attacker can no replay the original packet to get server to
    keep responding on the broken P1 path.
- Don't allow paths to change (for 4-tuple??) and require
  proper path migration (how?) or even a reconnect
- SOLUTION: The SCION layer MUST NOT cache paths locally, instead
  paths must be accepted by the QUIC layer before being used.

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
* Java over DatagramSocket: Problematic, it caches the paths...
* C + Rust: How exactly do we map PathID to paths? How are paths
  updated?

# Algorithm Considerations {#algcon}

What has changed:
- Much better MPU
- Knowledge about shared links/routers between paths
- indication of latency+bandwidth limits.
- (Potentially live traffic info: Avoid "pulsing" -> Simon Scherer?)


## General
{{QUIC-TRANSPORT}} requires that there is no connection migration
during the initial handshake, and that there are no other packets
send (including probing packets) during the initial handshake, see
{{Section 9 of QUIC-TRANSPORT}}, paragraphs 2 and 3.

We need to ensure on some level that no path change or probing occurs.

**TODO** Read {{QUIC-MP}} on 4-tuple change / path validation.
-> Section 9 in RFC 9000

## Algorithms

- MPU detection algorithm can be removed/replaced with metadata query
  See Path MTU Discovery in {{Section 14.3 of QUIC-TRANSPORT}} and
  {{Section 5.8 of QUIC-MP}}.
- Congestion control algorithms:
  - Must reset on path change (how?).
  - Should benefit from knowledge about path overlaps.
  - Follow BCP 133, see {{Section 7.10 of CC-ALGORITHMS}}.
- Roundtrip time estimator.
  - Must reset on path change (how?). See also from {{Section 5.1 of
    QUIC-MP}}:
    "If path validation process succeeds, the endpoints set the path's
    congestion controller and round-trip time estimator according to
    {{Section 9.4 of QUIC-TRANSPORT}}."
  - Should benefit from knowledge about minimum latency expected on
    a path.
- Path selections algorithms

- Latency polling
  - How bad is latency polling?
  - Traceroute can help to reduce polling (ideally, every links is
traversed only by one poll packet). Traceroute also allows to identify
  links with high variance or generally hogh latency.


# OLD PART BELOW - IGNORE

# OLD - Algorithms {#algorithmsold}

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




# QUIC-specific advantages?

**TODO** ?



# OLD - Applications {#apps}

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

See {{four-tuple-changes}}.

## Wormhole Attack

An attacking AS can manipulate metadata of its links and routers to
announce a very attractive route that would then be the first choice of
many route selection algorithms.
This would negatively affect performance of endhosts, especially if
the links are manipulated to drop all packets (break link) or drop
many packets (high drop rate).
See {{Section 7.2.4 of SCION-CP}}.

Mitigation: Due to the inherent unreliability, users should
implement sanity checks as to whether a link holds up to the
promised capabilities.


## More ?

See {{anon}}.

See other attacks in {{Section 7.2.4 of SCION-CP}}?

**TODO**

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
