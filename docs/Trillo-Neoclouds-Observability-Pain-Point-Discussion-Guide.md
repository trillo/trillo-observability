# Neoclouds Observability — Pain-Point Discussion Guide

**Purpose:** a field guide for the **Cursor / Aurora** conversations. Goal is to *listen* — surface real
pain in their words — and come away with the handful of facts that ground the detailed requirements. Feeds
`Trillo-Neoclouds-Observability-PRD.md` → the `...POC_Requirements_Final` spec.

**How to use:** don't run it top-to-bottom like a survey. Read the room, follow the pain. Each section notes
**what it decides** in the PRD so you can steer toward the answers that matter. Capture **verbatim** quotes —
"our tenants find failures before we do" is worth more than a checkbox.

---

## 0. First: read whether they're an *operator* or a *tenant*

The single most important thing to establish early, because it flips half the questions:

- **Do they *run* a GPU fleet** (provider / platform team) → **operator** lens (our V1 buyer).
- **Do they *consume* GPU capacity** (ML/eng team renting or running jobs) → **tenant** lens (our V2 user).
- Many will be **both** (run a private cluster *and* buy neocloud capacity). Note which hat each pain wears.

> *Ask:* "Walk me through your setup — do you own the hardware, rent it, or both? Who owns the fleet vs. who
> runs the jobs?"

**Decides:** which persona the POC leads with; whether this contact is a design partner (operator) or a
voice-of-customer (tenant).

---

## 1. Topology & fleet (priority 1)

- How do you know what's in your fleet today — inventory, and how it's *connected* (rack, NVLink domain,
  fabric)? Spreadsheet, CMDB, Prometheus labels, nothing?
- Do you have any view of the **network fabric** (IB/RoCE, leaf/spine), or is it a black box?
- When something's wrong, can you place it *on the fleet* — "which rack, which switch, which NVLink domain"?

**Decides:** how much topology auto-discovery vs. authoritative-source reconcile V1 needs; whether fabric
links earn more than "light ingestion."

---

## 2. Failure signals — the worst recurring pain (priority 2)

The core of the meeting. Get concrete.

- **What's your worst recurring hardware failure mode?** How often? What does it cost you when it hits?
- Walk me through the **last bad-GPU incident** — how did you find out, how long to identify the culprit,
  what did you do?
- Validate the detector catalog — which of these have bitten you?
  - Double-bit ECC / HBM row-remap exhaustion
  - GPU falling off the bus (XID 79)
  - **Silent Data Corruption** — loss goes NaN hours later, no hardware alert *(often the scariest — probe it)*
  - NVSwitch / Fabric-Manager faults
  - Silent thermal throttling / power capping (the "healthy but slow" node)
  - NCCL/RCCL ring deadlocks — hundreds of GPUs idle waiting on one peer
  - GPU starvation — VRAM full, compute idle (data-loader / storage bound)
  - Zombie actors holding VRAM after a job dies
- Do failures reach your **tenants/users before you see them**? How often?

**Decides:** which detectors are V1 must-haves vs. nice-to-have; whether SDC (V1 suspect-flag → V2
loss-confirm) resonates enough to headline; false-positive tolerance.

---

## 3. Blast radius & job correlation (priority 6)

- When a GPU/node/link fails, how do you figure out **which jobs and which tenants** are affected — and how
  long does that take today?
- What **job metadata** do you expose, and from where — **Slurm** (training)? **K8s** (inference)? Both?
- Could you map a device back to a job/tenant *right now* if I asked? What would you have to join by hand?

**Decides:** the job→tenant→device join that powers blast radius; the exact Slurm/K8s fields the connectors
must read; confirms "both schedulers in V1, Slurm first."

---

## 4. Utilization — the number you trust (priority 3)

- What **utilization number do you actually trust** today — and do you believe it? (Probe `GPU_UTIL` /
  nvidia-smi vs. real SM occupancy / MFU.)
- How much of your fleet do you think is **idle or underused** right now? How would you even know?
- For an operator: what's idle capacity *worth* — is this a margin conversation or a capacity one?

**Decides:** whether the "MFU vs the GPU_UTIL lie" headline lands; how prominent the utilization/margin story
is in V1 vs. billing in V2.

---

## 5. Performance questions people keep asking (priority 4)

- What's the recurring **"why is my job slow / why did it stall"** question — who asks it (ops? the ML
  team?), and how is it answered today?
- How much of the answer needs *inside-the-job* signal (loss, gradients, step time) vs. infra signal you
  already own?

**Decides:** the V1 scoped-chat question set (answerable from provider-native data) vs. what genuinely needs
the V2 framework tier / perf-Q&A copilot.

---

## 6. Prediction & alerting (priority 5)

- Do you want to be **warned before** a GPU dies, or is catching it fast enough enough?
- How bad is **alert fatigue** today? What channels do alerts go to (Slack / Teams / PagerDuty / webhook)?
- What would make an alert *trustworthy* enough to act on at 3am?

