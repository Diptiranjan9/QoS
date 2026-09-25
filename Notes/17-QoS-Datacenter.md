## QoS in the Data Center — PFC, ETS, DCBX, QCN, and RoCEv2

> 💡 **TL;DR:** Data-center Ethernet added a genuinely new capability the rest of this series hasn't needed: **lossless** delivery for specific traffic classes, on a shared Ethernet fabric. Three IEEE amendments, collectively branded **Data Center Bridging (DCB)**, make this possible: **PFC** (802.1Qbb) pauses traffic **per-priority** rather than pausing the whole link, so a lossless class (like storage) and a lossy class (like ordinary LAN traffic) can share one physical link without one blocking the other. **ETS** (802.1Qaz) guarantees each traffic class a **minimum bandwidth share** while still letting idle classes lend their unused capacity to busy ones — `10-QoS-Queuing-Scheduling`'s DRR/WFQ concepts, applied at the DCB layer. **DCBX** (also 802.1Qaz, an LLDP extension) lets neighboring devices **automatically discover and agree** on PFC/ETS configuration rather than needing it manually matched on both ends. **QCN** (802.1Qau) was IEEE's attempt at true end-to-end, hop-by-hop congestion control for this environment — it saw very limited real-world deployment. What actually succeeded is **DCQCN**, an industry-developed (Microsoft/Mellanox, 2015) scheme combining **ECN** (`12-QoS-ECN`) with QCN-style rate-limiting logic, purpose-built to make **RoCEv2** (RDMA over Converged Ethernet) perform well at scale — precisely because PFC alone, while it prevents packet loss, has serious side effects (head-of-line blocking, unfairness, even deadlock) when used as the *only* congestion-control mechanism.

> 🏷️ **Tags:** `[Standard-defined]` IEEE 802.1 standard · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / industry-developed mechanism / academic literature · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** IEEE 802.1Qbb (PFC), IEEE 802.1Qaz (ETS + DCBX) — ratified IEEE standards (2011). IEEE 802.1Qau (QCN) — ratified IEEE standard (2010), but with acknowledged minimal production deployment. DCQCN — an **academic paper** (Zhu et al., ACM SIGCOMM 2015), not a formal standard, though widely implemented in hardware. RoCEv2 itself is defined by the **InfiniBand Trade Association** (not IEEE or IETF).

---

## 1. Why Data Centers Need Something Beyond Everything Covered So Far

Every mechanism in notes 3–16 assumes **loss is an acceptable, even necessary, congestion signal** — RED, CoDel, and PIE (`11-QoS-Congestion-Avoidance`) all work by dropping or marking packets precisely *because* TCP is expected to interpret that as "slow down." Certain data-center workloads break this assumption entirely: **storage protocols carried over Ethernet** (historically Fibre Channel over Ethernet, FCoE) and **RDMA** (Remote Direct Memory Access, covered in §5) were originally designed for transports — Fibre Channel, InfiniBand — that are **inherently lossless**, and simply do not have a well-behaved response to an ordinary dropped frame the way TCP does. Running these protocols over ordinary Ethernet, sharing the same physical links as loss-tolerant LAN traffic, requires a way to make **specific priority classes** on that Ethernet link behave losslessly, while leaving other classes exactly as before.

```
 Same physical link, two different requirements:
   Priority 3 (storage/RDMA): MUST NOT drop frames under congestion
   Priority 0 (ordinary LAN): fine with drops -- TCP handles it (notes 8-11)
```

This is precisely the problem **Data Center Bridging (DCB)** — the umbrella term for PFC + ETS + DCBX together — exists to solve.

---

## 2. PFC — Priority-based Flow Control (IEEE 802.1Qbb)

### 2.1 The mechanism it extends

`[Standard-defined]` PFC builds directly on the older IEEE 802.3x **PAUSE** frame mechanism, which already existed for ordinary Ethernet flow control. Standard 802.3x PAUSE, when a receiving interface is congested, sends a single PAUSE frame that suspends **all** traffic on the link, for **all** priorities, for a specified time. PFC's specific improvement, confirmed consistently across research: the PFC pause frame can encode **a different pause time for each of the eight 802.1p priority values** (`4-QoS-Marking-Headers` §4) independently, rather than pausing everything.

```
 802.3x PAUSE:  [ STOP EVERYTHING on this link ]

 PFC:           [ STOP priority 3 only ] -- priorities 0,1,2,4,5,6,7 continue flowing normally
```

