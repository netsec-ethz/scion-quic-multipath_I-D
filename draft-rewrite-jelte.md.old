---
title: "Guidelines for QUIC Multipath over SCION"

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
Multipath Extension for QUIC {{QUIC-MP}} with the SCION
networking technology ({{SCION-CP}}, {{SCION-DP}}).

SCION is an inter-domain routing protocol that supports path-aware
multi-path networking.
The multiple path and their associated path information offered
by SCION provide opportunities as well as challenges for
combining QUIC-MP with SCION.

This document explores various aspects of this combination, such as
algorithms for congestion control, RTT estimation, or general application
scenarios.
In addition it provides techniques and guidance to maintain security
and to leverage path-aware multi-path networking with QUIC-MP.


--- middle

# Introduction

Multipath transport protocols such as the Multipath Extension for QUIC
{{QUIC-MP}} aim to increase performance, reliability, and resilience by
enabling endpoints to use multiple network paths simultaneously.
These capabilities are becoming increasingly relevant as applications
demand more flexible transport characteristics.

However, QUIC-MP has been designed with traditional IP-based routing in
mind, where path changes are typically inferred from endpoint address
changes (i.e., 4-tuples), and where routing is opaque to the transport
layer. In contrast, Path-Aware Networks (PANs), network architectures
where endpoints can discover and select paths with explicit control and
visibility, offer powerful capabilities that go well beyond those of
traditional IP-based routing. PANs can provide implicit metadata about
network paths, such as the hop count or path disjointness, with some
PANs also providing explicit metadata such as the supported bandwidth
or latency per path segment. These characteristics enable informed path
selection based on performance, policy or security requirements.
Examples of PAN architectures include SCION, RINA, and systems
leveraging segment routing.

Despite their complementary goals, QUIC-MP and PANs have evolved
largely in isolation. This represents a missed opportunity: the rich
semantics and control offered by PANs could directly enhance the
behavior of multipath transports; improving congestion control,
improving performance, enabling faster recovery from failures, and
enforcing routing policies.

This document provides guidance for using QUIC-MP over PANs, with a
focus on how to adapt to and leverage path-aware features. We consider
implementation and deployment issues, security implications, and usage
scenarios, and offer concrete recommendations to improve
interoperability and efficiency.

While this guidance applies to a broad class of PANs, we use SCION
({{SCION-CP}}, {{SCION-DP}}) as the primary reference architecture due
to its maturity, rich path semantics, and wide deployment in academic
and production environments.

**TODO** an explicit "this document is intended for ..."

We explicitly do not prescribe new standards, APIs, or algorithms;
rather, we aim to inform implementers and practitioners by highlighting
key considerations and offering pragmatic recommendations.

# Terminology and Conventions

{::boilerplate bcp14-tagged}

## Terminology {#terms}

This document uses terms from the QUIC base specifications
{{QUIC-TRANSPORT}}, the Multipath Extension for QUIC {{QUIC-MP}},
the Path-Aware Networking Vocabulary {{PATH-VOCABULARY}}.
For ease of reference, we define or recap key terms below, but refer
the reader to the aforementioned references for complete specifications
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
In PANs, however, network addresses may carry architecture-specific
semantics and structure that go beyond IP tuples:
- In IPv6 Segment Routing, the network address remains an IPv6 address
- In SCION, the network address consists of an _ISD-AS identifier_ and
a _host address_, typically expressed as `[ISD-AS, Host]`.
-  In other PANs, such as RINA, addressing may be based on naming
within distributed layers, further decoupling endpoint identity from
network topology.

**Network Path**: The network path is the sequence of network elements
between the two endpoints (e.g., ASes, interfaces, internal links).
In traditional IP networks, the network path is typically opaque to
the endpoints.

**QUIC-MP Path**: Consists of the network address at each
endpoint and a Path ID (see {{QUIC-MP}}). The Path ID allows having
multiple logical paths for the same set of network addresses.

**Path Metadata**: Path metadata refers to information about the
properties or characteristics of a network path, used by endpoints to
inform path selection, scheduling, congestion control, and validation.

