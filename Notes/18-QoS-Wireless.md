## Wireless QoS — WMM, EDCA, and the Complete RFC 8325 Mapping

> 💡 **TL;DR:** Wi-Fi's QoS mechanism, standardized as **802.11e** and branded commercially as **WMM (Wi-Fi Multimedia)**, sorts traffic into just **four Access Categories** (AC_VO, AC_VI, AC_BE, AC_BK) rather than DiffServ's much finer-grained DSCP space — so every marking scheme covered in this series must eventually be **compressed down to one of four buckets** before it reaches the air interface. **EDCA** (Enhanced Distributed Channel Access) is the mechanism that then gives each Access Category a *statistically* better chance of winning access to the shared wireless medium, by tuning four parameters — **AIFSN, CWmin, CWmax, TXOP limit** — per category. This note completes what notes 4 and 6 could only preview: **RFC 8325's full, verified 12-class mapping table**, which exists specifically because the default DSCP→UP truncation (`4-QoS-Marking-Headers` §5) misroutes several RFC 4594 classes into the wrong Access Category — most famously, Telephony/EF landing in AC_VI (Video) instead of AC_VO (Voice) by default.

> 🏷️ **Tags:** `[Standard-defined]` IEEE 802.11 standard / RFC Standards Track · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** IEEE 802.11e (folded into 802.11-2016 and later revisions) — ratified IEEE standard, defines EDCA and the four Access Categories. RFC 8325 — Standards Track (Szigeti, Henry, Szigeti, IETF, February 2018).

---

## 1. Why Wi-Fi Needs Its Own QoS Layer

