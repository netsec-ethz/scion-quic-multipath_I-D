---
title: "Guidelines for QUIC Multipath over SCION"

abbrev: "SCION-QUIC-MP"

category: info

docname: draft-zaeschke-scion-quic-multipath-latest

submissiontype: IRTF  # also: "independent", "editorial", "IAB", "IRTF"

date: 2025-07-04

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

v: 3

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
Multipath Extension for QUIC with the SCION networking technology.

SCION is an inter-domain routing protocol that supports path-aware
multi-path networking.
The multiple paths and their associated path information offered
by SCION provide opportunities as well as challenges for
combining QUIC-MP with SCION.

This document explores various aspects of this combination, such as
algorithms for congestion control, RTT estimation, and general
application scenarios.
In addition, it provides techniques and guidance to maintain the security
of QUIC-MP and SCION,
and to leverage path-aware multi-path networking with QUIC-MP.

--- middle

# Introduction

The Multipath Extension for QUIC {{QUIC-MP}}, is an extension for
the QUIC protocol that enables simultaneous usage of multiple
paths for a single QUIC connection.

SCION ({{SCION-CP}}, {{SCION-DP}}) is an inter-domain routing protocol
that offers explicit path selection between two endpoints,
typically from a large selection of paths, where paths have detailed
information on traversed autonomous systems (ASes), links, router
interfaces, and other information.

Despite their complementary goals, QUIC-MP and PANs have evolved
largely in isolation. QUIC-MP has been designed with traditional
IP-based routing in mind, where path changes are typically inferred
from endpoint address changes (i.e., 4-tuples), and where routing is
opaque to the transport layer.
In contrast, Path-Aware Networks (PANs), such as SCION, enable
informed path selection based on performance, disjointness,
policy, or security requirements.
This combination of QUIC-MP and SCION allows for optimizations, for
example, for congestion control, RTT estimation, failure recovery,
performance, and security.
However, the slightly different assumptions on endpoint addresses
(4-tuple + path ID vs 4-tuple + AS code + path) and path
lifecycles (path abandon vs expiry, etc.) can cause some pitfalls.

The purpose of this document is to explore how QUIC-MP and SCION
can interoperate, how we can leverage the path awareness offered
by SCION, and suggestions on how to overcome challenges.

This document
lists notable points when using QUIC-MP over SCION ({{overview}}),
looks at different usage scenarios ({{categories}}),
gives implementations considerations ({{teccon}}) for library
developers of SCION and QUIC-MP,
and discusses general security considerations
({{security-considerations}}).

While we provide guidelines for these areas, we do not
discuss concrete algorithms, APIs, QUIC-MP or SCION implementations,
or QUIC-MP user applications; these are considered out of scope.

Some considerations are independent of multipathing and may be
directly applicable for using {{QUIC-TRANSPORT}} over SCION.

# Terminology and Conventions

{::boilerplate bcp14-tagged}

## Terminology {#terms}

We assume that the reader is familiar with the terminology used in
{{QUIC-TRANSPORT}} and {{QUIC-MP}}. We also draw on the terminology
of {{PATH-VOCABULARY}} and of SCION ({{SCION-CP}} and {{SCION-DP}}).
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

**Network Address**:
In traditional IP networks, this refers to the tuple of address and
port of an endpoint.
In SCION, the network address consists of an _ISD-AS identifier_ and
a _host address_, typically expressed as `[ISD-AS, Host]`.

**Network Path**: The network path is the sequence of network elements
between the two endpoints (e.g., ASes, interfaces, internal links).
In traditional IP networks, the network path is typically opaque to
the endpoints.

**QUIC-MP Path**: Consists of the network address at each
endpoint and a Path ID (see {{QUIC-MP}}). The Path ID allows having
multiple logical paths for the same set of network addresses.

**Path Metadata**: Path metadata is additional data that is available
to endpoints when they request a selection of paths to a destination.
Path metadata is authenticated by the owner of each link, but is
otherwise not verified. This data describes properties of
traversed ASes and links, such as their identity or MTU.

**Metadata Extension**:
SCION offers additional path metadata via extensions. The metadata
includes properties such as bandwidth and latency. The extension
is widely supported but not further discussed here as it is not specified
in {{SCION-CP}} or {{SCION-DP}}.


# Overview of QUIC Multipath in Path-Aware Networks {#overview}

The Multipath Extension for QUIC (QUIC-MP) is primarily designed for
traditional IP networks, where each path is identified by a 4-tuple of
local and remote IP addresses and ports.
Routing is considered opaque to endpoints, and path changes are inferred
indirectly through changes in the 4-tuple. However, the
path between two endpoints may change unpredictably due to routing
dynamics, which is not captured by the 4-tuple.

In SCION, endpoints can discover and select explicit network paths,
which are described at the level of ASes and inter-AS links, and have
associated metadata, such as the MTU.

The different underlying assumptions of QUIC-MP and SCION result in
some mismatches, for instance:

- **Endpoint Ambiguity**: PANs such as SCION use composite addresses
  (e.g., ISD-AS + Host), where the host can be an IP address from a
  private IP range. This breaks the assumption that IP addresses
  are unique in the current network.
- **Path Identity Mismatch**: In QUIC-MP, paths are distinguished by
  4-tuples and Path IDs. In SCION, two distinct physical paths may share
  the same 4-tuple, rendering transport-layer path tracking incomplete.
