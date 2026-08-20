# Trillo AI Infra Observability for Neoclouds (Powered by OliverDB)

Trillo AI Infra Observability gives GPU cloud providers complete visibility,
reliability, and utilization control over their GPU fleets. It ingests GPU, node,
fabric, and job telemetry and turns it into centralized observability of health,
utilization, reliability, cost, and per-tenant SLAs — across every cluster and
region. Built on Trillo AOS and powered by the **OliverDB** telemetry engine, it
runs entirely in your own environment, so your fleet and tenant data never leave
it.

> **Business impact:** Higher fleet utilization and margin, faster failure detection, accurate per-tenant billing, stronger SLAs, and confident capacity planning.

## Why AI Infra Observability

- **Sell more of what you own** — surface idle and underused GPUs so occupancy, and margin, go up.
- **Catch failures before tenants do** — detect failing GPUs, nodes, and fabric early, with the blast radius.
- **Bill accurately** — metered, per-tenant GPU usage you can charge and reconcile.
- **Keep tenants happy** — per-tenant SLA visibility and noisy-neighbor detection.
- **Plan capacity with confidence** — utilization trends and oversubscription headroom.

## Why Trillo

- **OliverDB performance** — high-performance telemetry storage and analytics at GPU-fleet scale.
- **Model-driven customization** — customize dashboards, metrics, and policies and hot-deploy without redevelopment.
- **Complete platform out of the box** — health, utilization, reliability, billing, SLAs, and AI intelligence.
- **Runs in your environment** — your fleet and tenant data stay with you, with simple operations.

## Complete Platform

#### Monitor
- **Fleet inventory & topology** — GPUs, nodes, racks, clusters, and regions, discovered automatically.
- **GPU & node health** — utilization, memory, temperature, power, and ECC / Xid errors per device.
- **Fabric health** — NVLink and InfiniBand / RoCE link status and throughput.
- **Live fleet map** — health and occupancy by site, cluster, and rack; worst-case surfaced.
- **Utilization** — GPU occupancy and idle time across the fleet, per tenant and per cluster.

#### Reliability
- **Failure detection** — failing GPUs and nodes flagged from Xid, ECC, thermal, and fabric signals.
- **Blast radius** — which tenants and jobs a failing device or link affects.
- **Root cause** — AI-assisted analysis correlating hardware, job, and fabric evidence.
- **Alerting & on-call routing** — de-duplicated, blast-radius alerts to Slack / Teams / PagerDuty / webhook.

#### Utilization & Margin
- **Idle & underutilization** — reclaimable GPUs surfaced with the revenue at stake.
- **Occupancy & oversubscription** — allocation vs. actual use, with safe-headroom guidance.
- **Per-tenant metering & billing** — GPU-hours by tenant, reconciled and exportable.
- **Capacity forecast** — utilization trends to time the next buildout.

#### AI
- **AI Investigation Copilot** — investigate incidents conversationally from Claude Code and other AI coding agents through a secure MCP server.
- **Specialized AI analysis** — root-cause, anomaly, and utilization insights grounded in bounded evidence.
- **Background intelligence** — sweepers turn telemetry into findings, rollups, and baselines automatically.

## Architecture

*(Trillo AI Infra Observability architecture diagram — GPU fleet → exporters/OTLP → OliverDB → dashboards / alerts / MCP, all in your environment.)*

## Deployment Model

**Standard exporters. No proprietary agents.**

GPU, node, and fabric telemetry (DCGM, node, and network exporters) streams into
**OliverDB** using OTLP. Non-standard or extended metrics are mapped at ingestion,
at high speed, so you don't have to re-instrument. **Trillo AOS** runs in your
environment, with **PostgreSQL** for metadata and policy. Dashboards, alerts, APIs,
and **MCP** provide access to telemetry and AI-assisted analysis.

> **Enterprise Security:** In-environment deployment, provider- and tenant-owned data, encryption, RBAC and per-tenant isolation, versioned policies, and complete audit trails.

## Customization

- **Model-driven** — dashboards, metrics, billing rules, and policies are metadata, hot-deployed without redeploying.
- **Everything configurable** — SLOs, alert rules, oversubscription thresholds, and per-tenant billing.
- **Extensible** — add your own metrics, exporters, channels, and data sources.

## Business Outcomes

- **Higher margin** — raise fleet occupancy by reclaiming idle capacity.
- **Fewer tenant-visible incidents** — catch hardware and fabric failures before jobs fail.
- **Accurate revenue** — metered per-tenant billing you can defend.
- **Stronger SLAs** — per-tenant reliability you can report and improve.
- **Smarter buildouts** — capacity decisions grounded in real utilization.

## Pricing

*(By fleet size — per-GPU or per-GPU-hour tiers; table to be inserted.)*

> **Enterprise support:** Standard support included. Premium, 24×7 mission-critical, and dedicated engineering support available.

## Contact Us

*(Contact details — to be inserted.)*
