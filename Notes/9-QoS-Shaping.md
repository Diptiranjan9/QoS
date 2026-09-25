## QoS Shaping — Bc, Be, Tc, and Why Shaping Isn't Policing

> 💡 **TL;DR:** Shaping and policing both compare traffic against a rate contract, but they respond differently to excess: a **policer drops or re-marks** what doesn't conform (`8-QoS-Policing`); a **shaper buffers** it and releases it later, smoothing bursts instead of discarding them. RFC 3290 states this precisely (`7-QoS-Policy-Model` §2.6): shaping is modeled as a **property of a queuing element** — specifically, a **non-work-conserving scheduler** that can delay a packet's departure past its turn. The classic maths behind shaping — **Bc** (committed burst), **Be** (excess burst), **Tc** (time interval) — comes from Frame Relay (ANSI T1.606 / CCITT I.233.1), documented in IETF terms by RFC 3133, and the relationship **Tc = Bc / CIR** is the single formula that makes every other shaping calculation in this note derivable.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE / ANSI-CCITT · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 3133 (Frame Relay terminology, including Bc/Be/Tc) — Informational, but its definitions trace to ANSI T1.606 and CCITT I.233.1. RFC 2963 (rate-adaptive shaper) — Informational. RFC 3290 (shaping-as-queuing-property model) — Informational.

---

## 1. Shaping vs. Policing — the One Fact That Explains Everything Else

From `7-QoS-Policy-Model` §2.6, RFC 3290's own definition of **Policing**: *"comparing the arrival of data packets against a temporal profile and forwarding, delaying, or dropping them so as to make the output stream conformant to the profile."* Notice the definition already contains **both** policing and shaping as special cases — the difference is entirely in which action the "non-conforming" branch takes:

```
                          +-------+
              +--conform->| Queue | --> out, on schedule
              |           +-------+
  packets --->| Meter |
              |         +-conform?--> NO -->  [ POLICING: Absolute Dropper ]  -- discarded, gone
              +-------+                  -->  [ SHAPING:  delaying Queue   ]  -- buffered, sent LATER
```

`[Guidance]` RFC 3290 explicitly states: *"Shaping, sometimes considered as a TC [traffic conditioning] action, is treated as a function of queuing elements... in this model"* — not as an action element alongside marking and dropping. This is the formal reason a shaper always implies a **buffer** (packets that don't fit the current rate wait somewhere) while a pure policer never does (packets that don't conform are gone immediately, at zero delay cost).

| | **Policing** | **Shaping** |
|---|---|---|
| Excess traffic | Dropped or re-marked | Buffered, sent later |
| Needs a buffer? | No | **Yes** — this is the defining trait |
| Effect on burstiness | None — conforming traffic passes through unchanged | **Smooths** bursts into a steadier stream |
| Adds delay? | No (to conforming traffic) | **Yes**, to whatever is held back |
| RFC 3290 model | Meter → Absolute Dropper | A **non-work-conserving** queuing element |
| Typical placement | Ingress (enforce a contract before wasting resources) | Ingress (match a downstream bottleneck) or egress |
| Effect on TCP | Loss triggers TCP's congestion-avoidance backoff (can be abrupt) | Delay, not loss — TCP's flow generally continues, just slower |

> 📝 **The "non-work-conserving" term, precisely.** RFC 3290 defines this scheduler property (already introduced in note 7 §2.5) as one that *"services packets no sooner than a scheduled departure time, even if this means leaving packets queued while the output... is idle."* A shaper, by definition, is willing to let the outbound link sit idle rather than send a packet early — this is exactly what "smoothing" means mechanically: the shaper is refusing to release traffic faster than its configured rate, even when it has the opportunity to.

---

## 2. Where Bc, Be, Tc Come From