- **Routing and Path Lifecycle**: Paths in SCION can expire, be revoked,
  or be reissued, even when 4-tuple information is unchanged.
  This can interfere with RTT estimation, congestion control, and path
  validation logic if not properly accounted for.

However, SCION also opens up opportunities for enhancing
multipath transport:

- **Explicit path selection** enables endpoints to choose disjoint
  paths, i.e., paths that do not share links or segments, to improve
  fault tolerance against link failures, or to increase aggregate
  throughput.
- **Path metadata** allows endpoints to prioritize paths with more
  suitable properties. For instance, low-latency and low-hop-count
  paths can be prioritized for RTT measurements, avoiding wasting
  resources on probing paths with poorer characteristics. Or, path
  overlap or disjointness can easily be determined and used for
  congestion control.
- **Path-awareness** allows congestion control and RTT
  estimators to reset only when the underlying network path has actually
  changed, something not reliably detectable in traditional IP networks,
  where network path changes may occur due to routing, despite a
  constant 4-tuple.
- The availability of **stable network paths** in PANs allows the
  transport layer to distinguish between actual path changes and
  transient network conditions, enabling more accurate RTT estimation
  and congestion control.


#  Multipath Use Cases and Categorization {#categories}
This section categorizes common use cases for multipath transport and
highlights how PANs enhance each scenario. Many of these use cases can
be combined to meet complex application requirements.

## Fault Tolerance and Availability
Multipath transport can improve robustness against network failures in
several ways:

- An endpoint can immediately send data over a predetermined backup
  path if it suspects or obtains a network notification that a primary
  path is faulty.
- In SCION, the network may send error messages that explicitly specify
  the faulty link. This allows selecting a backup path that circumvents
  the faulty link.
- By sending redundant traffic over multiple paths, an application can
  maintain continuity even if one path becomes unavailable.

The use of backup paths or paths for redundant sending can be further
improved by analyzing paths for overlap and selecting disjoint paths.
This reduces the likelihood of multiple paths failing simultaneously.


## High Throughput

An application may aim to maximize the available bandwidth by spreading
traffic across multiple paths.
To optimize this, an application may:

- Select multiple paths with maximum disjointness.
- Select multiple paths such that disjointness is limited to
  sub-paths that are expected to have high bandwidth available.
- When congestion is detected, switch over entirely, or shift
  parts of traffic to an underutilized disjoint path to preserve the
  throughput.

## Low Latency

Latency-sensitive applications benefit from selecting the fastest
available path at any moment. In PANs, endpoints may estimate latency
from explicit metadata or infer it from probing. Because in PANs paths
are stable and explicitly selectable, the transport layer can maintain
multiple low-latency options in parallel, and either transmit in
parallel, or switch traffic to a different path once the latency
starts to fluctuate.

Latency may be determined by RTT estimation, see {{rtt}}.

For deadline sensitive applications, an algorithm as described in
{{DMTP}} may be useful.

Instead of probing many paths at once, an implementation
should probe only the most promising paths (based on the metadata).
Probing many paths should also be avoided to avoid overloading
individual links, and it may effectively be limited (except traceroute)
by the available path IDs and connection IDs, see
{{Section 7.2 of QUIC-MP}}.


## Policy-Driven Routing and Routing Constraints

Some applications or deployments may wish to avoid routing traffic
through certain ASes, e.g. to ensure path diversity or to enforce
routing policies. SCION enables this by
making path selection explicit and verifiable.

## Anonymity and Traffic Obfuscation

Multipath transport can be used to reduce the observability of traffic
by distributing it across multiple network paths. In PANs, endpoints
can explicitly select disjoint paths to minimize the risk that a single
AS observes the full traffic flow.

Randomizing path selection and packet scheduling can help obscure
traffic patterns. However, traffic characteristics such as packet
timing, flow duration, or volume may still be linkable across paths.
Mechanisms such as probing should therefore be designed and used such
that it avoids creating identifiable patterns.


## Gateways and Proxies {#sig}

There are gateways and proxies (including VPN) that translate
SCION traffic to IP traffic and back.
These are a special case because they are not used together with a
QUIC(-MP) implementation, instead they are, and should be, oblivious
to QUIC traffic.

**TODO**
These, along with NATs, will be discussed in a future version of this
document.


# Technical Considerations {#teccon}

Using QUIC or QUIC-MP over a PAN, such as SCION, changes some of the
underlying assumptions. This provides certain benefits, such as
additional information and control over paths, but also some challenges.


## Addressing {#endpoint-identity}

SCION uses composite addresses (ISD/AS + IP + port), where the IP
address can be from a private IP range. This breaks the assumption
that IP addresses are unique in the scope of the network.

### Key Implications

QUIC-MP relies on the 4-tuple changes to trigger path validation.
However, with SCION, the 4-tuple does not uniquely identify an endpoint.
Two endpoints with identical IP/port could be in different ASes.
An attacker could use endpoints with identical 4-tuple to reroute traffic
to a different machine without triggering path validation, see
{{attack-path-injection}} and {{token}}.

The implication of a skipped path validation is that a sever can be coaxed into
sending QUIC packets to an unexpecting client. This is similar to the attacks
described in {{Section 8 of QUIC-TRANSPORT}} and
{{Section 21.5.4 of QUIC-TRANSPORT}}.


