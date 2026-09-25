## Link Efficiency Mechanisms — LFI, Header Compression, MLPPP

> 💡 **TL;DR:** Two related but distinct problems appear on **slow links** (historically: dial-up modems, low-speed WAN circuits; still relevant today on constrained access links): a large data packet **blocks** a small voice packet from being sent for too long (the exact serialization-delay problem worked out numerically in `1-QoS-Fundamentals` §3.2), and the voice packet's own **headers** are disproportionately large relative to its payload (`1-QoS-Fundamentals` §6). **Link Fragmentation and Interleaving (LFI)** fixes the first problem: **MLPPP** (RFC 1990) provides the fragmentation/reassembly machinery, and its **Multi-Class Extension** (RFC 2686) defines exactly how to interleave high-priority fragments between the fragments of a large low-priority packet. **Header compression** fixes the second: **cRTP** (RFC 2508) compresses the combined 40-byte IP/UDP/RTP header down to as little as 2 bytes by exploiting the fact that most header fields don't change from packet to packet within one call. Both mechanisms are widely characterized as **legacy** — increasing link speeds have shrunk the serialization-delay problem, and modern access technologies rarely need this specific fix — but the underlying arithmetic remains exactly correct wherever a genuinely slow or narrow link still exists, and the concepts (fragmentation/interleaving of priority classes; state-based delta compression of mostly-static headers) recur in other forms elsewhere in networking.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / observed deployment status · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 1990 (MLPPP), RFC 2686 (Multi-Class Extension), RFC 2508 (cRTP) — all Standards Track. RFC 1717 (MLPPP's predecessor) — obsoleted by RFC 1990. RFC 1144 (the original TCP header compression this series builds on) — Standards Track.

---

## 1. The Two Problems, Precisely Restated

### 1.1 Problem 1: Serialization delay (recap from note 1)

`1-QoS-Fundamentals` §3.2 already computed this exactly: on a 128 kbps link, a 1500-byte packet takes **93.75 ms** to serialize. If a voice packet becomes ready to send just after that large packet has started transmitting, it must wait for the *entire remaining* serialization time — on a non-preemptible link, there is no way to "cut in line" once transmission of a frame has begun.

### 1.2 Problem 2: Header overhead (recap from note 1)

`1-QoS-Fundamentals` §6 already computed the second problem: a G.729 voice payload is only 20 bytes, but IP+UDP+RTP headers add another 40 bytes (or 60 for IPv6) — **the headers are literally larger than the data** for low-bit-rate codecs. On a slow link, this overhead consumes serialization time that carries zero actual voice information.

```
 Endpoint ---> [ slow WAN link, e.g. 28.8-128 kbps ] ---> Endpoint
                    ^                    ^
              Problem 1:            Problem 2:
              a large packet        a small voice packet's
              blocks a small        HEADERS are bigger than
              one for too long      its actual PAYLOAD
              (serialization        (40-60 bytes of header
               delay)                for 20-160 bytes of voice)
```

Both problems only matter on **slow enough** links — exactly the qualifier `1-QoS-Fundamentals` §3.2's table already demonstrated numerically: at 1 Gbps, the same 1500-byte packet serializes in 12 **microseconds**, not milliseconds — nobody needs LFI or header compression there. This entire note applies specifically to **narrow, typically WAN-access-speed** links.

---

## 2. LFI Part 1 — MLPPP Provides the Fragmentation Machinery (RFC 1990)

### 2.1 What MLPPP actually is

`[Standard-defined]` RFC 1990's own stated goal, quoted directly: *"to coordinate multiple independent links between a fixed pair of systems, providing a virtual link with greater bandwidth than any of the constituent members."* The aggregated set of links is called a **bundle**. This is worth stating plainly: **MLPPP's primary design purpose is bandwidth aggregation** (combine several links into one logical fatter pipe) — its usefulness for LFI is a **side effect** of the fragmentation machinery it happens to define, not MLPPP's original motivating problem.

### 2.2 The fragment header

`[Standard-defined]`, cross-verified: MLPPP introduces a new PPP protocol type (PID `0x003d`) and defines a fragment header carrying:

| Field | Long format (24-bit, default/required) | Short format (12-bit, negotiable) |
|---|:--:|:--:|
| **B** (Beginning) bit | 1 bit — set on the **first** fragment of a packet | Same |
| **E** (Ending) bit | 1 bit — set on the **last** fragment of a packet | Same |
| **Sequence number** | 24 bits | 12 bits |
| Total header size | 4 octets | 2 octets |

`[Standard-defined]` RFC 1990 §4.1, quoted directly: *"On each member link in a bundle, the sender MUST transmit fragments with strictly increasing sequence numbers"* — this single rule is what lets the receiver detect **lost fragments** (a gap in the sequence) and correctly **reassemble** fragments arriving out of order across multiple physical links back into the original packet, using the B/E bits to know where each reconstructed packet starts and ends.

```
 Original 1500-byte packet, fragmented into 3 pieces:

  Fragment 1: B=1 E=0 seq=100  [ first 500 bytes  ]
  Fragment 2: B=0 E=0 seq=101  [ middle 500 bytes ]
  Fragment 3: B=0 E=1 seq=102  [ last 500 bytes   ]
```

### 2.3 Why plain MLPPP fragmentation alone is not yet "LFI"

Fragmenting a large packet into smaller pieces, by itself, only helps if something **useful** can be sent in between those pieces. Plain MLPPP fragmentation across a bundle of links already reduces the *maximum* delay a small packet faces (since a fragment is smaller than the whole packet), but genuinely **interleaving** a high-priority packet's fragments between a low-priority packet's fragments — the "I" in LFI — needs the specific **priority/class distinction** that plain RFC 1990 does not define. That's exactly what RFC 2686 adds.

---

## 3. LFI Part 2 — The Multi-Class Extension Adds Interleaving (RFC 2686)

### 3.1 The exact motivating numbers, quoted directly

`[Standard-defined]` RFC 2686's own opening justification, quoted precisely — and it is the **same calculation** already verified independently in `1-QoS-Fundamentals` §3.2: *"a 1500 byte packet on a 28.8 kbit/s modem link makes this link unavailable for the transmission of real-time information for about 400 ms. This adds a worst-case delay that causes real-time applications to operate with round-trip delays on the order of at least a second -- unacceptable for real-time conversation."*

**Verifying this number independently**: 1500 bytes = 12,000 bits; at 28.8 kbps, 12,000 ÷ 28,800 ≈ **0.417 seconds ≈ 417 ms** — matching RFC 2686's stated "about 400 ms" almost exactly. This cross-check against the RFC's own worked example confirms the general serialization-delay formula from note 1 is being applied consistently across this entire series.

### 3.2 The fix, quoted directly

`[Standard-defined]` RFC 2686, quoted precisely: *"The PPP extensions defined in this document allow a sender to fragment the packets of various priorities into multiple classes of fragments, allowing high-priority packets to be sent between fragments of lower priorities."*

```
 WITHOUT interleaving (plain MLPPP or no fragmentation at all):

   [==== entire 1500-byte data packet, ~417ms on 28.8kbps ====] then [voice packet]
                                                                       ^ waits ~417ms

 WITH LFI (Multi-Class Extension):

   [data frag 1] [VOICE PACKET -- sent immediately!] [data frag 2] [data frag 3] ...
        ^ small fragments        ^ interleaved in            ^ resumes after
          of the large packet      the gap between              voice is sent
                                    fragments
```

By breaking the large packet into **small enough** fragments, a high-priority packet only ever has to wait for **one fragment's** worth of serialization time — not the entire original packet's — before it can be interleaved in.

### 3.3 The specific sizing numbers RFC 2686 works through

`[Standard-defined]` This is a genuinely useful piece of exact, quotable engineering guidance rather than a vague design principle. RFC 2686, quoted directly: *"Typically, the largest packet size to be expected on a PPP link is the default MTU of 1500 bytes. The smallest high-priority packets are likely to have on the order of 22 bytes (compressed RTP/G.723.1 packets). In the 1:72 range of packet sizes to be expected, this translates to a maximum requirement of about eight levels of suspension."*

**Verifying the ratio**: 1500 ÷ 22 ≈ **68**, which RFC 2686 rounds to its stated "1:72 range" — consistent, and directly connects to `12-QoS-ECN`... no, more precisely, it connects directly to the header-compression material in §4 below: the RFC's own smallest-high-priority-packet example is explicitly a **compressed** RTP packet (22 bytes), not an uncompressed one (which would be considerably larger, per §4.1's 40-byte figure) — meaning RFC 2686's own sizing rationale already assumes cRTP is deployed alongside LFI, which is exactly why these two mechanisms are conventionally paired together in practice.

