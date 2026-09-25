## QoS Models — Best Effort, IntServ (RSVP), DiffServ

> 💡 **TL;DR:** A *QoS model* is an architecture that answers three questions: **how are requirements expressed, who decides, and where is the state kept.** Three models matter. **Best effort** — no promises; the default (Default PHB). **IntServ** (RFC 1633) — applications *reserve* resources per flow with **RSVP** (RFC 2205) and every router on the path runs admission control and keeps per-flow soft state; the *Guaranteed* service gives a mathematically provable **queueing-delay bound** (RFC 2212). **DiffServ** (RFC 2474/2475) — no per-flow state or signaling in the core: traffic is **classified, conditioned and marked at the network edge** with a 6-bit **DSCP**, and each core node applies a **Per-Hop Behavior (PHB)** to whole aggregates. IntServ gives strong per-flow assurance at the cost of state that grows with the number of reservations (RFC 2208); DiffServ scales because the core sees only a handful of aggregates. Modern networks are built on DiffServ; IntServ ideas survive as admission control and at the edge.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / ITU / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 1633 (IntServ overview) — Informational · RFC 2205 (RSVP), RFC 2211 (Controlled-Load), RFC 2212 (Guaranteed) — Standards Track · RFC 2474 (DS field) — Standards Track · RFC 2475 (DiffServ architecture), RFC 2208 (RSVP applicability), RFC 2998 (IntServ over DiffServ), RFC 3260 (terminology updates) — Informational.

---

## 1. Why "Models" — the Big Picture

```
 Endpoint ---> Switch ---> Router ===WAN===> Router ---> Switch ---> Endpoint
     |                        |                  |
 Best effort:   nothing special anywhere — one queue, first-in first-out
 IntServ:       endpoint asks each router along the path to RESERVE for this flow
 DiffServ:      edge router marks + polices; every router treats the MARK (aggregate)
```

| | Best Effort | IntServ | DiffServ |
|---|---|---|---|
| **Idea** | Deliver as many packets as possible, as soon as possible | Reserve resources per flow, end to end | Mark packets into a few classes; treat classes per hop |
| **Granularity** | None | Per **flow** (simplex stream) | Per **behavior aggregate** (all packets with one DSCP) |
| **Signaling** | None | RSVP (receiver-initiated, hop by hop) | None — marking is in the packet header |
| **State in core** | None | Per-reservation soft state in every router | None per flow; only PHB configuration |
| **Admission control** | None | Per flow, at each hop | By provisioning / SLA at the edge (voice call admission is usually done by a call server via signaling at access points — RFC 4594) |
| **Assurance** | None | Strongest (Guaranteed service) | Relative or aggregate; quantitative only with provisioning + conditioning |
| **Scaling** | Trivial | State grows with number of reservations | Good — core sees few aggregates |

RFC 2475 §1.4 `[Guidance]` also classifies older/other approaches for contrast: **relative-priority marking** (IPv4 Precedence, 802.5 priority, the default reading of 802.1p), **service marking** (the IPv4 TOS bits of RFC 1349), **label switching** (Frame Relay, ATM, MPLS — per-path state), **IntServ/RSVP**, and **static per-hop classification** (no signaling, but rules configured on every node). DiffServ is described as a refinement of relative-priority marking that adds edge conditioning and a broader PHB concept.

---

## 2. Best Effort

- **Definition** `[Guidance]` (RFC 4594 §1.5.1): the network accepts packets but promises nothing; packets may be lost, reordered, duplicated or delayed. RFC 1633 notes that classic IP forwarding is egalitarian: all packets get the same service, typically strict FIFO.
- **In DiffServ terms** `[Standard-defined]` (RFC 2474 §4.1): best effort is the **Default PHB**. A DS-compliant node **MUST** provide it; its recommended codepoint is `000000`, which **MUST** map to it; unrecognized codepoints **SHOULD** be forwarded as Default (and left unchanged). A sensible implementation must not *starve* it — RFC 2474 suggests reserving minimal buffer/bandwidth for the Default aggregate.
- **Traffic behaviour** `[Guidance]`: best-effort traffic is expected to be *elastic* (TCP-like): the sender slows down when it sees loss or delay.
- **Why it still works** `[Guidance]`: RFC 4594 §1.2 (2006) observed that corporate LANs and ISP backbones are generally lightly utilised (commonly ~10 % at most) so congestion is mostly at access links and oversubscribed points. **Over-provisioning** is therefore the cheapest QoS — until it is not.

```
 Endpoint ---> Switch ---> Router ===(bottleneck)===> WAN
                              |
                    single FIFO queue: voice, video, file transfer all wait together
```

