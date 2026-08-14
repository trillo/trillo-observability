# Enterprise AI Agent Observability & Analytics

## Proof of Concept Requirements and Demonstration Specification

**Document Version:** 1.5  
**Platform:** Trillo AOS  
**POC Data Source:** Synthetic telemetry and operational data  
**POC Storage:** PostgreSQL  
**Primary Objective:** Deliver a compelling user experience across the seven evaluation scenarios

---

## 1. Purpose

This document defines the Proof of Concept (POC) experience for an enterprise AI Agent Observability & Analytics evaluation.

The POC is designed to demonstrate how users can discover, investigate, analyze, govern, and manage an enterprise AI-agent ecosystem through a cohesive Trillo AOS application.

The POC is **not intended to prove production-scale telemetry ingestion or storage throughput**. For the POC, PostgreSQL will store synthetic data representing realistic agent executions, traces, models, tools, systems, costs, tokens, users, governance events, and historical trends.

The primary goal is to delight the customer with:

- A clear and intuitive user experience
- Fast navigation from executive insight to execution-level evidence
- Consistent workflows across inventory, reliability, latency, cost, optimization, and governance
- Rich visualizations that make complex agent behavior easy to understand
- Strong traceability between agents, applications, owners, models, tools, systems, users, and policies

The POC application will be built on **Trillo AOS**. The same application foundation is intended to carry forward to production. In production, synthetic telemetry ingestion will be replaced by the production ingestion architecture described separately.

---

## 2. POC Design Principles

### 2.1 User Experience First

Every RFP scenario should be demonstrable in a small number of intuitive steps. The user should not need to understand OpenTelemetry, database schemas, or telemetry internals to answer operational questions.

### 2.2 One Connected Experience

The seven scenarios should not feel like seven disconnected dashboards. A user should be able to move naturally from:

```text
Executive Health
      ↓
Application
      ↓
Agent
      ↓
Execution
      ↓
Model / Tool / System
      ↓
Evidence / Recommendation / Governance Action
```

### 2.3 Drill Down, Not Search Again

When the user identifies a problem or cost spike, the next screen should retain the current context. For example:

- Executive dashboard → high failure rate → impacted application → failing agent → failed trace
- Cost dashboard → expensive application → agent → model → executions
- Governance finding → policy violation → execution → prompt/output → user/session

### 2.4 Synthetic but Realistic Data

Synthetic data should tell a believable operational story and remain internally consistent across all screens.

For example, if `OrderProcessingAgent` has an Inventory Gateway reliability issue:

- The Inventory screen should show the inventory dependency.
- The Reliability screen should show inventory-service failures.
- The Latency screen should show elevated inventory-tool latency.
- The Executive Dashboard should reflect the reliability degradation.

### 2.5 Exact RFP Language in the Demo

Where practical, navigation labels and section headings should mirror the source RFP terminology so evaluators can immediately map the demonstration to scoring criteria.

---

## 3. Core Information Model

The POC should model the following concepts as first-class entities or dimensions.

### 3.1 Application

Represents a business application containing one or more AI agents.

Example:

- Application: `Enterprise Operations Platform`
- Business Unit: `Enterprise Technology`
- Cost Center: `CC-4102`

### 3.2 Logical Agent

Represents a named AI capability independent of individual runtime deployments.

Example:

- Agent: `OrderProcessingAgent`
- Owner: `Business-Automation-Engineering`
- Business Purpose: `Automated Enterprise Order Processing`

### 3.3 Agent Instance

Represents a runtime deployment of a logical agent.

Example dimensions:

- Site
- Cluster
- Pod or service instance
- Agent version
- Environment
- Status
- First seen / last seen

### 3.4 Dependencies

An agent may depend on multiple resources in parallel.

```text
                         ┌──► Model: gemini-1.5-pro
                         │
OrderProcessingAgent ─────┼──► Tool: inventory_lookup ──► ERP Inventory DB
                         │
                         └──► Tool: payment_gateway_api ───► Payment Gateway
```

Dependency types should include:

- Models
- Tools
- External systems
- Vector stores / knowledge services
- Other agents

### 3.5 Execution

Represents one agent execution or trace and provides the common drill-down object across reliability, latency, cost, token, and governance workflows.

Key dimensions include:

- Application
- Agent
- Agent version
- Runtime instance
- User
- Session
- Site / location
- Trace ID
- Start/end time
- Status
- Model(s)
- Tool(s)
- Cost
- Tokens
- Governance outcomes

---

# 4. Scenario 1: Agent Inventory & Discovery

## 4.1 Demonstration Requirements

Show how a user can:

- View all AI agents
- Identify owners
- Identify business purpose
- Identify models used
- Identify dependent tools and systems

## 4.2 POC Requirements

The POC shall:

- Display all logical agents in a searchable inventory.
- Distinguish logical agents from runtime instances.
- Display application, owner, contact, business purpose, cost center, environment, and status.
- Display active instances, locations, versions, and last activity.
- Display models, tools, systems, vector stores, and agent-to-agent dependencies.
- Identify unowned or metadata-incomplete agents.
- Support filtering by application, owner, status, environment, model, tool, system, version, and location.

## 4.3 User Experience

**Screen:** `Agent Inventory`

**Top Summary Cards:**

- Logical Agents
- Active Instances
- Active Locations
- Degraded Agents
- Unowned Agents
- Metadata-Incomplete Agents

**Workflow:**

1. User opens **Agent Inventory**.
2. A filterable grid displays all logical agents.
3. User searches for `OrderProcessingAgent`.
4. Selecting the agent opens an **Agent Details Drawer**.
5. The header shows:
   - Owner
   - Contact
   - Business Purpose
   - Application
   - Cost Center
   - Environment
   - Active Instances
   - Active Locations
   - Deployed Versions
6. The **Dependencies** tab displays an interactive topology graph.
7. The **Instances** tab displays runtime deployments by location, version, environment, and status.
8. The **Metadata Quality** panel highlights missing ownership or business metadata.

## 4.4 Target Demonstration Outcome

The evaluator should be able to locate an agent, understand who owns it, why it exists, and what it depends on in under a minute.

---

# 5. Scenario 2: Reliability Investigation

## 5.1 Demonstration Requirements

An AI agent fails. Show how to:

- Locate the failed execution
- Determine root cause
- Review inputs
- Review outputs
- Review execution trace
- Identify impacted systems

## 5.2 POC Requirements

The POC shall:

- Surface failed executions and allow filtering by application, agent, location, user, session, time, error, and trace ID.
- Display a trace waterfall showing nested agent, model, retrieval, and tool operations.
- Display inputs, prompts, tool arguments, outputs, exceptions, and related logs.
- Identify the first failing span and likely root cause.
- Display directly impacted and potentially impacted dependencies.
- Show historical failure trends and comparisons with a prior baseline.
- Provide an optional AI-generated diagnostic summary grounded in the displayed evidence.

