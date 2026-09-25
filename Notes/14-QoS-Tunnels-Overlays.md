## QoS in Tunnels and Overlays

> 💡 **TL;DR:** A tunnel wraps one packet inside another, and that immediately raises a question none of the previous 13 notes had to answer: **which header does QoS look at — the outer, the inner, or some combination of both?** RFC 2983 (2000) answers this for DSCP with two named conceptual models: the **Uniform model** (inner and outer DSCP treated as one field — copied out on encapsulation, copied back in on decapsulation) and the **Pipe model** (the tunnel is opaque; the inner DSCP is set independently and survives untouched, no matter what happens to the outer). MPLS labeling (`4-QoS-Marking-Headers` §6) adds a third: the **Short Pipe model**, which behaves like Pipe for scheduling decisions but like Uniform for the final egress DSCP rewrite. **ECN cannot use either model as-is**, because ECN is a live congestion signal, not a static class — RFC 6040 (2010) defines its own precise combination rules so that a CE mark picked up anywhere along the outer path is never silently lost when the tunnel is decapsulated, fixing real gaps in the original RFC 3168/RFC 4301 rules.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / vendor guidance · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 2983 (Diffserv and Tunnels) — Informational. RFC 6040 (ECN Tunnelling) — Standards Track (updates RFC 3168, RFC 4301, RFC 4774). RFC 4301 (IPsec architecture) — Standards Track. RFC 3270 (MPLS Diffserv, `4-QoS-Marking-Headers` §6) — Standards Track.

---

## 1. Why Tunnels Break the Simple Model

Every note so far assumed **one** IP header per packet. A tunnel means there are now **two** (or more) — an **inner** header (the original packet, exactly as its source built it) wrapped inside an **outer** header (added by the tunnel ingress, read and acted on by everything between ingress and egress).

```
 Original packet:        [ IP header (DSCP=X) | payload ]

 After tunnel encap:     [ Outer IP header (DSCP=?) | [ Inner IP header (DSCP=X) | payload ] ]
                                    ^
                          this is the ONLY header any router between
                          ingress and egress can see or act on
```

Every mechanism in notes 3–13 — classification, marking, policing, queuing, ECN — operates on **whatever header is visible to a given device**. A router in the middle of a tunnel classifies, polices, and marks the **outer** header; it has no visibility into the inner one at all. This single fact is the source of every question this note answers.

---

## 2. RFC 2983's Two Conceptual Models for DSCP

### 2.1 The Uniform Model

`[Guidance]` RFC 2983 §3.1, quoted directly: *"a uniform model that views IP tunnels as artifacts of the end to end path from a traffic conditioning standpoint; tunnels may be necessary mechanisms to get traffic to its destination(s), but have no significant impact on traffic conditioning. In this model, any packet has exactly one DS Field that is used for traffic conditioning at any point, namely the DS Field in the outermost IP header; any others are ignored."*

Mechanically: **copy DSCP to the outer header at encapsulation; copy the outer header's DSCP back to the inner header at decapsulation** (overwriting whatever the inner DSCP was before, including any re-marking that happened to the outer header in transit).

```
 Encap:    inner DSCP=X  --copy-->  outer DSCP=X
 (transit: outer DSCP may be re-marked to Y by a DS domain along the path)
 Decap:    outer DSCP=Y  --copy-->  inner DSCP=Y     (X is overwritten, replaced by Y)
```

RFC 2983's own stated rationale: this lets IP tunnels be configured **"without regard to diffserv domain boundaries because diffserv traffic conditioning functionality is not impacted by the presence of IP tunnels."** In other words, Uniform model treats the tunnel as functionally invisible to DiffServ — any re-marking that happens along the tunnel path is meant to reach the original packet, exactly as it would if there were no tunnel at all.

### 2.2 The Pipe Model

`[Guidance]` RFC 2983 §3.1, the second model: *"a pipe model that views an IP tunnel as hiding the nodes between its ingress and [egress]"* from the rest of the path's traffic-conditioning perspective — the tunnel is opaque. The inner DSCP is set **independently** by the tunnel ingress (or by whatever set the DSCP before the packet reached the tunnel) and is **never overwritten** by whatever happens to the outer header.