`[Standard-defined]`, also quoted directly, RFC 2686's specific practical recommendation for the low end of link speeds this problem matters most for: *"On 28.8kbit/s modems, there seems to be a practical requirement for at least two levels of suspension (i.e., audio suspends any longer packet including video, video suspends other..."* — establishing that even a **minimal**, two-class (audio vs. everything else) priority scheme delivers most of the practical benefit, without needing the full eight-level scheme the byte-ratio analysis technically allows for.

### 3.4 Mechanism — the multiclass fragment header

`[Common practice]`, consistent across the vendor documentation captured in research: the Multi-Class Extension adds a **class field** to the MLPPP fragment header — 2 bits in the short (12-bit sequence number) format, 4 bits in the long (24-bit sequence number) format — so each fragment carries not just its sequence number and B/E bits, but **which priority class** it belongs to. This is the field a receiver uses to know it's looking at an interleaved, multi-priority fragment stream rather than a single-class one.

---

## 4. Header Compression — cRTP (RFC 2508)

### 4.1 The problem, in the RFC's own numbers

`[Standard-defined]` RFC 2508 (Casner & Jacobson, 1999), quoted directly: *"there is also concern that the 12-byte RTP header is too large an overhead for 20-byte payloads when operating over low speed lines such as dial-up modems at 14.4 or 28.8 kb/s."* Combined across all three layers: IPv4 (20 B) + UDP (8 B) + RTP (12 B) = **40 bytes** of header — exactly the figure already independently derived in `1-QoS-Fundamentals` §6's voice-bandwidth arithmetic.

