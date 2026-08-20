# Trillo AI Observability — Feature List (brochure source)

A comprehensive, bullet-point feature catalog with one-line descriptions, drawn
from the PRD (v1.5), the feature specs (A–F), and the **Module Reference** (the
as-built console). Organized by the console's own navigation — **Monitor · Spend ·
Govern** — plus the cross-cutting intelligence and platform layers.

> This is the **source list** (longer than three pages by design — trim for the
> brochure). Each bullet is one feature + a one-line description.
> **Naming:** the console shows "**Trillo AI Observability**"; internal docs also
> use "Trillo Agent Observability / TAO" — pick one for the brochure.

---

## Positioning (one line)

Enterprise observability, governance, and cost control for fleets of AI agents —
turning OpenTelemetry into answers about reliability, latency, spend, drift, and
compliance, with the evidence always one click away.

## What makes it different (lead differentiators)

- **Fleet-scale root-cause class** — classifies every failure cluster as **CODE vs DEPLOYMENT vs DEPENDENCY** at a glance, the triage pure-trace tools can't do.
- **Geographic health** — a live map of every runtime instance by store/zone, colored worst-of so one bad store can't hide.
- **De-duplicated alerting** — one issue across 47 instances is **one** alert carrying the blast radius, not 47.
- **Behavioral drift detection** — statistical early warning for slow degradation no single trace reveals.
- **Defensible health** — a Healthy/Needs-Attention/Critical verdict that always traces to observed-vs-target SLO evidence.
- **Chargeback/showback** — per-execution cost attributed to teams and cost centers, reconciled across every cut.
- **AI investigation copilot** — an SRE can investigate the fleet conversationally (Claude Code + MCP), report filed back into the product.
- **Deterministic facts, agentic reasoning** — background functions own the numbers; AI agents own the explanation.

---

## MONITOR

### Dashboard — fleet-wide health cockpit
- **Headline metrics** — success rate, P95 latency, spend, and executions for the whole fleet, each with a sparkline and movement badge.
- **Ecosystem SLO Health banner** — the overall Healthy/Needs-Attention/Critical verdict + score/100, with a "why this status" link.
- **Behavioral Drift banner** — a live count of agents drifting from baseline, with the critical share called out.
- **Execution Volume chart** — agent runs over time split into succeeded/failed — the activity heartbeat behind every rate.
- **Spend by Application donut** — cost split by business area, one level up from agents.
- **Latency Percentiles chart** — P50/P90/P95/P99 bands over time.
- **Spend Trend + forecast** — cost trajectory with a forward projection at the current run rate.
- **Error Rate by Location & Hour heatmap** — failure concentration by zone × hour, the pattern-finder for local vs systemic issues.
- **Governance & hygiene tiles** — guardrail pass rate, open security threats, open high-risk findings, unowned agents, metadata gaps, and savings-found.
- **Applications roster** — one row per application: environment, agent count, executions, and spend share.
- **Open Findings queue** — the live, severity-ranked to-do list (reliability clusters + drift findings).
- **Generate Summary** — an on-demand, plain-language health summary of the fleet and its likely priorities.

### Agent Topology — the "where" view
- **Store Health Map** — every runtime instance plotted geographically, colored by its own health inside worst-of zone circles; pans and zooms.
- **Zones panel** — regions ranked by health with agent/instance counts, to jump straight to the area that needs attention.
- **Summary strip** — zone count, instance count, and how many instances are erroring/warning, with a fleet-status chip.
- **Geographic drill-down** — region → zone → store → instance → the agent's detail page.

### Agent Inventory — the registry ("who owns what")
- **Searchable agent catalog** — every logical agent with owner, application, model, error rate, environment, status, metadata state, and last activity.
- **Inventory KPIs** — applications, agents, and instances, plus hygiene counts (unowned, metadata gaps, degraded) that recompute over the current filter.
- **Quick filters** — one-click All / Failing / Unowned views, name search, application/status dropdowns, and a column picker.
- **Logical-vs-runtime distinction** — one logical agent, many runtime instances, kept separate so the registry reads by record not by pod.
- **Agent Detail deep-dive** — a single agent's instances, metrics, recent traces, correlated logs, dependency tree, and metadata coverage.