## 5.3 User Experience

**Screen:** `Reliability Explorer`

**Workflow:**

1. User opens **Reliability Explorer** and filters status to `ERROR`.
2. User selects a failed `OrderProcessingAgent` execution.
3. The trace waterfall highlights `execute_tool inventory_lookup` as the first failing span.
4. The user reviews:
   - Request / input
   - Prompt
   - Model response
   - Tool arguments
   - Tool response / exception
   - Related logs
5. The user selects **Analyze Failure**.
6. The application displays:

> **Root Cause:** `inventory_lookup` failed after the Inventory Gateway returned HTTP 504. Related execution evidence shows repeated connection retries to the ERP Inventory endpoint.

7. The **Impacted Systems** panel displays:

| Dependency | Impact | Evidence |
| :--- | :--- | :--- |
| `inventory_lookup` | Failed | HTTP 504 |
| `ERP Inventory DB` | Unreachable | Connection retries exhausted |
| `OrderProcessingAgent` | Degraded | Execution failed |
| `OrderSuggestionAgent` | Potential impact | Shares dependency |

8. The user opens **Historical Trend** to compare the error rate with the previous seven days.

## 5.4 Target Demonstration Outcome

The evaluator should be able to move from "an agent failed" to a credible root cause and impacted-system view without manually correlating multiple tools.

---

# 6. Scenario 3: Latency Analysis

## 6.1 Demonstration Requirements

Show:

- Latency reporting
- Bottleneck identification
- Model latency
- Tool latency
- Historical trends

## 6.2 POC Requirements

The POC shall:

- Show end-to-end execution latency.
- Break latency into model, tool, retrieval, orchestration, and other components.
- Display P50, P90, P95, and P99 trends.
- Compare latency by application, agent, agent version, model, tool, location, and environment.
- Link aggregate latency spikes to individual contributing executions.
- Automatically highlight the largest latency contributor.

## 6.3 User Experience

**Screen:** `Latency Analytics`

**Workflow:**

1. User selects `OrderProcessingAgent`.
2. KPI cards show:
   - End-to-End P95
   - Model P95
   - Tool P95
   - Change vs. previous seven days
3. The latency breakdown shows, for example:
   - Model: 30%
   - External Tools: 62%
   - Orchestration / Other: 8%
4. User selects a P99 spike.
5. The trace view automatically filters to contributing executions.
6. The waterfall or flamegraph highlights `query_vector_store` as the largest bottleneck.

## 6.4 Target Demonstration Outcome

The evaluator should immediately see whether slowness comes from the model, tools, retrieval, or application orchestration and be able to drill into the evidence.

---

# 7. Scenario 4: Cost & Token Visibility

## 7.1 Demonstration Requirements

Show:

- Cost by agent
- Cost by model
- Cost by application
- Cost by owner
- Token consumption by agent
- Token consumption by model
- Historical trends

Bonus:

- Forecasting
- Chargeback
- Showback

## 7.2 POC Requirements

The POC shall:

- Track input, output, cached, reasoning, and total tokens when applicable.
- Calculate realistic synthetic costs using model pricing maintained in the POC dataset.
- Report cost and token consumption by:
  - Application
  - Agent
  - Model
  - Owner
  - Cost center
  - Site
  - Environment
- Display historical cost and token trends.
- Support drill-down from aggregate cost to individual execution.
- Demonstrate forecasting, chargeback, and showback.

## 7.3 User Experience

**Screen:** `Cost & Token Economics`

**Pivot Tabs:**

- By Application
- By Agent
- By Model
- By Owner
- By Cost Center
- By Location

**Workflow:**

1. KPI cards display:
   - Month-to-Date Cost
   - Total Tokens
   - Average Cost per Execution
   - Forecasted Month-End Cost
2. User opens **By Application** and selects `Enterprise Operations Platform`.
3. User drills into `OrderProcessingAgent`.
4. User drills into the highest-cost model.
5. User opens individual contributing traces.
6. Historical charts show cost, tokens, executions, and cost per execution.
7. **Chargeback** allocates cost by owner and cost center.
8. **Showback** displays the same allocation without financial posting.
9. **Forecast** projects 30-, 60-, and 90-day spend using synthetic historical trends.

## 7.4 Target Demonstration Outcome

The evaluator should understand where AI spend is going and be able to move from enterprise cost to the exact applications, agents, models, and executions driving it.

---

# 8. Scenario 5: Token Optimization

## 8.1 Demonstration Requirements

Show how the platform identifies:

- Excessive prompts
- Wasteful token usage
- Expensive models
- Optimization recommendations

## 8.2 POC Requirements

The POC shall identify examples of:

- Excessive prompt size
- High input-to-output token ratios
- Repeated context across turns
- Duplicate instructions
- Retrieved context that is not used
- Expensive model usage for simple tasks
- Prompt caching opportunities
- Session summarization opportunities
- Model right-sizing opportunities

Recommendations shall contain evidence, estimated savings, confidence, and the analyzed execution population.

## 8.3 User Experience

**Screen:** `Token Optimization`

The recommendation feed is ranked by projected savings and confidence.

Example:

| Field | Example |
| :--- | :--- |
| Issue | Model over-provisioning in `OrderRouterAgent` |
| Evidence | 94% of executions return fewer than 20 output tokens; average input 2,480 tokens; average output 7 tokens |
| Recommendation | Evaluate a lower-cost model for routing classification |
| Projected Impact | Estimated savings of `$3,200/month` |
| Confidence | High, based on 280,000 executions |

**Workflow:**

1. User opens a recommendation.
2. The **Prompt Inspector** highlights repeated or unnecessary context.
3. The platform quantifies avoidable tokens and estimated cost.
4. User filters recommendations by application, agent, model, owner, impact, and type.
5. Recommendations can be marked `Accepted`, `Dismissed`, `In Progress`, or `Validated`.

## 8.4 Target Demonstration Outcome

Recommendations should feel analytical and evidence-based rather than generic AI advice.

---

# 9. Scenario 6: Governance & Auditability

## 9.1 Demonstration Requirements

Show how to:

- Audit agent activity
- Review prompts
- Review outputs
- Review user activity
- Apply governance controls
- Export audit evidence

## 9.2 POC Requirements

The POC shall:

- Provide a searchable audit trail of agent executions.
- Display user requests, prompts, outputs, tool calls, evaluations, and policy decisions.
- Support search by application, agent, user, session, owner, policy, decision, trace, and time range.
- Demonstrate role-based masking of sensitive prompt and output content.
- Demonstrate versioned governance policies.
- Support policy outcomes such as `Allow`, `Warn`, `Redact`, `Require Approval`, and `Block`.
- Record simulated administrative policy changes.
- Export a demonstrable audit evidence package.

