## QoS Queuing and Scheduling

> 💡 **TL;DR:** A **queue** holds packets; a **scheduler** decides which queue's packet leaves next (`7-QoS-Policy-Model` §2.5). Everything in this note is a different answer to that one scheduling question. **FIFO** answers "whoever arrived first" — no differentiation at all. **Priority Queuing (PQ)** answers "always the highest non-empty queue" — RFC 4594's own definition, with a real starvation risk. **Round-robin family** (WRR, DRR) answers "take turns, weighted by importance" — DRR (Shreedhar & Varghese, 1995) is the practical, O(1)-complexity way to do this fairly even with variable-size packets. **WFQ** (Demers, Keshav & Shenker, 1989, analyzed by Parekh & Gallager) answers "approximate an ideal, bit-by-bit fair share as closely as packet-based scheduling allows." All of these trace back to **RFC 970** (Nagle, 1985), which first proposed per-flow queues serviced round-robin specifically to stop one bad actor from starving everyone else. **Hierarchical scheduling** (`7-QoS-Policy-Model` §6) simply nests these same algorithms — a parent scheduler's output feeding a child scheduler's input.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance / academic literature · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 970 — Informational (historic). RFC 4594's queuing definitions — Informational. WFQ (Demers/Keshav/Shenker 1989), its GPS analysis (Parekh/Gallager 1993–94), and DRR (Shreedhar/Varghese 1995) are **academic papers**, not RFCs — cited as such throughout.

---

## 1. Why Queuing and Scheduling Matter — Recap

From `1-QoS-Fundamentals` §1: queuing only matters when arrival rate can exceed departure rate, which happens almost exclusively at **egress**. From `7-QoS-Policy-Model` §2.5: a Queue holds packets; a Scheduler selects among one or more queues' heads. This note is entirely about **which scheduling algorithm** to use, and what each one actually guarantees.

```
  Queue A (voice)  --\
  Queue B (video)  ----> [ SCHEDULER ] ---> one packet at a time, out the interface
  Queue C (default) --/
```

---

## 2. FIFO — The Baseline

`[Guidance]` A single queue, one scheduler rule: **first in, first out**. RFC 3290 (`7-QoS-Policy-Model` §2.5) gives FIFO as its minimal queue example, parameterized by nothing but an output. No differentiation exists — every packet, regardless of class, waits behind everything that arrived before it.

**The problem FIFO creates**, stated by Nagle in RFC 970 (1985) in almost these exact words: *"If the packet switches queue on a strictly first in, first out basis, the badly behaved host will interfere with the transmission of data by other, better-behaved hosts."* This single sentence is the historical origin of essentially every queuing mechanism covered in the rest of this note — they all exist to prevent that one outcome.

---

## 3. RFC 970 — Where Per-Flow Fair Queuing Began

`[Guidance]` RFC 970 (Nagle, December 1985) is the foundational document. Its proposal, quoted directly: *"We can do this by replacing the single first in, first out queue associated with each outgoing link with multiple queues, one for each source host in the entire network. We service these queues in a round-robin fashion, taking one packet from each non-empty queue in turn."*

Nagle's own stated goal was **fairness**: *"each source host should be able to obtain an equal fraction of the resources of each packet switch."* Two mechanical details worth preserving exactly:

- **Empty queues are skipped** and lose their turn — a queue with nothing to send doesn't get to "save up" a turn for later.
- Under buffer exhaustion, RFC 970 recommends dropping **from the end of the longest queue**, since that packet would be transmitted last anyway — explicitly framed as intentionally unfavorable to whichever flow is consuming the most buffer space, "in keeping with our goal of fairness."

**The limitation RFC 970's simple round-robin has**, which the rest of this note exists to fix: it is fair only when **every packet is the same size**. A flow sending 1500-byte packets and a flow sending 64-byte packets get the same *number* of turns under plain round-robin, but the first flow gets roughly 23× the *bandwidth* — turns are equal, bytes are not.

---

## 4. Priority Queuing (PQ)

### 4.1 Definition

`[Guidance]` RFC 4594 §1.4.1.1 (already introduced in `6-QoS-Class-Design` §2.2), quoted precisely: *"A priority queuing system is a combination of a set of queues and a scheduler that empties them in priority sequence. When asked for a packet, the scheduler inspects the highest priority queue and, if there is data present, returns a packet from that queue. Failing that, it inspects the next highest priority queue, and so on."*