### Reliability — failure analysis & root cause
- **Header metrics** — success rate, failed executions, error rate, and active (open critical) incidents.
- **Tool-Calling Eval score** — how often the agent called the right tool correctly — catches "ran fine but did the wrong thing."
- **Error Rate over Time** — daily error-rate trend with an incident-spike badge and a plain-language caption naming the likely cause.
- **Succeeded vs Failed chart** — real counts (not rates) so a rate change can be read against traffic.
- **Failures by Agent** — donut + ranked table showing whether trouble is one agent or everywhere, with last-failure timestamps.
- **Failure Clusters** — repeated failures grouped by signature, each classified **CODE / DEPLOYMENT / DEPENDENCY** with instance/location blast-radius badges.
- **Ask SRE investigation** — an evidence-grounded AI root-cause analysis run on a specific cluster.
- **Live execution feed** — individual runs with input, output, status, eval result, feedback, and cost on one row; errors-only toggle; opens full trace detail.

### Latency — where time is spent
- **Distribution metrics** — P50/P95/P99 and max execution duration, computed from individual records (not daily averages).
- **Latency Percentiles over Time** — P50/P95/P99 as lines, to tell a system-wide slowdown from a broken-away tail.
- **Duration Histogram** — executions bucketed by duration band, revealing one-hump vs two-hump (e.g., cached vs uncached) behavior.
- **Percentile Breakdown** — expand P99 to the actual slowest executions, each opening its trace.
- **P95 by Agent** — the ten slowest agents by tail latency, always fleet-wide for comparison.
- **Trace Time Composition** — total trace time split across model / tool / retrieval / orchestration — proving where the time really goes.
- **Agent selector** — narrow the whole screen to one agent.

### Drift — slow degradation early warning
- **Statistical drift detection** — per agent, a baseline half vs a recent half compared across error rate, P95, cost/exec, tokens/exec, and composite quality.
- **Ranked drift feed** — open drift findings worst-first, each with direction, baseline→recent movement, and a trend sparkline.
- **Metric filter** — read the feed one drifting metric at a time (error/latency/quality) as separate work queues.
- **Drift detail + cause attribution** — baseline vs recent tiles, trend chart, and a cause chip: **deployment/provider** vs **input/data** drift.
- **Representative traces** — jump from the trend to the executions behind it.

### Alerts — the notification layer
- **Alert rules** — a condition (metric or finding type, operator, threshold, window), scope, severity, channels, and suppression window.
- **Multi-source conditions** — error rate, latency percentile, cost spike/budget, reliability clusters, security findings, guardrail-pass drop, and SLO breaches.
- **De-duplicated incidents** — one deduplicated alert per issue; recurrences refresh it rather than piling up rows.
- **Lifecycle** — FIRING → ACKNOWLEDGED → RESOLVED, with **auto-resolve** and a resolved notice when the condition clears.
- **Multi-channel routing** — Slack, Google Chat, PagerDuty, generic webhook, email, and the in-app inbox; severity-based routing; owner-scoped for Agent Owners.
- **Blast-radius on every alert** — agents/instances/locations/versions/executions affected, plus observed-vs-threshold.
- **Alert detail + triage** — a What/Why/Impact/Action summary, lifecycle timeline, per-channel delivery log, and deep-link to the evidence.
- **Alert bell** — a top-bar badge counting unread firing alerts on every screen.

---

## SPEND

### Cost & Tokens — spend, token economics & chargeback
- **Header metrics** — total spend, total tokens, cached %, and cost per 1K executions, each vs the previous window.
- **Spend by Model over Time** — daily spend stacked by model, to catch a pricier model quietly taking over the mix.
- **Total Spend + Forecast** — actuals plus a near-term dashed projection.
- **Share by Owner Team** — spend by owning team, with unowned spend called out separately.
- **Cost Pivot** — the same spend re-cut by application / agent / model / owner / cost center / location, always reconciling.
- **Cost Allocation (chargeback vs showback)** — post spend to each team's cost center, or show it without moving money.
- **30/60/90-day projections** — spend at the current daily rate over three horizons.
- **Per-execution costing** — cost built up from each execution (with model pricing), which is what makes every cut possible.

### Optimization — cost-efficiency workbench
- **Projected savings** — total monthly savings currently on the table across open recommendations.
- **Evidence-backed recommendations** — right-size the model, trim the prompt, cache repeated input — each with rationale, projected saving, and confidence.
- **Opportunity metrics** — open recommendations, average confidence, and average input tokens/execution (the headroom proxy).
- **AI Optimization Advisor** — a conversational way to interrogate the same findings and ask follow-ups.