Every mechanism in notes 3–17 assumes a **switched or routed** medium — one sender transmits, and collision is not a concern (or is handled by a completely separate mechanism, as in `10-QoS-Queuing-Scheduling`'s scheduler-per-egress-interface model). Wi-Fi is fundamentally different: it is a **shared, contended medium** — multiple stations compete for the same radio channel, and *access to the channel itself*, not just queuing order once a packet reaches an egress interface, is the resource being managed. This is why Wi-Fi's QoS mechanism (EDCA) operates by tuning **contention parameters** rather than a scheduler in the DRR/WFQ sense (`10-QoS-Queuing-Scheduling` §5–6) — there's no single device deciding "who goes next" the way a router's scheduler does; instead, each station independently participates in a randomized, priority-weighted contention process.

---

## 2. The Four Access Categories

`[Standard-defined]` IEEE 802.11e defines exactly **four** Access Categories, confirmed consistently across research:

| Access Category | Informal name | Intended traffic |
|---|---|---|
| **AC_VO** | Voice | Voice bearer traffic, and (per RFC 8325, §5 below) network control traffic |
| **AC_VI** | Video | Video conferencing, interactive/streaming video |
| **AC_BE** | Best Effort | Ordinary data traffic — the default |
| **AC_BK** | Background | Traffic the user is willing to have delayed |

This is an immediate, important **compression**: RFC 4594 (`6-QoS-Class-Design`) defines **12** service classes; Wi-Fi has only **4** buckets to sort them into. Every wireless QoS mapping decision in this note is fundamentally about **which of the 12 classes lands in which of these 4 categories** — there is no finer granularity available at the 802.11 MAC layer.

---

## 3. User Priority (UP) — the Bridge Between DSCP and Access Category

### 3.1 The exact UP-to-AC mapping

`[Standard-defined]` RFC 8325 Figure 2, reproduced exactly from the RFC text:

```
+-----------------------------------------+
|   User    |   Access   |  Designative   |
| Priority  |  Category  | (informative)  |
|===========+============+================|
|     7     |   AC_VO    |     Voice      |
+-----------+------------+----------------+
|     6     |   AC_VO    |     Voice      |
+-----------+------------+----------------+
|     5     |   AC_VI    |     Video      |
+-----------+------------+----------------+
|     4     |   AC_VI    |     Video      |
+-----------+------------+----------------+
|     3     |   AC_BE    |  Best Effort   |
+-----------+------------+----------------+
|     0     |   AC_BE    |  Best Effort   |
+-----------+------------+----------------+
|     2     |   AC_BK    |   Background   |
+-----------+------------+----------------+
|     1     |   AC_BK    |   Background   |
+-----------------------------------------+
```

> ⚠️ **Gotcha, confirmed directly by the RFC's own table:** notice **UP 3 maps to AC_BE, not AC_BK**, and **UP 0 also maps to AC_BE**, even though 0 < 3 numerically. This is the exact same "numeric order ≠ priority order" trap already established for 802.1Q PCP in `4-QoS-Marking-Headers` §4.1 (where PCP 0/Best-Effort outranks PCP 1/Background) — here the UP table additionally scrambles 3 and 0 into the *same* category, and puts 1 and 2 (not 0 and 1) into Background. Reading this table by "lower number = worse" will get the AC assignment wrong in multiple places.

### 3.2 Where UP comes from — the default derivation

`[Common practice]`, confirmed and consistent with `4-QoS-Marking-Headers` §4.2 and §5's earlier treatment: **User Priority reuses the identical 3-bit value as 802.1Q PCP** — for a wired 802.1Q-tagged frame, the PCP field *is* the UP value directly; for an untagged frame arriving at a wireless AP carrying an IP packet, the **default** derivation takes the **top 3 bits of the DSCP** (the same "n-bit truncation" pattern already established generally in `4-QoS-Marking-Headers` §8.2). This default is exactly what creates the mapping problems RFC 8325 exists to fix — reusing note 4's own worked calculation:

```
 DSCP EF   (46) = 101110   top 3 bits = 101 = 5   -> UP 5 -> AC_VI (default, WRONG per RFC 8325)
 DSCP AF41 (34) = 100010   top 3 bits = 100 = 4   -> UP 4 -> AC_VI (default, happens to be correct)
 DSCP CS1  (8)  = 001000   top 3 bits = 001 = 1   -> UP 1 -> AC_BK (default, happens to be correct)
```

---

## 4. EDCA — How Access Categories Actually Get Different Treatment

### 4.1 The four tunable parameters

`[Standard-defined]`, confirmed directly, 802.11e's EDCA mechanism differentiates the four Access Categories using four parameters, each configured **per AC**:

| Parameter | Meaning |
|---|---|
| **AIFSN** (Arbitration Inter-Frame Space Number) | How many additional time slots a station must sense the medium idle before it may even begin contending — a **smaller** AIFSN means a station can start contending **sooner** |
| **CWmin** (Contention Window minimum) | The smallest random backoff window size |
| **CWmax** (Contention Window maximum) | The largest random backoff window size (grows toward this on repeated collisions) |
| **TXOP limit** (Transmission Opportunity) | The maximum duration a station may transmit once it has won access, without re-contending |

### 4.2 The exact verified parameter table

`[Standard-defined]`, confirmed directly from research (values expressed relative to the PHY-defined base `aCWmin`, consistent with the 802.11e specification's own parameterization):

| AC | AIFSN | CWmin | CWmax | TXOP limit (example PHY) |
|---|:--:|---|---|---:|
| **AC_BK** (Background) | **7** (largest — waits longest before contending) | aCWmin | aCWmax | 0 |
| **AC_BE** (Best Effort) | **3** | aCWmin | aCWmax | 0 |
| **AC_VI** (Video) | **2** | (aCWmin+1)/2 − 1 (smaller window) | aCWmin | 3.008–6.016 ms |
| **AC_VO** (Voice) | **2** | (aCWmin+1)/4 − 1 (smallest window) | (aCWmin+1)/2 − 1 | 1.504–3.008 ms |

**Reading this table mechanically**: AC_VO and AC_VI both use the smallest AIFSN (2) — they're allowed to start contending soonest. Among those two, AC_VO's CWmin is roughly **half** of AC_VI's, meaning AC_VO's random backoff is drawn from a **smaller** range, statistically winning the contention race more often. AC_BK's AIFSN of 7 means it must wait through **five additional idle slots** compared to AC_VO/AC_VI (7 vs. 2) before it may even attempt to contend — a background-class frame arriving at the same instant as a voice-class frame will, on average, lose access to the medium simply because it isn't even allowed to *start* competing as early.

```
 Time axis (idle medium after previous transmission):

  AC_VO/VI:  |--2 slots--| can start contending, then small random backoff
  AC_BE:     |--3 slots-----| starts slightly later
  AC_BK:     |--7 slots-------------| starts much later -- AC_VO/VI/BE
                                         likely already transmitting by now
```

> 📝 This is a fundamentally **probabilistic** mechanism, not a deterministic guarantee — unlike `10-QoS-Queuing-Scheduling`'s Priority Queuing (§4, RFC 4594's exact delay formula), EDCA does not guarantee AC_VO always wins; it only makes AC_VO **statistically far more likely** to win each contention round. This distinction matters for exam-level precision: EDCA is a contention-weighting mechanism, not a scheduler in the strict-priority or DRR/WFQ sense already fully specified in note 10.

### 4.3 TXOP — bounding how long a "win" lasts

`[Standard-defined]` Winning contention doesn't mean unlimited transmission time — the **TXOP limit** caps how long a station may hold the medium after winning, per the table above (roughly 1.5–6 ms depending on AC and PHY). This exists specifically to prevent one long transmission (even from a high-priority AC) from monopolizing the shared medium indefinitely — a direct parallel to the anti-starvation reasoning already established generally in `10-QoS-Queuing-Scheduling` §4.3, applied here to a contention-based medium rather than a scheduler.

---

## 5. RFC 8325's Complete Recommended Mapping — All 12 Classes, Verified

This is the table `4-QoS-Marking-Headers` §5 and `6-QoS-Class-Design` §6 both promised in full. Every row below is taken directly from RFC 8325's own text (Section 4.3's summary table and the per-class recommendation sections), cross-verified against the RFC's stated rationale for each entry.