## 9.3 User Experience

**Screen:** `Governance & Audit`

**Tabs:**

- Audit Trail
- Policies
- Findings
- Evidence Export

### Audit Workflow

1. Auditor searches by user, session, application, agent, or policy decision.
2. Auditor selects an execution.
3. The inspection panel displays:
   - Prompt
   - Output
   - User and session
   - Application and agent
   - Model and tool activity
   - Guardrail results
   - Policy decision
4. Sensitive values appear masked for users without appropriate permission.

### Governance Policy Workflow

1. Administrator opens **Policies**.
2. Administrator selects `PCI Data Protection`.
3. The configured action changes from `Warn` to `Block`.
4. A test execution is run against the policy.
5. The result displays `BLOCKED`.
6. The audit trail records the administrative change and policy version.

### Evidence Export Workflow

1. Auditor selects application, date range, agents, and findings.
2. Auditor selects **Export Audit Evidence**.
3. The POC generates a JSON/CSV package containing selected audit records, policy information, a manifest, and verification hashes.

## 9.4 Target Demonstration Outcome

The evaluator should see governance as an operational workflow, not just a passive reporting dashboard.

---

# 10. Scenario 7: Executive Dashboard

## 10.1 Demonstration Requirement

Create a single dashboard answering:

> **How healthy is our AI ecosystem?**

Show:

- Reliability
- Cost
- Token usage
- Adoption
- Governance findings

## 10.2 POC Requirements

The dashboard shall consolidate:

- Reliability
- Latency
- Cost
- Token usage
- Adoption
- Governance findings
- Optimization potential

Each category shall display:

- Current value
- Trend or prior-period comparison
- Status
- Drill-down path

## 10.3 User Experience

**Screen:** `AI Ecosystem Health`

```text
┌────────────────────────────────────────────────────────────────────────────────────┐
│                       AI ECOSYSTEM HEALTH: HEALTHY                                  │
├──────────────────────────┬──────────────────────────┬───────────────────────────────┤
│ RELIABILITY & LATENCY    │ COST & TOKEN ECONOMICS   │ ADOPTION & VOLUME             │
│ Success Rate: 99.2%      │ MTD Cost: $14,250        │ Active Agents: 18             │
│ Change: +0.3%            │ Change: +12%             │ Active Locations: 7,420          │
│ P95 Latency: 1.1 sec     │ Total Tokens: 185M       │ Executions: 1.2M              │
│ Status: Healthy          │ Forecast: $18,900        │ Adoption Change: +14%         │
├──────────────────────────┴──────────────────────────┴───────────────────────────────┤
│ GOVERNANCE & OPTIMIZATION                                                          │
│ Guardrail Pass Rate: 99.9%       Open High-Risk Findings: 2                        │
│ Unowned Agents: 3                 Metadata-Incomplete Agents: 5                     │
│ Optimization Potential: $3,200/month                                               │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Workflow:**

1. Executive opens **AI Ecosystem Health**.
2. Executive immediately sees reliability, spend, token consumption, adoption, and governance posture.
3. Selecting a high-risk finding drills into the affected application, agent, execution, and policy evidence.
4. Selecting cost growth drills into the applications and owners driving the increase.
5. Selecting reliability degradation drills into failing agents and traces.

The POC should use a qualitative overall state such as `Healthy`, `Needs Attention`, or `Critical` unless a transparent scoring formula is explicitly defined.

## 10.4 Target Demonstration Outcome

An executive should understand the state of the AI ecosystem in seconds and be able to drill into any important finding without leaving the application.

---

# 11. POC Data and Storage

## 11.1 PostgreSQL

PostgreSQL will be used for the POC because it provides a fast and practical way to populate, query, and demonstrate synthetic telemetry and operational data.

The POC database is not intended to represent the final production telemetry architecture.

## 11.2 Consolidated Logical and OTel POC Data Model

The following data model supports the synthetic PostgreSQL dataset and the POC user experience. It combines:

1. **Business and operational entities** used by the Trillo AOS application, such as applications, logical agents, runtime instances, ownership, dependencies, policies, findings, and AI analyses.
2. **OpenTelemetry telemetry entities** used to represent traces/spans, metrics, events, and logs in a form that is easy to populate synthetically and query during the POC.
3. **Analytical entities** used by scheduled functions and platform agents for rollups, baselines, findings, recommendations, and job execution state.

This model is **not intended to prescribe the physical production telemetry storage architecture**. For the POC, PostgreSQL stores synthetic OTel-shaped data. Production ingestion and storage are defined separately in the Production Architecture document and may use OliverAI and the enhanced Arrow-based telemetry data plane.

### 11.2.1 Core Relationship Model

```text
Application
   └── Agent
         ├── AgentInstance
         ├── AgentDependency
         │      ├── Model
         │      ├── Tool
         │      └── ExternalSystem
         │
         └── AgentExecution                 (logical execution / UI root)
                ├── OTel Spans              (trace tree)
                ├── OTel Events             (exceptions, evaluations, policy events)
                ├── OTel Logs               (application/container diagnostics)
                ├── GovernanceEvaluation
                └── Finding
                       └── AIAnalysis

OTel Metrics ──► dimensioned by Application / Agent / Model / Tool / Owner / Site / Time
      │
      └──► MetricRollups / Baselines ──► Findings
```

Background intelligence uses the following processing relationship:

```text
SweeperRun / Function
        ↓
Deterministic Rollup / Finding
        ↓
Specialized AI Agent (when reasoning is useful)
        ↓
