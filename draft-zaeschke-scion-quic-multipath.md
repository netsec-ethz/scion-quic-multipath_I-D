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
* Fault tolerance (FT): Optimizing for fault tolerance by
  parallel transfer on multiple paths.
* Evasion (EVA): Avoid certain links or ASes, for example based on MTU,
  geolocation or AS number.

The discussions of these categories are written with multiple paths
per interface in mind (i.e. multiple paths per 4-tuple).  However, they
can be generalized to multipathing over multiple interfaces.

These categories can be combined, for example LAT and FT may often be
combined and EVA can be can be useful in combination with any other
category.


# Notable Differences and Benefits

## Endpoint Identity and Path Identity {#identity}

PANs may provide or require an extended definition of idenity.

In a PAN such as SCION ({{scion}}), networks paths are known to the
endpoints.
This can, and should, be used to detect changes even when the 4-tuple
of local/remote IP/port (or equivalent) stays the same. Change
detection can be useful to avoid unintended path changes or to trigger
actions, such as resetting congestion control or RTT estimation
algorithms.

Separately, PANs may use a network address beyond the 4-tuple of
local/remote IP/port. For example, SCION ({{scion}}), allows IPs
from the private IP ranges (e.g. 192.168.1.1/32). To uniquely identify
endhosts globally, the AS (autonomous system) identifier is added
to fully qualify a network address.
This has consquences for security mechanisms. Implementations need
to be careful to consider the full network address, for example when
triggering path validation (see {{four-tuple-changes}} and {{token}}).


### Path ID

{{QUIC-MP}} specifies a path ID that is used to identify and
distinguish multiple "paths". However, there is not always a 1:1 mapping
between path IDs and networks paths as they are use in SCION.

- A PAN path may expire and should be replaceable with a new version
  without requiring a new Path ID (**TODO TBD**: We could also require
  path migration in this case)


**TODO** In future we may have more dynamic path availablility with
live metadata. Do we really need path migration for every (small) route
change?

- Network address changes should trigger path validation
- Network path changes should trigger algorithm reset (CC, RTT
  estimate), see {{Section 5.1 of QUIC-MP}}.

**TODO** It seems path migration is only useful when the network
address changes?

In SCION, neither the AS no the network path are validated and may be
forged, see {{security}} for a discussion. However, SCION provides
extrensions for validating these properties, namely SPAO and EPIC.
(**TODO** remove or replace references to SPAO/EPIC?)


## Disjointness {#disjointness}

For FT, paths are only interesting if they are disjoint.
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

Path metadata may also be incomplete, ASes are not required to provide
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




# Some Pitfalls -- WIP -- Renema to Guidelines

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
send traffic to a new destination as described in {{Section 9.3.1 of
QUIC-TRANSPORT}}. This attack is still not easy to
execute because it requires the attacker to have control over an AS
that lies en-route between server and client.

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


**TODO**
This attack works regardless of path validation:
The attacker can just intercept path validation requests and answer
them!

However, for that, the attacker really neds to be on-path (e.g.
control a border router).

This attack also works without PAN, e.g. with BGP, but with PAN
it is more controlled, the attacker can use a wormhole to attract
traffic and then ensure that traffic goes via their controlled AS.



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
If we adopt an implementation to use the full nettwork address + path
for identity, and if we use this to generate tokens, then we may end
up generating many more or longer tokens.

This needs to be considered carefully.

See also {{Section 21.3 of QUIC-TRANSPORT}}.

**TODO** Move this section to _after_ discussing network addresses

**TODO** read the referenced sections and come up with recommendation.



# API Considerations {#apicon}

Concrete API design depends on many factors, such as the programming
language, intended use or simply personal preference. We therefore
suggest only features, not concrete API designs.


## Initialization

When a QUIC(-MP) socket is created, it can be useful (depending on
the programming language) to allow injection of a custom modules:

- Custom UDP underlay (for injecting a PAN, such as SCION).
- Custom DNS resolver (if the library resolves URLs)
- Custom congestion control and RTT estimation algorithms
- Custom Packet scheduling algorithm
- PAN exclusive: path selection algorithms. This differs from scheduling
  algorithms by determiniing which paths should be allowed at all and
  which should be held as backup paths.
  Path selection algorithms should have access to