**Decides:** V1 reactive degradation-cordon vs. V2 real prediction; alert routing + dedup requirements;
false-positive bar.

---

## 7. Remediation — how open are you to automation? (V2 trust ladder)

Directly validates the actuation posture.

- Today, when a node is bad, **who pulls the trigger** to drain/cordon/quarantine — and how?
- Would you ever let a tool **do that automatically**? What would you need to trust it — dry-run? approval
  gates? policy limits? blast-radius checks?
- Where's the line — "recommend and I'll click" vs. "act within my guardrails" vs. "never touch my control
  plane"?

**Decides:** confirms V1 = recommend-only; shapes the V2 trust-ladder rungs and per-operator opt-in.

---

## 8. Tooling, deployment & data (context)

- What do you use today — Prometheus/Grafana, DCGM dashboards, Datadog, homegrown? What do you love / hate?
- **Data residency:** does telemetry/tenant data have to stay in your environment? (Validates BYOC.)
- **Buy vs. build:** are you building this yourself? What would make you buy instead?

**Decides:** integration surface; confirms BYOC / in-environment; competitive framing.

---

## 9. Customization & fit — "nothing off-the-shelf fits our fleet"

Neoclouds differentiate by being specialized — bespoke hardware (NVL72, mixed fabrics), custom schedulers,
custom SLAs and billing. Their observability needs mirror that, and rigid tools can't keep up. This is where
Trillo's **model-driven customization** is a real wedge — but in a *discovery* conversation it's a **probe
first, a pitch last**. Surface the pain before you name the capability.

**Probe the pain (ask these):**
- What's **unique about your fleet** that off-the-shelf tools don't handle — custom hardware, scheduler,
  metrics, failure signatures, SLAs?
- What did you have to **build yourself** because Datadog / Grafana / DCGM couldn't — and what does it cost
  you to *maintain*?
- Are there **custom metrics, detectors, or correlations** you compute (or wish you could) that are specific
  to how you run?
- How custom are your **billing / SLA / chargeback** rules?

**Then — only after a fit-pain surfaces — articulate the capability, outcome-first:**
- *Outcome:* "The tool bends to your fleet, not the other way — add your own metrics, detectors,
  correlations, dashboards, and billing rules as **functions**, hot-deployed, no fork."
- *Proof (the HOW, in service of the outcome):* "Those functions run **distributed on pods / micro-VMs**, so
  your custom logic executes safely and **at fleet scale** — you're not standing up a data pipeline to do it."
- *Velocity (the discovery-meeting payoff):* "It also means we can **tailor the POC to your fleet in days** —
  your metrics, your scheduler, your quirks — not wait a product quarter."

**Why it's a clean wedge:** it beats **build-your-own** (no maintenance burden) *and* **buy-rigid**
(Datadog/Grafana bend to a fixed model; we bend to yours).

**Decides:** whether customization is a headline or a footnote for this buyer; which custom functions the POC
should showcase; how hard to lean on the model-driven / AOS story.

**Watch-out:** don't open with architecture. Lead with their pain; deploy "pods/micro-VMs" only as *proof it
scales and stays isolated*, never as the headline. Jargon before pain turns discovery into a demo.

---

## 10. Value & economics (close)

- If we solved the #1 pain you named, what's that **worth** — downtime avoided, margin recovered, people-time
  saved?
- Who signs off on a tool like this — you, or someone else?
- Would you run a **free 4-week POC in your own cloud**? What would make it a clear win for you?

**Decides:** pricing framing; the POC definition-of-done; the economic buyer.

---

## Listening checklist (bring back from each meeting)

1. Operator, tenant, or both — and the design-partner verdict.
2. Their **#1 recurring failure mode**, in their words, with a cost.
3. How they attribute a bad-GPU incident to a job/tenant **today**, and how long it takes.
4. The utilization number they trust (and whether they believe it).
5. Slurm vs. K8s — what job metadata they actually expose.
6. What they had to **build themselves** because no tool fit — and what it costs to maintain.
7. Their honest openness to **automated remediation**, and the guardrails they'd need.
8. Data-residency / BYOC requirement — hard or soft?
9. One verbatim quote we could put in front of the next prospect.

---

## Three hooks to lead with (if they stall)

If the conversation needs a spark, these are our sharpest, most-differentiated angles — all V1:

- **Silent Data Corruption** — "your loss goes NaN three hours in and *nothing* in your infra logged a
  thing. We flag the mathematically-rogue GPU."
- **The utilization illusion** — "nvidia-smi says 90% and half your fleet is actually idle. We show real
  MFU, per tenant — that's margin you're leaving on the floor."
- **It bends to your fleet** — "whatever's bespoke about your setup — custom metrics, detectors, billing —
  you add it as a function that runs at fleet scale, hot-deployed. And we'll shape the POC to *your* fleet in
  days." *(Deploy only after a fit-pain surfaces — see §9.)*
