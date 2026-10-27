
<p align="center">
  <img src="https://github.com/Bolshoi777/PRISM/blob/main/666.png"
       alt="PRISM logo"
       width="240">
</p>



# PRISM: Prismatic Relay with Invisible Session Morphing

**Design document for a new tunnelling protocol. Version 0.9, 27 October 2026.**

## Abstract

PRISM is a tunnelling protocol with no fixed wire form. The payload travels in a carrier-independent record layer; the outward appearance of the connection (TLS, QUIC, HTTP/2 behind a CDN, "raw" TLS) is chosen per session and can be replaced mid-session with a single re-key; the statistical shape of the traffic is governed by an ephemeral defense derived from the session secret; the route to the exit is mixed algebraically, packet by packet, without long-lived circuits; admission to the network is granted by threshold-issued anonymous credentials, so the exit never learns who is in front of it; and the state without which a client stops working is published under threshold signatures across several independent channels, surviving the seizure of any one of them.

The central idea: the observable surface of a session — wire form, packet statistics, route, identity, rotation schedule — is a single secret that can be reissued at will. What remains visible is a crowd indistinguishable from the rest.

This document contains the threat model, a seven-layer architecture, message formats, acceptance criteria, an evaluation plan, an MVP roadmap, and an honest list of limitations. No code is included.

## 1. Problem statement

### 1.1 Limits of existing solutions

Every solution available today solves one axis and dies on it. AmneziaWG 3.1 hides the static protocol signature but not the shape of the flow; Hysteria2 with gecko obfuscation and Chrome QUIC handshake copying hides "this is QUIC" but not frequency or volume; VLESS with REALITY and XHTTP withstands active probing but is vulnerable to behavioural filtering and to the client fingerprint signature; AnyTLS was built as an answer to "TLS inside TLS" detection and has already been caught in blocks reported in May 2026; Mieru abandoned TLS and won in indistinguishability, losing resemblance to HTTPS.

The common trait of all five: the protocol form is fixed at connection setup or at node deployment. It is either identical across all sessions, or chosen once and kept until reconnection. Once the censorship authority has learned that form, the next client version is obliged to change it — and everything bound to it changes with it, which means new engineering rather than new configuration.

The second common trait: a system is usually either anonymous or block-resistant, but not both. Nym delivers real anonymity through five hops, constant cover traffic, and per-packet routing, but it does not solve survival: a NymVPN client is exactly as exposed as any other to blocking of the node address and to the fingerprint of its own stack. Mullvad DAITA, conversely, holds the line against traffic analysis while leaving address and fingerprint open. Tor is strong against infrastructure blocking and weak against a global observer.

The third: in private deployments, the configuration channel is the most common point of failure. When the subscription is unreachable, the client does not learn the new address and stays with a dead node while the rest of the stack is healthy.

### 1.2 Threat model

| Adversary | Capabilities | What PRISM provides |
|---|---|---|
| A1 In-network observer (TSPU-like) | Passive analysis, classification, filtering by fingerprint, subnet, frequency | L0–L2: no fixed form, ephemeral statistics, budgeted padding |
| A2 Active prober | Junk requests, RTT measurement, consistency checks of responses | L6: the exit answers like an ordinary origin, RTT coherence, shared exit pool |
| A3 Global passive observer | Sees both ends, builds time- and volume-based correlation | L3: algebraic mixing, per-packet routing, symmetric cover traffic |
| A4 Compromised node | Sees the ingress and egress of its own segment, may interfere | L3: mixing, verifiable mode, per-hop authenticity |
| A5 Hostile exit | Wants to deanonymise the client, sees only the users it serves | L3+L4: shared exit pool, anonymous credentials, no logs |
| A6 Censor with infrastructure access | Node seizure, subnet blocking, compelled operators, blocking of control channels | L5: threshold state, five independent channels, emergency profile |
| A7 ML-based traffic analyst | Classification, fingerprinting, adaptive training | L2: ephemeral defenses, many-to-many morphing, direction symmetry |

Out of scope: compromise of the endpoint device; a global adversary able to cut all traffic to a jurisdiction; anonymity before the destination site itself beyond IP concealment; resistance to pressure compelling a node operator to disclose the contents of its share of a session.

### 1.3 Design goals

| Goal | Criterion |
|---|---|
| No persistent signature | No observable parameter is constant across sessions: wire form, sizes, timings, route, fingerprint |
| Blocking cost | Blocking a node is not the same as blocking access: costume and address change happens automatically |
| Recovery time | ≤ 60 seconds from block to working connection with no human involvement |
| Indistinguishability of directions | Statistical traffic profiles of both directions match in distribution; asymmetric delays are forbidden |
| Resistance to training | A classifier trained on defended traffic loses accuracy under adaptive training (the DAAE criterion) |
| Decentralised control | State is published under k-of-n threshold signatures; seizure of one channel does not stop the system |
| Post-quantum strength without a new signature | Quantum protection is realised inside the tunnel and does not change the size of the outer ClientHello |
| No identity required | Exit and relay never learn who the client is; identity is not required for admission either |

## 2. Central idea

### 2.1 The session surface as a single secret

Existing protocols fix the observable surface across several independent parameters: transport type is chosen at configuration time, fingerprint profile at configuration time, padding strategy at configuration time, node address when the configuration is obtained, route at connection establishment. Each parameter can be changed on its own, but an observer remembers the combination, and the training happens on the combination.

PRISM inverts this: **one session secret generates the entire surface**. From a 32-byte secret are derived: the ephemeral padding machine, the morphing targets, the costume rotation schedule, the record keys, the initial cover-traffic window, and the anonymous credential identifier. Changing the secret yields a new session with an entirely new surface rather than a targeted edit. An observer who trained a classifier on one session loses it on the next, because everything changes except the fact that "encrypted traffic to someone is happening here".

This is a direct transfer of Kerckhoffs's principle into the domain of traffic analysis: the defense is not what is visible, but what only the owner knows. The 2026 work on ephemeral defenses shows this transfer is implementable: a defense machine is derived from a seed deterministically in 43,6 ± 4,7 ms, which makes a unique defense per connection practically free [1].