### 4.2 Why compress all three headers together, not RTP alone

`[Standard-defined]` This is a specific, reasoned design decision worth preserving exactly, quoted directly: *"compression might be applied to the RTP header alone, on an end-to-end basis, or to the combination of IP, UDP and RTP headers on a link-by-link basis. Compressing the 40 bytes of combined headers together provides substantially more gain than compressing 12 bytes of RTP header alone because the resulting size is approximately the same (2-4 bytes) in either case."* The insight: compression overhead (the bits needed to signal *which* compressed form is being used, plus any necessary sequence/checksum information) is roughly **fixed**, regardless of how many bytes of header are being compressed — so compressing the full 40-byte stack costs barely more than compressing just the 12-byte RTP header alone, but recovers **far more** total bytes. RFC 2508 also notes the practical benefit of doing this **per-link** rather than end-to-end: *"delay and loss rate are lower"* on a single link than across the full end-to-end path, which matters because the compression scheme depends on the compressor and decompressor staying synchronized (§4.4).

### 4.3 The result — four packet formats, sized precisely

`[Standard-defined]`, quoted directly, the specific formats RFC 2508 defines:

| Format | What it carries | Typical size |
|---|---|:--:|
| **FULL_HEADER** | *"the uncompressed IP header plus any following headers and data to establish the uncompressed header state in the decompressor"* | Full 40+ bytes — sent to (re-)establish context |
| **COMPRESSED_UDP** | *"the IP and UDP headers compressed to 6 or fewer bytes (often 2 if UDP checksums are disabled), followed by any subsequent headers... in uncompressed form"* | 2–6 bytes (+ uncompressed RTP if present) |
| **COMPRESSED_RTP** | IP, UDP, and RTP headers all compressed together | **2 bytes**, or more if differences must be signaled |
| **CONTEXT_STATE** | (used for error recovery/resynchronization between compressor and decompressor) | — |