`initial_max_path_id` ({{Section 2.1 of QUIC-MP}}) and MAX_PATH_ID
frames ({{Section 4.6 of QUIC-MP}} in order to know when, and how many,
paths can be created. Path selection must exclude paths that are too
long to guarantee 1200 bytes MTU for QUIC packets. The algorithm
also needs to know about PATH_AVAILABLE and PATH_BACKUP, see {
{Section 3.3 of QUIC-MP}}, as wellas PATH_ABANDON {{Section 3.4 of
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
network path may change while the path ID stays the same.

For example:
- During path migration (**TODO** ref), a path ID may be (loosely)
associated with multiple network paths.
- A PAN library may have paths that can expire and that can be
  renewed automatically. However, this is should usually result in an
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
- Expose API for name resolution (only of library does that), not
  useful if API works with IP/port only.
- Expose API for usage profile.  Applications will have very
  different requirements on a multipath API, see {{categories}}.
  A comprehensive API should therefore allow for mostly automatic
  selection of Path Selection and Congestion Control algorithms.



# QUIC implementation Considerations {#impcon}

**TODO READ:**
See also "Implementation Considerations" in {{Section 5 of QUIC-MP}}.


## Automatic Path Changes - Initial Handshake

{{QUIC-TRANSPORT}} requires that there is no connection migration
during the initial handshake, and that there are no other packets
send (including probing packets) during the initial handshake, see
{{Section 9 of QUIC-TRANSPORT}}, paragraphs 2 and 3.

An implementation must ensure on some level that no path change or
probing occurs.

This may be covered by the recommendation that a PAN layer should
not automatically switch without explicit request by the QUIC(-MP)
layer. See also {{recommendation-path-switching}}

## Path Change Detection - Path Injection {#pathchange-injection}

### Problem

**TODO check SCION IRTF draft for this attack:**

This requires the attacker to spoof IP adresses in an AS or fully
control an AS between the attacked endpoints.
The attacker also needs to learn of the port/IP that the client is
using, either by probing the client or by reading traffic between
client and server. (Instead of knowing the port, the attacker may flood
the server with bad paths).

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

### Variant

Attack target: server / client.

The attacker can flood the server with packet that contain
slightly varying paths (maybe just client IP/port difference).
The server may hold paths in a hashmap. This map will either run
out of memory (server breaks down) or the map will start dropping
paths (including the client's path -> connection disrupted).
On the server, the SCION layer cannot easily drop paths because it
doesn't know which packets are valid or invalid.

Mitigation:
- The SCION layer should avoid storing state (such as a hashmap
with paths). If that cannot be avoided, it should monitor the size of
the hashmap and stop accepting paths if it is too full. It is important
that it does "stop accepting" rather than "cleaning out old", otherwise
it may drop the valid connection to the valid client.

**TODO** Check with scionproto




# Algorithm Considerations {#algcon}

What has changed:
- Much better MPU
- Knowledge about shared links/routers between paths
- indication of latency+bandwidth limits.
- (Potentially live traffic info: Avoid "pulsing" -> Simon Scherer?)




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
  - {{Section 5.4 of QUIC-MP}} describes how data packets and
acknowldegement packets may be sent on different paths, making it
difficult to detemine the RTT.
    - With PANs, the path is known, so it is easy to tell whether
      data and acknowledgement were sent on the same path or not.
    - With PANs, we could recommend a policy (**TODO recommend?**)
      that ACKs should be sent on the return path of the data
      (**TODO** why are they sent on different paths? Outside path
      abandon?)
    - With PANS, explicit path probing is easier.
    - We can try to derive lower and upper limits from analysing
      latency of non-disjoint paths.
  - Should benefit from knowledge about minimum latency expected on
    a path, see {{metadata}}.
  - This allso affects packet scheduling, see {{Section 5.5 of
    QUIC-MP}}.
- Retransmission & PTO:
  See {{Section 5.6 of QUIC-MP}} and {{Section 5.7 of QUIC-MP}}.
- Path selections algorithms

- Latency polling
  - How bad is latency polling?
  - Traceroute can help to reduce polling (ideally, every links is
traversed only by one poll packet). Traceroute also allows to identify
  links with high variance or generally hogh latency.


# OLD PART BELOW - IGNORE

# OLD - Algorithms {#algorithmsold}

**TODO** It would be good to have some introductory text here.
What type of algorithms are presented?

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

Note that SCION uses different paths from IP. While some of the links
may carry both IP and SCION traffic, the SCION traffic typically has
its own bandwidth allocation.

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
disjoint, or where the non-disjoint links allow higher throughput than
other links on the same paths (i.e. high enough to prevent the
link from being a bottleneck).

## Low Latency {#lola}

There are multiple approaches to transfer traffic with the lowest
possible latency.  For example:

  1) With separate latency measurements. Latency measurements run in
  parallel to data traffic. This allows performing measurements at a
  different frequency and over many more paths than payload traffic.
  This can be useful if payload packets are large and sending them
  redundantly over multiple links is to costly or considered too
  invasive with respect to other network users.
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
parallel. A tradeoff here is that sending on all available paths may
be infeasible because of the number of available paths (with SCION we
often see 100+ paths to a destination).
Depending on cost factors, and to avoid
overloading the network, any algorithms should keep redundant
sending to a minimum.

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

**TODO** Can we be more concrete here on how SCION would be used
with a current QUIC implementation?

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



# Security Considerations {#security}

THe aim is that {{QUIC-MP}} over PANs retains all security
properties of {{QUIC-MP}}. However, this requires some
implementation changes and additional consideration regarding:

- endhost identity: a 4-tuple is not sufficient to identify an endhost;

- netwotk path authenticity: paths may be spoofed;

- excessive probing patterns which may expose user intentions or
  identity.

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

See other attacks in {{Section 7.2.4 of SCION-CP}}?

**TODO**

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
