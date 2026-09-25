## PHB and DSCP Values — Binary to Decimal

> 💡 **TL;DR:** Five standards-defined PHBs use the DSCP space: **Default (DF/CS0)** = `000000` = 0; **Class Selector (CSn)** = `nnn000` = n × 8; **Assured Forwarding (AFxy)** = `xxxyy0` = (x × 8) + (y × 2), four classes × three drop precedences; **Expedited Forwarding (EF)** = `101110` = 46; **VOICE-ADMIT** = `101100` = 44 (EF's sibling, requiring capacity admission); **Lower Effort (LE)** = `000001` = 1. Every value here comes from an IANA-registered codepoint backed by a Standards Track RFC — nothing in this note is a vendor default.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IANA registry · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 2474 (DF, CS, pools), RFC 2597 (AF), RFC 3246 (EF, obsoletes RFC 2598), RFC 5865 (VOICE-ADMIT), RFC 8622 (LE, obsoletes RFC 3662) — all Standards Track.

---

## 1. The Conversion Method (do this once, reuse everywhere)

The DSCP is the **top 6 bits** of the 8-bit DS field (`3-QoS-Classification-Trust` calls this the whole-byte-vs-6-bit-match distinction; `4-QoS-Marking-Headers` §2 has the byte layout). To convert any 6-bit pattern to decimal, sum the place values of the bits that are `1`:

```
 bit position:   0    1    2    3    4    5
 place value:   32   16    8    4    2    1     (reading left to right, MSB first)
```

**Worked example — EF = `101110`:**

```
 bit:      1    0    1    1    1    0
 value:   32    0    8    4    2    0
```

**32 + 8 + 4 + 2 = 46 → EF = decimal 46**

Two numbers derived from any DSCP, useful throughout this series:

- **ToS/Traffic-Class byte value = DSCP × 4** (the DSCP occupies the top 6 of 8 bits, i.e., shifted left 2 places). EF: 46 × 4 = **184 = `0xB8`**.
- **Legacy IP-Precedence reading = top 3 bits of the DSCP**, as decimal 0–7 (see `4-QoS-Marking-Headers` §2.3, §8.1 for why this is not always meaningful for AF).

---

## 2. Default / Best Effort (DF, CS0)

`[Standard-defined]` (RFC 2474 §4.1) The Default PHB. Recommended codepoint `000000`. A DS-compliant node **MUST** provide it, and it is the fallback for any **unrecognized** codepoint.

| Name | Binary | Decimal | ToS byte |
|---|:--:|:--:|:--:|
| DF / CS0 | `000000` | **0** | 0 |

---

## 3. Class Selector (CS0–CS7)

`[Standard-defined]` (RFC 2474 §4.2) Backward-compatible with the legacy IP Precedence bits (RFC 791, see `4-QoS-Marking-Headers` §2). Pattern: **`nnn000`**, where `nnn` is the old 3-bit precedence value.

**Formula: CSn decimal = n × 8**

```
 CS3 = 011 000
        |     \
     n=3        always 000
 decimal = 3 × 8 = 24
```

| Name | Binary | n × 8 | Decimal |
|---|:--:|:--:|:--:|
| CS0 (= DF) | `000000` | 0 × 8 | 0 |
| CS1 | `001000` | 1 × 8 | 8 |
| CS2 | `010000` | 2 × 8 | 16 |
| CS3 | `011000` | 3 × 8 | 24 |
| CS4 | `100000` | 4 × 8 | 32 |
| CS5 | `101000` | 5 × 8 | 40 |
| CS6 | `110000` | 6 × 8 | 48 |
| CS7 | `111000` | 7 × 8 | 56 |

RFC 2474 §4.2.1 requires the CS PHBs to give at least two independently forwarded classes and to give `11x000` (CS6, CS7) preferential treatment relative to `000000` — the historical reservation of Precedence 110/111 for routing/control traffic (background in `2-QoS-Models` and `3-QoS-Classification-Trust` §6.3 on CS6/CS7 trust handling).

---

## 4. Assured Forwarding (AFxy)

`[Standard-defined]` (RFC 2597) Four **independent** classes (x = 1–4), each with three **drop precedences** (y = 1 low, 2 medium, 3 high). RFC 2597 §2: classes **MUST NOT** be aggregated together — a node must forward AF1x independently of AF2x, etc. Within a class, higher drop precedence means the packet is discarded **preferentially** under congestion; RFC 2597 §2 also requires that a node **not reorder packets of the same microflow**, whether they are in-profile or out-of-profile.

### 4.1 The bit pattern and formula

Pattern: **`xxx yy 0`** — the AF class occupies the same top-3-bit position as Class Selector, the drop precedence occupies bits 3–4, and bit 5 is always 0.

```
 AF41 = 100 01 0
         \   \  \
        x=4  y=1  (bit 5 always 0)
 decimal = (x × 8) + (y × 2) = 32 + 2 = 34
```

**Formula: AFxy decimal = (x × 8) + (y × 2)**

### 4.2 Full table (RFC 2597 §6, exact recommended values)

| | Class 1 (x=1) | Class 2 (x=2) | Class 3 (x=3) | Class 4 (x=4) |
|---|:--:|:--:|:--:|:--:|
| **Low drop (y=1)** | AF11 = `001010` = **10** | AF21 = `010010` = **18** | AF31 = `011010` = **26** | AF41 = `100010` = **34** |
| **Medium drop (y=2)** | AF12 = `001100` = **12** | AF22 = `010100` = **20** | AF32 = `011100` = **28** | AF42 = `100100` = **36** |
| **High drop (y=3)** | AF13 = `001110` = **14** | AF23 = `010110` = **22** | AF33 = `011110` = **30** | AF43 = `100110` = **38** |

Worked check for two more cells:
- **AF11** = `001010` = (1×8) + (1×2) = **10**. Binary sum check: bits set are position 2 (value 8) and position 4 (value 2) → 8+2 = 10. ✓
- **AF33** = `011110` = (3×8) + (3×2) = 24+6 = **30**. Binary sum check: `011110` = 16+8+4+2 = **30**. ✓

> 📝 Note the pattern **within a row**: each class is exactly **+8** from the previous (AF11=10, AF21=18, AF31=26, AF41=34 — all +8). Within a column, each drop precedence is **+2** from the previous (AF11=10, AF12=12, AF13=14). This is a fast way to reconstruct the whole table from memory during an exam: start at AF11=10, add 8 across, add 2 down.

### 4.3 AF and Class Selector relationship (the gotcha from note 4, restated)

Every AFx1/x2/x3 shares its top-3-bits with CSx: AF31/32/33 (26/28/30) all read as Precedence/CS **3** (24) at the top-3-bit level. Full explanation and the collapse table under bleaching is in `4-QoS-Marking-Headers` §8.1–8.2 and `3-QoS-Classification-Trust` §7.1.

---

## 5. Expedited Forwarding (EF)

`[Standard-defined]` (RFC 3246, obsoletes RFC 2598) A single codepoint intended for traffic needing **low delay, low jitter and low loss**, served at a configured rate regardless of the offered load of other traffic. RFC 3246 §4 states IANA allocated **one codepoint, `101110`, in Pool 1**.

| Name | Binary | Decimal | ToS byte |
|---|:--:|:--:|:--:|
| EF | `101110` | **46** | 184 (`0xB8`) |

**Conversion, shown fully:** `101110` → bits set at positions 0, 2, 3, 4 → 32 + 8 + 4 + 2 = **46**.

RFC 2598 §2.4 (the original EF definition, carried forward in spirit by RFC 3246) allows EF-marked packets to be **re-marked at a DS domain boundary only to other codepoints that also satisfy the EF PHB** — i.e., EF's meaning must be preserved even if its exact bit pattern changes at a boundary; it is never demoted silently.

---

## 6. VOICE-ADMIT

`[Standard-defined]` (RFC 5865, updates RFC 4542 and RFC 4594) A **second EF-conformant codepoint**, deliberately chosen to be close to EF's bit pattern: RFC 5865 explains IANA assigned `101100` specifically to **keep the first 4 (left-to-right) binary digits the same as EF's** `101110`.

```
 EF          = 1 0 1 1 1 0  = 46
 VOICE-ADMIT = 1 0 1 1 0 0  = 44
                     ^
              only this bit differs
```

**Conversion:** `101100` → bits at positions 0, 2, 3 → 32 + 8 + 4 = **44**.

| Name | Binary | Decimal | Distinguishing requirement |
|---|:--:|:--:|---|
| EF | `101110` | 46 | Conforms to the EF PHB |
| VOICE-ADMIT | `101100` | 44 | Conforms to the EF PHB **and REQUIRES capacity admission** (e.g., RSVP + AAA) at the User/Network Interface |

RFC 5865 §1 states the traffic is "admitted by the network using a Call Admission Control (CAC) procedure involving authentication, authorization, and capacity admission" — this is what separates it from plain EF, which has no such requirement. RFC 5865 also recommends that certain RFC 4594 video classes (Interactive Real-Time, Broadcast TV used for video-on-demand, Multimedia Conferencing) be treated as requiring the same kind of capacity admission, though it does not assign them a separate codepoint — they still use their own DSCPs from `6-QoS-Class-Design`.

---

## 7. Lower Effort (LE)

`[Standard-defined]` (RFC 8622, obsoletes RFC 3662, updates RFC 4594 and RFC 8325) A PHB intended for traffic that should get **less-than-best-effort** treatment — traffic the network is willing to delay or drop first, in exchange for not degrading everything else. RFC 8622 replaces RFC 3662's earlier recommendation that low-priority data use **CS1**; LE has its own dedicated codepoint instead.

| Name | Binary | Decimal | Pool |
|---|:--:|:--:|:--:|
| LE | `000001` | **1** | Pool 3 (`xxxx01`) |

**Conversion:** `000001` → only bit 5 (value 1) is set → **decimal 1**.

> ⚠️ **Gotcha (cross-referenced from note 3 §7.1):** under the **Bleach-ToS-Precedence** re-marking behaviour documented in RFC 9435 (`DSCP & 0x07`, zeroing the top 3 bits), LE's value of 1 **survives unchanged** (1 & 7 = 1), but under **Bleach-low** (`DSCP & 0x38`, zeroing the bottom 3 bits) LE is **wiped to 0**, silently promoting "lower effort" traffic to plain Default. This is a real, standards-documented failure mode, not a hypothetical.

---

## 8. All Codepoints, One Table

| PHB | Binary | Decimal | Pool | Defining RFC |
|---|:--:|:--:|:--:|---|
| DF / CS0 | `000000` | 0 | 1 | RFC 2474 |
| LE | `000001` | 1 | 3 | RFC 8622 |
| CS1 | `001000` | 8 | 1 | RFC 2474 |
| AF11 | `001010` | 10 | 1 | RFC 2597 |
| AF12 | `001100` | 12 | 1 | RFC 2597 |
| AF13 | `001110` | 14 | 1 | RFC 2597 |
| CS2 | `010000` | 16 | 1 | RFC 2474 |
| AF21 | `010010` | 18 | 1 | RFC 2597 |
| AF22 | `010100` | 20 | 1 | RFC 2597 |
| AF23 | `010110` | 22 | 1 | RFC 2597 |
| CS3 | `011000` | 24 | 1 | RFC 2474 |
| AF31 | `011010` | 26 | 1 | RFC 2597 |
| AF32 | `011100` | 28 | 1 | RFC 2597 |
| AF33 | `011110` | 30 | 1 | RFC 2597 |
| CS4 | `100000` | 32 | 1 | RFC 2474 |
| AF41 | `100010` | 34 | 1 | RFC 2597 |
| AF42 | `100100` | 36 | 1 | RFC 2597 |
| AF43 | `100110` | 38 | 1 | RFC 2597 |
| CS5 | `101000` | 40 | 1 | RFC 2474 |
| VOICE-ADMIT | `101100` | 44 | 1 | RFC 5865 |
| EF | `101110` | 46 | 1 | RFC 3246 |
| CS6 | `110000` | 48 | 1 | RFC 2474 |
| CS7 | `111000` | 56 | 1 | RFC 2474 |

Every entry above is **Pool 1** (`xxxxx0`, Standards Action) except **LE**, which is **Pool 3** (`xxxx01`) — see `2-QoS-Models` §4.2 for the pool rules. Notice the table is naturally sorted by decimal value, and every standards-defined value except LE is **even** (Pool 1's defining property is the last bit = 0).

---

## 9. CCIE-Depth Topics

### 9.1 Reconstructing any value from first principles under exam pressure

Rather than memorizing 23 numbers, memorize **three small formulas**:

| PHB family | Formula | Anchor value to remember |
|---|---|---|
| CS | n × 8 | CS1 = 8 |
| AF | (x × 8) + (y × 2) | AF11 = 10 |
| EF / VOICE-ADMIT / LE | fixed codepoints | EF = 46, VOICE-ADMIT = 44, LE = 1 |

From CS1 = 8, every other CS is a multiple of 8. From AF11 = 10, add 8 per class step and 2 per drop-precedence step (§4.2). The three fixed values are few enough to memorize directly, and EF/VOICE-ADMIT are adjacent (46/44) by design (§6).

### 9.2 Why EF and VOICE-ADMIT differ by exactly 2

`101110` vs `101100` differ only in **bit 4** (the second-to-last bit), which is worth **2** in the place-value table. RFC 5865's stated design goal — keep the first 4 bits identical — mathematically guarantees the two values are close together (within the same 8-value block: 40–47), which is also why both fall under **CS5's block** when read at the top-3-bit level (`101` = 5 = CS5 decimal 40, and both 44 and 46 sit inside the range 40–47 headed by that top-3-bit pattern).

### 9.3 Detecting a value's PHB family from the binary alone

Given a mystery decimal value in the 0–63 range, converting to binary and reading the pattern often identifies the PHB family before consulting a table:

| Pattern seen | Diagnosis |
|---|---|
| Ends in `000` | Class Selector |
| Ends in `010`, `100`, or `110` (and top 3 bits ≠ `101`) | Assured Forwarding — top 3 bits = class, bits 3–4 = drop precedence |
| `101110` exactly | EF |
| `101100` exactly | VOICE-ADMIT |
| `000001` exactly | LE |
| Ends in `01` or `11` and isn't one of the above | Local/experimental (Pool 2 or 3) — not a general-use PHB |

### 9.4 The two numbers every design conversation needs

`[Common practice]` When discussing capacity or SLAs, the DSCP decimal alone is not the number stakeholders usually want — the **ToS/Traffic-Class byte value** (DSCP × 4) is what appears in a packet capture's single-byte field, and is the number most often asked for in troubleshooting ("what does `0xB8` mean?" → EF, 46; verified in §1 and §5).

---

## 10. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | DSCP decimal ≠ ToS byte decimal | ToS byte = DSCP × 4; confusing the two misreads packet captures |
| 2 | AF and CS in the same class share top-3-bits | AF31 (26) reads as Precedence/CS3 (24) to top-3-bit-only equipment (`4-QoS-Marking-Headers` §8.1) |
| 3 | EF may be re-marked at a boundary **only to another EF-conformant codepoint** | It cannot be silently demoted to Default under RFC 2598/3246's mutability rule |
| 4 | VOICE-ADMIT is not "a slightly different EF" — it **requires** capacity admission | Marking traffic VOICE-ADMIT without CAC misrepresents its conformance |
| 5 | LE is Pool 3 (`000001`), not the older CS1-based recommendation | RFC 8622 obsoletes the RFC 3662 guidance to use CS1 for low-priority data |
| 6 | LE survives Bleach-ToS-Precedence but is wiped by Bleach-low | A re-marking event can silently turn "please deprioritize this" into "treat as normal" |
| 7 | Not every even DSCP value in 0–63 is a defined general-use PHB | Only the 23 values in §8's table have IANA/RFC meaning; others are local/experimental or unassigned |

---

## 11. Quick Recap

| PHB | Decimal | Binary | Formula |
|---|:--:|:--:|---|
| DF/CS0 | 0 | `000000` | — |
| LE | 1 | `000001` | fixed (Pool 3) |
| CSn | n×8 | `nnn000` | n × 8 |
| AFxy | — | `xxxyy0` | (x×8) + (y×2) |
| VOICE-ADMIT | 44 | `101100` | fixed |
| EF | 46 | `101110` | fixed |
| ToS byte | — | — | DSCP × 4 |

---

## References

**Standards Track**
- RFC 2474 — Definition of the DS Field (Default PHB, Class Selector codepoints)
- RFC 2597 — Assured Forwarding PHB Group
- RFC 3246 — An Expedited Forwarding PHB (obsoletes RFC 2598)
- RFC 5865 — A DSCP for Capacity-Admitted Traffic (VOICE-ADMIT; updates RFC 4542, RFC 4594)
- RFC 8622 — A Lower-Effort PHB (LE) for Differentiated Services (obsoletes RFC 3662; updates RFC 4594, RFC 8325)

**Referenced (covered in other notes)**
- RFC 791 — Internet Protocol (legacy Precedence, background for §1's byte-shift arithmetic)
- RFC 9435 — Considerations for Assigning a New Recommended DSCP (bleaching behaviour referenced in §7's LE gotcha)