The Abstract's own summary, quoted directly: *"In many cases, all three headers can be compressed to 2-4 bytes."* Compared against the 40-byte uncompressed baseline, this is roughly a **10-to-20-fold reduction** in header overhead for the common case.

### 4.4 Why this works — the underlying principle

`[Common practice]`, reasoned directly from RFC 2508's design: within one RTP session (one voice call, for instance), most header fields **never change** from packet to packet — source/destination IP addresses, source/destination UDP ports, the RTP SSRC (synchronization source identifier), and several other fields are **constant** for the life of the call. A handful of fields **do** change, but predictably — the RTP sequence number increments by exactly 1 each packet, and the RTP timestamp increments by a fixed amount (matching the audio sampling interval) each packet. cRTP's compressor and decompressor establish this **shared context** once (the FULL_HEADER packet), after which only the **small, predictable deltas** need to be signaled at all — and in the common case where the predictable increments hold exactly, essentially nothing about the header's content needs to be sent, only a tiny amount of framing/sequencing information to keep both ends synchronized.

> 📝 This is the same fundamental idea RFC 2508 itself credits as its lineage, quoted directly: *"Header size may be reduced through compression techniques as has been done with great success for TCP"* — referencing RFC 1144's earlier TCP header compression, confirmed separately in research to compress the 40-byte IP+TCP header down to 2–4 bytes using the identical "shared context, transmit only deltas" principle. cRTP is this same established technique, applied to the specific field-change patterns of RTP media streams rather than TCP's.

### 4.5 Context — a per-flow, per-link concept

`[Standard-defined]` The compression is **stateful**: each active flow gets its own **context** (identified by the 8- or 16-bit context identifier carried in the FULL_HEADER packet), and that context exists **independently on each link** the packet crosses — directly analogous to `3-QoS-Classification-Trust`'s classification key concept, except here the "key" identifies a shared compression state rather than a QoS treatment. If the compressor and decompressor's contexts ever fall out of sync (a FULL_HEADER packet lost, for instance), the decompressor cannot correctly reconstruct subsequent compressed packets until resynchronized.

---

## 5. Putting LFI and cRTP Together — the Complete Legacy Voice-over-Slow-Link Design

`[Common practice]` §3.3 already established that RFC 2686's own worst-case sizing example (1500:22 byte ratio) assumes cRTP is already in use — the two mechanisms are **designed to complement each other**, not to be deployed independently:

```
 Endpoint (VoIP call) ---> [ cRTP: 40-byte header -> 2-4 bytes ] ---> [ LFI: fragment large
                                                                        data packets so this
                                                                        tiny voice packet can
                                                                        be interleaved between
                                                                        fragments, not blocked
                                                                        by a whole 1500-byte
                                                                        packet ] ---> slow WAN link
```

- **cRTP alone**, without LFI, still leaves the *serialization-delay* problem (§1.1) untouched — a tiny compressed voice packet can still be forced to wait behind one large, uncompressed data packet's full serialization time.
- **LFI alone**, without cRTP, still leaves the *header-overhead* problem (§1.2) untouched — even a perfectly-interleaved voice fragment still carries 40 bytes of largely-redundant header for every 20–160 bytes of actual audio payload.
- **Together**, they address both of `1-QoS-Fundamentals`'s originally identified delay/bandwidth problems for real-time traffic on a slow link, each targeting a different root cause.

---

## 6. Why This Is Explicitly Legacy — and Why It's Still Worth Knowing