> ⚠️ **Gotcha:** "Best effort" is not "broken". It is the reference service that every other model is defined *against* (e.g., Guaranteed service reclassifies non-conforming packets *to* best effort — RFC 2212).

---

## 3. IntServ — Integrated Services

### 3.1 The idea `[Guidance]` (RFC 1633, 1994)

RFC 1633 introduced the term *integrated services* for a service model that combines **best effort, real-time service and controlled link-sharing**. Its core assumptions:

- Real-time service needs guarantees, and **guarantees cannot be achieved without reservations**; so **resource reservation** and **admission control** are key building blocks.
- Routers must therefore keep **flow-specific state** — a fundamental change to the Internet model (state had traditionally lived only in end systems). To limit the damage to robustness, the state is made **soft** (installed and refreshed, then timed out if not refreshed).
- The same IP network should carry real-time and non-real-time traffic (statistical sharing), using a single service model.

It also answered three objections, which is a good way to remember *why* IntServ was proposed:

| Objection | RFC 1633's answer (paraphrased) |
|---|---|
| "Bandwidth will be infinite" | Not soon, and not everywhere; congested links must still be handled |
| "Simple priority is enough" | Priority is a *mechanism*, not a *service model*; when too many real-time streams share the priority queue, **all** degrade — some users prefer a "busy signal" (admission control) |
| "Applications can adapt" | Adaptation cannot remove the need to bound delay; humans can't interact over multi-second delays |

**Reference implementation framework** (RFC 1633 §2.2): four components — **packet scheduler**, **classifier**, **admission control**, and a **reservation setup protocol**. Together the first three are called *traffic control*.

```
 Host/Router
  +-----------------------------------------------------------+
  |  Routing  |  Reservation setup (RSVP)  |  Management      |  <- background code
  |           |        |  Admission control                   |
  |===========|========|=====================================|
  | Input --> Classifier --> Packet scheduler --> Output      |  <- forwarding path
  +-----------------------------------------------------------+
```

### 3.2 IntServ service classes

| Service | Document | What it promises | Notes |
|---|---|---|---|
| **Guaranteed** | RFC 2212 `[Standard-defined]` | A **firm, mathematically provable bound on end-to-end queueing delay**, and **no queueing loss** for conforming traffic | Needs every hop to support it; bounds **maximum queueing delay only** |
| **Controlled-Load** | RFC 2211 `[Standard-defined]` | Service closely approximating what the flow would get from an **unloaded** network element, kept even when overloaded, via admission control | **No numeric delay or loss targets** are accepted or promised |
| **Best effort** | (default) | Nothing | No admission control |

RFC 1633 originally proposed *guaranteed* and *predictive* real-time services plus controlled link-sharing; the standards-track specifications are Guaranteed (RFC 2212) and Controlled-Load (RFC 2211), with the flow-description objects defined in RFC 2210 (referenced by RFC 2205/2208).

### 3.3 RSVP — the reservation protocol `[Standard-defined]` (RFC 2205)

**Key attributes** (RFC 2205 §1):

- **Simplex** — one direction per reservation.
- **Receiver-oriented** — the *receiver* initiates and maintains the reservation.
- **Soft state** — state is refreshed periodically and times out if not.
- **Not a routing protocol** — it follows whatever path routing chooses.
- Carries QoS and policy parameters that are **opaque to RSVP** (interpreted by traffic control / policy control).
- Transparent through routers that do not support it; supports IPv4 and IPv6.
- Reservation overhead is generally logarithmic rather than linear in the number of receivers of a multicast group.

**Session** = `(DestAddress, ProtocolId [, DstPort])`, agreed out of band before signaling.

**Messages:**

| Type | Name | Direction | Purpose |
|:--:|---|---|---|
| 1 | **Path** | Sender → receiver (follows the data path) | Installs *path state* (previous-hop address, Sender Template, **Sender TSpec**, optional **Adspec**) |
| 2 | **Resv** | Receiver → sender (reverse path, hop by hop) | Installs *reservation state*: **flowspec** (+ filter spec) after admission/policy control |
| 3 | PathErr | Upstream to sender | Reports error in a Path message; does not change state |
| 4 | ResvErr | Downstream to receivers | Reports reservation failure or preemption |
| 5 | PathTear | Downstream | Deletes path (and dependent reservation) state |
| 6 | ResvTear | Upstream | Deletes reservation state |
| 7 | ResvConf | To the requesting receiver | Probabilistic confirmation — **not a guarantee** |

**Walk-through:**