In PANs, path metadata can be:
- Implicit: Derived from the structure of the path itself. For example,
the number of AS hops or the presence of shared segments between paths.
- Explicit: Provided by the network’s control plane or path service.
This includes explicitly advertised values such as per-link latency,
available bandwidth and MTU.

Examples:
- In SCION, explicit metadata can be included via Path Construction
Beacon extensions that report bandwidth and latency, while hop count
and interface information are implicitly encoded in the path. Such an
extension is typically authenticated by the AS that provides the
metadata.
- In Segment Routing, some metadata (e.g., segment sequence) is
implicit in the segment list, but additional performance data may be
inferred or estimated externally.

# Overview of QUIC Multipath in Path-Aware Networks
The Multipath Extension for QUIC (QUIC-MP) enables endpoints to
transmit and receive data over multiple paths simultaneously within a
single QUIC connection. In traditional IP networks, each path is
identified by a 4-tuple of local and remote IP addresses and ports.
These networks treat routing as opaque to endpoints, and path changes
are inferred indirectly through changes in the 4-tuple. However, the
path between two endpoints may change unpredictably due to routing
dynamics, which is not captured by the 4-tuple.

Path-Aware Networks (PANs), by contrast, offer a more expressive and
transparent model of network connectivity. In PANs, endpoints can
discover and select explicit network paths, which are described at a
different semantic level, for example as a sequence of IP addresses,
ASes or interfaces. Another key differentiator in PANs is the
availability of implicit and explicit metadata.

These features allow endpoints to make informed transport decisions,
but they also conflict with assumptions embedded in current multipath
transport designs.

For instance:
- **Endpoint Ambiguity**: PANs such as SCION use composite addresses
(e.g., ISD-AS + Host), which break the assumption that IP addresses
are globally unique.
- **Path Identity Mismatch**: In QUIC-MP, paths are distinguished by
4-tuples and Path IDs. In PANs, two distinct physical paths may share
the same 4-tuple, rendering transport-layer path tracking incomplete.
- **Routing and Path Lifecycle**: Paths in PANs can expire, be revoked,
or be reissued, even when 4-tuple information is unchanged.
This can interfere with RTT estimation, congestion control, and path
validation logic if not properly accounted for.

Despite these mismatches, PANs open up new opportunities for improving
multipath transport:
- **Explicit path selection** enables endpoints to choose disjoint
paths, i.e. paths that do not share links or segments, to improve fault
tolerance against link failures, or to increase aggregate throughput.
- **Implicit and explicit metadata**, such as hop counts and latency
information, allows endpoints to prioritize paths with more suitable
properties. For instance, low-latency and low-hop-count paths can be
prioritized for early RTT measurements, avoiding wasting resources on
probing paths with poorer characteristics.
- **Path-awareness** in PANs allows congestion control and RTT
estimators to reset only when the underlying network path has actually
changed, something not reliably detectable in traditional IP networks,
where network path changes may occur due to routing, despite a constant
4-tuple.
- The availability of **stable network paths** in PANs allows the
transport layer to distinguish between actual path changes and
transient network conditions, enabling more accurate RTT estimation
and congestion control.

The integration of QUIC-MP with PANs introduces both challenges and
new capabilities. To better understand where these issues arise, and
how path-awareness can be leveraged in practice, we next consider a
range of multipath use cases. These scenarios illustrate the diversity
of goals that multipath transport seeks to address, and highlight how
PANs can support these goals through their path semantics, metadata
availability, and routing models.


#  Multipath Use Cases and Categorization
This section categorizes common use cases for multipath transport and
highlights how PANs, such as SCION, can enhance each scenario. Many of
these use cases can be combined to meet complex application
requirements.

## Fault Tolerance and Availability
In traditional IP networks, an endpoint typically has access to only
a single active path to a given destination. If that path fails—due to
a link or router outage—the network must first detect the failure,
recompute a new route, and update routing tables (possibly across
multiple routers) before traffic can resume. This convergence process
can take anywhere from hundreds of milliseconds to several hours,
during which the connection is disrupted.

Multipath transport can improve robustness against such failures.
By sending redundant traffic over multiple paths, an application can
maintain continuity even if one path becomes unavailable.