### 2.2 Protocol composition

| Layer | Name | Function | Key foundation |
|---|---|---|---|
| L0 | Surface | Set of external connection forms and fingerprint policy | Real browser network stack, Chrome QUIC copying |
| L1 | Prism-Link | Carrier-independent record layer | Fixed-size AEAD records, swappable codecs |
| L2 | Prism-Shade | Ephemeral defense of statistical shape | Maybenot, ephemerality, many-to-many morphing |
| L3 | Prism-Mix | Route anonymity | Algebraic mixing, per-packet routing |
| L4 | Prism-Pass | Admission without identity | Threshold anonymous credentials, network-dependent identity |
| L5 | Prism-Beacon | Control plane | Threshold state, blocking-cost map, emergency profile |
| L6 | Prism-Core | Exits and their observable behaviour | RTT coherence, consistency under probing |

The layer order reflects dependencies rather than aesthetics: without L1 a costume cannot be changed mid-session; without L2 L1 has no purpose; L3 requires a completed L1; L4 requires a completed L3; L5 requires completed L4 and L6; L6 requires L5.

### 2.3 Differences from the nearest work

| Proposed | Nearest work | Difference |
|---|---|---|
| Mid-session change of external form | Tor pluggable transports, Xray transport | In Tor the change happens at circuit level and preserves semantics; in Xray the transport field is fixed per node. In PRISM the record layer is carrier-independent, the switch is atomic, and the session is not broken |
| Ephemeral defense covering the entire surface | Pulls et al., PoPETs 2026: ephemeral padding [1] | There only the padding machine is ephemeral. In PRISM the same secret also yields the wire form, the morphing targets, and the rotation schedule |
| Many-to-many morphing of traces | Chameleon, arXiv 2608.20160 [5] | In Chameleon the targets are real website traces. In PRISM the targets are captured traces of real browser sessions rather than synthetic ones, and the synchronisation is implicit |
| Burst-matched cover traffic | WTF4Nym [4] | Same in idea; in PRISM it is extended with a direction-symmetry requirement and a ban on asymmetric delays, which follows from the result in [4] |
| Algebraic mixing inside a tunnel | Ge et al., 2025 [7] | There the goal is anonymity against a global observer. In PRISM mixing is part of the survival protocol: the route is recomposed on demand of a block, not on a timer |
| Threshold anonymous credentials in a tunnel | ARC (IETF), Threshold AC (UC Berkeley) [24][25] | Both works are about schemes. In PRISM they are applied as an admission layer, with the requirement that the exit and the relay receive no identity |
| PQ exchange without a new signature | RFC 10024, draft-rosomakho-mlkem512 [22][23] | Those works concern TLS. In PRISM the post-quantum part is moved inside the tunnel, because the size of the outer ClientHello has itself become a signature |
| Blocking-cost map as a routing input | OONI, Voidly, dpi-ch — as measurement | The measurements exist, but nowhere are they used as an input to route optimisation inside a tunnel |
| Emergency profile in the protocol | Tor: directory mirrors | Tor makes the catalogue fault-tolerant, but not the state of "which node, which costume, which route". In PRISM this is threshold state cryptographically bound to a pre-embedded profile |

## 3. L0 Surface: the costume set

### 3.1 Catalogue

| Costume | Outward appearance | What the observer sees | Probing resistance | Blocking cost |
|---|---|---|---|---|
| C-TLS | TLS 1.3 on 443, genuine Chromium stack, SNI of a real domain | An ordinary site visit | High: the origin answers like a website | High: indistinguishable from legitimate traffic |
| C-QUIC | QUIC with Chrome's parameters copied, UDP 443 | An ordinary QUIC visit | Medium: QUIC answers version negotiation | Medium: UDP is cut selectively |
| C-H2-CDN | HTTP/2 inside a CDN, TLS terminated at the CDN | A request to a CDN | High: the CDN answers junk as well | Very high: collateral damage at the CDN |
| C-RAW | TLS 1.3 on a non-standard port with a browser fingerprint | Unusual, but similar to a TLS site | Medium | Low: cut by port and by ASN |
| C-WEB | HTTP/2 inside DoH, a WebTunnel-like scheme | A request to a DoH resolver | High | High: DoH traffic is legitimate |

Costumes differ not by "level of protection" but by which axis they close. C-H2-CDN is nearly useless against signature-based detection and nearly immune to address-based blocking; C-RAW is the reverse. That is precisely why the set must contain the extremes rather than "one best option".

The reference point for C-TLS is the Chromium network stack, which in real deployment matches a browser byte for byte [40]. The same body of work also shows the converse: a browser client and a Go implementation produce different DTLS fingerprints, and it was by the fingerprint of the implementation that Snowflake was blocked [41]. The precedent for a protocol without a fixed form is the work on Hop, where unique channels, a hidden mode answering only to the correct key, and post-quantum forward secrecy are combined into a single construction [39].

The limitation of C-H2-CDN: TLS is terminated at the CDN, so the CDN sees the contents. The inner layer must therefore be end-to-end — an encryption scheme is needed in which the CDN sees only ciphertext, with the UUID and the destination address hidden. This is exactly the task that VLESS Encryption solves for CDN-hosted schemes, and the same argument against "encryption against the wall" applies: this layer protects against the intermediary, not against censorship.

### 3.2 Fingerprint policy

A separate L0 component, absent from every known solution: the fingerprint budget.

Each C-TLS and C-RAW costume requires a client fingerprint profile: a set of TLS extensions, their ordering, the GREASE pattern, HTTP/2 parameters, QUIC behaviour. Profile sources are captured real browser sessions rather than synthetic constructs — a synthetic fingerprint becomes a signal in its own right, as the mass migration of providers to the Chrome template demonstrated.

Policy rules:

