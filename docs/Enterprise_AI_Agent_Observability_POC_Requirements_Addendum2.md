# Enterprise AI Agent Observability & Analytics
## Requirements Addendum 2 — Scheduled Feature Decisions (with UI)

**Addendum Version:** 0.4 (in progress)
**Base Document:** POC Requirements v1.5; continues the decision log from
`Enterprise_AI_Agent_Observability_POC_Requirements_Addendum.md` (AD-001..AD-016).
**Platform:** Trillo AOS
**Status:** Living document — scheduled features from the gap backlog.

---

## 1. Purpose

Addendum-1 recorded requirements Q&A decisions. **Addendum-2** records the
**next tranche of features** pulled from the competitive/gap backlog after the
first three (Failure Spread, Security Evals, Alerting — AD-014) shipped. Same
entry pattern, plus a **UI** subsection per feature (the team asked to keep the UX
in the decision record).

**Backlog disposition (2026-08-18):**
- **Scheduled here:** G7 (A/B version comparison), G2 (behavioral drift), G3
  (health-status / SLO config), G4 (retention / sampling — partner-gated).
- **Moved out:** G1 (identity resolve-by-name) + G5 (OTLP→`gen_ai.*` mapping +
  coverage) → `OliverDB-Otel-Mapping-Requirements.md` (candidate OliverDB Rust
  plugin).
- **Tabled:** G6 (true inline guardrails — different product category; needs an
  enforcement point in the agent path).

Decision IDs continue globally (AD-017+) so references stay unique across both
addenda.

---

## 2. Decision Log

### AD-017 — Agent A/B / Version Comparison (observational)