```
 Sender H1 ---> R1 ---> R2 ---> R3 ---> Receiver H2

 (1) Path  ----------->------------>------------>     stores path state
     (Sender TSpec, PHOP)                              (previous hop) at each node
 (2) Resv  <-----------<------------<------------     goes back hop by hop using PHOP
     (flowspec + filter spec)                          each node: admission control
                                                       + policy control -> program
                                                       classifier + scheduler
 (3) Data  ============================================>  gets the reserved QoS
 (4) Path and Resv are REFRESHED periodically (soft state); else state times out
```

1. The session is defined out of band; receiver joins the group (multicast) and the sender starts sending **Path** messages.
2. Each RSVP node records path state, including the **previous hop (PHOP)**, so **Resv** can travel the exact reverse path.
3. The receiver sends **Resv** with a **flow descriptor** = **flowspec** + **filter spec**. The flowspec (Tspec + Rspec) parameterises the **packet scheduler**; the filter spec parameterises the **packet classifier**. Data for the session that matches no filter spec is handled as **best effort**.
4. At each hop the request goes through **admission control** ("enough resources?") and **policy control** ("allowed to reserve?"). Success → classifier and scheduler are programmed; failure → **ResvErr** to the receiver.
5. QoS is enforced at the **upstream end of each link** (where data enters the link), although the request comes from downstream.

**Soft state** `[Standard-defined]`:
- Path and Resv are **idempotent**; a route change is repaired by the next refresh (old state times out).
- RSVP sends messages as plain IP datagrams with no reliability; if the cleanup timeout is **K × the refresh period**, RSVP tolerates **K−1 successive lost messages**.
- **Timers (RFC 2205 §3.7):** the refresh period **R** is chosen locally; the **suggested default is 30 seconds** and should be configurable per interface. The refresh timer is randomised in **[0.5R, 1.5R]** to avoid synchronisation of periodic messages. State lifetime must satisfy **L ≥ (K + 0.5) × 1.5 × R**, with **K = 3** suggested — so with R = 30 s, L ≥ 157.5 s and K−1 = 2 successive lost refreshes are tolerated. A larger K may be needed on lossy hops; the ratio of successive R values must not grow faster than 1 + Slew.Max (0.30). RFC 2961 later added refresh-overhead reduction.
- Routers should give RSVP packets a preferred class so congestion does not cause false teardown.

**Transport details** `[Standard-defined]`: raw IP datagrams with **IP protocol number 46** (UDP encapsulation exists for hosts that cannot send raw IP); **Path, PathTear and ResvConf** are sent with the **Router Alert** IP option; Path messages use the same source/destination addresses as the data, so they cross non-RSVP routers correctly, while Resv is sent hop by hop to the previous RSVP hop's unicast address.

**Reservation styles** (how multiple senders/receivers share a reservation) `[Standard-defined]`:

| Sender selection ↓ / Reservation → | **Distinct** | **Shared** |
|---|---|---|
| **Explicit** (list senders) | **Fixed-Filter (FF)** | **Shared-Explicit (SE)** |
| **Wildcard** (all senders) | — (none defined) | **Wildcard-Filter (WF)** |

- **WF**: one shared "pipe" for all senders (e.g., audio conference where few people talk at once).
- **FF**: a distinct reservation per selected sender (e.g., video).
- **SE**: a shared reservation for an explicit list of senders.
- Styles are mutually incompatible and are not merged with each other.

**Merging** (RFC 2205 §1.4, Fig. 5 — simplified): a node that receives reservation requests from several downstream branches installs and forwards upstream a flowspec that is the **largest** (least upper bound) of them. Example: on one outgoing interface receivers ask **3B** and **2B** → installed **3B**; on another interface a receiver asks **4B** → upstream request = **4B**.

**Errors — the "killer reservation" problem** `[Standard-defined]`: merging can let one receiver's oversized request block another's smaller one. RFC 2205 solves this with **blockade state** created by ResvErr, which excludes the failing flowspec from the merge so a smaller request can still succeed. Existing reservations are always left in place when a *larger* request fails admission control.

> ⚠️ **Gotcha:** A **ResvConf** only says the request was probably installed where it was merged. A receiver can get a confirmation and later a ResvErr. It is not an end-to-end guarantee.

### 3.4 Guaranteed Service — the math `[Standard-defined]` (RFC 2212)

**Traffic description (TSpec)** = token bucket (**r** rate, **b** depth) + peak rate **p** + minimum policed unit **m** + maximum datagram size **M** (rates in bytes/s of IP datagrams). **Request (RSpec)** = reserved rate **R ≥ r** + slack term **S** (µs).