1. A profile is used at most once per network within a 24-hour window. A profile caught twice is never used again.
2. The domain in the SNI must have an A record pointing at the node address and must serve a substantive response: otherwise the A-record check exposes the substitution.
3. The profile set is replenished by a farm: regular collection of real fingerprints from different networks, different browser versions, with different extension sets.
4. A profile change is bound to a costume change: a new costume means a new profile, otherwise the "costume + profile" combination is remembered.
5. The client fingerprint matches no known browser fingerprint under any condition — and this is verified in CI against a set of reference JA4 values and h2 parameters.

### 3.3 Costume selection rules

Selection is performed not by speed but by expected blocking cost, with measurements taken into account. The selection function accounts for: the current blocking cost of the node's subnet according to the L5 map; the blocking cost of the costume class; the observed rate of successful connections per class; the remaining fingerprint budget; the required anonymity level; the traffic type (interactive, bulk, sensitive).

Downgrade rule: a transition to a costume that is cheaper in robustness (C-RAW) requires a fresh presentation of an anonymous credential. This makes a downgrade impossible "silently" and turns it into a deliberate decision that the network side validates.

## 4. L1 Prism-Link: the record layer

### 4.1 Record format

The record is the unit of transfer. The payload is cut into fixed-size blocks; each record is authenticated and encrypted separately; the record size is constant so that packet length does not encode the type of content.

| Field | Size | Purpose |
|---|---|---|
| Stream number | 2 bytes | Separation of outbound streams inside the tunnel |
| Record number | 4 bytes | Ordering and replay protection |
| Type | 1 byte | Data, cover-packet marker, control, costume change |
| Payload length | 2 bytes | Actual amount of data in the record |
| Record salt | 16 bytes | Unpredictability for the censor and the relay |
| Ciphertext | up to 1024 bytes | AEAD: ChaCha20-Poly1305 or AES-256-GCM |
| Tag | 16 bytes | Record authentication |

The resulting fixed record size is on the order of 1064 bytes. The important consequence: a record is indistinguishable from another record by length, and a cover packet differs from a real one only in a type field that is encrypted and therefore unavailable to the observer.

### 4.2 The morph operation

Morphing is an atomic change of the external form with the session preserved. Order of operations: client and node agree on a new "costume + codec" pair through a control record; a new record key is derived from the session root with a separate label; in-flight records are allowed to finish; the stream switches to the new codec; the outer transport is reopened with a new fingerprint profile and a new SNI.

The change takes one round-trip time plus the time to reopen the outer connection. The key point: **records do not change format during a morph** — only the outer wrapper changes. The switch therefore leaves no trace in the statistics of the content, and a complete session change is possible at any moment by the same means.

A limitation that has to be named plainly: the costume change is itself an event visible to the observer (a new TLS connection with a new SNI, a new endpoint). The morph schedule must therefore be generated by the ephemeral L2 machine rather than follow a fixed timetable.

### 4.3 Direction symmetry

A requirement derived from the work on fingerprinting in Nym [4]: the statistical profiles of the two directions must match in distribution. Asymmetric delays, asymmetric padding sizes, and different cover-traffic rates in the two directions create an observable gap between request bursts and response bursts, and that gap is the strongest feature available to a classifier.

Hence the implementation rule: the defense machine runs in both directions with identical distribution parameters but different seeds; cover traffic is generated symmetrically; in-network delays are never applied asymmetrically on any hop.

## 5. L2 Prism-Shade: ephemeral defense of shape

### 5.1 Deriving the defense from the secret

The defense is derived deterministically from the session secret with an xoshiro256 generator, following the method of ephemeral defenses [1]. Derivation takes on the order of 44 ms per session, which allows a unique defense to be issued for every connection without noticeable start-up delay. The defense is expressed as a probabilistic finite state machine within the Maybenot framework: events are records sent and received and timer expiry; actions are sending padding, delaying outbound data, changing the morphing target, and initiating a costume change.

The key property: the defense is unknown to the observer, so a classifier cannot be trained on a specific "original trace → defended trace" pair. The 2026 work shows that on defenses of this kind the accuracy of a DF attack falls from 89,0% to 10,2%, and RF from 90,1% to 14,7%; under unbounded attacker training, accuracy rises only to 29,2% and 24,3% against 92,7% and 94,7% for undefended traffic [1]. Such a scheme is already deployed in production: DAITA has shipped as an optional feature on top of a WireGuard fork and rests on three mechanisms — random background traffic, data pattern distortion, and constant packet size [29][30]. The cost is 97,5% bandwidth and 68,4% delay, and that is the price that will have to be brought down by tuning the budgets.

Properties to preserve in the design: padding and blocking budgets are set per class (framework-wide limits) rather than hard-coded; the defense engages from the first packets of a session rather than after accumulating N packets; combining base machines with stacking yields an exponential space of variants (in the original work, 6,88 × 10²⁵ combinations at height 5 over a base of 1000), and this space must be a profile parameter rather than a code constant.

### 5.2 Many-to-many morphing

A defense must not be a deterministic function of the original trace: such defenses are learned by an autoencoder trained on defended traffic and are stripped almost completely — over 90% DF accuracy against RegulaTor and over 80% against Palette and Adaptive Tamaraw, even though ordinary attacks against them yielded only 2–35% [5].

The solution carried over from Chameleon: a many-to-many mapping instead of one-to-one. Each original trace is mapped to several targets drawn from a pool by a criterion of high intra-class diversity and low inter-class disparity; different original traces may share a target. Robustness comes from the absence of a learnable stable mapping.

The transfer into PRISM changes the target pool: these are not website traces but captured traces of real browser sessions, collected by the fingerprint farm. Synchronisation between the peers is implicit: the side that determines the target trace first transmits a short sequence of packet directions, and the other side finds the match by prefix in a local trie. An explicit trace identifier never enters the wire.

A second source of surface diversity is the generation of protocol variants. The UPGen approach shows that a structured parametric choice yields on the order of 4,2 × 10²² mutually dissimilar protocols, with Zeek, nDPI, and libprotoident failing to recognise 93–100% of such flows [33]. For PRISM this argues for replenishing the costume pool not only by capturing real sessions, but also by generating variants subject to mandatory verification of unrecognisability.

### 5.3 Burst-matched cover traffic

