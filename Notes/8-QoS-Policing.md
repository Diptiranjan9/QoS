## QoS Policing — Token Buckets, srTCM, trTCM

> 💡 **TL;DR:** Policing measures arriving traffic against a **rate contract** and marks each packet **green** (conforming), **yellow** (partially conforming / in excess), or **red** (non-conforming) — in RFC 3290's terms, a **Meter** whose outputs feed a **Marker** and, for red traffic, usually an **Absolute Dropper** (`7-QoS-Policy-Model` §2.6). Three standards-defined algorithms cover almost every real policer: **srTCM** (RFC 2697) — one rate, two burst sizes, for when only burst *length* matters; **trTCM** (RFC 2698) — two independent rates (peak and committed), for when a *separate* peak needs enforcing; **RFC 4115's two-rate marker** — same two-rate idea as trTCM, but re-ordered specifically to stop committed (in-profile) traffic from ever being penalized by the peak-rate test, which RFC 2698's own algorithm can do. All three are built from the **token bucket** concept: tokens accumulate at a rate, a packet needs enough tokens to be admitted, and whether unused tokens can be "borrowed" or carried forward is exactly what separates the three algorithms and produces the burst behaviour of a policer.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 2697 (srTCM), RFC 2698 (trTCM), RFC 4115 (two-rate marker with efficient in-profile handling) — all **Informational**. None of these are Standards Track, despite being implemented almost universally; RFC 4115 carries an unusual IESG disclaimer noted in §5.

---

## 1. Policing, Recap and Precise Definition

From `7-QoS-Policy-Model` §2.6, RFC 3290's own definition: **"Policing** [is] the process of comparing the arrival of data packets against a temporal profile and forwarding, delaying or dropping them so as to make the output stream conformant to the profile." This note covers the **measurement** half of that definition — the Meter — in full mechanical detail; what happens to non-conforming traffic (drop vs. delay vs. re-mark) was already established as a wiring choice in note 7, and shaping (the "delaying" case) gets its own note (`9-QoS-Shaping`).

```
                         +------------+
                         |   Result   |
                         |  (color)   V
                     +-------+    +--------+
   Packet Stream ===>| Meter |===>| Marker |===> Marked Packet Stream
                     +-------+    +--------+
```

This is the exact diagram RFC 2697 and RFC 2698 both use — policing is modeled identically in both documents as **Meter → Marker**, with what happens after marking (drop red, queue green/yellow differently) left to the surrounding policy.

---

## 2. The Token Bucket, From First Principles

`[Guidance]` (RFC 3290 §5, `7-QoS-Policy-Model` §2.2) A token bucket holds up to **B** tokens (the burst size), refilled at rate **R** (the sustained/average rate). A packet of size **L** bytes can only be admitted if the bucket currently holds at least L tokens' worth of credit; admission consumes that many tokens.

```
   tokens trickle in at rate R  --->  [ bucket, capacity B ]  ---> packet of size L needs
                                                                    >= L tokens to pass
```

Two conformance rules exist for a *simple* single-bucket meter (RFC 3290 §5.1.3, already introduced in note 7):

- **Strict conformance:** the packet needs the **full L tokens already present** — nothing may be borrowed from future refills. srTCM and trTCM are both built this way.
- **Loose conformance:** the packet passes if the bucket has **any** tokens at all, and may borrow up to L bytes from future allocations.

**Why burst size matters at all:** a token bucket with rate R and burst B allows a **sustained rate of R indefinitely**, but permits a burst of up to **B bytes to arrive instantaneously** (all at once, consuming the full bucket) without being penalized — because the bucket had time to fill during any preceding idle period. This single fact — *rate limits the average, burst size limits the momentary spike* — is the entire reason burst size is a separate, independently tunable parameter from rate in every algorithm below.

---

## 3. srTCM — Single Rate Three Color Marker (RFC 2697)

### 3.1 Purpose, in the RFC's own words