---

## GOVERN

### Governance & Audit — the compliance & safety ledger
- **Decision KPIs** — counts of Allowed / Warned / Redacted-or-Held / Blocked evaluations across the window.
- **Decisions over Time** — the trend of policy outcomes, split by decision type.
- **Evaluations ledger** — every policy decision tied to a specific policy version on a specific execution at a specific time.
- **Role-based masking** — an Auditor sees sensitive prompt/output masked; a Compliance role sees it unmasked, with the active view stated.
- **Adversarial-input stream** — detected prompt-injection and jailbreak attempts with score, label, and governance decision.
- **Findings + evidence export** — a governance findings feed and an exportable audit-evidence package for a date range.

### Policies — the guardrail control surface
- **Policy authoring** — create/edit guardrails with structured JSON rules (match mode + conditions on keywords, patterns, or models).
- **Enforcement actions** — Allow / Warn / Redact / Approval / Block.
- **Versioned inline changes** — change a policy's action from its card; every change is versioned and written to the audit trail.
- **Test before ship** — evaluate a policy against sample input before enforcing it.
- **Active toggle** — draft a rule without enforcing it, separating authoring from deployment.

### Health & SLOs — defensible ecosystem health
- **Configurable SLOs** — each a metric + target + window + weight + warn factor, scoped fleet-wide, per-application, or per-agent.
- **Worst-of verdict + score** — category status = worst SLO, overall = worst category; a weighted score out of 100.
- **Observed-vs-target evidence** — every status resolves to the observed value beside its target, on the same line.
- **Category coverage** — reliability, latency, cost, governance, and quality.
- **Verdict travels** — surfaced read-only on the Dashboard banner and able to raise an SLO-breach alert.
- **Recompute on demand** — re-run the status any time.

---

## AI & BACKGROUND INTELLIGENCE (cross-cutting)

- **Deterministic detection, agentic investigation** — sweepers compute facts and create findings; AI agents interpret them.
- **SRE Root Cause Agent** — evidence-grounded probable cause, impact, and next action for a failure cluster or execution.
- **Token Optimization Agent** — turns token-waste findings into evidence-backed savings recommendations.
- **Executive SRE Summary Agent** — the plain-language "how are we doing and what should we look at" narrative.
- **Security Eval Agent** — classifies ambiguous adversarial inputs (injection/jailbreak/exfiltration).
- **AI Investigation Copilot (Claude Code + MCP)** — investigate the fleet conversationally from a coding agent; findings filed back into the product *(emerging)*.
- **Background sweepers** — inventory reconciler, dependency reconciler, reliability health, failure clusterer, latency analyzer, performance-regression analyzer, cost aggregator, cost forecaster, token-efficiency sweeper, governance-audit sweeper, metadata-completeness sweeper, executive-health aggregator.
- **Findings model** — one common, severity-ranked findings object consumed by every screen, with drill-down to evidence.

## PLATFORM & FOUNDATION

- **Built on Trillo AOS** — data access, APIs/functions, auth, RBAC, navigation, scheduling, and deployment as one foundation.
- **OpenTelemetry-native** — traces/spans, metrics, events, and logs, correlated by execution and trace ID.
- **Model pricing engine** — effective-dated pricing for deterministic per-execution cost.
- **Time-range aware** — a 7d/30d/90d control across every screen, with point-in-time "now" counts for open findings and hygiene.
- **Drill-anywhere** — every headline number and finding deep-links to the underlying evidence without losing context.
- **Production path** — the same application carries to production, replacing synthetic telemetry with a scalable Arrow-based data plane.

---

## Notes for the brochure

- **Three-page trim:** lead with *Positioning* + *What makes it different*, then one compact block per navigation group (Monitor / Spend / Govern) keeping ~4–6 top bullets each, then a short AI/Platform strip. The per-module detail above is the reservoir to draw one-liners from.
- **Screenshots:** the Module Reference PDF's annotated screens map 1:1 to these groups if the brochure wants figures.
- **Decide inclusion of the AI Investigation Copilot** (Claude Code + MCP) — powerful differentiator, but flag it as emerging if it isn't demo-ready.
