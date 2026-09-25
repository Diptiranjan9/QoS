## QoS Troubleshooting — A Vendor-Neutral Method

> 💡 **TL;DR:** QoS troubleshooting is fundamentally the process of walking the **RFC 3290 datapath model** from `7-QoS-Policy-Model` §3 — classifier → meter → marker → queue → algorithmic dropper → scheduler — and checking, **at each stage**, whether the packet was treated the way the design intended. Almost every real QoS fault reduces to one of a small number of root causes: a **classifier that didn't match what was expected** (note 3), a **marking that didn't survive** to where it was needed (notes 4, 12, 14), a **policer or shaper enforcing the wrong contract** (notes 8–9), a **scheduler or AQM configuration that doesn't match the traffic actually present** (notes 10–11), or a **measurement methodology that doesn't match what's actually being asked** (this note, closing the loop with note 1's parameters). This note gives the systematic walk-through, the standards-defined measurement tools (RFC 2544, RFC 7679/7680, RFC 9341), and a symptom-to-likely-cause table built entirely from facts already verified earlier in this series.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / observed deployment status · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 2544 — Informational (device benchmarking). RFC 7679/7680 — Standards Track (IPPM one-way delay/loss). RFC 9341 — Standards Track (obsoletes the Experimental RFC 8321, December 2022). RFC 6192 — Informational (control-plane protection, `16-QoS-Control-Plane`).

---

## 1. The Systematic Method — Walk the Datapath

Every QoS fault report ("voice is choppy," "the backup job is starving everything," "our marked traffic isn't getting priority") can be diagnosed by asking, in order, exactly the questions this series already answered stage by stage:

```
 Packet arrives
      |
      v
 [1] CLASSIFICATION  --  Did the packet match the class it was supposed to?      (note 3)
      |
      v
 [2] MARKING         --  Is the DSCP/CoS/MPLS-TC what's expected, HERE,          (notes 4, 5)
      |                  at this specific point in the path?
      v
 [3] TRUST/BOUNDARY  --  Was this marking trusted, re-marked, or bleached        (note 3 §6-7,
      |                  between where it was set and where it's being checked?  note 14)
      v
 [4] POLICING/SHAPING -- Is a meter dropping, delaying, or re-marking            (notes 7 §2.6,
      |                  more than the contract should allow?                    8, 9)
      v
 [5] QUEUING/         -- Is the packet in the queue you expect, and is           (notes 7 §2.5,
     SCHEDULING           the scheduler actually favoring it?                    10)
      |
      v
 [6] AQM/ECN          -- Is congestion-driven drop/marking happening             (notes 11, 12)
      |                  at the depth/threshold you configured?
      v
 [7] MEASUREMENT      -- Are you actually measuring the parameter               (note 1, this
                          the complaint is about, the right way?                 note §3-4)
```

**Why this order matters**: a fault at an earlier stage **masks** everything downstream. If classification (stage 1) puts a packet in the wrong class, checking the scheduler's configuration (stage 5) — even if it's configured perfectly — will never explain the symptom, because the packet was never in the queue you're looking at. `7-QoS-Policy-Model` §4's order of operations (classify → meter → mark → queue → AQM → schedule) is not just a design template; it is also the **correct diagnostic order** to check, because each stage's correctness is a precondition for the next stage's output being meaningful.

---

## 2. Stage-by-Stage Verification Checklist

### 2.1 Classification (note 3)

- **Completeness**: does every packet match *some* filter, including the wildcard/default (`3-QoS-Classification-Trust` §3)? An incomplete classifier produces undefined behaviour for whatever falls through the gap.
- **Precedence**: if two filters could both match the same packet, which one actually wins, and is that the intended one (`3-QoS-Classification-Trust` §3)?
- **Field visibility**: is the classifier trying to match a field that isn't actually present or visible at this point — a port number hidden by fragmentation or encryption, a DSCP hidden inside a tunnel's inner header the device never inspects (`3-QoS-Classification-Trust` §5, `14-QoS-Tunnels-Overlays` §1)?
- **Direction**: classification is per-direction (`3-QoS-Classification-Trust` §5.6) — has the *return* path been checked separately, not assumed identical to the forward path?

