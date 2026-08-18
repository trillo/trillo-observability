# Enterprise AI Agent Observability & Analytics
## Scheduled Feature Specifications: A/B Version Comparison, Behavioral Drift, Health & SLO Configuration

**Document Version:** 0.1 (draft)
**Companion to:** POC Requirements (PRD v1.5), SRS v1.1, Requirements Addendum +
Addendum 2 (decision log), Application & UX Design, Pre-Demo Feature Specs (A/B/C).
**Purpose:** Full PRD-style specs for the next-tranche features scheduled in
Addendum 2 — promoting decisions **AD-017 / AD-018 / AD-019** to implementable
specifications. Continues the feature-letter sequence after the pre-demo set
(A = Failure Spread, B = Security Evals, C = Alerting):

- **Feature D — Agent A/B / Version Comparison** (AD-017)
- **Feature E — Behavioral Drift Detection** (AD-018)
- **Feature F — Health-Status & SLO Configuration** (AD-019)

All three are **observability-native** — they read the telemetry/derived model,
run as **incremental sweepers/functions** (AD-006), and add no enforcement path.
Scope reminder (AD-003): observed agents are generic-framework agents; TAO is the
observability app on Trillo AOS.

---

## 1. Purpose and Scope

These three features extend the shipped seven scenarios + pre-demo A/B/C:
- **D** productizes version comparison for a **canary rollout decision** (the
  ops/FinOps reframe of the "experimentation" gap — observational, not
  experimental).
- **E** adds **statistical drift** detection (gradual degradation no single trace
  reveals).
- **F** makes the **executive health status defensible** (configurable thresholds
  + SLOs + transparency) and provides the **shared config surface** that D's
  materiality thresholds and F's SLOs both use.

Dependency note: **D's quality comparison and E's eval-drift both consume eval
scores** from the Security/eval framework (Feature B); where a customer emits no
evals, both degrade gracefully to ops-metrics only (demonstrable now on simulated
eval data).

---

## 2. Feature D — Agent A/B / Version Comparison

### 2.1 Concept

Compare **two versions of one logical agent already running in production** on
real traffic — **observational A/B, not experimental**: TAO does not generate or
split traffic and does not rerun customer prompts. The signature use case is a
**canary rollout decision**: change a prompt and/or model, deploy as **version B**
to a subset of locations, and after a short window (e.g. 24h) compare B vs A on
**normalized** reliability, latency, cost, tokens, and **quality** to decide
whether to roll B out wider.

**First principle — rate/normalized metrics, never absolute totals.** Two
versions rarely take equal traffic (a 5% canary vs 95%), so totals mislead; every
comparison metric is a rate or per-execution value, and per-version execution +
location counts are always shown.

### 2.2 Demonstration Requirements

Show how a user can:
- pick any two `agent_version`s of an agent and a comparison window (overlapping,
  or fixed lookback ≤ 30 days; windows down to hours for a canary);
- see side-by-side **normalized** reliability / latency / cost-per-exec /
  tokens-per-exec / **composite quality**, each with delta + direction;