AIAnalysis / Recommendation
```

This explicitly supports the POC architectural principle:

- **Functions own facts and computations.**
- **Agents own interpretation and reasoning.**
- A scheduled or event-driven function may create a finding and invoke a specialized AI agent.
- An AI agent may call trusted functions as tools to retrieve evidence or perform authoritative calculations.

### 11.2.2 Trace and Execution Modeling Rule

The POC uses both a logical execution entity and OTel spans:

- `agent_execution` is the application-level root used by the UI for a single end-to-end agent execution.
- `otlp_span` contains the actual trace tree. All spans sharing the same `trace_id` belong to the same distributed trace.
- The root span for the execution should carry the same `execution_id` and `trace_id` as the `agent_execution` row.
- `parent_span_id` establishes the waterfall/tree relationship among agent, model, tool, retrieval, and orchestration spans.
- `otlp_log` and `otlp_event` correlate back to the execution using `execution_id`, `trace_id`, and, when available, `span_id`.
- `otlp_metric` stores metric points or pre-aggregated metric values and is dimensioned independently for dashboard and trend queries.

This separation gives the POC a clean UI object while preserving OTel-compatible telemetry semantics.

---

### 11.2.3 `application`

Represents a business application containing one or more logical agents.

Key fields:

- `application_id`
- `application_name`
- `business_unit`
- `business_purpose`
- `owner_team`
- `owner_email`
- `cost_center`
- `environment`
- `status`
- `created_at`
- `updated_at`

### 11.2.4 `agent`

Represents a logical agent type independent of runtime instances.

Key fields:

- `agent_id`
- `application_id`
- `agent_name`
- `description`
- `business_purpose`
- `owner_team`
- `owner_email`
- `current_version`
- `environment`
- `status`
- `registration_status`
- `governance_classification`
- `first_seen_at`
- `last_seen_at`
- `metadata_source`
- `attributes_json`

The inventory UI should operate primarily on this entity rather than displaying every runtime instance as a separate agent.

### 11.2.5 `agent_instance`

Represents an observed runtime deployment of a logical agent.

Key fields:

- `instance_id`
- `agent_id`
- `service_instance_id`
- `location_id`
- `gcp_project_id`
- `cluster_id`
- `namespace_name`
- `pod_name`
- `agent_version`
- `environment`
- `status`
- `first_seen_at`
- `last_seen_at`
- `attributes_json`

A single logical agent may have thousands of instances.

### 11.2.6 `models`

Represents AI models available to or observed in the agent ecosystem.

Key fields:

- `model_id`
- `provider_name`
- `model_name`
- `model_version`
- `model_tier`
- `status`
- `approved_for_use`
- `metadata_json`

### 11.2.7 `tools`

Represents callable tools used by agents.

Key fields:

- `tool_id`
- `tool_name`
- `description`
- `tool_type`
- `owner_team`
- `external_system_id` (optional)
- `status`
- `metadata_json`

### 11.2.8 `external_systems`

Represents systems accessed directly or indirectly by agents, such as enterprise business systems, payment gateways, databases, vector stores, and enterprise APIs.

Key fields:

- `system_id`
- `system_name`
- `system_type`
- `owner_team`
- `environment`
- `criticality`
- `status`
- `metadata_json`

### 11.2.9 `agent_dependency`

Represents the operational dependency graph for an agent.

Key fields:

- `dependency_id`
- `agent_id`
- `dependency_type` (`MODEL`, `TOOL`, `SYSTEM`, `VECTOR_STORE`, `AGENT`)
- `target_id`
- `target_name`
- `parent_dependency_id` (optional)
- `relationship_type`
- `discovery_source` (`OBSERVED`, `REGISTERED`, `INFERRED`)
- `confidence` (optional)
- `first_seen_at`
- `last_seen_at`
- `evidence_trace_id` (optional)
- `is_active`
- `attributes_json`

This entity supports the topology view used in inventory, reliability investigation, and impacted-system analysis.

### 11.2.10 `agent_execution`

Represents one end-to-end agent execution and is the common application drill-down object across reliability, latency, cost, token, and governance workflows.

Key fields:

- `execution_id`
- `trace_id`
- `application_id`
- `agent_id`
- `instance_id`
- `agent_version`
- `user_id`
- `session_id`
- `location_id`
- `start_time`
- `end_time`
- `duration_ms`
- `status`
- `error_type`
- `error_message`
- `input_tokens`
- `output_tokens`
- `cached_tokens`
- `reasoning_tokens`
- `total_tokens`
- `estimated_cost_usd`
- `reported_cost_usd` (optional)
- `cost_source`
- `prompt_text` (subject to policy)
- `completion_text` (subject to policy)
- `metadata_json`

---

## 11.3 OpenTelemetry Telemetry Schema for the POC

The POC PostgreSQL database shall include OTel-shaped tables for spans, metrics, events, and logs. These tables are the primary grounding for trace visualization, reliability investigation, latency analysis, token/cost reporting, governance inspection, and correlated diagnostics.

The tables intentionally include frequently queried business dimensions as flat columns, even when the same values may also exist inside OTel resource/span attributes. This denormalization is acceptable for the POC because it simplifies synthetic-data generation and interactive querying.

Raw or unmapped OTel attributes should remain available in JSON fields so the POC is not limited to the explicitly modeled columns.

### 11.3.1 `otlp_span` — Traces, Agent Execution, Model Calls, Tool Calls, Retrieval

`otlp_span` represents individual spans within a distributed trace. A complete trace is reconstructed from all rows sharing the same `trace_id`; `parent_span_id` defines nesting.

| Field | PostgreSQL Type | Purpose |
| :--- | :--- | :--- |
| `span_id` | `VARCHAR(64)` | Primary span identifier |
| `trace_id` | `VARCHAR(64)` | Distributed trace identifier |
| `parent_span_id` | `VARCHAR(64)` | Parent span for waterfall/tree reconstruction |
| `execution_id` | `VARCHAR(128)` | Links to logical `agent_execution` |
| `name` | `VARCHAR(512)` | Span name, e.g. model chat or tool execution |
| `span_kind` | `VARCHAR(32)` | OTel span kind |
| `span_category` | `VARCHAR(32)` | `AGENT`, `MODEL`, `TOOL`, `RETRIEVAL`, `ORCHESTRATION`, `OTHER` |
| `start_time` | `TIMESTAMPTZ` | Span start time |
| `end_time` | `TIMESTAMPTZ` | Span end time |
| `duration_ms` | `DOUBLE PRECISION` | Computed duration |
| `status_code` | `VARCHAR(32)` | `OK`, `ERROR`, or mapped OTel status |
| `status_message` | `TEXT` | Error/status detail |
| `application_id` | `VARCHAR(128)` | Application dimension |
| `application_name` | `VARCHAR(256)` | Application display name |
| `agent_id` | `VARCHAR(128)` | Logical agent identifier |
| `agent_name` | `VARCHAR(256)` | Logical agent name |
| `agent_version` | `VARCHAR(128)` | Agent/service version |
| `service_name` | `VARCHAR(256)` | OTel service name |
| `service_instance_id` | `VARCHAR(256)` | Runtime service/agent instance |
| `environment` | `VARCHAR(64)` | Deployment environment |
| `location_id` | `VARCHAR(64)` | Enterprise site/location dimension |
| `owner_team` | `VARCHAR(128)` | Business ownership dimension |
| `cost_center` | `VARCHAR(64)` | Financial allocation dimension |
| `user_id` | `VARCHAR(256)` | End-user identifier, subject to RBAC/privacy |
| `session_id` | `VARCHAR(256)` | Conversation/session identifier |
| `operation_name` | `VARCHAR(128)` | GenAI/tool operation name |
| `provider_name` | `VARCHAR(128)` | Model provider |
| `request_model` | `VARCHAR(256)` | Requested model |
| `response_model` | `VARCHAR(256)` | Actual/provider-returned model |
| `tool_name` | `VARCHAR(256)` | Tool invoked by the span |
| `dependent_system` | `VARCHAR(256)` | External system reached by the tool |
| `input_tokens` | `BIGINT` | Input token count |
| `output_tokens` | `BIGINT` | Output token count |
| `cached_tokens` | `BIGINT` | Cached-token count when available |
| `reasoning_tokens` | `BIGINT` | Reasoning-token count when available |
| `total_tokens` | `BIGINT` | Total token count |
| `estimated_cost_usd` | `NUMERIC(18,8)` | Platform-calculated cost |
| `reported_cost_usd` | `NUMERIC(18,8)` | Provider-reported cost when available |
| `cost_source` | `VARCHAR(32)` | `PROVIDER_REPORTED` or `PLATFORM_ESTIMATED` |
| `pricing_record_id` | `BIGINT` | Effective-dated pricing record used |
| `prompt_text` | `TEXT` | Captured prompt/input subject to policy |
| `completion_text` | `TEXT` | Captured model output subject to policy |
| `tool_arguments` | `JSONB` | Tool arguments |
| `tool_result` | `JSONB` | Tool result or structured error |
| `raw_attributes` | `JSONB` | Catch-all for unmapped OTel/resource attributes |

Recommended indexes for the POC:

- `trace_id`
- `execution_id`
- `start_time DESC`
- `application_id`
- `agent_id`
- `status_code`
- `response_model`
- `tool_name`
- `location_id`
- `(user_id, session_id)`

### 11.3.2 `otlp_metric` — Raw or Aggregated Metric Points

`otlp_metric` stores OTel metric points or intentionally pre-aggregated synthetic metric values used for time-series dashboards. It may contain standard OTel/GenAI metrics plus POC-specific derived metrics.

| Field | PostgreSQL Type | Purpose |
| :--- | :--- | :--- |
| `metric_id` | `BIGSERIAL` | Primary key |
| `metric_name` | `VARCHAR(256)` | Metric name, e.g. token usage, operation duration, execution count |
| `metric_time` | `TIMESTAMPTZ` | Metric timestamp/window time |
| `aggregation_period` | `VARCHAR(32)` | Optional period such as `1m`, `5m`, `1h`, `1d` |
| `application_id` | `VARCHAR(128)` | Application dimension |
| `application_name` | `VARCHAR(256)` | Application display name |
| `agent_id` | `VARCHAR(128)` | Agent dimension |
| `agent_name` | `VARCHAR(256)` | Agent display name |
| `agent_version` | `VARCHAR(128)` | Version dimension |
| `model_name` | `VARCHAR(256)` | Model dimension |
| `tool_name` | `VARCHAR(256)` | Tool dimension |
| `owner_team` | `VARCHAR(128)` | Ownership dimension |
| `cost_center` | `VARCHAR(64)` | Financial dimension |
| `environment` | `VARCHAR(64)` | Environment dimension |
| `location_id` | `VARCHAR(64)` | Site/location dimension |
| `token_type` | `VARCHAR(32)` | `input`, `output`, `cached`, `reasoning`, `total` when applicable |
| `metric_value` | `DOUBLE PRECISION` | Numeric metric value |
| `sample_count` | `BIGINT` | Number of observations represented |
| `dimensions` | `JSONB` | Additional dimensions/attributes |

Recommended indexes for the POC:

- `(metric_name, metric_time DESC)`
- `application_id`
- `agent_id`
- `model_name`
- `owner_team`

Typical POC metrics include:

- `gen_ai.client.token.usage`
- `gen_ai.client.operation.duration`
- execution count
- success/error count
- cost
- P50/P90/P95/P99 latency
- adoption/activity volume
- governance pass/fail counts

### 11.3.3 `otlp_event` — Exceptions, Evaluations, Policy and Execution Events

`otlp_event` stores normalized execution events that should be independently searchable and correlatable. For the POC this includes exceptions, evaluation results, guardrail outcomes, and other meaningful execution events.

| Field | PostgreSQL Type | Purpose |
| :--- | :--- | :--- |
| `event_id` | `BIGSERIAL` | Primary key |
| `trace_id` | `VARCHAR(64)` | Correlated trace |
| `span_id` | `VARCHAR(64)` | Correlated span when available |
| `execution_id` | `VARCHAR(128)` | Logical execution |
| `event_name` | `VARCHAR(256)` | Event name such as exception or evaluation result |
| `event_time` | `TIMESTAMPTZ` | Event timestamp |
| `application_id` | `VARCHAR(128)` | Application dimension |
| `agent_id` | `VARCHAR(128)` | Agent dimension |
| `user_id` | `VARCHAR(256)` | User dimension subject to privacy policy |
| `session_id` | `VARCHAR(256)` | Session/conversation dimension |
| `eval_metric_name` | `VARCHAR(256)` | Evaluation name, e.g. toxicity, hallucination, PII leak |
| `eval_score` | `DOUBLE PRECISION` | Numeric evaluation score when applicable |
| `eval_label` | `VARCHAR(32)` | `PASS`, `FAIL`, or other evaluation label |
| `policy_id` | `VARCHAR(128)` | Governance policy when event represents a decision |
| `policy_version` | `VARCHAR(64)` | Evaluated policy version |
| `policy_decision` | `VARCHAR(32)` | `ALLOW`, `WARN`, `REDACT`, `REQUIRE_APPROVAL`, `BLOCK` |
| `event_body` | `TEXT` | Human-readable or raw event body |
| `attributes` | `JSONB` | Additional OTel/event attributes |

Recommended indexes for the POC:

- `(event_name, event_time DESC)`
- `trace_id`
- `agent_id`
- `(user_id, session_id)`
- `policy_decision`

### 11.3.4 `otlp_log` — Application, Agent, and Container Diagnostics

`otlp_log` stores diagnostic log records used for reliability investigation and AI SRE analysis.

| Field | PostgreSQL Type | Purpose |
| :--- | :--- | :--- |
| `log_id` | `BIGSERIAL` | Primary key |
| `log_time` | `TIMESTAMPTZ` | Log timestamp |
| `trace_id` | `VARCHAR(64)` | Correlated trace when available |
| `span_id` | `VARCHAR(64)` | Correlated span when available |
| `execution_id` | `VARCHAR(128)` | Logical execution |
| `severity_text` | `VARCHAR(16)` | `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` |
| `severity_number` | `INT` | OTel numeric severity |
| `application_id` | `VARCHAR(128)` | Application dimension |
| `agent_id` | `VARCHAR(128)` | Logical agent identifier |
| `agent_name` | `VARCHAR(256)` | Agent display name |
| `service_name` | `VARCHAR(256)` | Emitting service |
| `service_instance_id` | `VARCHAR(256)` | Emitting runtime instance |
| `environment` | `VARCHAR(64)` | Environment dimension |
| `location_id` | `VARCHAR(64)` | Site/location dimension |
| `body` | `TEXT` | Raw/unformatted log message |
| `log_attributes` | `JSONB` | Module, line, thread, exception, cloud, or other OTel attributes |

Recommended indexes for the POC:

- `log_time DESC`
- `trace_id`
- `execution_id`
- `severity_text`
- `agent_id`

### 11.3.5 OTel Correlation and Data-Quality Rules

For synthetic POC data, the following rules should be enforced so all seven scenarios remain internally consistent:

1. Every `agent_execution` record should have one root span in `otlp_span` with the same `execution_id` and `trace_id`.
2. Every child span should reference a valid `parent_span_id` within the same trace unless it intentionally represents a distributed boundary.
3. Failure scenarios should include an error span plus correlated exception/event and one or more diagnostic log records where appropriate.
4. Model spans should carry model/provider, token, latency, and cost attributes sufficient for Scenarios 3-5.
5. Tool spans should carry `tool_name` and `dependent_system` so the UI can identify impacted systems.
6. Governance demo executions should include evaluation/policy records in `otlp_event` and/or `governance_evaluation`.
7. `application_id`, `agent_id`, `agent_version`, `location_id`, `user_id`, and `session_id` should be consistently propagated across telemetry rows when known.
8. Historical dashboard values should reconcile with the underlying synthetic spans/metrics for the selected time window.
9. Prompt/output capture should respect the configured masking and retention policy even in synthetic data.
10. Unmapped telemetry should be preserved in JSON attribute fields rather than discarded.

---

## 11.4 Analytical, Governance, and Background-Processing Entities

### 11.4.1 `metric_rollup`

Stores dashboard-oriented time-window aggregates derived from `otlp_span`, `otlp_metric`, `otlp_event`, and `agent_execution`.

Key fields:

- `rollup_id`
- `metric_name`
- `window_start`
- `window_end`
- `application_id` (optional)
- `agent_id` (optional)
- `model_id` or `model_name` (optional)
- `tool_id` or `tool_name` (optional)
- `owner_team` (optional)
- `location_id` (optional)
- `metric_value`
- `sample_count`
- `percentile` (optional)
- `dimensions_json`

Examples include success rate, error rate, P50/P90/P95/P99 latency, token consumption, cost, and execution volume.

### 11.4.2 `platform_finding`

Represents a deterministic or analytical finding created by a sweeper, processor, governance evaluation, or user-triggered analysis.

Key fields:

- `finding_id`
- `finding_type` (`RELIABILITY`, `LATENCY`, `COST`, `TOKEN`, `GOVERNANCE`, `INVENTORY`)
- `severity`
- `application_id`
- `agent_id` (optional)
- `execution_id` (optional)
- `title`
- `description`
- `evidence_json`
- `baseline_json` (optional)
- `detected_at`
- `status`
- `source_job_run_id` (optional)
- `requires_ai_analysis`

Findings are the primary bridge between deterministic background processing and AI-assisted investigation.

### 11.4.3 `ai_analysis`

Stores the output of specialized AI agents such as the SRE Root Cause Agent, Token Optimization Agent, Governance Analysis Agent, or Executive SRE Summary Agent.

Key fields:

- `analysis_id`
- `finding_id` (optional)
- `execution_id` (optional)
- `analysis_agent_type`
- `invocation_type` (`USER`, `FUNCTION`, `SCHEDULED`)
- `summary`
- `likely_root_cause` (optional)
- `evidence_summary`
- `impacted_systems_json`
- `recommendations_json`
- `confidence`
- `model_used`
- `created_at`

The AI agent should reason over a bounded evidence package and may call approved Trillo AOS functions as tools for additional facts or calculations.

### 11.4.4 `governance_policy`

Represents configurable governance controls.

Key fields:

- `policy_id`
- `policy_name`
- `policy_type`
- `scope_type` (`ENTERPRISE`, `APPLICATION`, `AGENT`)
- `scope_id` (optional)
- `action` (`ALLOW`, `WARN`, `BLOCK`, `REDACT`, `REQUIRE_APPROVAL`)
- `configuration_json`
- `version`
- `enabled`
- `created_by`
- `created_at`
- `updated_at`

### 11.4.5 `governance_evaluation`

Represents the outcome of applying a policy or guardrail to an execution. This entity may reference or be materialized from corresponding `otlp_event` records where appropriate.

Key fields:

- `evaluation_id`
- `execution_id`
- `trace_id`
- `span_id` (optional)
- `policy_id`
- `policy_version`
- `evaluation_type`
- `score` (optional)
- `result` (`PASS`, `WARN`, `FAIL`, `BLOCKED`)
- `matched_rule` (optional)
- `details_json`
- `evaluated_at`

### 11.4.6 `administrative_audit`

Represents user and platform administrative activity required for auditability.

Key fields:

- `audit_event_id`
- `actor_user_id`
- `action`
- `resource_type`
- `resource_id`
- `old_value_json` (optional)
- `new_value_json` (optional)
- `result`
- `timestamp`
- `source_ip` (optional)
- `correlation_id` (optional)
- `previous_hash` (optional)
- `event_hash` (optional)
- `metadata_json`

Examples include changing a governance policy, changing ownership metadata, exporting evidence, or initiating an AI analysis.

### 11.4.7 `model_pricing`

Provides effective-dated model pricing used for deterministic cost calculation.

Key fields:

- `pricing_id`
- `provider_name`
- `model_name`
- `model_version` (optional)
- `effective_from`
- `effective_to` (optional)
- `input_token_price`
- `output_token_price`
- `cached_input_token_price` (optional)
- `reasoning_token_price` (optional)
- `currency`
- `pricing_source`
- `verified_at`

### 11.4.8 `analysis_baseline`

Stores historical baselines used by deterministic sweepers to identify regressions and anomalies.

Key fields:

- `baseline_id`
- `baseline_type`
- `application_id` (optional)
- `agent_id` (optional)
- `metric_name`
- `baseline_window_start`
- `baseline_window_end`
- `baseline_value`
- `statistics_json`
- `calculated_at`

### 11.4.9 `sweeper_run`

Tracks scheduled and event-driven background function executions.

Key fields:

- `job_run_id`
- `job_name`
- `job_type`
- `window_start`
- `window_end`
- `partition_key`
- `watermark_before`
- `watermark_after`
- `started_at`
- `completed_at`
- `status`
- `records_processed`
- `findings_created`
- `ai_agents_invoked`
- `error_message`

This entity supports batching, concurrency, retries, observability of background processing, and incremental processing without rescanning the entire historical dataset.

### 11.4.10 `optimization_recommendation`

Stores structured optimization opportunities generated from deterministic findings and, where appropriate, enriched by an AI optimization agent.

Key fields:

- `recommendation_id`
- `finding_id`
- `application_id`
- `agent_id`
- `model_name` (optional)
- `recommendation_type`
- `issue_summary`
- `evidence_json`
- `recommended_action`
- `estimated_monthly_savings_usd`
- `estimated_tokens_saved` (optional)
- `confidence`
- `analysis_start_time`
- `analysis_end_time`
- `execution_sample_count`
- `status`
- `disposition_notes` (optional)
- `validated_savings_usd` (optional)
- `created_at`

### 11.4.11 Recommended POC Tables / Trillo AOS Entities

The PostgreSQL POC should therefore contain the following logical tables or equivalent Trillo AOS entities:

**Business / inventory**

- `application`
- `agent`
- `agent_instance`
- `models`
- `tools`
- `external_systems`
- `agent_dependency`
- `agent_execution`

**OTel telemetry**

- `otlp_span`
- `otlp_metric`
- `otlp_event`
- `otlp_log`

**Analytics / intelligence**

- `metric_rollup`
- `platform_finding`
- `ai_analysis`
- `analysis_baseline`
- `sweeper_run`
- `optimization_recommendation`

**Governance / financial metadata**

- `governance_policy`
- `governance_evaluation`
- `administrative_audit`
- `model_pricing`

Historical trend data may be calculated from execution-level synthetic OTel data or pre-aggregated in `otlp_metric` / `metric_rollup` where that improves POC responsiveness.

## 11.5 Synthetic Dataset Requirements

The synthetic dataset should include enough history and variety to make the application feel operationally real.

Recommended characteristics:

- Multiple applications
- 15-25 logical agents
- Thousands of simulated agent instances across locations
- Multiple model providers and model tiers
- Multiple tools and enterprise systems
- Several agent versions
- At least 30-90 days of historical metrics
- Successful and failed executions
- Normal and anomalous latency periods
- Cost growth patterns
- Token-waste examples
- Governance pass and fail examples
- Unowned and metadata-incomplete agents

The same synthetic story should remain consistent across all seven scenarios.

---

# 12. Trillo AOS Role in the POC

Trillo AOS provides the application foundation for the POC, including the capabilities needed to rapidly build a secure, production-oriented experience.

The POC will use Trillo AOS for areas such as:

- Application data access
- Application APIs and server-side functions
- Authentication and user context
- Role-based access
- Application navigation and UI workflows
- Governance workflow implementation
- Audit-oriented application behavior
- Scheduled background analytical services and sweepers
- Findings, rollups, and baseline management
- AI-assisted diagnostic and optimization workflows
- Deployment and operational foundation

The POC is therefore not a disposable prototype. The intent is to carry the application, workflows, metadata model, access controls, and user experience forward to production while replacing the synthetic telemetry path with production ingestion and telemetry services.

---

# 13. POC Background Intelligence

The POC should demonstrate that the platform does more than display raw telemetry. Lightweight background analytical services should continuously convert synthetic executions into rollups, findings, trends, and optimization candidates that power the user experience.

These services are intentionally simple in the POC. They run against PostgreSQL and synthetic data using Trillo AOS scheduled/server-side functions. Their purpose is to create a realistic operational experience, not to prove production-scale batch processing.

## 13.1 Execution Modes

The POC uses three execution modes:

1. **Near-real-time processors** perform lightweight calculations when synthetic events are created or loaded.
2. **Periodic sweepers** analyze recent windows of data and create findings, rollups, baselines, and optimization candidates.
3. **On-demand AI agents** are invoked by the user to explain or investigate a specific finding, execution, or trend.

The intended processing pattern is:

```text
Synthetic Telemetry
       ↓