### Recommendations

Attacks can be prevented by ensuring that one of the following is achieved:

- Prevent the attacker from exploiting a skipped path validation.
- Ensure that path validation is not skipped.

#### Avoid Exploitation Of Skipped Path Validation

In order to exploit a skipped path validation event, an attacker must
first inject an incorrect path or SCION source address into the server.
This can be prevented as follows:

1) Border routers must verify that any packet whose path indicates origin
   outside of the local AS, the underlay IP must match a known border router
   IP.
2) Border routers must verify that for any packet that originates in the
   local AS, the underlay IP matches the SCION SRC IP.
3) SCION server libraries must respond to the underlay IP from which a
   request was received.

These measures ensure that every packet has a correct SCION SRC IP and path.
The only way to circumvent these checks is by spoofing the IP address in a
local AS or by taking control over a border router of an AS along the route
between two peers.


#### Ensure That Path Validation Is Performed

Implementations should ensure that path validation is triggered when
the network address of the destination changes; this includes
IP, port and AS number. This protects against several attacks, see
{{attack-path-injection}} and especially {{attack-amplification}}.

There are several ways to achieve this, for example:

- Adapt the QUIC-MP library to be aware of the AS number in SCION
  network addresses.
- If the network address is available as a single "object",
  the SCION layer can extend this with the AS code and the
  QUIC-MP implementation must only ensure to compare the whole
  object instead of port and IP separately.
- The SCION implementation could detect cases where only the AS
  changes and then mangle the port or IP to trigger a path validation
  in the QUIC-MP layer. This may be a pragmatic solution but is
  discouraged because:
  - Managing paths in the SCION layer is not trivial because it requires
    synchronizing the lifecycle of SCION paths and QUIC-MP paths, e.g.,
    knowing when is a path valid or when is it closed in QUIC-MP.
  - It creates opportunities for memory exhaustion attacks
    (for storing the mapping of mangled IP/port).
  - It reports a wrong IP/port to the application.


## Interoperability of QUIC-MP Path ID and Network Paths {#pathid}

The identification of "paths" varies between QUIC, QUIC-MP and SCION.

- {{QUIC-TRANSPORT}} uses a 4-tuple of local/remote IP/port to
  distinguish paths.
- {{QUIC-MP}} extends the 4-tuple with a path ID to distinguish
  logical paths (connections).
- SCION can distinguish paths based on the physical inter-domain
  network path and additional properties, such as an expiration
  time (the latter may or may not be used to distinguish path
  instances).

A path change occurs when at least one of the router interfaces
changes. The network address may stay the same.

With NAT rebinding, as described in {{Section 5.2 of QUIC-MP}},
the path can change, but not without changing the SCION network
address (IP, port, AS), so this case is not a concern.

Path change detection is required to trigger certain actions,
such as resetting congestion control or RTT estimation algorithms.
See also {{concon}} and {{rtt}}.
When using a PAN such as SCION, it is important to trigger these
actions even if the full network address (4-tuple + AS) stays the same.

Alternatively, the system can be implemented in a way so that
uncontrolled path changes cannot occur.
This is possible because path changes can only be initiated by
endpoints. However, this has some limitation if one of the
endpoints is not aware of transporting QUIC, for example a SCION
gateway or proxy, see {{sig}}.

Implementations should try to maintain a 1:1 mapping between QUIC-MP
path IDs and SCION network paths.
However, this is not always possible or useful:

- A SCION network path may expire. Replacing a path with an
  identical new path (except for the expiration date), should be allowed
  without triggering algorithm reset. Alternatively, refresh
  can be handled by the path selector, see {{patsel}}.
- A SCION implementation, when used with QUIC-MP should be configured
  such that every SCION network path is used for exactly one QUIC-MP
  path ID.
  However, it may not always be possible or feasible to configure SCION
  implementations in this way, for example. when they are part of a SCION
  gateway or proxy, and are unaware of transporting QUIC, see {{sig}}.
- A SCION path should be allowed to be reused, e.g., they may be assigned
  to one QUIC path ID, and when that path ID is closed, the path
  must be allowed to be assigned to another path ID.
  This should not cause any problems except for the
  marginal complexity of managing the associated state with a path ID.

### Key Implications

If a path change occurs undetected, the QUIC-MP layer cannot
reset congestion control ({{concon}}) or RTT estimation ({{rtt}}).
This is undesirable but not worse than traditional IP based non-PAN
transport where routes can change without the endpoints learning about
it.

### Recommendations

- Congestion control and RTT estimation algorithms should be
  designed to gracefully handle path changes that don't trigger a
  reset, unless it can be guaranteed that both SCION endpoints are
  configured to prevent automatic path changes.
- Within a QUIC-MP session, every SCION network path should be used only
  with one path ID. However, it may be reused if the path was abandoned
  or closed.
- Changes of the network path (while the network address stays the
  same), except for expired paths being renewed, should trigger
  algorithm reset (CC, RTT estimate), see {{Section 5.1 of QUIC-MP}}.

Analogous to {{endpoint-identity}}, except for replacing "AS" with
"network path". We list them here again because the implications
of not following the recommendation are much weaker and may be
considered acceptable. Recommendations:

- Adapt the QUIC-MP library to be aware of the full network path,
  including router interfaces.
