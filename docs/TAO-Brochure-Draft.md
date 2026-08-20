# Trillo Agent Observability (Powered by OliverDB)

Trillo Agent Observability (TAO) gives enterprises complete visibility, governance,
and cost control over their fleets of AI agents. It ingests the OpenTelemetry that
agents already emit and turns it into a single, drill-anywhere view of reliability,
latency, cost, behavioral drift, and compliance — from a fleet-wide dashboard down
to an individual execution. Built on Trillo AOS and powered by the **OliverDB**
telemetry engine, TAO runs entirely in your own cloud, so your data never leaves
your environment.

> **Business impact:** Lower MTTR, higher SRE productivity, lower AI spend, reduced compliance risk, and fewer point tools.

## Why Agent Observability

- **Find and fix fast** — failure clusters classified as code / deployment / dependency, latency bottlenecks, and drift.
- **Ship safely** — A/B-compare agent versions on real data before rollout.
- **Cut spend** — evidence-backed, confidence-scored token and cost optimization.
- **See the whole system** — every agent, instance, dependency, model, and location at a glance.
- **Govern with confidence** — every policy decision recorded, versioned, and audit-ready.

## Why Trillo Agent Observability

- **OliverDB performance** — high-performance telemetry storage and analytics that keeps fleet-scale queries fast.
- **Model-driven customization** — customize and hot-deploy without platform redevelopment.
- **Enterprise platform out of the box** — reliability, cost, governance, SLOs, and AI intelligence.
- **BYOC with lower operational cost** — runs in your environment with a simple architecture and minimal operational overhead.

## Complete Platform

#### Monitor
- **Auto inventory & dependency discovery** — agents, instances, models, tools, and systems discovered from telemetry.
- **Fleet dashboard** — reliability, latency, spend, governance, and savings; every tile drills in.
- **Agent topology** — live geographic map showing worst-case health by location and zone.
- **Reliability & root cause** — failure clusters, blast radius, and AI-assisted root-cause analysis.
- **Latency analysis** — P50–P99, slow-tail drill-down, and model / tool / retrieval breakdown.
- **Behavioral drift** — statistical early warning across error rate, latency, cost, and quality.
- **Alerting & on-call routing** — de-duplicated, blast-radius alerts to Slack / Teams / SMS / ServiceNow; fire → ack → auto-resolve.

#### Spend
- **Cost & token economics** — analyze spend by application, agent, model, owner, cost center, and location, with forecasts.
- **Chargeback & showback** — per-execution cost attributed to teams and cost centers.
- **Optimization** — evidence-backed savings recommendations with confidence, plus an AI advisor.

#### Govern
- **Governance & audit** — every decision tied to a policy version; role-based masking; exportable evidence.
- **Adversarial-input detection** — prompt-injection and jailbreak attempts scored and governed.
- **Policy control** — Allow / Warn / Redact / Approval / Block; versioned; test before shipping.
- **Health & SLOs** — a defensible Healthy / Needs-Attention / Critical verdict with observed-vs-target evidence.

#### AI
- **Specialized AI agents** — SRE root-cause, token optimization, executive summary, and security.
- **AI Investigation Copilot** — investigate the fleet conversationally from Claude Code and other AI coding agents through a secure MCP server.
- **Background intelligence** — sweepers turn telemetry into findings, rollups, and baselines automatically.

## Architecture

*(Trillo Agent Observability (TAO) architecture diagram — full width, kept large and readable.)*

## Deployment Model

**Standard OpenTelemetry. No proprietary SDK.**

Agents stream spans, logs, events, and metrics securely into **OliverDB** using
OTLP. **Trillo AOS** runs in the customer's cloud, with **PostgreSQL** for metadata
and policy. Dashboards, alerts, APIs, and **MCP** provide access to telemetry and
AI-assisted analysis.

> **Enterprise Security:** BYOC / on-premise deployment, customer-owned data, encryption, RBAC and field-level masking, versioned governance policies, and complete audit trails.

## Customization

- **Model-driven** — entities, dashboards, and policies are metadata, hot-deployed without redeploying.
- **Everything configurable** — SLOs, alert rules, policies, cost allocation, and thresholds.
- **Extensible** — add your own functions, agents, channels, models, and data sources.

## Business Outcomes

- **Higher SRE productivity** — investigate incidents conversationally through the built-in MCP server.
- **Lower MTTR** — root-cause classification + de-duplicated alerting cut triage time.
- **Lower AI spend** — continuous right-sizing, prompt trimming, and caching.
- **Reduced compliance risk** — versioned governance and a complete audit trail.
- **Fewer point tools** — one platform for observability, cost, and governance.

## Pricing

| Tier | Agent Runs / Month | Annual Price | Effective Price / 1M Runs |
| :--- | ---: | ---: | ---: |
| **Starter** | 5M | $25K / year | $417 |
| **Growth** | 25M | $50K / year | $167 |
| **Scale** | 100M | $100K / year | $83 |
| **Enterprise** | 500M | $200K / year | $33 |
| **Custom** | 500M+ | Custom | Negotiated |

> **Enterprise support:** Standard support included. Premium, 24×7 mission-critical, and dedicated engineering support available.

## Contact Us

*(Contact details — to be inserted.)*