**Element characterization:** two error terms per hop — **C** (rate-dependent, bytes) and **D** (rate-independent, µs) — summed along the path into **Ctot** and **Dtot**. For a datagram WFQ, RFC 2212 gives **C = M** and **D = link MTU ÷ link bandwidth**.

**End-to-end queueing delay bound:**

```
 for p > R >= r :   Dmax = [ (b - M)/R * (p - R)/(p - r) ] + (M + Ctot)/R + Dtot
 for r <= p <= R :  Dmax = (M + Ctot)/R + Dtot
 if p unknown    :  Dmax = b/R + Ctot/R + Dtot
```

Only **queueing** delay is bounded. Guaranteed service does **not** control minimum or average delay and does **not** try to minimise jitter; the path's fixed latency (propagation etc.) must be added separately.

**Worked example** (assumptions are mine, formulas are the RFC's): a G.711 voice flow, 200-byte packets, 50 packets/s → **r = p = 10,000 B/s**, **b = M = 200 B**. Path of **5 hops**, each a WFQ on a 10 Mbps link (1.25 MB/s), MTU 1500 B → per hop **C = 200 B**, **D = 1500 ÷ 1,250,000 = 1.2 ms** → **Ctot = 1000 B, Dtot = 6 ms**.

| Reserved rate R | Formula (p ≤ R) | Queueing-delay bound |
|---|---|---:|
| 10,000 B/s (= r, 80 kbps) | (200 + 1000) ÷ 10,000 + 0.006 | **126 ms** |
| 100,000 B/s (800 kbps) | (200 + 1000) ÷ 100,000 + 0.006 | **18 ms** |

Reserving more than the flow's average rate cuts the bound — RFC 2212's point that applications can control their delay through **R** and **b**.

**Policing vs reshaping** `[Standard-defined]`: policing against the TSpec is done **only at the network edge**; inside the network, elements that want to enforce the profile must **reshape** (buffer until conformant), needing about **b + Csum + Dsum × r** of buffer. Non-conforming traffic is by default treated as **best effort**, not dropped.

### 3.5 Deployment reality of IntServ/RSVP

`[Guidance]` RFC 2208 (1997) — the IETF's own applicability statement for RSVP:

- Processing and storage in a router **increase proportionally with the number of reservations**; many small reservations on a high-bandwidth link "may easily overly tax the routers and is inadvisable".
- Per-flow classification and scheduling may be very difficult on some high-speed interfaces (it cites OC-3 and above).
- Therefore RSVP was not considered appropriate for high-bandwidth backbones at that time; the expectation was **aggregation at the backbone edge**.
- First recommended use: **production intranet or limited ISP environments**.

Aggregation and hybrids:
- **RFC 2475 §1.4:** DiffServ mechanisms can be used to **aggregate IntServ/RSVP state in the core**.
- **RFC 2998** `[Guidance]`: a framework where a DiffServ region is treated as one network element on an IntServ path; RSVP-aware **edge/border routers** do admission control on behalf of the region, either with RSVP-unaware core (static SLAs, aggregate policing) or with per-flow or aggregated RSVP inside.
- **RFC 3175** `[Standard-defined]` (Sept 2001, *Aggregation of RSVP for IPv4 and IPv6 Reservations*): an **aggregator** and **deaggregator** carry many end-to-end reservations inside one aggregate reservation. The aggregator meters each end-to-end reservation against its token bucket, marks the packets with the DSCP for the aggregate, and drops, reshapes or re-marks out-of-compliance packets; service mapping follows RFC 2998.
- **RSVP-TE** `[Standard-defined]` (RFC 3209, Dec 2001, *RSVP-TE: Extensions to RSVP for LSP Tunnels*) is a different application of RSVP — signaling explicitly routed MPLS label-switched paths — and also relies on refresh (hence RFC 2961's refresh reduction).

---

## 4. DiffServ — Differentiated Services

### 4.1 The idea `[Guidance]` (RFC 2475, 1998)

Classify and condition traffic **only at the edge**, encode the result in the **DS field** of each packet, and let every core node apply a small set of **per-hop behaviors** to whole aggregates. RFC 2475 §1.3 lists its requirements, which read like a rejection list of IntServ's costs:

- accommodate many services and provisioning policies; **decouple service from the application**;
- **work with existing applications** without API or host changes (given edge marking);
- **not depend on hop-by-hop application signaling**;
- **avoid per-microflow or per-customer state in core nodes**; use only **aggregated classification** state there;
- need only a **small set of forwarding behaviors**;
- allow **incremental deployment** and reasonable interoperability with non-DS nodes.

```
 Endpoint ---> Access Switch ---> Edge Router (DS boundary / ingress) ====> Core Router (DS interior) ====> Edge Router (egress) ---> Endpoint
    |               |                    |                                        |
  marks?       classify /       classify (MF or BA) -> meter -> mark ->        PHB chosen from the DSCP only:
               pre-mark         police/shape  = "traffic conditioning"          queue, schedule, drop early
```

> 📝 RFC 2475 is **one-directional**: differentiated service is provided in one direction of traffic flow, so the architecture is asymmetric. Upstream and downstream are configured independently.

### 4.2 The DS field `[Standard-defined]` (RFC 2474)

RFC 2474 defines the DS field as the IPv4 **TOS octet** / IPv6 **Traffic Class octet**, superseding the earlier TOS definitions (it obsoletes RFC 1349 and RFC 1455).

```
   bit:   0   1   2   3   4   5   6   7        (bit 0 = leftmost / most significant)
        +---+---+---+---+---+---+---+---+
        |      DSCP (6 bits)    |  CU   |
        +---+---+---+---+---+---+---+---+
        |<-- Precedence -->|                    bits 0-2 = legacy IPv4 Precedence
                    CU = "currently unused" -- later assigned to ECN (RFC 3168, see 12-QoS-ECN)
```

Rules from RFC 2474 §3–4:
- The **DSCP is 6 bits**, giving 64 codepoints. Nodes **MUST select the PHB by matching the whole 6-bit DSCP**, treating it like a table index; the **CU bits MUST be ignored** for PHB selection.
- The **codepoint → PHB mapping MUST be configurable** (except the fixed `xxx000` cases below), and a node must support the logical equivalent of a configurable mapping table. If an operator uses different codepoints from the recommended ones, packets may need **re-marking at administrative boundaries** even where the same PHBs exist on both sides.
- An **unrecognized codepoint** SHOULD be treated as **Default** and its marking should not be changed; such packets **MUST NOT** cause a node to malfunction.
- **ToS byte value = DSCP × 4** (the DSCP sits in the top 6 bits) `[arithmetic]`.

**Codepoint pools** `[Standard-defined]` (RFC 2474 §6) — with decimal equivalents added for CCNP/CCIE convenience:

| Pool | Pattern | Count | Assignment policy | Decimal values (computed) |
|:--:|---|:--:|---|---|
| 1 | `xxxxx0` | 32 | Standards Action (recommended codepoints) | all **even** values: 0, 2, 4 … 62 |
| 2 | `xxxx11` | 16 | Experimental / Local Use | 3, 7, 11 … 63 |
| 3 | `xxxx01` | 16 | Experimental / Local Use (may be used for future standards if Pool 1 runs out) | 1, 5, 9 … 61 |

So a well-known value like **EF = 46 (`101110`)** and every **AF** value are even → Pool 1. (Codepoints for EF/AF/LE and their binary-to-decimal conversion are covered in `5-QoS-PHB-DSCP-Values`; the **LE** (Lower-Effort) PHB uses `000001` = decimal **1**, a **Pool 3** value — assigned by RFC 8622 (Standards Track, 2019), which obsoletes RFC 3662 and updates RFC 4594, replacing that document's earlier recommendation of CS1 for low-priority data.)

**Class Selector codepoints** `[Standard-defined]` (RFC 2474 §4.2): the eight codepoints `xxx000` (decimal **0, 8, 16, 24, 32, 40, 48, 56** = CS0…CS7) exist for backward compatibility with IP Precedence. The PHBs they map to **MUST**:
- yield **at least two independently forwarded classes** of traffic;
- give a packet a probability of timely forwarding **not lower** than that of a lower-order CS codepoint (under reasonable conditions);
- give `11x000` (CS6, CS7) **preferential treatment compared with `000000`**, to preserve the historic use of Precedence 110/111 for routing traffic;
- may be **re-ordered** relative to each other (different CS = different classes).

> 📝 RFC 2474 explicitly allows an operator to map codepoints **irrespective of bits 3–5** for IP-Precedence compatibility — e.g., `011010` mapped to the same PHB as `011000`. That is a **local choice**, not the default, and it is why a device matching only the top 3 bits cannot tell CS3 (24) from AF31 (26).

### 4.3 DiffServ vocabulary `[Guidance]` (RFC 2475 §1.2, 2)

| Term | Meaning |
|---|---|
| **DS domain** | Contiguous set of nodes with a **common service provisioning policy and PHB definitions** (e.g., an enterprise or an ISP) |
| **DS region** | One or more contiguous DS domains; peering domains need a **peering SLA** with a TCA |
| **DS boundary node** | Connects a DS domain to another DS or non-DS domain; acts as **ingress** (traffic entering) and **egress** (traffic leaving) for the two directions |
| **DS interior node** | Connects only to nodes of the same domain |
| **Behavior aggregate (BA)** | All packets with the same DSCP crossing a link in one direction |
| **BA classifier** | Selects packets by **DSCP only** — cheap; used in the core |
| **MF classifier** | Selects by a **combination of fields** (src/dst address, protocol, ports, DS field, incoming interface) — used at the edge |
| **Microflow** | One application-to-application flow (5-tuple) |
| **Traffic profile** | Rate/burst description (e.g., token bucket r, b); packets are in- or out-of-profile |
| **SLA / TCA** | Service Level Agreement (customer ↔ provider); Traffic Conditioning Agreement = the classifier and conditioning rules that apply |
| **PHB** | The externally observable forwarding behavior applied to a BA at a node |
| **PHB group** | PHBs that can only be meaningfully specified together (common constraint, e.g., one queue with several drop priorities) |
| **PDB** | Per-Domain Behavior — the expected treatment of an aggregate across a domain (RFC 3086, referenced by RFC 4594) |

### 4.4 Traffic conditioning — the logical block

`[Guidance]` (RFC 2475 Fig. 1)

```
                         +-------+
              +--------->| Meter |-------+
              |          +-------+       |
              |                          v
  packets ==> Classifier ==========> Marker ==========> Shaper / Dropper ==> out
```

- **Classifier** steers packets of a traffic stream to a conditioner instance.
- **Meter** measures the stream against a profile; its state (in/out of profile) drives the other blocks.
- **Marker** sets or **re-marks** the DSCP.
- **Shaper** delays packets to conform (finite buffer, may still drop); **dropper** discards non-conforming packets (**policing**) — a dropper is a shaper with a zero-size buffer.
- A conditioner need not contain all four; with no profile it may be just classifier + marker.
- Out-of-profile packets may be **queued (shaped), dropped (policed), re-marked** to an inferior codepoint, or forwarded while triggering accounting. Detail on metering/policing/shaping: `8-QoS-Policing`, `9-QoS-Shaping`.

### 4.5 Where conditioning and marking happen `[Guidance]`

- **Sophisticated work at the edge, simple work in the core.** Boundary nodes classify and condition; interior nodes just apply PHBs based on the DSCP.
- **Source domain / near the source:** marking near the source is easier (application preferences are known; classification is simpler before aggregation) — a host may itself act as a DS boundary node (RFC 2475 §2.1.1). If it does not, the nearest DS node acts as the boundary.
- **At a domain boundary:** an ingress node **must assume incoming traffic may not conform** to the TCA and be ready to enforce it; if the upstream domain is not DS-capable, the ingress node must perform *all* required conditioning.
- **Interior nodes are not required to check DSCPs** — they rely on boundary nodes; RFC 2474 §7.1 says boundary nodes **MUST** ensure all entering traffic carries codepoints appropriate to the domain (re-marking if necessary). Any link that cannot be secured against DSCP tampering should be treated as a boundary link.
- Example for voice `[Guidance]` (RFC 4594 §4.1): the Telephony class SHOULD use the EF PHB with guaranteed forwarding resources. At the network edge, marking from **untrusted sources SHOULD be verified with multifield (MF) classification** and those flows **SHOULD be policed** (e.g., a single-rate token bucket with a burst size) to keep telephony within its negotiated bounds; policing is **OPTIONAL** for trusted sources and across SLA peering points, because call admission control governs the traffic there. **Call admission control** is usually performed by a telephony **call server/gatekeeper using signaling** (SIP, H.323, H.248, MEGACO) at access points — RFC 4594 also lists RSVP among the signaling protocols that can negotiate admittance. Details in `6-QoS-Class-Design` and `8-QoS-Policing`.

### 4.6 PHB vs mechanism vs service `[Guidance]`

RFC 2475 §1.1 deliberately keeps four things separate:

| Layer | Example |
|---|---|
| **Service** provided to a traffic aggregate | "Low-delay service for voice, 10 % of the link" |
| **Conditioning functions + PHBs** used to realise it | Police at ingress; PHB = "priority, bounded rate" |
| **Codepoint (DSCP)** that selects the PHB | EF (46) |
| **Node mechanism** implementing the PHB | Strict priority queue, WFQ, WRR, CBQ, drop thresholds … |

The standard defines the **behavior**, not the algorithm: a vendor may use any mechanism that satisfies the PHB (RFC 2474 §5). PHB groups are specified as *groups* because their members share a constraint (e.g., one queue with several drop priorities). RFC 3260 `[Guidance]` clarifies that **AF is a *type* of PHB group and each AF class is an *instance*** of it. A PHB specification must also state the risk of **packet re-ordering** within a microflow when packets of one flow are marked for different PHBs.

### 4.7 What DiffServ does *not* give you

- **No end-to-end guarantee by itself.** Quantitative services need policing/shaping/re-marking plus adequate provisioning (RFC 2475 §2.5). Marking alone is just a *request* for treatment.
- **Per-hop, per-domain.** A non-DS-compliant node inside a domain "may result in unpredictable performance"; on lightly loaded fast links the effect may be negligible, but not where resources are scarce (RFC 2475 §4).
- **Multicast is awkward:** replication consumes more resources and group membership changes the load; RFC 2475 §5 suggests separate peering SLAs/codepoints for multicast.
- **Tunnels/IPsec:** IPsec does not cover the DS field in its integrity check, so it gives **no defence** against DSCP modification; the tunnel egress may act as a DS ingress (RFC 2475 §6.2, RFC 2474 §7.2). Detail in `14-QoS-Tunnels-Overlays`.
- **Theft of service** is the main attack: an adversary sets high-priority DSCPs. Defence = edge conditioning + secure infrastructure.

---

## 5. Comparing the Models and Why DiffServ Dominates in Practice

| Criterion | IntServ / RSVP | DiffServ |
|---|---|---|
| Unit of treatment | Flow | Aggregate (DSCP) |
| Router state | Per reservation, refreshed (soft) | None per flow |
| State growth | With number of concurrent reservations (RFC 2208) | With number of *classes* and edge policies |
| Signaling | RSVP end to end; apps/hosts must support it | None (marking) — works with unmodified applications if marked at the edge |
| Admission control | Per hop, per flow | Provisioning / SLA; call admission by a call server at access points |
| Strongest assurance | Guaranteed: provable queueing-delay bound | Relative or aggregate; depends on provisioning |
| Best fit | Small/edge domains, few high-value flows | Networks of any size; class-based policy |
| Multicast | Designed in from the start (RFC 1633) | Needs special care (RFC 2475 §5) |

**Why DiffServ became the default** — reasoning is grounded in the RFCs, the adoption claim itself is an observation:

1. RFC 2208 said RSVP resource use grows with the number of reservations and per-flow scheduling may be hard on high-speed interfaces → advice to **aggregate at the backbone edge**.
2. RFC 2475's requirements (no per-flow core state, no application changes, no hop-by-hop signaling, small set of PHBs) target exactly those costs.
3. `[Common practice]` In today's enterprise and service-provider networks, QoS policy is almost always **class-based on DSCP**: mark at the edge, PHB in the core, police/shape at boundaries, and **call admission control done by a call server using signaling at access points** (RFC 4594 §4.1) rather than per-flow reservation state in every core router. IntServ ideas live on as **admission control**, RSVP-TE for MPLS paths, RSVP aggregation (RFC 3175) and the hybrid framework of RFC 2998.

> 📝 IntServ's central warning survives inside DiffServ practice: **a priority class without admission control or policing fails for everyone when oversubscribed** (RFC 1633's "simple priority" objection). That is why RFC 4594 has telephony policed at the edge for untrusted sources and admitted by call-level admission control (details in `6-QoS-Class-Design`).

---

## 6. CCIE-Depth Topics

### 6.1 Mixing models — RFC 2998 `[Guidance]`

- Host/edge network speaks IntServ/RSVP; the **DiffServ region is a single hop-like network element**; its **border routers** act as admission-control agents.
- Two realizations: **RSVP-unaware DiffServ core** (border routers just police aggregates per SLA; admission info comes from static SLAs or a dynamic protocol) or **RSVP-aware DiffServ region** (per-flow RSVP inside — more accurate but heavier — or aggregated RSVP inside, or per-flow at the edges and aggregated in the core).
- The mapping from an IntServ service to a DSCP may be the well-known default or overridden by RSVP-aware routers; if marking is done upstream of the DS region, the mapping must be communicated to the marking device.

### 6.2 What is really enforced where

| Function | IntServ | DiffServ |
|---|---|---|
| Classification | MF (5-tuple) in every router on the path, from RSVP filter specs | MF at the edge, **BA (DSCP)** in the core |
| Metering/policing | At the edge against the TSpec; **reshaping** inside | At the edge against the TCA profile |
| Queueing/scheduling | Per-flow (fluid-model approximation, e.g., WFQ with rate R) | Per class; PHB (priority, WRR/WFQ, drop thresholds) |
| Admission | Explicit per flow (RSVP + policy control) | Implicit (provisioning) or external (call server) |

### 6.3 How RSVP limits control-plane growth — and where it still keeps per-flow state

RSVP merges reservations as they travel upstream and stops forwarding a change where merging produces no net state change — the property RFC 2205 calls essential for scaling to large multicast groups. Even so, every RSVP node still keeps **per-sender path state and per-reservation state** for each session; that per-flow state in the core is exactly what DiffServ set out to avoid.

---

## 7. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | Guaranteed service bounds **queueing** delay only | Add propagation/latency separately; it does not reduce jitter as such |
| 2 | Guaranteed service needs **every** hop (routers *and* link technologies) to support it | One unaware hop voids the bound |
| 3 | An RSVP **ResvConf is not a guarantee** | Later ResvErr is still possible |
| 4 | RSVP state disappears if refreshes stop | Refresh (default 30 s) must survive congestion — give RSVP packets a preferred class |
| 5 | Marking a DSCP ≠ getting service | Needs PHB configured on every hop + conditioning + provisioning |
| 6 | DiffServ is per hop **and per direction** | Configure both directions; one bad hop breaks the aggregate |
| 7 | PHB selection uses **all 6 DSCP bits**; CU/ECN bits are ignored | Matching only IP Precedence lumps AF31 (26) with CS3 (24) |
| 8 | Unknown DSCP → Default PHB, not dropped | Good for safety; don't rely on it for policy |
| 9 | Domains with different codepoint→PHB maps must **re-mark at the boundary** | Even when PHBs are identical |
| 10 | IPsec does not protect the DS field | Marking can be spoofed inside/across tunnels; condition at DS boundaries |

---

## 8. Quick Recap

| Concept | One-line answer |
|---|---|
| Three models | Best effort, IntServ (reserve per flow), DiffServ (mark per class) |
| IntServ components | Classifier, packet scheduler, admission control, reservation protocol (RSVP) |
| RSVP messages | Path (1), Resv (2), PathErr (3), ResvErr (4), PathTear (5), ResvTear (6), ResvConf (7) |
| RSVP transport | IP protocol 46, Router Alert on Path/PathTear/ResvConf, soft state, default refresh 30 s |
| RSVP styles | WF (wildcard, shared), FF (explicit, distinct), SE (explicit, shared) |
| Guaranteed vs Controlled-Load | Provable queueing-delay bound vs "like an unloaded network", no numeric targets |
| Guaranteed delay bound (p ≤ R) | (M + Ctot)/R + Dtot |
| DiffServ core principle | No per-flow state; PHB per DSCP; complexity at the edge |
| DS field | 6-bit DSCP + 2 bits (later ECN); ToS byte = DSCP × 4 |
| Pool 1 / 2 / 3 | `xxxxx0` (standards) / `xxxx11` (local) / `xxxx01` (local, future standards) |
| Class Selector | `xxx000` = CS0…CS7 = 0, 8, …, 56; IP-Precedence compatible |
| Default PHB | `000000`; must exist; unknown DSCPs map to it |
| Why DiffServ scales | Core state depends on classes, not flows (RFC 2208 vs RFC 2475 requirements) |

---

## References

**Standards Track**
- RFC 2205 — Resource ReSerVation Protocol (RSVP) Version 1 Functional Specification
- RFC 2211 — Specification of the Controlled-Load Network Element Service
- RFC 2212 — Specification of Guaranteed Quality of Service
- RFC 2474 — Definition of the Differentiated Services Field (DS Field) in the IPv4 and IPv6 Headers

**Informational / Guidance**
- RFC 1633 — Integrated Services in the Internet Architecture: an Overview
- RFC 2208 — RSVP Version 1 Applicability Statement: Some Guidelines on Deployment
- RFC 2475 — An Architecture for Differentiated Services
- RFC 2998 — A Framework for Integrated Services Operation over Diffserv Networks
- RFC 3260 — New Terminology and Clarifications for Diffserv
- RFC 4594 — Configuration Guidelines for DiffServ Service Classes

**Referenced (covered in later notes)**
- RFC 2210 (use of RSVP with IETF Integrated Services), RFC 2961 (RSVP refresh overhead reduction), RFC 3086 (Per-Domain Behaviors), RFC 3168 (ECN), RFC 3175 (RSVP aggregation), RFC 3209 (RSVP-TE)