Deterministic Analytics / Sweepers
       ↓
Findings + Rollups + Baselines
       ↓
AI Analysis / Explanation
       ↓
Trillo AOS User Experience
```

AI agents should generally reason over a bounded set of evidence prepared by deterministic services rather than scan the entire raw telemetry dataset.

## 13.2 Deterministic Detection, Agentic Investigation

The POC should explicitly use the Trillo AOS bidirectional orchestration model: **functions may invoke agents, and agents may invoke functions as tools**.

The governing design principle is:

> **Functions own facts and computations; agents own interpretation and reasoning.**

Scheduled or event-driven functions perform deterministic work such as SQL queries, aggregation, percentile calculation, threshold evaluation, baseline comparison, cost calculation, token analysis, clustering, dependency lookup, and policy evaluation. When that computation produces a finding that benefits from interpretation, correlation, root-cause reasoning, or recommendation, the function may invoke a specialized AI agent with a bounded evidence package.

The AI agent may then call trusted Trillo AOS functions as tools to retrieve additional evidence or request authoritative calculations before producing its explanation or recommendation.

The two primary orchestration patterns are:

```text
Function / Sweeper -> AI Agent
Detect or compute a meaningful finding -> invoke a specialized agent for investigation or explanation

AI Agent -> Function Tool
Agent requests traces, logs, dependencies, baselines, rollups, policy results, or calculations -> function returns authoritative evidence
```

A typical reliability flow is:

```text
Reliability Health Sweeper
        |
        |-- Calculate error rate and baseline deviation
        |-- Create finding with representative trace IDs
        |
        +--> SRE Root Cause Agent
                 |
                 |-- get_finding()
                 |-- get_trace()
                 |-- get_related_logs()
                 |-- get_dependency_graph()
                 |-- get_historical_baseline()
                 |
                 +--> Persist evidence-grounded diagnosis