A strictly constant packet stream is not the better choice: it destroys useful throughput and still does not reproduce the bursty nature of the web. The work on fingerprinting in Nym shows that cover traffic built on a Poisson process with a 10–20 ms step reduces the Tik-Tok F1 from 0,87 to 0,75–0,64, at 7,32–12,66× latency overhead [4]. The burst-oriented variant of the same approach yields F1 = 0,39 at 3,78× latency and 4,22× bandwidth [4]. The constant-rate mode yields F1 = 0,06, at 7,32× latency and 8,06× bandwidth [4].

The design conclusion: cover traffic must reproduce the burst distribution characteristic of the web rather than being flat. PRISM learns the burst distribution on the client's network and generates fake bursts with the same statistics. An important correction: in mixnet architectures a constant stream is expensive for a second reason — the fixed-size packet there is large, in Nym 2413 bytes, which makes a cover packet 4,7× the cost of a Tor cover cell [4]. In PRISM the record size is ours to set, and that is a separate lever of economy: a cover packet must not carry payload, but it must cost the same as a real one, otherwise it stands out by length. The price of an inconsistent size has been measured: without chunk padding a classifier identifies the site type from sizes with 91% accuracy, and with a 2 MB chunk accuracy falls to 12% at 21,3% overhead [34].

### 5.4 Defense acceptance criteria

A defense is not accepted without verification against a fixed set of criteria, and this list matters more than the mechanism itself.

| Criterion | Threshold | Basis |
|---|---|---|
| DF attack accuracy on defended traffic | ≤ 25% | The level reached by the best current defenses [5] |
| Resistance to a defense-aware autoencoder | The attack does not recover accuracy above 40% | Otherwise the defense is learned in a single pass [5] |
| Resistance to an attacker randomising the training set | At K = 20 variants per trace, accuracy ≤ 35% | Criterion from [5] |
| Early-stage leakage | F1 of the partial-load attack ≤ 20% | Early traffic is the most vulnerable phase [5][18] |
| Bandwidth overhead in the working mode | ≤ 40% | The affordability threshold; the 97,5% from [1] is the price of the maximal-protection mode |
| Delay overhead | ≤ 100 ms | The practical limit for interactive traffic |
| Resistance to training on defenses | A classifier trained on defense A loses accuracy on defense B | The ephemerality property [1] |

Separately: the results of state-machine simulations diverge from those of real implementations [3]. Acceptance measurements must therefore be taken on a working implementation rather than on a model, and this has to be recorded as a process requirement rather than a wish.

## 6. L3 Prism-Mix: route anonymity

### 6.1 Algebraic mixing

Classical mixnet constructions rely on permutation and delay; their weakness is that correlation by time and volume remains probabilistic, and current classifiers exploit it. A 2025 paper offers a construction with formal indistinguishability: per-round mixing at constant rate, random masks with global sum zero, PRF outputs with sum zero, and an inner product as the mixer, which erases leakage of time, size, and the sender–receiver pair [7]. Practical parameters: 128-bit security, 392-byte ciphertext, per-round mixing faster than a second, practical for a few tens of mixed packets [7].

In PRISM the mixing is performed over packets of the record layer rather than over application messages. The practical consequence: the mixing unit size is determined by the record layer and is therefore controllable, and the mixing serves the goals of anonymity and survival at the same time.

The route is recomposed per packet rather than per session: a long-lived circuit is an observable object, and every correlation attack targets it. Nym implements this, and it is worth adopting [11][12]. The properties of the mixnet construction underlying the calculation are set out in the network's technical document [31]. A separate technique, suitable for the relay selection rule, is per-inbound-edge routing over a quasi-regular anonymity graph, as in Dandelion++; the analysis of that scheme yields a median anonymity of roughly 5–7 bits even with 1% dishonest nodes [32].

### 6.2 Ban on asymmetric delays

The most counterintuitive and most useful result in the recent literature: **increasing the mixing delay makes the defense worse**. Raising the per-hop delay to 100 ms grows the Tik-Tok F1 on Nym from 0,91 to 0,98, and the Transposition feature set reaches 95,5% against 88,5% at zero delay [4]. The reason: a large delay creates a visible temporal gap between the request burst and the response burst, which helps the classifier separate the two directions.

Hence a direct and counter-intuitive design rule: **delay in PRISM is not a defense**. Mixing must work through permutation and constant rate, and delay serves only as a tool for equalising directions, with identical parameters in both. Cover traffic at the network boundary works noticeably better than the depth of the delay [4].

The dominant leakage vector in Nym is the inbound direction: 72,1% random-forest accuracy on features of the inbound flow against 70,2% for Tor, while on outbound features it is 43,9% against 65,0% [4]. Hence the asymmetry rule: cover traffic in both directions simultaneously, otherwise the separation of directions is exploitable.

### 6.3 Verifiable mode

Algebraic mixing loses its formal guarantees when a node is compromised: in a parallel cascade of mixers, the TVD rounding to 1/n is reached in O(log k) steps independently of the number of messages, but the property collapses with a single dishonest node [9]. Marked mixnet constructions, which economise on integrity, proved insecure in the presence of a single dishonest authority [10].

PRISM therefore provides two modes: an ordinary one, unverified, with minimal overhead, for bulk traffic; and a verifiable one, with a lattice-based shuffle proof, for important sessions. The cost of the verifiable mode: ciphertext expansion below 3 KB per layer, roughly 12× smaller than classical constructions, with post-quantum security [8]. This is still too expensive for bulk traffic, and the specification must explicitly cap the verifiable mode by volume.

## 7. L4 Prism-Pass: admission without identity

### 7.1 Threshold issuance

Entry into the network must be possible without revealing who is entering. The most practical scheme today is single-use anonymous tokens built on blind signatures (RFC 9497, blind RSA) and anonymous credentials with a bounded number of presentations (draft ARC) [24][26]. Blind signatures are already in commercial products: Apple Cloud Relay and Google One VPN use blind RSA [26].