```
 Encap:    inner DSCP=X  (untouched)   outer DSCP=Y  (set independently, per tunnel policy)
 (transit: outer DSCP may be re-marked to Z along the path)
 Decap:    outer header discarded entirely -- inner DSCP is still X, exactly as it started
```

RFC 2983 gives a clear rule for **which value actually matters** under each model, quoted directly: *"the outer DSCP value usually contains the useful information for tunnels based on the uniform model, and the inner DSCP value usually contains the useful information for tunnels based on the pipe model."*

### 2.3 Why IPsec specifically mandates Pipe

`[Guidance]` RFC 2983, quoted directly: *"IPSec tunnels are usually based on the pipe model, and for security reasons are currently required to select the inner DSCP value; they should not be configured to select the outer DSCP value in the absence of an adequate security analysis."* This connects directly to the security reasoning already established in `3-QoS-Classification-Trust` §6.1 — an encrypted tunnel's outer header is visible to (and potentially manipulable by) every device along the path, while the inner header is protected by the tunnel itself. Trusting a value from the visible, potentially-tampered outer header to silently overwrite the protected inner value (as Uniform model does) would undermine exactly the protection IPsec exists to provide.

### 2.4 The recommended practical simplification

`[Guidance]` RFC 2983 recognizes that a tunnel could, in principle, use a fully general traffic-conditioning function combining both DSCPs in arbitrary ways — but recommends against this complexity, quoted directly: *"the simpler approach of statically selecting either the inner or outer DSCP value at decapsulation is recommended... Tunnels should support static selection of one or the other DSCP value at tunnel egress."* The rationale given: *"usually only one of the two DSCP values contains useful information"* — so a tunnel implementation should pick Uniform or Pipe behaviour outright, rather than attempting some blended, per-packet combination logic.

---

## 3. MPLS's Third Model — Short Pipe (Recap and Extension)

`4-QoS-Marking-Headers` §6.2 already introduced E-LSP and L-LSP as *how* the MPLS TC field carries PHB information. RFC 9435's summary (verified directly in earlier research) names the **three LSR (Label Switching Router) models** that determine how an MPLS domain's egress treats a labeled packet — directly analogous to RFC 2983's Uniform/Pipe distinction, but specifically for the **MPLS TC field's relationship to the underlying IP DSCP**:

| Model | Scheduling decision at egress | DSCP at egress |
|---|---|---|
| **Uniform Model** | Based on the received MPLS TC | **Rewrites** the egress DSCP to match the (possibly re-marked) MPLS TC — the MPLS-domain equivalent of RFC 2983's Uniform model |
| **Pipe Model** | Based on the received MPLS TC | Leaves the DSCP **untouched** — the underlying IP DSCP is never affected by anything that happened at the MPLS TC layer |
| **Short Pipe Model** | Based on the received MPLS TC (**same as Pipe** for scheduling) | Leaves the DSCP **untouched** (**same as Pipe** for marking) — the distinguishing difference from Pipe is in *which node's own DSCP behaviour applies at final delivery*, not in the TC-based forwarding treatment itself |

`[Guidance]` The key structural point worth preserving from RFC 9435's summary, quoted directly: *"In the Uniform and Pipe models, the egress MPLS router forwards traffic based on the received MPLS TC. The Uniform Model includes an egress DSCP rewrite."* Short Pipe sits alongside Pipe as sharing this same received-TC-based forwarding behaviour, distinguishing itself in more subtle ways around final-hop treatment that are implementation-specific to a given MPLS domain's egress policy.

> 📝 **The naming parallel is deliberate, not coincidental.** MPLS's Uniform/Pipe/Short-Pipe terminology directly echoes RFC 2983's IP-tunnel Uniform/Pipe naming, because it is solving the **structurally identical problem** — one header (MPLS TC) riding alongside another (IP DSCP), with the same fundamental question of whether changes to the outer/label-layer marking should propagate back into the inner/IP-layer marking. Recognizing this parallel means the reasoning from §2.1–2.3 (Uniform = let outer changes flow back in; Pipe = keep them separate) transfers directly, without needing to learn a second, unrelated framework for MPLS.

