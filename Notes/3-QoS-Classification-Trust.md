## QoS Classification and Trust

> 💡 **TL;DR:** **Classification** sorts packets into classes so that later QoS functions (marking, metering, queuing, dropping) can treat them differently. RFC 3290 models a classifier as a fan-out element driven by **filters** over a **classification key**. Two basic kinds exist: the **BA** (behavior aggregate) classifier looks **only at the DSCP, by exact match** and is what core nodes use; the **MF** (multifield) classifier combines any fields — addresses, protocol, ports, DSCP, interface, VLAN, 802.1p priority — and is what the edge uses. A classifier must be **complete** (every packet matches something, usually a default filter) and any **overlap** between filters must be settled by **precedence**. Classification is only as good as the fields it can see: fragments, encryption and long IPv6 header chains hide the ports. **Trust** is the decision whether to believe the marking a packet already carries. Anyone can set DSCP bits, so the network edge (the DS boundary) must **condition** incoming traffic: trust it, re-mark it, or ignore it and classify from scratch. Beyond your own edge, DSCPs are often **bleached or re-marked** (RFC 9435), so design for markings that may not survive.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE / ANSI-TIA · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 3290 (router model — classifiers), RFC 9435 (DSCP re-marking in practice), RFC 4594, RFC 3260, RFC 8404 — Informational · RFC 2474 (DS field), RFC 6437 (IPv6 flow label), RFC 7112 (IPv6 header chains), RFC 2205 (RSVP) — Standards Track · ANSI/TIA-1057 (LLDP-MED).

---

## 1. Where Classification Fits

RFC 3290 `[Guidance]` describes classification as the job of a **classifier element**: a 1:N (fan-out) device that takes one input stream and sorts packets into several output streams using **filters** that match packet contents *or other attributes associated with the packet*. The simplest classifier matches everything and can be omitted.

In the router model, the elements are arranged in a fixed order inside a *Traffic Conditioning Block*: **classify → meter → act (mark, drop, count) → queue/schedule**. Classification therefore *decides*; it does not change the packet. **Marking** is a separate action element that writes a codepoint, often *because of* a preceding classifier match (see `4-QoS-Marking-Headers`).

```
 IP Phone --\
 PC ---------> [ Access Switch ] ---> [ Router ] ===== WAN =====> ...
                    |                     |
         classify + mark (+ police)   classify by DSCP (BA)
         = the trust boundary         -> choose queue at the egress interface
```

### Ingress and egress

- RFC 3290 §3.2 models the same functional elements at both the **ingress** and the **egress** of an interface. The difference: **all traffic at an egress is queued**, while traffic at an ingress is queued only for shaping, if at all.
- The classification needed to select an **egress queue** does not have to run at the egress. It may run at the **ingress** and pass the result across the switching core as in-band control information. This "internal QoS label" idea is what most devices do, but its name, size and lifetime are `[Implementation-dependent]`.
- A packet's class is therefore an **internal decision** that may or may not be written into any header.

---

## 2. Classifier Types

### 2.1 BA (behavior aggregate) classifier

- Uses **only the DSCP** to pick the output stream. RFC 3290 allows **only exact match**, because assigned DSCP values have no internal structure, so no subset of bits is significant.
- RFC 2474 `[Standard-defined]` agrees: a node **MUST select the PHB by matching the entire 6-bit DSCP**, and the two remaining bits (later defined as ECN) **MUST be ignored** for PHB selection.
- Cheap and scalable → used by **core / interior nodes** (and by an edge that *trusts* the incoming marking).

### 2.2 MF (multifield) classifier

- Classifies on **one or more fields**, possibly including the DSCP. RFC 3290 calls the common form a **6-tuple**: destination address, source address, IP protocol, source port, destination port, and DSCP.
- May also use **MAC addresses, VLAN tags, link-layer traffic-class fields**, and other higher-layer fields.
- More expensive and more specific → used at the **edge**, where traffic volumes are lower and the classification is what *creates* the marking.

### 2.3 Other keys