```
 Scheduler asks: "anything in Queue-Highest?"  --yes--> send it, ask again
                        |no
                  "anything in Queue-Next?"     --yes--> send it, ask again
                        |no
                  ... continue down the priority order ...
```

### 4.2 The delay guarantee — and its exact bound

RFC 4594 states the technical reason for using PQ directly: in a priority queuing system, a packet in the **highest**-priority queue experiences a delay that is *"proportional to the amount of data remaining to be serialized when the packet arrived plus the volume of data already queued ahead of it in the same queue."* This is a **calculable, bounded** delay — the reason `6-QoS-Class-Design` §2.2 confirmed Telephony/EF is the **only** RFC 4594 class assigned Priority Queuing: it's the one class whose delay bound this formula actually needs to be small.

### 4.3 Starvation — the cost of strict priority

RFC 4594 states plainly: *"A priority queue or queuing system needs to avoid starvation of lower-priority queues. This may be achieved through a variety of means, such as admission control, rate control, or network engineering."* **Strict** priority queuing has no built-in fairness mechanism at all — if the highest-priority queue never empties, lower-priority queues never get served, full stop. This is why PQ is never used alone as a network's entire scheduling strategy; it's always paired with something that limits how much traffic can enter the priority queue (policing, admission control — `8-QoS-Policing`, `6-QoS-Class-Design` §3.1's Telephony edge conditioning).

> ⚠️ **Gotcha:** Priority queuing solves delay for the top queue by definition — it says nothing about fairness or starvation for anything below it. The two problems (bounding delay for one class; sharing remaining bandwidth fairly among the rest) need two different mechanisms, which is exactly why real deployments combine PQ (for one or two classes) with a round-robin-family scheduler (for everything else) — covered in §7's hierarchical model.

---

## 5. Round-Robin Family: Fixing Nagle's Equal-Size Assumption

### 5.1 Weighted Round Robin (WRR)

`[Common practice]` The simplest fix for giving different classes different *shares* rather than strictly equal turns: assign each queue a **weight**, and serve it that many turns (or that much data) per cycle, rather than exactly one packet per cycle as in RFC 970's original scheme.

```
 Weights:  Voice=1, Video=3, Data=1   (relative shares, not priority order)
 One round: [Voice] [Video][Video][Video] [Data]   -- repeat
```

**The problem WRR alone doesn't solve:** if "one turn" still means "one packet" and packets are variable-sized, the same unfairness Nagle's round-robin had (§3) reappears — a queue whose packets happen to be large gets more bytes per turn than a queue whose packets are small, even with equal weights. WRR is normally implemented either on **byte counts per turn** (closer to fair) or with an explicit correction — which is exactly what DRR formalizes.

### 5.2 Deficit Round Robin (DRR / DWRR)

`[Common practice — academic]` Shreedhar and Varghese, *"Efficient Fair Queueing Using Deficit Round Robin,"* ACM SIGCOMM 1995. Their own framing of the problem they solved: earlier "nearly perfect fairness" schemes needed **O(log n)** work per packet (n = number of active flows) — too expensive at high speed — while cheaper round-robin approximations were provably unfair with variable packet sizes. DRR achieves **near-perfect fairness at O(1) work per packet**, and is "simple enough to implement in hardware."

**The mechanism**, verified against the algorithm's own description:

| Concept | Meaning |
|---|---|
| **Quantum (Qᵢ)** | Bytes of service flow *i* is entitled to per round — this is DRR's version of a "weight," expressed directly in bytes |
| **Deficit Counter (DCᵢ)** | Unused quantum carried forward from a previous round, when the head-of-queue packet was too big to send with what remained |
| **Active list** | A linked list of queues that currently have data, so **idle queues are skipped entirely** — no wasted cycles checking empty queues (the same principle RFC 970 already established) |

**The algorithm, per round, for queue i at the head of the active list:**

```
 DCi = DCi + Qi                       -- add this round's quantum to any leftover deficit
 while queue i is non-empty AND packet-at-head.size <= DCi:
     send the packet
     DCi = DCi - packet.size
 if queue i is now empty:
     DCi = 0                          -- reset; no "banking" credit while idle
 else:
     move queue i to the back of the active list  -- leftover DCi carries to next round
```

### 5.3 Worked example

Two flows sharing a link, both with Quantum = 1000 bytes.

| Round | Flow A packet size | Flow A: DC before | Flow A: sent? | Flow A: DC after | Flow B packet size | Flow B: DC before | Flow B: sent? | Flow B: DC after |
|--:|--:|--:|---|--:|--:|--:|---|--:|
| 1 | 1500 | 0+1000=1000 | No (1500>1000) | 1000 (carried) | 200 | 0+1000=1000 | Yes | 800 → (queue empty) → 0 |
| 2 | 1500 | 1000+1000=2000 | Yes (1500≤2000) | 500 | 200,200,200,200 | 0+1000=1000 | Yes×4=800 sent, 5th needs 200 more but queue may be empty | depends on arrivals |

**What this demonstrates:** Flow A's large 1500-byte packet couldn't be sent in round 1 (only 1000 bytes of quantum available), so its full 1000-byte quantum simply **carries forward** as deficit rather than being wasted — by round 2, it has 2000 bytes of credit, enough to send the packet, with 500 left over for whatever comes next. No flow is ever penalized for having temporarily too little credit for its packet size; the credit **accumulates** until it's enough. This is precisely the fairness property Shreedhar & Varghese designed the deficit counter to provide, and it is the direct structural fix for the "large packets get more bytes per turn" problem that plain WRR (§5.1) and Nagle's original round-robin (§3) both have.

### 5.4 DRR's known trade-off

`[Common practice — academic]` DRR provides fairness measured over the **long term** (across many rounds); it does not offer the same tight per-packet delay bound that a scheduler using per-packet timestamps (like WFQ, §6) does. Its delay bound is proportional to the **number of active sessions/queues** — the more queues sharing the scheduler, the longer a given queue may have to wait for its turn to come around, even if it's not misbehaving. This is a documented, structural property of round-robin-family scheduling, not a bug: DRR trades a small amount of per-packet timing precision for O(1) computational simplicity.

---

## 6. Weighted Fair Queueing (WFQ)

### 6.1 The ideal it approximates: Generalized Processor Sharing (GPS)

`[Common practice — academic]` The theoretical ideal behind WFQ is **Generalized Processor Sharing**: imagine a scheduler that could serve **all** non-empty queues *simultaneously*, each at a rate proportional to its weight, as if the link's capacity were a fluid being divided continuously rather than packets being sent one at a time. GPS is not implementable on a real link (you can only send one packet at a time), but it is the fairness **target** every packet-based scheduler in this section is trying to approximate.

### 6.2 Origin — Demers, Keshav & Shenker (1989)

`[Common practice — academic]` *"Analysis and Simulation of a Fair Queueing Algorithm,"* ACM SIGCOMM 1989, is the paper that introduced **Weighted Fair Queueing** and its packetized (i.e., real, one-packet-at-a-time) implementation — extending Nagle's 1985 per-flow round-robin idea (§3) specifically to handle **variable packet sizes fairly**, which is the exact gap identified in §3's closing paragraph.

**The mechanism, conceptually:** WFQ computes, for each arriving packet, a **virtual finish time** — the time that packet would have finished being transmitted if the link were serving all backlogged flows according to the ideal GPS fluid model. The scheduler then always transmits the packet across all queues with the **smallest virtual finish time** next. This is fundamentally different from DRR's approach (rounds and quantums): WFQ sorts by a computed timestamp rather than cycling through queues in physical order.

### 6.3 How good an approximation is it? — Parekh & Gallager

`[Common practice — academic]` Parekh and Gallager (1993, 1994) formally analyzed WFQ's packetized implementation and proved it to be a good, bounded approximation of ideal GPS — giving WFQ a rigorous mathematical foundation, not just an intuitive design. This analysis is why WFQ (and its many variants — Worst-case Fair WFQ, Self-Clocked Fair Queueing, Virtual Clock, and others in the same academic lineage) is treated in the networking literature as the reference standard for "how fair can a real, packet-based scheduler actually be."

### 6.4 WFQ vs. DRR — the real trade-off

| | **WFQ** | **DRR** |
|---|---|---|
| Basis | Computed virtual finish time (timestamp) per packet | Quantum + deficit counter, round-robin order |
| Per-packet complexity | O(log n) to find the minimum timestamp among n active flows (a sorted structure) | **O(1)** |
| Fairness precision | Very tight, closely bounded relative to ideal GPS | Good long-term fairness; looser short-term/per-packet bound, proportional to number of active queues |
| Practical use | Where accuracy matters more than raw scheduling speed | High-speed links, hardware implementations, where O(1) is a hard requirement |

> 📝 This is the exact trade-off referenced (without naming these specific algorithms) throughout the networking literature comparing "hardware-implementable" schedulers vs. "theoretically ideal" ones: **DRR trades some fairness precision for guaranteed constant-time work per packet; WFQ trades scheduling speed for a mathematically tighter fairness bound.** Which one an actual platform implements, and under what internal name, is `[Implementation-dependent]` — this note deliberately describes the two algorithms by their standards/academic names rather than any vendor's branding for them.

---

## 7. Hierarchical Scheduling — Applying Note 7's Model

`7-QoS-Policy-Model` §6 already established that hierarchy is nothing more than a scheduler's output feeding a further queue/scheduler stage. This note now supplies the concrete algorithms that get nested:

```
                     Parent scheduler (e.g., a shaper limiting to 40 Mbps, note 9)
                     +--------------------------------------------------+
                     |                                                  |
                     |  Child scheduler (e.g., Priority + DRR/WFQ mix)  |
                     |   +--------------------------------------+       |
                     |   | Queue: Voice   -> served by PQ FIRST  |       |
                     |   | Queue: Video   -> DRR/WFQ, weight 3   |       |
                     |   | Queue: Data    -> DRR/WFQ, weight 1   |       |
                     |   +--------------------------------------+       |
                     +--------------------------------------------------+
```

`[Common practice]` The most common real-world scheduling design combines **exactly these two ideas from §4 and §5/§6**: one (or occasionally two) queues get **strict priority** (§4) — bounded by policing so they can't starve everything else (§4.3) — and every remaining queue shares the leftover bandwidth through a **round-robin-family algorithm** (DRR) or a **timestamp-based algorithm** (WFQ). This combination is sometimes given a vendor-specific name, but structurally it is simply: *PQ for the top tier, DRR/WFQ for everything below it, wired hierarchically as note 7 describes.*

---

## 8. CCIE-Depth Topics

### 8.1 Why DRR's quantum should be at least one MTU

Directly from the algorithm in §5.2: if Quantum is smaller than the largest packet a queue can present, that queue's deficit counter may need **several rounds** to accumulate enough credit to send even one packet — needlessly delaying it and effectively giving it a smaller *effective* rate than its quantum implies, especially at the start of a busy period when its deficit counter is at zero. Sizing quantum to at least the maximum expected packet size for that queue avoids this multi-round stall entirely — this is a structural property of the algorithm in §5.2, not a platform-specific tuning tip.

### 8.2 Why "the deficit resets to zero when idle" matters

Also from §5.2: when a queue empties, its deficit counter is **reset to zero**, not preserved. This prevents a queue that goes idle for a long time from "banking" a huge credit and then bursting disproportionately when traffic resumes — a queue returning from idle starts exactly like a queue that's never sent anything, receiving one quantum's worth of credit per round like everyone else. This is a deliberate anti-burst design choice in the original algorithm, not an oversight.

### 8.3 PQ's delay formula, applied

Revisiting RFC 4594's formula from §4.2 with numbers: on a 10 Mbps link (1.25 MB/s), if a 1500-byte packet has just begun serializing when a voice packet in the priority queue arrives, and there are 3 other voice packets (200 bytes each) already queued ahead of it in the same priority queue, the bound is:

```
 delay ≈ (1500 + 3×200) / 1,250,000 bytes/sec = 2100 / 1,250,000 ≈ 1.68 ms
```

This is the exact calculation method RFC 4594's own definition implies — "remaining serialization of what's on the wire, plus what's already queued ahead in the same queue" — and it's why keeping the priority queue's own depth small (via admission control and policing, §4.3) is just as important as having a priority queue at all.

### 8.4 GPS as a limit, not a target you configure

It's worth being precise that GPS (§6.1) is a **theoretical construct** used to *analyze* schedulers, not something any real device implements or that appears as a configuration option. When vendor or academic material says a scheduler "approximates GPS," it means Parekh & Gallager-style formal analysis has bounded how far that scheduler's actual packet-by-packet behavior can deviate from the fluid ideal — a statement about provable worst-case fairness, not a literal operating mode.

---

## 9. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | Plain round-robin (RFC 970) is fair in **turns**, not in **bytes**, with variable packet sizes | A flow with larger packets gets disproportionately more bandwidth under naive round-robin |
| 2 | Strict Priority Queuing has **no built-in anti-starvation mechanism** | Must always be paired with policing/admission control on the priority queue |
| 3 | DRR's fairness is a **long-term** guarantee; its delay bound grows with the number of active queues | Not the same guarantee as a per-packet timestamp scheduler like WFQ |
| 4 | DRR's deficit counter resets to **zero** on going idle | A queue can't bank credit while idle and then burst disproportionately |
| 5 | DRR's quantum should be ≥ the queue's largest expected packet | Otherwise multi-round stalls occur before the first packet can even be sent |
| 6 | GPS is a theoretical analysis tool, not a real, implementable scheduler | "Approximates GPS" is a fairness-bound claim, not a literal feature |
| 7 | WFQ's O(log n) sorting cost vs. DRR's O(1) is a real, documented trade-off | Explains why very high-speed hardware schedulers often favor DRR-family algorithms |

---

## 10. Quick Recap

| Concept | One-line answer |
|---|---|
| FIFO | No differentiation — first in, first out |
| RFC 970's fix | Per-flow queues, serviced round-robin, empty queues skipped |
| RFC 970's limitation | Fair in turns, not bytes, when packet sizes vary |
| Priority Queuing (RFC 4594) | Always serve the highest non-empty queue; bounded, calculable delay for the top queue |
| PQ's cost | No anti-starvation for lower queues without external policing/admission control |
| WRR | Weighted turns; still imperfect with variable packet sizes unless byte-corrected |
| DRR (Shreedhar & Varghese, 1995) | Quantum + deficit counter; O(1) per packet; near-perfect long-term fairness |
| WFQ (Demers/Keshav/Shenker, 1989) | Virtual-finish-time scheduling approximating ideal GPS |
| GPS | The theoretical fluid-sharing ideal used to analyze scheduler fairness, not a real algorithm |
| Common real-world design | PQ for one top class + DRR/WFQ for the rest, nested hierarchically (note 7) |

---

## References

**Informational (RFC)**
- RFC 970 — On Packet Switches With Infinite Storage (Nagle, 1985 — foundational fairness/round-robin proposal)
- RFC 4594 — Configuration Guidelines for DiffServ Service Classes (Priority Queuing and Rate Queuing definitions, §1.4.1)
- RFC 3290 — An Informal Management Model for Diffserv Routers (Queue/Scheduler element definitions — background from `7-QoS-Policy-Model`)

**Academic literature (not RFCs — cited as such)**
- A. Demers, S. Keshav, S. Shenker — "Analysis and Simulation of a Fair Queueing Algorithm," ACM SIGCOMM Computer Communication Review 19(4), 1989 (origin of Weighted Fair Queueing)
- A. Parekh, R. Gallager — analysis of WFQ as an approximation of Generalized Processor Sharing, IEEE/ACM Transactions on Networking, 1993–1994
- M. Shreedhar, G. Varghese — "Efficient Fair Queueing Using Deficit Round Robin," ACM SIGCOMM 1995 (and the extended version, IEEE/ACM Transactions on Networking)

**Referenced (background and what feeds into this note)**
- `1-QoS-Fundamentals` (why queuing delay is the one controllable delay component)
- `7-QoS-Policy-Model` (Queue/Scheduler as RFC 3290 elements; hierarchy as nested scheduling)
- `6-QoS-Class-Design` (RFC 4594's assignment of Priority Queuing to Telephony only)
- `8-QoS-Policing`, `9-QoS-Shaping` (the policing/shaping that must accompany a priority queue)