`[Guidance]` These three parameters did not originate in an IP QoS RFC — they are **Frame Relay** traffic-contract terms, standardized in **ANSI T1.606** and **CCITT (ITU-T) I.233.1**, and later documented for IETF purposes by **RFC 3133** ("Terminology for Frame Relay Benchmarking"). Because Frame Relay's shaping model became the template most router vendors adopted for generic traffic shaping (including on non-Frame-Relay interfaces), this vocabulary persists across the industry as the common way to describe and configure a shaper, independent of the underlying Layer 2 technology.

### 2.1 The three parameters (RFC 3133 §1.2, verified definitions)

| Parameter | RFC 3133's definition (paraphrased, values preserved exactly) |
|---|---|
| **CIR** — Committed Information Rate | The guaranteed data rate between two endpoints under normal conditions. Data above CIR but below EIR (Excess Information Rate) may be marked Discard Eligible and may be dropped. |
| **Bc** — Committed Burst Size | The **maximum amount of data (in bits)** the network commits to transfer, under normal conditions, during time interval **Tc**. |
| **Be** — Excess Burst Size | The maximum amount of **uncommitted** data (in bits), in excess of Bc, that the network will **attempt** to deliver during Tc. Treated as discard-eligible. |
| **Tc** — Committed Rate Measurement Interval | The time interval during which the endpoint may send Bc committed data and Be excess data. |

### 2.2 The defining formula — Tc = Bc / CIR

RFC 3133 §1.2.7, quoted directly: *"Tc is computed (from the subscription parameters of CIR and Bc) as **Tc = Bc/CIR**. Tc is not a periodic time interval... it is used only to measure incoming data, during which it acts like a sliding window."*

Rearranged, this single relationship gives every other quantity:

```
   CIR = Bc / Tc          (the rate is "how much committed data, over how much time")
   Bc  = CIR * Tc
   Tc  = Bc / CIR
```

> ⚠️ **Gotcha — Tc is not periodic; it's a sliding window.** RFC 3133 is explicit that Tc is *"not a periodic time interval"* but is instead **triggered by incoming data** and measured as a sliding window from that point. Thinking of Tc as "the shaper wakes up every Tc milliseconds on a fixed clock" is a common but inaccurate mental model — verify against your specific platform's documentation for how it actually implements the interval, since RFC 3133 describes the *concept*, and different implementations may approximate it differently `[Implementation-dependent]`.

### 2.3 Worked example

CIR = 128,000 bps, Bc = 8,000 bits (1,000 bytes).

```
 Tc = Bc / CIR = 8,000 / 128,000 = 0.0625 s = 62.5 ms
```

So a shaper configured for CIR=128 kbps with Bc=8,000 bits releases up to 8,000 bits **every 62.5 ms** to average out to exactly 128 kbps over time — it does not have to send at a perfectly constant instantaneous rate, only average to CIR across each Tc window.

### 2.4 Adding Be — the mean-rate formula

`[Common practice]`, directly derivable from RFC 3133's definitions: if a shaper is also configured with an excess burst allowance Be (data the network will *attempt*, not guarantee, to carry), the **mean rate** achievable across a Tc window becomes:

```
 Mean rate = (Bc + Be) / Tc
```

Worked example, extending §2.3: Bc = 8,000 bits, Be = 8,000 bits, Tc = 62.5 ms (unchanged, since Tc is derived from Bc and CIR, not from Be):

```
 Mean rate = (8,000 + 8,000) / 0.0625 = 16,000 / 0.0625 = 256,000 bps = 256 kbps
```

So this shaper guarantees 128 kbps (CIR) but can sustain up to 256 kbps of throughput if the network downstream has room for the excess — exactly twice the CIR in this example, because Be was set equal to Bc.

---

## 3. Applying This to a Router-Based Shaper (Common Practice, Not Frame-Relay-Specific)