`[Guidance]` RFC 2697 Abstract: *"The srTCM is useful, for example, for ingress policing of a service, where only the length, not the peak rate, of the burst determines service eligibility."* One rate, two burst sizes — it distinguishes "a little over" from "way over" **purely by how big the burst is**, not by how fast it arrived.

### 3.2 Parameters and the two-bucket mechanism

| Parameter | Meaning |
|---|---|
| **CIR** — Committed Information Rate | The single rate shared by both buckets |
| **CBS** — Committed Burst Size | Size of bucket **C** |
| **EBS** — Excess Burst Size | Size of bucket **E** |

Both buckets fill at the **same rate, CIR**. RFC 2697 §3, verbatim: initially **Tc(0) = CBS** and **Te(0) = EBS** (both start full). Thereafter, the token counts are updated **CIR times per second** as follows:

```
 if Tc < CBS:        Tc += 1        (fill bucket C first)
 else if Te < EBS:    Te += 1        (only once C is full, start filling E)
 else:                (do nothing — both buckets already full)
```

> 📝 This ordering — **C fills before E ever gets a token** — is easy to misread as "independent" buckets. They are not: **E only accumulates credit once C is completely full.** This is the mechanical reason srTCM is a single-rate algorithm even though it has two buckets.

### 3.3 Marking algorithm (Color-Blind mode, RFC 2697 §3, verbatim logic)

A packet of size B bytes arriving at time t:

```
 if Tc(t) - B >= 0:        packet = GREEN;  Tc -= B (floor 0)
 else if Te(t) - B >= 0:   packet = YELLOW; Te -= B (floor 0)
 else:                     packet = RED;    neither bucket is touched
```

**Color-Aware mode** works identically, except the packet may already be pre-colored by an upstream device; a pre-colored **red** packet is never promoted, and a pre-colored **yellow** packet skips straight to the Te test (never checks Tc) — RFC 2697 leaves the exact pre-coloring mechanism as domain-specific and outside its scope.

### 3.4 Worked example

CIR = 1000 bytes/sec, CBS = 2000 bytes, EBS = 1000 bytes. Both buckets start full: Tc=2000, Te=1000.

| Event | Packet size | Test | Result | Tc after | Te after |
|---|--:|---|---|--:|--:|
| t=0 | 1500 B | Tc(2000)-1500=500 ≥ 0 | **GREEN** | 500 | 1000 |
| t=0 (same instant) | 1200 B | Tc(500)-1200=-700 < 0; Te(1000)-1200=-200 < 0 | **RED** | 500 | 1000 |
| t=0 (same instant) | 600 B | Tc(500)-600=-100 <0; Te(1000)-600=400 ≥ 0 | **YELLOW** | 500 | 400 |

This shows the core srTCM behaviour: a single **2000+1000 = 3000-byte instantaneous burst budget** exists (CBS+EBS) the first time the bucket is full, split by the marker into "definitely fine" (green, up to CBS) and "acceptable but marked" (yellow, the next EBS worth) — but only while C is being drawn down; once C is exhausted, further tokens accumulate in **E only**, so the sustained long-term rate is still governed by the single shared CIR.

> ⚠️ **Gotcha — a real, documented ambiguity in RFC 2697 itself:** Multiple implementers and analyses (cited by later patent literature reviewing RFC 2697) have pointed out that despite its name, **EBS is not "CBS plus some excess" and is not equal to PBS−CBS** — it is a completely independent bucket size. If EBS is configured expecting it to represent *additional* burst room on top of CBS, the actual maximum green+yellow burst achievable while both buckets are full is **CBS+EBS bytes**, which can be larger than a designer expects if they assumed EBS was meant to be small. Always treat CBS and EBS as two independent, directly-configured sizes — not "CBS plus a delta."

---

## 4. trTCM — Two Rate Three Color Marker (RFC 2698)

### 4.1 Purpose, in the RFC's own words