A single issuer is a point of failure and a point of compulsion. A 2026 paper from UC Berkeley constructs threshold anonymous credentials with issuer hiding, constant-size presentation, threshold issuance, and no interactive key setup (silent setup) [25]. Silent setup is exactly what makes the scheme suitable for issuers that are unwilling or unable to coordinate in advance — which is precisely the situation of independent operators in different countries.

In PRISM this looks as follows: at least three issuers, a threshold of at least two of three for validating a presentation, and the presentation carries nothing beyond a proof of the right to use the network. Exit and relay verify the presentation but receive neither a user identifier nor a stable pseudonym. The practical foundation already exists: single-step blind token issuance is specified in RFC 9497 [28], and the 2026 survey of anonymous credentials shows the transition from classical blind signatures to credentials with selective disclosure and threshold issuance [27].

### 7.2 Network-dependent identity

A requirement formulated in OONI materials: remain unlinkable across networks while remaining consistent within a single network [26]. For a tunnel this means a credential may be used many times inside one network, but the "credential — network" binding must not carry over to another network. The practical benefit is twofold: a relay can rate-limit per network without learning the user, and the user cannot be identified by the fact of using the same tunnel in different networks.

### 7.3 Abuse mitigation

Anonymous entry is also anonymous entry for abuse. Mechanisms of limited applicability: a presentation limit per credential (a property of the ARC scheme) [24]; a proof-of-work ticket issued blindly by the issuer, so that even the fact of requesting a ticket is not linked to the user; limiting by network address rather than by identity; and a strict issuer-side policy with the risk of exclusion from the threshold.

## 8. L5 Prism-Beacon: the control plane

### 8.1 Threshold state

The state without which the client does not work is published as a single threshold-signed document: the list of nodes with their costumes and addresses, the current blocking-cost map, the defense profile parameters, and the emergency transition schedule. The signature requires k of n signatures from independent publishers located in different networks and operated by different operators.

There must be at least five publication channels, and they must be heterogeneous with respect to failure mode: web endpoints on independent ASNs; distribution through a DHT with several seeds; rendezvous points of Tor hidden services [42]; the push channel of mobile platforms; and a fallback channel via DNS text over DoH. The push channel is valuable precisely because of cost asymmetry: blocking push infrastructure affects incomparably more applications than it gains a censor [38].

A client accepts state if the signatures verify and the signing window has not expired. Divergence of state between channels is not a conflict: the most recent signed version applies, and divergence is a signal to switch to the emergency profile.

### 8.2 The blocking-cost map

Existing measurers (OONI, Voidly, dpi-ch, siberian-check) collect availability data, but that data is not used as an input to route optimisation. In PRISM the cost map is a structured table — "ASN, costume class, estimated blocking probability, time of last check, number of independent confirmations" — published inside the signed state.

Sources: aggregated and anonymised measurements contributed by network participants; the client's own probes from different networks; and previously recorded failures, with each client reporting a block as a signed message, which makes the report itself a measurement.

The value of the approach: blocking cost becomes an optimisable quantity rather than an observed one. A client that knows subnet A is blocked in 40% of cases and subnet B in 3% chooses B even at the cost of 40 ms of extra delay. The practical test: a network of 20–30 independent operators in different countries with aggregated telemetry survives the seizure of individual participants, because the cost of blocking the network itself grows.

### 8.3 Emergency profile

If the signed state is not refreshed for longer than a given interval, the client switches to the emergency profile: a cryptographically fixed set of nodes, costumes, and selection rules, embedded in the client and covered by the signatures in effect at build time. The profile does not refresh, but it is known and verifiable. The transition happens automatically, without human involvement, and that is a separate design goal: recovery time after a block must not depend on the availability of the control infrastructure.

## 9. L6 Prism-Core: the exit and its observable behaviour

### 9.1 RTT coherence

Active probing detects a VPN through RTT discrepancy: the destination sees two RTT clusters — "destination-to-node" and "destination-to-node-to-client" — and their mismatch gives the tunnel away. The method works from the TCP three-way handshake or the SSH version exchange, that is, before any user data is transferred, and shows an average accuracy of 95,75% and 99,40% with cross-training [17]. The RTT approximation obtained by traceroute differs from the true value by only a few percent [17].

The design requirement: **the exit must not be a dedicated server**. Exits form a shared pool serving many users, and are placed in geographic proximity to the client's path so that the additional hop does not create a second RTT cluster. When selecting an exit, the client accounts not only for speed but also for the expected RTT difference.

A further consequence: the destination address must not correlate with the fact that a tunnel is in use. Shared exits with a residential or mobile egress pool give the destination a picture indistinguishable from an ordinary client.

### 9.2 Behaviour under probing

Every failure at the network boundary must look like ordinary origin behaviour: an invalid credential presentation, a wrong path, an exceeded limit — all of these answer with the same profile as a legitimate request, with the same response-time distribution and the same response size. Service-level error markers must not differ from application-level errors.

Separately: the protocol must provide protection against active probing at the relay level, not only at the exit. The technique from TARANET applies here — per-hop authenticity and tamper detection [6]: a node that receives a packet with broken integrity discards it, and the marked packet never reaches the next hop, which breaks watermark attacks.

### 9.3 Recovery time

Failure scenarios and the stack's reaction are defined in advance: outer transport failure — change costume on the same node; node failure — change node within the same ASN, then within another ASN, then switch to the emergency profile; subnet block — move to a node in another subnet, change costume, and on failure switch to the emergency profile with a 60–90 s delay; blocking of an entire cluster — switch to another costume of the class and another set of ASNs.

The target recovery time is 60 seconds in urban conditions with a single blocked node, and 5–10 minutes when the client's entire ASN is blocked — and this is the only metric worth showing to a user.

## 10. Message formats

The protocol's control messages are described below as sets of fields; an implementation encodes them into the L1 codec, and they enter the wire encrypted inside a record.

