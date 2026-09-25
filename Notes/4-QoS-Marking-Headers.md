## QoS Marking and Headers

> 💡 **TL;DR:** Marking writes a class into a packet or frame so that later hops can classify by a **BA (behavior aggregate)** match instead of re-inspecting every field. Four independent fields carry marking at different layers: **IP Precedence / DSCP** (IPv4 ToS byte, RFC 791 → RFC 1349 → RFC 2474), **IPv6 Traffic Class** (same 8-bit layout as the DS field, RFC 8200), **802.1Q PCP** (3 bits in the VLAN tag, IEEE 802.1Q, informally "802.1p"), and **MPLS Traffic Class** (3 bits in the label stack entry, RFC 3032/3270, renamed from EXP by RFC 5462). None of these fields talk to each other automatically — a device must be explicitly configured to **map** DSCP → PCP → MPLS TC (and back) at every layer boundary, and the default mappings some platforms use are not always the ones a standard recommends (RFC 8325's EF example is the classic case). ECN occupies the 2 low bits of the same byte as DSCP and is covered in `12-QoS-ECN`.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE / ANSI-TIA · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 791, RFC 1349, RFC 2474, RFC 3032, RFC 3270, RFC 5462, RFC 8200, RFC 8325 — Standards Track · RFC 3260, RFC 4594 — Informational · IEEE 802.1Q — IEEE standard.

---

## 1. Marking vs Classification — Recap

Classification (note 3) **decides** the class; marking **writes it down** so the decision survives to the next hop without being re-computed. RFC 2475's traffic-conditioning block puts the **marker** right after the meter: classify → meter → **mark** → shape/drop. Marking only pays off if the **next** node reads the same field the way you wrote it — which is why this note is mostly about keeping several independent fields consistent with each other.

```
 Endpoint ---> Access Switch ---> Router ===WAN===> Router ---> Switch ---> Endpoint
                    |                  |
              PCP (802.1Q)        DSCP (IP header)
              -- Layer 2 only --  -- survives across L3 hops, lost if
                                     re-encapsulated at L2 without mapping
```

---

## 2. IPv4: From IP Precedence to DSCP

### 2.1 RFC 791 (1981) — the original TOS octet

`[Standard-defined]` RFC 791 §3.1 defines an 8-bit **Type of Service** octet with three unrelated ideas packed into one byte: a 3-bit **Precedence**, then independent 1-bit flags for low delay, high throughput and high reliability, with the last 2 bits reserved.

```
  bit:    0   1   2   3   4   5   6   7
        +---+---+---+---+---+---+---+---+
        |  PRECEDENCE   | D | T | R | 0 | 0 |     (RFC 791, 1981)
        +---+---+---+---+---+---+---+---+
```

| Precedence (binary) | Decimal | Name |
|:--:|:--:|---|
| 111 | 7 | Network Control |
| 110 | 6 | Internetwork Control |
| 101 | 5 | CRITIC/ECP |
| 100 | 4 | Flash Override |
| 011 | 3 | Flash |
| 010 | 2 | Immediate |
| 001 | 1 | Priority |
| 000 | 0 | Routine |

RFC 791 itself calls this "an independent measure of the importance of this datagram," and D/T/R as a three-way trade-off — at most two should normally be set. RFC 795 (the same day, 1981) mapped these onto specific 1981-era network technologies (ARPANET's single priority bit, etc.), confirming Precedence was meant for **inter-network** treatment, not just IP.

> 📝 In practice the D/T/R bits and Precedence values above CS0 were essentially unused for the Internet's first 15 years — the "why" is architectural, not technical, and is covered further in `2-QoS-Models`.

### 2.2 RFC 1349 (1992) — reinterpreting the low bits

`[Standard-defined, since superseded]` RFC 1349 kept the same 3 Precedence bits but redefined the delay/throughput/reliability idea as a **single 4-bit enumerated TOS value** (only one of a fixed set of values, not independent flags) and added a **low-cost** bit. This is why some older material shows a 4th TOS bit — it existed only under RFC 1349, and RFC 1349 is now obsolete.

### 2.3 RFC 2474 (1998) — DSCP replaces both

As established in `2-QoS-Models`, RFC 2474 **obsoletes RFC 1349** and redefines the whole octet as a 6-bit **DSCP** plus 2 bits later assigned to **ECN** (RFC 3168):

```
  bit:    0   1   2   3   4   5   6   7
        +---+---+---+---+---+---+---+---+
        |      DSCP (6 bits)    |  ECN  |     (RFC 2474 + RFC 3168)
        +---+---+---+---+---+---+---+---+
        |<-- old Precedence -->|
```

Two consequences worth stating plainly:

- **DSCP is not "Precedence with 3 more bits bolted on."** The DSCP is matched as a **whole 6-bit value**; there is no rule that says "the top 3 bits still mean what RFC 791 said." RFC 2474 §4.2.1 only requires the **Class Selector** codepoints (`xxx000`) to give ordering *at least as good as* legacy Precedence-based forwarding, for backward compatibility, not that every AF/EF value respects Precedence ordering.
- **The field is the same 8 bits it always was.** A device reading "IP Precedence" against a DSCP-marked packet is reading the **top 3 bits of the DSCP**, which is why AF11 (binary `001010`) reads as Precedence 1 and AF41 (`100010`) reads as Precedence 4 — coincidentally aligned by design (RFC 2474 built AF that way on purpose), but **CS3 (`011000` = 24) and AF31 (`011010` = 26) both read as Precedence 3**, because bits 3–4 (the AF drop-precedence bits) are outside Precedence's 3-bit window.

### 2.4 Binary → decimal conversion (cross-reference)

The full DSCP name/binary/decimal table (CS, AF, EF, LE, VOICE-ADMIT) is in `5-QoS-PHB-DSCP-Values`. The arithmetic rule needed here: **DSCP decimal × 4 = ToS byte decimal** (DSCP occupies the top 6 bits, so shifting left 2 bits multiplies by 4). Example: EF = 46 → ToS byte = 184 = `0xB8`.

---

## 3. IPv6: The Traffic Class Field

`[Standard-defined]` RFC 8200 §3 (IPv6 spec, obsoletes RFC 2460) defines an **8-bit Traffic Class** field with exactly the layout above: it plays the same role as the IPv4 DS field, and RFC 2474 explicitly defines the DS field for **both** IPv4 and IPv6 in one document.

```
   0   1   2   3   4   5   6   7                     0                   1                   2                   3
 +---+---+---+---+---+---+---+---+       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 |    DSCP (6 bits)      |ECN|          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 +---+---+---+---+---+---+---+---+      |Version| Traffic Class |           Flow Label                 |
                                         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Key points:

- RFC 8200 §3 says explicitly: **"The value of the Traffic Class bits in a received packet or fragment might be different from the value sent"** — the field can legitimately be rewritten in transit (that is exactly what DSCP re-marking, and ECN marking, does).
- The **Flow Label** (20 bits, immediately after Traffic Class) is a *different* mechanism — it identifies a flow, not a class. Do not confuse the two; see `3-QoS-Classification-Trust` §9 for the flow label's rules (RFC 6437) and its trust limitations.
- Because IPv4 and IPv6 share the identical DSCP/ECN layout and definition (RFC 2474), the **PHB tables in `5-QoS-PHB-DSCP-Values` apply unchanged to IPv6**. There is no separate IPv6 DSCP registry.

> ⚠️ **Gotcha:** IPv4 has no field equivalent to the Flow Label. Any classification design that relies on the Flow Label works only in an IPv6-only or dual-stack-aware design (see `3-QoS-Classification-Trust` §9 on why it should not carry class information anyway).

---

## 4. Layer 2: 802.1Q PCP and DEI

`[Standard-defined]` IEEE 802.1Q defines a 4-byte VLAN tag inserted between the source MAC address and the EtherType/length field:

```
 Ethernet frame (tagged):
 +-------------+-------------+------+-----+-----+--------------------+
 | Dest MAC    | Src MAC     | TPID | TCI       | EtherType/Length … |
 | (6 bytes)   | (6 bytes)   |0x8100| (2 bytes) |                    |
 +-------------+-------------+------+-----+-----+--------------------+
                                      \___/\_/\_________/
                                     PCP(3) DEI VID(12)
                                          (1)
```

- **PCP (Priority Code Point)** — 3 bits, 8 values, **0 = lowest priority normally used, 7 = highest**, informally called "802.1p" (802.1p was folded into 802.1Q as Annex G rather than remaining a separate standard).
- **DEI (Drop Eligible Indicator)** — 1 bit (formerly CFI). Marks a frame as a candidate for discard first under congestion — the Ethernet-frame equivalent of an AF drop-precedence sub-bit.
- **VID (VLAN ID)** — 12 bits, the VLAN the frame belongs to. VID `0x000` with a real PCP is called a **priority-tagged frame**: the tag exists purely to carry priority, and the frame is treated as belonging to the port's native VLAN.

**Existence condition:** PCP and DEI live **only inside the 802.1Q tag**. An **untagged frame carries no PCP at all** — there is nothing to match. This is why, as noted in `3-QoS-Classification-Trust` §6.4, an untagged voice VLAN policy makes the Layer-2 priority field meaningless and only DSCP matters.

### 4.1 The traffic-type table (IEEE 802.1D Annex G, carried into 802.1Q)

`[Standard-defined]` The eight PCP values map to named traffic types, consistently documented across implementations:

| PCP | Traffic type | Typical use |
|:--:|---|---|
| 1 | Background | Bulk transfers the user is willing to wait for |
| 0 | Best Effort (default) | Default, untagged traffic |
| 2 | Excellent Effort | Important business traffic |
| 3 | Critical Applications | |
| 4 | Video | < 100 ms latency and jitter |
| 5 | Voice | < 10 ms latency and jitter |
| 6 | Internetwork Control | |
| 7 | Network Control | Highest priority — reserved for network control traffic, not user data |

> ⚠️ **Gotcha (verified in note 3, repeated here because it is a marking mistake, not just a mapping one):** **PCP 0 (Best Effort) ranks above PCP 1 (Background)** — the order is **not** simply numeric. Marking something "PCP 1, a little better than default" actually places it **below** default. This is the same trap as the DSCP→PCP default mapping discussed next.

### 4.2 DSCP ↔ PCP mapping is a local decision

There is **no IETF or IEEE standard that fixes** DSCP-to-PCP mapping — it is a per-device configuration. Two mappings are common in practice `[Common practice / Implementation-dependent]`:

**Default (top-3-bits) mapping** — take the DSCP's top 3 bits directly as PCP:

```
 DSCP EF   (46) = 101110   top 3 bits = 101 = 5   -> PCP 5 (Video, not Voice!)
 DSCP AF41 (34) = 100010   top 3 bits = 100 = 4   -> PCP 4 (Video)
 DSCP AF31 (26) = 011010   top 3 bits = 011 = 3   -> PCP 3
 DSCP AF11 (10) = 001010   top 3 bits = 001 = 1   -> PCP 1 (Background!)
 DSCP CS1  (8)  = 001000   top 3 bits = 001 = 1   -> PCP 1 (Background)
```

RFC 8325 §4.2.1 documents exactly this failure mode for the wireless case (see §5 below): EF maps by default to UP/PCP **5**, landing in the **Video** class, not Voice — the DSCP-to-UP default mapping problem is a general instance of the same top-3-bits arithmetic, and it recurs at the wired 802.1Q boundary too. Likewise AF1x and CS1 land on PCP 1 = Background, confirming the §4.1 gotcha.

**Reverse (PCP → DSCP) mapping**, when a device must derive a DSCP from an incoming PCP, is often implemented as **PCP × 8** (placing PCP in the DSCP's top 3 bits, zero-filling the bottom 3) — this is exactly the "Bleach-low" pattern documented in RFC 9435 and referenced in note 3 §7.3: it **loses AF drop-precedence information**, because every AFxN in a class collapses onto the same CSx.

### 4.3 Practical implication

`[Common practice]` Because the default mapping can misplace EF and AF1x, a marking design that spans an L2 segment must **explicitly verify or override** the DSCP↔PCP table on the device that adds or reads the 802.1Q tag, rather than trust the factory default. This is the same principle as `3-QoS-Classification-Trust` §7.3's gotcha, restated from the marking side: get the mapping right at the point where the tag is written, not just where it is read.

---

## 5. Wireless: 802.11 User Priority — the Standard Fix (RFC 8325)

`[Standard-defined]` RFC 8325 (Feb 2018) exists **specifically because** the default DSCP→UP mapping described in §4.2 misplaces real-time traffic on Wi-Fi. IEEE 802.11 reuses the same 3-bit **User Priority (UP)** concept as 802.1Q PCP, mapped in turn to four **Access Categories** (AC_VO, AC_VI, AC_BE, AC_BK — detailed in `18-QoS-Wireless`).

**The problem, in RFC 8325's own words:** EF DSCP maps **by default** to UP 5, which lands in **AC_VI (Video)**, not **AC_VO (Voice)**, "for which it is intended."

**The fix RFC 8325 recommends:**

| Service class | DSCP | Default UP (top-3-bits) | RFC 8325 recommended UP | Resulting AC |
|---|:--:|:--:|:--:|---|
| Telephony | EF (46) | 5 | **6** | AC_VO |
| VOICE-ADMIT (RFC 5865) | 44 | 5 | **6** | AC_VO |
| Low-Priority Data | CS1 (8) | 1 | **1** (kept) | AC_BK |
| Standard / Default | DF (0) | 0 | 0 | AC_BE |

RFC 8325 also states: **all unused/unrecognized codepoints are RECOMMENDED to map to UP 0** (a security-motivated default — an unrecognized mark should not be able to claim a high-priority queue). This is the wireless-specific version of the "unknown DSCP → Default" rule from `2-QoS-Models`/`3-QoS-Classification-Trust`.

> 📝 The full RFC 4594-to-802.11 table (all 12 service classes) belongs in `18-QoS-Wireless`; the point to take from here is the **general lesson**: whenever a marking crosses from a 6-bit space (DSCP) to a 3-bit space (PCP/UP/MPLS TC), a **naive truncation loses information and can misroute traffic into the wrong class**, and standards bodies sometimes have to publish an explicit override table (RFC 8325) to fix exactly this.

---

## 6. MPLS: The Traffic Class (TC) Field

### 6.1 Definition and history

`[Standard-defined]` RFC 3032 (2001) defined the 4-byte **MPLS label stack entry**:

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |          Label (20 bits)             |  TC   |S|   TTL (8)     |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Label** — 20 bits, the forwarding identifier (swapped hop by hop).
- **TC** — 3 bits. Originally named the **"EXP" (Experimental Use) field**, because at the time RFC 3032 was published its exact use was not agreed. RFC 3270 (2002) then defined its use for carrying Diffserv information, and RFC 5129 (2008) extended it for ECN. RFC 5462 (2009) **formally renamed** the field to **"Traffic Class"** to reflect this settled usage, and updated RFC 3032, RFC 3270, RFC 5129 and several other RFCs accordingly — the field's bit position and size never changed, only its name and the fact that it is no longer "just experimental."
- **S** — 1 bit, bottom-of-stack indicator (for stacked labels, e.g., MPLS VPN or traffic engineering).
- **TTL** — 8 bits, decremented per hop like IP TTL.

RFC 5462 states the general relationship directly: **the MPLS TC field relates to an MPLS-encapsulated packet the way the IPv6 Traffic Class field relates to an IPv6 packet, or the IPv4 Precedence field relates to an IPv4 packet** — i.e., it is MPLS's own, independent, 3-bit marking, not something that is automatically derived from the IP header underneath.

### 6.2 E-LSP vs L-LSP

`[Standard-defined]` (RFC 3270, terminology corrected by RFC 5462) There are two ways an LSR decides the PHB (including drop precedence) for a labeled packet:

| Type | How the PHB is determined | TC field's role |
|---|---|---|
| **E-LSP** (Explicitly TC-encoded-PSC LSP) | The **TC field value itself** determines the PHB Scheduling Class (PSC) **and** the drop precedence, using either a signaled or a pre-configured TC→PHB mapping | TC field carries the full class information; **one LSP can carry up to 8 behavior aggregates** (since TC is 3 bits) |
| **L-LSP** (Label-only-inferred-PSC LSP) | The **label itself** determines the PSC (one LSP = one PSC, signaled at setup); the TC field carries **only the drop precedence** within that PSC | TC field's meaning is narrower — just drop precedence |

`[Common practice]` E-LSP is more common in practice because it needs fewer LSPs (one LSP can multiplex several classes via its TC values), at the cost of a slightly less strict per-class bandwidth guarantee than a dedicated L-LSP.

### 6.3 Where the TC value comes from

Since MPLS TC is only 3 bits, a common `[Common practice]` design copies the **top 3 bits of the IP DSCP** into TC at the point where the label is imposed — the same truncation problem as §4.2, with the same information loss for AF drop precedence (all AFx1/x2/x3 collapse to the same TC value unless the device does something more deliberate, such as an explicit DSCP→TC table). Exactly how a given platform derives or restores the TC/DSCP relationship at label imposition and disposition is `[Implementation-dependent]`; the general uniform-vs-pipe-mode question for how markings survive tunneling/labeling is covered fully in `14-QoS-Tunnels-Overlays`.

---

## 7. Putting the Four Fields Together

```
 Endpoint (App) --> NIC --> Access Switch --> Router (PE) --> MPLS Core --> Router (PE) --> Switch --> Endpoint
      |                       |                   |               |
   app may set          802.1Q PCP           IP DSCP marked    MPLS TC set from
   DSCP via socket     (if VLAN tagged)      or verified at    DSCP at label
   option                                    the trust         imposition (E-LSP)
                                              boundary
```

| Field | Size | Where it lives | Standard | Survives a re-encapsulation at that layer? |
|---|:--:|---|---|---|
| IP Precedence / DSCP | 6 bits (of 8) | IPv4 ToS byte / IPv6 Traffic Class | RFC 2474 | Yes, as long as the IP header itself is preserved (e.g., through a GRE/IPsec tunnel's **inner** header) |
| ECN | 2 bits (of 8) | Same byte as DSCP | RFC 3168 | Same as DSCP; see `12-QoS-ECN` |
| 802.1Q PCP / DEI | 3+1 bits | VLAN tag | IEEE 802.1Q | No — lost the moment the frame is untagged or the L2 segment ends; must be re-derived at the next L2 hop |
| MPLS TC | 3 bits | Label stack entry | RFC 3032/3270/5462 | No — lost when the label is popped; must be re-derived (from DSCP or configuration) if further MPLS or L2 marking is needed downstream |
| IPv6 Flow Label | 20 bits | IPv6 main header | RFC 6437 | Not a class field — see §3 |

**The single most important operational fact in this note:** only the **DSCP/Traffic-Class byte survives across router hops by default**, because it lives in the IP header that routers forward unchanged (aside from deliberate re-marking). **PCP and MPLS TC are local to one layer/segment** and must be **explicitly re-derived** every time a packet crosses into or out of that layer. A design that marks DSCP once "at the edge" and assumes PCP/TC will automatically follow is assuming a mapping that no standard guarantees.

---

## 8. CCIE-Depth Topics

### 8.1 Why AF31 and CS3 are indistinguishable to old Precedence-only equipment

From §2.3: AF31 = `011010` = 26, CS3 = `011000` = 24. Both have top-3-bits = `011` = Precedence 3. Any device that classifies on "IP Precedence" (i.e., only the top 3 bits, ignoring bits 3–5) **cannot tell AF31 from CS3**, even though DiffServ intends them as different classes with different PHBs (Assured Forwarding vs. Class Selector). This is a direct consequence of RFC 2474's design choice to make Class Selector backward-compatible with Precedence ordering (§2.3) without making that compatibility exact for AF.

### 8.2 The "collapse" arithmetic in one place

Restating §4.2/§6.3 as a single generalizable rule: whenever an *n*-bit field (n < 6) is derived from the 6-bit DSCP by taking the **top n bits**, every DSCP value that shares those top n bits collapses to the same derived value. For n = 3 (PCP, UP, MPLS TC via top-3-bits): the collapse groups are exactly the **Class Selector families** — {CS3, AF31, AF32, AF33} all → 3; {CS4, AF41, AF42, AF43} all → 4; and so on. This is *why* AF's drop-precedence sub-bits (bits 3–4 of the DSCP) are the first casualty of any 3-bit derived marking.

### 8.3 Signaled vs pre-configured E-LSP mapping

RFC 3270 allows an E-LSP's TC→PHB mapping to be **either** signaled at label setup (e.g., via LDP/RSVP-TE extensions) **or** relying on a pre-configured, well-known mapping shared by convention across the domain. A domain that mixes routers using the well-known mapping with routers expecting a signaled one will silently apply the wrong PHB to a given TC value — this is an interoperability class of fault distinct from ordinary mis-marking.

### 8.4 Priority-tagged frames and VID 0

A priority-tagged frame (VID = `0x000`, real PCP) is a **legitimate use of the 802.1Q tag purely to carry a priority marking** without VLAN segmentation — worth knowing because it looks unusual on a packet capture (VLAN 0 with a non-zero PCP) but is standard-defined behaviour, not a fault.

---

## 9. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | DSCP is matched as a whole 6-bit value; it is **not** "Precedence + extra bits" | AF31 (26) and CS3 (24) both read as Precedence 3 to old equipment |
| 2 | RFC 8200 explicitly allows the Traffic Class value to **change in transit** | Do not assume what you sent is what arrives |
| 3 | PCP exists **only inside an 802.1Q tag** | Untagged frames have no Layer-2 priority to match |
| 4 | **PCP 0 (Best Effort) outranks PCP 1 (Background)** | Numeric order ≠ priority order |
| 5 | Default DSCP→PCP/UP mapping takes the **top 3 bits** | EF → PCP/UP 5 (Video), not 6 (Voice); AF1x/CS1 → PCP/UP 1 (Background) |
| 6 | RFC 8325 exists specifically to override the Wi-Fi version of gotcha #5 | Recommended: EF and VOICE-ADMIT → UP 6, not the default 5 |
| 7 | MPLS TC was called "EXP" and was officially "experimental" until RFC 5462 (2009) renamed it | Old documentation calling it EXP is not wrong, just superseded terminology |
| 8 | Deriving PCP or MPLS TC from DSCP's top 3 bits **loses AF drop precedence** | All of AFx1/x2/x3 in one class collapse to the same derived value |
| 9 | PCP and MPLS TC do **not** survive across the layer/segment they belong to | Must be explicitly re-derived at every L2 segment or MPLS label boundary; only DSCP naturally survives router hops |
| 10 | IPv6 Flow Label is not a class field, and it sits right next to Traffic Class in the header | Do not conflate flow identification with QoS marking |

---

## 10. Quick Recap

| Concept | One-line answer |
|---|---|
| Four independent marking fields | IP DSCP/Precedence, IPv6 Traffic Class (same layout), 802.1Q PCP, MPLS TC |
| Original IPv4 TOS octet (RFC 791) | 3-bit Precedence + independent D/T/R flags |
| RFC 1349 | Redefined D/T/R as one 4-bit enumerated TOS value (now obsolete) |
| RFC 2474 | Replaced both with the 6-bit DSCP (+ 2 bits later given to ECN) |
| DSCP → ToS byte decimal | DSCP × 4 |
| IPv6 equivalent of the DS field | Traffic Class, 8 bits, identical layout, defined in the same RFC 2474 |
| 802.1Q priority field | PCP, 3 bits, inside the VLAN tag only; 0 = best effort, 1 = background |
| Default DSCP→PCP/UP mapping | Top 3 bits of the DSCP |
| RFC 8325's fix | EF/VOICE-ADMIT → UP 6 (not the default 5) for correct Wi-Fi AC |
| MPLS marking field | TC, 3 bits in the label stack entry; renamed from EXP by RFC 5462 |
| E-LSP vs L-LSP | TC determines full PHB vs. TC determines only drop precedence within a label-determined PSC |
| What survives across hops by default | Only DSCP/Traffic Class (IP header); PCP and MPLS TC are layer-local |

---

## References

**Standards Track**
- RFC 791 — Internet Protocol (original TOS octet, IP Precedence)
- RFC 1349 — Type of Service in the Internet Protocol Suite (obsoleted by RFC 2474)
- RFC 2474 — Definition of the DS Field in the IPv4 and IPv6 Headers
- RFC 3032 — MPLS Label Stack Encoding (defines the label stack entry, originally "EXP")
- RFC 3270 — MPLS Support of Differentiated Services (E-LSP / L-LSP)
- RFC 5462 — MPLS Label Stack Entry: "EXP" Field Renamed to "Traffic Class" Field (updates RFC 3032, 3270, 5129 and others)
- RFC 8200 — Internet Protocol, Version 6 (IPv6) Specification (Traffic Class field; obsoletes RFC 2460)
- RFC 8325 — Mapping Diffserv to IEEE 802.11 (DSCP → User Priority recommendations)
- RFC 5865 — A Differentiated Services Code Point (DSCP) for Capacity-Admitted Traffic (VOICE-ADMIT, 44)
- RFC 6437 — IPv6 Flow Label Specification
- IEEE 802.1Q — Bridges and Bridged Networks (VLAN tag, PCP, DEI; Annex G traffic-type table)

**Informational / Guidance**
- RFC 3260 — New Terminology and Clarifications for Diffserv
- RFC 4594 — Configuration Guidelines for DiffServ Service Classes
- RFC 795 — Service Mappings (historical context for RFC 791's Precedence)
- RFC 9435 — Considerations for Assigning a New Recommended DSCP (re-marking observations, referenced for the PCP×8 "Bleach-low" pattern)
