# Reading the Attack Surface: A Spatial, Progressive, WebXR-Native Visualization of Autonomous-Pentest Results

**Research notes — documentation only, no source code.**
Version 1. Status: design description and evaluation plan; not an empirical study.

---

## 1. Problem statement

An autonomous penetration test produces two data structures: an **inventory of
assets** (hosts, services, applications, endpoints, data stores) related by a
connectivity/containment graph, and a **set of findings** (vulnerabilities), each
with a severity, a technique reference, an affected endpoint, and evidence. The
industry's default way to present this is a node-link diagram with the findings
attached as annotations. In practice that artifact fails its readers in four
recurring ways.

1. **Findings look random.** When a finding's asset linkage is weak or missing,
   naïve attachment heuristics (e.g. match by host) collapse many findings onto
   whichever node they can be tied to. All 57 findings of a demo target sharing
   one IP land on a single node; the rest of the graph reads as empty. The
   distribution the reader sees is an artifact of the attachment code, not of the
   security reality.

2. **Everything piles on one asset.** The corollary of (1): a single "application"
   node carries a heap of dozens of findings while the database, the exposed key,
   the file share and the metrics endpoint — each genuinely at risk — appear
   clean. The map actively misleads about *where* the risk is.

3. **Attack paths are implicit.** The single most decision-relevant question —
   *how does an attacker get from the internet to my most sensitive data?* — is
   not shown. It lives in the analyst's head as they trace edges manually. A
   node-link diagram encodes adjacency, not the **kill chain** that crosses it.

4. **Expert-only.** The result is legible to someone who already understands the
   stack, the severities, and the ATT&CK matrix. A CISO, a developer, or a
   customer sees a hairball. The artifact cannot carry its own explanation.

These are not cosmetic complaints. Each one changes the decision a reader makes.
A visualization that misrepresents *where* risk concentrates, or that hides the
path an adversary would take, is worse than a table.

## 2. Design goals

- **Truthful attachment.** Every finding is bound to the *most specific asset it
  actually concerns*, deterministically, and *nothing is fabricated* to fill the
  picture.
- **Paths as first-class objects.** The attacker's route from the exposed entry
  point to each sensitive asset is reconstructed and drawn.
- **Self-explaining.** A first-time viewer can understand the headline risk
  without training, through a guided narrative, while an expert can still drill to
  raw evidence.
- **Spatial and immersive by option.** The same scene is navigable in 3-D and can
  be entered in VR, because attack-path reasoning is a spatial task — but VR must
  never be a prerequisite, and must not depend on a proprietary SDK.
- **Faithful to the product's visual language**, not a separate toy.

## 3. Approach

### 3.1 Deterministic finding → asset attachment

We replace host-matching with a deterministic resolution order that prefers the
*strongest evidence of which asset is affected*:

1. **Endpoint route signature.** The finding's endpoint (URL/path) is the most
   concrete evidence. Route signatures map to the specific asset:
   configuration/admin disclosure → the config endpoint; `/ftp` → the file share;
   an exposed key path → the key/secret asset; a metrics path → the metrics
   endpoint.
2. **Substance of the finding.** A leak of credentials, password hashes, user
   records or database dumps is attached to the **data store**, even when the
   request nominally hit the web tier — because the asset genuinely at risk is the
   data, not the router in front of it.
3. **Declared asset id** (`node_id`) when present and valid.
4. **Host\:port** of the endpoint, then host.
5. A conservative fall-back to the primary application, then the target — so that
   **no finding is silently dropped**, without ever inventing an asset.

The resolution runs **server-side, on every request**, and is idempotent: the
graph is correct for any campaign regardless of how completely the collection
agents populated asset links. This is the property that lets us *guarantee* the
map is coherent without re-running the assessment.

> Consequence: on the same 57-finding demo the pile of dozens collapses into a
> readable spread across the application, the database, the file share, the
> exposed key and the metrics endpoint — a distribution that reflects the
> assessment, not the attachment code.

### 3.2 Attack-path reconstruction