- **Date:** 2026-08-18
- **Area / Topic:** Version comparison; the productized, ops-persona reframe of
  the "experimentation/regression" gap (competitive #3)
- **Relationship:** ADDS — extends §6 (Latency), §7 (Cost), §13.3 (Performance
  Regression Analyzer); builds on AD-001 (`agent_version`) and AD-014 Feature A
  (version-correlation / spread)
- **Positioning:** **Observational A/B, not experimental A/B.** Compare **two
  versions of an agent already running in production** on real traffic — TAO does
  **not** generate or split traffic, and does **not** rerun the customer's prompts
  (which we can't, for generic agents). This sidesteps what made the dev-loop
  experimentation gap a 🔴, and is arguably more valuable to ops/FinOps: "v1.4 vs
  v1.5, real traffic, real cost/latency/error/quality."
- **Decision:**
  1. Compare any **two `agent_version`s** of one logical agent over a **time
     window** — **overlapping** (both active now; default) or **fixed lookback**
     (≤ 30 days) for a completed rollout.
  2. **Rate/normalized metrics, NOT absolute totals** — error *rate*, latency
     percentiles, **cost per execution**, **tokens per execution**, eval/quality
     score. *(Two versions rarely take equal traffic — a 5% canary vs 95% — so
     absolute totals lie. Normalization is a first-class requirement, not a
     nice-to-have.)*
  3. **Metrics compared** (all already in the model): reliability (error rate,
     failure-cluster signatures by version — new signature in the new version =
     regression), latency P50–P99, cost/execution, tokens/execution, eval scores
     (when populated), and **spread** (did the new version introduce a
     wide-blast-radius CODE failure?).
  4. **Version pairing:** user **picks any two** versions (auto-select the two
     most recent as the default). 
  5. **Quality score is IN v1** (updated 2026-08-18) — not deferred. A per-version
     **composite quality score** is the headline metric, derived from the
     underlying eval scores (eval/Security framework `otlp_event`
     eval_score/eval_label). **Composite** for the verdict + KPI card;
     **breakdown** always available — the score decomposes into its per-eval-metric
     components (e.g. hallucination, toxicity, PII-leak, relevance, pass rate) so a
     reviewer sees *why* quality moved, not just that it did. The composite is a
     **weighted blend** of the components (weights configurable — see materiality
     thresholds below); where a customer emits no evals, the quality panel shows
     "no eval data" and A/B falls back to ops-metrics only. Demonstrable now on
     simulated eval data.
  6. **Materiality thresholds are tunable** — the deltas that drive the
     rollout-decision verdict (what counts as a *material* change in cost, latency,
     error rate, and each quality component; and the composite-quality weights) are
     **configurable**, not hard-coded. Reuse the AD-019 threshold/SLO config surface
     rather than a parallel one. Tune on simulated data for the demo.
  7. **Primary use case — canary rollout decision:** change a prompt and/or model,
     deploy it as **version B** to a subset of locations, and after a short window
     (e.g. **24 hours**) compare B vs A on **normalized** reliability, latency,
     **cost/exec**, **tokens/exec**, and **quality** to decide **whether to roll B
     out to more locations**. The window control supports short comparison windows
     (down to hours), and the board must read cleanly at low B-volume (few
     locations) — which is exactly why normalization + explicit per-version counts
     matter.
- **Data model:** no new telemetry; comparison is a query over `agent_version` +
  time, joining ops metrics + eval scores. Optional small **`AbTestComparison`**
  class (`agent_id`, `versionA`, `versionB`, `window`, `savedBy`, notes,
  `decision` {roll-out / hold / roll-back}) to persist a saved comparison and its
  outcome. New background: an **A/B Comparison function** that computes the
  normalized metric set (ops + quality) for `(agent_id, versionA, versionB,
  window)`.
- **Simulator implication:** to demo this, the Telemetry Simulator generates
  **per-version eval scores** (a believable quality delta between A and B — e.g. B
  cheaper/faster but slightly lower quality, or better quality at higher cost), so
  the rollout-decision story is exercisable on synthetic data. (Add to the
  simulator's version-boundary/regression seeding.)
- **UI:**
  - **Entry points:** an **"A/B / Versions"** tab on the Agent view; a "compare
    versions" shortcut from a version-correlated failure cluster (Feature A) and
    from the regression finding.
  - **Version picker:** two dropdowns (A vs B) defaulting to the two most recent
    versions; a **window control** (overlapping / fixed ≤30d).
  - **Comparison board:** side-by-side **normalized** KPI cards — error rate, P95,
    cost/exec, tokens/exec, **composite quality score** — each with **delta +
    direction** (green/red, quality-aware: cheaper-but-worse is not auto-green) and
    the **traffic volume + location count of each version shown explicitly**. Trend
    lines per metric over the window.
  - **Quality breakdown:** the composite quality card **expands** into its
    per-eval-metric components (hallucination, toxicity, PII-leak, relevance, pass
    rate, …), each with its own A-vs-B delta — so a reviewer can see the composite
    moved because, e.g., hallucination rose even though relevance improved.
  - **Materiality highlighting:** deltas that cross the **configured** materiality
    threshold are visually flagged; a settings affordance links to the AD-019
    threshold/weights editor.
  - **Rollout-decision banner:** a plain-language verdict for the canary case —
    e.g. "B is 22% cheaper and 8% faster at equal quality (±1%) over 24h across 40
    locations → **candidate to roll out**" or "B is cheaper but quality −6% →
    **hold**." User records the decision (roll-out / hold / roll-back) on the saved
    comparison.
  - **"What changed" panel:** highlights metrics that crossed a materiality
    threshold at the A→B boundary; deep-links to representative traces of each
    version and to the relevant failure cluster.
  - **Guardrail note in UI:** label it "production comparison," surface per-version
    execution + location counts, and flag **low-confidence** when B's sample is
    small (short window / few locations) so a canary isn't over-read.
- **Status:** Accepted; **promoted to full PRD-style spec — Feature D** in `Enterprise_AI_Agent_Observability_Scheduled_Feature_Specs.md`.

### AD-018 — Behavioral Drift Detection

- **Date:** 2026-08-18
- **Area / Topic:** Statistical drift over time (competitive #6)
- **Relationship:** ADDS — extends §13.3 (Performance Regression Analyzer,
  `analysis_baseline`); complements AD-017 (A/B is version-vs-version; drift is
  same-version-over-time)
- **Decision:**
  1. Detect **gradual degradation** that no single trace makes obvious: rising
     hallucination/eval-fail rate over weeks, shifting output/token distributions,
     latency creep, declining eval scores.
  2. **Statistical, not threshold:** compare a **recent window** against a
     **baseline window** using distribution/trend measures (e.g. population
     shift, moving-average slope, percentile drift) — deliberately different math
     from single-failure detection.
  3. Scope drift by agent / agent+version / model / tool; a drift signal becomes a
     **`finding_type = DRIFT`** (ties into Alerting AD-014 Feature C as a source).
  4. Distinguish **provider-driven drift** (model silently updated — detectable as
     `response_model` change or a step at a model-version boundary) from
     **input/data drift** (query-pattern shift) where evidence allows.
- **Data model:** reuses `analysis_baseline` + rollups; adds drift findings
  (extends the findings model). New background: a **Drift Sweeper** (periodic,
  wider window than the 5-min health sweeper) computing the statistical measures
  and emitting DRIFT findings; incremental (AD-006).
- **UI:**
  - **Drift feed** (in Reliability / a "Trends" area): ranked DRIFT findings with
    the metric that's drifting, magnitude, direction, and window.
  - **Drift detail:** the recent-vs-baseline **distribution/trend chart**, the
    suspected cause (provider vs input), affected agent/version/model, and
    deep-links to representative executions across the window.
  - **Executive dashboard:** a drift indicator alongside guardrail pass-rate
    (posture, not a single incident).
- **Status:** Accepted; **promoted to full PRD-style spec — Feature E** in `Enterprise_AI_Agent_Observability_Scheduled_Feature_Specs.md`.

### AD-019 — Health-Status Thresholds & SLO Configuration

- **Date:** 2026-08-18
- **Area / Topic:** Making the executive health status defensible (SRS §7.3, §4.4;
  deferred there)
- **Relationship:** CLARIFIES / ADDS — §10 (Executive Dashboard), §7.3 health
  calc, AD-009 status model
- **Decision:**
  1. Category and overall health status (`Healthy` / `Needs Attention` /
     `Critical`) are computed from **configurable thresholds**, not hard-coded or
     an unexplained composite — per the SRS honesty requirement (§7.3).
  2. Support **SLO definitions** per agent/application (e.g. success rate ≥ 99.5%,
     P95 ≤ X, cost/exec ≤ Y) that drive category status and feed Alerting
     (AD-014 C).
  3. **Transparency:** when a composite/score is shown, expose category **weights,
     metric thresholds, calculation period, missing-data treatment, and the
     metrics that caused the status** (SRS §7.3 verbatim).
  4. **Shared config surface:** this same threshold/weight editor is the home for
     **AD-017's A/B materiality thresholds + composite-quality weights** — one
     config concept (scope, metric, operator, threshold, window, weight), reused,
     not a parallel surface.
- **Data model:** a **`HealthPolicy` / `Slo`** config class (scope, metric,
  operator, threshold, window, weight). The Executive Health Aggregator (§13.3)
  reads it instead of constants; the A/B Comparison function (AD-017) reads the
  materiality/weight rows.
- **UI:**
  - **SLO / Threshold editor** (admin): per-scope rows (metric, operator,
    threshold, window, weight), enable/disable.
  - **Health-calc transparency panel** on the Executive dashboard: a "why this
    status?" popover listing the contributing metrics, thresholds, weights, period,
    and missing-data handling.
  - Category cards show current value vs. its configured threshold.
- **Status:** Accepted; **promoted to full PRD-style spec — Feature F** in `Enterprise_AI_Agent_Observability_Scheduled_Feature_Specs.md`. (Low build — config surface + aggregator wiring.)

### AD-020 — Retention, Sampling & Cost Controls  ⚠️ partner-gated

- **Date:** 2026-08-18
- **Area / Topic:** Data-lifecycle + storage-cost transparency (competitive #8)
- **Relationship:** ADDS — §11.1/§4.2 (retention), Gap-Analysis §3.E (OliverDB)
- **Decision:**
  1. **Configurable retention by data class** (traces / logs / prompts / outputs /
     metrics / audit) — independent periods; preserve **aggregate rollups when raw
     payloads expire** (SRS §4.2).
  2. **Sampling / down-sampling** of high-volume raw telemetry by policy
     (application / agent / severity), with **transparent** reporting of what is
     sampled/aged and when (the competitive gap: cost/completeness trade-off made
     visible).
  3. **Cost/volume transparency:** show ingest volume + storage footprint by
     dimension so the trade-off is legible before/after go-live.
- **Partner dependency (OliverDB):** **who enforces** retention/sampling — at
  ingest (OliverDB) or app-side — and the **config interface** are **open items in
  Gap-Analysis §3.E**. This feature is **gated on the OliverDB discussion**;
  Trillo owns the **policy + UI + reporting**, OliverDB likely owns **enforcement**
  on the raw store.
- **Data model:** a **`RetentionPolicy`** / **`SamplingPolicy`** config class;
  volume/footprint metrics into rollups.
- **UI:**
  - **Data-lifecycle editor** (admin): retention period per data class; sampling
    rules per scope; preview of resulting volume/cost.
  - **Storage & ingest transparency** panel: volume by application/agent/data
    class over time, what's sampled/aged, projected footprint.
- **Status:** Accepted for **policy + UI + reporting**; **enforcement gated on
  OliverDB** (Gap-Analysis §3.E). Sequence after the partner conversation.

---

## 3. Sequencing (recommended)

1. **AD-019** (health/SLO config) + **AD-017** (A/B) — build on shipped features,
   high visibility, no partner dependency.
2. **AD-018** (drift) — statistical layer on existing baselines.
3. **AD-020** (retention/sampling) — **after** the OliverDB discussion resolves
   enforcement ownership.
- **G1/G5** (mapping/identity) proceed in parallel via the OliverDB doc.
- **G6** (inline guardrails) remains **tabled** — a category decision, not backlog.

## 4. Change History

| Version | Date | Summary |
| :-- | :-- | :-- |
| 0.1 | 2026-08-18 | Created; AD-017 (A/B version comparison), AD-018 (drift), AD-019 (health/SLO config), AD-020 (retention/sampling, partner-gated) with UI specs. G1/G5 → OliverDB mapping doc; G6 tabled. |
| 0.2 | 2026-08-18 | AD-017: quality score promoted into v1 (not deferred); added canary-rollout use case (change prompt/model -> deploy B -> compare 24h normalized -> roll-out decision), rollout-decision banner + low-confidence guard, simulator per-version eval-score seeding. |
| 0.3 | 2026-08-18 | AD-017: composite quality score (headline) + always-available per-eval-metric breakdown; materiality thresholds + composite-quality weights tunable via the shared AD-019 config surface. |
| 0.4 | 2026-08-18 | AD-017/018/019 promoted to full PRD-style specs (Features D/E/F) in Scheduled_Feature_Specs.md; statuses updated to point at them. |