| Message | Sender | Contents | Purpose |
|---|---|---|---|
| Hello | client | Version, credential identifier, costume proposals, defense parameters | Surface negotiation |
| HelloAck | node | Selected costume, defense parameters, issuance window, record lifetime | Session establishment |
| Morph | client or node | Costume, fingerprint profile, key label, switch window | Mid-session form change |
| Mix | relay | Packet within a mixing group, round number | Mixing |
| Rekey | either | New root label, application window | Key rotation |
| Beacon | publisher | Signed state, signing window, version number | Control |
| Cover | client | Nothing (marker in an encrypted field) | Cover traffic |
| Probe | observer | Arbitrary request | Liveness and behaviour check |

## 11. Operating modes and the overhead budget

| Mode | Contents | Bandwidth overhead | Added delay | What it defends against |
|---|---|---|---|---|
| Fast | One costume, one node, no cover traffic | < 5% | 0–30 ms | Address-based blocking only |
| Steady | Ephemeral defense, one node, no mixing | 30–40% | 50–100 ms | Blocking and traffic analysis |
| Shielded | Steady + relay + mixing + cover traffic | 100–150% | 100–200 ms | Plus route anonymity |
| Vault | Shielded + verifiable mixing + PQ credentials | 300%+ | 250 ms+ | Plus distrust of nodes |

The overhead figures rest on published measurements: Maybenot defenses cost about 80% bandwidth for the FT-1 configuration and 125% for FT-2, with 69,8% and 22,0% delay for the lighter and 104,2% for the heavier constant-rate mode, while the regulator machine demands 178–213% bandwidth and is considered impractical [2]. Ephemeral defenses in a production deployment cost 97,5% bandwidth and 68,4% delay [1]. The "Steady" mode at 30–40% bandwidth is not a measured figure but a design assumption that has to be validated experimentally; that validation is the priority of the experiment.

Mixing in mixnet architectures is expensive: Nym adds about 45 ms of latency for three layers of mixers and about 1 Mbit/s of cover traffic by default, with a tuning range of 0,7–2 Mbit/s and 0–200 ms per hop [11][13]. The direction of development is fixed: a new cover-traffic scheme developed jointly with EPFL is intended to speed up connection establishment and reduce exposure to statistical attacks [15]. For comparison, Tor slows transfers by 3–4×, and adding a mixnet costs another 3–4× on top [16].

## 12. Threat-by-threat analysis

| Threat | Protection mechanism | Residual risk |
|---|---|---|
| Signature-based blocking | Real browser stack, absence of a fixed form, ephemeral morphing | A censor may train a detector on a particular costume set; mitigated by ephemerality and pool diversity |
| Subnet-based blocking | ASN rotation, cost map, CDN costume, emergency profile | Simultaneous blocking of all involved ASNs; the attack cost grows with the number of independent operators |
| Client fingerprint blocking | Fingerprint budget, JA4 verification in CI, ban on synthetic profiles | Profile leakage on first use; compensated by single use |
| Behavioural filtering | Parallelism limits, ephemeral shape, absence of a fixed rate | The filter may adapt to the measured profile of the stack |
| Active probing | Shared exit pool, RTT coherence, consistent responses | Compromise of an exit allows it to be distinguished by RTT |
| Time and volume correlation | Algebraic mixing, per-packet routing, direction symmetry, burst-matched cover | Collective attack on a mixer given one dishonest node; small mixing group size |
| ML-based fingerprinting | Ephemeral defenses, many-to-many morphing, symmetry, early phase | An attacker with unlimited access to defended traffic; the DAAE criterion as a mandatory check |
| User identification | Threshold anonymous credentials, no logs, shared exit pool | A compromised issuer or collusion between issuers |
| Control-plane failure | Threshold state, five channels, emergency profile | Simultaneous blocking of all channels; the cryptographic binding of the emergency profile becomes stale |
| Node seizure | Absence of address-to-protocol binding, CDN fronting, emergency profile | Seizure of a single node reveals its address and makes blocking the address class easier |

## 13. Comparison with existing solutions

| Property | WireGuard | XHTTP/REALITY | Hysteria2 | Tor | Nym | Mullvad DAITA | PRISM |
|---|---|---|---|---|---|---|---|
| Mid-session form change | No | No | No | At circuit level | No | No | Yes, atomic |
| Ephemeral defense of shape | No | No | No | Limited | Fixed scheme | No (static) | Yes, per session |
| Resistance to classifier training | No | No | No | Limited | Limited | No | Yes, by construction |
| Route anonymity | No | No | No | Limited | Yes | No | Yes, with verifiable mode |
| Admission without identity | No | No | No | Limited | No | No | Yes, threshold-issued |
| Control-plane resilience | No | No | No | Directory mirrors | No | No | Threshold + 5 channels |
| Blocking of a node address | Fatal | Fatal | Fatal | Not applicable | Unaddressed | Fatal | Rotation + emergency profile |
| Overhead | None | Low | Low | Medium | High | High | Configurable budget |

## 14. Evaluation plan

Evaluation must be reproducible and verifiable, otherwise it turns into marketing.

**Offline, traffic analysis.** Collection of reference traffic in the four modes. Classifier training: CNN1D on length-time sequences of the early flow, XGBoost, random forest. The measured baseline for WireGuard: 0,98 balanced accuracy when transferring from the "non-VPN" domain to the "VPN" domain without any VPN training samples, macro F1 = 0,89 [18]. That is the threshold below which PRISM is not worth discussing: a classifier of that class trains on the very first packets, so the defense must work from the first packet. Benchmarks for accuracy come from the work on ML-based VPN detection and the 2025–2026 surveys: traffic-classification foundation models outperform conventional convolutional networks by 2–10 percentage points, and temporal features prove the most discriminative [19][20][21].

**Offline, defense robustness.** Three mandatory tests: ordinary accuracy; robustness to training on defended traffic; and a defense-aware autoencoder [5]. Plus verification on partial loads and on robustness to a randomised training set at K = 20.

**Offline, correlation.** Evaluation of ingress-egress linkability by time and volume on a simulated network; verification of the anonymity-set bound; verification of mixing robustness against one dishonest node.

**Offline, cryptography.** Verification of quantum resistance: the handshake inside the tunnel, not outside; assessment of the effect on outer ClientHello size — zero, since the outer form does not change; comparison of the cost of verifiable mixing on lattices.

