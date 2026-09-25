## QoS Congestion Avoidance — Tail Drop, RED/WRED, and Modern AQM

> 💡 **TL;DR:** **Tail drop** — the default behaviour of any queue that just fills up and then discards new arrivals — causes **global synchronization**: many TCP flows see loss at the same moment, all cut their windows together, and the link alternately empties and refills in a wave. **RED** (Floyd & Jacobson, 1993, later an IETF-documented AQM in RFC 2309/RFC 7567) fixes this by dropping **before** the queue is full, **randomly**, with probability rising as the **average** queue size grows — so only a few flows back off at a time. RFC 3290's **Algorithmic Dropper** (`7-QoS-Policy-Model` §2.4) is where this lives in the vendor-neutral model. RFC 7567 (2015) **formally retracted** RED as the recommended default AQM, in favour of newer, delay-based algorithms — **CoDel** (RFC 8289) and **PIE** (RFC 8033) — designed specifically to fight **bufferbloat**: the problem of oversized buffers adding seconds of latency instead of preventing loss. **FQ-CoDel** (RFC 8290) combines per-flow fairness (DRR, `10-QoS-Queuing-Scheduling` §5.2) with CoDel, so one greedy flow's queue gets managed without punishing everyone else's.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or BCP · `[Common practice]` engineering practice / academic literature · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 2309 (obsolete) → **RFC 7567** — Best Current Practice (BCP 197). RFC 8289 (CoDel), RFC 8033 (PIE), RFC 8290 (FQ-CoDel) — all **Experimental**. The original RED paper (Floyd & Jacobson, 1993) is an **academic paper**, not an RFC.

---

## 1. Where This Fits — Recap from Note 7

From `7-QoS-Policy-Model` §2.4: RFC 3290 defines the **Algorithmic Dropper** as an element that watches a **queue's depth** (not a per-packet meter result) and decides to drop — or, with ECN, mark — packets *before* the queue physically overflows. This entire note is about what discipline that Algorithmic Dropper runs. RFC 3290's own worked example named the discipline `RED`; this note explains RED itself, why it was later un-recommended, and what replaced it.

```
                    +-------------------+
  arriving  ------->|  Queue (depth d)  |-------> departing (scheduled)
  packets           +-------------------+
                              |
                     [ Algorithmic Dropper ]  <-- watches d (or, in modern AQM, watches
                       Discipline: RED /            how LONG a packet has been waiting)
                       CoDel / PIE / FQ-CoDel        and drops/marks BEFORE d hits capacity
```

---

## 2. Tail Drop — the Default, and Its Problem

`[Guidance]` **Tail drop** is simply: the queue has a maximum size; when it's full, every new arriving packet is discarded, until room opens up. It requires no configuration and is the behaviour of a queue with no Algorithmic Dropper attached at all.

### 2.1 Global synchronization

`[Common practice — well-documented phenomenon]` The problem: when a link is shared by many TCP flows and the shared queue fills, tail drop discards packets from **whichever flows happen to be sending when the queue is full** — which, under sustained congestion, tends to be **most or all of them simultaneously**. Every affected TCP flow interprets its loss as a congestion signal and cuts its window at the same moment. The link's aggregate throughput then **oscillates**: a synchronized collective slowdown empties the queue, all the flows ramp back up together, the queue fills again, and the cycle repeats — instead of a smooth, stable aggregate rate. This oscillation, not the isolated loss event itself, is the specific failure tail drop is criticized for.

### 2.2 The bufferbloat problem — a second, distinct issue