| Key | Notes (RFC 3290 §4.1, §4.2.3–4.2.4) |
|---|---|
| **Free-form** | Arbitrary fields defined as {bit-field size, offset from the head of the packet, mask}; several can be grouped into powerful filters |
| **Data-link fields** | 802.1p priority and 802.1Q VLAN ID (the RFC's own example combines both with AND) |
| **Interface** | Ingress/egress physical or logical interface identifier, e.g., the incoming channel on a channelized interface |
| **Derived attributes** | Any attribute the router associates with the packet — RFC 3290's example is the **BGP community of the packet's best-matching route** |
| **Context** | Knowledge that an interface faces a DS domain or a legacy TOS domain can decide whether a DSCP is present at all |

### 2.4 Comparison

| | **BA** | **MF** | **Free-form** |
|---|---|---|---|
| Key | DSCP only | Combination of header fields | Arbitrary bits at an offset |
| Match type | Exact | Exact, prefix, range, masked, wildcard | Masked bit pattern |
| Typical location | Core / interior; trusted edge | Network edge | Special cases |
| Cost | Lowest | Higher | Platform-dependent |
| Depends on sender's marking? | Yes | Not necessarily | No |

---

## 3. Filters, Precedence and Completeness

Everything here comes from RFC 3290 §4.1 `[Guidance]`, which is the formal model behind vendor "match" rules.

**Filter.** A filter is a set of **conditions on components of the classification key**. It matches only if **every** condition is satisfied (an AND across fields). Each condition can be **exact, prefix, range, masked or wildcard**. A range can always be expressed as a set of prefixes, but less efficiently.

**Classifier = filters + output streams.** A class that needs an OR is expressed as **several filters** pointing to the same output.

```
              +------------+
 packets ---->| classifier |--> match Filter1 ------> Output A
              |            |--> match Filter2 ------> Output B
              |            |--> no match (default) -> Output C
              +------------+
```

**Overlap and precedence.** It is easy to write filters that both match the same packet:

| Filter | Source | Destination |
|---|---|---|
| A | 10.1.1.0/24 | any |
| B | any | 10.9.9.9/32 |

A packet from 10.1.1.5 to 10.9.9.9 matches **both**. The classification is ambiguous until a **precedence** is established. RFC 3290 says that precedence must be set either by the manager (knowing what the router can do) or by the router itself, **together with a way to report which precedence it uses**. Sequence ("inspect the DSCP only *if* the earlier tests did not match") is also expressed as precedence.

**Completeness.** An unambiguous classifier needs every possible key to match at least one filter, so a classifier normally ends with an **"everything else" wildcard filter at the lowest precedence**. Only the **first classifier a packet meets on an interface** must be complete; later classifiers in a cascade only handle traffic known to reach them.

**Evaluation model.** RFC 3290 assumes all filters of the *same* precedence are applied **simultaneously**. That defines the required *result*, not how hardware does it. How a given device orders and ties filters (top-down first match, longest match, other) is `[Implementation-dependent]`.

**Illustrative branch-edge classifier** (example subnets and classes are mine, for arithmetic and structure only):

| Precedence | Filter (all conditions must match) | Output class |
|:--:|---|---|
| 1 | source in phone subnet 10.10.20.0/24 **and** IP protocol = UDP | VOICE |
| 2 | source in video-room subnet 10.10.30.0/24 **and** DSCP = AF41 | VIDEO |
| 3 | destination in ERP prefix 10.50.0.0/16 **and** protocol = TCP | BUSINESS |
| 4 | wildcard | DEFAULT |

RFC 3290 §8.3's own example shows the same structure taken to the extreme: a classifier separating customers by **source MAC address**, with an **"everything unmatched" output that is dropped**.

---

## 4. What Can Be Matched — and Who Controls It

| Layer | Field | Who sets it | Notes |
|---|---|---|---|
| Device | Ingress interface / logical channel | **The network** | Robust: the sender cannot change which port it is plugged into |
| L2 | VLAN ID | Network (access port) or sender (tagged trunk) | Depends on how the port is configured |
| L2 | MAC addresses | Sender | Spoofable |
| L2 | **802.1Q PCP** (3 bits) | Sender | Exists **only in the 802.1Q tag**; an untagged frame has no PCP |
| MPLS | Traffic Class (3 bits) | Network | Covered in `4-QoS-Marking-Headers` |
| L3 | Source / destination address | Sender | Forgeable unless ingress filtering (BCP 38) is in place |
| L3 | IP protocol | Sender | |
| L3 | **DSCP** (6 bits) | Sender (or an upstream network) | The classic trust question (§6) |
| L3 | **ECN** bits (2 bits) | Sender / any router on the path | **Must not** be part of a DSCP match (§5.4) |
| L3 (IPv6) | **Flow label** (20 bits) | Sender (or first-hop router) | See §9 |
| L4 | Source / destination port | Sender | May be dynamic, hidden or absent (§5) |
| Derived | Routing attributes (e.g., BGP community) | **The network** | RFC 3290's example of a non-header attribute |
| Signaling | RSVP filter spec | Receiver, via RSVP | RFC 3290 notes a DiffServ router may **snoop RSVP messages to learn how to classify** without taking part in RSVP |
| L7 | Application signature | Derived by inspection | See §8 |

> 📝 **Principle** `[Common practice]`: prefer keys the **network** controls (interface, VLAN assignment, routing attributes) over keys the **sender** controls (DSCP, PCP, ports) whenever a wrong class would let someone steal service.

---

## 5. What Breaks Classification

### 5.1 Fragmentation
RFC 3290 §4.1.1 states that **MF classification of IP-fragmented packets is impossible if the filter uses transport-layer ports**, because only the first fragment carries the transport header; it therefore calls MTU discovery a prerequisite for a DiffServ network that uses port-based classifiers. RFC 6437 says the same for IPv6: the complete 5-tuple is not readily available for fragmented packets.

### 5.2 Encryption
Ports are unavailable when the transport header is encrypted (e.g., IPsec ESP). RFC 2205 §1 notes the same problem for RSVP filter specs and points to a variant that uses the **IPsec SPI** in place of the ports (RFC 2207). RFC 6437 also lists fragmentation and encryption as the reasons 5-tuple classification can fail. `[Guidance]` RFC 8404 observes that operator equipment is generally built to use the **data-link, network and transport headers**, and that increasing encryption impacts middlebox functions that go beyond that.

### 5.3 IPv6 extension-header chains
RFC 6437 notes that locating the ports **past a chain of IPv6 extension headers may be inefficient**. RFC 7112 `[Standard-defined]` (updating RFC 2460) requires that a **fragmented packet's first fragment contain the entire header chain**, up to and including the upper-layer header, so a stateless device *can* see the ports in the first fragment; a host receiving a first fragment that breaks this rule SHOULD discard it. How deep any device can parse is `[Implementation-dependent]`.

### 5.4 ECN bits inside the ToS byte
RFC 2474 says the two low bits **MUST be ignored** for PHB selection, and RFC 3168 defines them as the ECN field. A match on the **whole ToS byte** (e.g., 0xB8 for EF) silently fails for ECN-marked packets (0xB9, 0xBA, 0xBB). Match the **6-bit DSCP**.

### 5.5 Dynamic ports
Real-time media often uses ports negotiated at call setup, not fixed numbers, so static port lists go stale `[Common practice]`. Options that keep classification correct: classify on **who** is sending (interface, subnet), rely on a **trusted marking** from a managed endpoint, or learn flows from **signaling** (RSVP snooping as above).

### 5.6 Direction
Classification is **per direction** (RFC 2475: differentiated service is one-directional). The reply to a classified flow has swapped addresses and ports and is *not* automatically in the same class; the return path needs its own filters.

### 5.7 Tunnels
A node classifying an encapsulated packet sees the **outer** header. For an IPsec tunnel, RFC 6437 §6.3 notes that intermediate nodes operate on the outer header's flow label, and only the tunnel egress restores the inner one. Classifying *before* encapsulation, and how the marking is copied outward, is covered in `14-QoS-Tunnels-Overlays`.

---

## 6. Trust and the Trust Boundary

### 6.1 The problem
A DSCP is just six bits in a header. Any host, application or compromised device can set any value, including the ones that get priority. RFC 3290 §9 lists **theft of service** among the threats and suggests authenticating traffic marked for higher QoS; RFC 2475's answer is **conditioning at the boundary**.

RFC 3290 §7.2.2 gives a second reason for caution: priority is often abused. It cites networks that put business-critical traffic above routing-protocol traffic, causing the network's control plane to collapse.

### 6.2 The DS boundary is where trust is decided
`[Guidance]` (RFC 2475/2474, as covered in `2-QoS-Models`)
- A **DS boundary node** must assume incoming traffic **may not conform** to the agreement and be ready to enforce it.
- RFC 2474 §7.1: boundary nodes **MUST ensure** entering traffic carries codepoints appropriate to the domain, re-marking where necessary.
- Interior nodes normally rely on the boundary and do not check.
- A host may itself act as a boundary node; marking near the source is easier because the application's needs are known.

"Trust boundary" as a phrase is industry usage `[Common practice]`; the standards express the same idea with *DS boundary/ingress node* and *traffic conditioning*.

### 6.3 Trust models

| Model | Meaning | When it fits | What the ingress still does |
|---|---|---|---|
| **Trust DSCP** | Accept the incoming DSCP | Managed sources (servers, well-controlled endpoints); upstream domain you control | Police the accepted classes; RFC 4594 makes policing **optional for trusted sources** |
| **Trust L2 marking** | Accept the incoming 802.1p PCP and map it to a DSCP | An L2 domain you control | Map 3 bits → 6 bits (loses precision, §7.3) |
| **Untrusted** | Ignore incoming marks; classify by MF; **re-mark** | User PCs, guest devices, customers without an SLA | RFC 4594: markings from untrusted sources **SHOULD be verified by MF classification** and flows **SHOULD be policed** |
| **Conditional trust** | Trust only if the device proves to be the expected type | IP phones on a mixed access port | Accept only the DSCPs that device type may use; re-mark the rest |
| **Zero-trust re-classify** | Overwrite everything at the boundary | Provider edge, peering points | Apply the SLA/TCA; RFC 9435 documents providers doing exactly this |

**Network-control markings** are a special case. RFC 4594 (§3.2, as quoted in RFC 9435) says **CS6-marked flows from untrusted sources SHOULD be dropped or re-marked at ingress**, and (§3.1) that **CS7 SHOULD NOT be sent across peering points**.

### 6.4 Conditional trust and LLDP-MED
`[Standard-defined]` ANSI/TIA-1057 (April 2006) extends IEEE 802.1AB (LLDP) for media endpoints. Its **Network Policy** advertisement carries **VLAN ID, 802.1p priority and DSCP** per application type (voice, voice signaling, video conferencing, streaming video, and so on), so a switch can tell a phone which VLAN and marking to use. An LLDP-MED **Capabilities** TLV identifies the endpoint class (an IP phone is a Class III endpoint).

> ⚠️ **Gotcha:** LLDP-MED tells the endpoint what to mark — it does **not force** it, and the endpoint identifies **itself**. Treat it as configuration convenience; the switch must still verify or re-mark. Note also that with an **untagged** policy the Layer 2 priority is ignored and only the DSCP matters (vendor documentation), because the PCP lives in the 802.1Q tag.

### 6.5 A trust decision flow

```
 packet arrives on an access port
          |
   Is the sender a managed, identified device type? --no--> UNTRUSTED: ignore marks,
          | yes                                             classify by MF, re-mark
   Is the mark one this device type may use? --no--> re-mark (often to CS0)
          | yes
   Police what is accepted (rate + burst): excess dropped or re-marked
```

### 6.6 Where to put the boundary

```
 IP Phone --\                                         provider / peer
 PC ---------> [ Access Switch ] ---> [ Router ] ---> [ WAN edge ] ===> [ Provider ] ===> [ Peer ]
                trust boundary #1     BA classify     boundary #2:       interior:          boundary #3:
                (conditional/         on DSCP;        police, re-mark    BA classify        map or bleach
                 untrusted)           trusted         per SLA/TCA        only               per peering
                                      inside                                                agreement
```

- **As close to the source as is feasible** `[Guidance]` (RFC 2475): classification is simpler before traffic is aggregated, but more devices need the policy.
- **At every change of administrative domain**: RFC 9435 describes three kinds of operator — those that **do not re-mark** (pass DSCPs transparently), those that **condition** (enforce SLAs and may re-mark to fit their own PHBs), and those that re-mark **by accident or legacy behaviour** (misconfiguration, obsolete equipment, or lower-layer interactions).

### 6.7 Unknown or unexpected DSCPs

- **Interior node:** RFC 2474 says forward with **Default treatment and leave the DSCP unchanged**.
- **Boundary/ingress node:** RFC 3260 clarifies that this rule is meant for interior nodes; at an ingress node the **traffic conditioning of RFC 2475 applies first**, so the policy decides.
- Some networks nevertheless re-mark unrecognized codepoints to CS0 — often using an MF classifier — although the RFC advice is to leave them unchanged (RFC 9435 §4.3).

---

## 7. What Actually Happens to DSCPs on the Internet — RFC 9435

`[Guidance]` RFC 9435 (July 2023) summarises measurement studies (2017–2021, IPv4 and IPv6) that grouped observed re-marking into **seven behaviours**:

| Behaviour | What it does |
|---|---|
| **Bleach-DSCP** | Sets the whole DSCP to zero (more common at the network edge than in cores) |
| **Bleach-ToS-Precedence** | Zeroes the top 3 bits (the old IP Precedence), leaves the low 3 |
| **Bleach-some-ToS** | As above, **except** when the top 2 bits are `11` (CS6/CS7 survive) |
| **Re-mark-ToS** | Replaces the top 3 bits with a different non-zero value |
| **Bleach-low** | Zeroes the **low** 3 bits |
| **Bleach-some-low** | As above, except when the top 2 bits are `11` |
| **Re-mark-DSCP** | Re-marks all traffic to one or more particular non-zero DSCPs |

The RFC stresses that an observer **cannot tell from the changed value alone which mechanism** did it. Causes include old equipment that predates DiffServ and still acts on the IP Precedence bits, boundary conditioning, misconfiguration, and lower-layer interactions. It also reports that IPv6 routers performed all these re-markings **less** than IPv4 routers did, and that around 40 % of ICMP traffic seen at one large Internet exchange carried CS6.

### 7.1 Effect on common codepoints

Arithmetic from RFC 9435's definitions: Bleach-ToS-Precedence gives `DSCP & 0x07`; Bleach-low gives `DSCP & 0x38`.

| DSCP | Decimal | After Bleach-ToS-Precedence | After Bleach-low |
|---|--:|--:|--:|
| EF | 46 | **6** | 40 (CS5) |
| VOICE-ADMIT | 44 | 4 | 40 (CS5) |
| AF41 | 34 | **2** | 32 (CS4) |
| AF31 | 26 | **2** | 24 (CS3) |
| AF21 | 18 | **2** | 16 (CS2) |
| AF11 | 10 | **2** | 8 (CS1) |
| CS5 / CS4 / CS3 | 40 / 32 / 24 | 0 | unchanged |
| CS6 / CS7 | 48 / 56 | 0 (or unchanged under Bleach-some) | unchanged |
| LE | 1 | 1 | **0 (CS0)** — LE is promoted to default |

Two lessons: after Bleach-ToS-Precedence, **AF11/21/31/41 all collapse into DSCP 2** (as the RFC notes), and after Bleach-low **drop-precedence information disappears** (every AFxy in a class becomes the same CSx).

### 7.2 Design consequences `[Guidance / Common practice]`

- Do not assume a DSCP survives beyond your boundary. Build the design so a **degraded** marking still gets sensible treatment.
- **Re-classify at every trust boundary** rather than trusting a value that has crossed someone else's network.
- **CS6/CS7** are the values most likely to survive bleaching (which is also why they must be protected from misuse — §6.3).
- For inter-provider service classes, RFC 8100 recommends a small standard set of DSCPs and that the **ingress DSCP be restored at network egress**; unexpected DSCPs may be re-marked or bleached to zero unless the operators agree otherwise. RFC 9435 also cites GSMA guidelines under which an IPX provider that cannot trust a marking re-marks it to a **static default value**.

### 7.3 Interaction with Layer 2 (classification by PCP / UP)

`[Guidance]` (RFC 9435 §5, describing IEEE 802.1Q and 802.11)

- 802.1Q carries a **3-bit PCP**; the mapping in 802.1Q takes the **first three bits of a suitable DSCP** (EF → PCP 5). PCP 0 is default best effort and **PCP 1 is the background class**; the rest rise in priority. The older 802.1D standard used both PCP 1 and 2 as *lower than default*, and some equipment still does not treat PCP 1 as lower.
- Most Wi-Fi equipment by default maps the **first three bits of the DSCP** to the 3-bit 802.11 UP (which then maps to four WMM access categories). RFC 8325 recommends a different mapping, and notes that an AP deriving a DSCP from a UP as `UP × 8` produces the **Bleach-low** behaviour above.

```
 DSCP EF (46) = 1 0 1 1 1 0     top 3 bits = 1 0 1 = 5   -> PCP 5
 DSCP AF31 (26) = 0 1 1 0 1 0   top 3 bits = 0 1 1 = 3   -> PCP 3   (same as CS3 = 24)
 DSCP AF11 (10) = 0 0 1 0 1 0   top 3 bits = 0 0 1 = 1   -> PCP 1   (background!)
```

> ⚠️ **Gotcha (derived from the two facts above):** with the default top-3-bits mapping, **AF1x and CS1 land on PCP 1**, which IEEE 802.1Q treats as *background* (below best effort). A class you meant to be "a bit better than default" can end up **below** default on an Ethernet segment. Check what your platform does with PCP 1, and see `4-QoS-Marking-Headers` and `18-QoS-Wireless`.

---

## 8. Application Recognition (Brief)

`[Implementation-dependent]` Some devices classify by **recognising the application** (signatures, behaviour, or by consulting a signaling/policy source) rather than by header fields alone. RFC 3290 allows a classification key to include packet **contents**; whether and how a device does so is up to the vendor. Limits worth knowing:

- Encryption reduces what can be seen (§5.2); RFC 8404 notes that equipment built around the L2–L4 headers achieves good accuracy from **header information and packet sizes**, but content-dependent functions are impacted by encryption.
- Recognition is a **classification input**, not a trust mechanism: it says what the traffic *looks like*, not who is allowed to use priority.
- Recognition still has to be **combined with policing** — a recognised class that is allowed a queue must still be rate-limited, or a misclassification becomes a denial of service on the priority queue.

---

## 9. IPv6-Specific Classification Notes

`[Standard-defined]` RFC 6437 (Nov 2011, obsoletes RFC 3697):

- The **Flow Label** is 20 bits; **zero means "not labeled"**. A classifier can use the triplet {flow label, source address, destination address} to identify a flow using only **fixed-position main-header fields** — no extension-header walk, no ports.
- Sources **SHOULD** label flows (typically per 5-tuple, with values that look uniformly random). A forwarding node **MUST leave a non-zero label unchanged** (except for compelling operational security reasons). A node that receives a **zero** label **MAY** set one; that ability **MUST be configurable and disabled by default**.
- The label is **unprotected — not covered by IPsec**, even AH — so any node on the path can alter it undetectably, and it can be forged.
- **Trust rule:** networks **SHOULD NOT** make resource-allocation decisions on flow labels without some external assurance. A stateful classifier should detect and ignore suspect values; a stateless load-distribution hash must not rely on the label alone.
- The flow label is **not a QoS class**. Classes come from the **DSCP** (Traffic Class byte); the flow label identifies *which flow*.

IPv4 has no equivalent field, so IPv4 flow classification uses the 5-tuple.

---

## 10. CCIE-Depth Topics

### 10.1 Ranges cost more than prefixes
RFC 3290 says a range can be expressed as prefixes, less efficiently. Example — destination ports **1024–65535** as 16-bit prefixes:

| Prefix (16-bit) | Covers |
|---|---|
| `000001xxxxxxxxxx` | 1024–2047 |
| `00001xxxxxxxxxxx` | 2048–4095 |
| `0001xxxxxxxxxxxx` | 4096–8191 |
| `001xxxxxxxxxxxxx` | 8192–16383 |
| `01xxxxxxxxxxxxxx` | 16384–32767 |
| `1xxxxxxxxxxxxxxx` | 32768–65535 |

One range condition becomes **6** prefix entries; combined with source-address and other conditions on a masked-match (ternary) memory, the count multiplies. How a platform stores rules is `[Implementation-dependent]`, but the multiplication is arithmetic.

### 10.2 Cascaded classifiers
Classifiers can be cascaded to do "if not matched here, then inspect there". Only the **first** must be complete (§3); later ones handle only what reaches them. Precedence expresses the order.

### 10.3 Classification that is not header-based
The RFC's "derived attributes" mean policy can follow the **routing system** (e.g., a BGP community attached to the best-matching route) instead of packet fields — network-controlled and therefore trustworthy. Vendor names differ; the concept is standard-model.

### 10.4 Internal label lifetime
Since classification for an egress queue may happen at ingress (§1), a packet's class can be decided **before** a routing lookup, encapsulation, decryption or NAT-like rewrite. Whether the decision uses the header the packet arrived with or the one it leaves with is `[Implementation-dependent]` and is a common source of "policy matched but did nothing" faults.

### 10.5 Filtering vs classifying
The flow label does not remove the need for header-based **security** filtering past the IP header (RFC 6437 §6.4). A classifier that lets a class through is not a firewall.

---

## 11. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | A classifier must be **complete** and its overlaps resolved by **precedence** | An unmatched or ambiguous packet gets undefined treatment |
| 2 | **Ports vanish** in fragments and encrypted traffic | Port-based classes silently stop matching |
| 3 | Match the **6-bit DSCP**, not the whole ToS byte | ECN-marked packets miss the class |
| 4 | Sender-controlled fields (DSCP, PCP, ports, flow label) are **not proof of anything** | Priority without policing is theft of service |
| 5 | LLDP-MED **advertises** policy; it does not enforce it | The endpoint may ignore it or lie about its type |
| 6 | **Policing follows trust** | Even trusted classes must be rate-limited (RFC 4594: optional for trusted, expected for untrusted) |
| 7 | DSCPs are **bleached or re-marked** across the Internet (RFC 9435) | Never build end-to-end behaviour that needs a foreign network to preserve a DSCP |
| 8 | Bleach-ToS-Precedence maps AF11/21/31/41 → DSCP 2; Bleach-low removes drop precedence and can promote LE | Predict what a degraded mark will do |
| 9 | Default DSCP → PCP uses the top 3 bits; **PCP 1 = background** | AF1x/CS1 can end up **below** best effort at Layer 2 |
| 10 | Classification is **per direction** | The return path needs its own filters |
| 11 | Tunnels expose only the **outer** header unless you classify before encapsulating | Class decided too late or on the wrong header |
| 12 | **Flow label ≠ QoS class**, and it is unprotected | Do not allocate priority resources from it |

---

## 12. Quick Recap

| Concept | One-line answer |
|---|---|
| Classifier | Fan-out element: filters over a classification key → output streams |
| BA vs MF | BA = DSCP only, exact match (core); MF = any field combination (edge) |
| Filter conditions | Exact, prefix, range, masked, wildcard; AND across fields |
| Completeness | Every possible key matches a filter; last one is the wildcard default |
| Overlap | Resolved by precedence; the device must be able to report it |
| PHB selection | Whole 6-bit DSCP; the low 2 bits (ECN) are ignored |
| Where trust is decided | The DS boundary / ingress node |
| Untrusted sources (RFC 4594) | Verify markings by MF classification; police; drop or re-mark CS6 |
| CS7 | SHOULD NOT cross peering points |
| Conditional trust | Trust a marking only from a device identified as the expected type (LLDP-MED advertises, does not enforce) |
| DSCP survival | Not guaranteed — seven observed re-marking behaviours (RFC 9435) |
| DSCP → PCP (default) | Top 3 bits; EF → 5; AF1x/CS1 → 1 (background) |
| IPv6 flow label | 20 bits; zero = unlabeled; unprotected; identifies a flow, not a class |

---

## References

**Standards Track**
- RFC 2474 — Definition of the DS Field in the IPv4 and IPv6 Headers
- RFC 6437 — IPv6 Flow Label Specification (obsoletes RFC 3697)
- RFC 7112 — Implications of Oversized IPv6 Header Chains (updates RFC 2460)
- RFC 2205 — RSVP Version 1 Functional Specification
- RFC 3168 — The Addition of ECN to IP (defines the two low bits of the DS field)
- IEEE 802.1Q — Bridges and Bridged Networks (PCP; referenced through RFC 9435)
- IEEE 802.1AB — Station and Media Access Control Connectivity Discovery (LLDP)
- ANSI/TIA-1057 (April 2006) — LLDP for Media Endpoint Devices

**Informational / Guidance**
- RFC 3290 — An Informal Management Model for Diffserv Routers
- RFC 9435 — Considerations for Assigning a New Recommended DSCP (re-marking observations)
- RFC 4594 — Configuration Guidelines for DiffServ Service Classes
- RFC 3260 — New Terminology and Clarifications for Diffserv
- RFC 2475 — An Architecture for Differentiated Services
- RFC 8404 — Effects of Pervasive Encryption on Operators
- RFC 8100 — Diffserv-Interconnection Classes and Practice
- RFC 8325 — Mapping Diffserv to IEEE 802.11

**Vendor documentation** (used only for the untagged-policy detail in §6.4)
- Ruckus / Brocade FastIron command reference — LLDP-MED network-policy application
