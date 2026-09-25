## QoS Class Design — RFC 4594 Service Classes

> 💡 **TL;DR:** RFC 4594 defines **12 service classes** (2 network-control + 10 user/subscriber) and gives each one a **recommended DSCP, PHB, queuing type and edge-conditioning policy**. Only the **Standard (Default Forwarding)** class is REQUIRED — everything else is OPTIONAL, and RFC 4594 explicitly expects most networks to deploy a **subset**, "starting off with three or four service classes... and adding others as the need arises." The **"4/8/12-class model"** terminology used across the industry is not RFC wording — it's a common-practice framing of how many of the 12 classes a given network actually turns on. This note gives the full 12-class table with verified DSCPs, walks through RFC 4594's own worked deployment examples (which are, in effect, 6-class and 9-class designs), and covers the Wi-Fi mapping problem that RFC 8325 was written to fix.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC (RFC 4594 itself is Informational) · `[Common practice]` engineering practice / industry framing · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 4594 — **Informational** (it is guidance, not a protocol standard — every "SHOULD" in it is a recommendation, not a requirement). RFC 2474, RFC 2597, RFC 3246, RFC 5865 (the PHBs it recommends) — Standards Track. RFC 8325 (Wi-Fi mapping) — Standards Track. RFC 8100 (inter-provider classes) — Informational.

---

## 1. Why "Class Design" Is Its Own Topic

Notes 3–5 covered the pieces: how to classify (note 3), how to mark (note 4), and what each DSCP value *is* (note 5). This note answers a different question: **which classes should a network actually build, and which DSCP goes with which class?** RFC 4594 §1.3 defines a **service class** as a statement of the delay, loss and jitter characteristics a traffic aggregate requires — the class is defined by application need, not by which PHB happens to implement it.

```
 Applications with similar          -->  one Service Class -->  one recommended DSCP
 delay/loss/jitter needs                 (RFC 4594)              (RFC 2474/2597/3246/5865)
```

RFC 4594 §1.3 is explicit that this is guidance, not a mandate: *"There is no intrinsic requirement that particular DSCPs, traffic conditioners, PHBs, and AQM be used for a certain service class, but as a policy and for interoperability it is useful to apply them consistently."*

---

## 2. The Twelve Service Classes

`[Guidance]` RFC 4594 §1.3 states it defines **twelve different service classes, two for network operation/administration and ten for user/subscriber applications/services**. Figure 1 (§2.2) groups the ten user classes into four application categories:

```
                    Application Control  --  Signaling
                                |
   Media-Oriented  --  Telephony, Real-Time Interactive,
                        Multimedia Conferencing, Broadcast Video,
                        Multimedia Streaming
                                |
   Data            --  Low-Latency Data, High-Throughput Data,
                        Low-Priority Data
                                |
   Best Effort     --  Standard
```

Plus the two **Network Control** classes (Network Control, OAM), covered separately in RFC 4594 §3 because they carry the network's own control traffic, not user traffic.

### 2.1 The full table (RFC 4594 Figure 3, exact values)

| Service Class | DSCP Name(s) | DSCP Decimal | PHB | Queuing | AQM | Example applications |
|---|---|:--:|---|:--:|:--:|---|
| Network Control | CS6 | 48 | RFC 2474 CS | Rate | Yes | Network routing (OSPF, BGP, ISIS, RIP) |
| Telephony | EF | 46 | RFC 3246 EF | **Priority** | **No** | VoIP bearer (G.711, G.729) |
| Signaling | CS5 | 40 | RFC 2474 CS | Rate | No | IP telephony signaling (SIP, H.323, H.248) |
| Multimedia Conferencing | AF41, AF42, AF43 | 34, 36, 38 | RFC 2597 AF | Rate | Yes, per DSCP | H.323/V2 video conferencing (rate-adaptive) |
| Real-Time Interactive | CS4 | 32 | RFC 2474 CS | Rate | No | Video conferencing / interactive gaming (non-adaptive) |
| Multimedia Streaming | AF31, AF32, AF33 | 26, 28, 30 | RFC 2597 AF | Rate | Yes, per DSCP | Streaming video/audio on demand |
| Broadcast Video | CS3 | 24 | RFC 2474 CS | Rate | No | Broadcast TV, live events |
| Low-Latency Data | AF21, AF22, AF23 | 18, 20, 22 | RFC 2597 AF | Rate | Yes, per DSCP | Client/server transactions, web-based ordering |
| OAM | CS2 | 16 | RFC 2474 CS | Rate | Yes | OAM&P (SNMP, TFTP, FTP, Telnet, COPS) |
| High-Throughput Data | AF11, AF12, AF13 | 10, 12, 14 | RFC 2597 AF | Rate | Yes, per DSCP | Store-and-forward apps |
| Standard | DF (CS0) | 0 | RFC 2474 DF | Rate | Yes | Undifferentiated applications |
| Low-Priority Data | CS1 | 8 | RFC 3662* | Rate | Yes | Any flow with no bandwidth assurance |