| RFC 4594 Service Class | DSCP | Recommended UP | Access Category | Matches default (top-3-bit) mapping? |
|---|---|:--:|---|:--:|
| **Network Control** | CS6 | **7** | AC_VO (Voice) | Yes (default is 6, RFC 8325 recommends 7 — see §5.1 below) |
| **Telephony** | EF | **6** | AC_VO (Voice) | **No** — default is 5 (AC_VI) |
| VOICE-ADMIT | 44 (RFC 5865) | **6** | AC_VO (Voice) | **No** — default is 5 (AC_VI) |
| **Signaling** | CS5 | **5** | AC_VI (Video) | Yes |
| **Multimedia Conferencing** | AF41, AF42, AF43 | **4** | AC_VI (Video) | Yes |
| **Real-Time Interactive** | CS4 | **4** | AC_VI (Video) | Yes |
| **Multimedia Streaming** | AF31, AF32, AF33 | **4** | AC_VI (Video) | **No** — default is 3 (AC_BE) |
| **Broadcast Video** | CS3 | **4** | AC_VI (Video) | **No** — default is 3 (AC_BE) |
| **Low-Latency Data** | AF21, AF22, AF23 | **3** | AC_BE (Best Effort) | **No** — default is 2 (AC_BK) |
| **OAM** | CS2 | **0** | AC_BE (Best Effort) | **No** — default is 2 (AC_BK) |
| **High-Throughput Data** | AF11, AF12, AF13 | (maps to AC_BE per the RFC's default-alignment discussion) | AC_BE (Best Effort) | Yes |
| **Standard** | DF (CS0) | **0** | AC_BE (Best Effort) | Yes |
| **Low-Priority Data** | CS1 | **1** | AC_BK (Background) | Yes |

### 5.1 Network Control's specific rationale, quoted directly

`[Standard-defined]` RFC 8325, quoted precisely: *"it is RECOMMENDED to map Network Control Traffic marked CS6 to UP 7... thereby admitting it to the Voice Access Category (AC_VO), albeit with a marking distinguishing it from (data-plane) voice traffic."* This is a deliberate, specific choice: both Network Control (UP 7) and Telephony (UP 6) land in the **same** Access Category (AC_VO), but at **different** UP values *within* that category — recall from `6-QoS-Class-Design` §4 that user traffic is never permitted to use the Network Control class at all, so this distinction preserves that separation even though both share one Access Category at the 802.11 MAC layer.

### 5.2 Telephony and VOICE-ADMIT — the headline problem, confirmed exactly

`[Standard-defined]`, quoted directly, restating and now fully sourcing what notes 4 and 6 previewed: *"Traffic marked to DSCP EF will map by default... to UP 5 and, thus, to the Video Access Category (AC_VI) rather than to the Voice Access Category (AC_VO), for which it is intended. Therefore, a non-default DSCP-to-UP mapping is RECOMMENDED, such that EF DSCP is mapped to UP 6."* VOICE-ADMIT receives the identical treatment and rationale, for the identical reason (`5-QoS-PHB-DSCP-Values` §6 already established VOICE-ADMIT as EF's near-identical sibling).

### 5.3 Multimedia Streaming and Broadcast Video — a second, distinct instance of the same problem

`[Standard-defined]`, quoted directly, RFC 8325 explicitly lists this as a **second** concrete inconsistency, structurally identical to Telephony's: *"Multimedia Streaming (AF3-011xx0) will be mapped to UP 3 (011) and treated in the Best Effort Access Category (AC_BE) rather than the Video Access Category (AC_VI), for which it is intended"* and, immediately following, *"Broadcast Video (CS3-011000) will be mapped to UP 3 (011) and treated in the Best Effort Access Category (AC_BE) rather than the Video Access Category (AC_VI), for which it is intended."* Both are fixed identically: **remap to UP 4**, landing correctly in AC_VI. This is worth highlighting as a *pattern*, not a one-off: RFC 8325 documents **at least three separate instances** (Telephony/VOICE-ADMIT, Multimedia Streaming, Broadcast Video) of the identical root cause (`4-QoS-Marking-Headers` §8.2's n-bit-truncation collapse) producing the identical symptom (traffic landing one Access Category below where RFC 4594 intends it).

### 5.4 OAM — a deliberate demotion, not a bug fix

`[Standard-defined]`, quoted directly, this case is structurally different from §5.2–5.3 — RFC 8325 doesn't just correct OAM to its "intended" category, it **actively demotes** it further: *"By default... packets marked DSCP CS2 will be mapped to UP 2 and serviced with the Background Access Category (AC_BK). Such servicing is a contradiction to the intent expressed in [RFC4594]... it is RECOMMENDED that a non-default mapping be applied to OAM traffic, such that CS2 DSCP is mapped to UP 0, thereby admitting it to the Best Effort Access Category (AC_BE)."* This exact reasoning was already previewed in `6-QoS-Class-Design` §6, and it's worth restating precisely why: RFC 8325 reasons that OAM traffic mapped to AC_VI (which is what the raw top-3-bits default would actually produce for CS2 in some contexts, or AC_BK as stated here) would either compete with genuine video traffic for airtime it doesn't need, or be needlessly delayed behind Background traffic — **AC_BE** is judged the better fit precisely because OAM is neither as urgent as video nor as dismissible as background bulk transfer.

### 5.5 Low-Latency Data — remapped for a specifically wireless-technical reason

`[Standard-defined]`, quoted directly, this case's justification is unlike any of the others — it isn't just about landing in the "intended" category, it's about exploiting a specific behaviour of the WMM hardware queue implementation: *"Mapping Low-Latency Data to UP 3 may allow targeted traffic to receive a superior level of service via per-UP transmit queues servicing the EDCAF hardware for the Best Effort Access Category (AC_BE)... Therefore it is RECOMMENDED to map Low-Latency Data traffic marked AF2x DSCP to UP 3."* This is worth flagging as a distinct category of rationale: most of RFC 8325's recommendations correct a **category-level** mismatch (wrong AC entirely); this one is a **within-category, UP-level** optimization (both UP 2 and UP 3 map to the same AC_BE, but RFC 8325 still recommends UP 3 specifically, for a hardware-queue-ordering reason internal to how some EDCAF implementations service multiple UPs within one AC).

---

## 6. Upstream vs. Downstream — a Distinction Worth Preserving

`[Standard-defined]` RFC 8325 explicitly separates its guidance into **downstream** (AP → client) and **upstream** (client → AP) recommendations — confirmed directly by the document's own structure (Section 4 covers downstream DSCP-to-UP mapping; Section 5 covers upstream mapping and marking within the wireless client's own operating system). This distinction matters because the **AP** is typically a trusted, network-operator-controlled device applying the mapping table in §5 directly, while the **client** (a laptop, phone, or IoT device) is running its own operating system's DSCP-marking and UP-derivation logic — which may or may not follow RFC 8325's recommendations, exactly the same trust-boundary concern already fully established in `3-QoS-Classification-Trust` §6 for wired endpoints. A wireless design that only configures the AP correctly but doesn't address how client operating systems mark and derive UP values from their own outbound traffic has only solved half of RFC 8325's stated problem.