---

## 4. GRE and VXLAN — Applying the Same Models to Common Encapsulations

`[Common practice]` Neither GRE (RFC 2784/2890) nor VXLAN (RFC 7348) defines its own DSCP-handling model — both simply **inherit** RFC 2983's Uniform/Pipe choice, since both are ordinary IP-in-IP-style encapsulations (VXLAN is UDP/IP encapsulation of an Ethernet frame; GRE is IP encapsulation of an arbitrary payload, commonly another IP packet). The design question for either is exactly RFC 2983's: **does the outer tunnel header's DSCP get set independently (Pipe), or copied from the inner header and copied back on exit (Uniform)?**

```
 GRE:     [ Outer IP (DSCP=?) | GRE header | Inner IP (DSCP=X) | payload ]
 VXLAN:   [ Outer IP (DSCP=?) | UDP | VXLAN header | Inner Ethernet | Inner IP (DSCP=X) | payload ]
```

**Why VXLAN and other network-virtualization overlays tend toward Pipe, not Uniform** — verified from research directly relevant to RFC 2983's application to modern overlays: network virtualization is described as *"typically more closely aligned with the Pipe model... where the DSCP value on the tunnel header is set based on a policy (which may be a fixed value, one based on the inner traffic class, or some other mechanism for grouping traffic)"* — and, tellingly, *"the Uniform model is not conceptually consistent with network virtualization, which seeks to provide strong isolation between encapsulated traffic and the physical network."* This is the exact same isolation reasoning already given for IPsec (§2.3): an overlay network's entire purpose is often to **decouple** the tenant's/customer's traffic classification from the underlying physical fabric's own classification — Uniform model's automatic bidirectional copying works directly against that goal.

> ⚠️ **Gotcha:** A common but avoidable design mistake is deploying a GRE or VXLAN overlay with **no explicit DSCP policy on the outer header at all** — leaving it at whatever platform default applies (often DSCP 0). This silently produces best-effort treatment for the entire tunnel's traffic across the underlay, regardless of how carefully the inner traffic was classified and marked — a Pipe-model tunnel still needs its outer header **actively marked** according to some policy (fixed value, or derived from the inner class), or the tunnel simply has no differentiated treatment across the physical network it traverses.

---

## 5. ECN in Tunnels — Why It Needs Its Own Rules (RFC 6040)

### 5.1 Why DSCP's models don't work for ECN

`[Standard-defined]` DSCP is a **static classification** — it doesn't change based on what's happening on the wire *right now*. ECN is fundamentally different: a router marks CE **live**, in direct response to momentary congestion (`12-QoS-ECN` §2.2). If a tunnel simply applied the **Pipe model** to ECN — discard the outer header entirely at decapsulation, keep only the inner value — any CE mark a congested router set on the **outer** header during transit would be **thrown away** the moment the tunnel egress removes that header. The original sender, watching only the inner header's fate, would never learn that congestion happened on the outer, tunneled portion of the path at all.

```
 Sender's original packet:  ECT(0) marked, sender expects to see CE if congested anywhere on the path
 Tunnel encap:               outer header copies ECT(0)
 Congested router mid-tunnel: marks OUTER header CE  (never touches the inner header at all!)
 Tunnel decap, naive Pipe:    discards outer entirely, forwards inner unchanged (still just ECT(0))
                              --> the CE signal is LOST. Sender never finds out.
```

This is precisely the motivating problem RFC 6040 exists to solve — ECN needs a **third, ECN-specific set of rules**, not a reuse of DSCP's Uniform or Pipe choice.

### 5.2 RFC 6040's purpose and scope

`[Standard-defined]` RFC 6040 (Briscoe, 2010) Abstract, quoted directly: *"This document redefines how the explicit congestion notification (ECN) field of the IP header should be constructed on entry to and exit from any IP-in-IP tunnel. On encapsulation, it updates RFC 3168 to bring all IP-in-IP tunnels (v4 or v6) into line with RFC 4301 IPsec ECN processing. On decapsulation, it updates both RFC 3168 and RFC 4301 to add new behaviours for previously unused combinations of inner and outer headers."*