### 2.2 The mechanics — XOFF, XON, and quanta

`[Standard-defined]`, cross-verified across multiple independent sources: a PFC-capable receiver, when its buffer for a specific priority approaches exhaustion, sends a PFC frame containing an **8-bit priority-enable vector** (which of the 8 priorities this pause frame applies to) and, for each enabled priority, a **pause time measured in quanta** — one quantum being defined as the time to transmit **512 bits**. A pause time of **0** for a given priority signals **unpause (XON)**; any nonzero value signals **pause (XOFF)** for that duration. The sender, on receiving this frame, halts transmission of frames at the specified priority/priorities until either the pause timer expires or an explicit XON (zero-quanta) frame arrives.

### 2.3 Why PFC alone is only half the answer

`[Common practice]`, directly and explicitly confirmed by the RoCEv2 research in §5 below: PFC succeeds at its **narrow** goal — preventing frame loss due to buffer overflow, for the priorities it's enabled on — but it is a purely **hop-by-hop, link-local** mechanism with **no end-to-end signaling** at all. It has no concept of *why* a queue is filling up beyond "my immediate downstream neighbor told me to stop," and no mechanism to tell an *original sender several hops away* to slow down proactively. This limitation is exactly what motivates §4's QCN and §5's DCQCN — both exist specifically to add an **end-to-end** signal on top of PFC's purely local, reactive pause mechanism.