We treat the asset graph as an undirected reachability graph and the ATT&CK
technique references on findings as a partial ordering over the **kill chain**
(Reconnaissance → Initial Access → Execution → Privilege Escalation → Credential
Access → Discovery → Collection → Impact). From the **internet-facing entry
node** that is actually wired into the graph, we compute a shortest path
(breadth-first search) to **each asset that carries a high/critical finding**.
The union of these paths — plus the tactic-ordered chain — is the set of attack
paths drawn on the map. Paths are derived from *real* edges and *real* technique
mappings; we never draw an edge that does not exist.

This makes the decision-relevant question answerable at a glance: the reader sees
*every* route that reaches sensitive data, and can follow one from entry to
impact.

### 3.3 Orbital exposure encoding

Assets are placed on **concentric rings by exposure zone** — an outer ring for
what is internet-facing, a middle ring for applications and services, an inner
ring for data and secrets. Ring membership answers "how exposed is this?" by
*position*, before any label is read (a preattentive spatial cue). Severity is
encoded on each asset by a **triple**: a colored ring, a size step, and — for
high/critical — an explicit glyph, so that severity survives color-blindness and
low-contrast displays. A small **count badge** carries the number of findings per
asset, restoring the per-asset density that the pile-up had destroyed.

The disc is rendered facing the viewer in a 3-D scene (three.js), tilt-navigable,
so that the exposure rings read as depth and the attack paths as trajectories
crossing zones.

### 3.4 Progressive-disclosure guided narrative

We apply Shneiderman's *overview first, zoom and filter, details on demand*
directly:

- **Overview:** a briefing card states the one-sentence headline (how many
  findings across how many assets, how many paths reach sensitive data) with a
  severity distribution bar.