### 2.2 Marking (notes 4–5)

- **Whole-DSCP match, not whole-byte match**: is anything in the path matching the full 8-bit ToS/Traffic-Class byte instead of the 6-bit DSCP? This silently misclassifies any packet carrying a non-zero ECN codepoint (`4-QoS-Marking-Headers` §2.3, `12-QoS-ECN` §2.2's gotcha).
- **Layer-boundary truncation**: if the path crosses an L2 segment or an MPLS domain, has the DSCP survived the 6-bit→3-bit truncation intact, or has it collapsed into the wrong Class-Selector family (`4-QoS-Marking-Headers` §4.2, §6.3, §8.2)?
- **Binary/decimal sanity check**: does the observed DSCP decimal value actually correspond to the PHB believed to be in use? (`5-QoS-PHB-DSCP-Values` §1's conversion method is the fastest way to catch a mis-typed or mis-remembered codepoint.)

### 2.3 Trust and boundary behaviour (notes 3, 14)

- **Where is the trust boundary, actually** — not where the design document says it is, but where the device configuration actually enforces it (`3-QoS-Classification-Trust` §6.6)?
- **Has the marking been bleached or re-marked** crossing an administrative or tunnel boundary? RFC 9435's seven documented re-marking behaviours (`3-QoS-Classification-Trust` §7) and RFC 2983's Uniform/Pipe tunnel models (`14-QoS-Tunnels-Overlays` §2) are the two places this most often happens silently.
- **Compare the DSCP at ingress to the DSCP at egress** of every administrative domain the path crosses — this single comparison, done hop by hop, will surface almost every marking-related fault.

### 2.4 Policing and shaping (notes 7–9)

- **Which meter algorithm, and which parameters** — srTCM, trTCM, or the RFC 4115 variant (`8-QoS-Policing` §3–5)? A misdiagnosed meter type will make burst behaviour look inexplicable when it's actually predictable from the RFC's own worked formulas.
- **CBS/PBS sized correctly**: is the burst size at least as large as the biggest packet expected in that class? An undersized bucket makes *every* such packet fail the conformance test, every time (`8-QoS-Policing` §7.3).
- **Shaper buffer sized correctly**: is it at least Bc+Be, or is the "shaper" actually behaving like a policer under burst load because its buffer overflows (`9-QoS-Shaping` §3.2, §7 gotcha #5)?
- **Re-mark vs. drop**: does the policy re-mark exceeding traffic (common for AF-class handling, `6-QoS-Class-Design` §3.3) or drop it outright (`8-QoS-Policing` §4.7)? Confusing the two produces very different-looking symptoms for the same underlying policer.

### 2.5 Queuing, scheduling, and AQM (notes 10–11)

- **Is the packet in the queue believed to be configured** — verify by checking actual queue-depth/drop counters per class, not just the configuration syntax.
- **Priority Queuing starvation** (`10-QoS-Queuing-Scheduling` §4.3): is a lower-priority queue being serviced at all, or is an unpoliced high-priority queue consuming 100% of the link?
- **DRR quantum sizing** (`10-QoS-Queuing-Scheduling` §8.1): if a queue's throughput seems lower than its configured weight implies, is its quantum smaller than its typical packet size, causing multi-round stalls?
- **AQM threshold vs. actual link rate** (`11-QoS-Congestion-Avoidance` §8.2): classic RED's minth/maxth are configured in packets/bytes, which must be re-tuned per link speed — a threshold copied from a different-speed link is a common, easily overlooked cause of either too-aggressive or ineffective dropping.
- **Per-DSCP AQM actually configured**: recall from `7-QoS-Policy-Model` §7.2 that an Algorithmic Dropper cannot classify on its own — if a queue carries a mix of DSCPs (as every AF class does by design) and drop behaviour isn't differentiated, check whether the per-DSCP thresholds were actually configured, not just assumed from the class design.

### 2.6 ECN (notes 12–13)

- **Negotiation actually completed**: for TCP, was this an ECN-setup SYN/SYN-ACK exchange (`12-QoS-ECN` §3.2), or did a middlebox silently break it, falling back to non-ECN operation (`12-QoS-ECN` §4)?
- **CE being generated, but is anything listening?** A router can mark CE correctly, but if the endpoint stack doesn't support ECN, or a device on the path clears the ECN field, the signal is lost — check both ends, not just the marking device.
- **L4S/ECT(1) confusion** (`13-QoS-L4S-AccECN` §3.4): if any devices on the path are DSCP-only classifiers unaware of ECN semantics, they will never distinguish ECT(1)-marked L4S traffic from ordinary traffic — verify this is the *intended* behaviour (DualQ needs no deeper inspection) and not a sign that L4S traffic is being mishandled.

---

## 3. Measurement — Using the Right Metric, the Right Way

### 3.1 Recap: which parameter is actually being reported

`1-QoS-Fundamentals` §2 established four parameters (bandwidth, delay, jitter, loss) and, critically, §4.1 established that **"jitter" from different tools is not the same measurement** — RFC 3393's IPDV, RFC 3550's smoothed mean deviation, and a simple peak-to-minimum range are three different numbers that can legitimately disagree by an order of magnitude for the identical traffic. **The single most common troubleshooting error in this category is comparing two "jitter" numbers from different tools as if they measured the same thing.** Before treating a jitter complaint as a real problem, confirm which of the three definitions the reporting tool actually uses.

### 3.2 Standards-defined measurement tools

| Tool / RFC | What it measures | Scope |
|---|---|---|
| **RFC 7679** (one-way delay) / **RFC 7680** (one-way loss) | IPPM Standards Track metrics — precise, per-packet definitions | End-to-end, active measurement, needs synchronized or correlatable clocks (`1-QoS-Fundamentals` §9.4) |
| **RFC 2544** | Device benchmarking: throughput, latency, frame loss rate, back-to-back frame handling | `[Guidance]`, single-device or lab characterization — **not** a live-network troubleshooting tool |
| **RFC 9341** (Alternate-Marking Method, obsoletes RFC 8321) | Packet loss, delay, and jitter measurement on **live production traffic**, without needing a separate active test stream | Passive or hybrid; splits the flow into time-based "batches" using a color bit, and compares counters at two points along the path |

### 3.3 RFC 2544 — what it is and, just as importantly, what it isn't

`[Guidance]` RFC 2544 Abstract, quoted directly: *"This document discusses and defines a number of tests that may be used to describe the performance characteristics of a network interconnecting device."* Its own introduction states its motivation bluntly: *"Vendors often engage in 'specsmanship'... This document defines a specific set of tests that vendors can use to measure and report the performance characteristics of network devices."* This is a **device benchmarking** methodology — comparing Vendor A's box against Vendor B's box under controlled lab conditions (a specific frame-size sweep, a defined trial duration, "SHOULD be at least 120 seconds" per trial) — **not** a live-traffic diagnostic tool. Using RFC 2544-style single-device throughput/latency/loss numbers to explain a *live network's* QoS complaint conflates two different questions: "how fast is this box, in isolation" versus "why is this specific traffic flow, right now, on this specific path, experiencing a problem."

> ⚠️ **Gotcha:** RFC 2544 predates many modern protocols and traffic patterns — later documents (RFC 5180 for IPv6, RFC 5695 for MPLS, and further work on containerized/NFV environments) extend it, confirming that the original 1999 methodology needed supplementing rather than being treated as permanently complete. Citing "RFC 2544 numbers" for a modern, encrypted, tunneled, or virtualized path without checking whether a more specific companion document applies is the same "check for later updates" discipline already emphasized in `6-QoS-Class-Design` §7.2 for RFC 4594/RFC 8622.

### 3.4 RFC 9341 (Alternate-Marking) — measuring live traffic without a separate test stream

`[Standard-defined]` RFC 9341 (December 2022, obsoleting the Experimental RFC 8321) Abstract, quoted directly: *"This document describes the Alternate-Marking technique to perform packet loss, delay, and jitter measurements on live traffic."* The mechanism, quoted and confirmed directly: the flow is split into consecutive **"batches"** — a bit in the packet header is **toggled** (alternated) at fixed time intervals, so every device along the path can unambiguously tell which batch a given packet belongs to; comparing packet **counts** for the same batch at two different points along the path directly yields the packet loss between those two points, with no need to correlate individual packet identifiers.

**Why this matters for troubleshooting specifically** — it directly solves a problem every earlier note in this series has run into implicitly: measuring **in-path, live production traffic** (as opposed to injecting a synthetic RFC 2544-style test stream) has historically required either intrusive packet tagging or statistical sampling. Alternate-Marking's batching approach, quoted and confirmed directly, needs only loose clock synchronization for loss and two-way delay measurement (accuracy of **±L/2**, where L is the batch duration — e.g., ±0.5 second for a 1-second batch) and requires tight synchronization only for **one-way** delay measurement specifically. This is a direct, practical answer to `1-QoS-Fundamentals` §9.4's observation that one-way delay measurement is fundamentally harder than round-trip measurement because it needs synchronized clocks — Alternate-Marking is one standards-defined way to make that requirement as loose as possible for the loss and two-way-delay cases.

> 📝 A specific, confirmed operational detail worth knowing: because Alternate-Marking works by toggling a bit (commonly implemented using a spare bit in the DSCP/marking field, in some deployments), a monitoring domain's egress node can **restore the original DSCP value** before the packet leaves the monitored section — confirmed directly, this makes the technique *"completely transparent outside its monitoring domain,"* meaning devices beyond the measured section never see any evidence that marking-based measurement was happening at all. This is a direct, practical application of `3-QoS-Classification-Trust` and `14-QoS-Tunnels-Overlays`'s repeated theme that a marking's meaning is scoped to a specific domain — here, deliberately and by design.

---

## 4. Symptom → Likely Cause — A Practical Lookup Table

Built entirely from facts already established across this series; each row names the specific note and section where the underlying mechanism is fully explained.

| Symptom | Most likely stage | Specific things to check |
|---|---|---|
| Voice/video quality is fine on the LAN, degrades over the WAN | Serialization delay, or missing priority queue on the slower link | `1-QoS-Fundamentals` §3.2 (serialization math); `10-QoS-Queuing-Scheduling` §4 (is EF actually in a Priority queue on the WAN egress?) |
| A specific application's traffic isn't getting the class it's marked for | Classification/marking mismatch, mid-path | `3-QoS-Classification-Trust` §3 (completeness/precedence); `4-QoS-Marking-Headers` §7 (does the marking survive every layer boundary on the path?) |
| Traffic marked correctly at the source arrives with DSCP 0 (or some other unexpected value) at the destination | Bleaching or re-marking crossing an administrative or tunnel boundary | `3-QoS-Classification-Trust` §7 (RFC 9435's seven behaviours); `14-QoS-Tunnels-Overlays` §2 (Uniform vs. Pipe model — check which one the tunnel is actually using) |
| A "priority" class is starving everything else | Unpoliced strict Priority Queue | `10-QoS-Queuing-Scheduling` §4.3 (PQ needs external policing/admission control, always) |
| A class's throughput is lower than its configured bandwidth share | DRR quantum too small for the traffic's packet size, or ETS/WRR percentage misconfigured | `10-QoS-Queuing-Scheduling` §8.1; `17-QoS-Datacenter` §3.2 (if this is a DCB/data-center link) |
| Bursty traffic is dropped even though average utilization looks low | Undersized policer burst size (CBS/PBS), or undersized shaper buffer, or a microburst invisible to 5-minute utilization averages | `8-QoS-Policing` §7.3; `9-QoS-Shaping` §3.2; `1-QoS-Fundamentals` §6 (utilization averages hide microbursts) |
| Latency is fine on average but occasionally spikes for seconds at a time | Bufferbloat — an oversized buffer hiding delay until it finally overflows | `11-QoS-Congestion-Avoidance` §2.2; consider whether a modern AQM (CoDel/PIE, §5) is even configured |
| TCP throughput on a long-haul link is far below the link's rated capacity | Loss (even a small amount) interacting with RTT, per the Mathis-model relationship | `1-QoS-Fundamentals` §9.2 — recompute the theoretical bound for the actual RTT and loss rate before assuming a QoS misconfiguration |
| ECN appears configured on routers, but end hosts never seem to react to it | Negotiation never completed (middlebox interference), or DSCP-byte-vs-ECN-bits confusion in a classifier along the path | `12-QoS-ECN` §4 (documented middlebox failure, with RFC-mandated fallback); §2.2's whole-byte-match gotcha |
| A device's own management/SSH/routing sessions become unreliable under high transit traffic load | Missing or misconfigured control-plane protection | `16-QoS-Control-Plane` §1.2 — confirm this is even a control-plane-destined traffic problem, not a transit QoS problem (they are structurally separate, §3.1 of that note) |
| RDMA/storage traffic on a converged Ethernet fabric experiences unpredictable stalls or even total lockup | PFC-induced deadlock, or PFC used without a proactive congestion-control layer | `17-QoS-Datacenter` §2.3, §5.2 — is DCQCN (or an equivalent ECN-based proactive mechanism) actually keeping PFC pauses rare, or is PFC the *only* congestion mechanism in play? |
| Voice quality is poor specifically over Wi-Fi, though wired performance is fine | Default DSCP→UP truncation misrouting EF into the wrong Access Category | `18-QoS-Wireless` §3.2, §5.2 — verify the AP is actually applying RFC 8325's recommended UP 6 mapping, not the default UP 5 |
| Two "jitter" measurements from different tools disagree substantially for the same traffic | Comparing different jitter *definitions*, not a real discrepancy | `1-QoS-Fundamentals` §4.1 — confirm which of the three definitions (RFC 3393 IPDV / RFC 3550 mean deviation / peak-to-minimum) each tool is actually reporting |

---

## 5. CCIE-Depth Topics

### 5.1 Why "it works in the lab but not in production" is often a measurement-methodology gap, not a configuration gap

Section 3.3 already established that RFC 2544 characterizes a **device in isolation**, under controlled, repeatable lab conditions. A device that scores well on every RFC 2544 metric can still perform poorly in production for reasons the benchmark never tests: real traffic mixes many classes simultaneously (RFC 2544's trials are typically single-stream or controlled-mix, not the organic, bursty multi-class mixture of live traffic), real paths include multiple hops each with their own queuing behaviour (RFC 2544 primarily characterizes one device), and real traffic includes the microbursts and TCP dynamics (`1-QoS-Fundamentals` §9.2's Mathis-model interaction) that a synthetic, evenly-paced test stream doesn't reproduce. This is precisely why RFC 9341-style live-traffic measurement (§3.4) and lab benchmarking (§3.3) answer **different questions** and neither substitutes for the other.

### 5.2 The diagnostic value of checking DSCP at every hop, not just endpoints

Section 2.3's recommendation — compare DSCP at ingress and egress of every administrative domain — is worth justifying explicitly: because RFC 9435 (`3-QoS-Classification-Trust` §7) documents **seven distinct** re-marking behaviours, and because a bleach can occur due to old equipment, deliberate policy, or accidental misconfiguration (three different root causes producing visually similar symptoms), the *only* way to localize **where** a marking was altered is to check it at every boundary, not just compare source and final destination. A source-to-destination-only comparison tells you *that* something changed the marking somewhere on a potentially many-hop path; it cannot tell you *where*, and therefore cannot tell you *which* of RFC 9435's seven behaviours (or which specific device) is responsible.

### 5.3 Why a "policer that isn't dropping anything" can still be the problem

Extending `8-QoS-Policing` §7.1's strict-conformance point: a policer with CBS/PBS sized smaller than the traffic's actual largest packet will, per the algorithm's own logic, **mark that packet non-conforming (yellow or red) on every single occurrence** — but depending on how the policy is wired (`7-QoS-Policy-Model` §2.6, §7.1), "non-conforming" might mean **re-mark to a lower drop precedence** rather than outright drop. A monitoring dashboard showing "zero drops" on that policer can therefore still be hiding a real problem: every packet of that size is being systematically re-marked to a worse class, silently degrading its treatment at every subsequent hop, without ever showing up as a drop counter on the policer itself. Checking **conformance-level counters** (green/yellow/red, per `8-QoS-Policing` §3–5's terminology), not just drop counters, is necessary to catch this class of fault.

### 5.4 Why control-plane and transit QoS faults are easy to conflate, and why they shouldn't be

`16-QoS-Control-Plane` §3.1 already established that CoPP and transit QoS are **structurally separate** classification-and-policy paths, even though they reuse the same underlying toolkit. A device experiencing routing-protocol flaps *during* a period of high transit traffic volume can present symptoms that look identical whether the actual cause is (a) transit QoS correctly prioritizing other traffic *ahead of* CS6-marked control traffic on an egress interface, or (b) a control-plane protection policy that is itself too aggressive and dropping the control-plane's own routing-protocol packets before they ever reach the CPU. Distinguishing these requires checking **which side of the forwarding-plane/control-plane boundary** (`16-QoS-Control-Plane` §1.1's Figure 1 concept) the drop is actually occurring on — a transit-egress-queue drop counter and a control-plane-filter drop counter are different counters, on different logical paths, even on the same physical device.

---

## 6. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | An earlier-stage fault **masks** everything downstream | Always verify the datapath in order — classify, mark, trust, police/shape, queue/schedule, AQM/ECN, measure |
| 2 | "Jitter" numbers from different tools may use different definitions entirely | Confirm the definition before treating a discrepancy as a real problem (`1-QoS-Fundamentals` §4.1) |
| 3 | A classifier matching the whole ToS byte instead of the 6-bit DSCP silently misclassifies ECN-marked traffic | The single most common "why isn't my marking being recognized" root cause |
| 4 | RFC 2544 characterizes a **device**, not a **live network path** | Don't substitute lab benchmark numbers for live-traffic diagnosis |
| 5 | A policer showing zero drops can still be silently re-marking every oversized packet to a worse class | Check conformance-level (green/yellow/red) counters, not just drop counters |
| 6 | DSCP bleaching/re-marking can only be **localized** by checking every hop, not just source and destination | RFC 9435 documents seven distinct behaviours with different root causes and different symptoms |
| 7 | Control-plane and transit QoS drops are on **structurally separate** paths, even on the same device | A routing-protocol flap under load could be either — check which counter is actually incrementing |
| 8 | RFC 9341 (2022) obsoletes RFC 8321 (2018) — the *mechanism* is the same, but the citation and standards status changed | The same "check for later updates" discipline this series has applied to RFC 4594/8622, RFC 2680/7680, and others |

---

## 7. Quick Recap

| Concept | One-line answer |
|---|---|
| The systematic method | Walk the RFC 3290 datapath: classify → mark → trust → police/shape → queue/schedule → AQM/ECN → measure |
| Why order matters | An earlier-stage fault masks everything downstream from it |
| Most common marking-related fault | Whole-byte matching instead of 6-bit DSCP matching, breaking on any ECN-marked packet |
| Most common policing-related fault | CBS/PBS smaller than the largest expected packet — silent, systematic non-conformance |
| Where to check for bleaching | Every administrative/tunnel boundary, not just source and destination |
| RFC 2544's role | Lab device benchmarking — not a live-network diagnostic tool |
| RFC 9341's role | Live production traffic measurement (loss/delay/jitter) via a batching/coloring technique, no synthetic test stream needed |
| Jitter troubleshooting rule | Always confirm which of the three jitter definitions (RFC 3393 / RFC 3550 / peak-to-min) a tool actually reports |
| Control-plane vs. transit faults | Structurally separate paths — check which specific counter is incrementing before assuming which policy is at fault |

---

## References

**Informational**
- RFC 2544 — Benchmarking Methodology for Network Interconnect Devices (obsoletes RFC 1944)
- RFC 6192 — Protecting the Router Control Plane (background, `16-QoS-Control-Plane`)

**Standards Track**
- RFC 7679 — A One-Way Delay Metric for IPPM
- RFC 7680 — A One-Way Loss Metric for IPPM
- RFC 9341 — Alternate-Marking Method (obsoletes the Experimental RFC 8321)

**Referenced (this note is a synthesis of the entire series)**
- `1-QoS-Fundamentals` through `18-QoS-Wireless` — every stage of the systematic method, and every row of the symptom table, traces back to a specific, previously-verified mechanism earlier in this series.
