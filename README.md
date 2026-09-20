# QoS Notes — Vendor-Neutral, RFC-Verified

Study notes on **Quality of Service (QoS)** for network engineers preparing at **CCNP and CCIE level**. Every topic is written to be **OEM-neutral** and checked against primary sources — **RFCs, ITU-T Recommendations and IEEE standards** — rather than a single vendor's documentation.

---

## Contents

Click a subject to open the note.

| # | Note | Covers |
|--:|---|---|
| 1 | [QoS Fundamentals](Notes/1-QoS-Fundamentals.md) | Bandwidth, delay, jitter, loss; delay budget (ITU-T G.114); Y.1541 classes; RTP jitter; loss vs TCP throughput; traffic types; QoS toolset |
| 2 | [QoS Models](Notes/2-QoS-Models.md) | Best effort, IntServ/RSVP (Guaranteed & Controlled-Load), DiffServ architecture, DS field and DSCP pools, comparison and hybrid models |
| 3 | [Classification and Trust](Notes/3-QoS-Classification-Trust.md) | BA vs multifield classification, trust boundary |
| 4 | [Marking and Headers](Notes/4-QoS-Marking-Headers.md) | IPv4 ToS / IPv6 Traffic Class, IP Precedence, 802.1Q PCP/DEI, MPLS TC, ECN bits, layer mapping |
| 5 | [PHB and DSCP Values](Notes/5-QoS-PHB-DSCP-Values.md) | Default, CS, AF, EF, LE, VOICE-ADMIT, with binary → decimal conversion |
| 6 | [Class Design](Notes/6-QoS-Class-Design.md) | RFC 4594 4/8/12-class models, Wi-Fi mapping |
| 7 | [Policy Model](Notes/7-QoS-Policy-Model.md) | Generic class → match → action model, order of operations, hierarchy |
| 8 | [Policing](Notes/8-QoS-Policing.md) | Token bucket, single-rate and dual-rate metering (RFC 2697/2698/4115) |
| 9 | [Shaping](Notes/9-QoS-Shaping.md) | Bc, Be, Tc maths; shaping vs policing |
| 10 | [Queuing and Scheduling](Notes/10-QoS-Queuing-Scheduling.md) | FIFO, PQ, WRR, DRR, WFQ, priority queuing, hierarchical scheduling |
| 11 | [Congestion Avoidance](Notes/11-QoS-Congestion-Avoidance.md) | Tail drop, RED/WRED, buffers, microbursts, CoDel/PIE/FQ-CoDel |
| 12 | [ECN](Notes/12-QoS-ECN.md) | RFC 3168 end to end, negotiation, bleaching, RFC 8311 |
| 13 | [L4S and AccECN](Notes/13-QoS-L4S-AccECN.md) | AccECN, L4S, DualQ |
| 14 | [Tunnels and Overlays](Notes/14-QoS-Tunnels-Overlays.md) | GRE, IPsec, VXLAN, MPLS uniform/pipe/short-pipe |
| 15 | [Link Efficiency](Notes/15-QoS-Link-Efficiency.md) | LFI, header compression, MLPPP |
| 16 | [Control Plane](Notes/16-QoS-Control-Plane.md) | Protecting control-plane traffic |
| 17 | [Data Center](Notes/17-QoS-Datacenter.md) | PFC, ETS, DCBX, QCN, RoCEv2 congestion control |
| 18 | [Wireless](Notes/18-QoS-Wireless.md) | WMM/EDCA, 802.11 UP → AC mapping |
| 19 | [Troubleshooting](Notes/19-QoS-Troubleshooting.md) | Vendor-neutral method: where to measure, remarking checks, drop analysis |

**Suggested reading order:** follow the numbers. Notes 1–2 are foundations; 3–5 explain how traffic is identified and marked; 6–11 cover class design, rate control, scheduling and congestion handling; 12–13 cover ECN and L4S; 14–19 cover specialised environments and troubleshooting.

---

## Repository Layout

```
.
├── README.md
└── Notes/
    ├── 1-QoS-Fundamentals.md
    ├── 2-QoS-Models.md
    ├── 3-QoS-Classification-Trust.md
    └── ...            (one file per topic, numbered in reading order)
```

Files are named `N-QoS-<Topic>.md`. The contents table above always shows the correct reading order.

---

## How to Read the Notes

### Claim tags

Statements in the notes carry a tag showing where they come from:

| Tag | Meaning |
|---|---|
| `[Standard-defined]` | Defined in a standards document (RFC Standards Track, ITU-T Recommendation, IEEE standard) |
| `[Guidance]` | Recommendation in an Informational RFC or similar — not a protocol requirement |
| `[Common practice]` | Widely used engineering practice or vendor design guidance |
| `[Implementation-dependent]` | Varies by platform — check your device documentation |

### Note layout

Every note follows the same structure:

1. **💡 TL;DR** — the whole topic in one paragraph
2. Numbered sections with tables for comparisons and values
3. **⚠️ Gotcha** callouts — common misconceptions and traps
4. **CCIE-depth** section — maths, corner cases, interactions
5. **Quick recap** table
6. **References** — standards first, vendor documentation second

### Diagrams

Diagrams are plain **ASCII** so they render anywhere and stay editable, for example:

```
 Endpoint ---> Switch ---> Router ===WAN===> Router ---> Switch ---> Endpoint
```

Bit-level header layouts are drawn as ASCII bit fields.

### No vendor CLI

Notes contain **no OEM-specific configuration**. Behaviour that differs between platforms is described in words and tagged `[Implementation-dependent]`. Configuration is shown, where needed, as generic policy tables (class → match → action).

---

## Source Policy

1. **Primary sources first:** IETF RFCs, ITU-T Recommendations, IEEE standards. Section numbers and document status (Standards Track, Informational, Obsoleted) are noted.
2. **Vendor documents second:** used only to show how a concept is implemented or for widely cited design targets, and always labelled as such.
3. **Blogs and articles** are treated as readable starting points, never as authority.
4. **Conflicts and ambiguity** are stated openly rather than resolved silently.
5. **Numbers are worked, not just quoted:** worked examples state their assumptions so the arithmetic can be checked.

---

## Corrections and Contributions

Found an error or an outdated reference? Open an issue or pull request with:

- the note and section,
- what is wrong,
- the primary source (RFC/ITU-T/IEEE document and section number) that supports the correction.

Corrections backed by a section number are the fastest to merge.

---

## Disclaimer

These are study notes, not a substitute for the standards themselves or for your equipment vendor's documentation. RFCs and Recommendations are revised over time; always check the current version and status before relying on a value in a design, SLA or exam answer. RFC and ITU-T texts remain the property of their respective publishers; this repository paraphrases and summarises them and does not reproduce them.
