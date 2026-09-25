## Control Plane Protection (CoPP)

> 💡 **TL;DR:** Every device in this series has been treated, so far, as a black box that classifies, marks, polices, and queues **transit** traffic — packets passing *through* the device on their way somewhere else. But a router is also a **destination** in its own right: routing protocol packets, SSH sessions, SNMP polls, and ICMP all terminate **at** the device, addressed to the device's own control plane, not forwarded onward. RFC 6192 (2011) — the IETF's own guidance on this exact problem — calls this **"router control plane protection"**: applying the identical classify/meter/police toolkit already built up in notes 3–11, but aimed at **traffic destined for the router itself**, to stop that traffic from overwhelming the control plane's comparatively limited processing capacity. Its central design principle: identify **all** legitimate control-plane traffic explicitly, then filter or rate-limit everything else, applied **as close to the forwarding plane as possible** — because a control-plane CPU that gets starved of resources can lose routing adjacencies, drop management sessions, or fail outright, regardless of how well the *transit* traffic is being handled.

> 🏷️ **Tags:** `[Standard-defined]` RFC Standards Track / IEEE · `[Guidance]` Informational RFC or similar · `[Common practice]` engineering practice / observed deployment status · `[Implementation-dependent]` varies by platform.
>
> 📎 **Status of the key documents:** RFC 6192 — Informational, and specifically a **Best Current Practice-style operational guide** (it demonstrates a worked example rather than defining a protocol).

---

## 1. Why Control-Plane Traffic Is a Different Category Entirely

### 1.1 The two-plane architecture, as RFC 6192 itself defines it

`[Guidance]` RFC 6192 §2, quoted directly: *"The router control plane supports routing and management functions. It is generally described as the router architecture hardware and software components for handling packets destined to the device itself as well as building and sending packets originated locally on the device."* Contrasted directly against forwarding: *"The forwarding plane is typically described as the router architecture hardware and software components responsible for receiving a packet on an incoming interface, performing a lookup to identify the packet's IP next hop and determine the best outgoing interface towards the destination, and forwarding the packet out through the appropriate outgoing interface."*

```
                    +----------------------+
                    |  Router Control      |
                    |  Plane                |
                    +-----------+----------+
                                | (traffic destined
                                |  TO or built BY
                                |  this device)
                    +-----------+----------+
 Interface X =======| Forwarding Plane     |======= Interface Y
                    +----------------------+
                    (traffic passing THROUGH
                     the device to somewhere else)
```

RFC 6192's own Figure 1 (verified directly, reproduced in concept above) shows the control plane sitting **on top of, and interfacing with**, the forwarding plane — meaning any packet addressed to the device itself must first be recognized by the forwarding plane and specifically **diverted upward**, rather than simply forwarded out an egress interface.

### 1.2 Why this distinction matters for QoS specifically

