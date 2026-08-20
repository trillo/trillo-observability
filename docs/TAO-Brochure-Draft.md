# Trillo Agent Observability (Powered by OliverDB)

Trillo Agent Observability is an enterprise platform for **monitoring, governing,
and optimizing fleets of AI agents**. It turns the OpenTelemetry your agents
already emit into a single, drill-anywhere view of reliability, latency, cost,
drift, and compliance — deployed **in your own cloud** and powered by the
high-performance **OliverDB** telemetry engine.

---

## Why Agent Observability (Value proposition)

- **Find and fix issues fast** — failure clusters (grouped failure patterns) classified as code / deployment / dependency, latency bottlenecks, and behavioral drift over time.
- **Ship changes safely** — A/B compare two agent versions on real observability data before rolling out.
- **Cut spend** — cost and token optimization with evidence-backed, confidence-scored recommendations.
- **See the whole system** — a bird's-eye view of every agent, instance, dependency, model, and location.
- **Govern with confidence** — every policy decision recorded, versioned, and audit-ready.

## Why Trillo Agent Observability (Differentiator)

- **Powered by OliverDB** — a very high-performance telemetry engine (2×–10× performance) that keeps fleet-scale queries fast.
- **Model-driven & hot-deployable** — customize entities, dashboards, policies, and workflows and deploy changes without downtime.
- **Enterprise features out of the box, at a lower cost** — reliability, cost, governance, drift, alerting, and SLOs, without assembling point tools.
- **BYOC deployment** — runs entirely in your VPC or on-premise; your data never leaves your environment.
- **Simple to operate** — one platform, one data plane, minimal moving parts.
- **Optimal design lowers your cloud bill** — efficient storage and compute, plus continuous cost optimization of your agents.

## ROI

- **25% SRE productivity boost** — investigate incidents conversationally from desktop agents such as Claude Code via the built-in MCP server.
- **Continuous cost savings** — model right-sizing, prompt trimming, and caching recommendations, tracked as realized savings.
- **Lower MTTR** — failure-cluster root-cause classification (code vs deployment vs dependency) and de-duplicated alerting cut triage time.
- **Tool consolidation** — one platform replaces separate observability, cost, and governance tools (lower total cost of ownership).
- **Reduced compliance risk** — audit-ready governance with versioned policies and one-click evidence export.

## Out of Box Features

**Monitor**
- **Automatic agent inventory & dependency discovery** — agents, instances, models, tools, and external systems discovered from telemetry, or pulled from an agent catalog.
- **Fleet dashboard** — reliability, latency, spend, governance, ownership, and savings on one landing page, each tile a jump-off into the detail.
- **Agent topology** — a live geographic map of every runtime instance by location/zone, colored worst-of so one bad site can't hide.
- **Reliability & root cause** — error trends, failures-by-agent, and **failure clusters** classified code / deployment / dependency with blast radius and AI root-cause analysis.
- **Latency analysis** — P50/P95/P99 distributions, slow-tail drill-down, and trace time split across model / tool / retrieval / orchestration.
- **Behavioral drift detection** — statistical early warning for slow degradation across error rate, latency, cost, tokens, and quality.
- **Alerting & on-call routing** — de-duplicated, blast-radius-aware alerts to Slack / Teams / SMS / ServiceNow / webhook / email, with fire → ack → auto-resolve.

**Spend**
- **Cost & token economics** — spend and tokens re-cut by application / agent / model / owner / cost center / location, with cached-token savings and forecasts.
- **Chargeback & showback** — per-execution cost attributed to teams and cost centers, reconciled across every view.
- **Optimization workbench** — evidence-backed recommendations (right-size model, trim prompt, cache) with projected savings and confidence, plus an AI advisor.

**Govern**
- **Governance & audit ledger** — every policy decision tied to a policy version on a specific execution, with role-based masking of sensitive content.
- **Adversarial-input detection** — prompt-injection and jailbreak attempts scored, labeled, and governed.
- **Policy control** — author guardrails, set Allow / Warn / Redact / Approval / Block, test before shipping, and version every change to the audit trail.
- **Health & SLOs** — configurable objectives with a defensible Healthy / Needs-Attention / Critical verdict that always shows observed-vs-target evidence.
- **Audit evidence export** — a signed, exportable evidence package for any date range.

**AI intelligence**
- **Specialized AI agents** — SRE root-cause, token optimization, executive summary, and security evaluation, each grounded in bounded evidence.
- **AI Investigation Copilot** — investigate the fleet conversationally from Claude Code (and other coding agents) over a secure MCP server.
- **Background intelligence** — sweepers continuously turn telemetry into findings, rollups, and baselines — nobody watches a dashboard.

## Deployment Model

Agents send telemetry to Trillo Agent Observability using **standard OpenTelemetry —
no proprietary SDK required**.

- **Emit** — agents at any number of locations (Loc 1…N) export OTel **spans, logs, events, and metrics**.
- **Transport** — telemetry streams over **OTLP (gRPC/HTTP)** as Arrow RecordBatches into OliverDB's high-throughput receiver, across **VPN / DirectConnect / VPC Peering / Private Endpoint**.
- **Store & serve** — OliverDB holds telemetry; **PostgreSQL** holds metadata and policy (inventory, dependencies, pricing, governance, audit); the Trillo application runs on **Google Kubernetes Engine** in your project, with models via **Gemini or open models**.
- **Configure** — point each agent's OTel exporter at the Trillo endpoint; inventory and dependencies build automatically via background sweepers.
- **Consume** — dashboards & reports, alerts to Slack / Teams / SMS / ServiceNow, and SRE analysis through the **MCP server / API gateway** (Claude, Codex, Cursor, …).

## Cost

*(Cost table from the proposal — to be inserted.)*

## Support

*(Support tiers / SLAs — to be inserted.)*

## Security & Compliance (including ownership of data)

- **BYOC — your cloud, your control** — the entire platform runs in your VPC or on-premise; telemetry and metadata never leave your environment.
- **You own your data** — all telemetry, metadata, and analysis stay under your control, retention, and governance.
- **Role-based access + field-level masking** — sensitive prompts and outputs are masked by role (e.g., Auditor vs Compliance).
- **Enforced governance** — versioned policies with Allow / Warn / Redact / Approval / Block, recorded per execution.
- **Complete audit trail** — every policy decision and administrative change is versioned and exportable as an evidence package.
- **Private, encrypted connectivity** — VPN / DirectConnect / VPC Peering / Private Endpoint, encrypted in transit and at rest.

## Customization

- **Model-driven design** — entities, functions, agents, dashboards, and UI are metadata, changed and **hot-deployed** without redeploying the platform.
- **Everything configurable** — SLOs, alert rules, governance policies, cost allocation, and thresholds are all user-defined.
- **Extensible** — add your own functions and agents, and integrate new channels, models, and data sources.

## Architecture Diagram

*(Insert the Trillo AI Observability architecture diagram.)*

Telemetry flows from agents at every location, over private connectivity, into
**OliverDB**; the Trillo application on **GKE** serves dashboards, alerts, and an
**MCP / API gateway** for desktop SRE agents, while **PostgreSQL** holds metadata
and policy.

## Contact Us

*(Contact details — to be inserted.)*
