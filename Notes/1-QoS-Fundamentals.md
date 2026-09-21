## QoS Fundamentals

> 💡 **TL;DR:** QoS is the set of mechanisms a network uses to decide **which packets are delayed, dropped or sent first when a link is congested**. It does not create bandwidth. Service quality is described by four measurable parameters — **bandwidth, delay, jitter, loss**. Real-time traffic (voice, interactive video) is *inelastic* and needs bounded delay/jitter/loss; TCP data is *elastic* and adapts. ITU-T G.114 gives a planning target of **≤ 150 ms one-way, mouth-to-ear** for interactive voice; ITU-T Y.1541 gives numeric IP performance classes; RFC 4594 maps traffic types to service classes. The QoS toolset is: **classify → mark → police/shape → queue/schedule → drop or mark early (AQM/ECN)**.

> 🏷️ **How claims are tagged in this series**
> - `[Standard-defined]` — defined in a standards document (RFC Standards Track, ITU-T Recommendation, IEEE).
> - `[Guidance]` — recommendation in an Informational RFC or similar; not a protocol requirement.
> - `[Common practice]` — widely used engineering practice or vendor design guidance.
> - `[Implementation-dependent]` — varies by platform; check your device documentation.

---

## 1. What QoS Is (and Is Not)

**Working definition** `[Common practice]`: QoS is the ability of a network to treat different traffic differently, so that each kind of traffic gets the delay, jitter, loss and bandwidth it needs.

**Service class** `[Guidance]` (RFC 4594): a set of traffic that needs specific delay, loss and jitter characteristics from the network. QoS design starts by grouping applications into a small number of such classes, not by treating every application individually.

**Best effort** `[Guidance]` (RFC 4594 §1.5.1): the network accepts packets and makes no promises. Packets can be lost, reordered, duplicated or delayed. Best effort is the default behaviour; QoS is any deliberate departure from it.

### QoS matters only where there is congestion

```
 PC-1 ------\
 PC-2 --------> [ Switch ] ==== 1 Gbps ====> [ Router ] ---- 50 Mbps ----> WAN / Internet
 IP Phone --/                                    |
                                    Up to 1 Gbps can arrive,
                                    only 50 Mbps can leave
                                    --> a queue builds at the
                                        router's WAN egress
```

- If the outgoing link is never full, queues stay empty and every class gets the same excellent service. QoS mechanisms do nothing useful.
- When the arrival rate exceeds the departure rate, a queue forms. Packets wait (**delay**), wait different amounts (**jitter**), and when the queue is full some are discarded (**loss**). QoS decides *who* waits and *who* is dropped.
- RFC 4594 §1.2 `[Guidance]` says its recommendations are meant for any link that is itself congested, and observes that congestion is more typical in access networks and oversubscribed points than in lightly loaded backbones.

> 📝 **QoS is a zero-sum tool** `[Common practice]`: giving one class better treatment necessarily gives another class worse treatment. QoS cannot make a permanently overloaded link healthy — that needs more capacity or admission control.

---

## 2. The Four QoS Parameters

| Parameter | Meaning | Main causes | Standard metric |
|---|---|---|---|
| **Bandwidth** | Rate the network can carry (bits/s) | Link speed, shaping, policing, oversubscription | Link rate; throughput measured by test |
| **Delay (latency)** | Time for a packet to travel source → destination | Serialization, propagation, processing, queuing | One-way delay: RFC 7679; round-trip: RFC 2681; ITU-T Y.1540 IPTD |
| **Jitter** | Variation in delay between packets | Mostly variable queuing delay | IPDV: RFC 3393; RTP jitter estimator: RFC 3550 |
| **Loss** | Packets that do not arrive (or arrive too late to be useful) | Queue overflow, policing, link errors, late discard | One-way loss: RFC 7680; ITU-T Y.1540 IPLR |

Other parameters exist — packet error ratio (IPER) and reordering (IPRR) appear in ITU-T Y.1540/Y.1541 — but the four above drive almost all QoS design.

> ⚠️ **Gotcha:** "Bandwidth" (link rate), "throughput" (what actually crosses the link) and "goodput" (useful application data after headers and retransmissions) are three different numbers. QoS policies are written in link/L3 rates; users complain about goodput.

---

## 3. Delay

### 3.1 Components of one-way delay