Every mechanism in `7-QoS-Policy-Model` through `11-QoS-Congestion-Avoidance` assumes a policy is applied to traffic **crossing** an interface. Control-plane traffic breaks that assumption in a specific, consequential way: the forwarding plane is almost always **purpose-built, high-performance hardware** capable of line-rate processing (per RFC 6192's own characterization, quoted directly, forwarding-plane functionality is *"typically realized in high-performance Application [-Specific Integrated Circuits]"*), while the control plane is typically a **general-purpose CPU** with orders of magnitude less packet-processing capacity. A volume of traffic the forwarding plane handles without noticing can completely saturate the control plane's CPU.

> ⚠️ **Gotcha:** This means a QoS policy that carefully protects voice, video, and business traffic across every transit interface (notes 6–11) does **nothing at all** to protect the device's own routing protocol sessions, SSH access, or SNMP management — those are a **structurally separate** traffic path with its own, much smaller capacity ceiling, and need their own, separate policy.

---

## 2. RFC 6192's Core Methodology

### 2.1 The stated goal, precisely

`[Guidance]` RFC 6192 §1, quoted directly: *"It is advisable to protect the router control plane by implementing mechanisms to filter completely or rate-limit traffic not required at the control plane level (i.e., unwanted traffic). 'Router control plane protection' is the concept of filtering or rate-limiting unwanted traffic that would be diverted from the forwarding plane up to the router control plane."*

### 2.2 The identify-first, filter-second approach

`[Guidance]` RFC 6192's Abstract, quoted directly: *"This memo provides a method for protecting a router's control plane from undesired or malicious traffic. In this approach, **all legitimate router control plane traffic is identified**. Once legitimate traffic has been identified, a filter is deployed in the router's forwarding plane. That filter prevents traffic not specifically identified as legitimate from reaching the router's control plane, or rate-limits such traffic to an acceptable level."*

This is a specific, deliberate design choice — a **default-deny, explicitly-permit** posture, rather than a default-permit posture with exceptions carved out for known bad traffic. It's the same underlying philosophy already familiar from `3-QoS-Classification-Trust` §3's completeness requirement (every classifier needs a defined behaviour for the "everything else" case) — here applied specifically to **which traffic is allowed to reach the CPU at all**.

### 2.3 The recommended practical audit method

`[Guidance]` RFC 6192 acknowledges directly that building a complete, accurate list of legitimate control-plane traffic is harder than it sounds, quoted precisely: *"In an actual production environment, predicting a complete and exhaustive list of traffic necessary to reach the router's control plane for day-to-day operation may not be as obvious as the example described herein. One recommended method to gauge this set of traffic is to allow all traffic initially, and audit the traffic"* actually seen, before tightening the policy to match reality. This is a pragmatic, explicitly-stated compromise: implement the filter in a permissive, monitor-only posture first, observe what legitimate traffic actually looks like in that specific network, and **then** convert to enforcing.

### 2.4 Why placement matters — "closer is more effective"

`[Guidance]` RFC 6192, quoted directly: *"The closer the filters and rate limiters are to the forwarding plane and line-rate hardware, the more effective the protection is and the more resistant the system is to DoS attacks."* This is a direct, practical consequence of §1.2's capacity asymmetry: if a filter or policer is itself implemented **in software on the same constrained CPU** it's trying to protect, an attack large enough to overwhelm that CPU can overwhelm the *protection mechanism* right along with everything else it's supposed to be guarding. Implementing the filter in the same high-performance forwarding-plane hardware that already handles line-rate transit traffic means the filtering decision itself never becomes the bottleneck.

### 2.5 "Vulnerable surface is inversely proportional to filter granularity"

`[Guidance]` RFC 6192, quoted directly: *"The goal of the method for protecting the router control plane is to minimize the possibility for disruptions by reducing the vulnerable surface, which is inversely proportional to the granularity of the filter design."* In plain terms: a coarse filter (e.g., "allow all ICMP, rate-limited to some generous number") leaves more room for a bad actor operating *within* that broad allowance; a finer-grained filter (e.g., "allow ICMP only from specific known-good sources, at a tighter rate") shrinks the room available for abuse — at the cost of more configuration complexity and more careful, ongoing maintenance of exactly which traffic is legitimate.

---

## 3. What Kind of Traffic Needs Protecting — RFC 6192's Own Worked Categories

`[Guidance]` RFC 6192's appendix provides a full worked example (a sample policy, in specific vendor syntaxes, which this vendor-neutral series does not reproduce as configuration — but the **traffic categories** it identifies are directly useful, restated here in class/match/action form consistent with `7-QoS-Policy-Model`'s vendor-neutral policy table):

| Class (traffic type) | Typical treatment | Why this category exists |
|---|---|---|
| **IP fragments** destined to the router | Drop | RFC 6192's own example treats fragments addressed to the control plane as inherently suspicious — legitimate control-plane protocols rarely need to be fragmented |
| **ICMP** (and ICMPv6) | Rate-limited | Needed for operational tools (ping, traceroute, path MTU discovery) but a classic vector for flooding-style abuse |
| **Routing protocols** (e.g., OSPF, iBGP, eBGP) | Permitted, often unlimited or generously limited, from known neighbor addresses | The control plane's most safety-critical traffic — losing these sessions can cascade into much larger outages |
| **Management/administrative protocols** (SSH, SNMP, RADIUS, NTP, DNS) | Permitted, often source-restricted to specific known administrative or infrastructure addresses | Legitimate operational necessity, but a narrower, more identifiable set of expected sources than general Internet traffic |
| **All other IP traffic to the router** | Tightly rate-limited (a conservative catch-all rate) | The default-deny/rate-limit posture (§2.2) applied to everything not explicitly classified above |
| **Non-IP traffic to the router** | Tightly rate-limited (a separate catch-all) | Covers link-layer or other non-IP protocols that can still reach the control plane on some platforms |