- **Guided story:** an optional automatic walkthrough steps along the attack path,
  one plain-language sentence per hop ("reachable from the internet → the web
  application's authentication is too weak → …"), dimming everything off-path so
  attention follows the kill chain.
- **Details on demand:** selecting any asset opens its full vulnerability list
  with severities, scores, categories and, one click deeper, the raw
  finding/technique/endpoint evidence.

A first-time viewer can consume the narrative; an expert can ignore it and drill.
The same scene serves both without a mode switch imposed on either.

### 3.5 WebXR with no vendor SDK

The visualization is a web page. Where the browser exposes an immersive session
(`navigator.xr` reporting `immersive-vr` support), the *same* scene is entered in
a headset; where it does not, the page explains the conditions and remains fully
usable in 2-D. There is **no Meta or Apple SDK**, no app-store binary, no native
build: delivery is two ordinary containers behind a browser. VR is an optional
lens on the identical data, not a separate product — which is what makes it
deployable in environments (SOCs, regulated networks) that cannot install vendor
runtimes.

## 4. Iterations, and what each taught us

The current form is the residue of a sequence of discarded ones. We record them
because the failures were informative.

- **Radial dial / "cadran".** A rotating dial established that a *facing* disc
  with graduated rings reads as an instrument, not a chart — spatial exposure
  encoding was legible. But a pure aesthetic device carried no analytic weight.
- **Tactical / flat layouts.** Table-flat network layouts made adjacency clear
  and paths invisible — reinforcing that adjacency ≠ path, and that flatness
  hides the very thing the analyst needs.
- **Orbital variants.** Moving assets onto exposure rings recovered "where is the
  risk" as position. This is retained.
- **A UX lab of three philosophies — Minimal, Guided, Spatial.** We built three
  genuinely distinct interaction models over one engine and tested them on
  non-experts (informally, including a lay reader). The lesson was blunt: the
  *Minimal* and *Spatial* variants were admired but not *understood* by
  non-experts; the *Guided* variant was. Choice was a burden, not a feature.
- **Guided-only distillation.** We removed the variant switcher and shipped the
  guided experience alone. Restraint improved comprehension more than any added
  affordance had.
- **Charte alignment.** A final pass discarded a bespoke palette (a sky-cyan
  accent, decorative gradients) in favor of the product's own type and color
  system. Consistency with the surrounding application did more for perceived
  quality and trust than novelty did.

The through-line: **every element must earn its place by changing a reader's
understanding.** Ornament that impresses experts and loses novices is a
regression.

## 5. HCI and accessibility reasoning

- **Triple-encoded severity.** Color + size + glyph, so severity is never carried
  by hue alone (deuteranopia/protanopia safe, and robust on poor displays). This
  is a direct application of the redundant-encoding principle.
- **Preattentive spatial encoding.** Exposure by ring position is read before
  labels, exploiting position as one of the strongest preattentive channels.
- **Keyboard navigation.** Assets are traversable and selectable by keyboard, in
  severity order.
- **Accessible DOM mirror.** The canvas is shadowed by an off-screen,
  screen-reader-addressable list of assets with their risk and finding counts, so
  the scene is not opaque to assistive technology; an aria-live region announces
  selection.
- **Reduced motion.** Animation (camera tweens, the traveling attack particle)
  respects `prefers-reduced-motion`; the static frame is fully informative.
- **Plain-language narrative.** The guided story reformulates each finding's
  category into one non-jargon sentence, so comprehension does not require the
  ATT&CK matrix.

Accessibility here is not a compliance afterthought; it is the same goal as the
whole design — *make the state readable by someone who is not the author.*

## 6. Why VR

Attack-path reasoning is a spatial cognition task: an analyst mentally traverses
a graph, holding a route in working memory while comparing alternatives. A
first-person, room-scale rendering offloads that traversal onto the visual and
proprioceptive systems — the analyst *walks* the kill chain rather than tracing it
on a plane. Our hypothesis (untested, stated as such) is that immersive
presentation reduces the effort of path-tracing and comparison for review-style
tasks in a SOC, and improves recall of which assets sit on which path. The design
constraint that makes this worth pursuing operationally is delivery: because it is
WebXR with no vendor SDK, the same artifact a customer opens in a browser is the
one an analyst enters in a headset. There is no second build to maintain and no
runtime to get approved.

## 7. Evaluation plan (not yet run)

We have **not** conducted a user study. A defensible evaluation would measure:

- **Time-to-understand.** Time for a non-expert to correctly state the headline
  risk and the primary attack path, guided narrative vs. a conventional
  node-link + table baseline.
- **Path-tracing accuracy.** Given a target asset, can the participant name a
  correct entry-to-target path, and how long does it take — 2-D vs. VR.
- **Attribution correctness.** Does the reader correctly identify *which* assets
  carry the most/most-severe findings (the metric the pile-up destroys)?
- **Expert drill-down cost.** For an analyst, time to reach the raw evidence for a
  named finding.
- **Accessibility conformance.** Screen-reader task completion; color-blind
  simulation legibility; keyboard-only operation.

We would report effect sizes and confidence intervals, or we would report
nothing. No numbers appear in this document because none have been measured.

## 8. A vision for cybersecurity tomorrow

Two curves are crossing. Autonomous agents are beginning to *conduct* assessments
at machine speed and breadth; humans remain the arbiters of *meaning, priority and
consent*. The bottleneck is moving from *finding* to *understanding*. Our thesis
is that the interface between an autonomous offensive engine and a human decision
is itself a research problem, and that it is a **spatial** one: the natural home
for "how would I be attacked, and what do I fix first" is a navigable model of the
attack surface, delivered where the human already is — the browser — and enterable,
when the stakes justify it, as a space the analyst can stand inside. The machine
maps the terrain deterministically and truthfully; the human walks it, judges it,
and decides. That division of labour — autonomous discovery, spatial
human-in-the-loop comprehension — is the direction this work points toward.

---

## Concepts referenced (by name, honestly)

This work builds on established ideas rather than claiming to originate them:
**attack graphs** and kill-chain modeling; the **MITRE ATT&CK** framework;
Shneiderman's **"overview first, zoom and filter, details on demand"** information-
seeking mantra; **preattentive processing** and **redundant (multi-channel)
encoding** from perception and visualization research; **progressive disclosure**
from interaction design; and the open **WebXR** device API. We cite these by name
and deliberately do **not** attach specific page numbers, DOIs or dates we have not
verified.

## Suggested citation

> DarkMoon Research. *Reading the Attack Surface: A Spatial, Progressive,
> WebXR-Native Visualization of Autonomous-Pentest Results.* Research notes, v1.
> (Documentation; design description and evaluation plan.)
