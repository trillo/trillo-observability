# Trillo AI-Agent Observability (TAO)

Enterprise AI-Agent Observability & Analytics — the observability application
built on **Trillo AOS** that discovers, monitors, and governs AI agents from
their OpenTelemetry data.

**Private repository.** Contains customer-POC (Wendy's) material and competitive
analysis — do not distribute externally.

## Scope reminder

The **observed agents** are generic-framework agents (LangChain / other) deployed
by the customer on Google Cloud / edge. **Trillo AOS** is the platform the
observability application is built on — the analytics/consumer side, not the
observed agent runtime. (See the addendum, AD-003.)

## Documents (`docs/`)

| Document | What it is |
| :-- | :-- |
| `Enterprise_AI_Agent_Observability_POC_Requirements_Final.md` | Base PRD (v1.5) — the seven demo scenarios + data model. |
| `Enterprise_AI_Agent_Observability_SRS_Revised.md` | System Requirements Spec (v1.1) — architecture, functional reqs, PostgreSQL schema. |
| `Enterprise_AI_Agent_Observability_POC_Requirements_Addendum.md` | **Decision log** (AD-001…AD-016) — the authoritative delta from Q&A. |
| `Enterprise_AI_Agent_Observability_POC_Requirements_Addendum2.md` | **Decision log continued** (AD-017…) — next-tranche scheduled features (A/B version comparison, drift, health/SLO config, retention/sampling) **with UI specs**. |
| `OliverDB-Otel-Mapping-Requirements.md` | G1+G5: OTLP→`gen_ai.*` mapping, agent-identity resolve-by-name, coverage report — candidate OliverDB Rust plugin. **For partner discussion.** |
| `Enterprise_AI_Agent_Observability_SRS_Addendum_Reconciliation.md` | SRS v1.1 ↔ addendum reconciliation (aligned / conflicts / gaps). |
| `Enterprise_AI_Agent_Observability_POC_Telemetry_Simulator_Requirements.md` | Simulator that generates telemetry for N agents (POC). |
| `Enterprise_AI_Agent_Observability_POC_Application_and_UX_Design.md` | Platform logic (inventory/dependency/status) + UX. |
| `Enterprise_AI_Agent_Observability_Pre_Demo_Feature_Specs.md` | Pre-demo features A/B/C: failure-spread (code-vs-deployment) classifier, security evals, **and alerting**. |
| `Enterprise_AI_Agent_Observability_Scheduled_Feature_Specs.md` | Full specs for features D/E/F: A/B version comparison (canary rollout), behavioral drift, health/SLO config. |
| `Enterprise_AI_Agent_Observability_Gap_Analysis_and_OliverDB_Interface.md` | Gaps, OliverDB partner questionnaire, cursor/completeness design, and proposal-claim contingencies. **Revisit after OliverDB discussion.** |
| `Enterprise_AI_Agent_Observability_Competitive_Positioning.md` | Positioning vs Phoenix × Galileo. **Internal.** |

## Status

Requirements/design phase. Open cross-doc decisions are tracked in the
reconciliation doc (B1–B5) and the addendum.