`[Common practice]` Modern shaping on a router interface (not necessarily a Frame Relay circuit) borrows this exact vocabulary. A widely-used, vendor-documented rule of thumb: **Tc is deliberately kept short** — commonly configured or defaulting somewhere in the **10 ms to 125 ms** range — because a shorter Tc means smaller, more frequent bursts, which produces smoother output and less added delay per burst, at the cost of more frequent scheduling decisions. This specific numeric range (10–125 ms) is a widely used **implementation default/limit on Cisco platforms**, not an IETF-mandated value — `[Implementation-dependent]`, included here because it's the range most engineers will encounter in practice, not because any RFC specifies it.

### 3.1 Why a shorter Tc produces smoother traffic — worked comparison

Using CIR = 512,000 bps, compare Bc calculated for two different target Tc values (Bc = CIR × Tc, from §2.2):

| Target Tc | Bc = CIR × Tc | What this means in practice |
|---|--:|---|
| 125 ms | 512,000 × 0.125 = **64,000 bits** (8,000 bytes) | Up to 8,000 bytes can leave the shaper in one instantaneous burst, every 125 ms |
| 10 ms | 512,000 × 0.010 = **5,120 bits** (640 bytes) | Only 640 bytes per burst, but bursts happen far more often — smoother output, lower peak instantaneous rate |

Same **average** rate (512 kbps) either way — §2.2's formula guarantees that, since CIR is unchanged — but the **instantaneous** burst size at the shaper's output differs by more than 12×. This is the direct, calculable reason shaper tuning discussions focus on Tc: it trades off burst size against scheduling frequency while holding the long-term average constant.

### 3.2 Sizing the buffer

`[Common practice]` A shaper's buffer must be large enough to hold at least **Bc + Be** bits without overflowing during a single Tc window's worth of burst, or the shaper degenerates into a policer (dropping, rather than delaying, the excess) at exactly the moments it's supposed to be smoothing. Undersizing this buffer is a common, entirely avoidable cause of a "shaper" that behaves like a lossy policer under bursty load.

---

## 4. Rate-Adaptive Shaping — RFC 2963

### 4.1 Why a fixed-rate shaper isn't always enough

`[Guidance]` RFC 2963 (Bonaventure & De Cnodder, 2000) observes that a classical shaper — fixed rate, discards on overflow — has a specific weakness when placed upstream of a marker (srTCM/trTCM, `8-QoS-Policing`): it doesn't know the meter's state, so it can **unnecessarily delay a packet even when the downstream meter has plenty of token-bucket credit available to mark it green**. For TCP traffic specifically, that unnecessary delay slows the growth of TCP's congestion window, hurting throughput for no real benefit.

### 4.2 The Rate Adaptive Shaper (RAS) design

RFC 2963's shaper is deliberately **not** a classical leaky-bucket shaper. Quoted directly: *"The main objective of the shaper is to produce at its output a traffic that is less bursty than the input traffic, but the shaper [avoids discarding] packets in contrast with classical token bucket based shapers. The shaper itself consists of a tail-drop FIFO queue which is emptied at a variable rate."*

```
 Incoming Packet ==> [ Rate Adaptive Shaper ] ==> [ Meter (srTCM/trTCM) ] ==> [ Marker ] ==> Outgoing Stream
                      (variable-rate FIFO,
                       placed UPSTREAM of the meter —
                       shapes BEFORE metering, not after)
```

