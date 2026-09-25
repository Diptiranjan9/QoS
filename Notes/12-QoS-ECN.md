## ECN — Explicit Congestion Notification

> 💡 **TL;DR:** ECN (RFC 3168) lets a router signal congestion **without dropping the packet** — it sets 2 bits already reserved in the DS field (`4-QoS-Marking-Headers` §2.3) to **CE (Congestion Experienced)** instead of discarding, provided the packet was marked **ECT** (ECN-Capable Transport) by its sender. For TCP, this requires a **three-way negotiation** at connection setup (the SYN/SYN-ACK exchange sets two new TCP flags, **ECE** and **CWR**) and a defined **feedback loop**: receiver sees CE → sets ECE on ACKs until acknowledged → sender cuts its congestion window exactly as it would for a dropped packet → sender sets CWR once → receiver stops echoing. ECN doesn't invent a new congestion-response algorithm — RFC 3168 is explicit that CE is meant to trigger **the same TCP reaction as a dropped packet**, just without actually losing the data. RFC 8311 (2018) later relaxed several of RFC 3168's original restrictions — retiring the never-widely-deployed **ECN nonce** and freeing the **ECT(1)** codepoint for new experimentation, which is exactly the codepoint **L4S** (`13-QoS-L4S-AccECN`) now uses.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / observed deployment behaviour · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 3168 — Standards Track (updates RFC 2474, RFC 2401, RFC 793; obsoletes RFC 2481). RFC 8311 — Standards Track (updates RFC 3168 and several others). RFC 3540 (ECN nonce) — reclassified from Experimental to **Historic** by RFC 8311.

---

## 1. The Core Idea, Precisely

`[Standard-defined]` RFC 3168 Abstract: *"This memo specifies the incorporation of ECN (Explicit Congestion Notification) to TCP and IP, including ECN's use of two bits in the IP header."* Those two bits are exactly the ones already identified in `4-QoS-Marking-Headers` §2.3 as the low 2 bits of the DS field — RFC 3168 **updates RFC 2474** to formally claim them.

```
   bit:    0   1   2   3   4   5   6   7
         +---+---+---+---+---+---+---+---+
         |      DSCP (6 bits)    |  ECN  |
         +---+---+---+---+---+---+---+---+
                                   \_ /
                            these two bits, RFC 3168
```

**Why this matters mechanically**: without ECN, a router facing incipient congestion has exactly one signal to send an end host — **drop the packet**. RFC 3168 gives it a second option: **mark** the packet (change 2 bits, forward it intact) instead of dropping it, **provided the sender indicated it can understand that mark**. The packet still arrives; no retransmission is needed; but the receiver — and, through TCP's feedback loop, the sender — still learns that congestion is building.

---

## 2. The Four Codepoints