---

## 7. CCIE-Depth Topics

### 7.1 Why UP 3 and UP 0 are deliberately merged into one Access Category

Revisiting §3.1's gotcha: RFC 8325's own Low-Latency Data rationale (§5.5) reveals *why* the standard permits UP 0 and UP 3 to share AC_BE rather than treating this as a wasted opportunity for finer granularity — some EDCAF hardware implementations maintain **per-UP internal ordering within a single Access Category's transmit queue**, so a traffic engineer can still express a *soft* preference between UP 0 and UP 3 traffic even though both ultimately contend for the medium under identical AC_BE parameters (§4.2's table). This is a subtle, `[Implementation-dependent]` behaviour worth knowing about but not relying on as a guaranteed mechanism — RFC 8325 itself frames it as "may allow... a superior level of service," not a certainty.

### 7.2 The AIFSN mechanism as a form of admission bias, not admission control

It's worth being precise that EDCA's AIFSN/CWmin/CWmax weighting (§4.1–4.2) is **not** admission control in the sense already established in `6-QoS-Class-Design` §3.1 (RFC 4594's call-server-based CAC for Telephony) — EDCA never refuses a station's attempt to transmit; it only biases the **statistical odds** of winning each individual contention round. A wireless cell overloaded with AC_VO traffic from many simultaneous voice calls can still degrade badly, exactly the same structural risk already flagged for strict Priority Queuing without accompanying policing (`10-QoS-Queuing-Scheduling` §4.3) — EDCA's per-AC weighting reduces *relative* disadvantage for lower-priority traffic but does nothing to cap the *absolute volume* of high-priority traffic contending at once. This is why real deployments still layer call admission control (per `6-QoS-Class-Design` §3.1's telephony CAC discussion) on top of EDCA, rather than relying on EDCA's contention weighting alone to protect voice quality under heavy load.

