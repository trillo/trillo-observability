# Trillo Private Cloud Observability (Powered by OliverDB)

Trillo Private Cloud Observability gives enterprises complete visibility, reliability,
and cost control over the GPU infrastructure in their own data centers. It ingests
GPU, node, fabric, and job telemetry and turns it into centralized observability of
health, utilization, reliability, cost, and capacity — across every cluster and
site. Powered by the **OliverDB** telemetry engine, it runs
entirely on-premise or in your private cloud, so your data never leaves your
environment.

> **Business impact:** Higher GPU utilization, cost recovery through chargeback, lower MTTR, confident capacity planning, and audit-ready governance.

## The Problem

- **Expensive GPUs sitting idle.** You bought the hardware, but you cannot see which GPUs are idle or underused across teams — so costly capacity is wasted.
- **No fair way to recover cost.** You cannot attribute GPU spend to the teams and projects that use it, so there is no chargeback and no accountability.
- **Jobs failing without warning.** GPU, node, or fabric failures interrupt long training and inference runs before you catch them, and you cannot see which jobs are hit.
- **Buy-or-schedule decisions by guesswork.** You cannot tell whether you need more GPUs or just better scheduling, so budgets are set blind.

## How Trillo Solves It

- **Reclaim idle capacity** — surface idle and underused GPUs with the cost at stake, per team and per cluster. → *Higher utilization.*
- **Recover cost fairly** — chargeback and showback that attribute GPU spend to teams, projects, and cost centers. → *Defensible cost recovery.*
- **Catch failures before jobs die** — detect failing GPUs, nodes, and fabric from Xid / ECC / thermal / fabric signals, with job-interruption analysis and AI-assisted root cause. → *Lower MTTR, fewer failed runs.*
- **Plan buy-vs-schedule with data** — utilization trends that show whether to buy more GPUs or schedule better, with audit-ready governance throughout. → *Confident capacity planning.*

## Why Trillo

- **100% ownership of your data** — runs on-premise or in your private cloud, air-gap-capable; your data never leaves your environment.
- **A fraction of the cost** — comparable capability at a fraction of the total cost of ownership.
- **OliverDB** — petabyte-scale telemetry storage and analytics at a fraction of the cost, without the operational complexity.
- **Easy to customize** — model-driven and hot-deployable; and we help you customize dashboards, cost rules, and workflows.

## Complete Platform

#### Monitor
- **Fleet inventory & topology** — GPUs, nodes, racks, clusters, and sites, discovered automatically.
- **GPU & node health** — utilization, memory, temperature, power, and ECC / Xid errors per device.
- **Fabric health** — NVLink and InfiniBand / RoCE link status and throughput.
- **Live fleet map** — health and utilization by site, cluster, and rack; worst-case surfaced.
- **Utilization** — GPU occupancy and idle time across the fleet, per team and per cluster.

#### Reliability
- **Failure detection** — failing GPUs and nodes flagged from Xid, ECC, thermal, and fabric signals.
- **Job-interruption analysis** — which training and inference jobs a failure affects, and why.
- **Root cause** — AI-assisted analysis correlating hardware, job, and fabric evidence.
- **Alerting & on-call routing** — de-duplicated, blast-radius alerts to Slack / Teams / ServiceNow / webhook.

#### Utilization & Cost
- **Idle & underutilization** — reclaimable GPUs surfaced with the cost at stake.
- **Right-sizing** — match allocations to real usage across teams and jobs.
- **Chargeback & showback** — GPU cost attributed to teams, projects, and cost centers.
- **Capacity planning** — utilization trends that show whether to buy or schedule better.

#### Govern & AI
- **Quotas & policy** — fair-share allocation and guardrails across teams.
- **Compliance & audit** — versioned policies and a complete audit trail, on-prem or air-gapped.
- **AI Investigation Copilot** — investigate incidents conversationally from Claude Code and other AI coding agents through a secure MCP server.
- **Background intelligence** — sweepers turn telemetry into findings, rollups, and baselines automatically.

## Architecture

*(Trillo Private Cloud Observability architecture diagram — GPU fleet → exporters/OTLP → OliverDB → dashboards / alerts / MCP, all on-premise or in your private cloud.)*

## Deployment Model

**Standard exporters. No proprietary agents.**

GPU, node, and fabric telemetry (DCGM, node, and network exporters) streams into
**OliverDB** using OTLP. Non-standard or extended metrics are mapped at ingestion,
at high speed, so you don't have to re-instrument. The platform runs on-premise
or in your private cloud, with **PostgreSQL** for metadata and policy. Dashboards,
alerts, APIs, and **MCP** provide access to telemetry and AI-assisted analysis.

> **Enterprise Security:** On-premise / air-gap-capable deployment, customer-owned data, encryption, RBAC and field-level masking, versioned governance policies, and complete audit trails.

## Customization

- **Model-driven** — dashboards, metrics, cost rules, and policies are metadata, hot-deployed without redeploying.
- **Everything configurable** — SLOs, alert rules, quotas, cost allocation, and thresholds.
- **Extensible** — add your own metrics, exporters, channels, and data sources.

## Business Outcomes

- **Higher utilization** — put expensive GPUs to work instead of leaving them idle.
- **Cost recovery** — chargeback and showback that attribute spend to who uses it.
- **Lower MTTR** — catch failures early and reduce failed or restarted training runs.
- **Confident capacity planning** — buy or schedule based on real usage, not guesswork.
- **Audit-ready governance** — policies and trails for internal and regulatory needs.

## Pricing

*(By fleet size — per-GPU or per-GPU-hour tiers; table to be inserted.)*

> **Enterprise support:** Standard support included. Premium, 24×7 mission-critical, and dedicated engineering support available.

## Contact Us

*(Contact details — to be inserted.)*