```

The POC should not use an AI agent to scan the entire telemetry dataset when the same result can be obtained deterministically. Agents should receive bounded evidence and use functions as tools when they need additional facts.

## 13.3 POC Background Services and Platform Agents

| Service / Agent | Type | POC Frequency | Primary Scope | Orchestration | POC Output |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Agent Inventory Reconciler** | Sweeper | Every 15 min | Application + agent | Function only | Last-seen state, active instances, versions, metadata-completeness findings |
| **Dependency Reconciler** | Sweeper | Every 15 min | Agent | Function only | Agent-to-model, tool, system, vector-store, and agent relationships |
| **Reliability Health Sweeper** | Sweeper | Every 5 min | Application + agent | Function -> SRE Agent for significant findings when configured | Success/error rollups, repeated-failure findings, degraded-agent findings |
| **Failure Clusterer** | Analytical service | Every 5 min | Recent failed executions | Function -> SRE Agent for selected/significant cluster when configured | Groups repeated failures by error signature/root span/dependency |
| **SRE Root Cause Agent** | AI agent | On demand or function-invoked | Selected execution or failure cluster | Agent -> functions for finding, trace, logs, dependencies, and baselines | Evidence-grounded probable root cause, impact, and recommended next action |
| **Latency Analyzer** | Sweeper | Every 5 min | Agent + model/tool | Function only; findings may be investigated by SRE Agent | P50/P90/P95/P99 rollups and bottleneck findings |
| **Performance Regression Analyzer** | Analytical service | Every 30 min | Agent + version | Function only; findings may be investigated by SRE Agent | Current-vs-baseline regression findings |
| **Cost Aggregator** | Sweeper | Every 15 min | Application/agent/model/owner | Function only | Cost and token rollups used by dashboards |
| **Cost Forecast Analyzer** | Analytical service | Daily or on demand | Application/owner | Function only | 30/60/90-day forecast data for the bonus scenario |
| **Token Efficiency Sweeper** | Sweeper | Hourly | Agent + model | Function -> Token Optimization Agent for selected candidates | High prompt/output ratio, repeated context, high-cost usage candidates |
| **Token Optimization Agent** | AI agent | On demand or function-invoked | Selected optimization candidate | Agent -> functions for token statistics, cost model, prompt evidence, and baselines | Explanation, recommendation, estimated savings, confidence |
| **Governance Audit Sweeper** | Sweeper | Hourly | Application/agent/user | Function only | Policy and audit findings across the synthetic dataset |
| **Metadata Completeness Sweeper** | Sweeper | Daily | Agent inventory | Function only | Unowned, purpose-missing, application-missing, or stale metadata findings |
| **Executive Health Aggregator** | Sweeper | Every 15 min | Enterprise/application | Function -> Executive SRE Summary Agent on demand | Reliability, cost, tokens, adoption, and governance summary metrics |
| **Executive SRE Summary Agent** | AI agent | On demand | Enterprise/application | Agent -> functions for findings, rollups, trends, and baselines | Plain-language summary of the most important findings and likely priorities |

The above frequencies are POC defaults and should be configurable rather than hard-coded.

## 13.4 Findings Rather Than Alerts

Alert delivery and outbound notifications such as email or Slack are **not required for the POC** and are outside the current POC evaluation scenarios.

Background services should therefore create **platform findings** rather than notifications.

A finding should contain at least:

- Finding ID
- Finding type
- Severity (`Info`, `Warning`, `Critical`)
- Application
- Agent when applicable
- Related model/tool/system when applicable
- Detection time
- Analysis window
- Short title
- Evidence summary
- Related metric or execution IDs
- Status (`Open`, `Acknowledged`, `Resolved`)
- Optional recommendation
- Optional estimated financial impact

Example:

```text
Finding: Inventory Tool Failure Cluster
Severity: Critical
Application: Enterprise Operations Platform
Agent: OrderProcessingAgent
Window: 10:40–10:45 AM
Evidence: 1,248 failed executions across 317 locations
Primary Dependency: inventory_lookup → ERP Inventory
Status: Open
```

The Executive Dashboard, Reliability Explorer, Token Optimization screen, and Governance screen can all consume this common findings model.

## 13.5 SRE Root Cause Agent

The SRE Root Cause Agent should operate on a selected execution or failure cluster and gather only the evidence needed for that investigation.

Conceptual tool flow:

```text
SRE Root Cause Agent
       │
       ├── Get execution and trace
       ├── Get correlated logs/events
       ├── Get failure cluster statistics
       ├── Get dependency topology
       ├── Get recent latency/reliability baseline
       ├── Get related impacted-agent findings
       └── Produce evidence-grounded analysis
