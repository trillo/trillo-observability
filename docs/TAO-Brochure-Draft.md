# Trillo Agent Observability (Powered by OliverDB) — 4-page brochure

> Layout target: **pages 1–3 = details**, **page 4 = Architecture + Deployment +
> Contact**. Bullets tightened to ~8 words; set the feature list in two columns if
> still long. Terminology: "failure clusters (grouped failure patterns)" — lead
> with the product term (matches the UI), gloss once.

---

## PAGE 1

# Trillo Agent Observability (Powered by OliverDB)

An enterprise platform to **monitor, govern, and optimize fleets of AI agents** —
turning the OpenTelemetry your agents already emit into a single, drill-anywhere
view of reliability, latency, cost, drift, and compliance, deployed **in your own
cloud** and powered by the high-performance **OliverDB** telemetry engine.

## Why Agent Observability

- **Find and fix fast** — failure clusters (grouped failure patterns) by code / deployment / dependency, latency bottlenecks, and drift, caught early.
- **Ship safely** — A/B-compare agent versions on real data before rollout.
- **Cut spend** — evidence-backed, confidence-scored token and cost optimization.
- **See the whole system** — every agent, instance, dependency, model, and location at a glance.
- **Govern with confidence** — every decision recorded, versioned, audit-ready.

## Why Trillo Agent Observability

- **Powered by OliverDB** — 2×–10× telemetry performance that keeps fleet-scale queries fast.
- **Model-driven & hot-deployable** — customize entities, dashboards, and policies and deploy without downtime.
- **Enterprise features out of the box** — reliability, cost, governance, drift, alerting, SLOs; no point-tool assembly, lower cost.
- **BYOC** — runs entirely in your VPC or on-premise; your data never leaves.
- **Simple to operate, lower cloud bill** — one data plane, efficient storage and compute by design.

## ROI

- **25% SRE productivity boost** — investigate incidents conversationally from Claude Code via the built-in MCP server.
- **Continuous cost savings** — model right-sizing, prompt trimming, and caching, tracked as realized savings.
- **Lower MTTR** — failure-cluster root-cause classification and de-duplicated alerting cut triage time.

*Plus: tool consolidation (one platform, not several) and audit-ready compliance.*

---

## PAGE 2

## Out of Box Features

**Monitor**
- **Auto inventory & dependency discovery** — agents, instances, models, tools, and systems, from telemetry or an agent catalog.
- **Fleet dashboard** — reliability, latency, spend, governance, and savings; every tile drills into the detail.
- **Agent topology** — a live geographic map, worst-of health per zone.
- **Reliability & root cause** — failure clusters classified code / deployment / dependency, with AI analysis.
- **Latency analysis** — P50–P99 distributions, slow-tail drill-down, model / tool / retrieval split.
- **Behavioral drift** — statistical early warning across error rate, latency, cost, and quality.
- **Alerting & on-call routing** — de-duplicated, blast-radius alerts to Slack / Teams / SMS / ServiceNow / webhook / email; fire → ack → auto-resolve.

**Spend**
- **Cost & token economics** — spend re-cut by application / agent / model / owner / cost center / location, with forecasts.
- **Chargeback & showback** — per-execution cost attributed to teams and cost centers.
- **Optimization** — evidence-backed savings recommendations with confidence, plus an AI advisor.

**Govern**
- **Governance & audit** — every decision tied to a policy version; role-based masking; exportable evidence package.
- **Adversarial-input detection** — prompt-injection and jailbreak attempts scored and governed.
- **Policy control** — Allow / Warn / Redact / Approval / Block; versioned; test before shipping.
- **Health & SLOs** — a defensible Healthy / Needs-Attention / Critical verdict with observed-vs-target evidence.

**AI intelligence**
- **Specialized AI agents** — SRE root-cause, token optimization, executive summary, and security evaluation.
- **AI Investigation Copilot** — investigate the fleet conversationally from Claude Code over a secure MCP server.
- **Background intelligence** — sweepers turn telemetry into findings, rollups, and baselines automatically.

---

## PAGE 3

## Cost

| Tier | Agent Runs / Month | Annual Price | Effective Price / 1M Runs |
| :--- | ---: | ---: | ---: |
| **Starter** | 5M | $25K / year | $417 |
| **Growth** | 25M | $50K / year | $167 |
| **Scale** | 100M | $100K / year | $83 |
| **Enterprise** | 500M | $200K / year | $33 |
| **Custom** | 500M+ | Custom | Negotiated |

## Support

| Support Tier | Price | Response SLA | Coverage |
| :--- | ---: | :--- | :--- |
| **Standard** | Included | 1 business day | Business hours |
| **Premium** | 15% of ARR — $15K min | 4-hour Sev-1 | Extended hours |
| **Mission Critical** | 25% of ARR — $30K min | 1-hour Sev-1 | 24×7 |
| **Dedicated** | $75K–$150K+ / year | 30-minute Sev-1 | 24×7 + named engineer |

## Security & Compliance (including data ownership)

- **BYOC — your cloud, your control** — the platform runs in your VPC or on-premise; telemetry and metadata never leave.
- **You own your data** — all telemetry, metadata, and analysis stay under your control, retention, and governance.
- **RBAC + field-level masking** — sensitive prompts and outputs masked by role (Auditor vs Compliance).
- **Enforced, versioned governance** — every policy decision and admin change is audited and exportable as evidence.
- **Private, encrypted connectivity** — VPN / DirectConnect / VPC Peering / Private Endpoint, encrypted in transit and at rest.

## Customization

- **Model-driven** — entities, dashboards, and policies are metadata, changed and hot-deployed without redeploying.
- **Everything configurable** — SLOs, alert rules, policies, cost allocation, and thresholds.
- **Extensible** — add your own functions, agents, channels, models, and data sources.

---

## PAGE 4

## Architecture

*(Insert the Trillo AI Observability architecture diagram — full width.)*

Telemetry flows from agents at every location, over private connectivity, into
**OliverDB**; the Trillo application on **GKE** serves dashboards, alerts, and an
**MCP / API gateway** for desktop SRE agents, while **PostgreSQL** holds metadata
and policy.

## Deployment Model

Agents send telemetry using **standard OpenTelemetry — no proprietary SDK required**.

- **Emit** — agents at any number of locations (Loc 1…N) export OTel spans, logs, events, and metrics.
- **Transport** — OTLP (gRPC/HTTP) as Arrow RecordBatches into OliverDB's high-throughput receiver, across VPN / DirectConnect / VPC Peering / Private Endpoint.
- **Store & serve** — OliverDB holds telemetry; PostgreSQL holds metadata and policy; the app runs on GKE in your project, with models via Gemini or open models.
- **Configure** — point each agent's OTel exporter at the Trillo endpoint; inventory and dependencies build automatically.
- **Consume** — dashboards & reports, alerts to Slack / Teams / SMS / ServiceNow, and SRE analysis through the MCP server / API gateway (Claude, Codex, Cursor, Antigravity).

## Contact Us

*(Contact details — to be inserted.)*