**Online, survivability.** Deployment in a controlled network able to simulate blocks along each of the six detection axes. Measurable quantities: time to block, recovery time, number of manual actions, share of successful sessions.

**Online, probing.** Verification of resistance to the RTT-discrepancy method: the destination-to-node and destination-to-client paths must produce a single cluster.

**Online, quality of service.** Overhead in each mode, impact on video and gaming, behaviour on a lossy path.

## 15. Roadmap

**Stage 0, two weeks: the core.** The record layer, one costume on a genuine browser stack, one node, entry by proof-of-work ticket without credentials. This already yields a marked improvement over "just XHTTP": form is separated from content, and the attachment point for the later layers is in place.

**Stage 1, four weeks: shape.** Ephemeral defense derived from the session secret; CI measurement of classifier accuracy and robustness to training; modes with a configurable budget; the first set of costumes and profiles.

**Stage 2, six weeks: route and anonymity.** The relay, algebraic mixing, per-packet routing, burst-matched cover traffic, direction symmetry; a test bench for measuring correlation.

**Stage 3, four weeks: admission and control.** Threshold anonymous credentials with silent setup across three independent issuers; threshold state; five publication channels; the blocking-cost map; the emergency profile.

**Stage 4, as cryptography matures.** Post-quantum exchange inside the tunnel, including a variant with a pre-shared secret in the spirit of the Lewes protocol [14]; verifiable mixing on lattices; multipath via MASQUE and Multipath QUIC.

**Stage 5, operations.** The fingerprint and domain farm, the measurement loop, network capacity planning, an incident response procedure.

## 16. Limitations and open problems

No protocol is invulnerable, and PRISM is no exception: it merely shifts the trade from "being unrecognisable" to "being hard to train and quick to recover from".

The limitations that have to be named plainly.

The price of anonymity is bandwidth and delay. The "Shielded" mode costs 100–150% bandwidth and 100–200 ms of delay; the "Vault" mode costs multiples more. Video calls, gaming, and remote desktop are not viable in these modes. A mode without mixing is needed, and then route anonymity disappears with it.

A formal mixing guarantee requires group size. Algebraic mixing is practical with a few tens of packets per round [7]; at low network load the group never fills, and the guarantee degrades. This is a question of network growth rather than cryptography, and it is not solved by engineering.

A single compromised node destroys the formal guarantees [9][10]. The verifiable mode cures this, but expensively [8]. The specification must state explicitly that in the ordinary mode mixing provides a statistical rather than a formal guarantee.

Threshold credential issuance requires governance: who issues, how it is funded, what happens under collusion. Silent setup removes the need for prior coordination but not the need for policy. This is an operational rather than a technical problem, and it is solved only by a community of operators.

The main risk of the project is not the protocol but its operation. The costume set, the pool of domains with correct A records, the fingerprint farm, the cost map — all of this demands constant manual work, and without it the protocol degrades into an ordinary tunnel with an extra layer.

Open questions: synchronisation of morphing-target selection when the first packets are lost; behaviour under one-way packet loss, where direction symmetry is inevitably violated; compatibility with TUN and system proxy during a mid-session costume change; client behaviour when the cost map changes in a way that requires an immediate route change; and scalability of verifiable mixing on lattices to volumes suitable for bulk traffic.

The anti-DPI layer for direct access remains an orthogonal problem solved separately: automatic selection of desync strategies for a particular filter is provided by an evolutionary algorithm [35], and ready-made tools can select and remember a strategy [37]; the measurement of the 2026 detection axes is described in a separate analysis [36]. Embedding that layer in PRISM is not planned — it would enlarge the protocol surface without a corresponding gain in survivability.

## 17. Legal notes

This document describes means of protecting one's own traffic and circumventing access restrictions. It contains no techniques for actively counteracting censorship infrastructure, and it should not: the effect of filtering degradation through the computational cost of decryption reported in the academic literature is used here only to explain a limitation and to argue against applying such techniques.

The legal framework in Russia: personal use of circumvention tools is not itself prohibited; since 1 September 2025 liability applies to advertising means of access to restricted resources (Article 14.3 of the Code of Administrative Offences) and to deliberate access to knowingly extremist material through circumvention tools (Article 13.53 of the same code); the use of circumvention tools in the commission of a crime is an aggravating circumstance; and Roskomnadzor Order No. 196 of October 2024 restricts the dissemination of information about circumvention services. This document is not legal advice and is intended for a personal context; the public promotion of circumvention tools in Russia carries independent legal risk.

## 18. References