`[Guidance]` A **larger** buffer doesn't fix tail drop's oscillation — it makes a **different** problem worse. RFC 7567 (§1, and its citation of Gettys' work — see §5.1 below) explains that oversized buffers let a queue grow for a long time **before** tail drop ever triggers, and every one of those queued packets is sitting there adding **delay**. TCP itself won't slow down until it sees a loss or an ECN mark — so a large-enough buffer can let queuing delay climb to hundreds of milliseconds or more, entirely masked from TCP's own congestion control, before the buffer finally fills and drops something. This specific failure mode — large buffers hiding latency instead of preventing loss — is what the term **bufferbloat** refers to, and it's the direct motivation for the modern AQM algorithms in §5.

---

## 3. RED — Random Early Detection

### 3.1 The core idea, in the original paper's own words

`[Common practice — academic]` Floyd & Jacobson, *"Random Early Detection Gateways for Congestion Avoidance,"* IEEE/ACM Transactions on Networking, August 1993. Their own framing, quoted directly: the gateway "detects incipient congestion by computing the **average** queue size... When the average queue size exceeds a preset threshold, the gateway drops or marks each arriving packet with a certain probability, where the exact probability is a function of the average queue size." The stated design goals: keep the average queue size **low** while still allowing **occasional bursts**, and — critically — during congestion, "the probability that the gateway notifies a particular connection to reduce its window is roughly proportional to that connection's share of the bandwidth through the gateway," which is the mechanism that avoids global synchronization: only a randomly-selected fraction of flows are signaled at any one time, roughly in proportion to how much of the link each one is using.

### 3.2 The algorithm — three regions

`[Common practice — academic, cross-verified]` RED tracks an **average** queue size, `avg` — an exponentially-weighted moving average of the *instantaneous* queue size, deliberately smoothed so that short bursts don't trigger drops. Two thresholds are configured: `minth` and `maxth`.

```
 avg < minth:              no drops at all — queue is healthy
 minth <= avg <= maxth:    drop/mark with probability p, rising linearly from 0 to max_p
 avg > maxth:              drop everything (behaves like tail drop above this point)
```

The linear probability function in this middle region (verified across multiple independent technical sources, consistent with the original paper's description):

```
 p = max_p * (avg - minth) / (maxth - minth)
```

### 3.3 Threshold guidance from the original paper

`[Common practice — academic]` Floyd & Jacobson's own guidance, as documented in later analyses of the paper: the optimal values of `minth` and `maxth` depend on the **desired average queue size**, and the optimal `maxth` depends partly on the **maximum average delay** the link can tolerate. They also state a specific rule of thumb: **`maxth` should be at least twice `minth`.**

### 3.4 Worked example

`minth` = 20 packets, `maxth` = 60 packets, `max_p` = 0.1 (i.e., 10% max drop probability in the linear region).

| avg (queue depth) | Region | Drop probability |
|--:|---|--:|
| 15 | Below minth | 0% |
| 30 | Linear region | 0.1 × (30−20)/(60−20) = 0.1 × 0.25 = **2.5%** |
| 50 | Linear region | 0.1 × (50−20)/(60−20) = 0.1 × 0.75 = **7.5%** |
| 65 | Above maxth | **100%** (tail-drop behaviour) |

This shows exactly why RED avoids the synchronization problem of §2.1: at avg=30, only about 1 in 40 packets is dropped — spread across whichever flows happen to be sending — rather than every flow losing a packet at once the moment the queue physically fills.

### 3.5 WRED — the multi-class extension (recap from note 6)

`[Guidance]` `6-QoS-Class-Design` §3.3 already established, directly from RFC 4594's own worked example, that a **single queue carrying multiple DSCPs** (as every AF-based service class does by design) needs **separate `minth`/`maxth` pairs per DSCP**, nested so higher drop-precedence values (AFx3) hit their `maxth` at a **lower** queue depth than lower drop-precedence values (AFx1). This per-DSCP variant of RED is what's commonly called **WRED** (Weighted RED) in the industry — the "weight" being exactly these per-DSCP threshold pairs. `7-QoS-Policy-Model` §7.2 already noted the structural reason this must be configured explicitly: an Algorithmic Dropper "operates indiscriminately" (RFC 3317) and cannot itself tell DSCPs apart — the differentiation comes entirely from pre-configured, separate threshold pairs.

---

## 4. RFC 2309 → RFC 7567 — The IETF Formally Changes Its Recommendation

### 4.1 What RFC 2309 said (1998)

`[Guidance]` RFC 2309 **introduced** the term **Active Queue Management (AQM)** to the IETF vocabulary — described, per RFC 7567's own retrospective, "in terms of the length of a queue" — and it **recommended RED specifically be the default AQM algorithm**, widely implemented and used by default in routers.

### 4.2 What RFC 7567 changed (2015) — quoted precisely

`[Standard-defined]` (Best Current Practice, BCP 197) RFC 7567, published 17 years later "based on 15 years of experience and new research," makes two changes worth stating exactly, both quoted directly from the RFC:

1. **Redefinition of AQM itself:** *"Whereas RFC 2309 described AQM in terms of the length of a queue, this memo uses AQM to refer to any method that allows network devices to control the queue length **and/or the mean time that a packet spends in a queue**."* This single sentence is the conceptual bridge to CoDel and PIE (§5), both of which manage **time in queue** rather than queue length directly.
2. **Explicit retraction of RED as the default:** *"This memo also explicitly obsoletes the recommendation that Random Early Detection (RED) be used as the default AQM mechanism for the Internet. This is replaced by a detailed set of recommendations for selecting an appropriate AQM algorithm."*

> ⚠️ **Gotcha:** RFC 7567 does **not** say RED is broken or should never be used — it says RED should **no longer be assumed as the automatic default**, and that algorithm selection should follow a more detailed set of criteria the BCP lays out. Many networks (and much still-current vendor documentation) continue to implement classic RED/WRED; that isn't non-compliant with RFC 7567, but citing RFC 2309 today as still recommending RED-by-default would be citing an obsoleted position.

### 4.3 The "lemmings vs. elephants" problem, named directly in RFC 7567

`[Standard-defined]` RFC 7567 names a specific, documented failure mode worth preserving in the vendor-neutral vocabulary of this series, quoted directly: *"lemmings' are flash crowds of 'mice' that the network inadvertently tries to signal to as if they were 'elephant' flows, resulting in head-of-line blocking in a data center deployment scenario."* In plain terms: an AQM (or any queue-depth-based mechanism) tuned to manage a few large, long-lived flows ("elephants") can misfire against a **sudden burst of many small, short-lived flows** ("mice") that happen to arrive together ("lemmings") — punishing traffic that was never actually the source of sustained congestion. This is one of the concrete "new research" findings RFC 7567 cites as justification for moving beyond simple queue-depth-based RED.

---

## 5. Modern, Delay-Based AQM: CoDel and PIE

### 5.1 The shared motivation — bufferbloat

`[Guidance]` Both algorithms below cite the same problem directly: RFC 8033 (PIE) states it plainly — TCP "continuously increases its sending rate and causes network buffers to fill up. TCP cuts its rate only when it receives a packet drop or mark... However, drops and marks usually occur when network buffers are full or almost full. As a result, excess buffers, initially designed to avoid packet drops, would lead to highly elevated queuing latency." Both RFCs cite Jim Gettys' 2011 IEEE Internet Computing article, *"Bufferbloat: Dark Buffers in the Internet,"* as the source that brought this problem to wide attention.

### 5.2 CoDel — Controlled Delay (RFC 8289)

`[Standard-defined]` (Experimental) Authored by Nichols & Jacobson (the same Jacobson as the 1993 RED paper) with McGregor and Iyengar, RFC 8289's core innovation, quoted directly: it uses **"packet sojourn time as the observed datum (rather than packets, bytes, or rates)"** — i.e., CoDel measures **how long each individual packet actually waited** in the queue, not how many packets or bytes are currently in it.

**The two parameters** (RFC 8289's own defaults):

| Parameter | Default | Meaning |
|---|--:|---|
| **TARGET** | **5 ms** | The acceptable sojourn time — chosen, per the RFC's design rationale, to be "close to zero (for better delay) but not so small that the queue would run empty" |
| **INTERVAL** | **100 ms** | The window over which CoDel tracks the **local minimum** sojourn time — chosen because 100ms approximates a typical Internet RTT |

**The mechanism**: CoDel tracks the **minimum** sojourn time observed over each INTERVAL window (using the minimum, rather than an average, specifically distinguishes brief, harmless bursts from sustained, genuine congestion). If that local minimum stays **above TARGET for longer than INTERVAL**, CoDel enters a "drop state" and begins dropping (or ECN-marking) packets — with the interval between successive drops **shrinking** the longer the bad-queue condition persists (RFC 8289 states this control-theoretic detail: the drop rate increases roughly with the square root of the count of drops since entering the bad-queue state, though the note's aim here is the concept, not the precise control law).

**Non-starvation safeguard**, quoted directly: *"To keep from making drops when it would starve the output link, CoDel makes another check before dropping to see if at least an MTU worth of bytes remains in the buffer."* If not, it **does not drop**, and exits the drop state — a deliberate protection against under-running a slow or variable-rate link.

**CoDel needs no manual tuning for the target rate/link speed** — RFC 8289 explicitly designed it to require "no configuration in normal Internet deployments," working "across a wide range of conditions, with varying links and the full range of terrestrial round-trip times," which is a deliberate design departure from RED's need for link-rate-dependent `minth`/`maxth` tuning (§3.3).

### 5.3 PIE — Proportional Integral controller Enhanced (RFC 8033)

`[Standard-defined]` (Experimental) PIE also targets **latency**, but via a different control-theory approach: quoted directly, it is "the classical Proportional Integral (PI) controller method, which is known for eliminating steady-state errors" — it periodically recalculates a drop probability based not just on the **current** measured latency, but on whether that latency is **trending up or down**.

**Key parameters (RFC 8033's own defaults, Appendix B):**

| Parameter | Default | Meaning |
|---|--:|---|
| **QDELAY_REF** (AQM Latency Target) | **15 ms** | The target queueing delay |
| **MAX_BURST** (Burst Allowance) | **150 ms** | A grace period allowing short bursts through without triggering drops |
| **T_UPDATE** | **15 ms** | How often the drop probability is recalculated |
| **alpha, beta** (PI controller weights) | 1/8, 1¼ | Control how strongly the *current* latency deviation (alpha) vs. its *trend* (beta) each influence the updated drop probability |

**Why PIE separates control-path and data-path work**: RFC 8034 (the DOCSIS-specific companion to PIE) frames this cleanly — a periodically-running **control path** calculates the drop probability from the latency trend, while a lightweight **data path** function, run per packet, simply applies that already-computed probability. This division keeps the expensive trend calculation off the per-packet critical path, an important property for high-speed implementation.

### 5.4 CoDel vs. PIE, compared

| | **CoDel** | **PIE** |
|---|---|---|
| Signal used | Sojourn time (time actually spent in queue), tracked as a **local minimum** | Queueing latency estimate, tracked with a **trend** (PI control) |
| Target default | 5 ms | 15 ms |
| Tuning philosophy | Designed to need **no** manual configuration | Configurable target latency and burst allowance |
| Theoretical basis | State-space controller, "network power" setpoint | Classical control theory (Proportional-Integral controller) |
| Standards status | RFC 8289, Experimental | RFC 8033, Experimental |
| Deployment note | RFC 8289 itself recommends the **FQ-CoDel** combination (§6) over plain CoDel for most real deployments | Adapted for DOCSIS cable-modem use in the companion RFC 8034 |

---

## 6. FQ-CoDel — Combining Fairness and Delay Control

### 6.1 Why combine at all

`[Standard-defined]` (RFC 8290, Experimental) A single shared queue, even with CoDel or PIE managing its overall delay, still lets one greedy, high-rate flow dominate the queue and inflict most of the induced delay onto every other flow sharing it. FQ-CoDel's own summary, quoted directly, describes it as *"a hybrid of DRR [Deficit Round Robin, `10-QoS-Queuing-Scheduling` §5.2] and CoDel, with an optimisation for sparse flows"* — and explicitly calls this **"flow queueing" rather than "fair queueing,"** because, as the RFC states, *"flows that build a queue are treated differently than flows that do not."*

### 6.2 The mechanism

```
 Packets ---> [ hash on 5-tuple ] ---> per-flow sub-queue 1 (CoDel manages its own delay)
                                  ---> per-flow sub-queue 2 (CoDel manages its own delay)
                                  ---> per-flow sub-queue N (CoDel manages its own delay)
                                            |
                              [ DRR scheduler chooses which sub-queue's packet leaves next ]
```

- **Hashing**: by default, on the same **5-tuple** (source/destination address, source/destination port, protocol) already familiar from `3-QoS-Classification-Trust`'s MF classification — RFC 8290 notes this can be customized, but warns that doing so risks losing the per-flow distinction that makes the scheme work.
- **DRR scheduling between sub-queues**: RFC 8290 states its DRR is **byte-based**, tracking "byte credits" per queue exactly as described in `10-QoS-Queuing-Scheduling` §5.2 — quoted directly: *"if one queue contains packets of, for instance, size quantum/3, and another contains quantum-sized packets, the first queue will dequeue three packets each time it gets a turn, whereas the second only dequeues one... the DRR scheme approximates a byte-based fairness queueing scheme."* This is the exact DRR fairness property already verified in note 10, now applied specifically to separate per-flow sub-queues rather than QoS classes.
- **CoDel runs independently inside each sub-queue**, managing that one flow's sojourn time exactly as described in §5.2.

### 6.3 The sparse-flow optimization — "new" vs. "old" queues

`[Standard-defined]` RFC 8290's specific departure from plain DRR, quoted directly: *"Unlike plain DRR, there are two sets of flows: a 'new' list for flows that have not built a queue recently and an 'old' list for queues that build a backlog."* When a previously-idle queue receives a packet, it joins the **new** list; a queue that keeps sending enough to still have data queued after its turn moves to the **old** list. The **new list is served preferentially** — this is precisely what "flow queueing... rather than fair queueing" means in practice: a flow that has just started sending (e.g., a single DNS lookup, or the start of a web page load) gets prompt service *because* it hasn't yet built up a backlog, without needing any explicit classification or marking to identify it as latency-sensitive.

**Anti-starvation rule for this scheme**, quoted directly from RFC 8290's security considerations: *"To prevent packets in the new queues from starving old queues, it is important that when a queue on the list of new queues empties, it is moved to the end of the list of old queues."* — a queue can't stay on the favored "new" list indefinitely just by continuing to trickle in packets.

### 6.4 What FQ-CoDel achieves that plain CoDel or plain DRR do not

Quoted directly from RFC 8290's own summary: it *"mixes packets from multiple flows and reduces the impact of head-of-line blocking from bursty traffic. It provides isolation for low-rate traffic such as DNS, web, and videoconferencing traffic. It improves utilisation across the networking fabric, especially for bidirectional traffic, by keeping queue lengths short."* Note the direct connection to `10-QoS-Queuing-Scheduling`: DRR alone gives byte-fair *scheduling* between flows, and CoDel alone gives delay control *within* a queue — FQ-CoDel's contribution is combining both, plus the new/old sparse-flow rule, so that fairness and low latency are achieved **together**, without needing the network operator to pre-classify which flows are "important."

> 📝 RFC 8289 itself recommends this combination directly: *"Implementers and users SHOULD use the fq_codel multiple-queue approach as it deals with many problems beyond the reach of an AQM on a single queue."* This is a deliberate, RFC-stated preference for FQ-CoDel over standalone CoDel wherever a multi-queue implementation is feasible.

---

## 7. Microbursts — Recap and AQM's Relevance

`[Common practice]`, extending `6-QoS-Fundamentals` §6's observation about utilisation averages: a monitoring tool reporting "30% average utilisation" over five minutes can hide millisecond-scale **microbursts** that briefly saturate a queue completely. All of the algorithms in this note interact with microbursts differently:

- **Tail drop** reacts only when the buffer physically fills — a microburst that's shorter than the buffer's drain time never triggers anything, which is fine, but a slightly larger burst causes an abrupt, synchronized loss event (§2.1).
- **RED's smoothing (`avg`, §3.2)** is deliberately designed to **absorb** brief bursts without reacting — the exponentially-weighted average doesn't spike the way instantaneous queue depth does.
- **CoDel's INTERVAL (100 ms) and minimum-tracking** serve exactly the same purpose from a different angle: a burst that clears within 100ms never persists long enough to drop CoDel's tracked local minimum above TARGET, so short bursts pass unaffected — this is the direct answer to why CoDel uses a *minimum* over a *window* rather than reacting to any single high-sojourn-time sample.

---

## 8. CCIE-Depth Topics

### 8.1 Why "average" queue size in RED and "minimum sojourn time" in CoDel serve the same design goal by different means

Both are answers to the identical underlying question: **how do we tell a genuine, sustained congestion episode apart from a harmless transient burst?** RED answers it by smoothing queue *depth* over time (an EWMA); CoDel answers it by tracking the *minimum* delay experienced over a sliding window, reasoning that if even the **best-case** packet in that window still waited too long, the queue truly isn't draining fast enough — a transient burst, by contrast, will still produce some packets with near-zero sojourn time as the queue briefly empties between bursts, keeping the tracked minimum low. Two different statistics, same underlying purpose: filter out noise, react only to persistent congestion.

### 8.2 Why RFC 7567's redefinition of AQM (queue length vs. time-in-queue) is not merely cosmetic

A queue's **length** (RFC 2309's framing) depends on the **link's own rate** — the same 100 packets represent a much shorter delay on a 10 Gbps interface than on a 1 Mbps one. A fixed `minth`/`maxth` pair configured in *packets* or *bytes* (classic RED) therefore needs re-tuning whenever the link rate changes. Measuring **time in queue** directly (CoDel, PIE) sidesteps this entirely — 5ms of sojourn time means the same thing regardless of link speed. This is the precise, mechanical reason RFC 8289 could claim CoDel needs "no configuration... across a wide range of conditions, with varying links" while classical RED explicitly could not make that claim (§3.3's link-rate-dependent tuning guidance).

### 8.3 The "lemmings" problem as a reason to prefer flow-aware AQM

RFC 7567's lemmings/elephants distinction (§4.3) is precisely the class of problem FQ-CoDel's **per-flow hashing** (§6.2) structurally avoids: because each flow gets its own sub-queue and its own CoDel instance, a sudden burst of many small, short-lived flows cannot be mistaken for one sustained large flow — each small flow's own sojourn-time history is tracked independently, and the "new list" preferential service (§6.3) specifically protects exactly this kind of traffic.

### 8.4 ECN as the alternative to dropping — a forward reference

Every algorithm in this note (RED, CoDel, PIE) is described as able to **either drop or mark** a packet as its congestion signal. Marking (setting the ECN CE codepoint instead of discarding the packet) lets a compliant sender receive the same "slow down" signal **without any packet loss at all** — full mechanism, wire format, and end-to-end negotiation covered in `12-QoS-ECN`.

---

## 9. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | Tail drop's failure mode is **global synchronization**, not simply "loss" | Many flows cut their windows at once, causing throughput to oscillate rather than stabilize |
| 2 | A **larger** buffer doesn't fix tail drop — it causes **bufferbloat** instead | Delay grows silently until the (now much later) drop finally occurs |
| 3 | RFC 7567 **retracted** RED as the recommended default AQM (2015) | Citing RFC 2309's RED-by-default recommendation today cites an obsoleted position |
| 4 | RED's thresholds (`minth`/`maxth`) are configured in queue **length**, so they need retuning per link rate | This is exactly the limitation CoDel/PIE's time-based approach was designed to avoid |
| 5 | WRED needs **separate threshold pairs per DSCP**, pre-configured — the dropper cannot classify on its own | Structural consequence of RFC 3317's "operates indiscriminately" rule (`7-QoS-Policy-Model`) |
| 6 | CoDel tracks a **minimum**, not an average, sojourn time | Deliberately distinguishes brief bursts (which still produce some near-zero samples) from sustained congestion |
| 7 | CoDel's non-starvation check can **override** a pending drop if less than one MTU remains buffered | Protects against under-running slow or variable-rate links |
| 8 | The "lemmings vs. elephants" problem is a **named, RFC-documented** failure mode | Not a hypothetical — RFC 7567 cites it explicitly as a reason to move past simple queue-depth AQM |
| 9 | FQ-CoDel's "new" queue list gets **preferential** service, not equal service | This is what lets sparse flows (DNS, web, video calls) get low latency without any explicit marking |

---

## 10. Quick Recap

| Concept | One-line answer |
|---|---|
| Tail drop's problem | Global synchronization — many TCP flows back off together |
| Bufferbloat | Oversized buffers hide latency instead of preventing loss |
| RED's mechanism | Random drop, probability rising with **average** queue depth, between minth and maxth |
| RFC 2309 → RFC 7567 | AQM redefined around **time in queue**; RED formally un-recommended as the default |
| "Lemmings vs. elephants" | RFC 7567's named failure mode: bursts of small flows mistaken for one sustained large flow |
| CoDel's signal | Packet **sojourn time**, tracked as a local minimum over a 100ms window; target 5ms |
| PIE's signal | Queueing latency **and its trend**, via a Proportional-Integral controller; target 15ms |
| FQ-CoDel | Per-flow (5-tuple) DRR scheduling + CoDel per sub-queue + new/old list sparse-flow preference |
| Common thread across all modern AQM | Manage **delay**, not just queue length, and avoid needing link-rate-specific tuning |

---

## References

**Best Current Practice / Standards Track (Experimental)**
- RFC 7567 — IETF Recommendations Regarding Active Queue Management (BCP 197; obsoletes RFC 2309)
- RFC 8289 — Controlled Delay Active Queue Management (CoDel)
- RFC 8033 — Proportional Integral Controller Enhanced (PIE)
- RFC 8034 — PIE-Based AQM for DOCSIS Cable Modems (control-path/data-path split)
- RFC 8290 — The Flow Queue CoDel Packet Scheduler and Active Queue Management Algorithm (FQ-CoDel)

**Obsoleted (cited for history only)**
- RFC 2309 — Recommendations on Queue Management and Congestion Avoidance in the Internet (obsoleted by RFC 7567)

**Academic literature (not RFCs — cited as such)**
- S. Floyd, V. Jacobson — "Random Early Detection Gateways for Congestion Avoidance," IEEE/ACM Transactions on Networking 1(4), August 1993
- J. Gettys — "Bufferbloat: Dark Buffers in the Internet," IEEE Internet Computing, April 2011

**Referenced (background and what feeds into this note)**
- `7-QoS-Policy-Model` (Algorithmic Dropper element; per-DSCP AQM's structural requirement)
- `6-QoS-Class-Design` (RFC 4594's per-class AQM assignment and the nested-RED-threshold worked example)
- `10-QoS-Queuing-Scheduling` (DRR mechanics, reused directly inside FQ-CoDel)
- `1-QoS-Fundamentals` (utilisation averages hiding microbursts)
- `12-QoS-ECN` (marking as the alternative to dropping, for every algorithm in this note)