- If the network address is available as a single "object",
  the SCION layer can extend this with the network path (possibly
  excluding the expiration date), and the
  QUIC-MP implementation must only ensure to compare the whole
  object instead of port and IP separately.
- The SCION implementation could detect cases where only the router
  interfaces
  change and then mangle the port or IP to trigger a path validation
  in the QUIC-MP layer. This may be a pragmatic solution but is
  discouraged, because:
  - Managing paths in the SCION layer is not trivial because it requires
    synchronizing the lifecycle of SCION paths and QUIC-MP paths, e.g.,
    knowing when is a path valid or when is it closed in QUIC-MP.
  - It creates opportunities for memory exhaustion attacks
    (for storing the mapping of mangled IP/port).
  - It reports a wrong IP/port to the application.


## Initial Handshake {#handshake}

{{QUIC-TRANSPORT}} requires that there is no connection migration
during the initial handshake, and that there are no other packets
sent (including probing packets) during the initial handshake, see
{{Section 9 of QUIC-TRANSPORT}}, paragraphs 2 and 3.

### Key Implications

Changing the path during handshake would violate
{{Section 9 of QUIC-TRANSPORT}}:

> The design of QUIC relies on endpoints retaining a stable address
> for the duration of the handshake. An endpoint MUST NOT initiate
> connection migration before the handshake is confirmed, as defined in
> {{Section 4.1.2 of QUIC-TLS}}.

### Recommendations

- A SCION implementation should not automatically change network
  paths switch without explicit request by the QUIC(-MP) layer.
  The only exception allowed is replacing an expiring path with
  a new path that is identical except for the expiration time.
  We also need to ensure this for gateways etc., see {{sig}}.

## Congestion Control {#concon}

Following {{Section 5.1 of QUIC-MP}}, the CC algorithm should be reset
when the 4-tuple of a QUIC path changes. With SCION, 4-tuples are not
sufficient to identify paths, see {{pathid}}.

To avoid missing a path change, the SCION implementation should never
change a network path unless explicitly instructed by the QUIC-MP
implementation, see {{recommendations}}.


### Coupled Congestion Control

{{Section 5.3 of QUIC-MP}} mentions coupled congestion control
algorithms, such as {{CC-MULTIPATH-TCP}}. {{CC-MULTIPATH-TCP}} states:

> One of the prominent
   problems is that running existing algorithms such as standard TCP
   independently on each path would give the multipath flow more than
   its fair share at a bottleneck link traversed by more than one of its
   subflows.

This can be avoided in PANs, such as SCION, through link-level analysis
of paths and selecting paths that do not share a bottleneck link.
Instead, this bottleneck knowledge can be used to effectively use
separate congestion control for each path.
Alternatively, a CC algorithm could be employed that focuses
on known shared links (which may be bottlenecks).

There are several congestion control algorithms proposed in literature,
e.g., LIA, OLIA, BALIA and RSF.  These combine congestion control with
path selection algorithms.
For simplicity, we suggest separating concerns in terms of
congestion control and path selection. This allows us to better
tailor the solutions to the different usage scenarios.

The proposition is to use non-coupled congestion control per path,
tailored for each use case in {{categories}}, and use separate
independent path selection algorithms.

CC algorithms can also benefit from the SCION metadata extension that
provides bandwidth and latency data for each link on a
network path.


### Key Implications

A network path change goes unnoticed in case
a SCION implementation changes a path that happens to have the same
IP/port for both endpoints.

Congestion control (CC) algorithms can also benefit from exact knowledge
of a path:

- When using multiple paths, a CC algorithm can access path metadata
  as to if and where the paths overlap and some of the properties of the
  overlapping sections.

- CC algorithms should be notified of every path change, allowing them
  to reset only when necessary. A reset may not be necessary if the
  network path remains the same and only the IP or port of an endpoint
  changes. This can make sense if any congestion is assumed to be on the
  network path rather than behind the remote IP/port (e.g., behind a proxy).

See also {{Section 5.3 of QUIC-MP}}.

### Recommendations

- Congestion control algorithms should be reset when the network path
  changes (beyond 4-tuple). This is best achieved by ensuring that
  the network path changes only in conjunction with QUIC path migration
  events.

Congestion control algorithms can also benefit from exact
knowledge of a network path:

- Congestion control algorithms should use the path metadata to
  detect and, if necessary and possible, avoid overlap with
  other paths. Congestion control can then be simplified to work
  independently for each path.
- Path selection algorithms should try to avoid multiple path
  that share bottleneck links.


## RTT Estimation {#rtt}

Similarly to congestion control, and following
{{Section 5.1 of QUIC-MP}}, the RTT estimation algorithm should be
reset when the 4-tuple of a QUIC path changes.
As described in {{concon}} this can be avoided
by forbidding SCION implementations to change a network path unless
instructed otherwise by the QUIC-MP implementation.

### Key Implications

If a path change occurs undetected, the QUIC-MP layer may fail to
reset RTT estimation.
This is undesirable but not worse than traditional IP based non-PAN
transport where routes can change without the endpoints ever
learning about it.

### Recommendations

- RTT-algorithms should be reset when the network path
  changes (beyond 4-tuple). This is best achieved by ensuring that
  the network path changes only when requested by QUIC.

Round-trip time estimation algorithms can also benefit from exact
knowledge of a path:

