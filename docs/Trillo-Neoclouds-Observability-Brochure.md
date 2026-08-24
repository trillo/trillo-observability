# Trillo Neoclouds Observability (Powered by OliverDB)

Trillo Neoclouds Observability gives GPU cloud providers complete visibility,
reliability, and utilization control over their GPU fleets. It ingests GPU, node,
fabric, and job telemetry and turns it into centralized observability of health,
utilization, reliability, cost, and per-tenant SLAs — across every cluster and
region. Powered by the **OliverDB** telemetry engine, it
runs entirely in your own environment, so your fleet and tenant data never leave
it.

> **Business impact:** Higher fleet utilization and margin, faster failure detection, accurate per-tenant billing, stronger SLAs, and confident capacity planning.

## The Problem

- **Idle GPUs quietly burning margin.** You own the hardware, but you cannot see which GPUs sit idle or underused — so occupancy, and margin, leak.
- **Failures your tenants find first.** A failing GPU, node, or fabric link takes down tenant jobs before you know, and you cannot see the blast radius.
- **Billing you cannot defend.** Without accurate per-tenant metering, GPU-hours go unbilled and disputes cost you revenue.
- **Capacity planning by guesswork.** You cannot tell whether to buy more GPUs or pack tenants tighter, so you over-build or oversubscribe blind.

## How Trillo Solves It

- **Reclaim idle capacity** — surface idle and underused GPUs with the revenue at stake, per tenant and per cluster. → *Higher occupancy and margin.*
- **Catch failures before tenants do** — detect failing GPUs, nodes, and fabric from Xid / ECC / thermal / fabric signals, with blast radius and AI-assisted root cause. → *Fewer tenant-visible incidents.*
- **Bill every GPU-hour** — metered, per-tenant GPU usage, reconciled and exportable, with per-tenant SLA and noisy-neighbor visibility. → *Accurate, defensible revenue.*
- **Plan the next buildout with data** — utilization trends and safe oversubscription headroom. → *Confident capacity decisions.*

## Why Trillo

- **100% ownership of your data** — runs entirely in your environment; your fleet and tenant data never leave it.
- **A fraction of the cost** — comparable capability at a fraction of the total cost of ownership.
- **OliverDB** — petabyte-scale telemetry storage and analytics at a fraction of the cost, without the operational complexity.
- **Easy to customize** — model-driven and hot-deployable; and we help you customize dashboards, billing rules, and workflows.

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

*(Trillo Neoclouds Observability architecture diagram — GPU fleet → exporters/OTLP → OliverDB → dashboards / alerts / MCP, all in your environment.)*

## Deployment Model

**Standard exporters. No proprietary agents.**

GPU, node, and fabric telemetry (DCGM, node, and network exporters) streams into
**OliverDB** using OTLP. Non-standard or extended metrics are mapped at ingestion,
at high speed, so you don't have to re-instrument. The platform runs in your
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