`[Standard-defined]` (RFC 3168 §5, exact values, cross-referenced with `4-QoS-Marking-Headers`'s bit-numbering convention)

| ECN field (binary) | Name | Meaning |
|:--:|---|---|
| `00` | **Not-ECT** | Not an ECN-Capable Transport — the sender does not support (or is not using) ECN for this packet |
| `10` | **ECT(0)** | ECN-Capable Transport, codepoint "0" |
| `01` | **ECT(1)** | ECN-Capable Transport, codepoint "1" |
| `11` | **CE** | Congestion Experienced — set by a router, never by the original sender |

RFC 3168's own reasoning for having **two** ECT codepoints instead of one, quoted directly from the RFC text captured in research: the `01` codepoint was left undefined in the predecessor document (RFC 2481), and RFC 3168 recommends using **ECT(0)** *"when only a single ECT codepoint is needed"* by a sender — meaning, for ordinary (non-experimental) use, a sender picks ECT(0) and ECT(1) exists as a **second, distinguishable "I am ECN-capable" signal**, historically reserved for the ECN nonce (§7) and now repurposed for L4S (§8).

### 2.1 The critical asymmetry: only routers set CE

`[Standard-defined]` **CE is never set by the original sender.** A sender marks its own packets ECT(0) or ECT(1) (or leaves them Not-ECT); only a congested router along the path **changes an ECT-marked packet's bits to CE**. This asymmetry is the entire mechanism: the sender's ECT mark is *permission* ("you may mark my packets instead of dropping them"), and the router's CE mark is the *actual congestion signal*.

```
 Sender marks: ECT(0) or ECT(1)  ---->  [ congested router ]  ---->  rewrites to CE  ---->  receiver sees CE
                (permission)              (Algorithmic Dropper,        (the actual signal)
                                           `11-QoS-Congestion-
                                            Avoidance` §3-6)
```

### 2.2 The rule for when a router may set CE instead of dropping

`[Standard-defined]` RFC 3168 §5, quoted precisely: *"A router MUST NOT set the CE codepoint if the ECN field is set to Not-ECT."* — i.e., a router can only mark, never invent ECN-capability the sender didn't already signal. Also quoted directly: *"A router MUST NOT set CE instead of dropping a packet when the drop that would occur is caused by reasons other than congestion or the desire to indicate incipient congestion... (e.g., a diffserv edge node may be configured to unconditionally drop certain classes of traffic to prevent them from entering its diffserv domain)."* And: *"We expect that routers will set the CE codepoint in response to incipient congestion as indicated by the average queue size, using the RED algorithms suggested in [Floyd & Jacobson 1993, RFC 2309]."*

This is the exact, precise link back to `11-QoS-Congestion-Avoidance`: **every AQM algorithm covered in that note (RED, CoDel, PIE) can choose to mark CE instead of dropping, for any packet already carrying an ECT codepoint** — this is not a separate mechanism bolted on afterward, it is what RFC 3168 designed those algorithms' "drop decision" to become when ECN is available. This is also the reason `2-QoS-Models` §4.2's DS field description flagged the CU bits as "later assigned to ECN" — RFC 3168 is precisely that later assignment.

> ⚠️ **Gotcha (ties to note 3 §5.4):** Because a marking `[Standard-defined]` PHB selection **MUST ignore** the ECN bits (RFC 2474, `3-QoS-Classification-Trust` §5.4), a device that matches the **entire 8-bit ToS byte** rather than the 6-bit DSCP will treat an ECT/CE-marked packet as a *different* value than its Not-ECT counterpart, even though its DSCP — and therefore its intended PHB — is identical. EF (0xB8) and EF-with-ECT(0) (0xBA) are the same class; a byte-match classifier would treat them as different classes entirely.

---

## 3. TCP's Use of ECN — the Full Mechanism

### 3.1 Why TCP needs new flags at all

`[Standard-defined]` RFC 3168 §6, quoted directly, states TCP needs **three** new pieces of functionality: *"negotiation between the endpoints during connection setup to determine if they are both ECN-capable; an ECN-Echo (ECE) flag in the TCP header so that the data receiver can inform the data sender when a CE packet has been received; and a Congestion Window Reduced (CWR) flag in the TCP header so that the data sender can inform the data receiver that the congestion window has been reduced."*

Two new bits are defined in TCP's previously-Reserved header field (RFC 793's 6-bit Reserved field, bits 4–9): **ECE** (bit 9, ECN-Echo) and **CWR** (bit 8, Congestion Window Reduced).

### 3.2 Negotiation — the three-way handshake

`[Standard-defined]` RFC 3168 §6.1.1, terms and rules quoted precisely:

- **"ECN-setup SYN packet"**: a SYN packet with **both ECE and CWR flags set**.
- **"ECN-setup SYN-ACK packet"**: a SYN-ACK with **ECE set but CWR not set**.
- *"If a host has received an ECN-setup SYN packet, then it MAY send an ECN-setup SYN-ACK packet. Otherwise, it MUST NOT send an ECN-setup SYN-ACK packet."*
- *"A host MUST NOT set ECT on data packets unless it has sent at least one ECN-setup SYN or ECN-setup SYN-ACK packet, and has received at least one ECN-setup SYN or ECN-setup SYN-ACK packet, and has sent no non-ECN-setup SYN or SYN-ACK packet."*
- **Neither SYN nor SYN-ACK ever carries ECT in the IP header** — RFC 3168 §6.1.1: *"A host MUST NOT set ECT on SYN or SYN-ACK packets."* Only ordinary **data** packets, after successful negotiation, are ECT-marked.

```
 Host A (initiator)                          Host B (responder)
       |                                             |
       |----- SYN, ECE=1, CWR=1  (ECN-setup SYN) --->|
       |                                             |
       |<---- SYN-ACK, ECE=1, CWR=0 -----------------|   (ECN-setup SYN-ACK:
       |      (ECN-setup SYN-ACK)                     |    "I am ECN-capable too")
       |                                             |
       |----- ACK ---------------------------------->|
       |                                             |
       |===== Data packets, ECT(0) set in IP =======>|   (ECN now in use for this
       |<==== Data packets, ECT(0) set in IP ========|    connection, both directions)
```

**Why the flag pattern is deliberately asymmetric between SYN and SYN-ACK** — quoted directly, this is the exact mechanism that lets ECN-capability be negotiated safely even against a broken middlebox: *"Because the TCP SYN packet sets the ECN-Echo and CWR flags to indicate ECN-capability, while the SYN-ACK packet sets only the ECN-Echo flag, the sending TCP correctly interprets a receiver's reflection of its own flags in the Reserved field as an indication that the receiver is not ECN-capable. The sending TCP is not misled by a faulty TCP implementation sending a SYN-ACK packet that simply reflects the Reserved field of the incoming SYN packet."* If a naive/non-ECN-aware stack simply echoed back whatever Reserved-field bits it received (both ECE and CWR set, mirroring the SYN), the initiator can tell this apart from a genuine ECN-aware reply (ECE set, CWR clear) — a deliberately asymmetric design specifically to detect **reflection bugs**, not just absence of support.

### 3.3 The steady-state feedback loop

`[Standard-defined]`, verified step by step against RFC 3168 §6.1.2–6.1.3:

```
 1. Sender:    transmits a data packet with ECT set in the IP header.
 2. Router:    (congested) rewrites ECT --> CE, forwards the packet unchanged otherwise.
 3. Receiver:  sees the CE-marked packet arriving. Sets the ECE flag on EVERY ACK it
               sends back, starting now, and KEEPS setting it on every subsequent ACK
               -- until it receives a CWR-flagged packet from the sender.
 4. Sender:    receives an ACK with ECE set. Treats this EXACTLY as it would treat a
               dropped packet -- reduces its congestion window (e.g., halves it, under
               standard TCP congestion control) -- but the data itself was never lost.
 5. Sender:    sets the CWR flag on the very next NEW data packet it sends.
 6. Receiver:  sees CWR. Stops setting ECE on further ACKs (until the next CE event).
```

**Why the receiver must keep repeating ECE, not send it just once** — quoted directly from RFC 3168 (already surfaced in earlier research, consistent across multiple independent citations): the receiver keeps setting ECE on **every** subsequent ACK, precisely because a single ACK carrying the ECE flag could itself be **lost** on the return path. If the receiver only signaled once, that one signal could vanish without the sender ever learning about the congestion event. By repeating it until acknowledged (via CWR), at least one ECE-carrying ACK is virtually guaranteed to get through, even under packet loss on the reverse path.