- An implementation may use SCION's SCMP traceroute
  ({{Section 6 of SCION-CP}}) to measure the latency of individual
  links and then use this information to select new network paths
  that favor low latency links and avoid high latency links. See also
  {{bottleneck}}.

- An implementation could use the SCION metadata extension to
  get propagation latency information of links in a path without
  having to measure it.
  This latency does not include queueing latency but may in
  many cases be sufficient for practical use.

## MTU Discovery {#mtu}

The MTU may be used to calculate the available payload size.
SCION inserts an additional header ({{Section 2 of SCION-DP}}) into
each packet. The header size depends on the IP family (e.g., IPv4 vs
IPv6 addresses) and on the "length" of the path, i.e., the number of
ASes that are traversed. This must be taken into account when
calculating the available payload size.

The difference between typical MTU (1500 bytes) and QUIC's required packet
size (1200 bytes) is sufficient for typical real-world SCION headers.

PMTU discovery ({{Section 14.3 of QUIC-TRANSPORT}}, {{MTU-DISCOVERY}})
can be used to discover or verify MTU sizes.
However, the MTU from the path metadata can (at least on the client
side) be used to preselect paths with desirable MTU values.

In SCION, when an endpoint requests a network path, it will be
provided with MTU information for every hop on a path, see also {{mtu}}
and {{Section 4.4 of SCION-CP}}. However, in SCION, paths are
typically only requested by client endpoints, not by server endpoints.

There are several ways for a server to determine the MTU.
If a server wants to know the MTU, it may:

- Try to determine the MTU from the size of incoming packets.
- Use an algorithm to determine the MTU, see Path MTU Discovery in
  {{Section 14.3 of QUIC-TRANSPORT}} and {{Section 5.8 of QUIC-MP}}.
- Try to look up the path to the client endpoint that is identical to
  the incoming path. However, this requires time
  and effort on the server side, and there is no guarantee that
  the incoming path is available in the local AS.

Also, note that the MTU information is authenticated but not verified,
it may be incorrect due to misconfiguration or malicious ASes.

### Key Implications and Recommendations

PMTU discovery for multi-path may be improved by using path metadata.
PMTU will be explored more in detail in a future version of this
document (**TODO**)).

It may be possible for a client to send the PMTU size directly to a
server, for example as a parameter via the QUIC Transport Parameter
Extension, see {{Section 18 of QUIC-TRANSPORT}} and
{{Section 8.2 of QUIC-TLS}}.

## Retransmission & PTO {#retransmission}

See {{Section 5.6 of QUIC-MP}} and {{Section 5.7 of QUIC-MP}}.
For retransmission, a SCION client or server may compare available
paths and choose one or more paths that have minimum overlap
with the current (unreliable) path.


## Paths Having Different PMTU Sizes

{{Section 5.8 of QUIC-MP}} suggests determining a single MTU size
in order to simplify retransmission.

### Key Implications

{{Section 5.8 of QUIC-MP}} explains that the benefit of using a
single MTU size is
to simplify retransmission processing, as the content of lost packets
initially sent on one path can be sent on another path without
further frame scheduling adaptations.

### Recommendations

- On the client, this can be facilitated by computing a viable minimum
  MTU size from all available network paths. However, it must be
  considered that these values are not verified.
- On the server, MTU values from path metadata are not available.
  The server may request these from the local AS, but the exact
  path may not be available (in SCION, different ASes may offer
  different sets of paths to their customers).
  Also, except for initially proposing a preferred address
  ({{Section 9.6 of QUIC-TRANSPORT}}), new paths must be opened by
  the client, not the server, see {{Section 9 of QUIC-TRANSPORT}}.


## Path Selection {#patsel}

The path selection component is responsible for requesting paths
to a destination, ordering the path based on policy and preferences,
using them when new QUIC-paths are opened, and retiring them or listing
them for reuse when they are closed.

### Dynamic Approach

A dynamic approach could start with using low latency paths. If the
connection appears to be long lasting, it could start (and keep)
adding additional paths as long as the traffic increases.

As an example, if the algorithm detects traffic that lasts for
at least one second and transfers at least 100MB of traffic,
the algorithm could trigger creation of additional QUIC paths.

### Bottleneck Detection {#bottleneck}

If live traffic information is not available, bottleneck detection
can help to identify links that should be avoided. In PANs,
this can be achieved using approaches such as {{UMCC}}.

An alternative is to use SCION's SCMP `traceroute` command
({{Section 6 of SCION-CP}})
to measure the latency between two consecutive AS border routers.
The measured latency can be compared to earlier measurements or to
the latency given in the path metadata. Discrepancies can be an
indication of high traffic volume and queueing on the measured link.

While traceroute may be useful, it should be used with care:

- traceroute traffic is not congestion controlled.
- It is clearly distinguishable from QUIC traffic, so it
  may affect anonymity.


### Key Implications

Path selection is a key feature of SCION and PANs in general.
For more details, see {{SCION-CP}} and {{SCION-DP}}.

### Recommendations

In order to manage paths effectively, the path selection algorithm
probably requires access to the following fields and events:

- `initial_max_path_id` ({{Section 2.1 of QUIC-MP}})
- MAX_PATH_ID frames ({{Section 4.6 of QUIC-MP}}
- PATH_AVAILABLE and PATH_BACKUP, see {{Section 3.3 of QUIC-MP}},
- PATH_ABANDON {{Section 3.4 of QUIC-MP}}.

Moreover, path selection must exclude paths whose MTU is too
small to guarantee 1200 bytes MTU payload for QUIC packets.
The effective MTU also depends on the length of the paths.


## Packet Scheduling {#scheduling}

Packet scheduling helps to distribute the transfer load efficiently
over multiple paths, see also {{Section 5.5 of QUIC-MP}}.


### Key Implications

SCION paths are stable, at least on the inter-AS level,
i.e., they cannot change without initiative from the endpoints.

### Recommendations

Path stability may simplify packet scheduling algorithms because the
performance of individual QUIC-paths is more reliable if they
cannot unexpectedly be rerouted.

## Address Validation Token {#token}

{{Section 9.3 of QUIC-TRANSPORT}} specifies that a server is
expected to send a new address validation token to a client
following the successful validation of a new client address.
{{Section 8.1.4 of QUIC-TRANSPORT}} states:

 - "Tokens sent in Retry packets SHOULD include information that
allows the server to verify that the source IP address and port
in client packets remain constant."

 - "Tokens sent in NEW_TOKEN frames MUST include information that
allows the server to verify that the client IP address has not
changed from when the token was issued."

These must be read such that "IP address" becomes "network address",
meaning that it includes the ISD/AS code of a peer.


## Path ID Allocation {#path-id-allocation}

SCION application may use many paths. This poses two problems:
initial path ID allocation and incremental path ID allocation.
See also {{Section 4.6 of QUIC-MP}} and {{Section 4.7 of QUIC-MP}}.

When initializing a connection, servers and clients should be careful to
set the maximum path ID high enough to allow as many paths as an application
may need, e.g., through the initial_max_path_id transport parameter or the
MAX_PATH_ID frame.

Afterwards, on a bad connection, paths IDs may still run out earlier than
expected, so servers should allocate new path IDs either automatically or
on demand when receiving a PATHS_BLOCKED frame.

### Key Implications

SCION path IDs may run out quickly unless the server provides enough
initial path IDs and provides enough path IDs once the initial contingent
runs out.

### Recommendations

- Ensure that servers and clients are prepared to agree on an initial
  number of path IDs that is high enough for the user application.
  This should be configurable.
- Ensure that either the server proactively allocates new path IDs
  (MAX_PATH_ID frame) when they are running out, or that the server
  allocates sufficiently many new path IDs once the client sends a
  PATHS_BLOCKED frame.
- Optional: Allow the client to send PATHS_BLOCKED proactively even before
  the path IDs are exhausted. Prepare the server to accept early
  PATHS_BLOCKED frames and respond by allocating a gracious amount of
  additional path IDs.


# Summary of Recommendations {#all-recommendations}

This memo is informational. However, we use {{!RFC2119}}
imperative language here for recommendations that are
relevant to security or performance.


## Recommendations for QUIC-MP Implementations

- To prevent attackers circumventing path validation, a QUIC-MP
  implementation MUST ensure to trigger path validation when the
  network address of the destination changes; this includes
  IP, port and AS number. This protects against several attacks,
  see {{attack-path-injection}} and especially
  {{attack-amplification}}.

  There are several ways to achieve this, for example:

  - Adapt the QUIC-MP library to be aware of the AS number in SCION
    network addresses.
  - If the network address is available as a single "object",
    the SCION layer can extend this with the AS code, and the
    QUIC-MP implementation must only ensure to compare the whole
    object instead of port and IP separately.
  - The SCION implementation could detect cases where only the AS
    changes and then mangle the port or IP to trigger a path validation
    in the QUIC-MP layer. This may be a pragmatic solution but is
    discouraged, because:
    - Managing paths in the SCION layer is not trivial because it requires
      synchronizing the lifecycle of SCION paths and QUIC-MP paths, e.g.,
      knowing when is a path valid or when is it closed in QUIC-MP.
    - It creates opportunities for memory exhaustion attacks
      (for storing the mapping of mangled IP/port).
    - It reports a wrong IP/port to the application.

- A QUIC-MP implementations SHOULD be able to recognize network
  path changes beyond 4-tuple or AS changes. This enables resetting
  congestion control and RTT algorithms.

- A QUIC-MP implementation, especially congestion control, RTT estimation,
  and path selection algorithms should be careful to not accept or probe
  all available SCION network paths. There may be a large number of path
  available and probing or using them all may amount to a denial of service
  attack on the receiver.

- To prevent problems with path ID exhaustion with multi-path applications,
  see {{path-id-allocation}}:
  - Servers and clients should initialize the connection with
    sufficiently many path IDs.
  - Servers should proactively allocate new path IDs when they run low.
  - Clients should proactively send PATHS_BLOCKED when path IDs run low,
    Servers should accept and react to PATHS_BLOCKED before path IDs are
    actually exhausted.


## Recommendations for SCION Implementations

- SCION implementations can prevent attackers from expoloiting a skipped
  path validation by preventing path injection ({{attack-path-injection}}).
  To prevent path injection:

  1) Border routers MUST verify that any packet whose path indicates origin
     outside of the local AS, the underlay IP must match a known border router
     IP.
  2) Border routers MUST verify that for any packet that originates in the
     local AS, the underlay IP matches the SCION SRC IP.
  3) SCION server libraries MUST respond to the underlay IP from which a
     request was received.