PANs further enhance the fault tolerance of multipath transport by
enabling the selection of disjoint paths. This reduces the likelihood
of multiple paths simultaneously failing. Moreover, because endpoints
in PANs have access to multiple explicit, usable paths, traffic can be
switched immediately to an alternate path upon detecting a failure,
avoiding delays associated with routing convergence in traditional IP
networks.


## High Throughput
Some applications aim to maximize available bandwidth by spreading
traffic across multiple paths. In PANs, implicit path metadata can be
used to identify disjoint paths. Using disjoint paths helps reduce the
risk of shared bottlenecks and can increase throughput. Furthermore,
when congestion is detected, PANs can switch over entirely, or shift
parts of traffic to an underutilized disjoint path to preserve the
throughput.

PANs may also offer explicit bandwidth metadata (e.g., in SCION),
allowing the sender to choose paths with high capacity.

## Low Latency
Latency-sensitive applications benefit from selecting the fastest
available path at any moment. In PANs, endpoints may estimate latency
from explicit metadata or infer it from probing. Because in PAN paths
are stable and explicitly selectable, the transport layer can maintain
multiple low-latency options in parallel, and either transmit in
parallel, or switch traffic to a different path once the RTT starts to
fluctuate.

For deadline sensitive applications, an algorithm as described in
{{DMTP}} may be useful.


## Policy-Driven Routing and Routing Constraints
Some applications or deployments may wish to avoid routing traffic
through certain ASes or jurisdictions, e.g. to reduce exposure to
surveillance, or enforce routing policies. PANs enable this by making
path selection explicit and verifiable.

## Anonymity and Traffic Obfuscation
Multipath transport can be used to reduce the observability of traffic
by distributing it across multiple network paths. In PANs, endpoints
can explicitly select disjoint paths to minimize the risk that a single
AS observes the full traffic flow.

Randomizing path selection and packet scheduling can help obscure
traffic patterns. However, traffic characteristics such as packet
timing, flow duration, or volume may still be linkable across paths.
Mechanisms, such as probing should therefore be designed and used such
that it avoids creating identifiable patterns.

## Summary

The availability of explicit paths in PANs significantly expands the
design space for multipath transport. While these capabilities
introduce complexity, they also make it possible to tailor path usage
to application needs with greater precision than is possible in IP
networks.

The next sections will explore how these use cases map to technical
requirements for congestion control, RTT estimation, path validation,
and scheduling — highlighting practical considerations for implementing
QUIC-MP over PANs.


# Technical Considerations for Multipath QUIC over Path-Aware Networks

## Adressing and Path Identity

### Key Implications

### Recommendations

## Interoperability of the QUIC-MP Path ID and the Network Paths

## Endpoint Identity

## Congestion Control

## RTT Estimation

## MTU Considerations

## Retransmission and Path Scheduling

## Path Lifecycle

## Maximum Concurrent Paths
A tradeoff here is that sending on all available paths may
be infeasible because of the number of available paths (with SCION we
often see 100+ paths to a destination).
Depending on cost factors, and to avoid overloading the network, any
algorithms should keep redundant sending to a minimum. See also
{{Section 7.2 of QUIC-MP}} for DoC security considerations.


# Security Considerations

# Implementation Recommendations
Unify and summarize all implementation advice

## For QUIC-MP

## For PANs

## For Combined Libraries



# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

Thanks to the Path Aware Networking Research Group for the discussion
and feedback. Specifically we would like to thank Kevin Meynell and
Nicola Rustignoli from the SCION Association for their valuable input
on mmany
iterations of this document.




# QUIC Implementation Considerations {#impcon}

This section provides guidance for implementors of SCION libraries
and QUIC-MP libraries.
Recommendations are summarized in {{recommendations}}.


## Automatic Path Changes - Initial Handshake

{{QUIC-TRANSPORT}} requires that there is no connection migration
during the initial handshake, and that there are no other packets
sent (including probing packets) during the initial handshake, see
{{Section 9 of QUIC-TRANSPORT}}, paragraphs 2 and 3.

An implementation must ensure at some level that no path change or
probing occurs.

This is covered by the recommendation that a SCION implementation should
not automatically switch without explicit request by the QUIC(-MP)
layer, see {{recommendations}}.

THere may be implicit path switching to an identical new path due to
refreshing expired paths, however, this can be safely ignored.


