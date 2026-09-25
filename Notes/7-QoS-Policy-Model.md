## QoS Policy Model — Class, Match, Action, and Order of Operations

> 💡 **TL;DR:** Every vendor's QoS configuration language — Cisco's MQC (class-map/policy-map/service-policy), Juniper's firewall filters and forwarding classes, or anything else — is an implementation of the **same underlying model**: RFC 3290's Diffserv router datapath. That model wires together a small set of **functional elements** — **classifier → meter → action (mark/count) → algorithmic dropper/queue → scheduler** — into a directed graph. A **policy**, in vendor-neutral terms, is nothing more than: **(1) a set of classes** (defined by filters over a classification key, note 3), **(2) an action per class** (mark, police, queue, drop — notes 4, 5, 8, 9, 10), and **(3) a place and direction the policy is applied** (an interface, sub-interface, or logical construct, ingress or egress). RFC 3290 models **shaping and AQM as functions of queuing elements**, and **policing as either a Meter+AbsoluteDropper pair or an AlgorithmicDropper+Scheduler pair** — meaning the "queuing" and "policing" topics later in this series are really just different wirings of the same handful of building blocks.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 3290, RFC 3317 (its companion PIB / Policy Information Base) — Informational. RFC 2475 (traffic conditioning block) — Informational. RFC 2697, RFC 2698 (concrete meters referenced by the model) — Informational (historically labeled Informational despite defining widely-implemented mechanisms).

---

## 1. Why a Vendor-Neutral Policy Model Is Possible at All

Every QoS platform lets an operator say, in some syntax: *"traffic that looks like X gets treatment Y, applied here."* That sentence has exactly three parts, and RFC 3290 formalizes all three without reference to any vendor's command syntax:

```
  1. WHAT is it?        -->  Classifier   (note 3: BA vs MF, filters, precedence)
  2. HOW MUCH of it,     -->  Meter        (measures against a rate/burst profile)
     and is it within
     the agreed profile?
  3. WHAT do we DO       -->  Action(s)    (mark, count) then
     about it?                Queue / Algorithmic Dropper / Scheduler
                               (hold it, drop it early, or schedule its departure)
```

RFC 3290 §1 states its purpose directly: it defines **functional datapath elements** (classifiers, meters, actions — marking, absolute dropping, counting, multiplexing — algorithmic droppers, queues and schedulers), describes their configuration parameters, and describes **how they might be interconnected** to realize the traffic-conditioning and PHB behaviours of the DiffServ architecture (RFC 2475, covered in `2-QoS-Models`). This is precisely a **vendor-neutral policy language** — it just isn't usually taught as one, because most engineers meet it only through one vendor's syntax for it.

---

## 2. The Elements, One at a Time

### 2.1 Classifier (recap — full detail in note 3)

Fan-out element: one input, filters, several outputs. A **BA classifier** matches only the DSCP; an **MF classifier** matches any combination of fields. Nothing new here — this note picks up **after** classification has produced separate streams per class.

### 2.2 Meter

`[Guidance]` A meter measures a stream of packets against a **temporal profile** and produces a **conformance result** — not an action by itself, just a measurement. RFC 3290 §5 gives the simplest case, the **Simple Token Bucket (TB) meter**, parameterized by an **average rate** and a **burst size**, and defines two conformance tests:

- **Strict conformance:** a packet of length L conforms only if the bucket currently holds **≥ L** tokens — no borrowing from future allocations. RFC 3290 cites the srTCM/trTCM meters (RFC 2697/2698) as examples of this.
- **Loose conformance:** a packet conforms if the bucket holds **any** tokens at all; up to L bytes may be borrowed from future allocations.

**Multi-stage TB meters** extend this to **two burst sizes and three conformance levels** — conforming, partially-conforming, non-conforming — which RFC 3290 explicitly connects to the **"colors"** used in the industry: **green** = conforming, **yellow** = partially conforming, **red** = non-conforming. This is exactly the color-marking language used by srTCM (RFC 2697) and trTCM (RFC 2698), which are full mechanisms built on this same TB-meter concept and are covered in depth in `8-QoS-Policing`.