> 📝 **The specific case RFC 6192 highlights as a cautionary example**: dropping *all* ICMP is tempting for security but has a real operational cost, quoted directly: *"discarding all ICMP traffic will have a negative impact on the operational use of ICMP tools such as ping or traceroute to debug network issues or to test deployment of a new circuit."* Its own suggested refinement: *"an astute operator could define varying rate limits for ICMP such that internal traffic is granted uninhibited access to the router control plane, while traffic from external addresses is rate-limited."* This is a direct, worked illustration of §2.5's granularity principle — a single blunt "block/allow ICMP" decision is replaced with a **source-aware**, differentiated policy.

### 3.1 Scope — this is about traffic *to* the router, not traffic passing *through* it

`[Guidance]` RFC 6192 states this limitation explicitly and directly: *"the filters described in this memo are applied only to traffic that is destined for the router, and not to all traffic that is passing through the router."* This is the precise, formal boundary between this note and everything preceding it in the series: notes 3–15 all concern **transit** traffic; this note concerns traffic whose destination **is** the device applying the policy. A packet can be simultaneously subject to a transit QoS policy (as it's forwarded onward) and, on a different packet entirely, be subject to a control-plane protection policy (if it's addressed to the router itself) — these are two structurally separate classification-and-policy decisions, even though both reuse the identical underlying toolkit (classify → meter → mark/drop, from `7-QoS-Policy-Model`).

---

## 4. Where This Fits the Vendor-Neutral Model

`[Common practice]` Nothing about control-plane protection requires a *new* mechanism beyond what notes 3, 7, and 8 already established:

- **Classification** (`3-QoS-Classification-Trust`): the classifier's key is exactly the same kind already covered — protocol, source/destination address, port — the only difference is that the destination address being matched is the **router's own** address(es), not some transit destination.
- **Policing** (`7-QoS-Policy-Model` §2.6, `8-QoS-Policing`): the "rate-limit" actions RFC 6192 describes for ICMP, the catch-all classes, and so on are ordinary meter-plus-marker/dropper constructions — the same srTCM/trTCM-style token-bucket mechanisms already fully specified in note 8, just applied to a different traffic population.
- **The default-deny posture** (§2.2): this is exactly `3-QoS-Classification-Trust` §3's **completeness** requirement — every classifier needs a defined, catch-all behaviour for anything that doesn't match an explicit filter — here deliberately chosen to be "drop or heavily rate-limit," rather than "pass through as best-effort," precisely because the traffic in question is destined for a resource-constrained CPU rather than an abundant transit link.

```
 Packet arrives on an interface
          |
   Is this packet addressed TO the router itself? --no--> ordinary transit QoS policy
          | yes                                            (notes 3-11, this note doesn't apply)
   Apply control-plane classifier (protocol/source/dest)
          |
   Matches a known-legitimate category?  --no--> default class: drop or tightly
          | yes                                   rate-limit (§2.2, §2.5)
   Meter against that category's rate (§8's token-bucket mechanisms)
          |
   Conform --> deliver to control plane CPU
   Exceed  --> drop (or, per RFC 6192's examples, simply drop -- CoPP typically
               does not re-mark, since there's no "downstream" to re-mark for)
```

> 📝 One structural difference from transit QoS worth noting explicitly: transit policing (`8-QoS-Policing`) often **re-marks** excess traffic to a lower class rather than dropping it outright, because that excess traffic still has somewhere useful to go. Control-plane protection policies, by contrast, essentially always **drop** excess traffic once past its rate limit (as RFC 6192's own worked example does throughout) — there is no "lower class of control-plane service" for a routing protocol packet to be demoted into; the choice is really only "deliver to the CPU" or "not."

---

## 5. CCIE-Depth Topics

### 5.1 Why routing protocols get the most permissive treatment, structurally

Revisiting the traffic-category table in §3: routing-protocol sessions (OSPF, BGP) are typically given the **most generous** allowance among all control-plane categories, not because they're inherently more trustworthy, but because **losing them has the largest blast radius**. A dropped SSH session inconveniences one administrator; a dropped OSPF or BGP adjacency, triggered by an overly aggressive control-plane rate limit during a legitimate traffic spike, can cause a **route flap or a full outage** propagating well beyond the single device being protected. This is a direct extension of the "network control traffic" reasoning already established in `6-QoS-Class-Design` §4 (CS6's protection, and the "user traffic is not allowed to use this service class" rule) — CoPP is the mechanism that specifically protects that same traffic *at the device itself*, complementing (not duplicating) the CS6/DSCP-based protection that applies as that traffic transits *other* devices.

### 5.2 The relationship between CoPP and DSCP-based transit QoS