- **expand quality** into its per-eval-metric breakdown to see *why* it moved;
- read a plain-language **rollout verdict** ("B 22% cheaper at equal quality →
  candidate to roll out" / "B cheaper but quality −6% → hold");
- record the decision (roll-out / hold / roll-back); and
- trust it at low canary volume (explicit counts + low-confidence flag).

### 2.3 POC Requirements

The platform shall:
- Compare exactly two `agent_version`s of one `agent_id` over a window
  (overlapping default; fixed lookback ≤ 30 days; minimum window in hours).
- Compute **normalized** metrics per version: error rate, latency P50–P99,
  **cost per execution**, **tokens per execution**, **composite quality score**,
  and failure-cluster signatures (a new signature in B = regression; Feature A).
- Compute a **composite quality score** as a weighted blend of eval components
  (hallucination, toxicity, PII-leak, relevance, pass rate) from `otlp_event`
  eval scores; **always decompose** it into its per-component breakdown with
  per-component A/B deltas.
- Show **per-version execution count + location count**; flag **low confidence**
  when B's sample is small (short window / few locations).
- Flag deltas that cross a **configured materiality threshold** (Feature F config)
  and produce a **rollout verdict** from them; the verdict is quality-aware
  (cheaper-but-worse is not a pass).
- Persist a **saved comparison** with its recorded decision.
- Degrade to **ops-metrics-only** when no eval data exists (quality panel shows
  "no eval data").

### 2.4 Data Model Changes

- **No new telemetry.** Comparison is a query over `agent_version` + time window,
  joining rollups (ops metrics) + `otlp_event` (eval scores).
- New AOS class **`AbTestComparison`** (`ab_test_comparison_tbl`; camelCase per
  AD-015): `agentId`, `versionA`, `versionB`, `windowType` (overlapping/fixed),
  `windowStart`/`windowEnd`, `savedBy`, `notes`, `decision`
  (ROLL_OUT/HOLD/ROLL_BACK), `verdictSummary`, timestamps.
- Materiality thresholds + composite-quality weights live in Feature F's config
  class (`HealthPolicy`/`Slo`) — **not** a parallel surface.

### 2.5 Background Functions and Agents

- **A/B Comparison function** — computes the normalized metric set (ops + quality
  composite + breakdown) for `(agent_id, versionA, versionB, window)` on demand
  from rollups + eval events; applies the materiality config to produce the
  verdict + low-confidence flag. Reads, doesn't sweep (on-demand per request);
  may cache a saved comparison's snapshot.
- Reuses **Feature A** (failure-cluster signatures per version) and the
  **Performance Regression Analyzer** (§13.3) for the reliability/regression axis.

### 2.6 User Experience

- **Entry points:** "A/B / Versions" tab on the Agent view; "compare versions"
  shortcut from a version-correlated failure cluster (Feature A) and a regression
  finding.
- **Version picker:** two dropdowns (A vs B, default = two most recent) + window
  control (overlapping / fixed ≤ 30d, hours-granularity).
- **Comparison board:** side-by-side normalized KPI cards (error rate, P95,
  cost/exec, tokens/exec, composite quality) with delta + direction (quality-aware
  coloring) and explicit per-version traffic + location counts; trend lines over
  the window.
- **Quality breakdown:** composite card expands into per-eval-metric components,
  each with its A-vs-B delta.
- **Materiality highlighting:** deltas crossing the configured threshold are
  flagged; settings link to the Feature F editor.
- **Rollout-decision banner:** plain-language verdict; user records decision;
  low-confidence badge when B's sample is thin.
- **"What changed" panel:** metrics that crossed materiality at the A→B boundary;
  deep-links to representative traces of each version + the relevant cluster.

### 2.7 Evaluation / Positioning

| Criteria | Target | Capability |
| :-- | :-: | :-- |
| Rollout decision speed | 5/5 | One board + verdict decides canary promotion. |
| Trustworthy at low volume | 5/5 | Normalized metrics + explicit counts + low-confidence flag. |
| Quality-aware, not just cost | 5/5 | Composite quality + breakdown; cheaper-but-worse ≠ pass. |
| Differentiation | 5/5 | Production, version-vs-version, cost×quality trade-off — ops persona, not dev experimentation. |

---

## 3. Feature E — Behavioral Drift Detection

### 3.1 Concept

Detect **gradual degradation over time** that no single trace makes obvious —
rising hallucination/eval-fail rate over weeks, shifting output/token
distributions, latency creep, declining eval scores. Complements Feature D
(version-vs-version) by watching **the same version drift over time**, and
complements the single-failure detection of Feature A/reliability.

### 3.2 Demonstration Requirements

Show how the platform:
- surfaces a slow-moving degradation (e.g. hallucination rate creeping up over two
  weeks) that thresholds/alerts on a single window would miss;
- shows the recent-vs-baseline distribution/trend and the drift magnitude;
- distinguishes **provider-driven** drift (a silent model update) from
  **input/data** drift where evidence allows; and
- routes the drift as a finding (and, via Feature C, an alert).

### 3.3 POC Requirements

The platform shall:
- Compare a **recent window** to a **baseline window** using **statistical**
  measures (distribution/population shift, moving-average slope, percentile
  drift) — deliberately distinct from single-failure thresholds.
- Track drift on: eval scores (hallucination/toxicity/relevance), eval-fail rate,
  output/token distributions, latency percentiles.
- Scope drift by agent / agent+version / model / tool.
- Emit a **`finding_type = DRIFT`** (severity from magnitude), a valid alert
  source for Feature C.
- Attribute cause where possible: **provider drift** (detectable as a
  `response_model` change or a step at a model-version boundary) vs **input
  drift**.

### 3.4 Data Model Changes

- Reuses **`analysis_baseline`** + rollups; no new telemetry.
- Adds **DRIFT** to the findings model (`platform_finding.finding_type`), with a
  drift-specific `evidence_json` (baseline vs recent distributions, measure,
  magnitude, suspected cause).

### 3.5 Background Functions and Agents

- **Drift Sweeper** — periodic (wider window / lower frequency than the 5-min
  health sweeper), incremental (AD-006); computes the statistical measures per
  scope against `analysis_baseline`; emits/updates DRIFT findings (UPSERT so a
  persistent drift stays one finding).
- **Baseline maintenance** — rolls the baseline window forward on a schedule so
  "normal" tracks legitimate long-term shifts without masking drift (configurable
  baseline period).

### 3.6 User Experience

- **Drift feed** (Reliability / a "Trends" area): ranked DRIFT findings — drifting
  metric, magnitude, direction, window, scope.
- **Drift detail:** recent-vs-baseline distribution/trend chart, suspected cause
  (provider vs input), affected agent/version/model, deep-links to representative
  executions across the window.
- **Executive dashboard:** a drift/posture indicator alongside guardrail
  pass-rate (a trend, not a single incident).

### 3.7 Evaluation / Positioning

| Criteria | Target | Capability |
| :-- | :-: | :-- |
| Catches slow degradation | 5/5 | Statistical baseline comparison, not single-window thresholds. |
| Cause attribution | 5/5 | Provider-update vs input-drift distinction. |
| Actionable | 5/5 | DRIFT finding → alert (Feature C) → evidence drill-down. |

---

## 4. Feature F — Health-Status & SLO Configuration

### 4.1 Concept

Make the executive health status **defensible and configurable** rather than a
hard-coded or unexplained composite (SRS §7.3 honesty requirement), and provide
the **single shared config surface** for thresholds/weights that Feature D
(materiality) and the executive dashboard (SLOs) both consume.

### 4.2 Demonstration Requirements

Show how an admin can:
- define SLOs/thresholds per agent/application (success rate ≥ X, P95 ≤ Y,
  cost/exec ≤ Z);
- see the executive status (`Healthy` / `Needs Attention` / `Critical`) change as
  those thresholds are crossed; and
- open a **"why this status?"** panel exposing the weights, thresholds, period,
  missing-data handling, and the specific metrics that drove the status.

### 4.3 POC Requirements

The platform shall:
- Compute category + overall health status from **configurable thresholds**, not
  constants (per SRS §7.3).
- Support **SLO definitions** per scope (agent/application) that drive category
  status and feed **Feature C** alerting.
- **Expose the calculation**: category weights, metric thresholds, calculation
  period, missing-data treatment, and the metrics that caused the status.
- Serve as the **shared config store** for Feature D's materiality thresholds +
  composite-quality weights (one config concept, reused).

### 4.4 Data Model Changes

- New AOS class **`HealthPolicy`** (a.k.a. SLO/threshold config;
  `health_policy_tbl`, camelCase per AD-015): `scopeType`
  (ENTERPRISE/APPLICATION/AGENT), `scopeId`, `metric`, `operator`, `threshold`,
  `window`, `weight`, `purpose` (HEALTH_SLO / AB_MATERIALITY / QUALITY_WEIGHT),
  `enabled`. Single class, `purpose` distinguishes consumers.
- The Executive Health Aggregator (§13.3) reads it instead of constants; the A/B
  Comparison function (Feature D) reads the AB_MATERIALITY / QUALITY_WEIGHT rows.

### 4.5 Background Functions and Agents

- **Executive Health Aggregator** (existing, §13.3) — modified to read
  `HealthPolicy` thresholds/weights and to record, per status, the contributing
  metrics + the config values used (so the transparency panel can render "why").
- No new sweeper; this is config + wiring.

### 4.6 User Experience

- **SLO / Threshold editor** (admin): per-scope rows (metric, operator, threshold,
  window, weight), grouped by `purpose`; enable/disable. This is the same editor
  Feature D links to for materiality/weights.
- **Health-calc transparency panel** on the Executive dashboard: a "why this
  status?" popover — contributing metrics, thresholds, weights, period,
  missing-data handling.
- **Category cards** show current value vs. its configured threshold.

### 4.7 Evaluation / Positioning

| Criteria | Target | Capability |
| :-- | :-: | :-- |
| Defensible status | 5/5 | Configurable thresholds + full calculation transparency (SRS §7.3). |
| Reuse, not sprawl | 5/5 | One config surface for health SLOs + A/B materiality + quality weights. |
| Auditor-ready | 5/5 | "Why this status" is inspectable and versionable. |

---

## 5. Cross-Cutting Requirements

- **Scale (AD-006):** E runs as an incremental sweeper; D computes on demand from
  rollups (never raw-span scans); F is config + aggregator wiring.
- **Eval dependency:** D's quality + E's eval-drift consume Feature B eval scores;
  both degrade gracefully to ops-metrics when evals are absent.
- **Shared config:** F is the single home for thresholds/weights used by D and the
  executive dashboard — no parallel config surfaces.
- **Findings/alerts:** E emits DRIFT findings; D's regression signal and F's SLO
  breaches are alert sources for **Feature C**.
- **RBAC/masking:** representative-trace drill-downs obey field-level masking
  (PRD §9, SRS §4.1).

## 6. Simulator Implications

The Telemetry Simulator must generate the data these features consume:
- **Feature D:** **per-version eval scores** with a believable A↔B quality delta
  (e.g. B cheaper/faster but slightly lower quality, or better quality at higher
  cost), across an **uneven traffic split** (canary B on few locations), so the
  normalized comparison + rollout verdict are exercisable — extends the existing
  version-boundary/regression seeding (Simulator §3.1 / §12).
- **Feature E:** a **slow-moving degradation** over the history window (e.g.
  hallucination rate creeping up, or a step at a model-version boundary
  simulating a provider update) distinct from acute failure spikes.
- **Feature F:** a spread of metric values around configurable thresholds so
  status transitions (`Healthy` → `Needs Attention` → `Critical`) are
  demonstrable.

## 7. Demo Integration

- From **Cost/FinOps or a regression finding**: open **Feature D**, compare the
  canary version B over 24h, read the cost×quality verdict, and record a
  roll-out/hold decision — the sharpest FinOps story in the set.
- From **Reliability/Trends**: **Feature E** surfaces a slow drift the dashboards
  wouldn't flag as a single incident, with provider-vs-input attribution.
- On the **Executive dashboard**: **Feature F** makes the health status
  inspectable ("why this status?") and is where the thresholds behind D/E/C live.

## 8. Sequencing & Open Items

**Sequencing (from Addendum 2 §3):** F + D first (build on shipped features, no
partner dependency), then E. (AD-020 retention/sampling is separate and
partner-gated.)

**Open items:**
- **D:** composite-quality weight defaults + materiality threshold values — tune
  on simulated data.
- **D:** minimum-sample / confidence rule for the low-confidence flag (what
  execution count is "enough" for a canary verdict).
- **E:** which statistical measure(s) per signal (distribution shift vs slope vs
  percentile drift) and baseline-window period defaults.
- **F:** default SLO set shipped out-of-the-box vs all-blank; composite-score
  formula (if a numeric score is shown at all vs qualitative-only per SRS §7.3).