The document explicitly unifies what had been **two separate, slightly inconsistent rule sets** — RFC 3168's original ECN tunneling rules (§9 of that document, not covered in `12-QoS-ECN`, which focused on TCP's use of ECN) and RFC 4301's IPsec-specific ECN rules — into **one consistent specification** covering both IPsec and non-IPsec tunnels alike.

### 5.3 Encapsulation — two modes

`[Standard-defined]` Quoted directly: *"an encapsulator forwards the inner header without changing the ECN field. In normal mode, an encapsulator compliant with this specification MUST construct the outer encapsulating IP header by copying the two-bit ECN field of the incoming IP header. In compatibility mode, it clears the ECN field in the outer header to the Not-ECT codepoint."*

| Mode | Outer ECN field set to | When used |
|---|---|---|
| **Normal mode** (REQUIRED) | Copy of the inner header's ECN field | Default — the tunnel egress is known/assumed to support RFC 6040-compliant decapsulation |
| **Compatibility mode** | Always **Not-ECT**, regardless of the inner value | Interworking with a legacy egress that predates RFC 6040 and would not correctly propagate ECN marks back inward |

RFC 6040 explains directly why compatibility mode still needs to exist: *"This is necessary for the ingress to interwork with legacy decapsulators ([RFC2481], [RFC2401], [RFC2003]) that do not propagate ECN markings added to the outer header. Otherwise, such legacy decapsulators would throw away congestion notifications before they reached the transport layer."* Forcing the outer header to Not-ECT in compatibility mode means a congested router along the tunnel **must drop** rather than mark the packet (since RFC 3168 §5 already established a router cannot set CE on a Not-ECT packet, `12-QoS-ECN` §2.2) — sacrificing ECN's loss-avoidance benefit for that tunnel segment, but at least preserving the **congestion signal itself** in a form (a drop) that even a legacy egress will correctly interpret.

### 5.4 Decapsulation — the combination rules

`[Standard-defined]` This is RFC 6040's central contribution — precise rules for combining the inner and outer ECN fields into a single outgoing value, verified directly:

```
 if outer == CE:                     output = CE           (a marked outer ALWAYS wins)
 else if inner == Not-ECT:           output = Not-ECT       (outer is ignored in this specific case)
 else:                               output = inner value   (outer otherwise ignored; inner passes through)
```

Plus one mandatory exception carried forward from the original RFC 3168 rule, quoted directly: *"RFC 3168 (but not RFC 4301) also specified that the decapsulator must drop a packet with a Not-ECT inner and CE in the outer."* This specific combination — an endpoint that never signaled ECN-capability (Not-ECT), yet somehow arrives with the outer header marked CE — is treated as anomalous enough that RFC 3168's original rule (preserved by RFC 6040 for that specific document's applicability) says to **drop**, not merely forward as Not-ECT.

### 5.5 Worked table — every inner/outer combination

| Inner | Outer | Outgoing (decapsulated) result | Why |
|:--:|:--:|---|---|
| Not-ECT | Not-ECT | Not-ECT | No congestion signaled anywhere |
| Not-ECT | ECT(0) or ECT(1) | Not-ECT | Outer's ECT status is irrelevant once we know the inner never opted in |
| Not-ECT | CE | **Drop** (RFC 3168 rule) or Not-ECT (RFC 4301 context) | Anomalous: congestion marked on a flow that never signaled ECN-capability |
| ECT(0) | Not-ECT | ECT(0) | Outer never saw a mark; inner's original value passes through |
| ECT(0) | ECT(0) | ECT(0) | Consistent, no congestion |
| ECT(0) | ECT(1) | **ECT(1)*** | See the specific RFC 6040 update below |
| ECT(0) | CE | **CE** | The critical case §5.1 was built around — the outer's live congestion mark is now correctly propagated inward |
| ECT(1) | Not-ECT | ECT(1) | Outer never marked; inner passes through |
| ECT(1) | ECT(0) or ECT(1) | ECT(1) | Consistent |
| ECT(1) | CE | **CE** | Same principle as the ECT(0)/CE case |
| CE | (anything) | CE | Already marked; stays marked |