*RFC 4594's Figure 4 cites RFC 3662 for the Low-Priority Data PHB. RFC 3662 has since been **obsoleted by RFC 8622** (note 5 §7), which recommends a dedicated **LE** codepoint (`000001` = 1) instead of CS1 for this purpose, and RFC 8622 explicitly **updates RFC 4594** on this point. **Current guidance is LE, not CS1**, for low-priority data — the CS1 value in RFC 4594's original table is superseded.

> ⚠️ **Gotcha:** Notice the table's **DSCP decimal values are not in priority order**. Reading top-to-bottom by traffic importance (Network Control highest, Low-Priority Data lowest), the DSCP numbers go 48, 46, 40, 34–38, 32, 26–30, 24, 18–22, 16, 10–14, 0, 8. **DSCP value is not a priority ranking** — CS1 (8) is *below* Standard/DF (0) in intended treatment despite being numerically larger. Priority comes from the PHB and queuing configuration, not from comparing DSCP numbers directly.

### 2.2 Queuing type — the one column that most affects behaviour

RFC 4594 §1.4.1 distinguishes only **two** queuing types, and Figure 3's "Queuing" column assigns each class one of them:

- **Priority Queuing** (§1.4.1.1): scheduler drains the highest-priority queue first, always. `[Guidance]` RFC 4594 states this gives a **readily calculated delay** — proportional to the remaining serialization time of whatever is currently on the wire plus whatever is already queued ahead in that same queue. **Telephony is the only class assigned Priority queuing** in the whole table.
- **Rate Queuing** (§1.4.1.2): scheduler gives each queue a share of bandwidth (WFQ/WRR-style). Delay depends on the queue's own occupancy *and* what it competes with. **Every other class** in the table uses Rate queuing.

> 📝 This is a specific, checkable fact, not a general impression: **only one class (Telephony/EF) gets priority treatment**; the rest — including Network Control (CS6) and Real-Time Interactive (CS4) — are rate-queued. A design that puts multiple classes into a single hardware priority queue is not following RFC 4594's guidance as written.

### 2.3 AQM column — why EF says "No"

`[Guidance]` RFC 4594 §4.1 states plainly: *traffic in the Telephony service class does not respond dynamically to packet loss, so AQM SHOULD NOT be applied to EF marked packet flows.* AQM (RED/WRED, covered fully in `11-QoS-Congestion-Avoidance`) works by dropping or ECN-marking packets *to signal congestion to a sender that will slow down* — inelastic EF/CS4/CS5/CS3 traffic won't respond to that signal, so AQM on those queues only adds loss without the intended benefit. The AF classes (which carry elastic traffic) get **"Yes, per DSCP"** — a separate RED threshold per drop-precedence value within the class, exactly as shown for Multimedia Conferencing in §5 below.

---

## 3. Per-Class Detail: Verified from RFC 4594 §4

### 3.1 Telephony (§4.1)

