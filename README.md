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
| `Telemetry-Ingestion-Endpoint-Design.md` | OTLP-native ingestion into Postgres for small/demo setups; app-scoped IngestionKey (schema0) + metering; same otlp_* schema as OliverDB. |
| `AOS-Issues-Triage-Plan.md` | Consolidated triage + phased work plan for all AOS-05/06/07 issues (numbering reconciled, security-first). **Internal.** |
| `Enterprise_AI_Agent_Observability_SRS_Addendum_Reconciliation.md` | SRS v1.1 ↔ addendum reconciliation (aligned / conflicts / gaps). |
| `Enterprise_AI_Agent_Observability_POC_Telemetry_Simulator_Requirements.md` | Simulator that generates telemetry for N agents (POC). |
| `Enterprise_AI_Agent_Observability_POC_Application_and_UX_Design.md` | Platform logic (inventory/dependency/status) + UX. |
| `Enterprise_AI_Agent_Observability_Pre_Demo_Feature_Specs.md` | Pre-demo features A/B/C: failure-spread (code-vs-deployment) classifier, security evals, **and alerting**. |
| `Enterprise_AI_Agent_Observability_Scheduled_Feature_Specs.md` | Full specs for features D/E/F: A/B version comparison (canary rollout), behavioral drift, health/SLO config. |
| `Enterprise_AI_Agent_Observability_SRE_Copilot_Plugin_Design.md` | Claude Code plugin + MCP: TAO as an SRE investigation copilot (two-plane security; runbooks-as-skills). |
| `SRE-Copilot-Tool-Manifest.md` | SRE Copilot: curated investigation tools → app functions (`agent_tool` allow-list), redaction posture, tool gaps + feature deps. **Internal** (backs the public `sre-claude-plugin`). |
| `SRE-Copilot-Investigation-Report-Handoff.md` | App-team spec for the plugin's write path: extend `AiAnalysis` + implement `write_investigation_report` (entity JSON + function JSON + checklist). **Internal.** |
| `Enterprise_AI_Agent_Observability_Gap_Analysis_and_OliverDB_Interface.md` | Gaps, OliverDB partner questionnaire, cursor/completeness design, and proposal-claim contingencies. **Revisit after OliverDB discussion.** |
| `Enterprise_AI_Agent_Observability_Competitive_Positioning.md` | Positioning vs Phoenix × Galileo. **Internal.** |
| `TAO-Brochure-Feature-List.md` | Comprehensive feature catalog (one-liners) organized by the console nav (Monitor/Spend/Govern) — source for the 3-page brochure. **Internal.** |
| `SRE-Plugin-Internal-Setup.md` | Internal team guide: OAuth client registration (OAuthClientRegistry), pointing the plugin at dev via SRE_MCP_URL, and invalid_client/auth troubleshooting. **Internal.** |
| `SRE-Plugin-UI-Login-Handoff.md` | UI-team spec: the login-first OAuth completion handshake an AOS app SPA must implement (detect oauth_state → authenticate → POST /oauth/complete → loopback redirect). **Internal.** |
| `Trillo-AI-Infra-Observability-Neoclouds-Brochure.md` | Infra-observability brochure for **neoclouds** (GPU cloud providers): utilization/margin, per-tenant billing, SLAs. |
| `Trillo-AI-Infra-Observability-Private-DataCenter-Brochure.md` | Infra-observability brochure for **private data centers**: utilization, chargeback, capacity, on-prem governance. |
| `TAO-Brochure-Draft.md` | Filled 3-page brochure content (overview, value prop, differentiators, ROI, out-of-box features, deployment, security, customization) — cost/support/contact are placeholders. **Internal.** |

## Status

Requirements/design phase. Open cross-doc decisions are tracked in the
reconciliation doc (B1–B5) and the addendum.
