## L4S and AccECN

> 💡 **TL;DR:** **L4S** (RFC 9330/9331/9332, 2023) is a new Internet service built on the exact codepoint RFC 8311 freed up in note 12: **ECT(1)**. L4S's core insight, stated directly in RFC 9330: the root cause of queuing delay isn't the queue — it's that classic ("Reno/CUBIC-style") congestion controllers **need** a queue roughly one RTT deep to avoid underutilizing the link. L4S defines a **new class of congestion controllers** that can find available capacity with a **much shallower** queue, marks their packets **ECT(1)** instead of ECT(0), and requires the network to give that traffic **separate queuing treatment** — most commonly a **DualQ Coupled AQM** (RFC 9332): two queues, one for each traffic type, with their marking probabilities mathematically **coupled** so neither type can starve the other. **AccECN** (RFC 9768, 2024) is a separate, related fix: classic RFC 3168 ECN can only report **one** congestion signal per RTT (`12-QoS-ECN` §3.3's "once per event" rule) — far too coarse for L4S, DCTCP, or Congestion Exposure, which all need to know **how many** packets were marked, not just "at least one." AccECN repurposes 3 TCP header bits (including the now-Historic ECN nonce's old bit) to carry an actual **count**.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / Experimental · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / observed deployment status · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 9330 (Architecture), RFC 9331 (ECN Protocol), RFC 9332 (DualQ Coupled AQM) — all **Experimental**, January 2023. RFC 9768 (AccECN) — Standards Track (it formally updates RFC 3168), 2024.

---

## 1. Why This Note Follows Directly from Note 12

`12-QoS-ECN` §7.3 already established the exact mechanism this note depends on: RFC 8311 retired the never-deployed **ECN nonce**, reclassified RFC 3540 to Historic, and explicitly stated this **"enables new experimental use of the ECT(1) codepoint."** L4S is precisely that new experimental use — RFC 9330 itself notes, quoted directly (already surfaced in research): *"the ECT(1) codepoint was previously assigned as the experimental ECN nonce"* before being freed for this purpose. Without RFC 8311, L4S's entire identification mechanism (§3 below) would not have had a codepoint available to use.

```
 RFC 3168 (2001): ECT(0), ECT(1), CE, Not-ECT   -- ECT(1) tied to the nonce experiment
 RFC 8311 (2018): nonce retired -> ECT(1) freed for new experimentation
 RFC 9330-9332 (2023): L4S uses ECT(1) as its traffic identifier
```

---

## 2. The Problem L4S Solves — In RFC 9330's Own Words

`[Standard-defined]` (Experimental) RFC 9330 Abstract, quoted directly: *"L4S is based on the insight that the root cause of queuing delay is in the capacity-seeking congestion controllers of senders, not in the queue itself."* This is a deliberately different diagnosis from `11-QoS-Congestion-Avoidance`'s entire framing (RED, CoDel, PIE) — those algorithms all try to **manage a queue better**; L4S instead targets **why senders build a queue at all**.

**Why classic congestion control needs a deep queue**, reasoned from first principles already established in this series: a classic TCP sender (Reno/CUBIC-style) probes for capacity by increasing its window until it sees loss or an ECN mark, then backs off — this is the same "AIMD" (Additive Increase, Multiplicative Decrease) behaviour underlying the Mathis-model calculation in `1-QoS-Fundamentals` §9.2. To keep a link **fully utilized** during the additive-increase phase (before the next loss/mark event), a classic sender relies on there being a queue roughly the size of the **bandwidth-delay product** (`1-QoS-Fundamentals` §6's BDP concept) — otherwise, in the moments between one congestion event and the next, there wouldn't be enough data in flight to keep the link busy. **That queue, needed for full utilization, is exactly the delay bufferbloat (`11-QoS-Congestion-Avoidance` §2.2) made visible.**

**L4S's proposed fix**: define congestion controllers that don't need that deep a queue to stay efficient — ones that react to **much earlier, more frequent** signals (mark on very shallow queue occupancy) and adjust **smoothly** rather than with a large multiplicative cut. RFC 9330's own summary, quoted directly: *"With the L4S architecture, all Internet applications could (but do not have to) transition away from congestion control algorithms that cause substantial queuing delay and instead adopt a new class of congestion controls that can seek capacity with very little queuing."*

---

## 3. Why Two Traffic Types Can't Share One Queue — The "Semi-Permeable Membrane"

### 3.1 The core architectural constraint

`[Standard-defined]` RFC 9330, quoted directly: *"Latency isolation (network): L4S congestion controls keep queue delay low, whereas Classic congestion controls need a queue of the order of the RTT to avoid underutilization. **One queue cannot have two lengths**; therefore, L4S traffic needs to be isolated in a separate queue (e.g., DualQ) or queues (e.g., FQ)."*

This single sentence is the entire architectural justification for everything else in this note: if L4S and Classic traffic shared one FIFO queue, that queue would have to be **either** shallow (good for L4S's latency, but starves Classic traffic of the buffering it needs to stay efficient) **or** deep (good for Classic, but destroys L4S's whole point). No single queue depth satisfies both — hence "one queue cannot have two lengths."

### 3.2 Coupled congestion notification

`[Standard-defined]` Simply separating the traffic into two queues isn't enough by itself — RFC 9330 also requires **"coupled congestion notification"**, ensuring that a Classic flow and an L4S flow, all else equal, still get a **fair, comparable share** of the link — specifically calibrated against **DCTCP** (Data Center TCP), the reference scalable congestion control RFC 9330 uses as its comparison point. Without this coupling, giving L4S traffic its own low-latency queue could otherwise let it simply out-compete Classic traffic for bandwidth, rather than genuinely coexisting with it.

### 3.3 Three architectural patterns RFC 9330 recognizes

`[Standard-defined]`, quoted and summarized directly:

| Pattern | Description |
|---|---|
| **(a) DualQ Coupled AQM** | One L4S AQM in one queue, coupled to one Classic AQM in a separate queue (the primary, most-deployed pattern — full mechanics in §4) |
| **(b) Per-flow queues** | An instance of a Classic AQM and an L4S AQM in **each** per-flow queue (e.g., combined with FQ-CoDel-style per-flow queuing from `11-QoS-Congestion-Avoidance` §6) |
| **(c) Dual queues with per-flow AQMs, no per-flow queues** | A hybrid approach |

RFC 9330 explicitly states a **specific, structural reason** DualQ is generally preferred over per-flow approaches, quoted directly: *"Per-flow forms of L4S, like FQ-CoDel, are incompatible with full end-to-end encryption of transport layer identifiers for privacy and confidentiality (e.g., IPsec or encrypted VPN tunnels...), because they require packet inspection to access [flow identifiers]... In contrast, the DualQ form of L4S requires no deeper inspection than the IP layer."* This directly connects to `3-QoS-Classification-Trust` §5.2's established fact that encryption hides transport-layer ports from any classifier — DualQ's entire design advantage is that it **never needs those ports**, classifying purely on the ECN field itself (§3.4).

### 3.4 The traffic identifier — RFC 9331's specific requirement

`[Standard-defined]` RFC 9332 §2.3, quoted directly: *"Both the Coupled AQM and DualQ mechanisms need an identifier to distinguish L4S (L) and Classic (C) packets... A separate specification [RFC9331] requires the network to treat the **ECT(1) and CE codepoints** of the ECN field as this identifier."*

```
 ECN field value    Classified as
 ---------------    -------------
 Not-ECT            Classic queue  (no ECN support at all)
 ECT(0)             Classic queue  (ordinary RFC 3168 ECN)
 ECT(1)             L4S queue      (the new identifier)
 CE                 -- ambiguous by the bits alone; a DualQ implementation
                        must track which queue a packet came from before
                        it was marked CE, since CE alone doesn't distinguish
                        L4S-origin from Classic-origin traffic
```

This is precisely why `12-QoS-ECN` §7.3 flagged ECT(1) as the single most consequential thing RFC 8311 changed: **the entire L4S traffic-separation mechanism is built on one previously-idle ECN codepoint.** No new IP header field, no DSCP change, no new classification key beyond what note 3 and note 4 already established as existing — just a different, standards-assigned meaning for a 2-bit combination that used to be reserved for an abandoned experiment.

> ⚠️ **Gotcha (direct continuation of `4-QoS-Marking-Headers` §8.2's "n-bit truncation" pattern):** Because DSCP and ECN are **independent** fields in the same byte (`12-QoS-ECN` §1), an L4S-marked packet (ECT(1)) can carry **any** DSCP — L4S is a congestion-signaling mechanism, not a traffic class in the RFC 4594 sense (`6-QoS-Class-Design`). A network device that only classifies on DSCP, ignoring the ECN bits entirely, will never notice L4S traffic passing through at all — which is by design (DualQ needs no deeper inspection, §3.3), but it also means DSCP-based QoS policy and L4S's queue separation are two **independent, simultaneous** classification decisions on the same packet, not a single unified one.

---

## 4. The DualQ Coupled AQM Mechanism (RFC 9332)

### 4.1 Structure

`[Standard-defined]` RFC 9332 §2, quoted directly: *"A DualQ Coupled AQM implementation MUST utilize two queues, each with an AQM algorithm. The AQM algorithm for the low-latency (L) queue MUST be able [to mark very early, at shallow occupancy]..."*

```
              +-----------------+
 L4S    ----->| L-queue (AQM_L) |---\
 (ECT1)       +-----------------+    \    [ priority-conditional
                                       +--- scheduler, coupled  ]---> out
 Classic ---->| C-queue (AQM_C) |----/    [ marking probabilities ]
 (ECT0,       +-----------------+
  Not-ECT)
```

### 4.2 Scheduling — conditional priority

`[Standard-defined]` RFC 9332, quoted directly: *"a separate queue is provided for L4S traffic, and it is scheduled with **priority** over the Classic queue. Priority is **conditional** to prevent starvation of Classic traffic in certain conditions."* This is deliberately **not** the unconditional strict priority queuing already covered in `10-QoS-Queuing-Scheduling` §4 — RFC 9332 explicitly builds in an anti-starvation safeguard, since unconditional strict priority for L4S would otherwise let a misbehaving or unusually large volume of L4S traffic starve all Classic traffic, exactly the failure mode `10-QoS-Queuing-Scheduling` §4.3 already flagged as strict priority's known weakness.

### 4.3 Why "coupled" marking, not independent marking per queue

`[Standard-defined]` The word "Coupled" in the mechanism's name is load-bearing: the L-queue's AQM and the C-queue's AQM don't compute their drop/mark probabilities **independently**. Instead, the Classic queue's marking probability is **derived from** the L4S queue's congestion signal (mathematically related, not simply copied) — this is what RFC 9332 relies on to guarantee, quoted directly from research above: *"coupled marking ensures that giving priority to L4S traffic still leaves the right amount of spare scheduling time for Classic flows to each get equivalent throughput to DCTCP flows (all other factors, such as RTT, being equal)."* Without this coupling, the two queues' AQMs could drift into giving one traffic type a persistent, unfair advantage over the other.

### 4.4 What the L-queue's own AQM does when there's no Classic traffic

`[Standard-defined]` Quoted directly from research above: *"When there is no Classic traffic, the L4S queue's own AQM comes into play. It starts congestion marking with a very shallow queue, so L4S traffic maintains very low queuing [delay]."* Two named example L-queue AQM implementations are mentioned in the RFC: a RED variant called **Curvy RED**, and a PIE-based DualQ Coupled AQM specified for **Low Latency DOCSIS** (cable-modem) deployments — connecting directly back to `11-QoS-Congestion-Avoidance` §5.3's RFC 8033/8034 material, since PIE's DOCSIS-specific companion (RFC 8034) is exactly the base this L4S cable variant builds on.

---

## 5. AccECN — Why L4S Needs More Than "Once Per RTT"

### 5.1 The specific limitation of classic ECN feedback

`[Standard-defined]` RFC 9768 Abstract, quoted directly: *"ECN was originally specified for TCP in such a way that only one feedback signal can be transmitted per Round-Trip Time (RTT)."* This is exactly the mechanism `12-QoS-ECN` §3.3 documented in detail: the receiver sets ECE on every ACK until CWR arrives, and the sender reacts **once** per event — a design that deliberately treats several CE marks within one RTT as **the same** congestion event, not several distinct ones.

**Why this is a problem specifically for L4S, DCTCP, and Congestion Exposure**, quoted directly: *"More recently defined mechanisms like Congestion Exposure (ConEx), Data Center TCP (DCTCP), or Low Latency, Low Loss, and Scalable Throughput (L4S) need more accurate ECN feedback information whenever **more than one marking is received in one RTT.**"* L4S's whole design (§2) depends on frequent, fine-grained, **early** marking at shallow queue depths — but classic ECN feedback can only tell the sender "yes, at least one packet was marked this RTT," collapsing potentially many distinct marks (which might indicate a rapidly worsening or improving congestion trend) into a single binary bit of information. A scalable congestion controller that needs to respond **proportionally** to how much marking occurred, not just whether any occurred at all, cannot function well on that coarse a signal.

### 5.2 What AccECN actually changes

`[Standard-defined]` RFC 9768 (Briscoe & Kühlewind, 2024) **updates RFC 3168** directly. Its mechanism, quoted precisely: *"Given TCP header space is scarce, it allocates a reserved header bit previously assigned to the ECN-nonce"* — this is the **exact same bit** `12-QoS-ECN` §7.2 already identified as freed by RFC 8311's retirement of the nonce (historically called the **NS**, "Nonce Sum," flag). AccECN repurposes this bit, together with the existing ECE and CWR flags, into a **3-bit field** used to negotiate and then carry accurate feedback.

**The counters maintained**, quoted directly: *"A Data Receiver maintains four counters initialized at the start of the half-connection. Three count the number of arriving payload **bytes** marked CE, ECT(1), and ECT(0) in the IP-ECN field."* This is a fundamentally different feedback model from classic ECN's single sticky ECE bit — AccECN tracks **running byte counts per codepoint**, giving the sender a continuously-updated, precise picture of exactly how much of its traffic has been marked with each codepoint, not just a single "congestion happened" flag.

### 5.3 Negotiation — backward compatible with classic ECN

`[Standard-defined]` Quoted directly: *"The TCP Client signals support for AccECN on the initial SYN of a connection, and the TCP Server signals whether it supports AccECN on the SYN/ACK. The TCP flags on the SYN that the TCP Client uses to signal AccECN support have been carefully chosen so that a TCP Server will interpret them as a request to support the most recent variant of ECN feedback that it supports. Then the TCP Client falls back to the same variant of ECN feedback."* This negotiation design deliberately echoes the exact SYN/SYN-ACK asymmetry principle already verified in `12-QoS-ECN` §3.2 and §8.1 — a specific flag pattern is chosen so that a stack unaware of AccECN, or aware only of classic ECN, or aware of neither, each produces a **distinguishable** response, letting both ends settle on the most capable **common** feedback scheme without ambiguity.

**A structural design decision worth noting**, quoted directly: *"An AccECN TCP Client does not send an AccECN Option on the SYN as SYN option space is limited. The TCP Server sends an AccECN Option on the SYN/ACK, and the TCP Client sends one on the first ACK to **test whether the network path forwards these options correctly.**"* This is a deliberate, defensive design against the exact class of real-world problem `12-QoS-ECN` §4 documented for classic ECN — middleboxes that mangle or strip unfamiliar TCP options — AccECN proactively **probes** for this rather than discovering it only after a connection has silently degraded.

### 5.4 AccECN's relationship to L4S — complementary, not identical

`[Standard-defined]` AccECN is not L4S-specific — RFC 9768 lists ConEx and DCTCP as equally motivating use cases — but it is a **precondition** for L4S's scalable congestion controllers to get the fine-grained feedback they need to actually behave "scalably" over ordinary TCP. Without AccECN (or an equivalent accurate-feedback mechanism), an L4S-capable sender using classic ECN's once-per-RTT feedback would be starved of the very information its improved congestion-response algorithm needs to take advantage of.

---

## 6. CCIE-Depth Topics

### 6.1 Why "shallow marking" doesn't just mean "lower RED thresholds"

It's tempting to read L4S's shallow-queue marking as simply "RED/CoDel with smaller numbers," but RFC 9330's own framing (§2) is structurally different: classic AQM (`11-QoS-Congestion-Avoidance`) tries to find the best **compromise** queue depth for senders that fundamentally need a deep queue to stay efficient. L4S instead changes the **sender's own algorithm** so a deep queue is never needed at all — the AQM's shallow marking threshold is only *safe* to set that low **because** the traffic sharing that queue is specifically the kind that reacts well to frequent, early signals. Applying an L4S-style shallow AQM threshold to ordinary Classic traffic (without the queue separation in §3) would simply starve Classic traffic's throughput, since Classic senders would be triggered into backing off far too early and far too often relative to what their AIMD algorithm expects.

### 6.2 The CE-codepoint ambiguity in a DualQ implementation

Section 3.4 noted that a **CE**-marked packet doesn't, by its bits alone, reveal whether it originated as ECT(1) (L4S) or ECT(0) (Classic) before a router marked it. A DualQ Coupled AQM implementation therefore must make its L/C queue-assignment decision **before** any local marking happens (i.e., classify on the packet's ECT state as it **arrives**, before that same device potentially remarks it to CE) — a subtlety that matters for anyone implementing or auditing a DualQ AQM, since a naive implementation that reclassified **after** its own marking step would lose the L/C distinction for every packet it just marked CE.

### 6.3 Why RFC 9330 treats DCTCP as the fairness reference, not "generic" Classic TCP

RFC 9330's coupling requirement (§3.2, §4.3) is calibrated to give L4S flows throughput comparable to **DCTCP** specifically — not to classic Reno/CUBIC directly. This matters because DCTCP itself is already a **scalable-style** congestion controller (originally designed for data-center ECN marking at very shallow thresholds), so calibrating L4S against DCTCP, rather than against Reno/CUBIC, means the *coupling math* only has to reconcile two **already-compatible-in-spirit** algorithms — the harder compatibility problem (Classic Reno/CUBIC senders sharing capacity fairly with the new L4S/DCTCP-style senders) is what the **Coupled AQM's** cross-queue probability relationship (§4.3) is specifically engineered to solve.

### 6.4 The Experimental status of RFC 9330-9332 matters for citation

Unlike RFC 9768 (AccECN, which is Standards Track and formally updates RFC 3168), RFC 9330, RFC 9331, and RFC 9332 are all published as **Experimental**. This doesn't mean they're untested — RFC 9330 itself references real-world measurement studies and implementations (Low Latency DOCSIS deployments, for instance) — but per the IETF's own document-status conventions, Experimental status signals these specifications are still expected to evolve based on further deployment experience, distinct from a Standards Track RFC's higher bar of IETF consensus review (recall `8-QoS-Policing` §5.5's similar caution about RFC 4115's non-standard IESG note — Experimental and "not a candidate for standard" are related but distinct status markers, and both call for citing the document's exact status rather than treating it as settled protocol).

---

## 7. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | L4S targets the **sender's congestion-control algorithm**, not just the network's queue management | Different problem framing from all of `11-QoS-Congestion-Avoidance`'s AQM algorithms |
| 2 | "One queue cannot have two lengths" — RFC 9330's own stated reason L4S and Classic traffic must be queued separately | Explains why DualQ (or an equivalent per-flow approach) is structurally required, not optional |
| 3 | L4S's identifier is the **ECT(1)** codepoint — the same one RFC 8311 freed from the retired ECN nonce | Directly continues `12-QoS-ECN` §7.3; no new IP header field was needed |
| 4 | DualQ's priority for L4S traffic is **conditional**, not strict | Prevents the exact starvation risk `10-QoS-Queuing-Scheduling` §4.3 already flagged for unconditional strict priority |
| 5 | "Coupled" marking means the two queues' AQMs are **mathematically linked**, not independent | Ensures Classic flows still get a fair (DCTCP-equivalent) share, not simply whatever L4S leaves over |
| 6 | DSCP and the L4S ECT(1) identifier are **independent** classification decisions on the same packet | A DSCP-only classifier will never notice L4S traffic passing through — by design |
| 7 | Classic ECN feedback (`12-QoS-ECN`) can only signal "at least one mark this RTT," never a count | Insufficient for L4S/DCTCP/ConEx, which need proportional, not binary, feedback |
| 8 | AccECN reuses the **same TCP bit** freed by the ECN nonce's retirement (the old NS flag) | A second, transport-layer consequence of RFC 8311, alongside ECT(1)'s reuse at the IP layer |
| 9 | RFC 9330/9331/9332 are **Experimental**; RFC 9768 (AccECN) is Standards Track | Different levels of IETF review — cite accordingly |

---

## 8. Quick Recap

| Concept | One-line answer |
|---|---|
| L4S's core diagnosis | Deep queues exist because classic congestion control needs them to stay efficient — not because the network requires them |
| L4S's fix | A new class of congestion controllers using much shallower, more frequent marking |
| Why separate queues are required | RFC 9330: "one queue cannot have two lengths" |
| L4S traffic identifier | ECT(1) and CE (RFC 9331) — the codepoint freed by RFC 8311's retirement of the ECN nonce |
| Primary mechanism | DualQ Coupled AQM (RFC 9332): two queues, conditional priority for L4S, coupled marking probabilities |
| Fairness reference | DCTCP-equivalent throughput for Classic flows, all else equal |
| Why DualQ over per-flow (FQ) approaches | DualQ needs no deeper inspection than the IP layer — works even with encrypted transport headers |
| AccECN's purpose | More than one ECN feedback signal per RTT — a byte-count per codepoint, not a single sticky bit |
| AccECN's mechanism | Repurposes the old ECN-nonce TCP bit (NS) plus ECE/CWR into a 3-bit negotiated feedback scheme |
| Standards status | L4S RFCs (9330-9332): Experimental. AccECN (RFC 9768): Standards Track, updates RFC 3168 |

---

## References

**Experimental**
- RFC 9330 — Low Latency, Low Loss, and Scalable Throughput (L4S) Internet Service: Architecture
- RFC 9331 — The Explicit Congestion Notification (ECN) Protocol for L4S (ECT(1)/CE as the traffic identifier)
- RFC 9332 — Dual-Queue Coupled Active Queue Management (AQM) for L4S

**Standards Track**
- RFC 9768 — More Accurate Explicit Congestion Notification (AccECN) Feedback in TCP (updates RFC 3168)

**Referenced (background and what feeds into this note)**
- `12-QoS-ECN` (RFC 3168's codepoints and feedback loop; RFC 8311's retirement of the ECN nonce and freeing of ECT(1))
- `1-QoS-Fundamentals` (bandwidth-delay product; the Mathis-model reasoning behind why classic TCP needs a deep queue)
- `11-QoS-Congestion-Avoidance` (RED, CoDel, PIE — the AQM algorithms L4S's L-queue and C-queue build on; RFC 8034's DOCSIS/PIE work reused by an L4S DualQ variant)
- `10-QoS-Queuing-Scheduling` (strict priority and its starvation risk — the exact risk DualQ's conditional priority avoids)
- `3-QoS-Classification-Trust` (why encryption hides transport-layer identifiers — the structural reason DualQ is preferred over per-flow/FQ-based L4S)