- RFC 4594 states the fundamental service is *"minimum jitter, delay, and packet loss... similar to an ATM CBR service."*
- **SHOULD use EF PHB**, **SHOULD** get guaranteed forwarding resources, **SHOULD** use Priority Queuing.
- Traffic characteristics (RFC 4594's own words): *"Mostly fixed-size packets for VoIP (60, 70, 120 or 200 bytes in size). Packets emitted at constant time intervals."*
- **Edge conditioning:** untrusted-source marking **SHOULD be verified** by MF classification; untrusted flows **SHOULD be policed** (single-rate + burst-size token bucket). Policing is **OPTIONAL** for trusted sources, and **OPTIONAL** across peering points where an SLA and admission control already govern the traffic.
- **Admission control:** typically performed by a *"telephony call server/gatekeeper using signaling (SIP, H.323, H.248, MEGACO, etc.) on access points to the network"* — not by per-flow network signaling.

### 3.2 Signaling (§4.2)

- **SHOULD use CS5**, Class Selector PHB, Rate Queuing.
- Exists specifically to be **distinguished from Low-Latency Data**, even though the two have similar performance needs — RFC 4594's reasoning: Signaling is *"administrative control and management"* traffic tied to a media session, so it is marked differently.
- RFC 4594 defines **"ring clipping"**: the risk that the front of a PSTN ringing tone is cut off because the IP bearer path isn't ready in time — a direct, concrete reason Signaling needs low queuing delay even though it isn't the bearer traffic itself.
- Same edge-conditioning pattern as Telephony: verify/police untrusted sources, optional for trusted sources, policed to the SLA at peering points.

### 3.3 Multimedia Conferencing (§4.3) — the AF4x example, fully worked

- **SHOULD use AF PHB** across **AF41/AF42/AF43**, Rate Queuing.
- This is rate-**adaptive** traffic (distinct from Real-Time Interactive below, which is inelastic): the source reduces its encoding rate when the receiver reports loss.
- **RECOMMENDED marking logic**, quoted directly: *AF41 = up to specified rate "A"; AF42 = in excess of "A" but below rate "B"; AF43 = in excess of "B"*, where A < B — this is exactly the two-rate, three-color marking behaviour covered fully in `8-QoS-Policing` (RFC 2698 trTCM).
- **Drop-precedence ordering is a MUST**, not a SHOULD: *"The probability of loss of AF41 traffic MUST NOT exceed the probability of loss of AF42 traffic, which in turn MUST NOT exceed the probability of loss of AF43."*
- The RED/AQM threshold ordering RFC 4594 specifies for this class:

```
 min-threshold AF43 < max-threshold AF43 <= min-threshold AF42
 min-threshold AF42 < max-threshold AF42 <= min-threshold AF41
 min-threshold AF41 < max-threshold AF41 <= queue memory
```

This nested-threshold pattern (drop the highest-numbered sub-class first, at the lowest queue depth) is the general template RFC 4594 repeats for **every** AF-based class (Multimedia Streaming's AF3x gets the identical structure; High-Throughput Data's AF1x and Low-Latency Data's AF2x follow the same pattern, per RFC 4594 §4.8 and §4.7 respectively).

### 3.4 Real-Time Interactive (§4.4)

- **SHOULD use CS4**, Class Selector PHB, Rate Queuing — **not** AF, despite being "real-time." RFC 4594's own note: this class *"MAY be configured as a second EF PHB that uses relaxed performance parameter, a rate scheduler, and CS4 DSCP value"* — i.e., it is deliberately positioned as EF-like-but-rate-queued, for inelastic traffic that cannot reduce its rate (unlike Multimedia Conferencing) and doesn't get Telephony's strict priority queue.
- Traffic that "cannot change encoding rates or mark packets with different importance" — e.g., non-adaptive video conferencing gear and interactive gaming.

### 3.5 Multimedia Streaming (§4.5)

- **SHOULD use AF31/AF32/AF33**, same nested-threshold AQM pattern as §3.3.
- Key distinguishing trait from Broadcast Video (§3.6): this class assumes **buffering at the source/destination**, so it tolerates more delay/jitter than the inelastic classes — directly consistent with the tolerance table in `1-QoS-Fundamentals` §7.3 (RFC 4594 Figure 2).

### 3.6 Broadcast Video, Low-Latency Data, High-Throughput Data, OAM, Standard, Low-Priority Data

These follow the same structural pattern already demonstrated above (CS-based inelastic classes get Rate Queuing with no AQM; AF-based elastic classes get the nested RED thresholds), summarized in the Figure 3 table (§2.1). Full per-class prose is available in RFC 4594 §4.6–§4.10 for anyone implementing a specific class; the structural logic is fully captured in this note.

---

## 4. Network Control Traffic (RFC 4594 §3)

Distinct from user Signaling (§3.2 above): this is traffic *between routers and network nodes*, not between user endpoints.

| Class | DSCP | Key rule |
|---|:--:|---|
| **Network Control** | **CS6 (48)** | Used for OSPF, BGP, ISIS, RIP, and LSP setup (CR-LDP, RSVP-TE). **"User traffic is not allowed to use this service class."** |
| **OAM** | **CS2 (16)** | Provisioning, performance monitoring, alarms (SNMP, TFTP, FTP, Telnet, COPS) |

**CS7 handling** `[Guidance]` (RFC 4594 §3.1, quoted): *"CS7 DSCP value SHOULD be reserved for future use... CS7 marked packets SHOULD NOT be sent across peering points... Drop or remark CS7 packets at ingress to DiffServ network domain."* This is the exact rule cross-referenced in `3-QoS-Classification-Trust` §6.3.

**CS6 edge conditioning:** at peering points, CS6 **SHOULD be policed** to the SLA rate; **CS6 marked packets from untrusted sources (end-user devices) SHOULD be dropped or remarked at ingress** — this is the "network-control markings are a special case" rule already introduced in note 3.

---

## 5. What "4/8/12-Class Model" Actually Means

`[Common practice]` — this exact three-tier phrase is **not RFC terminology**; RFC 4594 itself only says networks should start with "three or four service classes... and add others as the need arises" (§1.3, repeated in §2.4). The "4/8/12" framing is an industry shorthand (widely used in Cisco design literature and elsewhere) for **how many of the 12 defined classes a given network turns on**. RFC 4594 provides the raw material for this by including **three worked deployment examples** — which are, in effect, real instances of a "6-class" and "9-class" design:

### 5.1 Example 1 — a minimal deployment (RFC 4594 §2.4.1, verified, 6 classes)

A service-provider network needing reliable VoIP, a low-delay data service, and normal Internet service. RFC 4594's exact Figure 5:

| Class | DSCP | PHB | Queuing | AQM |
|---|:--:|---|:--:|:--:|
| Network Control | CS6 | RFC 2474 | Rate | Yes |
| Telephony | EF | RFC 3246 | Priority | No |
| Signaling | CS5 | RFC 2474 | Rate | No |
| Low-Latency Data | AF21/22/23 | RFC 2597 | Rate | Yes, per DSCP |
| OAM | CS2 | RFC 2474 | Rate | Yes |
| Standard (+other) | DF (CS0) | RFC 2474 | Rate | Yes |

This is a **6-class** design — a concrete, RFC-sourced example of what the industry would call a "starter" model. It maps almost exactly onto what many treat as a "4-class" *user-facing* model if Network Control and OAM (infrastructure classes) are set aside and only the four user classes (Telephony, Signaling, Low-Latency Data, Standard) are counted.

### 5.2 Example 2 — adds video and bulk data (§2.4.2, verified, adds 4 more classes = 10 total)

Extends Example 1 with **Real-Time Interactive** (desktop video conferencing), **Broadcast Video** (IPTV), **Multimedia Streaming** (on-demand movies), and **High-Throughput Data** (network storage/backup) — reaching **10 of the 12** classes (all except Multimedia Conferencing and Low-Priority Data). RFC 4594's own Figure 6 lists this full set with DSCPs, PHBs, queuing and AQM exactly as in §2.1's master table.

### 5.3 Example 3 — an enterprise deployment (§2.4.3, verified, 9 classes)

RFC 4594's own words: *"the enterprise's network needs are addressed with the deployment of the following **nine service classes**"* — Network Control, OAM, Standard, Telephony, Signaling, **Multimedia Conferencing** (inter-conference-room video), **Multimedia Streaming** (prerecorded audio/video), **High-Throughput Data** (engineering file transfer), and **Low-Priority Data** (background applications, reduced during peak load). This example specifically **omits** Real-Time Interactive and Broadcast Video — showing that "which classes" is driven entirely by which applications a given network actually needs to differentiate, not by any fixed progression.

### 5.4 What this means for the "4/8/12" framing

| Common label | What it typically means in practice | Closest RFC 4594 example |
|---|---|---|
| "4-class model" | Network Control (or none) + Telephony + one data/business class + Standard | Simplified version of Example 1 |
| "6/8-class model" | Example 1's structure, or Example 1 + a couple of video/data classes | Example 1 (6 classes, verified) / partial Example 2 |
| "12-class model" | All 12 RFC 4594 classes deployed | Full Figure 3 table (§2.1) |

**The number is not the point.** RFC 4594 §2.4's own framing is: identify which applications need differentiated treatment, then deploy exactly the classes that distinguish them — no more. A network that deploys all 12 classes without a distinct traffic type for each one is adding complexity RFC 4594 does not ask for; a network that lumps two genuinely different traffic types into one class loses the differentiation the class structure exists to provide.

---

## 6. Mapping to Wi-Fi — the RFC 8325 Problem (Preview of Note 18)

`[Standard-defined]` RFC 8325 exists because the **default** DSCP→802.11 User Priority mapping (top 3 bits of the DSCP, as covered in `4-QoS-Marking-Headers` §4.2) misroutes several RFC 4594 classes into the wrong Wi-Fi Access Category. Verified directly from RFC 8325 §4.2.1 and §4.2.2:

| RFC 4594 class | DSCP | Default UP (top-3-bits) | Default AC | RFC 8325 recommends | Resulting AC |
|---|:--:|:--:|---|:--:|---|
| Telephony | EF (46) | 5 | AC_VI (Video) — **wrong** | **UP 6** | AC_VO (Voice) — correct |
| (VOICE-ADMIT) | 44 | 5 | AC_VI — wrong | **UP 6** | AC_VO — correct |
| Signaling | CS5 (40) | 5 | AC_VI | **UP 5** (kept) | AC_VI (deliberate — RFC 8325 reasons Signaling deserves better than best-effort but not the Voice queue) |
| OAM | CS2 (16) | 2 | AC_VI | **UP 0** | AC_BE (deliberately demoted) |
| Network Control | CS6 (48) | 6 | AC_VO | 6 (kept — matches default) | AC_VO |

RFC 8325's own stated reasoning for each override is worth quoting precisely, since it shows this is a deliberate design choice, not an arbitrary table:

- **EF and VOICE-ADMIT → UP 6:** quoted directly, *"Traffic marked to DSCP EF will map by default... to UP 5 and, thus, to the Video Access Category (AC_VI) rather than to the Voice Access Category (AC_VO), for which it is intended."*
- **Signaling stays at UP 5 (not UP 6):** RFC 8325 reasons Signaling is *not* control-plane traffic from the network's own perspective (it's user data-plane traffic, even though it's control-plane from the telephony application's perspective), so it does not merit the Network Control treatment (UP 6) — but it does deserve better than best effort, leaving AC_VI as the appropriate landing point.
- **OAM → UP 0 (not the default UP 2):** RFC 8325 reasons that OAM traffic, by default, would also land in AC_VI — the same contention domain as real user video traffic — which contradicts RFC 4594 §3.3's own intent for OAM (a low-priority, delay-tolerant management class). Demoting it to AC_BE (UP 0) avoids OAM traffic competing with real video for the Video Access Category's airtime.

> ⚠️ **Gotcha (ties together notes 4, 5 and 6):** The single root cause behind **every** row in this table is the same fact established in `4-QoS-Marking-Headers` §4.2: **an n-bit field derived by truncating the DSCP's top n bits collapses distinct DSCP values together and loses the fine-grained meaning RFC 4594 assigned them.** RFC 8325 is the standards body's own fix for exactly this problem — proof that it's a real, documented failure mode, not a hypothetical.

Full Access Category definitions, EDCA parameters, and the complete 12-class RFC 8325 table are covered in `18-QoS-Wireless`.

---

## 7. CCIE-Depth Topics

### 7.1 Why Real-Time Interactive uses CS4, not a second EF codepoint

RFC 3246 allocated exactly **one** EF codepoint. RFC 4594 needed a way to express "EF-like, but rate-scheduled and less strict" without minting a new DSCP for it, so it explicitly permits configuring the *behaviour* of CS4 to resemble a relaxed second EF PHB while keeping the *value* as the already-defined CS4 Class Selector codepoint. This is a clean example of RFC 2474's PHB-vs-codepoint separation (`2-QoS-Models` §4.6): the codepoint identifies the class; what mechanism actually implements it is a local, configurable choice.

### 7.2 The RFC 8622 conflict inside RFC 4594's own table

Figure 4 of RFC 4594 (2006) cites **RFC 3662** for Low-Priority Data's PHB. RFC 8622 (2019) **obsoletes RFC 3662** and **formally updates RFC 4594**, replacing the CS1-based recommendation with the dedicated **LE codepoint**. This is a genuine, documented case of guidance changing over time — a design built strictly from RFC 4594's original table without checking for later updates would deploy an obsoleted recommendation. Always check an RFC's "Updated by" field (note 5 flagged this same fact from the LE side).

### 7.3 Why AQM's "per DSCP" thresholds matter more than the class name

The nested RED-threshold pattern in §3.3 is the actual mechanism that makes "three drop precedences" mean anything operationally. Marking traffic AF41/42/43 without configuring the corresponding staggered min/max thresholds (or an equivalent AQM configuration) produces **no differentiated drop behaviour at all** — the marks would sit in the same queue with the same drop treatment, and the AF PHB's defining guarantee (RFC 2597 §2: probability of loss ordered by drop precedence) is not actually met. Full mechanism in `11-QoS-Congestion-Avoidance`.

### 7.4 The "peering point" pattern repeats for every class

Every per-class section in RFC 4594 §4 follows an identical three-tier trust structure: **untrusted end-user sources** (verify + police), **trusted internal sources** (policing optional), and **peering points** (policed to the SLA, or governed by the admission-control mechanism already in place). This is the same DS-boundary trust model from `3-QoS-Classification-Trust` §6, applied consistently across all 12 classes — recognizing the pattern once means the per-class edge-conditioning rules become predictable rather than needing separate memorization for each class.

---

## 8. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | RFC 4594 is **Informational**, not Standards Track | Every recommendation is a SHOULD, not a protocol MUST |
| 2 | Only the **Standard** class is REQUIRED; all 11 others are OPTIONAL | Deploying all 12 is not "more correct" — it's a design choice matching actual traffic diversity |
| 3 | DSCP decimal value is **not** a priority ranking | CS1 (8) is a *lower*-priority class than Standard/DF (0) |
| 4 | Only **Telephony/EF** gets Priority Queuing in RFC 4594's own table | Everything else — even Network Control (CS6) — is Rate-queued |
| 5 | AQM is explicitly **SHOULD NOT** for EF and the CS-based inelastic classes | It only helps elastic traffic that can respond to a drop/mark signal |
| 6 | RFC 4594's Figure 4 cites **RFC 3662 (obsolete)** for Low-Priority Data | RFC 8622's LE codepoint is the current guidance |
| 7 | "4/8/12-class model" is industry shorthand, not RFC terminology | RFC 4594 itself only ever says "start with three or four, add as needed" |
| 8 | Wi-Fi's default DSCP→UP mapping breaks Telephony and OAM specifically | RFC 8325 exists to correct exactly these two classes |
| 9 | AF drop-precedence ordering (AF41 ≤ AF42 ≤ AF43 loss) is a **MUST**, not a SHOULD | It's the one hard requirement inside an otherwise all-guidance document |

---

## 9. Quick Recap

| Concept | One-line answer |
|---|---|
| Total classes defined | 12 (2 network-control + 10 user/subscriber) |
| Required class | Standard (DF/CS0) only |
| Only Priority-queued class | Telephony (EF) |
| Classes with mandatory drop-precedence ordering | All AF-based classes (Multimedia Conferencing, Multimedia Streaming, Low-Latency Data, High-Throughput Data) |
| Network-control-only DSCPs | CS6 (Network Control), CS2 (OAM) — user traffic not permitted |
| CS7 rule | Reserved for future use; SHOULD NOT cross peering points |
| Low-Priority Data current guidance | LE (RFC 8622), not the RFC 4594-original CS1/RFC 3662 |
| "4/8/12-class model" | Industry framing for how many of the 12 classes are deployed — not RFC terminology |
| RFC 4594's own worked examples | 6 classes (Example 1), 10 classes (Example 2), 9 classes (Example 3) |
| Wi-Fi's biggest mapping problem | EF defaults to UP 5 (Video), not UP 6 (Voice) — fixed by RFC 8325 |

---

## References

**Informational (the class-design guidance itself)**
- RFC 4594 — Configuration Guidelines for DiffServ Service Classes (all 12 classes, Figures 1–7, deployment examples)
- RFC 8100 — Diffserv-Interconnection Classes and Practice (inter-provider class recommendations, referenced in `3-QoS-Classification-Trust`)

**Standards Track (the PHBs and fields RFC 4594 recommends)**
- RFC 2474 — Definition of the DS Field (Default, Class Selector PHBs)
- RFC 2597 — Assured Forwarding PHB Group
- RFC 3246 — An Expedited Forwarding PHB
- RFC 5865 — A DSCP for Capacity-Admitted Traffic (VOICE-ADMIT)
- RFC 8622 — A Lower-Effort PHB (LE) — obsoletes RFC 3662, updates RFC 4594's Low-Priority Data guidance
- RFC 2698 — A Two Rate Three Color Marker (trTCM, used for AF4x/AF3x/AF1x conditioning)
- RFC 2697 — A Single Rate Three Color Marker (srTCM, used for AF2x conditioning)
- RFC 8325 — Mapping Diffserv to IEEE 802.11 (Wi-Fi UP/Access Category recommendations)

**Referenced (background covered in other notes)**
- RFC 2309 — Recommendations on Queue Management and Congestion Avoidance (AQM/RED background)
- RFC 3175 — Aggregation of RSVP Reservations (referenced by RFC 4594 §1.5.5)