`[Guidance]` RFC 2698 Abstract: *"A packet is marked red if it exceeds the Peak Information Rate (PIR). Otherwise it is marked either yellow or green depending on whether it exceeds or doesn't exceed the Committed Information Rate (CIR)."* Unlike srTCM, trTCM has **two independent, differently-configured buckets and rates** — useful, per RFC 2698 §1, "for ingress policing of a service, where a peak rate needs to be enforced separately from a committed rate."

### 4.2 Parameters

| Parameter | Meaning |
|---|---|
| **PIR** — Peak Information Rate | Must be ≥ CIR |
| **PBS** — Peak Burst Size | Size of bucket **P** |
| **CIR** — Committed Information Rate | The committed (sustained) rate |
| **CBS** — Committed Burst Size | Size of bucket **C** |

Both are measured in **bytes of IP packets per second** (IP header included, link-layer headers excluded) — RFC 2698 §2, verbatim. RFC 2698 recommends PBS and CBS both be **configured ≥ the largest expected IP packet** in the stream.

### 4.3 Bucket refill (independent, unlike srTCM)

RFC 2698 §3, verbatim: Tp(0) = PBS and Tc(0) = CBS. Thereafter, **Tp is incremented by one PIR times per second** (up to PBS) and, **completely independently**, **Tc is incremented by one CIR times per second** (up to CBS). Unlike srTCM's cascaded C-then-E filling, **both buckets fill on their own schedule from the start.**

### 4.4 Marking algorithm — Color-Blind mode (RFC 2698 §3, verbatim logic)

```
 if Tp(t) - B < 0:            packet = RED                       (exceeds the peak — always red)
 else if Tc(t) - B < 0:       packet = YELLOW;  Tp -= B           (within peak, but exceeds committed)
 else:                        packet = GREEN;   Tp -= B; Tc -= B  (within both — decrement BOTH buckets)
```

> 📝 **The detail most summaries get wrong:** a **green** packet decrements **both** Tp and Tc. A **yellow** packet decrements **only** Tp (Tc is left untouched). This asymmetry is deliberate and exactly what RFC 4115 (§5 below) identifies as a design cost: every packet, green or yellow, draws down the peak bucket, so sustained yellow traffic competes with green traffic for the *same* Tp budget.

### 4.5 Marking algorithm — Color-Aware mode

Identical structure, but a packet **pre-colored red** is immediately red regardless of bucket state; a packet **pre-colored yellow** skips the Tp-only test and is evaluated only against Tc (still only decrementing Tp if it passes); a packet not pre-colored as red or yellow follows the color-blind logic. RFC 2698 §3 states explicitly: *"The actual implementation of a Meter doesn't need to be modeled according to the above formal specification"* — the algorithm is the **conformance definition**, not a mandated implementation.

### 4.6 Worked example

PIR = 2000 B/s, PBS = 3000 B; CIR = 1000 B/s, CBS = 1500 B. Both full at t=0: Tp=3000, Tc=1500.

| Packet | Size | Tp test | Tc test | Result | Tp after | Tc after |
|---|--:|---|---|---|--:|--:|
| 1 | 1200 B | 3000-1200=1800 ≥0 (pass) | 1500-1200=300 ≥0 (pass) | **GREEN** | 1800 | 300 |
| 2 | 1000 B | 1800-1000=800 ≥0 (pass) | 300-1000=-700 <0 (fail) | **YELLOW** | 800 | 300 (unchanged) |
| 3 | 900 B | 800-900=-100 <0 (fail) | — | **RED** | 800 (unchanged) | 300 (unchanged) |

Packet 2 illustrates the asymmetry directly: it is yellow (exceeded CIR budget) but still **spends peak-bucket credit**, leaving less Tp available for the packet that follows — which is exactly why packet 3, at only 900 bytes, goes red despite CBS/PBS looking generous on paper.

### 4.7 Marking application to AF PHB