> ⚠️ **Gotcha (a real, documented failure mode, confirmed directly in the RoCEv2 research below):** Because PFC pauses are purely hop-by-hop and can cascade backward through a multi-hop topology (a paused switch's own queues fill, so it pauses its own upstream neighbors, and so on), large-scale RoCEv2 deployments have documented **PFC-induced deadlock** as an actual production issue — a cyclic dependency of pauses that never resolves on its own. This is not a hypothetical risk; it is explicitly named as a challenge that had to be engineered around in a large-scale operational deployment.

---

## 3. ETS — Enhanced Transmission Selection (IEEE 802.1Qaz)

### 3.1 The problem ETS solves

`[Standard-defined]`, confirmed directly: pure strict-priority scheduling among the eight 802.1p priorities (as `10-QoS-Queuing-Scheduling` §4 already established generally) means a busy high-priority class can **starve** lower-priority classes completely — exactly the same starvation risk already flagged in that note, now appearing in the specific context of a converged DCB link carrying storage, RDMA, and ordinary LAN traffic together, where starving any one of those categories is operationally unacceptable.

### 3.2 The mechanism — Priority Groups and bandwidth percentages

`[Standard-defined]`, verified precisely: ETS introduces the **Priority Group ID (PGID)**, a 4-bit field allowing one or more of the eight 802.1p priorities to be assigned to a shared group. Sixteen PGID values exist; **PGID 15 is a special "No Bandwidth Limit" value** reserved specifically for priorities that should continue to receive **strict priority** treatment (exempted from ETS's bandwidth-sharing scheme entirely) — PGIDs 8–14 are reserved, leaving 0–7 as the configurable bandwidth-sharing groups. Each configured (0–7) Priority Group is assigned a **percentage of available link bandwidth**, and — critically — those percentages **must sum to 100**.

```
 Example ETS configuration (illustrative percentages, not a standard requirement):

  PGID 15 (strict priority, no BW limit): Priority 5 (e.g., voice-equivalent, always serviced first)
  PGID 0 (40% of remaining bandwidth):    Priority 3 (storage/RDMA, PFC-enabled, lossless)
  PGID 1 (60% of remaining bandwidth):    Priorities 0,1,2,4,6,7 (ordinary LAN traffic)
```

### 3.3 What happens to unused bandwidth

`[Standard-defined]`, confirmed directly: *"If a traffic class doesn't use its allocated bandwidth, ETS allows other traffic classes to use the available bandwidth that the traffic class is not using."* This is precisely the DRR/WFQ fairness property already established in detail in `10-QoS-Queuing-Scheduling` §5–6 — ETS is, structurally, **that same scheduling concept**, specified for the DCB environment: a Priority Group's percentage is a **guaranteed minimum**, not a hard ceiling, and idle capacity is redistributed to whichever groups are actually backlogged, exactly like DRR's active-list mechanism skipping empty queues (`10-QoS-Queuing-Scheduling` §5.2).

> 📝 One independent academic source, cross-verified, describes ETS precisely this way: *"a hierarchical scheduler that combines static priority scheduling and a bandwidth-sharing algorithm (such as Weighted Round Robin or Deficit Round Robin)"* — this is a direct, explicit confirmation that ETS is not a new scheduling *theory*, but a specific **application** of the exact algorithms already fully covered in note 10, layered with an additional strict-priority tier (PGID 15) on top.

### 3.4 DCBX — automatic configuration exchange

`[Standard-defined]`, also part of the 802.1Qaz standard, confirmed directly: **Data Center Bridging Capability Exchange (DCBX)** is explicitly built as **"an extension of Link Layer Discovery Protocol (LLDP)"** (`3-QoS-Classification-Trust` §6.4 already introduced LLDP-MED as a comparable capability-advertisement mechanism for voice endpoints — DCBX applies the identical underlying idea to DCB parameters between switches and adapters). DCBX carries specific TLVs for **Priority Groups (ETS), PFC, and Applications** — allowing two directly-connected DCB-capable devices to **discover each other's configuration and confirm they agree**, rather than requiring an administrator to manually and separately configure matching PFC/ETS settings on both ends of every link and hope nothing drifts out of sync. Confirmed directly: if multiple, conflicting peer configurations are somehow discovered on a link, "the peer's TLVs should be ignored until the multiple peers condition is resolved" — a safety behaviour preventing DCBX from acting on ambiguous or conflicting negotiation results.

> ⚠️ **Gotcha:** DCBX's Application TLV specifically advertises **which 802.1p priority value a given application (e.g., FCoE, iSCSI) should use** — directly analogous to LLDP-MED's Network Policy TLV telling an IP phone which VLAN/DSCP to mark (`3-QoS-Classification-Trust` §6.4). The same caution from that note applies here too: DCBX **advertises** a configuration; it does not, by itself, enforce that an endpoint actually complies with it — an endpoint that ignores the advertised priority and marks its own traffic differently is a classification/trust problem, not a DCBX failure.

---

## 4. QCN — Quantized Congestion Notification (IEEE 802.1Qau)

### 4.1 What QCN was designed to do

`[Standard-defined]` QCN's IEEE PAR (Project Authorization Request), quoted directly: *"This standard specifies protocols, procedures and managed objects that support congestion management of long-lived data flows within network domains of limited bandwidth delay product. This is achieved by enabling bridges to signal congestion information to end stations capable of transmission rate limiting to avoid frame loss."* This is precisely the **end-to-end signal PFC lacks** (§2.3) — QCN's explicit goal was to let a congested switch, deep inside the fabric, tell the **original sending end station** to slow down, well before that congestion cascades backward as a chain of PFC pauses.

### 4.2 The mechanism — Congestion Points and Reaction Points

`[Standard-defined]`, cross-verified across multiple independent sources: QCN defines a **Congestion Point (CP)** — a switch or end-station port function monitoring a queue — and a **Reaction Point (RP)** — the traffic source, equipped with a rate limiter. The CP **samples** outgoing frames (nominally at a **1% sampling rate**, rising to as much as **10%** under severe congestion, per the confirmed research) and, for sampled frames, computes a **quantized feedback value** describing the severity of congestion — this quantized value is what gives QCN its name. If that feedback indicates genuine congestion, the CP sends a **Congestion Notification Message (CNM)** back to the sampled frame's source; the RP, on receiving a CNM, **reduces its transmission rate** for that flow, later probing to recover bandwidth if congestion eases.

```
                     [ CP: congested switch queue ]
                              | samples 1-10% of frames, computes severity
                              v
   [ RP: original sender ] <--- CNM (quantized congestion feedback) ---
        |
        reduces its own send rate for this flow
```

### 4.3 Why QCN saw very limited real-world adoption

`[Common practice]`, directly and explicitly confirmed: an industry source tracking the standard's own history, quoted directly: *"IEEE working group changed the congestion notification details a few times, the last time from simple Backward Congestion Notification (BCN) to Quantized Congestion Notification (QCN)... Taking all these details into account, I don't expect to see QCN deployed in production networks any time soon (if at all)."* This is a candid, contemporaneous industry assessment (not a retrospective judgment invented for this note) — and later academic literature on QCN independently documents specific technical weaknesses discovered after ratification, including **rate unfairness between flows sharing a bottleneck** (confirmed directly by multiple follow-on academic papers proposing "Fair QCN" and similar corrections) and QCN's probabilistic sampling meaning that, as flow counts scale up, some flows can go unnotified even during genuine congestion. **QCN did not become the dominant data-center congestion-control mechanism in practice.**

---

## 5. What Actually Succeeded — RoCEv2 and DCQCN

### 5.1 RoCEv2 and why it needs congestion control at all

`[Common practice]`, confirmed directly across multiple independent, cross-verifying sources: **RDMA (Remote Direct Memory Access)**, originally an InfiniBand-native concept, allows a NIC to transfer data directly into a remote host's pre-registered memory, **bypassing the host's own networking stack and CPU** entirely for the data path — confirmed directly: standard TCP/IP stacks were measured consuming **"over 20% CPU cycles across all cores"** at large message sizes, a cost RDMA is specifically designed to eliminate. **RoCEv2** ("RDMA over Converged Ethernet, version 2") is the InfiniBand Trade Association's specification for carrying this RDMA traffic over ordinary, **routable** IP/UDP networks rather than requiring dedicated InfiniBand hardware — confirmed directly, it uses a **fixed, well-known destination UDP port (4791)**, with a **randomized source UDP port per Queue Pair (QP)** specifically so that ordinary five-tuple ECMP hashing (standard multi-path routing) spreads different RDMA flows across different paths, exactly the same load-distribution principle already familiar from ordinary IP routing.

### 5.2 Why PFC alone is not enough for RoCEv2

Directly confirmed, and consistent with §2.3's general limitation: RoCEv2 depends on PFC to guarantee the lossless delivery RDMA's transport protocol was never designed to recover from on its own — but production deployment experience (confirmed directly, from large-scale operational reporting) documented that **"PFC can lead to poor application performance due to problems like head-of-line blocking and unfairness,"** and, as already flagged in §2.3's gotcha, genuine **PFC-induced deadlock** in production. Relying on PFC as the *sole* congestion-management mechanism for RoCEv2 at scale was found to be operationally inadequate.

### 5.3 DCQCN — combining ECN with QCN-style end-host rate control

`[Common practice — academic, but very widely implemented in hardware]` **DCQCN** (Data Center Quantized Congestion Notification), introduced by Zhu et al. at ACM SIGCOMM 2015, is the mechanism that emerged to solve exactly this gap. Its own stated design choice, quoted directly from the confirmed research: *"We use DCQCN... because it directly reacts to the queue lengths at the intermediate switches and ECN is well supported by all the switches we use. Small queue lengths reduce the PFC generation"* — i.e., DCQCN's explicit goal is to make **actual PFC pauses rare**, by reacting to congestion **earlier**, via ECN, so the lossless PFC mechanism is only ever needed as a last-resort backstop rather than the primary congestion signal.

**The mechanism, confirmed directly**: DCQCN builds on **pure rate control** (not a TCP-style congestion window) and explicitly **"leverages IEEE 802.1Qau Quantized Congestion Notification (QCN)"** — reusing QCN's quantized-feedback *concept* — but replaces QCN's switch-side sampling-and-CNM mechanism with the **already-standardized ECN marking** already fully specified in `12-QoS-ECN`: switches mark packets (using the same CE codepoint mechanism from note 12) as their queues build, **before** any drop or PFC pause would be needed; the RDMA receiver, on seeing ECN-marked packets, generates a **Congestion Notification Packet (CNP)** back to the sender; the sender's NIC hardware, on receiving a CNP, cuts its per-flow (per Queue Pair) rate — directly analogous to `12-QoS-ECN`'s TCP ECE/CWR feedback loop, but implemented in NIC hardware for RDMA rather than in a TCP/IP software stack.

```
 [ Switch queue congested ] --marks CE (RFC 3168 mechanism, note 12)--> [ RDMA receiver ]
                                                                                |
                                                                     generates CNP, sends back
                                                                                |
                                                                                v
                                                                [ RDMA sender: NIC hardware
                                                                  cuts per-QP rate ]
                                                                          |
                                            (only if this ALSO fails to relieve congestion
                                             in time does PFC's hop-by-hop pause ever trigger
                                             -- DCQCN's whole design goal is to make this rare)
```

**Why DCQCN chose ECN over reusing QCN's own switch-side mechanism directly** — confirmed directly in the reasoning already quoted above: ECN was **already well-supported** in existing commodity switch silicon (because of its prior, independent adoption for ordinary IP/TCP traffic, `12-QoS-ECN`), while QCN's own native switch-side sampling mechanism saw far less hardware adoption (consistent with §4.3's adoption problem) — DCQCN's designers pragmatically reused the **standard, already-deployed** ECN marking mechanism rather than depending on QCN's less-adopted native switch feature, while still keeping QCN's **end-host rate-control philosophy** (react to a quantized feedback signal by adjusting a per-flow send rate, rather than a TCP-style window).

### 5.4 The overall data-center congestion-management stack

`[Common practice]`, synthesizing everything verified in this note into one coherent picture:

| Layer | Mechanism | Role |
|---|---|---|
| Link-level, last-resort loss prevention | **PFC** (802.1Qbb) | Hop-by-hop pause; guarantees no drops for lossless priorities, but reactive and purely local |
| Bandwidth sharing among classes | **ETS** (802.1Qaz) | Guaranteed minimum share per Priority Group, DRR/WFQ-style redistribution of idle capacity |
| Configuration discovery | **DCBX** (802.1Qaz, an LLDP extension) | Automatic negotiation of PFC/ETS/Application settings between neighbors |
| End-to-end proactive signal (IEEE's own attempt) | **QCN** (802.1Qau) | Switch-sampled feedback to source rate limiters — technically sound, but saw very limited production deployment |
| End-to-end proactive signal (what actually succeeded at scale) | **DCQCN** (industry-developed, SIGCOMM 2015) | ECN-based congestion signaling (reusing `12-QoS-ECN`'s standard mechanism) + QCN-style end-host rate-limiting logic, specifically to keep PFC pauses rare |

---

## 6. CCIE-Depth Topics

### 6.1 Why DCQCN's design is a clean illustration of "reuse what's already deployed"

Section 5.3's reasoning is worth generalizing: DCQCN's designers had **two** existing, standardized building blocks available — QCN's native switch-sampling mechanism (§4) and ECN's already-widely-deployed marking mechanism (`12-QoS-ECN`) — and chose to combine the **philosophy** of one (QCN's end-host rate-control response model) with the **already-adopted mechanism** of the other (ECN marking), rather than waiting for QCN's own native mechanism to see wider silicon support. This is a specific, real-world instance of a general engineering principle: a new protocol's success often depends less on inventing an entirely novel mechanism than on **correctly identifying which existing, already-deployed pieces can be recombined** to solve the actual operational problem.

### 6.2 The parallel between PFC-induced deadlock and general priority-queuing starvation

`10-QoS-Queuing-Scheduling` §4.3 already established that **strict priority queuing has no built-in anti-starvation mechanism**, requiring external policing or admission control. PFC's deadlock risk (§2.3) is a **structurally related but distinct** failure mode: rather than one *priority class* starving another on a single device (the ordinary strict-priority starvation problem), PFC's cascading pause behaviour can create a **cyclic dependency across multiple devices** — device A pauses device B, whose queues then fill and cause it to pause device C, which (in a poorly designed or unlucky topology) can eventually cascade back around to pause device A again, with no device able to make progress. Recognizing these as related-but-distinct risks (single-device starvation vs. multi-device cyclic pause deadlock) matters because the fixes are different: starvation is fixed by policing/admission control (note 10); PFC deadlock is fixed by careful **topology design, buffer sizing, and exactly the kind of proactive, ECN-based rate control DCQCN provides** to keep pauses rare enough that the cyclic condition rarely, if ever, arises.

### 6.3 ETS's PGID 15 as a callback to note 10's hierarchical model

Recall `7-QoS-Policy-Model` §6 and `10-QoS-Queuing-Scheduling` §7: hierarchical scheduling is nothing more than a parent scheduler's output feeding a child scheduler's input, and a common real-world pattern combines **strict priority for one tier** with a **round-robin-family algorithm for everything else**. ETS's PGID 15 (unconditional strict priority, exempt from the 0–7 percentage-based groups) alongside PGIDs 0–7 (DRR/WFQ-style bandwidth sharing) is a **direct, standards-defined instance** of exactly that same hierarchical pattern — confirming, once again, that the DCB-specific standards in this note are applications of general QoS principles already fully derived earlier in the series, not a separate body of theory.

---

## 7. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | PFC pauses **per-priority**, not per-link — the entire point that distinguishes it from ordinary 802.3x PAUSE | Lets lossless and lossy traffic classes coexist on one physical link |
| 2 | PFC is purely **hop-by-hop and reactive** — it has no end-to-end signaling of its own | This is precisely why QCN and DCQCN had to be developed as separate, additional mechanisms |
| 3 | PFC-induced **deadlock** is a real, documented production failure mode, not a theoretical concern | Confirmed directly in large-scale operational RoCEv2 deployment reporting |
| 4 | ETS's Priority Group percentages are **guaranteed minimums**, not hard ceilings | Idle capacity is redistributed — structurally identical to DRR/WFQ's fairness property from note 10 |
| 5 | DCBX **advertises** configuration; it doesn't enforce endpoint compliance | Same caution already established for LLDP-MED in note 3 |
| 6 | QCN was ratified as an IEEE standard but saw **very limited production deployment** | Don't assume standardization implies widespread real-world adoption |
| 7 | What succeeded in practice (**DCQCN**) is an industry paper, not a formal IEEE/IETF standard | Widely implemented in hardware despite its non-standard-body origin |
| 8 | DCQCN's whole design goal is to make **actual PFC pauses rare**, not to replace PFC | PFC remains the lossless backstop; ECN-based DCQCN is the proactive, earlier-acting layer above it |
| 9 | RoCEv2's fixed destination UDP port (4791) with randomized source ports exists specifically for **ECMP compatibility** | The same load-distribution principle as ordinary IP multi-path routing, applied to RDMA traffic |

---

## 8. Quick Recap

| Concept | One-line answer |
|---|---|
| Why data centers need something new | Storage/RDMA protocols need lossless delivery; ordinary Ethernet is loss-tolerant by default |
| PFC (802.1Qbb) | Per-priority pause frames — lets lossless and lossy classes share one link |
| PFC's limitation | Hop-by-hop, reactive only, no end-to-end signal — and can cause cascading deadlock |
| ETS (802.1Qaz) | Guaranteed minimum bandwidth per Priority Group, with idle capacity redistributed — DRR/WFQ applied to DCB |
| DCBX | LLDP extension for automatic PFC/ETS/Application configuration discovery between neighbors |
| QCN (802.1Qau) | IEEE's own end-to-end congestion signal (Congestion Points sample, notify Reaction Points) — saw limited real-world adoption |
| RoCEv2 | InfiniBand Trade Association's spec for routable RDMA over UDP/IP; fixed destination port 4791 |
| DCQCN | Industry-developed (SIGCOMM 2015) mechanism combining standard ECN marking with QCN-style end-host rate control — what actually succeeded at scale |
| The overall stack | PFC (lossless backstop) + ETS (bandwidth sharing) + DCBX (auto-config) + DCQCN (proactive, ECN-based rate control to keep PFC pauses rare) |

---

## References

**IEEE Standards**
- IEEE 802.1Qbb — Priority-based Flow Control (PFC)
- IEEE 802.1Qaz — Enhanced Transmission Selection (ETS) and Data Center Bridging Exchange (DCBX)
- IEEE 802.1Qau — Congestion Notification (QCN)
- IEEE 802.3x — the original Ethernet PAUSE mechanism PFC extends

**Industry / Academic (not formal standards, but widely implemented)**
- Y. Zhu, H. Eran, D. Firestone, C. Guo, M. Lipshteyn, et al. — "Congestion Control for Large-Scale RDMA Deployments" (DCQCN), ACM SIGCOMM 2015
- Follow-on Microsoft/academic reporting on RoCEv2 production deployment experience (head-of-line blocking, unfairness, PFC-induced deadlock)

**Specification body (not IEEE/IETF)**
- InfiniBand Trade Association — RoCEv2 (Annex A17 to the InfiniBand Architecture Specification)

**Referenced (background and what feeds into this note)**
- `4-QoS-Marking-Headers` (802.1p/PCP priority field, reused directly by PFC and ETS)
- `10-QoS-Queuing-Scheduling` (DRR/WFQ and strict-priority starvation — the general theory ETS and PFC apply in the DCB context)
- `12-QoS-ECN` (RFC 3168's CE marking mechanism, reused directly by DCQCN)
- `3-QoS-Classification-Trust` (LLDP-MED, the direct conceptual precedent for DCBX's capability-advertisement approach)