```
 Endpoint ---> Switch ---> Router ===WAN===> Router ---> Switch ---> Endpoint
    |             |           |        |         |
    |             |           |        |         +-- processing / forwarding delay (per device)
    |             |           |        +-- propagation delay (distance)
    |             |           +-- queuing delay (VARIABLE) + serialization delay (size / rate)
    |             +-- processing / forwarding delay
    +-- codec + packetization delay (for voice/video)
```

| Component | Formula / nature | Fixed or variable | Can QoS change it? |
|---|---|---|---|
| **Serialization** (transmission) | packet size (bits) ÷ link rate (bits/s); paid at **every** store-and-forward hop | Fixed for a given size and rate | Only indirectly (fragmentation/LFI, faster link) |
| **Propagation** | distance ÷ signal speed | Fixed | No |
| **Processing / forwarding** | lookup, switching, ASIC/CPU time | Mostly fixed `[Implementation-dependent]` | No |
| **Queuing** | time spent waiting behind other packets | **Variable** | **Yes — this is what QoS controls** |
| **Codec / packetization / playout** (end system) | e.g. 20 ms of voice per packet, de-jitter buffer | Mostly fixed / configurable | No (but it eats the budget) |

### 3.2 Serialization delay — worked table

Serialization delay = bits ÷ rate. (IP packet size shown; Layer-2 overhead adds a little more on the wire.)

| Link rate | 1500 B packet (12,000 bits) | 200 B voice packet (1,600 bits) |
|---|---:|---:|
| 64 kbps | 187.5 ms | 25 ms |
| 128 kbps | 93.75 ms | 12.5 ms |
| 1.544 Mbps (T1) | 7.77 ms | 1.04 ms |
| 10 Mbps | 1.2 ms | 0.16 ms |
| 100 Mbps | 120 µs | 16 µs |
| 1 Gbps | 12 µs | 1.6 µs |
| 10 Gbps | 1.2 µs | 0.16 µs |

Why this matters: on a 128 kbps link, a voice packet that arrives just after a 1500-byte data packet has started transmitting must wait up to ~94 ms. That single event can consume most of a voice delay/jitter budget. This is the reason link fragmentation and interleaving (LFI) exists (covered in `15-QoS-Link-Efficiency`). ITU-T Y.1541 §1.2 makes the same point: its objectives are primarily for access links at T1/E1 and higher, because IPTD includes serialization time and sub-T1 rates can produce serialization times over 100 ms for 1500-octet packets. Its Appendix IV works an example on a T1 access link where two 1500-byte packets served ahead of a voice packet add about 15.6 ms (consistent with 2 × 7.77 ms from the table above).

> 📝 RFC 4594 §1.4.1.1 `[Guidance]` makes the same point for priority queues: the delay a packet in the top-priority queue sees is roughly the remaining serialization time of the packet already on the wire plus the data queued ahead of it in the same queue.

### 3.3 Propagation delay