RFC 2698 §4: the marker's color-to-DSCP mapping is domain-specific, but for **AF PHB (RFC 2597)**, "the color can be coded as the drop precedence of the packet" — i.e., green→low drop precedence (AFx1), yellow→medium (AFx2), red is usually dropped rather than marked AFx3 in many designs, though RFC 2698 leaves the exact mapping open. This is the mechanism behind the AF41/AF42/AF43 marking logic already covered from the RFC 4594 side in `6-QoS-Class-Design` §3.3.

---

## 5. RFC 4115 — Fixing trTCM's In-Profile Problem

### 5.1 The problem RFC 4115 identifies in RFC 2698, stated precisely

`[Guidance]` RFC 4115 §2 (quoted): in trTCM's algorithm, "traffic is marked green **after it passes two conformance tests** (those for PIR and CIR)." Because of this, RFC 4115 identifies **two specific failure modes**:

1. In either color mode, needing to pass **two** tests "could result in packets being dropped at the PIR token bucket **even though they are perfectly within their CIR**" (in-profile traffic).
2. In color-aware mode specifically, this "could make yellow traffic **starve** incoming in-profile green packets" — because (per §4.4/§4.6 above) yellow packets still consume Tp budget, competing with genuinely in-profile green traffic for the same peak-rate bucket.

This is a direct, RFC-documented consequence of the exact asymmetry flagged in the §4.4 gotcha above — RFC 4115 exists specifically to fix it.

### 5.2 RFC 4115's parameters and refill model

| Parameter | Meaning |
|---|---|
| **CIR** — Committed Information Rate | Token generation rate for bucket C |
| **CBS** — Committed Burst Size | Size of bucket C |
| **EIR** — Excess Information Rate | Token generation rate for bucket E (note: **EIR**, not PIR — different naming from trTCM) |
| **EBS** — Excess Burst Size | Size of bucket E |

RFC 4115 §3 also permits **linking** CIR/EIR via a single burst-duration parameter **T**, where **T = CBS/CIR = EBS/EIR** — letting an operator specify one duration instead of four independent numbers, when that simplification fits the deployment.