**The specific new rule for ECT(0) inner / ECT(1) outer**, quoted directly from the exact update RFC 6040 makes to RFC 3168 §9.1.1: *"The outer, not the inner, is propagated when the outer is ECT(1)"* (in this specific inner=ECT(0) case) — this is one of the **"previously unused combinations"** RFC 6040's abstract refers to, added specifically to let a tunnel carry **two distinguishable severity levels** of congestion signal (e.g., for schemes like Pre-Congestion Notification, PCN) rather than RFC 3168's original single-severity-level design.

> 📝 **The general pattern worth remembering**: *"if the outer is CE, the outgoing ECN field is set to CE; otherwise, the outer is ignored and the inner is used for the outgoing ECN field"* (quoted directly) — with the ECT(1)-outer-over-ECT(0)-inner case as the one specific, deliberate exception, and the Not-ECT-inner cases (rows 1–3) as a separate, self-contained rule. Getting this right matters because it's the exact mechanism that prevents the data-loss scenario in §5.1: **a CE mark set anywhere on the outer header during transit is guaranteed to reach the inner header at decapsulation**, in every single combination where it's set — that guarantee is the entire point of the specification.

---

## 6. CCIE-Depth Topics

### 6.1 Why RFC 6040 explicitly chose not to add a matching "Limited Functionality" decapsulation mode

Research surfaced RFC 6040's own reasoning on a related design question — whether to give the tunnel **egress** two modes (mirroring the ingress's normal/compatibility split), specifically to guard against a covert-channel security concern via the CU (then-unused) codepoint combinations. RFC 6040's own conclusion, quoted directly: *"we decided not to add the extra complexity of two modes on a compliant tunnel egress merely to cater for an historic security concern that is now considered manageable."* This is a deliberate simplicity-over-defense-in-depth trade-off, explicitly reasoned through in the RFC rather than an oversight — worth knowing because it shows the asymmetry between ingress (two modes, for backward-compatibility reasons) and egress (one mode, a conscious simplification) is intentional, not an inconsistency.

### 6.2 The ECN-nonce interaction RFC 6040 deliberately did not fully solve

Research also surfaced a specific, named limitation: under the decapsulation rule "if inner and outer headers carry contradictory ECT values, only the inner header is preserved," a since-Historic mechanism (the ECN nonce, `12-QoS-ECN` §7.2) could in principle have detected a CE mark that was set and then illegitimately stripped somewhere along the tunnel, but RFC 6040's new rules "do not solve this problem" for that specific detection use case. This is a good illustration of a general principle worth internalizing across this whole series: a standard fixing one class of problem (here, ensuring CE marks propagate correctly in the *common* case) does not automatically fix every adjacent, more exotic concern (here, cryptographic-style detection of deliberate mark-stripping) — read a fix's stated scope carefully rather than assuming it's exhaustive.

### 6.3 Applying the Uniform/Pipe choice to `3-QoS-Classification-Trust`'s trust boundary concept

Since a **Pipe-model** tunnel deliberately isolates the inner DSCP from whatever happens to the outer header, a Pipe tunnel's egress is functionally acting as its own **independent DS boundary node** (`3-QoS-Classification-Trust` §6.2) for the inner traffic — whatever trust decisions applied to the outer, physical-network path are **irrelevant** to how the newly-decapsulated inner packet should be treated next; that decision starts fresh, exactly as if the packet had just arrived from any other untrusted or trusted source. A **Uniform-model** tunnel, by contrast, explicitly does **not** grant this fresh start — a re-marking event anywhere along the outer path becomes the inner packet's new marking too, meaning the tunnel's far end inherits whatever trust conditioning happened mid-path, for better or worse.

---