Light in optical fibre travels at about two-thirds of *c* — roughly **2 × 10⁸ m/s**, i.e. about **5 µs per km**, or **~1 ms per 200 km** one-way `[Physics — arithmetic]`. Published engineering values differ, so state which one you use: ITU-T Y.1541 (Appendix III, using G.114's figure for optical transport) budgets **5 µs per km of route length** and takes route length as **1.25 × the air-route distance** beyond 1,200 km; the Cisco Press VoIP delay figure (2004) uses **6.3 µs/km**. A 10,000 km path costs at least ~50 ms one-way by the physics figure; Y.1541's rule gives 1.25 × 10,000 km × 5 µs = 62.5 ms. Real paths are longer than great-circle distance. No QoS feature can reduce this.

### 3.4 The delay budget — ITU-T G.114

`[Standard-defined]` ITU-T G.114 (One-way transmission time) recommends, for connections with echo adequately controlled:

| One-way transmission time | G.114 view |
|---|---|
| 0 – 150 ms | Acceptable for most user applications; interactivity is essentially transparent |
| 150 – 400 ms | Acceptable provided administrations are aware of the impact on applications |
| > 400 ms | Not recommended for general network planning (with rare, recognised exceptions such as unavoidable double satellite hops) |

> ⚠️ **Gotcha — the 150 ms is end-to-end, "mouth-to-ear", not network-only.** For speech it includes codec and packetization delay, the de-jitter buffer, and decoding, *plus* the network's serialization + propagation + queuing. G.114 also notes that highly interactive tasks can be affected by delays below 150 ms, and that G.107 (E-model) should be used to combine delay with other impairments.

**Worked examples from ITU-T Y.1541 Appendix VII** `[Standard-defined — informative appendix]` (G.711, 20 ms packets; every number below is Y.1541's, not mine):

| Item | Typical endpoint | Low-delay endpoint |
|---|---:|---:|
| Packet formation (two × frame size) | 40 ms | 20 ms (10 ms frames) |
| De-jitter buffer — **average** delay | 30 ms (centre of a 60 ms buffer) | 25 ms (centre of a 50 ms buffer) |
| Packet-loss concealment | 10 ms | 0 ms ("repeat previous") |
| Other equipment | — | 5 ms |
| **Endpoint total** | **80 ms** | **50 ms** |
| + Class 0 network delay (mean) | 100 ms | 100 ms |
| **Mouth-to-ear** | **180 ms** | **150 ms** |

What the example teaches:
- A **typical endpoint plus a class-0 network already exceeds 150 ms** (180 ms). Meeting the 150 ms target needs a low-delay endpoint (50 ms) or a shorter network delay.
- A de-jitter buffer adds its **average** occupancy time to mouth-to-ear delay, not its peak size: packets that arrive with minimum delay wait longest in the buffer, packets that arrive with the maximum accommodated delay wait least.
- Y.1541's class 0 example path (Appendix III) reaches ~100 ms mean delay over about **4,070 km**; its class 1 example (27,500 km route, three network sections) gives 233 ms of network delay, i.e. **313 ms** with the typical endpoint.

### 3.5 IP-level delay objectives — ITU-T Y.1541

`[Standard-defined]` ITU-T Y.1541 (12/2011, edition 3.0) defines network QoS classes with numeric objectives for the IP performance parameters of ITU-T Y.1540 (IPTD, IPDV, IPLR, IPER, IPRR). Compliance with an ITU-T Recommendation is voluntary; the classes are intended as the basis for agreements between users and providers and between providers.

**Table 1 — stable classes**

| Class | IPTD (upper bound on the **mean**) | IPDV | IPLR | IPER | Guidance (Y.1541 Table 2) |
|:--:|:--:|:--:|:--:|:--:|---|
| 0 | 100 ms | 50 ms | 1×10⁻³ | 1×10⁻⁴ | Real-time, jitter sensitive, high interaction (VoIP, video conferencing); separate queue with preferential servicing |
| 1 | 400 ms | 50 ms | 1×10⁻³ | 1×10⁻⁴ | Real-time, jitter sensitive, interactive (VoIP, video conferencing) |
| 2 | 100 ms | U | 1×10⁻³ | 1×10⁻⁴ | Transaction data, highly interactive (signalling); separate queue, drop priority |
| 3 | 400 ms | U | 1×10⁻³ | 1×10⁻⁴ | Transaction data, interactive |
| 4 | 1 s | U | 1×10⁻³ | 1×10⁻⁴ | Low loss only (short transactions, bulk data, video streaming); long queue, drop priority |
| 5 | U | U | U | U | Traditional applications of default IP networks; separate queue, lowest priority |

**Table 3 — provisional classes** (need not be met by networks until revised from operational experience)

| Class | IPTD (mean) | IPDV (1−10⁻⁵ quantile − min) | IPLR | IPER | IPRR |
|:--:|:--:|:--:|:--:|:--:|:--:|
| 6 | 100 ms | 50 ms | 1×10⁻⁵ | 1×10⁻⁶ | 1×10⁻⁶ |
| 7 | 400 ms | 50 ms | 1×10⁻⁵ | 1×10⁻⁶ | 1×10⁻⁶ |

Classes 6 and 7 target high-bit-rate applications with stricter loss/error needs (e.g., digital television transport); Y.1541 states that even they need FEC/interleaving or other loss mitigation for broadcast-quality video.

**How to read the objectives**
- **IPDV** (classes 0 and 1) = upper bound on the **1 − 10⁻³ quantile (99.9th percentile) of IPTD minus the minimum IPTD**. For planning, the mean-IPTD bound may be taken as an upper bound on the minimum IPTD, so the 99.9th-percentile delay ≈ mean bound + IPDV (class 0: 100 + 50 = **150 ms**).
- **IPTD includes packet insertion (serialization) time**; Y.1541 suggests evaluating with a maximum packet information field of 1500 bytes. IPTD objectives of classes 0 and 2 will not always be achievable over very long paths — hence classes 1 and 3.
- **Evaluation interval:** 1 minute suggested (and recorded with the result); any minute observed should meet the objectives.
- **"U"** = unspecified/unbounded: ITU-T sets no objective, and performance on that parameter may at times be arbitrarily poor (general expectation: mean IPTD no greater than 1 s).
- **Access rates:** the objectives apply primarily when access links are **T1/E1 or faster**; meeting the IPDV objective effectively requires QoS mechanisms on the access device.
- **Loss basis:** for classes 0 and 1 the 10⁻³ IPLR is partly based on studies showing that high-quality voice codecs are essentially unaffected at that ratio.
- **Scope:** public IP networks, **UNI-to-UNI** — not per-device settings, and "end-to-end" here does **not** mean mouth-to-ear.
- **Composition (§8.2):** mean IPTD **adds** across network sections; loss combines as `IPLR = 1 − ∏(1 − IPLRᵢ)`; IPDV is sub-additive and hard to compose accurately.
- **DiffServ mapping (Appendix VI, informative):** Default ↔ class 5; AF ↔ classes 2, 3, 4; EF ↔ classes 0 and 1.

---

## 4. Jitter

### 4.1 Definitions — three different things called "jitter"

| Definition | Source | What it is |
|---|---|---|
| **IPDV** (IP packet delay variation) | RFC 3393 `[Standard-defined]` | Difference in **one-way delay** between two selected packets (e.g., consecutive packets). Valid with or without synchronized clocks; clock drift affects precision. |
| **RTP interarrival jitter** | RFC 3550 §6.4.1 / A.8 `[Standard-defined]` | A running **mean deviation (smoothed absolute value)** kept by the receiver over packets in **order of arrival** and sampled into RTCP reports: `J = J + (\|D\| − J) / 16`, where `D(i,j) = (Rj − Ri) − (Sj − Si)`. The 1/16 gain gives good noise reduction with a reasonable rate of convergence. Reported in RTP timestamp units. |
| **Y.1541 IPDV objective** | ITU-T Y.1541 `[Standard-defined]` | The **1 − 10⁻³ quantile (99.9th percentile) of IPTD minus the minimum IPTD** over the evaluation interval (1 − 10⁻⁵ quantile for provisional classes 6–7). A statistical bound, not a per-packet difference. |

> ⚠️ **Gotcha:** These are **not interchangeable**. A tool reporting "jitter = 8 ms" might mean peak-to-min delay, a standard deviation, an RFC 3550 estimate, or something else. Vendors do not always state which. Compare like with like.
>
> 📝 RFC 3550 also describes the RTCP reception-report field as "an estimate of the statistical variance" of interarrival time, yet defines J as a smoothed **mean absolute deviation**. Treat J as a mean deviation, not a variance.

### 4.2 Worked example — one stream, three "jitter" numbers

Voice packets sent every 20 ms. Transit time = arrival − send time.

```
 Sent (ms):      0     20     40     60     80    100
 Arrived (ms):  30     55     70     98    110    130
```

| # | Sent | Arrived | Transit (R−S) | \|D\| vs previous | RFC 3550 J (running) |
|--:|--:|--:|--:|--:|--:|
| 1 | 0 | 30 | 30 | — | 0 |
| 2 | 20 | 55 | 35 | 5 | 0.31 |
| 3 | 40 | 70 | 30 | 5 | 0.61 |
| 4 | 60 | 98 | 38 | 8 | 1.07 |
| 5 | 80 | 110 | 30 | 8 | 1.50 |
| 6 | 100 | 130 | 30 | 0 | 1.41 |

- **IPDV between consecutive packets (RFC 3393):** +5, −5, +8, −8, 0 ms (signed).
- **Peak-to-minimum transit:** 38 − 30 = **8 ms** — this is what a de-jitter buffer must absorb.
- **RFC 3550 J after 6 packets:** **≈ 1.4 ms** — much smaller than the 8 ms swing, because the estimator's gain is 1/16 and it needs many packets to converge.

### 4.3 What causes jitter

- **Variable queuing delay** at congested egress queues — the dominant cause `[Common practice]`.
- Serialization of a large packet ahead of a small one on a slow link (see §3.2).
- Path changes (routing, load balancing over unequal paths), and link-layer retransmission (e.g., wireless).
- Software forwarding or CPU load on a device `[Implementation-dependent]`.

### 4.4 De-jitter (playout) buffer

A receiver holds packets briefly and plays them at a steady rate. This converts jitter into a fixed extra delay.

```
 Network output (uneven):   |  |    |  | |     |  |
                              \ \    \  \ \     \  \
                           +------ de-jitter buffer ------+
                              / /    /  / /     /  /
 Playout (steady):          |   |   |   |   |   |   |
```

- Buffer **too small** → late packets are discarded (effectively loss).
- Buffer **too large** → adds delay (eats the G.114 budget).
- Cisco's VoIP QoS documentation states that jitter buffers add to end-to-end delay and are usually effective only for delay variations below ~100 ms, so jitter must be minimised in the network `[Common practice — vendor documentation]`.
- RFC 4594 §4.6 notes that broadcast-video receivers typically use a de-jitter buffer of about 2–8 video frames (roughly 66 ms to several hundred ms), which is why that class tolerates more delay than interactive classes `[Guidance]`.

---

## 5. Loss

### 5.1 Where loss comes from

| Cause | Notes |
|---|---|
| **Queue overflow** (tail drop, or early drop by AQM) | Most common in enterprise/WAN congestion |
| **Policing** | Traffic above a contracted rate is dropped or re-marked (see `8-QoS-Policing`) |
| **Link errors** | CRC/FCS errors, bad optics/cables — unrelated to congestion, QoS cannot fix |
| **Late discard** | A packet that arrives after its playout time is useless; ITU-T Y.1540 counts congestion discards and delay-variation discards inside IPLR |
| **Device faults / overload** | Control-plane or ASIC drops `[Implementation-dependent]` |

`[Standard-defined]` RFC 7680 (STD 82, January 2016) defines the one-way loss metric and **obsoletes RFC 2680**; RFC 7679 is the current one-way delay metric (it obsoletes RFC 2679). For measurement, a packet that does not arrive within a chosen waiting-time threshold is counted as lost — Y.1541 uses the same idea (Tmax = the delay beyond which a packet is declared lost).

### 5.2 Loss tolerance differs by traffic type

- **Inelastic real-time (voice, live video):** lost packets are gone — no useful retransmission in time. Codecs use concealment up to a limit. Loss also tends to hurt more when it is **bursty** than when it is spread out `[Common practice]`.
- **Elastic TCP traffic:** loss is *the congestion signal*. TCP recovers by retransmitting and slowing down, so occasional loss is normal — but sustained loss collapses throughput (see §9).
- Y.1541 sets the IPLR objective at **10⁻³ (0.1 %) for classes 0–4** (for classes 0 and 1 it cites studies showing high-quality voice codecs are essentially unaffected at 10⁻³) and at **10⁻⁵ for the provisional classes 6–7**.
- `[Common practice — Cisco Press 2004]` G.711 packet-loss concealment (Appendix I) can mask roughly 20 ms of lost samples; with 20 ms packets, losing **two or more consecutive packets** is noticeable. With random loss, 1 % gives an unconcealable loss about every 3 minutes and 0.25 % about every 53 minutes (check: p² × 50 packets/s ≈ 1 event per 200 s and per 3,200 s).

> ⚠️ **Gotcha:** The same 1 % loss can be tolerable for a voice call with concealment yet devastating to a long-distance TCP transfer, and vice-versa a "fine for TCP" 0.1 % loss on a video stream can be visible. Judge loss against the traffic type, not in the abstract.

---

## 6. Bandwidth

- **Bottlenecks** appear wherever capacity drops or traffic converges: LAN → WAN speed mismatch (as in the diagram in §1), many-to-one aggregation, server uplinks, oversubscribed uplinks.
- **Utilisation averages hide bursts.** A link that averages 30 % over 5 minutes can be at 100 % for many milliseconds at a time; those micro-bursts fill queues and create loss/jitter that a 5-minute counter never shows `[Common practice]`.
- **Bandwidth-delay product (BDP)** = link rate × round-trip time. It is the amount of data "in flight" needed to keep a pipe full. Example: 1 Gbps × 20 ms = 20 Mbit = **2.5 MB**. BDP matters later for TCP window sizing and buffer sizing.

### Voice bandwidth arithmetic

Header overhead per voice packet: IPv4 (20 B) + UDP (8 B) + RTP (12 B) = **40 B** (IPv6 header is 40 B, giving 60 B). Ethernet adds 14 B header + 4 B FCS = 18 B (preamble/inter-frame gap and any 802.1Q tag excluded).

| Codec (20 ms packets, 50 pkt/s) | Payload | IP packet | Rate at L3 | Rate at Ethernet L2 |
|---|--:|--:|--:|--:|
| G.711 (64 kbps) | 160 B | 200 B | 80 kbps | 87.2 kbps |
| G.729 (8 kbps) | 20 B | 60 B | 24 kbps | 31.2 kbps |

RFC 4594 §4.1 `[Guidance]` notes that VoIP packets are mostly fixed-size (60, 70, 120 or 200 bytes) and emitted at constant intervals, which matches this arithmetic. Note the overhead: for G.729, headers are twice the payload.

> 📝 **Overhead conventions differ.** Cisco Press (2004, Table 2-2) quotes **93 kbps (G.711) and 37 kbps (G.729A)** at 50 pps for 802.1Q Ethernet because it counts **32 bytes** of Layer 2 overhead including preamble. The 87.2 / 31.2 kbps figures above count 18 bytes (header + FCS only). Same traffic, different conventions; the Layer 3 figures (80 / 24 kbps) agree with Cisco's Table 2-1. Always state which overhead a provisioning number includes.

---

## 7. Traffic Types

### 7.1 Elastic vs inelastic vs rate-adaptive

`[Guidance]` (terms from RFC 1633 §3.1, used by RFC 4594):

| Type | Behaviour | Example |
|---|---|---|
| **Elastic** | Sender adjusts its rate in response to available capacity, loss or delay | TCP transfers, web, email |
| **Inelastic (real-time)** | Sends at the rate the application produces, regardless of network capacity | G.711 voice, fixed-rate video |
| **Rate-adaptive** | Real-time, but reduces its encoding rate when it detects loss | Some video conferencing (RFC 4594 cites H.323/V2-style adaptation) |

Inelastic traffic has the potential to congest a network if unconstrained — which is why RFC 4594 recommends policing it and using admission control for calls.

### 7.2 User-perception categories — ITU-T G.1010 (via RFC 4594)

| Category | Typical case | Sensitivity |
|---|---|---|
| **Interactive** | Human-to-human conversation, or server-to-server needing very low delay/loss | Most sensitive to delay, loss, jitter |
| **Responsive** | Human waiting on a server | Less jitter-sensitive; tolerates more delay |
| **Timely** | Server/human, longer delay tolerance | Much longer delay tolerance |
| **Non-critical** | Machine-to-machine, delay acceptable | Least sensitive |

### 7.3 Tolerances by service class — RFC 4594 Figure 2

`[Guidance]` "Very Low" in the loss/delay column means the class tolerates very little loss/delay — the network must keep it very small. "Tolerant" in the jitter column means the receiver buffers data so moderate variation does not matter (typical of TCP-based applications).

| Service class | Typical traffic | Loss | Delay | Jitter |
|---|---|---|---|---|
| Telephony | Fixed small packets, constant rate, inelastic | Very low | Very low | Very low |
| Real-time interactive | RTP/UDP, inelastic, variable rate | Low | Very low | Low |
| Multimedia conferencing | Constant interval, rate-adaptive | Low–Medium | Very low | Low |
| Broadcast video | Constant/variable rate, inelastic, non-bursty | Very low | Medium | Low |
| Multimedia streaming | Buffered, elastic, variable rate | Low–Medium | Medium | Tolerant |
| Signaling | Short-lived, somewhat bursty | Low | Low | Tolerant |
| Network control | Small messages; can burst (e.g., BGP) | Low | Low | Tolerant |
| Low-latency data | Bursty, short-lived elastic flows | Low | Low–Medium | Tolerant |
| OAM | Variable, elastic and inelastic | Low | Medium | Tolerant |
| High-throughput data | Bursty, long-lived elastic flows | Low | Medium–High | Tolerant |
| Low-priority data | Non-real-time, elastic | High | High | Tolerant |
| Standard (default) | "A bit of everything" | Not specified | Not specified | Not specified |

Class names and DSCP recommendations are covered in `5-QoS-PHB-DSCP-Values` and `6-QoS-Class-Design`.

### 7.4 Vendor design guidance for voice/video

`[Common practice — vendor guidance]` Cisco Press, *End-to-End QoS Network Design* (2004), Chapter 2, gives these design targets:

| Traffic | DSCP | Loss | One-way latency | Jitter |
|---|---|---|---|---|
| Voice (bearer) | EF (46) | ≤ 1 % | ≤ 150 ms, mouth-to-ear | average one-way jitter targeted < 30 ms |
| Interactive video | AF41 | ≤ 1 % | ≤ 150 ms | ≤ 30 ms |
| Streaming video | CS4 | ≤ 5 % | ≤ 4–5 s (depends on application buffering) | no significant requirement |

Points made in the same chapter:
- The 150 ms figure is taken from ITU-T G.114; Cisco's lab testing found a negligible MOS difference at 200 ms budgets, so the boundary can be extended to 200 ms when 150 ms cannot be met.
- The 30 ms jitter target comes from lab testing showing that voice quality degrades significantly when jitter consistently exceeds 30 ms.
- Interactive video carries an embedded G.711 voice call, so it inherits voice's loss, delay and jitter requirements.
- Voice needs 21–320 kbps of priority bandwidth per call, depending on codec, sampling rate and Layer 2 overhead.

Treat these as **design targets**, not protocol requirements. The 30 ms and 5 % figures are Cisco's own; Y.1541 classes 0/1 express similar needs differently (IPDV ≤ 50 ms as a 99.9th-percentile-minus-minimum; IPLR ≤ 10⁻³). Different definitions give different numbers — do not mix them in one SLA.


---

## 8. The QoS Toolset and Where It Applies

```
 Endpoint ---> Access Switch ---> Router ---> WAN
   marks?      classify / mark    classify / mark        queue / schedule / shape /
               (trust boundary)   police (ingress)       drop early (egress)
```

| Stage | Purpose | Covered in |
|---|---|---|
| **Classification** | Identify which traffic belongs to which class | `3-QoS-Classification-Trust` |
| **Marking** | Write the class into the packet/frame header (DSCP, CoS/PCP, MPLS TC) | `4-QoS-Marking-Headers`, `5-QoS-PHB-DSCP-Values` |
| **Policing** | Measure against a rate; drop or re-mark excess | `8-QoS-Policing` |
| **Shaping** | Buffer excess to smooth traffic to a rate | `9-QoS-Shaping` |
| **Queuing / scheduling** | Decide which queue's packet is sent next | `10-QoS-Queuing-Scheduling` |
| **Congestion avoidance (AQM/ECN)** | Drop or mark early to slow senders before queues fill | `11-QoS-Congestion-Avoidance`, `12-QoS-ECN` |
| **Link efficiency** | Fragmentation/interleaving, header compression | `15-QoS-Link-Efficiency` |

Key placement principles:
- Most **queuing effects** occur at **egress** interfaces, where the arrival rate can exceed the departure rate. `[Common practice]`
- **Classification, marking and policing** typically happen as close to the traffic source as is trustworthy (the *trust boundary*) so downstream devices can act on the marking cheaply. `[Common practice]`
- DiffServ behaviour is defined **per hop** (Per-Hop Behavior, RFC 2474/2475): the end-to-end result is only as good as the weakest congested hop, and every hop must treat the same marking consistently.
- QoS is **directional**: upstream and downstream traffic are handled independently, and the two directions may need different policies.

---

## 9. CCIE-Depth Topics

### 9.1 Buffer depth is delay

Queuing delay at a full queue = queued bytes × 8 ÷ link rate.

| Buffered data | at 1 Gbps | at 100 Mbps | at 10 Mbps |
|---|---:|---:|---:|
| 1 MB | 8 ms | 80 ms | 800 ms |

A buffer that is "just big enough" at one speed becomes hundreds of milliseconds of latency at a lower speed. This is the origin of **bufferbloat** and the motivation for AQM (`11-QoS-Congestion-Avoidance`).

### 9.2 TCP throughput vs loss and RTT (Mathis model)

`[Common practice — academic model, not an RFC]` Mathis, Semke, Mahdavi & Ott (1997) give an upper bound for a single long-lived TCP flow in congestion avoidance:

```
 Rate  <=  (MSS / RTT) x (C / sqrt(p))       C ~ a constant of order 1;  p = packet loss probability
```

Order-of-magnitude example (C ≈ 1, MSS = 1460 B = 11,680 bits, RTT = 50 ms):

| Loss p | Approx. max rate |
|---|---:|
| 0.01 % (10⁻⁴) | ≈ 23 Mbps |
| 0.1 % (10⁻³) | ≈ 7.4 Mbps |
| 1 % (10⁻²) | ≈ 2.3 Mbps |

Two takeaways: (1) the bound does **not** depend on link capacity — a 10 Gbps link with 0.1 % random loss still limits one classic TCP flow to single-digit Mbps at this RTT; (2) doubling RTT halves the bound. Modern congestion-control algorithms (CUBIC, BBR, etc.) behave differently, so use this only to build intuition.

### 9.3 Mean vs percentile

Y.1541 states IPTD as a **mean** and the delay-variation objective as an **upper quantile**. SLAs written on averages can hide a bad tail that voice actually suffers from. Ask what statistic, over what interval, is being measured.

### 9.4 Measuring one-way delay and jitter

One-way delay needs synchronized clocks. RFC 3393's IPDV is a *difference* of two one-way delays, so a constant clock offset cancels, but clock drift still affects precision. Round-trip delay (RFC 2681) avoids clock sync but mixes both directions — problematic when paths or QoS treatment are asymmetric.

---

## 10. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | G.114's 150 ms is **mouth-to-ear**, not network-only | Endpoints can consume more than half the budget |
| 2 | Average utilisation hides micro-bursts | A "30 % utilised" link can still drop voice |
| 3 | QoS cannot create bandwidth | Persistent overload needs capacity or admission control |
| 4 | Prioritising everything prioritises nothing | Priority only helps if it is scarce and policed |
| 5 | "Jitter" from different tools is not the same measurement | RFC 3393 ≠ RFC 3550 ≠ Y.1541 statistic |
| 6 | Serialization delay is per hop and depends on packet size and link rate | Small on fast links, huge on slow ones |
| 7 | Loss is judged relative to traffic type | Same loss %, very different effect on voice vs TCP |
| 8 | QoS is per hop and per direction | One unmanaged congested hop breaks the end-to-end result |

---

## 11. Quick Recap

| Concept | One-line answer |
|---|---|
| When does QoS matter? | Only when a link is congested |
| Four parameters | Bandwidth, delay, jitter, loss |
| Only controllable delay component | Queuing delay |
| Voice delay target | ≤ 150 ms one-way **mouth-to-ear** (ITU-T G.114); 400 ms upper planning limit |
| Y.1541 class for VoIP | Class 0 (≤ 100 ms, ≤ 50 ms IPDV, ≤ 10⁻³ loss) |
| Elastic vs inelastic | Elastic adapts (TCP); inelastic does not (voice) |
| G.711 voice bandwidth | 80 kbps at L3, 87.2 kbps on Ethernet (20 ms packets) |
| QoS toolset order | Classify → mark → police/shape → queue/schedule → AQM/ECN |

---

## References

**Standards / Recommendations**
- ITU-T G.114 — One-way transmission time
- ITU-T Y.1540 — IP packet transfer and availability performance parameters
- ITU-T Y.1541 (12/2011) — Network performance objectives for IP-based services
- ITU-T G.1010 — End-user multimedia QoS categories (referenced through RFC 4594)
- RFC 3393 — IP Packet Delay Variation Metric for IPPM (Standards Track)
- RFC 3550 — RTP: A Transport Protocol for Real-Time Applications (STD 64; obsoletes RFC 1889)
- RFC 7679 — A One-Way Delay Metric for IPPM (obsoletes RFC 2679); RFC 7680 — A One-Way Loss Metric for IPPM (STD 82; obsoletes RFC 2680); RFC 2681 — A Round-trip Delay Metric for IPPM

**Informational / Guidance**
- RFC 4594 — Configuration Guidelines for DiffServ Service Classes (Informational)
- RFC 5976 — Y.1541-QOSM (Experimental; a secondary summary of the Y.1541 classes)
- RFC 1633 — Integrated Services in the Internet Architecture (elastic / real-time terms)

**Academic**
- M. Mathis, J. Semke, J. Mahdavi, T. Ott — "The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm", ACM CCR 27(3), 1997

**Vendor documentation**
- Cisco Press — *End-to-End QoS Network Design: Quality of Service in LANs, WANs, and VPNs* (2004), Chapter 2 "QoS Design Overview"
- Cisco — Quality of Service for Voice over IP (jitter-buffer and G.114 statements)