**Refill is periodic** (verbatim from the RFC's flowchart), unlike RFC 2697/2698's "N times per second" phrasing:

```
 every T seconds:
   Tc(t+) = MIN(CBS, Tc(t-) + CIR*T)
   Te(t+) = MIN(EBS, Te(t-) + EIR*T)
```

### 5.3 The reordered marking logic — the actual fix

The key structural change: **test the committed bucket first, and only fall through to the excess bucket if the committed test fails** — the reverse emphasis from trTCM, which always tests peak first. The RFC 4115 flowchart logic (color-blind mode):

```
 if B <= Tc(t):           packet = GREEN;  Tc -= B                 (committed budget covers it — done, no peak test at all)
 else if B <= Te(t):      packet = YELLOW; Te -= B                 (fell through to the excess bucket)
 else:                    packet = RED
```

Compare this directly against trTCM's §4.4 logic: **a packet that fits within Tc never touches the excess/peak bucket at all.** This is precisely what removes both failure modes RFC 4115 identified — in-profile (green) traffic is judged **solely** against Tc and can never be penalized by excess-bucket pressure, and yellow traffic draws only from **Te**, so it can never crowd out green traffic's budget the way yellow traffic could deplete Tp in trTCM.

### 5.4 Worked example, contrasted directly against trTCM's §4.6 numbers

Using equivalent parameters (CIR=1000 B/s, CBS=1500 B; EIR=2000 B/s "excess," EBS=3000 B — same numeric input as trTCM's example, relabeled):

| Packet | Size | Tc test | Result | Tc after | Te after |
|---|--:|---|---|--:|--:|
| 1 | 1200 B | 1200 ≤ 1500 | **GREEN** | 300 | 3000 (untouched) |
| 2 | 1000 B | 1000 ≤ 300? No → check Te: 1000 ≤ 3000 | **YELLOW** | 300 (untouched) | 2000 |
| 3 | 900 B | 900 ≤ 300? No → check Te: 900 ≤ 2000 | **YELLOW** (not RED!) | 300 (untouched) | 1100 |

**Packet 3 is yellow under RFC 4115's algorithm but was RED under trTCM's algorithm with the same input numbers (§4.6).** This is the concrete, worked demonstration of the exact problem RFC 4115 describes: trTCM's shared, peak-bucket-first testing let packet 2's yellow marking consume budget that then caused packet 3 to fail; RFC 4115's committed-bucket-first, separately-budgeted design does not have that interaction.

### 5.5 An important caveat about this document

`[Guidance]` RFC 4115 carries an unusual **IESG Note**, quoted directly: *"This RFC is not a candidate for any level of Internet Standard... the decision to publish is not based on IETF review for such things as security, congestion control, or inappropriate interaction with deployed protocols... Readers of this document should exercise caution in evaluating its value for implementation and deployment."* RFC 2697 and RFC 2698 carry no such disclaimer. This does not mean RFC 4115 is wrong — its algorithm is coherent and its stated rationale directly addresses a real, demonstrable gap in RFC 2698 — but it is a lower level of IETF review than its two companion documents, and should be cited as such.

---

## 6. Comparing the Three Algorithms

| | **srTCM (RFC 2697)** | **trTCM (RFC 2698)** | **RFC 4115** |
|---|---|---|---|
| Rates | One (CIR) | Two (CIR, PIR) | Two (CIR, EIR) |
| Buckets | Two, **cascaded** (C fills, then E) | Two, **independent** refill | Two, **independent** refill |
| What triggers RED | Exceeding CBS **and** EBS combined capacity | Exceeding **PIR** (peak) | Exceeding **both** CIR and EIR budgets |
| Test order | C first, then E | **P (peak) first**, then C | **C (committed) first**, then E |
| Green packet decrements | Tc only | **Both** Tp and Tc | Tc only |
| Yellow packet decrements | Te only | **Tp only** (not Tc) | Te only |
| In-profile (CIR-conforming) traffic can be penalized by peak-bucket pressure? | N/A (one rate only) | **Yes** (identified defect, §5.1) | **No** (fixed by design) |
| Distinguishes by | **Burst length** only | **Rate** (peak vs. committed) | **Rate**, with in-profile protection |
| Typical use (per the RFCs' own framing) | Burst-length-based service eligibility | Enforcing a separate peak rate from a committed rate | Data services (e.g., Frame Relay-style) needing committed traffic protected from excess-traffic interference |

---

## 7. CCIE-Depth Topics

### 7.1 Why "borrowing" from RFC 3290 explains all three algorithms at once

Recall `7-QoS-Policy-Model` §2.2's **strict vs. loose conformance** distinction (RFC 3290 §5.1.3). All three algorithms in this note (srTCM, trTCM, RFC 4115) are **strict-conformance** meters — a packet needs the **full byte count already present** in the relevant bucket; none of them allow borrowing from a future refill. RFC 3290 explicitly names RFC 2697 and RFC 2698 as its examples of strict-conformance meters. This is *why* all three produce clean green/yellow/red decisions per packet rather than the more complex accounting a loose-conformance, borrowing-capable meter would need.

### 7.2 The mathematical reason trTCM's asymmetry causes problems

In trTCM, decrementing **Tp on every green or yellow packet** means Tp's depletion rate, over any window, equals the **total** green+yellow throughput, not just the green throughput. If sustained yellow traffic exists, Tp depletes faster than CIR alone would predict, so its refill (at rate PIR) may not keep pace — pushing genuinely CIR-conforming traffic into the red zone purely because of **yellow traffic's presence**, exactly as RFC 4115 states. RFC 4115's fix works because Tc's depletion rate depends **only** on green traffic, decoupling the committed budget from whatever is happening in the excess/yellow tier.

### 7.3 Why CBS/PBS "≥ largest packet" is a stated requirement, not a suggestion

RFC 2698 §2 states PBS and CBS "must be configured to be greater than 0" and recommends (not mandates) they be "≥ the largest possible IP packet." If a bucket's capacity is smaller than one packet, **no packet of that size can ever pass**, regardless of how empty or full the bucket is or how generous the rate is — the strict-conformance test (§7.1) requires the *entire* packet's worth of tokens to be present simultaneously. This is a structural property of strict-conformance token buckets, not a tuning recommendation that can be safely ignored for "efficiency."

### 7.4 Linking parameters via burst duration T (RFC 4115 only)

RFC 4115's optional **T = CBS/CIR = EBS/EIR** linkage means an operator can specify a single "how many seconds of burst do I allow at this rate" duration and derive both burst sizes from it, rather than picking CBS and EBS independently (as srTCM and trTCM require). This is useful when the desired behaviour is expressed naturally in time ("allow a 2-second burst") rather than in bytes, but it constrains CBS/CIR and EBS/EIR to the **same ratio** — which may not fit every design; nothing in RFC 4115 requires using the linked form.

---

## 8. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | srTCM's E bucket only fills **after** C is completely full | It is not an independent second rate — srTCM is fundamentally single-rate |
| 2 | srTCM's EBS is **not** "CBS plus extra" — it's an independently-sized bucket | Max green+yellow burst is CBS+EBS combined, which may exceed a designer's expectation |
| 3 | trTCM's **yellow packets still decrement Tp** (the peak bucket) | Sustained yellow traffic can push otherwise-conforming green traffic into red |
| 4 | trTCM tests **peak first**; RFC 4115 tests **committed first** | Same input numbers can classify differently — verified directly in §5.4's worked comparison |
| 5 | All three algorithms use **strict conformance** — no token borrowing | A packet needs the full byte count present *now*; nothing is deferred |
| 6 | Bucket size smaller than the largest expected packet means that packet **can never pass** | Not a tuning nicety — a hard structural consequence of strict conformance |
| 7 | RFC 4115 carries an explicit **"not reviewed for these purposes"** IESG disclaimer | Lower formal review status than RFC 2697/2698 — cite accordingly |
| 8 | "Color" (green/yellow/red) is RFC 3290/2697/2698's own vocabulary | Not vendor slang — it's the standards' term for conformance level |

---

## 9. Quick Recap

| Concept | One-line answer |
|---|---|
| What a meter produces | A conformance color: green / yellow / red |
| srTCM's distinguishing factor | Burst **length**, one rate (CIR), two cascaded buckets (C then E) |
| trTCM's distinguishing factor | Two independent rates (CIR, PIR); peak tested first |
| trTCM's known issue | Yellow packets consume peak-bucket (Tp) credit, can starve green traffic |
| RFC 4115's fix | Test committed bucket first; excess bucket is fully independent |
| All three share | Strict-conformance token buckets — no borrowing from future refills |
| Standards status | All three are **Informational**, not Standards Track |
| Where policing sits in the model | Meter → Marker (→ Absolute Dropper for red), per `7-QoS-Policy-Model` |

---

## References

**Informational**
- RFC 2697 — A Single Rate Three Color Marker (srTCM)
- RFC 2698 — A Two Rate Three Color Marker (trTCM)
- RFC 4115 — A Differentiated Service Two-Rate, Three-Color Marker with Efficient Handling of in-Profile Traffic
- RFC 3290 — An Informal Management Model for Diffserv Routers (Meter element, strict/loose conformance — background from `7-QoS-Policy-Model`)
- RFC 2597 — Assured Forwarding PHB Group (color-to-drop-precedence mapping referenced by RFC 2698 §4)
- RFC 2475 — An Architecture for Differentiated Services (traffic-conditioning framework both markers are built for)