### 7.3 Why RFC 8325 documents at least three structurally identical "one category too low" cases

Section 5.3 already noted the pattern; it's worth stating the general principle explicitly, tying back to `4-QoS-Marking-Headers` §8.2's CCIE-depth analysis: **any DSCP whose top-3-bit value is one less than the top-3-bit value of the class "above" it in RFC 4594's intended AC assignment will exhibit this exact failure mode** under the default mapping. EF (top-3-bits=5) sits one below VOICE's target category boundary (6); AF3x and CS3 (top-3-bits=3) sit one below VIDEO's target category boundary (4). Recognizing this as a **general arithmetic consequence** of DiffServ's DSCP numbering colliding with Wi-Fi's coarser 4-category structure — rather than three unrelated, coincidental bugs — is the same kind of pattern-recognition this series has emphasized since `4-QoS-Marking-Headers`' original truncation-collapse analysis.

---

## 8. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | 802.11e defines only **four** Access Categories — every DSCP-based scheme must compress into this coarser structure | The 12 RFC 4594 classes map many-to-one onto 4 buckets, inevitably |
| 2 | UP-to-AC mapping is **not** monotonic by UP number — UP 3 and UP 0 share AC_BE; UP 1 and 2 share AC_BK | Reading the table by "lower number = lower priority" gives wrong answers in multiple places |
| 3 | EDCA is **probabilistic** contention-weighting, not a deterministic scheduler | Unlike Priority Queuing's exact delay bound (note 10), EDCA only shifts statistical odds |
| 4 | RFC 8325 documents **at least three separate instances** of the identical "one category too low" failure (Telephony, Multimedia Streaming, Broadcast Video) | This is a general, predictable arithmetic consequence, not three unrelated coincidences |
| 5 | OAM's fix isn't "restore intended treatment" — it's a **deliberate demotion** to AC_BE, reasoned independently | Different rationale category from the Telephony/Streaming/Broadcast fixes |
| 6 | Low-Latency Data's UP 3 recommendation is a **within-category** hardware-queue optimization, not a category-level fix | Both UP 2 and UP 3 already map to the same AC_BE |
| 7 | RFC 8325's guidance is split into **downstream** (AP-controlled) and **upstream** (client-OS-controlled) recommendations | Configuring only the AP addresses half the problem — client OS marking behaviour is a separate, often less controllable, trust boundary |
| 8 | TXOP bounds how long a contention "win" lasts, even for AC_VO | Prevents one high-priority transmission from monopolizing the shared medium indefinitely |