**A meter's output is not one stream — it's several**, one per conformance level, each wired to a different downstream element:

```
                         +-------+
              +--------->| Queue1 |   (conforming / "green")
              |          +-------+
 packets ---->| Meter  |
              |        |---------->| AbsoluteDropper1 |   (non-conforming / "red")
              +--------+
```

RFC 3290's own example (paraphrased into the vendor-neutral form used throughout this series): a **loose-conformance Simple Token Bucket meter**, average rate 200 kbps, burst size 100 kbytes, sends conforming traffic to a queue and non-conforming traffic to an absolute dropper.

### 2.3 Actions: Marker, Counter, Multiplexor

`[Guidance]` RFC 3290 §6 groups these as simple, largely stateless **action elements** that sit downstream of a classifier or meter result:

- **Marker:** rewrites a field — a **DSCP Marker** is parameterized by a single value, the 6-bit DSCP to write (RFC 3290's own example: `Mark: 010010`, i.e., DSCP 18 = AF21). RFC 3290 explicitly notes the model supports marking **based on a preceding classifier match**, and that the mark written here **determines the packet's treatment at downstream nodes**, and possibly at later stages within the same router. This is the formal basis for everything in `4-QoS-Marking-Headers`.
- **Absolute Dropper:** simply discards the packet — no parameters. RFC 3290 notes it is a **terminating point** of the datapath (no outputs), so it's often preceded by a **Counter** action for instrumentation before the packet is discarded.
- **Counter:** counts packets/bytes passing a point, for statistics — has no effect on the packet's treatment.
- **Multiplexor (M:1, fan-in):** the mirror image of a classifier — merges several streams into one, for feeding into a single downstream element (e.g., a queue or scheduler that accepts only one input).

### 2.4 Algorithmic Dropper — where AQM lives

`[Guidance]` RFC 3290 explicitly distinguishes the **Algorithmic Dropper** from the Absolute Dropper: it operates on the state of one or more **queues** (not on a classifier's match result), which is why RFC 3290 treats it as closely tied to queuing elements even though, mechanically, it too discards packets. RFC 3317 (its companion Policy Information Base document) states this precisely: *"An Algorithmic Dropper is assumed to operate indiscriminately on all packets that are presented at its input; all traffic separation should be done by classifiers and meters preceding it."*

RFC 3290's own worked example names the discipline explicitly:

```
 AlgorithmicDropper:
   Type: AlgorithmicDropper
   Discipline: RED
   Trigger: Internal
   Output: Queue
   MinThresh: Queue.Depth > 20 kbyte
```

This is the formal, vendor-neutral seat of **RED/WRED and other AQM mechanisms** — fully covered in `11-QoS-Congestion-Avoidance`. The key structural point for this note: **AQM is not a separate stage bolted onto a queue from outside — RFC 3290 models it as a first-class datapath element that watches a queue's depth and decides whether to drop (or, per RFC 3168/ECN, mark) packets before the queue physically fills.**

### 2.5 Queue and Scheduler

`[Guidance]` A **Queue** is a data structure holding packets awaiting transmission; a **Scheduler** is the element that selects the next packet to serve from among one or more queues. RFC 3290's minimal example: a `FIFO` queue type with a single parameter, its `Output` (which scheduler it feeds). This pairing — one or more Queues feeding one Scheduler — is the formal basis for everything in `10-QoS-Queuing-Scheduling` (priority queuing, WFQ/WRR, etc.).

RFC 3290 also defines **non-work-conserving** as a property of a scheduling algorithm: it services packets **no sooner than a scheduled departure time**, even leaving packets queued while the output is otherwise idle. This is the formal definition behind **shaping** (`9-QoS-Shaping`) — RFC 3290 explicitly states that *shaping, sometimes considered a traffic-conditioning action, is treated in this model as a function of queuing elements*, not as a separate action element.

### 2.6 Policing, formally

`[Guidance]` RFC 3290 gives **two equivalent ways to model policing**, both already built from the elements above:

1. **Meter → Absolute Dropper**: measure conformance, discard what doesn't conform. This is the simplest, "hard drop" policer.
2. **Algorithmic Dropper → Scheduler**: drop based on queue-depth state rather than a per-packet rate test.

RFC 3290's own definition of **Policing**: *"The process of comparing the arrival of data packets against a temporal profile and forwarding, delaying or dropping them so as to make the output stream conformant to the profile."* Note that this definition already covers **shaping** too (delaying, not just dropping) — the RFC's model draws the classifier/meter/action boundary in one place, and treats shaping and policing as differing only in **which downstream element** the meter's non-conforming output is wired to (a dropper for policing; a delaying queue for shaping). This single fact is the cleanest possible statement of the policing-vs-shaping distinction covered in depth in `9-QoS-Shaping`.

---

## 3. The Full Wiring Diagram

Putting every element from §2 together into one vendor-neutral picture — this is the general shape that **any** vendor's policy configuration is ultimately compiled into:

```
                                    +-------------------------------------------------+
                                    |                  ONE CLASS                        |
                                    |                                                   |
 packets --> [ Classifier ] --+--> | [ Meter ] --conform--> [ Marker/Counter ] --> [ Queue ] --+
             (note 3)         |    |    |                                                      |
                               |    |    +--exceed---------> [ Marker (re-mark) ] --> [ Queue ] |--> [ Scheduler ] --> out
                               |    |                                                            |    (note 10)
                               |    +----exceed(worse)-----> [ Counter ] --> [ AbsoluteDropper ] |
                               |                                                                 |
                               +--> ... (repeat this whole block per class, in parallel) ---------+
                                                                                          ^
                                                                    [ Algorithmic Dropper ] watches
                                                                    each Queue's depth and drops/ECN-
                                                                    marks BEFORE the queue fills (AQM)
```

Every box in this diagram maps directly onto a topic already covered or still to come in this series:

| Element in this diagram | Covered in |
|---|---|
| Classifier | `3-QoS-Classification-Trust` |
| Marker | `4-QoS-Marking-Headers`, `5-QoS-PHB-DSCP-Values` |
| Meter + conform/exceed outputs | `8-QoS-Policing` |
| Queue with a non-work-conserving output | `9-QoS-Shaping` |
| Queue(s) + Scheduler | `10-QoS-Queuing-Scheduling` |
| Algorithmic Dropper | `11-QoS-Congestion-Avoidance` |
| Marker writing ECN instead of dropping | `12-QoS-ECN` |

> 📝 **This is the single most useful fact in this note:** there is no independent "policing topic" or "queuing topic" at the level of the underlying model — there is **one datapath model**, and "policing," "shaping," "queuing" and "AQM" are just names for particular sub-graphs of it. Vendor documentation splits these into separate chapters because separate hardware blocks often implement them, but the RFC 3290 model shows they are all instances of the same small vocabulary: classify, meter, mark, queue, drop, schedule.

---

## 4. Order of Operations

`[Common practice]`, structured directly from the element wiring above, plus RFC 2475's traffic-conditioning-block ordering (`2-QoS-Models` §4.4):

```
 1. CLASSIFY      -- decide which class a packet belongs to (note 3)
 2. METER         -- (if this class is policed/shaped) measure against a profile
 3. MARK          -- write or rewrite DSCP / CoS / MPLS TC based on class and/or
                      meter result (notes 4, 5)
 4. QUEUE         -- place the packet into the class's queue
 5. (AQM)         -- an Algorithmic Dropper may drop/ECN-mark based on that
                      queue's depth, independently of step 2's per-packet metering
 6. SCHEDULE      -- the Scheduler selects which queue's packet leaves next
 7. (SHAPE)       -- if the queue/scheduler pairing is configured non-work-
                      conserving, departure may be delayed past its turn
```

Two ordering facts worth stating precisely, because they are common sources of confusion:

- **Metering (step 2) and AQM (step 5) are two independent mechanisms** that can both apply to the same class. A meter tests each packet **on arrival** against a contracted rate (policing); an algorithmic dropper watches a **queue's depth** and drops/marks **without reference to any per-flow contract**. A class can be both policed at ingress *and* subject to AQM at its egress queue — these are not the same control and are not mutually exclusive.
- **Marking (step 3) can happen more than once.** RFC 3290's own marker example shows a mark being applied **based on a preceding classifier match** — and a meter's conformance result can drive a **second** marking decision (e.g., re-mark to a lower AF drop precedence on "exceed" rather than dropping outright — the AF-class behaviour from `6-QoS-Class-Design` §3.3's AF41/42/43 remarking logic). "Classify then mark then meter then re-mark" is a legitimate, RFC-modeled sequence, not a vendor quirk.

### 4.1 Ingress vs egress — recap from note 3, restated for policy design

`[Guidance]` (RFC 3290 §3.2, already introduced in `3-QoS-Classification-Trust` §1): the same functional elements exist at both ingress and egress of an interface, but **queuing (and therefore scheduling and most AQM) is primarily an egress phenomenon** — an ingress only queues if it shapes. A policy design should therefore ask, for each element in §3's diagram, **which side of the interface it needs to sit on**:

| Element | Typical side | Why |
|---|---|---|
| Classifier | Ingress (to decide class as early as possible) | Cheapest to decide once |
| Meter (policing) | Ingress | Enforce the contract before wasting downstream resources on excess traffic |
| Marker | Ingress or egress | Wherever the class is first known; re-marking can happen anywhere trust changes (note 3 §6) |
| Queue + Scheduler | **Egress** | Congestion happens where departure rate < arrival rate (note 1 §1) |
| Algorithmic Dropper (AQM) | Egress (tied to the egress queue) | Same reason as queuing |
| Shaping (non-work-conserving queue) | Ingress (to a downstream bottleneck) or egress | Covered fully in `9-QoS-Shaping` |

---

## 5. The Vendor-Neutral Policy as Three Named Parts

Every vendor's syntax ultimately names the same three things RFC 3290's model requires; this series intentionally uses vendor-neutral names throughout, with a translation table so the mapping to what you may already know is explicit rather than hidden:

| Vendor-neutral term (used in this series) | What it defines | Cisco MQC equivalent (for translation only — not used in this series' notes) |
|---|---|---|
| **Class definition** | The classifier: filters over the classification key (note 3) | `class-map` |
| **Policy** | The set of (class, action) pairs — meter, mark, queue parameters, drop behaviour | `policy-map` |
| **Application** | Where and in which direction the policy is attached | `service-policy {input\|output}` under an interface |

> 📝 This table exists once, here, purely as an orientation aid for readers coming from a specific vendor background. The rest of this series — including this note — describes the **class definition / policy / application** structure directly, in the vendor-neutral terms of the left column, exactly as instructed for this series.

### 5.1 A complete vendor-neutral policy, worked end to end

Bringing together notes 3–7, here is a full worked policy in the class/action table format already used earlier in this series, extended to show every element from §2:

| Class definition | Meter (profile) | Conform action | Exceed action | Queue | Application |
|---|---|---|---|---|---|
| Match DSCP = EF | Single-rate TB, average 1 Mbps, burst 32 KB (loose) | Send to queue, no re-mark | Drop (Absolute Dropper) | Priority queue, no AQM | Egress, WAN interface |
| Match DSCP = AF41/42/43 | Two-rate TB (trTCM-style, RFC 2698, `8-QoS-Policing`), rates A < B | AF41 unchanged | Re-mark AF42→AF43 above rate B (not drop) | Rate queue, per-DSCP AQM thresholds (`6-QoS-Class-Design` §3.3) | Egress, WAN interface |
| Match interface = access-port AND untrusted source | MF classify + verify (note 3 §6.3) | Re-mark to CS0 if mark not permitted for this source type | — | Default queue | Ingress, access interface |
| No match (wildcard, note 3 §3) | — | — | — | Default queue, AQM enabled | Egress, WAN interface |

This table **is** a complete policy in the RFC 3290 sense — every cell corresponds to a named element (or absence of one) from §2, wired in the order given in §4.

---

## 6. Hierarchy — Policies Within Policies

`[Common practice]` A single flat policy cannot express two independent requirements at once: **(a)** which class a packet is in, and **(b)** what happens to the *aggregate* of several classes sharing a parent construct (e.g., "these three classes together get 40% of the link, and *within* that 40%, EF gets priority"). This is solved, across virtually every vendor's QoS implementation, by **nesting one policy inside another** — a **hierarchical (parent/child) policy**.

```
                    Parent (applied to the interface)
                    +---------------------------------------+
                    |  Overall shaper: 40 Mbps               |
                    |                                        |
                    |   Child policy (applied "inside" the   |
                    |   parent's 40 Mbps)                    |
                    |   +---------------------------------+  |
                    |   | Class EF     -> priority queue   |  |
                    |   | Class AF4x   -> rate queue, 30%  |  |
                    |   | Class default-> rate queue, rest |  |
                    |   +---------------------------------+  |
                    +---------------------------------------+
```

In RFC 3290's element terms, hierarchy is simply **queues (or a scheduler's output) feeding into another queue/scheduler stage**, rather than directly out the physical interface — a Scheduler's output can itself be the input to a shaping Queue, which is why RFC 3290 needed no special "hierarchy" primitive: it falls out naturally from allowing scheduler outputs to be wired to further queuing elements, the same as any other output.

`[Common practice]` Two situations that specifically require hierarchy, independent of vendor:
- **A shaped aggregate containing internally-prioritized traffic** — e.g., a sub-rate WAN circuit (shape to the contracted rate) that still needs EF to jump the queue *within* that shaped rate (worked further in `9-QoS-Shaping`).
- **Per-customer or per-tunnel shaping with a shared class structure** — many child policies (one per customer/tunnel), each shaped to its own contracted rate, all built from the same class definitions.

Full mechanics of nested scheduling (three-level hierarchies, bandwidth-remaining percentages, interactions between a parent shaper and a child priority queue) belong in `10-QoS-Queuing-Scheduling`; this note establishes only that **hierarchy is a wiring pattern of the same elements**, not a separate mechanism.

---

## 7. CCIE-Depth Topics

### 7.1 Why "policing = meter + dropper" also explains conditional marking

Because RFC 3290 models a meter's outputs as **multiple streams**, each wired independently, nothing in the model restricts the "non-conforming" output to an Absolute Dropper. Wiring it instead to a **Marker** (which then feeds a Queue) is exactly how "police and re-mark instead of drop" policies (used throughout `6-QoS-Class-Design`'s AF handling) are built — it's the identical policing structure with one output wire moved. This is why srTCM/trTCM (`8-QoS-Policing`) can offer a "mark-down" action as a first-class alternative to "drop," rather than needing a separate mechanism.

### 7.2 The Algorithmic Dropper's "operates indiscriminately" rule has a consequence

RFC 3317's statement that an Algorithmic Dropper "operates indiscriminately on all packets... all traffic separation should be done by classifiers and meters preceding it" means: **an AQM mechanism cannot itself tell classes apart.** If a queue carries a mix of DSCPs (as AF classes do, by design — `6-QoS-Class-Design` §3.3), differentiated AQM behaviour per DSCP (the nested-threshold RED pattern already verified in that note) must come from the Algorithmic Dropper being **parameterized per-DSCP** ahead of time (multiple threshold pairs, one per expected DSCP value arriving at that queue), not from the dropper doing its own classification at drop time. This is a structural reason, not an implementation quirk, for why "AQM per DSCP" configurations exist as their own concept in `11-QoS-Congestion-Avoidance`.

### 7.3 Cascaded meters

The historical draft version of this model (draft-ietf-diffserv-model, an earlier revision of what became RFC 3290) shows **cascaded TCBs** (Token bucket meters wired output-to-input in series) — e.g., `Meter10: Output A --> Queue A; Output B --> Dropper10`, then a second meter measuring only what the first meter already passed. This is the formal basis for **two-rate policers being expressible as two chained single-rate meters** if a platform lacks a native two-rate meter — again, no new element type, only a different wiring of the same Meter element repeated.

### 7.4 Multiplexing before a shared resource

The M:1 Multiplexor element exists specifically because most downstream elements (a Queue, an Algorithmic Dropper) are defined with **one input**. Any point where several classes must share one physical resource — e.g., several MF-classified sub-streams that all need to land in the same egress queue — is modeled as a Multiplexor feeding that shared element. This is the formal reason "many classes, one queue" (as required for the entire AF PHB group by RFC 2597, `2-QoS-Models` §4.2) is representable at all within a model built from single-output classifiers and single-input queues.

---

## 8. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | Policing and shaping differ only in **which output a meter's "exceed" result is wired to** (a dropper vs. a delaying queue) | They are not fundamentally different mechanisms — RFC 3290 defines both from the same Meter element |
| 2 | AQM (Algorithmic Dropper) and metering (policing) are **independent controls** that can both apply to the same class | Configuring one does not substitute for the other |
| 3 | An Algorithmic Dropper **cannot classify** — it needs per-DSCP parameters set in advance if it must treat DSCPs differently in one queue | "AQM isn't working per-class" is usually a missing per-DSCP threshold configuration, not a broken dropper |
| 4 | Marking can legitimately happen **more than once** in one packet's path through a single device (classify-based mark, then meter-result-based re-mark) | This is modeled behaviour, not a misconfiguration |
| 5 | Hierarchy is not a separate primitive — it's a Scheduler's output feeding another Queue/Scheduler stage | Nested policies are just deeper wiring, not a fundamentally different construct |
| 6 | "Strict" vs "loose" token-bucket conformance changes whether a packet can **borrow from future tokens** | Affects burst behaviour even with identical rate/burst-size numbers — detailed in `8-QoS-Policing` |
| 7 | Shaping is defined as a property of a **queuing element** (non-work-conserving scheduling), not a standalone action | Explains why shapers always imply a buffer/delay, while a pure Absolute-Dropper policer never buffers |

---

## 9. Quick Recap

| Concept | One-line answer |
|---|---|
| The model behind every vendor's QoS syntax | RFC 3290's Diffserv router datapath elements |
| Core elements | Classifier, Meter, Action (Marker/Counter/Multiplexor), Algorithmic Dropper, Queue, Scheduler |
| A "policy" in vendor-neutral terms | Class definitions + per-class actions + where/direction it's applied |
| Policing, formally | Meter → Absolute Dropper (or Algorithmic Dropper → Scheduler) |
| Shaping, formally | A property of a queuing element: non-work-conserving scheduling |
| AQM, formally | An Algorithmic Dropper watching queue depth, independent of per-packet metering |
| Order of operations | Classify → meter → mark → queue → (AQM) → schedule → (shape) |
| Where queuing/AQM mostly lives | Egress (arrival rate can exceed departure rate there) |
| Hierarchy | A scheduler's output feeding a further queue/scheduler stage — not a new element type |
| "Green/yellow/red" | RFC 3290's own terms for conforming / partially-conforming / non-conforming meter output |

---

## References

**Informational (the policy model itself and its supporting documents)**
- RFC 3290 — An Informal Management Model for Diffserv Routers (classifiers, meters, actions, algorithmic droppers, queues, schedulers; policing and shaping definitions)
- RFC 3317 — Differentiated Services Quality of Service Policy Information Base (companion PIB; Algorithmic Dropper behaviour)
- RFC 2475 — An Architecture for Differentiated Services (traffic-conditioning block: classifier → meter → marker → shaper/dropper)
- RFC 2697 — A Single Rate Three Color Marker (srTCM; referenced by RFC 3290 as an example strict-conformance meter)
- RFC 2698 — A Two Rate Three Color Marker (trTCM; referenced by RFC 3290 as an example strict-conformance meter)

**Referenced (full mechanism detail in later notes)**
- `8-QoS-Policing`, `9-QoS-Shaping`, `10-QoS-Queuing-Scheduling`, `11-QoS-Congestion-Avoidance`, `12-QoS-ECN`