## MTU Detection {#mtu-con}

In SCION, when an endpoint requests a network path, it will be
provided with MTU information for every hop on a path, see also {{mtu}}
and {{Section 4.4 of SCION-CP}}.

However, in SCION, paths are typically only requested by client endpoints,
not by server endpoints.

There are several ways for a server to determine the MTU.
If a server wants to know the MTU, it may:

- Try to determine the MTU from incoming packets.
- Use an algorithm to determine the MTU, see Path MTU Discovery in
  {{Section 14.3 of QUIC-TRANSPORT}} and {{Section 5.8 of QUIC-MP}}.
- Try to look up the path to the client endpoint that is identical to
  the incoming path. This is not recommended because it requires time
  and effort on the server side, and there is no guarantee that
  the incoming path is available in the local AS.

Also, note that the MTU information is authenticated but not verified,
it may be incorrect due to misconfiguration or malicious ASes.


## Congestion Control {#concon}

Following {{Section 5.1 of QUIC-MP}}, CC algorithm should be reset when
the 4-tuple of a QUIC-path changes.
With SCION, 4-tuples are not sufficient to identify paths,
see {{path-identity}}.
To avoid missing a path change, SCION implementation should never change a
network path unless instructed otherwise by the QUIC-MP implementation,
see {{recommendations}}.
If this is not followed, a network path change may go unnoticed in case
a SCION implementation changes a path that happens to have the same IP/port
for both endpoints.

Congestion control (CC) algorithms can also benefit from exact knowledge
of a path:

- When using multiple paths, a CC algorithm can access information
  as to if and where the paths overlap and some of the properties of the
  overlapping sections.

- CC algorithms should be notified of every path change, allowing them
  to reset only when necessary. A reset may not be unnecessary if the
  network path remains the same and only the IP or port of an endpoint
  changes. This can make sense if any congestion is assumed to be on the
  network path rather than behind the remote IP/port.

See also {{Section 5.3 of QUIC-MP}}.


### Coupled Congestion Control

{{Section 5.3 of QUIC-MP}} mentions coupled congestion control
algorithms, such as {{CC-MULTIPATH-TCP}}. {{CC-MULTIPATH-TCP}} states that:

> "One of the prominent
   problems is that running existing algorithms such as standard TCP
   independently on each path would give the multipath flow more than
   its fair share at a bottleneck link traversed by more than one of its
   subflows.".

This can be avoided in PANs, such as SCION, through link-level
analysis of paths and selecting paths that do not share a bottleneck link.
Instead, this bottleneck knowledge can be used to effectively use
separate congestion control for each path.
Alternatively, a CC algorithm could be employed that focuses
on known shared shared links (which may be bottlenecks).

There are several congestion control algorithms proposed in literature,
e.g. LIA, OLIA, BALIA and RSF.  These combine congestion control with
path selection algorithms.
For simplicity, we suggest separating concerns in terms of
congestion control and path selection. This allows us to better
tailor the solutions to the different usage scenarios.

The proposition is to use non-coupled congestion control per path,
tailored for each use case {{categories}} and on top of that separately
use path selection algorithms.

CC algorithms can also benefit from the SCION metadata extension that
provide bandwidth and latency data for each node and link on a
network path.


## RTT Estimation {#rtt}

Similarily to congestion control, and following {{Section 5.1 of QUIC-MP}},
RTT estimation algorithm should be reset when the 4-tuple of a
QUIC-path changes. As described in {{concon}} this can be avoided
if SCION implementation should never change a network path unless instructed
otherwise by the QUIC-MP implementation, see {{recommendations}}.

Round-trip time estimation (RTT) algorithms can also benefit from exact
knowledge of a path:

- An implementation may use SCIONs SCMP traceroute
  ({{Section 6 of SCION-CP}}) to measure latency of individual links
  and then use this information to select new network paths that
  favour low latency links and avoid low latency links. See also
  {{bottleneck}}.

- An implementation could use the SCION metadata extension to
  get latency on links in a path without having to measure it.
  This latency may not by fully accurate, but may in many cases be
  "good enough".


## Retransmission & PTO {#retransmission}