```

The generated analysis should distinguish facts from inference and contain:

- Likely root cause
- Supporting evidence
- Impacted systems/agents
- What was ruled out when evidence supports it
- Recommended next action
- Confidence level

Example experience:

> **Likely Root Cause:** `inventory_lookup` is experiencing Inventory Gateway timeouts. 1,248 executions across 317 locations share the same failure signature. ERP Inventory latency increased from the recent baseline while model latency remained normal. The evidence indicates a tool/system-side issue rather than a model-performance problem.

## 13.6 POC Scheduling and Batching

For the synthetic POC dataset, each periodic service may execute as a Trillo AOS scheduled function.

A simple reusable job model should be used:

```text
job_type
window_start
window_end
partition_key
watermark
started_at
completed_at
status
records_processed
findings_created
error_message
```

Sweepers should process only the new or relevant analysis window wherever possible rather than rescan the entire historical dataset on every run.

Default partitions should use `application_id + agent_id` or groups of agents. A configurable worker pool may process multiple partitions concurrently. The implementation should use bounded workers/functions rather than creating an uncontrolled thread for every agent.

For POC scale, batching can remain simple. The design merely establishes the same processing contract that can later be backed by production telemetry services.

## 13.7 POC User Experience Integration

Background intelligence should appear naturally in the seven scenarios:

- **Inventory:** metadata completeness and dependency findings
- **Reliability:** failure clusters, reliability findings, and SRE root-cause analysis
- **Latency:** percentile rollups and regression/bottleneck findings
- **Cost:** cost/token rollups and forecast data
- **Optimization:** candidates created by token-efficiency analysis and explained by the Optimization Agent
- **Governance:** governance findings and audit summaries
- **Executive Dashboard:** consolidated findings, trends, and an optional Executive SRE Summary

The UI should allow every important finding to drill down to the underlying evidence.

---

# 14. Demonstration Storyline

The preferred demonstration should feel like one connected story rather than seven independent feature tours.

### Step 1: Start with the Executive Dashboard

Show that the ecosystem is generally healthy, but highlight:

- A recent reliability degradation
- A cost increase
- Two governance findings
- A token optimization opportunity

### Step 2: Investigate Reliability

Drill into the affected agent, locate the failed execution, identify the inventory dependency as root cause, and show impacted systems.

### Step 3: Analyze Latency

Show that the same or another agent has increased P95/P99 latency and identify the tool or retrieval bottleneck.

### Step 4: Understand Cost

Navigate from application cost to agent, model, and execution. Show forecast, chargeback, and showback.

### Step 5: Optimize Tokens

Show an evidence-backed recommendation with projected savings.

### Step 6: Review Governance

Inspect a policy finding, review the prompt/output, change a policy from `Warn` to `Block`, and show the resulting audit entry.

### Step 7: Close on Inventory

Show that every finding is tied back to a governed inventory containing owner, purpose, models, tools, systems, versions, and runtime instances.

This storyline reinforces that the product is a cohesive operational platform rather than a collection of dashboards.

---

# 15. POC Success Criteria

The POC will be considered successful when:

1. Every source RFP requirement can be demonstrated explicitly.
2. Evaluators can navigate the seven scenarios without understanding telemetry internals.
3. The UI maintains context while drilling from aggregate insight to execution evidence.
4. Data is visually coherent and internally consistent across scenarios.
5. The application feels responsive and polished.
6. The POC clearly communicates how the same Trillo AOS application transitions to production telemetry without redesigning the user experience.