RFC 2963 §2.2 gives the srRAS (single-rate version) four configuration parameters: **CIR**, **MIR** (Maximum Information Rate — a ceiling, typically the shaper's output link rate), and two buffer-occupancy thresholds, **CIR_th** and **MIR_th**. The shaping rate is **variable**, calculated as a function of the queue's own occupancy and the estimated incoming data rate — the RFC states the shaping rate is *"the maximum of the estimated average incoming data rate and some [threshold-driven minimum]"* rather than a single fixed number.

### 4.3 The "green RAS" refinement

`[Guidance]` RFC 2963 §3 goes further: a basic RAS still doesn't know whether the *meter* currently has room to mark a packet green, so it can still add avoidable delay. The **green RAS** fixes this by coupling the shaper directly to the meter's live status:

```
                        Status      Result
                       +--------+  +----------+
                       |        |  |          V
   Incoming    +--------+     +-------+    +--------+   Outgoing
   Packet  ==> | green  |====>| Meter |===>| Marker |==>  Packet
               | RAS    |     +-------+    +--------+   Stream
               +--------+
```

If the meter reports it currently has token credit to mark a packet green, the green RAS releases it **immediately**, bypassing the shaping delay that would otherwise apply — because there is no benefit to smoothing a packet that was already going to be marked favorably. RFC 2963 states this specifically targets improving TCP goodput and **reducing the number of marked (yellow/red) packets**, since fewer unnecessary delays mean TCP's own congestion-avoidance behaviour is disturbed less.

### 4.4 Where this fits in the vendor-neutral model

RFC 2963's shaper-then-meter ordering is worth noting explicitly against `7-QoS-Policy-Model`'s general order-of-operations (§4 of that note): the *general* order given there is classify → meter → mark → queue, but RFC 2963 states plainly that its Rate Adaptive Shapers *"are thus different from the shapers described in [RFC2475] since they shape the traffic **before** the traffic is metered."* This is a deliberate, documented exception to the usual ordering, justified by the specific goal of protecting TCP performance — not a contradiction of the general model, but a reminder that **note 7's ordering is the common case, not an absolute rule**; RFC 3290's model permits any wiring the equipment implements.

---

## 5. Shaping and Hierarchy — Recap from Note 7

`[Common practice]` §6 of `7-QoS-Policy-Model` already established that hierarchy is nothing more than a scheduler's output feeding a further queue/scheduler stage. Shaping is the most common reason to build such a hierarchy: a **parent shaper** enforces an overall contracted rate (e.g., a sub-rate WAN circuit), while a **child policy** determines how that shaped rate is shared among classes (e.g., EF still gets priority *within* the shaped envelope). Nothing new is needed here beyond what note 7 already covers — this note supplies the Bc/Be/Tc maths that determines *how smooth* the parent shaper's output actually is, which the child policy's classes then compete for.

---

## 6. CCIE-Depth Topics

### 6.1 Why shaping to exactly link speed is meaningless

If Bc/Tc/CIR are configured such that the shaper's rate equals the physical link's own rate, the shaper cannot do anything a plain FIFO wouldn't already do — there's no "excess" to smooth, since the link itself is the bottleneck and nothing arrives faster than it can leave. Shaping only has an effect when its configured rate is **below** some downstream capacity (a sub-rate access circuit, a slower remote-site link, a provider-enforced CIR) — the shaper exists specifically to match the sender's output to a bottleneck that is *not* the local physical interface.

### 6.2 Interaction between shaping delay and jitter-sensitive traffic

Because a shaper's entire mechanism is *deliberately delaying* packets to smooth bursts, naively shaping a mixed-class aggregate (voice + bulk data together) reintroduces exactly the queuing-delay variability that `1-QoS-Fundamentals` identified as the one delay component QoS is supposed to control. This is why, per §5 above, a shaped hierarchy almost always needs an inner priority mechanism for delay-sensitive classes — shaping alone, applied uniformly, can turn a smooth-average-rate problem into a jitter problem for the traffic least able to tolerate it.

### 6.3 Be's "attempt, not guarantee" wording has a real consequence

RFC 3133's definition of Be says the network will "attempt" to deliver it — it is explicitly **not** part of the committed contract. A shaper's Be setting therefore represents *opportunistic* capacity, not a number safe to provision applications against. Designing a service around "Bc + Be" as if it were a guaranteed rate contradicts the very definition of Be; only Bc (via CIR = Bc/Tc) carries a delivery commitment.

### 6.4 The token-bucket-vs-leaky-bucket terminology overlap

RFC 2963 uses "leaky-bucket" as its term for the classical (non-adaptive) shaper it's improving on, while RFC 2697/2698 (`8-QoS-Policing`) and RFC 3290 use "token bucket" for the metering mechanism. These describe **related but distinct** bucket analogies — a token bucket accumulates *credit* to admit bursts up to a size, while a leaky-bucket shaper's queue *drains* at a fixed or variable rate regardless of how it fills. Conflating the two names across different documents is a common source of confusion; each RFC in this series is cited with its own specific terminology intact rather than normalized to a single word.

---

## 7. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | Shaping buffers excess traffic; policing drops or re-marks it | The presence of a buffer (and therefore added delay) is the defining structural difference |
| 2 | Tc is a **sliding window triggered by traffic**, not a periodic fixed-clock interval | RFC 3133 states this explicitly — don't assume a fixed-tick implementation |
| 3 | Be represents capacity the network will only **attempt** to deliver | Never provision a service against Bc+Be as if it were a committed rate |
| 4 | A shorter configured Tc produces smaller bursts at the **same** average rate | The 12×+ burst-size difference in §3.1's worked example, for identical CIR |
| 5 | An undersized shaper buffer (smaller than Bc+Be) turns a shaper into a de facto policer | Bursts overflow and are dropped instead of smoothed |
| 6 | RFC 2963's Rate Adaptive Shaper deliberately shapes **before** metering | A documented exception to the usual classify→meter→mark→queue order from note 7 |
| 7 | Shaping to exactly the physical link rate accomplishes nothing | A shaper only matters when its rate is below some real downstream bottleneck |
| 8 | Uniformly shaping a mixed voice+data aggregate can reintroduce the jitter QoS was meant to remove | Shaped hierarchies still need an inner priority mechanism for delay-sensitive classes |

---

## 8. Quick Recap

| Concept | One-line answer |
|---|---|
| Shaping vs. policing | Shaping buffers and delays excess; policing drops or re-marks it |
| RFC 3290's model of shaping | A property of a queuing element (non-work-conserving scheduling) |
| Origin of Bc/Be/Tc | Frame Relay (ANSI T1.606 / CCITT I.233.1), documented for IETF purposes by RFC 3133 |
| The defining formula | Tc = Bc / CIR |
| Mean rate with excess burst | (Bc + Be) / Tc |
| Common Tc range in practice | Roughly 10–125 ms `[Implementation-dependent]` |
| Why shorter Tc = smoother output | Smaller Bc per interval, at the same average CIR |
| RFC 2963's Rate Adaptive Shaper | Variable-rate FIFO, placed upstream of a meter, avoids discarding packets |
| "Green RAS" | RAS coupled to live meter status — skips shaping delay for packets already guaranteed green |

---

## References

**Informational (IETF)**
- RFC 3133 — Terminology for Frame Relay Benchmarking (Bc, Be, Tc, CIR definitions; Tc = Bc/CIR)
- RFC 2963 — A Rate Adaptive Shaper for Differentiated Services
- RFC 3290 — An Informal Management Model for Diffserv Routers (shaping-as-queuing-property, non-work-conserving scheduling; background from `7-QoS-Policy-Model`)
- RFC 2475 — An Architecture for Differentiated Services (referenced by RFC 2963 as the shaper model it deliberately departs from)

**Underlying standards (Frame Relay, referenced by RFC 3133)**
- ANSI T1.606 — Frame Relay bearer service, architectural framework and service description
- CCITT/ITU-T I.233.1 — Frame Relay bearer service

**Referenced (full mechanism detail in other notes)**
- `7-QoS-Policy-Model` (policing/shaping definitions, order of operations, hierarchy)
- `8-QoS-Policing` (srTCM, trTCM — the meters RFC 2963's RAS is designed to sit upstream of)
- `10-QoS-Queuing-Scheduling` (full scheduler mechanics)