See {{Section 5.6 of QUIC-MP}} and {{Section 5.7 of QUIC-MP}}.
For retransmission, a SCION client or server may compare available
paths and choose one or more paths that have minimum overlap
with the current (unreliable) path.


## Paths Having Different PMTU Sizes

{{Section 5.8 of QUIC-MP}} suggests determining a single MTU size
in order to simplify retransmission. At least on the client,
this can be facilitated by computing a viable minimum MTU size from all
available network paths.
However, these MTU values are not available on the server, and
it may be incorrect, see {{mtu}} and {{mtu-con}}.


## Path Selection {#patsel}

The path selection component is responsible for requesting paths
to a destination, ordering the path based on policy and preferences,
using them when new QUIC-paths are opened, and retiring them or listing
them for reuse when they are closed.

### General Path Management

In order to manage paths effectively, the path selection algorithm
probably requires acces to the following fields and events:

- `initial_max_path_id` ({{Section 2.1 of QUIC-MP}})
- MAX_PATH_ID frames ({{Section 4.6 of QUIC-MP}}
- PATH_AVAILABLE and PATH_BACKUP, see {{Section 3.3 of QUIC-MP}},
- PATH_ABANDON {{Section 3.4 of QUIC-MP}}.

Moreover, path selection must exclude paths whose MTU is too
small to guarantee 1200 bytes MTU payload for QUIC packets.
The effective MTU also depends on the length of the paths.


### Dynamic Approach

A dynamic approach could start with using low latency paths. If the
connection appears to be long lasting it could start, and keep, adding additional paths and as long as the traffic increases. Additional paths
can be chosen following the guidelines discussed in {{datra}}.

As an example, if the algorithm detects traffic that lasts for
at least one second and transfers at least 100MB of traffic,
the algorithm could trigger creation of additional QUIC-paths.


### Bottleneck Detection {#bottleneck}

If no live traffic information is available, bottleneck detection
can help to identify linkks that should be avoided. In path-aware
networks, this can be done using approaches such as {{UMCC}}.

One alternative is to use SCION's SCMP `traceroute` command
({{Section 6 of SCION-CP}})
to measure the latency between two consecutive AS border routers.
The measured latency can be compared to earlier measurements or to
the latency given in the path metadata. Discrepancies can be an
indication of high traffic and queueing problems on the measured link.

While traceroute may be useful, it should be used with care:

- traceroute traffic is not congestion controlled.
- it is clearly distinguishable from QUIC traffic, so it
  may affect anonymity.


## Packet Scheduling {#scheduling}

Packet scheduling helps distributing the transfer load efficiently
over multiple paths, see also {{Section 5.5 of QUIC-MP}}.

One advantage over a PAN such as SCION is that network paths
are stable and cannot change unexpectedly.
This may simplify packet scheduling algorithms because the
performance of individual QUIC-paths is more reliable if they
cannot unexpectedly be rerouted.



# Summary of Recommendations {#recommendations}

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
    the SCION layer can extend this with the AS code and the
    QUIC-MP implementation must only ensure to compare the whole
    object instead of port and IP separately.
  - The SCION implementation could detect cases where only the AS
    changes and then mangle the port or IP to trigger a path validation
    in the QUIC-MP layer. This may be pragmatic solution but is
    discouraged, because:
    - Managing paths in the SCION layer is difficult (when is a path
      valid? When is it closed?  ...).
    - It creates opportunities for memory exhaustion attacks
      (for storing the mapping of mangled IP/port).
    - It reports a wrong IP/port to the application.

- A QUIC-MP implementations SHOULD be able to recognize network path
  changes beyond 4-tuple or AS changes. This enables resetting
  congestion control and RTT algorithms.


## Recommendations for SCION Implementations

- A SCION implementation SHOULD NOT store or cache paths,
  especially (MUST?) not on the server side. This prevents memory
  exhaustion attacks, see {attack-memory-exhaustion}.
  This also avoid the problem of determining which paths are still
  alive a which have been closed or abandoned.

- When used with QUIC-MP, a SCION implementation MUST not change the
  network paths, possibly with the exception of refreshing expired paths.
  When a path stops working, the implementation should instead report an
  error to the QUIC(-MP) layer or time out silently.


## Recommendations for both QUIC-MP and SCION Implementations

- A server should return packets on the same path on which they were
  received.
  - Generally a server SHOULD respond on the same path on which the data
    was originally requested, unless the new path has been validated.
    This ensures that the path does not violate the path policy of the
    client.
  - A server SHOULD NOT create new path. Any new path may violate
    a client's path policy.
  - Returning probing packets on the same network path on which they
    were received: this greatly simplifies RTT estimation, see {{rtt}}.

- Clients may replace expired paths with identical paths without
  performing path migration / validation.

- Within a QUIC-MP session, every SCION network path should be used only
  with one path ID. However, it may be reused if the path was abandoned or
  closed. This is the respponsibility of the path selection algorithm,
  independent of whether it is considered part of SCION or part of QUIC-MP.


## Address Validation Token {#token}

**TODO** This section needs a lot more work!
See discussion in https://github.com/quicwg/multipath/issues/550

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

The aim is that {{QUIC-MP}} over SCION retains all security
properties of {{QUIC-MP}}. However, this requires some
implementation changes and additional consideration regarding:

- endhost identity: a 4-tuple is not sufficient to identify an endhost;

- netwotk path authenticity: paths may be spoofed;

- probing patterns which may expose user intentions or identity.



## Probe Fingerprinting

An endpoint may probe multiple paths in order to determine the best
path(s) for a given usecase. One example of probing packets are
packets that measure round trip time (RTT).

If sent en-block, probing packets can be detected because they
may be sent in bulk, to the same destination, in regular intervals,
and all with slightly different paths attached.

This can be used to fingerprinting an endpoints or their intentions
(applications may have unique intervals definded).

This can be mitigated by varying and generally reducing the
number of probing packets, and by sending probing packets
not en block but time-shifted.


## Path Injection {#attack-path-injection}

There are several potential attacks that build on injecting paths
(valid or invalid) into the server-side software stack.

These attacks can be prevented in several ways, we recommend the
following where possible:

1. SCION layers should avoid storing/caching paths and network addresses
   (beyond IP/port) internally.
   Instead, they should be given to the QUIC(-MP) layer or the
   application layer. That means that path information would only be
   accepyted and retained if the QUIC(-MP) or application layer.
2. SCION layers and QUIC(-MP) layers should interface  by using
   network addersses that include all information that identifies an
   andpoint, including, for example, AS code. Any change to a
   network address (including the AS code) should trigger path
   validation.

Alternatives:

1. If paths and network addresses must be stored in the SCION layer, an
   alternative solution would be to implement some kind of signalling
   which would indicate that a packet is (or would be) rejected/dropped
   by the QUIC(-MP) layer. These addresses and path from such packets
   should not be added to storage. However, to avoid connection
   drop, they should not be removed if they were previously used with
   a valid connection.

Examples of attacks include memory exhaustion attacks, traffic
redirection attacks, and traffic amplification attacks.



### Memory Exhaustion {#attack-memory-exhaustion}

An attacker may flood a server with packets that each have a
different source network address. If these are stored in the SCION layer,
they may cause memory exhaustion.

Mitigation: do not store state in the SCION layer, or implement
a way to clean up state without affecting valid connection.


### Traffic Redirection to Different AS

An attacker may craft a packet that appears to originate from the same
IP/port, but is located in a different AS than an existing connection.
If the server's SCION layer stores paths internally, and uses IP/port
as key to look them up, then the new paths may replace the exisitng one
and outgoing traffic is redirected to the new paths and destination.

Mitigation:

- The QUIC(-MP) layer MUST trigger path validation if the
  network address changes, and must consider every attribute of the address, not just IP and port.
- If a packet is rejected by the QUIC(-MP) layer, the SCION layer MUST
  NOT add it to any local state (including not replacing exisint state).
  This can be achieved trivially by not having state in the SCION layer.


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


### Traffic Amplification {#attack-amplification}

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
  (e.g. legacy library), the SCION layer (server side) should have an
  option to perform port mangling or IP mangling: when the SCION layer
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


## More ?

- Use multipathing for anonymity, see EVA in {{categories}}.

- See other attacks in {{Section 7.2.4 of SCION-CP}}?

**TODO**