`[Common practice]` The serialization-delay arithmetic from `1-QoS-Fundamentals` §3.2 already shows exactly why: at 1 Gbps, the identical 1500-byte packet that took ~417ms at 28.8 kbps takes **12 microseconds** — six orders of magnitude less. As access-link speeds have risen from dial-up-era rates into the tens or hundreds of megabits (and beyond), the specific delay problem LFI was built to solve has largely disappeared **for most modern access circuits**. Similarly, header overhead as a *fraction* of total bandwidth consumed matters far less when the underlying link has abundant capacity to spare.

**Where this still matters today**, reasoned directly from the same arithmetic:

- Any genuinely **narrow or high-latency** link — some satellite links, certain constrained IoT/cellular backhaul scenarios, or any circuit whose effective rate is still low enough that `1-QoS-Fundamentals` §3.2's serialization-delay table produces a non-trivial number.
- **The underlying concepts recur elsewhere**, even where the specific RFC 1990/2686/2508 mechanisms aren't deployed: fragmentation-plus-interleaving as a general technique for bounding one traffic class's impact on another's latency is conceptually the same idea behind link-layer scheduling more broadly (`10-QoS-Queuing-Scheduling`); and state-based delta compression of mostly-static headers is the same underlying principle later generalized into **ROHC** (Robust Header Compression, RFC 3095/4995), designed specifically to also tolerate the higher loss and reordering rates of wireless/cellular links where classic cRTP's context-synchronization assumptions are less reliable — ROHC is outside this note's scope, but its motivating problem is identical to cRTP's.

> ⚠️ **Gotcha:** Because LFI's benefit is entirely a function of link speed and packet-size mix, blindly enabling it on a link where it isn't needed adds **fragmentation and reassembly overhead** (extra header bytes per fragment, extra processing) for no corresponding latency benefit. This is the same principle already established for shaping in `9-QoS-Shaping` §6.1 — a mechanism that only matters when a specific bottleneck condition exists provides no benefit, and some cost, when applied where that condition doesn't hold.

---

## 7. CCIE-Depth Topics

### 7.1 Why RFC 2686 frames its problem in terms of "levels of suspension," not just "two priority classes"

Section 3.3's exact quotation — up to **eight levels of suspension** derived from the 1500:22 byte ratio — reflects a more general truth about interleaving than a simple "voice vs. data" binary: if a network genuinely carries several distinct traffic sizes/priorities (say, voice, video, and bulk data, each with different acceptable delay budgets), a **single** priority split (interleave only for "the top class") still lets a *medium*-priority packet be blocked by a large low-priority one for the medium packet's own unacceptable duration. RFC 2686's multiclass design (not just a two-class one) exists specifically to let a sender express **several** simultaneous suspension relationships — reflecting the real-world observation, quoted directly in the RFC as background, that video and voice may both need protection from bulk data, while voice may separately need protection from being blocked by video.

### 7.2 The reassembly cost of small fragments

Section 2.2's fragment header (2 or 4 bytes) is overhead added **per fragment**, not per original packet. Choosing a very small fragment size to minimize interleaving delay (per §3.2's diagram) trades against a **rising per-byte header cost**, since more, smaller fragments means proportionally more of these 2-4 byte headers relative to the payload actually carried. This is structurally the same trade-off already encountered in `10-QoS-Queuing-Scheduling` §8.1 for DRR's quantum sizing — a smaller unit of work reduces worst-case delay/unfairness but increases the fixed per-unit overhead cost, and the right balance depends on the specific link's speed and traffic mix rather than being a fixed, universal number.

### 7.3 Why cRTP's context concept foreshadows a more general pattern

The "establish shared state once, then send only deltas" principle underlying cRTP (§4.4) is not unique to header compression — it is structurally the same idea behind RSVP's **soft state** (`2-QoS-Models` §3.3: install, then refresh periodically rather than resending everything) and, in a very different context, BGP's incremental updates (send only what has changed, not the full table repeatedly). Recognizing "shared context plus small deltas" as a recurring networking design pattern — rather than a cRTP-specific trick — helps generalize this note's lesson well beyond legacy slow-link voice engineering.

---