- A SCION implementation SHOULD NOT store or cache paths,
  especially not on the server side. This prevents memory
  exhaustion attacks, see {attack-memory-exhaustion}.
  This also avoids the problem of path lifecycle maintenance, i.e.,
  determining which paths are still alive and which have been closed
  or abandoned.
  Sometimes, storing paths is inevitable, see {{sig}}.
  For security concerns, see also {{attack-path-injection}}.

- If a SCION implementation stores paths internally, it must be careful
  to avoid using IP/port as key to look up paths. IP/port are not unique
  to identify endpoints.
- When used with QUIC-MP, a SCION implementation MUST NOT change the
  network paths, possibly with the exception of refreshing expired
  paths.
  When a path stops working, the implementation should instead report
  an error to the QUIC(-MP) layer or time out silently.


## Recommendations For Both QUIC-MP And SCION Implementations

- A server should return packets on the same path on which they were
  received.
  - Generally, a server SHOULD respond on the same path on which the data
    was originally requested, unless the new path has been validated.
    This ensures that the path does not violate the path policy of the
    client.
  - A server SHOULD NOT create new paths. This is also stated in
    {{Section 9 of QUIC-TRANSPORT}}. Servers may communicate a preferred
    address after the initial handshake. However, it is recommended to
    avoid that because any new path may violate a client's path policy.
  - Returning probing packets on the same network path on which they
    were received: This greatly simplifies RTT estimation, see {{rtt}}.

- Clients may replace expired or soon-to-expire paths with identical
  paths without performing path migration / validation.

- Within a QUIC-MP session, every SCION network path should be used only
  with one path ID. However, it may be reused if the path was abandoned
  or closed by QUIC. This is the responsibility of the path selection
  algorithm, regardless of whether it is considered part of SCION or
  part of QUIC-MP.


## Other Recommendations
 - Servers and client should aim to be located in respective local networks
   (local ASes) that have protection against IP spoofing from machines in the
   same network, see {{attack-path-injection}}.


# Security Considerations

The aim is that QUIC-MP over SCION retains all security
properties of QUIC-MP and SCION. However, this requires some
implementation changes and additional consideration regarding:

- endpoint identity: a 4-tuple is not sufficient to identify an endhost;

- network path authenticity: paths may be forged by malicious clients;

- path probing patterns may expose user intentions or identity.


## IP Ambiguity {#attack-ip-ambiguity}

In SCION, IP addresses are not sufficient to uniquely identify a peer endpoint.
ASes are free to use IP addresses from "private" IP ranges.


To attack a client with IP address X, an attacker could set up an endpoint
with identical IP X in a different AS. The attacker can then contact a server
endpoint that is also used by the client victim.

If the server endpoint stores path internally with IP addresses as keys,
then this would result in a key collision, which can cause two types of problems:

- If the attacker contacts the server after the client, the contact may result
  in overwriting the existing entry (of the victim).
- If overwriting is prevented, the attacker may contact the server before the victim,
  thus potentially preventing the victim from establishing a connection,

The second case is more difficult to achieve. Often, connections may be store by
IP+port, so the attacker must guess the victims port when launching the attack.

### Mitigation

- A SCION implementation should avoid storing paths while using
  the IP address as key to look up paths.
  For QUIC-MP, the best solution is to use the QUIC Path ID as key.
- SCION libraries could us port/IP mangling when they detect multiple paths with
  the same IP/port. However, this may have unintended consequences in the
  application layer.
- Higher level libraries, such as QUIC(-MP) should be carefuol to not
  rely only on IP addresses to trigger path validation or resetting
  congestion control or RTT estimation algorithms. Instead, QUIC-MP should
  rely on the QUIC Path ID.

Circumventing path validation is not possible, because that would require injecting
a new path, see {{attack-path-injection}}.


## Path Injection {#attack-path-injection}

There are several potential attacks that are based on injecting
valid or invalid paths into the server-side software stack.
Injecting a path means sending a packet with an incorrect path to an endpoint
such that the endpoint sends a response along that path. The path is incorrect
insofar as it does not accurately describe the route of the original request packet.

In summary, these attacks can be prevented by the recommendations
listed in {{all-recommendations}}. The following scenarios are considered:

1. The SCION SRC address is wrong. I.e., the packet originates in the correct AS
   but from a different IP than what is announced in the SCION SRC address field.

   To avoid this, border routers MUST drop egress packets whose path indicates
   a local origin, but whose SCION SRC address does not match the underlay
   SRC address.

2. The path is too long. The packet originates in an AS that is different from the
   first AS announced in the path.

   To avoid this, a border router MUST drop egress packets whose paths indicate
   AS-external origin but whose underlay address does not
   match that of a known border router.

3. The path is not empty. This is an extreme case of the previous example. The
   packet originates in the destination AS and is sent directly to the endpoint.
   In this case the path should be empty.

   To avoid this, on a server, the SCION layer MUST always respond to the same
   underlay address from which a request was received.
   This is anyway useful for servers because responding to the same address also
   avoids the need for the server to look up the first hop when sending a response.

An attacker can inject paths into a server only in the following circumstances:
- The attacker controls the border routers of an AS that is between a victim and
  a server.
- The attacker can spoof the address of a border router in an AS that is between
  the victim and a server.
- The attacker can spoof the address of a victim while being located inside the
  victims AS.