1. Pulls T., Korhonen T., Witwer E., Carlsson N. Ephemeral Network-Layer Fingerprinting Defenses. Proceedings on Privacy Enhancing Technologies 2026(1), 426–449 — https://petsymposium.org/popets/2026/popets-2026-0022.pdf
2. Pulls T., Witwer E. Maybenot: A Framework for Traffic Analysis Defenses. WPES 2023 — https://ethanwitwer.com/assets/pdf/2023-wpes.pdf
3. State Machine Frameworks for Website Fingerprinting Defenses: Maybe Not. arXiv:2310.10789 — https://arxiv.org/pdf/2310.10789v1
4. Jollès E., Wicky S., Piotrowska A. M., Halpin H., Troncoso C. Website fingerprinting on Nym: Attacks and Defenses. Proceedings on Privacy Enhancing Technologies 2026(2), 436–451 — https://petsymposium.org/popets/2026/popets-2026-0055.php
5. Cui Y., Wei K., Shen K., Wang N., Lu Z., Liu Y., Wang G. Chameleon: Robust Defense Against Tor Website Fingerprinting via Many-to-Many Traffic Morphing. arXiv:2608.20160 — https://arxiv.org/pdf/2608.20160v1
6. Chen C., Asoni D. E., Perrig A., Barrera D., Danezis G., Troncoso C. TARANET: Traffic-analysis resistant anonymity at the network layer. EuroS&P 2018 — https://netsec.ethz.ch/publications/papers/chen_taranet_eurosp18.pdf
7. Ge T. et al. A Provably Secure Network Protocol for Private Communication with Analysis and Tracing Resistance (2025) — facts and numerical parameters extracted via a secondary survey: https://www.emergentmind.com/topics/traffic-analysis-resistance
8. Bootle J., Lyubashevsky V., Merino-Gallardo A. Efficient Verifiable Mixnets from Lattices, Revisited. IACR eprint 2025/658 — https://eprint.iacr.org/2025/658.pdf
9. Klonowski M., Kutyłowski M. Provable Anonymity for Networks of Mixes — https://www.freehaven.net/anonbib/cache/ih05-Klonowski.pdf
10. Haines T., Pereira O., Röenne P. Short paper: An Update on Marked Mix-Nets: An Attack, A Fix and PQ Possibilities — https://orbilu.uni.lu/bitstream/10993/45357/1/MarkedMixnets.pdf
11. Nym. Mixnet Mode — https://nym.com/docs/network/mixnet-mode
12. Nym. Cover Traffic — https://nym.com/docs/network/mixnet-mode/cover-traffic
13. Nym. Mixnet Tuning in NymVPN explained — https://nym.com/blog/mixnet-tuning
14. Nym. Post-quantum security on NymVPN: Lewes Protocol — https://nym.com/blog/lewes-protocol-post-quantum
15. Nym. Roadmap for 2026 — https://nym.com/blog/nym-roadmap-2026
16. Nym. Nym vs VPNs, Tor, I2P and E2EE: threat model — https://nym.com/docs/network/threat-model/comparisons
17. Tian Y., Cao Z., Huang S. S. Detecting VPN Traffic in Real-Time with Active Probing. NCA 2024 — https://www.nca-ieee.org/assets/slides/paper16.pdf
18. Razooqi Y., Pekár A. Matched-View Cross-Domain Evaluation of WireGuard VPN Traffic Classification Using Early-Flow Fingerprints. arXiv:2608.30000 — https://arxiv.org/html/2608.30000v1
19. Fesl J., Naas M. A comprehensive machine learning-based approach for VPN traffic detection, classification and hiding. Computer Networks 270, 2025 — https://dl.acm.org/doi/10.1016/j.comnet.2025.111530
20. Enisoglu R., Alp Inan B. Network traffic classification from handcrafted features to foundation models: a comprehensive survey. Computer Networks, 2026 — https://dl.acm.org/doi/10.1016/j.comnet.2026.112578
21. Razooqi Y., Pekár A. VPN Traffic Analysis: A Survey on Detection and Application Identification. IEEE Access 13, 2025 — https://doi.org/10.1109/ACCESS.2025.3592152
22. RFC 10024. Post-Quantum Traditional (PQ/T) Hybrid Key Agreement Mechanisms for TLS 1.3 — https://datatracker.ietf.org/doc/rfc10024/
23. draft-rosomakho-tls-ecdhe-mlkem512-01. Post-Quantum hybrid ECDHE-MLKEM512 Key Agreement for TLSv1.3 (Zscaler, 14.09.2026) — https://datatracker.ietf.org/doc/html/draft-rosomakho-tls-ecdhe-mlkem512-01
24. draft-yun-privacypass-crypto-arc-00. Anonymous Rate-Limited Credentials (IETF) — https://www.ietf.org/archive/id/draft-yun-privacypass-crypto-arc-00.html
25. Policharla G.-V., Roberts B. Threshold Anonymous Credentials with Silent Setup. UC Berkeley EECS-2026-124 — https://www2.eecs.berkeley.edu/Pubs/TechRpts/2026/EECS-2026-124.pdf
26. OONI. Probe Security Without Identification — https://ooni.org/post/2025-probe-security-without-identification/
27. Cloudflare. Policy, privacy and post-quantum: anonymous credentials for everyone — https://blog.cloudflare.com/pq-anonymous-credentials/
28. RFC 9497. Verifiable Oblivious Pseudorandom Functions (VOPRF) — https://www.rfc-editor.org/rfc/rfc9497.html
29. Mullvad. DAITA: Defense Against AI-guided Traffic Analysis — https://mullvad.net/en/vpn/daita
30. VPN Protocol Registry. Mullvad DAITA dossier: WireGuard base, Maybenot framework, no cryptographic replacement — https://github.com/steveprice-dev/vpn-protocol-registry/blob/main/generated/dossiers/mullvad-daita.md
31. Nym. The Nym Network (whitepaper): Sphinx, cover-traffic loops, mixnet properties — https://www.cryptocompare.com/media/39838452/nym-whitepaper.pdf
32. Anonymity of peer-to-peer networks: the case of Dandelion++. arXiv:2201.11860 — https://arxiv.org/html/2201.11860v4
33. UpGen. Unidentified Protocol Generation. USENIX Security 2025 — https://www.usenix.org/system/files/usenixsecurity25-wails.pdf
34. Huma. NDSS 2026 — https://www.ndss-symposium.org/wp-content/uploads/2026-f328-paper.pdf
35. Geneva. Automatic search for circumvention strategies — https://geneva.cs.umd.edu/
36. Teleproxy. DPI Resistance section: 2026 blocking waves, re-assembly, correlation — https://github.com/teleproxy/teleproxy/blob/main/docs/features/dpi-resistance.md
37. nfqws2-ui. Changelog: automatic strategy selection, catalogue, the "16 KB cut" probe — https://github.com/zemidala/nfqws2-ui/blob/main/CHANGELOG.md
38. CenPush. Push notifications as a control channel. PoPETs 2025 — https://petsymposium.org/popets/2025/popets-2025-0153.php
39. Hop. USENIX Security 2026 — https://www.usenix.org/system/files/conference/usenixsecurity26/sec26_prepub_flammarion.pdf
40. NaiveProxy. Architecture on the Chromium network stack — https://github.com/klzgrad/naiveproxy
41. Obscura. WebRTC encapsulation and ephemeral proxies. PoPETs 2026 — https://petsymposium.org/popets/2026/popets-2026-0030.php
42. Tor Project. Connecting from censored regions — https://support.torproject.org/tor-browser/circumvention/connecting-from-censored-regions/
