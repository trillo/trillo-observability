# Enterprise AI Agent Observability & Analytics
## Competitive Positioning

**Document Version:** 0.1 (draft)
**Companion to:** POC Requirements v1.5 + Requirements Addendum (decision log) +
Simulator Requirements + Application & UX Design.
**Reference input:** "Strategic Roadmap: High-Impact Features for Trillo
Observability" (Phoenix × Galileo review) — the competitive frontier defined by
Phoenix (Arize) and Galileo plus production-agent must-haves.

Product under assessment: **TAO** — the Trillo AI-Agent Observability application
(this POC + its production successor), per the PRD and Addendum decisions
AD-001..AD-011.

---

## 1. Framing — different centers of gravity

Phoenix and Galileo are **developer-inner-loop** tools: tracing, evaluations,
datasets, experiments, prompt playgrounds, guardrails. Their strength is the
build/eval/experiment loop.

TAO's center of gravity is **enterprise operations & fleet**: inventory,
ownership, cost / chargeback-showback, reliability / SRE, governance & audit,
executive health — across **thousands of instances and locations** — with
**multi-agent topology** as a first-class concern.

Most of the reviewed frontier pulls toward the competitors' strengths (dev loop +
production safety); two items land squarely on TAO's (multi-agent, fleet-scale
operations). We should compete where our spec is strongest and make deliberate
choices about the rest.

## 2. Scorecard

Legend: 🟢 Strong (differentiator) · 🟡 Partial · 🔴 Gap

| # | Frontier capability | TAO | Grounding in our spec |
| :-- | :-- | :-: | :-- |
| 5 | **Multi-agent / A2A observability** | 🟢 | AD-002/004/005: agent-tree projection, nearest-ancestor attribution, cross-boundary trace-linking (AD-004 distributed case), agent→agent dependencies, and **spread**-based code-vs-deployment triage. The review names this the differentiation opportunity — already designed. |
| 1 | **Alerting & on-call routing** | 🟡 | Detection exists (findings + thresholds + baselines, §13); §13.4 *deliberately scoped alerts out of the POC*. Routing is a wiring exercise — Trillo AOS has webhooks, scheduler, email/SMS. |
| 4 | **Security evals (prompt injection / jailbreak)** | 🟡 | Eval framework exists (toxicity / hallucination / PII as `otlp_events` with score/label); a dedicated **adversarial-input** eval category is a model add + detector work. |
| 6 | **Behavioral drift detection** | 🟡 | `analysis_baselines` + Performance Regression Analyzer + eval-scores-over-time exist; **statistical drift** (distribution shift, gradual eval decline) is a layer on top, not yet specced. |
| 8 | **Retention / sampling / cost controls** | 🟡 | AD-006 (columnar + incremental sweepers + rollups) is genuinely scale-aware, and rollups *enable* tiered retention; explicit **sampling / downsampling / aging / cost-transparency** policy is a gap. |
| 7 | **One-line auto-instrumentation** | 🟡 | Strength: **OTel-native** (GenAI semconv, AD-004/010) → any OpenInference/OpenLLMetry-instrumented agent already works, no proprietary lock-in. Gap: no *we-provide-it* one-line libs; the **emit-`agent_id`** requirement (AD-001a) adds friction — the deferred **resolve-by-name** is the mitigation. |
| 2 | **True inline guardrails** | 🔴 | §9 has versioned policy + BLOCK, but that's **observability-triggered / post-hoc**. True inline prevention runs *in the agent's request path* — an **enforcement-point product (SDK / proxy)**, architecturally distinct from observability. We observe the agents; we're not in their path. |
| 3 | **Prompt / dataset experimentation + regression** | 🔴 | Not in the PRD; table-stakes for the *dev* persona (both competitors lead). AOS has authoring/testing primitives (Designer, agent-test) but no closed-loop **dataset → rerun → no-regression** workflow — and for generic (customer-owned) agents we don't own their prompts. |

## 3. Where we win (differentiators)

1. **Multi-agent / A2A + fleet triage.** The most-designed part of our spec, and
   precisely the gap the review flags as differentiation. Agent-vs-instance-vs-
   location + **spread → code-vs-deployment** is an enterprise-fleet insight the
   dev-centric tools don't emphasize.
2. **Platform-as-app, not just dashboards.** RBAC, governance *workflows*, audit
   hash-chain, chargeback/showback, scheduler-driven sweepers, and **agentic
   investigation** ("functions own facts, agents own interpretation" — SRE Root
   Cause / Optimization agents). Competitors surface problems; TAO can also *act*.
3. **Enterprise IT-governance posture** — inventory / ownership / cost-center /
   policy / audit — vs. a dev-tool posture.
4. **Existing platform primitives shrink several "gaps" to wiring** (see §5).

## 4. Where we're exposed (gaps)

- **Dev inner-loop (item 3)** and **inline guardrails (item 2)** are the two
  biggest, most-architectural lifts — item 2 is a different product *category*
  (enforcement), item 3 a different *persona* (dev). Decide deliberately whether
  to chase or cede.
- **Security evals (4)** and **drift (6)** are cheaper to add given our eval +
  baseline foundations, and are increasingly table-stakes.
- **Instrumentation polish (7)** is adoption-critical; OTel-native is the right
  bet, but the frictionless-onboarding story needs **resolve-by-name** pulled
  forward.
- **Retention/sampling (8)** — architecture is ready; the *policy/controls* +
  cost transparency are the missing product surface.

## 5. Platform primitives that shrink the gaps

Several "gaps" are wiring exercises because Trillo AOS already provides the
primitive:

| Gap | Existing AOS primitive | Effort |
| :-- | :-- | :-- |
| Alerting / on-call (1) | Webhooks (plan-73), scheduler, email/SMS templates | Wire findings → routing |
| Experimentation seed (3) | Agent authoring (Designer), agent-test, prompt templates | Add dataset/rerun loop |
| Governance actions (2, post-hoc) | Policy model, KMS keyring, audit | Present; inline is separate |
| Sweepers/rollups (6, 8) | Scheduler + incremental sweeper contract (AD-006) | Add drift math / retention policy |
| Security-eval category (4) | `otlp_events` eval framework | Add category + detectors |

## 6. Strategic read

- **Lead with multi-agent + enterprise-ops / fleet / governance** as the wedge —
  where the spec is strongest and Phoenix/Galileo are weakest.
- **Close the cheap, high-visibility gaps** using primitives we already have:
  **alerting via webhooks (1)**, a **security-eval category (4)**, and **drift on
  top of existing baselines (6)**.
- **Treat experimentation (3) and inline guardrails (2) as explicit product-
  strategy decisions**, not backlog items — one is a new persona, the other a new
  product category.

## 7. Caveats

- Assessment is against a *review summary* of the frontier, not a hands-on
  feature audit of current Phoenix/Galileo builds — treat competitor specifics as
  directional.
- TAO status reflects the **spec** (PRD + AD-001..011), not shipped code; several
  🟡/🔴 items are "not specced/ built for the POC," not "impossible."
- Scope reminder (AD-003): the observed agents are generic-framework (customer-
  deployed) agents; TAO is the observability application on Trillo AOS.