Because CoPP and transit QoS are structurally separate decision paths (§3.1), marking a control-plane-destined packet with CS6 (per `5-QoS-PHB-DSCP-Values` and `6-QoS-Class-Design`) helps that packet get good **transit** treatment on its way to the router, but does **nothing** to guarantee it will be treated well once it *arrives* and needs to actually reach the CPU — that arrival-time protection is exactly what CoPP governs, and it operates on its own classification criteria (protocol/address/port), independent of whatever DSCP the packet happened to carry in transit. A well-designed network needs **both**: DSCP-based prioritization (notes 4–11) to get routing-protocol traffic across the network reliably, **and** CoPP (this note) to make sure that traffic, once it arrives, doesn't get crowded out by a flood of unrelated or malicious traffic also directed at the same device.

### 5.3 Why "allow all, then audit" (§2.3) is a sound methodology despite sounding insecure

It might seem contradictory that RFC 6192's own recommended starting point is a **permissive** policy — but this is a deliberate, reasoned trade-off, not a security compromise: deploying an **incomplete** restrictive policy first risks silently dropping legitimate traffic the operator didn't anticipate (a category of self-inflicted outage), whereas deploying a **permissive, monitored** policy first risks nothing beyond continuing to have no protection for a limited audit period — a strictly smaller and more recoverable risk. This mirrors a general operational principle already implicit in `9-QoS-Shaping` §6.4's caution about undersized buffers turning a shaper into a policer unexpectedly: **understand real traffic patterns before applying a hard restriction**, rather than guessing and discovering the gaps only when something legitimate breaks.

---

## 6. Gotchas Summary

| # | Gotcha | Why it matters |
|--:|---|---|
| 1 | Transit QoS policies (notes 3–11) provide **zero** protection for the device's own control plane | These are structurally separate traffic paths and separate policies, even though built from the same underlying toolkit |
| 2 | The control plane's processing capacity is typically **orders of magnitude smaller** than the forwarding plane's | A traffic volume the forwarding plane handles trivially can overwhelm the CPU handling control-plane traffic |
| 3 | RFC 6192's approach is **default-deny, explicit-permit** — not default-permit with exceptions | A more restrictive, but more auditable and defensible, security posture |
| 4 | Filters implemented in **software on the same CPU** they protect can themselves be overwhelmed by the attack they're meant to stop | RFC 6192 explicitly recommends filtering as close to line-rate forwarding-plane hardware as possible |
| 5 | Blanket-dropping all ICMP breaks real operational tools (ping, traceroute, PMTUD) | RFC 6192's own cited example — favor a differentiated, rate-limited approach over an outright block |
| 6 | CoPP policies typically **drop** excess traffic rather than re-marking it | Unlike transit policing, there's no lower-priority "downstream" for control-plane traffic to be demoted into |
| 7 | This note's scope is traffic **destined for** the router, not traffic **passing through** it | RFC 6192 states this boundary explicitly — don't conflate CoPP with ordinary transit QoS |
| 8 | Starting with a **permissive, audited** policy is a deliberate, reasoned methodology | Reduces the risk of a restrictive policy silently breaking legitimate, unanticipated traffic |

---

## 7. Quick Recap

| Concept | One-line answer |
|---|---|
| What CoPP protects | The router's own control plane — traffic destined to or originated by the device itself |
| RFC 6192's core method | Identify all legitimate control-plane traffic explicitly; filter or rate-limit everything else |
| Why placement matters | Filters closest to line-rate forwarding-plane hardware are most effective and most DoS-resistant |
| "Vulnerable surface" principle | Inversely proportional to filter granularity — finer-grained filters leave less room for abuse |
| Recommended rollout method | Start permissive, audit real traffic, then tighten to an enforcing policy |
| Routing protocols' treatment | Usually the most generously allowed category — losing these has the largest operational blast radius |
| CoPP vs. transit QoS | Two structurally separate decision paths, built from the same classify/meter/police toolkit |
| Excess-traffic handling | Typically dropped outright — no lower-priority class exists for control-plane traffic to fall back to |

---

## References

**Informational**
- RFC 6192 — Protecting the Router Control Plane

**Referenced (background and what feeds into this note)**
- `3-QoS-Classification-Trust` (classifier completeness — the same principle behind CoPP's default-deny catch-all class)
- `7-QoS-Policy-Model` (the classify → meter → mark/drop toolkit CoPP reuses)
- `8-QoS-Policing` (the token-bucket mechanisms underlying CoPP's rate-limiting actions)
- `6-QoS-Class-Design` (CS6/Network Control class and its "user traffic is not allowed" rule — the transit-side complement to this note's device-side protection)