## 7. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | A mid-tunnel router only ever sees and acts on the **outer** header | Every classification, marking, and policing mechanism in notes 3–11 is scoped to whichever header is currently outermost |
| 2 | Uniform model **overwrites** the inner DSCP with whatever the outer became — even if that's a re-mark from an untrusted domain | Only appropriate when the tunnel is meant to be functionally invisible to DiffServ end to end |
| 3 | IPsec is **required** to use Pipe model for DSCP, specifically for security reasons | An outer header visible to the whole path should not be trusted to silently overwrite a protected inner value |
| 4 | Network-virtualization overlays (VXLAN and similar) are **conceptually inconsistent** with Uniform model | Uniform model undermines the tenant/underlay isolation such overlays exist to provide |
| 5 | A Pipe-model tunnel with **no explicit outer DSCP policy** silently gets best-effort treatment across the underlay | Pipe model still requires *actively* setting the outer DSCP by some policy — it doesn't happen automatically |
| 6 | MPLS's Uniform/Pipe/Short-Pipe naming directly parallels RFC 2983's IP-tunnel models | Same underlying question (does outer-layer re-marking flow back to the inner layer), just at the label layer instead of the tunnel-header layer |
| 7 | DSCP's Uniform/Pipe choice **cannot** be applied as-is to ECN | ECN is a live per-packet congestion signal, not a static class — naive Pipe-model handling would silently lose CE marks |
| 8 | RFC 6040's rule: **if the outer is CE, the outgoing field is always CE** | This single rule is what prevents a live congestion signal picked up mid-tunnel from being discarded at decapsulation |
| 9 | A Not-ECT inner combined with a CE outer is treated as **anomalous** (drop, under RFC 3168's original rule) | Distinct from the ordinary "outer wins" rule — this specific combination signals something unexpected happened |
| 10 | RFC 6040's ingress has two modes (normal/compatibility); its egress deliberately has only **one** | A conscious simplicity trade-off, not an oversight — verified directly in the RFC's own reasoning |

---

## 8. Quick Recap

| Concept | One-line answer |
|---|---|
| The core tunnel problem | Mid-path devices only see the outer header; the inner header's fate must be decided at encapsulation/decapsulation |
| Uniform model | Inner and outer DSCP treated as one field — copied out, copied back in |
| Pipe model | Tunnel is opaque; inner DSCP is set independently and never overwritten |
| IPsec's requirement | Pipe model, for security reasons — never let the outer (untrusted, visible) header overwrite the protected inner value |
| MPLS's three models | Uniform (DSCP rewritten at egress), Pipe, Short Pipe (egress DSCP left alone in both) |
| GRE/VXLAN's approach | Inherit RFC 2983's choice; overlays typically favor Pipe for isolation reasons |
| Why ECN needs its own rules | It's a live signal, not a static class — Pipe-model handling would lose in-transit CE marks |
| RFC 6040's decapsulation rule | If outer is CE → output CE; else if inner is Not-ECT → output Not-ECT; else → output inner |
| The one mandatory drop case | Not-ECT inner + CE outer → drop (anomalous combination, RFC 3168's original rule) |
| RFC 6040's ingress modes | Normal (copy inner to outer) vs. Compatibility (force outer to Not-ECT, for legacy egresses) |

---

## References

**Informational**
- RFC 2983 — Differentiated Services and Tunnels (Uniform and Pipe models for DSCP)

**Standards Track**
- RFC 6040 — Tunnelling of Explicit Congestion Notification (updates RFC 3168, RFC 4301, RFC 4774)
- RFC 4301 — Security Architecture for the Internet Protocol (IPsec's original ECN/DSCP handling, updated by RFC 6040)
- RFC 3270 — Multi-Protocol Label Switching (MPLS) Support of Differentiated Services (E-LSP/L-LSP, background from `4-QoS-Marking-Headers` §6)
- RFC 2784, RFC 2890 — Generic Routing Encapsulation (GRE) and key/sequence-number extensions
- RFC 7348 — Virtual eXtensible Local Area Network (VXLAN)

**Referenced (background and what feeds into this note)**
- `4-QoS-Marking-Headers` (MPLS TC field, E-LSP/L-LSP)
- `3-QoS-Classification-Trust` (DS boundary nodes; why encryption limits classification; the security reasoning behind IPsec's Pipe-model requirement)
- `12-QoS-ECN` (RFC 3168's codepoints and TCP feedback loop — the mechanism RFC 6040 ensures survives a tunnel)
- `9435` re-marking behaviours (`3-QoS-Classification-Trust` §7) — the general phenomenon of markings changing in transit, of which tunnel-boundary re-marking is one specific, structured case