### Traffic Redirection over Different Path

An attacker may craft a path with a network address that is identical
to an existing valid endpoint, but with a different path.

The new route may be invalid (e.g., contain nonexistent links)
or faulty (contain links that are broken or have high latency
or drop rate).

The new route may also work fine, but violate the client's path policy
or be used for traffic analysis.

The attacker injects the crafted path into the server, with the intent that the
non-unique IP causes an existing path/connection mapping to be overwritten, and
thus replace the victims path with the updated path.


~~~~
     AS #100               AS #200                   AS #300
  +-----------------+        +--------------+        +--------------+
  | Server          |        | Attacker     |        | Victim       |
  | IP=198.51.100.1 | ------ | IP=192.0.2.1 | ------ | IP=192.0.2.1 |
  | port=42         |        | port=12345   |        | port=12345   |
  +-----------------+        +--------------+        +--------------+

~~~~
{: #fig-example-non-unique-ip title="Example of non-unique IPs"}

Mitigation:

This attack requires either spoofing of the client's IP address
(when the attacker is in the same AS as the client),
spoofing of a border router's IP address (when the attacker
is in the server's AS), or injection of a path (which requires
control over an AS that is en-route between the client and server).

This can be further mitigated by the recommendation that path validation
should always be triggered when the network address or path
changes, even if the 4-tuple stays the same.


### Traffic Amplification {#attack-amplification}

An attacker may establish a connection with a server,
request a large amount of data, and then inject a path that
redirects to a victim that has the same IP/port, but in a different AS.

If the server-side QUIC-MP does not trigger path validation
(because IP/port are the same), then it may implicitly accept
the new path and send the requested data to a victim.


Mitigation:

This attack requires either spoofing of the client's IP address
(when the attacker is in the same AS as the client),
spoofing of a border router's IP address (when the attacker
is in the server's AS), or injection of a path (which requires
control over an AS that is en-route between the client and server).
- A QUIC(-MP) library must consider all attributes
  (not just the 4-tuple) when checking for a change in the network
  address. This would then trigger path validation, and the attack
  can be averted.
- If a QUIC(-MP) library cannot compare additional attributes
  (e.g., legacy library), the SCION layer (server side) should have an
  option to perform port mangling or IP mangling: when the SCION layer
  detects a new network address that differs only in the AS number
  from a previously seen address (IP/port are the same), then it
  should perform IP/port mangling, i.e., reporting a modified IP or
  port to the QUIC(-MP) layer. This new IP/port would trigger a path
  validation or algorithm reset where required.

Caveats:

- Offering a mangled IP/port to the application may have implications
  for application correctness, such as displaying an unexpected IP/port.


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




## Memory Exhaustion {#attack-memory-exhaustion}

An attacker may flood a server with packets that each have a
different source network address or path. If these are stored in the
SCION layer, they may cause memory exhaustion.

Mitigation: do not store state in the SCION layer, or implement
a way to clean up state without affecting a valid connection.
Specifically, the SCION layer must drop a path if the QUIC layer
closes the path, abandons the path, or rejects a packet with a new path.

## Number of Open Paths {#security-many-paths}

The number of open paths should be limited, see
{{Section 7.2 of QUIC-MP}}.
This is important in the context of applications that may open many
paths in parallel.

Mitigation:

- Same as {{Section 7.2 of QUIC-MP}}: endpoints
   need to limit the maximum number of paths and might consider
   additional measures to limit the number of concurrent path validation
   processes, e.g., by pacing them out or limiting the number of path
   initiation attempts over a certain time period.


## Probe Fingerprinting

An endpoint may probe multiple paths to determine the best
path(s) for a given use case. One example of probing packets is
packets that measure round-trip time (RTT).

Probing packets may be detected if they are sent in bulk, to the
same destination, in regular intervals, and all with slightly
different paths attached.

This can be used to fingerprint an endpoint or their intentions
(applications may have unique intervals defined).

This can be mitigated by varying and generally reducing the
number of probing packets, and by sending probing packets
not en-block but time-shifted.


## Additional Points

TODO: Complete this section in a future version of this document.

- Use multipathing for anonymity, see {{categories}}.
- See other attacks in {{Section 7.2.4 of SCION-CP}}?


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

Thanks to the Path Aware Networking Research Group for discussion
and feedback. Specifically, we would like to thank Kevin Meynell and
Nicola Rustignoli from the Scion Association for their valuable input
on several iterations of this document.


# Change Log
{:numbered="false"}

Changes made to drafts since initial submission. This section is to be removed before publication.

## draft-zaeschke-scion-quic-multipath-01
{:numbered="false"}

Major changes:

- "Path Injection": clarify that path injection in SCION is, depending on the situation, either
  impossible or only marginally useful for an attacker.
  Rewrite sections {{endpoint-identity}} and {{attack-path-injection}}.
- New section "IP Ambiguity" in security considerations: {{attack-ip-ambiguity}}.
- Completed section on Address Validation Token, see {{token}}.
- Added recommendation to avoid using all SCION paths.
- Added section {{path-id-allocation}} on path ID allocation and exhaustion.


Minor changes:

- Nits and wording improvements
- Reviewed use of normative language
- Added Changelog
- Moved memory exhaustion attack into own section.

## draft-zaeschke-scion-quic-multipath-00
{:numbered="false"}

Initial submission.