## 8. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | LFI and header compression solve **two different problems** (blocking delay vs. header overhead) | Deploying one without the other leaves the other problem fully unsolved |
| 2 | MLPPP's **original** purpose is bandwidth aggregation; LFI relies on its fragmentation machinery as a side benefit | Don't assume "multilink" automatically implies interleaving — the Multi-Class Extension (RFC 2686) is a separate, additional standard |
| 3 | RFC 2686's own sizing example (1500:22 bytes) already **assumes cRTP is in use** | The two mechanisms are designed to be paired, not deployed in isolation |
| 4 | Compressing all three headers together (IP+UDP+RTP) is far more efficient than compressing RTP alone | Fixed compression overhead means the marginal cost of compressing more header bytes together is small |
| 5 | cRTP's compression state (**context**) is per-flow and per-link, and can fall out of sync | A lost FULL_HEADER packet can break decompression until resynchronized |
| 6 | LFI's fragment headers add **per-fragment** overhead | Very small fragments reduce blocking delay but increase relative header cost — a genuine trade-off, not a free win |
| 7 | Both mechanisms are effectively obsolete on **high-speed** links | The serialization-delay math (`1-QoS-Fundamentals` §3.2) shows the underlying problem shrinks by orders of magnitude as link speed rises |
| 8 | The "shared context, transmit only deltas" principle behind cRTP recurs elsewhere in networking | RSVP soft state and incremental routing updates are the same underlying design pattern |

---

## 9. Quick Recap

| Concept | One-line answer |
|---|---|
| Problem 1 | A large packet blocks a small, urgent one for its entire serialization time |
| Problem 2 | 40 bytes of IP/UDP/RTP header for as little as 20 bytes of voice payload |
| MLPPP (RFC 1990) | Bundles multiple links; defines the fragment header (B/E bits, sequence number) that makes fragmentation/reassembly possible |
| Multi-Class Extension (RFC 2686) | Adds a class field so high-priority fragments can be interleaved between low-priority fragments — this is what makes it genuine LFI |
| RFC 2686's own numbers | ~400ms blocking on a 28.8 kbps link for a 1500-byte packet; up to 8 suspension levels from a 1500:22 byte ratio |
| cRTP (RFC 2508) | Compresses the combined 40-byte IP/UDP/RTP header to as little as 2 bytes, using shared per-flow context and predictable field deltas |
| Why compress all three headers together | Fixed compression overhead means combined compression is far more efficient than compressing RTP alone |
| Why this is "legacy" | Link speeds have risen by orders of magnitude since these RFCs were written (1996–1999), shrinking the problem they solve |
| Where it still matters | Genuinely narrow/high-latency links; and the underlying design patterns (interleaving, delta compression) recur elsewhere |

---

## References

**Standards Track**
- RFC 1990 — The PPP Multilink Protocol (MP) (obsoletes RFC 1717)
- RFC 2686 — The Multi-Class Extension to Multi-Link PPP
- RFC 2508 — Compressing IP/UDP/RTP Headers for Low-Speed Serial Links (cRTP)
- RFC 1144 — Compressing TCP/IP Headers for Low-Speed Serial Links (the earlier TCP-specific technique cRTP's design builds on)

**Referenced (background and what feeds into this note)**
- `1-QoS-Fundamentals` (§3.2 serialization-delay arithmetic; §6 voice-bandwidth/header-overhead arithmetic — both independently re-verified against RFC 2686's and RFC 2508's own numbers in this note)
- `2-QoS-Models` (RSVP soft state — the same "shared context, incremental updates" pattern as cRTP)
- `9-QoS-Shaping` (the general principle that a mechanism solving a specific bottleneck provides no benefit where that bottleneck doesn't exist)
- `10-QoS-Queuing-Scheduling` (DRR quantum-sizing trade-off — structurally the same overhead-vs-fairness trade-off as LFI fragment sizing)

**Mentioned for context, not detailed in this note**
- RFC 3095, RFC 4995 — Robust Header Compression (ROHC), the generalization of cRTP's principle for lossier/higher-latency links such as cellular