**Why the sender reduces its window only once per event, not once per ECE-flagged ACK received**: RFC 3168 treats a CE-marking event the same way TCP already treats a loss event under standard congestion avoidance — a **single congestion-window reduction per round-trip time**, not a cumulative reduction for every duplicate signal describing the *same* underlying event. Since the receiver keeps echoing ECE for roughly one RTT (until the sender's CWR arrives and is acknowledged), any additional ECE-flagged ACKs received within that same window are understood to refer to the **same** congestion event already responded to, not a new one.

**What CWR itself means and when it's resent** — quoted directly: *"When an ECN-Capable TCP sender reduces its congestion window for any reason (because of a retransmit timeout, a Fast Retransmit, or in response to an ECN Notification), the TCP sender sets the CWR flag in the TCP header of the first new data packet sent after the window reduction. If that data packet is dropped in the network, then the sending TCP will have to reduce the congestion window again and retransmit the dropped packet."* — meaning CWR is not just an ECN-specific flag; it announces *any* congestion-window reduction, whatever triggered it.

### 3.4 The special case: congestion window already at 1

`[Standard-defined]` RFC 3168 §6.1.2, quoted directly: *"It is necessary to still reduce the sending rate of the TCP sender even further, on receipt of an ECN-Echo packet when the congestion window is one. We use the retransmit timer as a means of reducing the rate further in this circumstance. Therefore, the sending TCP MUST reset the retransmit timer on receiving the ECN-Echo packet when the congestion window is one."* This is a deliberately-specified edge case: if the window is already at its minimum (1 segment), TCP can't literally halve it any further in a meaningful way, so RFC 3168 mandates a different mechanism (resetting the retransmit timer) to still produce a real slowdown.

### 3.5 CE packets signal persistent, not transient, congestion

`[Standard-defined]` RFC 3168 (quoted from research above): *"CE packets indicate persistent rather than transient congestion... and hence reactions to the receipt of CE packets should be those appropriate for persistent congestion."* This is a subtle but important framing: ECN doesn't turn every brief queue fluctuation into a signal — recall from `11-QoS-Congestion-Avoidance` that AQM algorithms (RED's averaging, CoDel's minimum-over-a-window) are already designed to filter out short-lived bursts before deciding to mark or drop at all. By the time a CE mark reaches the receiver, the underlying AQM mechanism has already concluded the congestion is sustained enough to act on.

---

## 4. A Documented Real-World Failure Mode: Middleboxes That Break ECN

`[Guidance]` This is not a hypothetical concern — it is why RFC 3168 itself includes a specific, mandatory work-around, and it is documented with actual measurement data. Research directly confirmed: *"In March 2002, six months after ECN was approved as Proposed Standard, ECN-setup SYN packets were answered by a reset from 203 of the 12,364 web sites tested, and ECN-setup SYN packets were dropped for 420 of the web sites."* Some firewalls and middleboxes, seeing unexpected bits set in TCP's Reserved field, treated ECN-setup SYN packets as malformed or suspicious and either dropped them silently or actively reset the connection.

**RFC 3168's mandated work-around**, quoted directly:

- *"a host that receives a RST in response to the transmission of an ECN-setup SYN packet MAY resend a SYN with CWR and ECE cleared. This could result in a TCP connection being established without using ECN."*
- *"A host that receives no reply to an ECN-setup SYN within the normal SYN retransmission timeout interval MAY resend the SYN and any subsequent SYN retransmissions with CWR and ECE cleared."*

This is a genuine, RFC-specified **fallback**: if ECN negotiation appears to be actively breaking connectivity, the TCP stack retries **without** the ECN flags, sacrificing ECN's benefit for that connection in favour of establishing the connection at all. This history connects directly to `3-QoS-Classification-Trust` §7's coverage of the general phenomenon of markings being altered or stripped in transit (RFC 9435) — ECN's TCP flags experienced an analogous "middlebox interference" problem, just at the transport layer's flag bits rather than the IP layer's DSCP bits.

---

## 5. ECN and Tunnels — Recap and Forward Reference

`[Guidance]` `3-QoS-Classification-Trust` §5.7 already noted that a device classifying an encapsulated packet sees only the **outer** header. ECN raises a specific, additional question beyond ordinary DSCP tunneling: if an **outer** tunnel header gets CE-marked by a router along the tunnel path, that congestion signal must somehow be **copied to the inner header** at the tunnel egress, or the signal is lost entirely — the original sender, operating on the inner header's information, would never learn the outer path was congested. This exact problem, and its standards-defined solution (RFC 6040), is covered in full in `14-QoS-Tunnels-Overlays`; it is flagged here because it is a direct, ECN-specific consequence of the tunneling behaviour already established in note 3.

---

## 6. Beyond TCP — a Brief Note

`[Guidance]` RFC 3168 §6 states plainly that its TCP mechanism is the RFC's focus, *"leaving issues of ECN in other transport protocols to further research."* ECN has since been specified for other protocols — RFC 8311 itself lists updates it makes to ECN specifications for **RTP** (RFC 6679) and **DCCP** (RFCs 4341, 4342, 5622) — but the detailed feedback mechanisms for those protocols are outside this note's scope, which focuses on the TCP mechanism as the reference case.

---

## 7. RFC 8311 — Relaxing RFC 3168's Original Restrictions

### 7.1 Purpose, in the RFC's own words

`[Standard-defined]` RFC 8311 (Black, January 2018) Abstract, quoted directly: *"This memo updates RFC 3168... It relaxes restrictions in RFC 3168 that hinder experimentation towards benefits beyond just removal of loss."* Its key structural mechanism, also quoted directly: *"An Experimental RFC in the IETF document stream is required to take advantage of any of these enabling updates"* — RFC 8311 itself doesn't mandate any new behaviour; it removes blanket prohibitions from RFC 3168 so that **future, specific Experimental RFCs** are permitted to try new things without contradicting RFC 3168's original text.

### 7.2 Retiring the ECN nonce

`[Standard-defined]` RFC 3168's original design touched on an optional mechanism called the **ECN nonce** (fully specified separately, in RFC 3540) — a scheme using the **choice** between ECT(0) and ECT(1) as a way for a sender to detect a receiver or network element that was **lying about receiving CE marks** (deliberately hiding congestion signals to gain an unfair throughput advantage). RFC 8311, quoted directly: *"This memo also records the conclusion of the ECN nonce experiment in RFC 3540 and provides the rationale for reclassification of RFC 3540 from Experimental to Historic; this reclassification enables new experimental use of the ECT(1) codepoint."*

In concrete terms, RFC 8311 **removes** several specific pieces of RFC 3168's original text (verified directly, itemized in the RFC): the paragraph in §5 motivating two ECT codepoints via the nonce, the entire discussion section on the nonce (§11.2 in the original), and nonce-related material in two other sections — specifically **freeing ECT(1)** from its old association, since the nonce experiment concluded without becoming a deployed mechanism.

### 7.3 Why this matters: ECT(1) becomes available for L4S

`[Standard-defined]` This is the direct, single most consequential change RFC 8311 makes for the rest of this series: with ECT(1) no longer tied to the (retired) nonce, RFC 8311 states explicitly that Congestion Marking Differences experiments **MUST NOT** change network behaviour for ECT(0)-marked traffic in ways that require a different sender response — but **may** do so for **ECT(1)**, provided the Experimental RFC defining that behaviour specifies both the sender's congestion response and any router behaviour changes. This is precisely the standards mechanism that **L4S** (`13-QoS-L4S-AccECN`) uses: L4S traffic is marked **ECT(1)**, specifically because RFC 8311 opened that codepoint up for exactly this kind of differentiated treatment, while ECT(0) remains locked to classic ECN's original, conservative semantics.

### 7.4 Other areas RFC 8311 opened for experimentation

`[Standard-defined]`, briefly (full detail deferred to `13-QoS-L4S-AccECN` where relevant): RFC 8311 also permits experiments in **Alternative ECN Semantics** (letting a marked packet convey something other than "reduce your window as if this were a drop") and adjustments to **router forwarding behaviour for CE-marked packets** that are part of a defined experiment — again, always gated on ECT(1) and an Experimental RFC, never applying by default to ordinary ECT(0) traffic.

---

## 8. CCIE-Depth Topics

### 8.1 Why the SYN/SYN-ACK flag asymmetry is a genuinely clever piece of protocol design

Revisit §3.2's mechanism once more, because it rewards close reading: a broken TCP stack that doesn't understand the new flags at all will, by RFC 793's own rules, typically send a SYN-ACK with the Reserved field **cleared** — which correctly signals "not ECN-capable" without the stack needing to know anything about ECN. A **different** category of broken stack — one that naively reflects whatever Reserved-field bits it received — would echo back **both** ECE and CWR set (mirroring the ECN-setup SYN it received), rather than the genuine ECN-capable reply pattern of **ECE set, CWR clear**. Because genuine ECN support requires **asymmetric** flags between SYN and SYN-ACK, this reflection bug is detectable and RFC 3168 explicitly designed around it — a single-flag design (just "I support ECN, yes/no", mirrored identically each direction) could not have caught this class of implementation bug at all.

### 8.2 The "once per RTT" rule connects directly to standard TCP congestion control

`8-QoS-Policing` and general TCP theory both rest on the principle that congestion response should happen **once per round-trip**, not once per lost/marked packet, because many losses or marks within a single RTT typically describe the **same** underlying congestion event, not several independent ones. RFC 3168's ECE/CWR handshake is precisely engineered to preserve this same-event correlation for CE-marking exactly as TCP's existing loss-based logic already required for drops — this is why RFC 3168 could plug into existing TCP implementations by triggering the **same** congestion-window-reduction code path already used for loss, rather than needing an entirely new response algorithm.

### 8.3 Why RFC 3168 forbids ECT on SYN/SYN-ACK, and why that later needed revisiting

Recall §3.2: neither SYN nor SYN-ACK may be ECT-marked under RFC 3168 — meaning the connection's very first two packets get **no** ECN protection and, if dropped due to congestion, must rely on ordinary retransmission timeouts, which are especially costly this early in a connection (before any RTT/RTO estimate is well established). This specific gap was significant enough that a **separate**, later Experimental RFC (RFC 5562, briefly encountered in research above, not detailed further here) proposed extending ECN to SYN-ACK packets specifically — illustrating that RFC 3168's original restrictions, even before RFC 8311, were understood by the community to be conservative on purpose, with room identified for further extension via the normal Experimental-RFC process rather than by silently deviating from the Standards Track text.

### 8.4 The DSCP/ECN independence has a practical corollary for policy design

Because `4-QoS-Marking-Headers` and this note jointly establish that DSCP (6 bits, class) and ECN (2 bits, congestion signal) are **independent** fields in the same byte, a policy (`7-QoS-Policy-Model`) that matches on DSCP alone will correctly group ECT(0)/ECT(1)/CE variants of the same class together — but a policy relying on the **literal ToS byte value** will not. This is the same point made as a gotcha in §2.2 above, restated as a design principle: **always build classification and policy rules on the 6-bit DSCP, never the full 8-bit ToS/Traffic-Class byte**, specifically because ECN's bits will vary independently of class.

---

## 9. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | Only a **router** ever sets CE; the original sender only ever sets ECT(0)/ECT(1) | ECT is *permission* to mark; CE is the actual *signal* |
| 2 | A router **MUST NOT** set CE for a drop unrelated to congestion (e.g., a policy-based deny) | ECN is specifically a congestion signal, not a general substitute for dropping |
| 3 | Neither SYN nor SYN-ACK may carry ECT in the IP header | The connection's first two packets get no ECN protection; only established-connection data packets do |
| 4 | The receiver **repeats** ECE on every ACK until it sees CWR — it doesn't send the signal only once | Protects against the ECE-carrying ACK itself being lost on the return path |
| 5 | The sender reduces its window **once per event** (effectively once per RTT), not once per ECE-flagged ACK | Multiple ECE-flagged ACKs within one RTT typically describe the same congestion event |
| 6 | Real middleboxes have historically dropped or reset ECN-setup SYN packets | RFC 3168 mandates a fallback: retry without ECN flags if negotiation appears to break connectivity |
| 7 | ECN bits must be **ignored** for PHB/class matching, but a full-byte match will still see them change | Match the 6-bit DSCP, not the 8-bit ToS/Traffic-Class byte, for classification (`3-QoS-Classification-Trust` §5.4) |
| 8 | The ECN nonce (RFC 3540) is now **Historic**, not active | Don't design around it as if it were current guidance |
| 9 | ECT(1) is no longer tied to the nonce — RFC 8311 freed it specifically for new experimentation | This is the exact codepoint L4S uses (`13-QoS-L4S-AccECN`) |
| 10 | ECN doesn't survive a tunnel's outer-to-inner header boundary automatically | Needs the explicit copy-forward mechanism covered in `14-QoS-Tunnels-Overlays` (RFC 6040) |

---

## 10. Quick Recap

| Concept | One-line answer |
|---|---|
| What ECN adds | A way to signal congestion by marking, instead of dropping, a packet |
| Where the bits live | The 2 low bits of the same DS-field byte as DSCP (RFC 3168 updates RFC 2474) |
| Four codepoints | Not-ECT (00), ECT(0) (10), ECT(1) (01), CE (11) |
| Who sets what | Sender sets ECT; only a router sets CE |
| TCP's new flags | ECE (ECN-Echo, receiver→sender) and CWR (Congestion Window Reduced, sender→receiver) |
| Negotiation | ECN-setup SYN (ECE+CWR set) → ECN-setup SYN-ACK (ECE set, CWR clear) |
| Steady-state loop | CE seen → receiver echoes ECE on every ACK → sender cuts window once → sender sets CWR once → receiver stops echoing |
| Sender's reaction to ECE | Exactly the same as reacting to a dropped packet |
| Real deployment problem | Some middleboxes broke on ECN-setup SYN packets; RFC 3168 specifies a no-ECN fallback |
| RFC 8311's main effect | Retired the ECN nonce (RFC 3540 → Historic); freed ECT(1) for new experimentation, enabling L4S |

---

## References

**Standards Track**
- RFC 3168 — The Addition of Explicit Congestion Notification (ECN) to IP (updates RFC 2474, RFC 2401, RFC 793; obsoletes RFC 2481)
- RFC 8311 — Relaxing Restrictions on Explicit Congestion Notification (ECN) Experimentation (updates RFC 3168, RFC 4341, RFC 4342, RFC 5622, RFC 6679)
- RFC 793 — Transmission Control Protocol (original definition of the Reserved field ECE/CWR occupy)

**Historic**
- RFC 3540 — Robust Explicit Congestion Notification (ECN) Signaling with Nonces (reclassified Experimental → Historic by RFC 8311)

**Referenced (background and what feeds into this note)**
- `4-QoS-Marking-Headers` (the DS field's bit layout, shared by DSCP and ECN)
- `3-QoS-Classification-Trust` (why PHB matching must ignore the ECN bits; RFC 9435 middlebox/marking-alteration parallel)
- `11-QoS-Congestion-Avoidance` (RED/CoDel/PIE as the algorithms deciding when to mark CE instead of dropping)
- `13-QoS-L4S-AccECN` (ECT(1) and the experimentation RFC 8311 enabled)
- `14-QoS-Tunnels-Overlays` (RFC 6040 — copying CE from an outer tunnel header to the inner one)