---

## 9. Quick Recap

| Concept | One-line answer |
|---|---|
| Why Wi-Fi needs its own QoS layer | Shared, contended medium — access to the channel itself is the resource, not just egress-queue ordering |
| Four Access Categories | AC_VO (Voice), AC_VI (Video), AC_BE (Best Effort), AC_BK (Background) |
| EDCA's four parameters | AIFSN, CWmin, CWmax, TXOP limit — tuned per Access Category |
| Default UP derivation | Top 3 bits of the DSCP — the same truncation pattern from `4-QoS-Marking-Headers` |
| RFC 8325's headline fix | EF/VOICE-ADMIT: default UP 5 (AC_VI) → recommended UP 6 (AC_VO) |
| Second documented instance | Multimedia Streaming/Broadcast Video: default UP 3 (AC_BE) → recommended UP 4 (AC_VI) |
| OAM's fix | Deliberate demotion: default AC_BK → recommended AC_BE (not AC_VI) |
| Network Control | UP 7, distinct from Telephony's UP 6, both within AC_VO |
| Low-Latency Data's UP 3 | A within-AC_BE hardware-queue optimization, not a category correction |
| Upstream vs. downstream | AP-side (network-controlled) mapping vs. client-OS-side (less controllable) marking — two separate trust boundaries |

---

## References

**Standards Track**
- RFC 8325 — Mapping Diffserv to IEEE 802.11 (Szigeti, Henry, Szigeti, February 2018)
- IEEE 802.11e (folded into IEEE 802.11-2016 and later) — EDCA, the four Access Categories, AIFSN/CWmin/CWmax/TXOP parameters

**Referenced (background and what feeds into this note)**
- `4-QoS-Marking-Headers` (802.1Q PCP; the default top-3-bit DSCP→UP truncation and its general collapse pattern)
- `6-QoS-Class-Design` (RFC 4594's 12 service classes; the original preview of this note's EF→UP6 and OAM→UP0 cases)
- `3-QoS-Classification-Trust` (the trust-boundary concept, applied here to upstream client-OS marking vs. downstream AP configuration)
- `10-QoS-Queuing-Scheduling` (Priority Queuing's deterministic delay bound, contrasted against EDCA's probabilistic contention-weighting; anti-starvation reasoning)
- `5-QoS-PHB-DSCP-Values` (VOICE-ADMIT as EF's near-identical sibling, receiving identical RFC 8325 treatment)
