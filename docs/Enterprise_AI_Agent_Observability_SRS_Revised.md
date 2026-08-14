# **System Requirements Specification (SRS)**

## **Enterprise AI Agent Observability & Analytics Platform**

**Document Version:** 1.1  
**Target Platform:** Trillo AOS (Application Operating System)  
**Telemetry Standard:** OpenTelemetry (OTel), including applicable GenAI semantic conventions  
**Ingestion Processing:** OTLP gRPC/HTTP with Apache Arrow RecordBatches  
**Primary Storage:** PostgreSQL using denormalized telemetry and operational metadata tables

---

## **1. Purpose and Scope**

This system provides enterprise-wide discovery, observability, cost analysis, optimization, governance, and executive reporting for AI agents. It is designed to support the seven demonstration scenarios defined in the Wendy's RFP:

1. Agent Inventory & Discovery
2. Reliability Investigation
3. Latency Analysis
4. Cost & Token Visibility
5. Token Optimization
6. Governance & Auditability
7. Executive Dashboard

The design distinguishes between:

- **Logical agents:** Named, governed software capabilities such as `DriveThruOrderAgent`.
- **Agent instances:** Runtime deployments of a logical agent across stores, clusters, pods, versions, or environments.
- **Observed metadata:** Information discovered from telemetry, including agent identity, model usage, tool calls, runtime instances, versions, and dependencies.
- **Registered metadata:** Enterprise information maintained through configuration or integration, including owner, business purpose, application, cost center, contact, and governance classification.

---

## **2. System Architecture & Ingestion Design**

### **2.1 Ingestion Pipeline Architecture**

The platform ingests high-throughput OpenTelemetry Protocol (OTLP) gRPC/HTTP streams through an asynchronous ingestion pipeline. Incoming OTLP protobuf frames are validated, normalized, and converted into Apache Arrow RecordBatches for efficient in-memory processing. Telemetry is then placed into a durable write queue before acknowledgment and asynchronously persisted into PostgreSQL tables.

```text
[ GCP GKE / Agent Pods ]
             │
             │ OTLP gRPC / HTTP
             ▼
[ Trillo AOS Ingestion Gateway ]
             │
             ├── Validate and Normalize OTel Attributes
             ├── Convert to Arrow RecordBatches
             ├── Resolve Agent and Instance Identity
             ▼
[ Durable Write Queue / WAL ]
             │
             ├──► [ Inventory Discovery and Dependency Updates ]
             │
             └──► [ PostgreSQL Telemetry Storage ]
                    ├── otlp_span
                    ├── otlp_metric
                    ├── otlp_event
                    └── otlp_log
```

The durable queue or write-ahead mechanism prevents acknowledged telemetry from being lost if an ingestion worker is restarted before database persistence completes.

### **2.2 Agent and Instance Discovery Engine**

To support discovery across large numbers of stateless runtime instances and store locations:

1. The ingestion pipeline extracts available identity attributes, including:
   - `gen_ai.agent.id`
   - `gen_ai.agent.name`
   - `service.name`
   - `service.instance.id`
   - `service.version`
   - deployment environment
   - cloud project, cluster, namespace, pod, and store identifiers
2. The platform resolves the logical agent using `agent_id` or a configured identity mapping.
3. The platform resolves the runtime instance using `agent_id + service.instance.id`.
4. An in-memory cache is checked for the logical agent and instance identities.
5. On cache miss, the platform performs idempotent `UPSERT` operations into `agent_inventory` and `agent_instance`.
6. The platform records observed model, tool, system, vector-store, and agent-to-agent dependencies in `agent_dependency`.
7. The platform updates `first_seen_at`, `last_seen_at`, version, environment, and runtime health indicators.
8. Registered business metadata is merged from platform configuration, CMDB, service catalog, deployment metadata, or administrative entry.
9. Agents that are observed but lack required ownership or purpose metadata are marked **Unregistered** or **Metadata Incomplete** for governance follow-up.

The platform automatically discovers all instrumented agents that emit telemetry through the configured ingestion path. It does not claim discovery of uninstrumented or disconnected codebases.

### **2.3 Common Analytics Dimensions**

The following dimensions must be available across trace, latency, cost, token, governance, and executive reporting:

- Application ID and application name
- Agent ID and agent name
- Agent version
- Runtime instance ID
- Owner team and owner contact
- Cost center
- Model provider and model name
- Tool and dependent system
- Environment
- Store ID or business location
- User ID and session ID, subject to access controls
- Trace ID and execution ID
- Time range

---

## **3. Functional Requirements & UI Scenarios**

### **Scenario 1: Agent Inventory & Discovery**

#### **1.1 Requirements**

The platform shall:

- Automatically discover instrumented logical agents and their running instances from incoming OTLP telemetry.
- Display registered metadata, including owner team, contact email, business purpose, application, cost center, and deployment environment.
- Display observed metadata, including active instances, agent versions, last activity, models used, tools used, and dependent systems.
- Distinguish logical agents from runtime instances.
- Identify agents with missing ownership, business purpose, or application metadata.
- Support dependency relationships to models, tools, systems, vector stores, and other agents.
- Provide filters for application, owner, environment, store, version, model, status, and metadata completeness.

#### **1.2 User Interface & Workflow Specification**

**Screen Layout:** Filterable inventory grid with a global search bar and summary cards.

**Summary Cards:**

- Logical Agents
- Active Instances
- Unowned Agents
- Metadata-Incomplete Agents
- Degraded Agents
- Inactive Agents

**UI Workflow:**

1. The user navigates to **Agent Inventory**.
2. The inventory grid lists logical agents with status badges: `Active`, `Degraded`, `Inactive`, `Unregistered`, or `Metadata Incomplete`.
3. The user searches by agent name, application, owner, store, model, tool, or system.
4. Selecting `DriveThruOrderAgent` opens the **Agent Details Drawer**.
5. The header displays:
   - Owner: `POS-Automation-Engineering`
   - Contact: `ai-ops@wendys.com`
   - Business Purpose: `Voice AI Drive-Thru Order Processing`
   - Application: `Digital Ordering Platform`
   - Cost Center: `CC-4102`
   - Active Instances: `10,482`
   - Active Stores: `7,420`
   - Deployed Versions: `3`
6. The **Instances** tab lists runtime instances by store, cluster, version, status, first seen, and last seen.
7. The **Dependency Topology Graph** displays the logical relationships:

```text
                         ┌──► Model: gemini-1.5-pro
                         │
DriveThruOrderAgent ─────┼──► Tool: pos_inventory_lookup ──► System: Oracle POS DB
                         │
                         └──► Tool: payment_gateway_api ───► System: Payment Gateway
```

8. The **Metadata Quality** panel identifies missing or stale ownership and business metadata and allows an authorized administrator to update it.

#### **1.3 Evaluation Scoring Mapping**

| Criteria | Scoring Target | Platform Capability |
| :--- | :---: | :--- |
| **Easy to locate agents** | **5 / 5** | Global search, filters, status badges, application grouping, and automatic discovery of instrumented agents. |
| **Owner visibility** | **5 / 5** | Owner team, contact, cost center, and metadata-quality status appear in the grid and details drawer. |
| **Dependency visibility** | **5 / 5** | Interactive topology shows models, tools, systems, vector stores, and agent-to-agent dependencies. |
| **Inventory completeness** | **5 / 5** | The platform discovers all instrumented agents observed through configured telemetry and explicitly flags unregistered or metadata-incomplete agents. |

---

### **Scenario 2: Reliability Investigation**

#### **2.1 Requirements**

The platform shall:

- Surface failed agent executions in near real time.
- Locate a failed execution by agent, application, store, user, session, time range, error type, or trace ID.
- Display an interactive waterfall trace from the top-level agent execution through nested agent, model, retrieval, and tool spans.
- Correlate spans, events, and container logs using `trace_id` and `span_id`.
- Display execution inputs, outputs, tool arguments, model responses, exceptions, and relevant logs, subject to RBAC and data-masking policies.
- Identify the likely root-cause span and the systems directly or potentially impacted by the failure.
- Display historical error-rate trends and compare current behavior with a prior baseline.
- Optionally use an embedded **AI SRE Root Cause Agent** to summarize evidence in plain language.

#### **2.2 User Interface & Workflow Specification**

**Screen Layout:** Split-pane **Reliability Explorer** with execution list, trace waterfall, diagnostic summary, payload inspector, and impacted-systems panel.

**UI Workflow:**

1. The user opens **Reliability Explorer** and filters status to `ERROR`.
2. The user selects a failed execution with `trace_id: 8f3b921a4001c9b8`.
3. The **Waterfall Inspector** displays the complete execution trace and highlights `execute_tool pos_inventory_lookup` as the first failing span.
4. The user reviews:
   - Input prompt or request
   - Model request and response
   - Tool arguments
   - Tool output or exception
   - Correlated application and container logs
5. The user clicks **Analyze Failure with AI SRE**.
6. The diagnostic panel displays an evidence-backed summary:

> **Root Cause:** `pos_inventory_lookup` failed because the POS Gateway returned HTTP 504. Correlated container logs show `urllib3.exceptions.MaxRetryError` while connecting to the Oracle POS endpoint.

7. The **Impacted Systems** panel displays:

| Dependency | Impact | Evidence |
| :--- | :--- | :--- |
| `pos_inventory_lookup` | Failed | HTTP 504 on selected trace |
| `Oracle POS DB` | Unreachable | Connection retries exhausted |
| `DriveThruOrderAgent` | Degraded | Execution failed at dependent tool |
| `OrderSuggestionAgent` | Potentially impacted | Shares the same tool dependency |

8. The **Historical Trend** panel compares the current failure rate with the previous seven-day baseline and groups errors by root cause, agent version, store, tool, and system.

#### **2.3 Evaluation Scoring Mapping**

| Criteria | Scoring Target | Platform Capability |
| :--- | :---: | :--- |
| **Ease of troubleshooting** | **5 / 5** | Unified search, trace, payload, logs, root-cause summary, and dependency impact in one workflow. |
| **Trace visualization** | **5 / 5** | Color-coded waterfall showing timing, nesting, status, and span type for every execution step. |
| **Error detail** | **5 / 5** | Exceptions, HTTP status, tool arguments, model response, logs, and correlated evidence. |
| **Historical trend visibility** | **5 / 5** | Error trends, baseline comparison, and grouping by version, location, model, tool, system, and root cause. |

---

### **Scenario 3: Latency Analysis**

#### **3.1 Requirements**

The platform shall:

- Report end-to-end execution duration.
- Break down latency into model, tool, retrieval, agent orchestration, queueing, and other framework overhead.
- Report model latency and tool latency independently.
- Display P50, P90, P95, and P99 latency trends.
- Compare latency by application, agent, version, model, tool, environment, store, and time period.
- Link aggregate latency anomalies to the contributing executions and spans.
- Identify the largest bottleneck for a selected trace or time range.

#### **3.2 User Interface & Workflow Specification**

**Screen Layout:** Aggregated latency charts above an interactive trace flamegraph or waterfall.

**UI Workflow:**

1. The user opens **Latency Analytics** and selects `DriveThruOrderAgent`.
2. KPI cards show:
   - End-to-End P95: `3.1 seconds`
   - Model P95: `930 ms`
   - Tool P95: `1.92 seconds`
   - Change vs. Previous 7 Days: `+31%`
3. The **Latency Breakdown Chart** displays:
   - Model Latency: `30%`
   - External Tool Latency: `62%`
   - Orchestration and Other: `8%`
4. The user selects a P99 spike on the historical chart.
5. The trace view filters to executions contributing to the spike.
6. The flamegraph pinpoints the bottleneck: `query_vector_store` consumed `2,200 ms` of a `3,100 ms` execution.
7. The user compares latency by model, agent version, tool, store group, and deployment environment.

#### **3.3 RFP Coverage**

| RFP Requirement | Platform Capability |
| :--- | :--- |
| Latency reporting | End-to-end and percentile-based latency reporting |
| Bottleneck identification | Span-level waterfall and flamegraph with largest contributor highlighted |
| Model latency | Model-specific duration and percentile trends |
| Tool latency | Tool-specific duration and percentile trends |
| Historical trends | Time-series comparison and prior-period baseline |

---

### **Scenario 4: Cost & Token Visibility**

#### **4.1 Requirements**

The platform shall:

- Track input, output, cached, reasoning, and total tokens when provided by the model provider or instrumentation.
- Calculate cost using an effective-dated model-pricing registry.
- Preserve provider-reported cost when available and identify whether a cost is provider-reported or platform-estimated.
- Report cost and token consumption by:
  - Agent
  - Model
  - Application
  - Owner team
  - Cost center
  - Store or location
  - Environment
  - Time period
- Display historical cost and token trends.
- Provide drill-down from aggregate cost to individual executions.

**Bonus Capabilities:**

- 30-, 60-, and 90-day forecasting
- Budget thresholds and anomaly alerts
- Internal chargeback by cost center or owner
- Showback reports for visibility without financial posting

#### **4.2 User Interface & Workflow Specification**

**Screen Layout:** **Cost & Token Economics** dashboard with pivot tabs:

- By Application
- By Agent
- By Model
- By Owner
- By Cost Center
- By Store

**UI Workflow:**

1. The user opens **Cost & Token Economics**.
2. KPI cards display:
   - Month-to-Date Cost: `$14,250`
   - Total Tokens: `185M`
   - Average Cost per Execution: `$0.012`
   - Forecasted Month-End Cost: `$18,900`
3. The user selects **By Application** and sees cost and token usage for the `Digital Ordering Platform`.
4. The user drills into `DriveThruOrderAgent`, then into `gemini-1.5-pro`, and finally into individual traces.
5. The **Historical Trend** chart displays daily cost, token consumption, execution volume, and cost per execution.
6. The **Chargeback** view groups monthly cost by owner team and cost center and exports CSV or JSON.
7. The **Showback** view presents the same allocation without generating an accounting transaction.
8. The **Forecasting** panel displays a forecast range and identifies its input assumptions, including recent execution volume and store growth.

#### **4.3 Pricing and Cost Calculation**

The model-pricing registry shall support:

- Provider
- Model name and model version
- Effective start and end dates
- Input-token price
- Output-token price
- Cached-token price, where applicable
- Additional provider-specific units
- Currency
- Pricing source and last verification date

Historical costs shall be calculated using the pricing record effective at the time of execution.

#### **4.4 RFP Coverage**

| RFP Requirement | Platform Capability |
| :--- | :--- |
| Cost by agent | Agent pivot and trace-level drill-down |
| Cost by model | Provider and model pivot |
| Cost by application | First-class application dimension and application pivot |
| Cost by owner | Owner team and cost-center pivots |
| Tokens by agent | Input/output/total token reporting by agent |
| Tokens by model | Token reporting by provider and model |
| Historical trends | Daily and monthly trends with prior-period comparison |
| Forecasting | 30/60/90-day projections |
| Chargeback | Allocated reports by cost center and owner |
| Showback | Non-posting visibility reports |

---

### **Scenario 5: Token Optimization**

#### **5.1 Requirements**

The platform shall identify:

- Excessive system or user prompts
- High input-to-output token ratios
- Repeated context across multi-turn sessions
- Context that is retrieved but not used
- Duplicate instructions sent on every turn
- Model over-provisioning for simple tasks
- High-cost agents, applications, owners, or workflows
- Opportunities for prompt caching, summarization, retrieval tuning, batching, or model right-sizing

Recommendations shall include supporting evidence, estimated savings, confidence, and the population of executions analyzed.

#### **5.2 User Interface & Workflow Specification**

**Screen Layout:** Recommendation feed ranked by projected savings, confidence, and operational impact.

**UI Workflow:**

1. The user opens **Token Optimization**.
2. The platform displays a recommendation:

| Field | Example |
| :--- | :--- |
| **Issue** | Model over-provisioning in `OrderRouterAgent` |
| **Evidence** | 94% of executions return fewer than 20 output tokens; average input is 2,480 tokens and average output is 7 tokens |
| **Recommendation** | Evaluate a lower-cost model for routing classification |
| **Projected Impact** | Estimated savings of `$3,200 per month` |
| **Confidence** | High, based on 280,000 executions over 30 days |

3. The user opens the **Prompt Inspector**.
4. The inspector highlights repeated static instructions sent during later turns of a session.
5. The platform recommends prompt caching or session summarization and estimates the number of tokens and dollars that could have been avoided.
6. The user filters recommendations by application, owner, agent, model, impact, and recommendation type.
7. An authorized user can mark a recommendation as `Accepted`, `Dismissed`, `In Progress`, or `Validated` and record measured savings after implementation.

#### **5.3 RFP Coverage**

| RFP Requirement | Platform Capability |
| :--- | :--- |
| Excessive prompts | Prompt-length, repetition, and input/output-ratio analysis |
| Wasteful token usage | Repeated context and unused retrieval detection |
| Expensive models | Cost and task-complexity comparison by model |
| Optimization recommendations | Evidence-backed, dollar-quantified recommendations with confidence |

---

### **Scenario 6: Governance & Auditability**

#### **6.1 Requirements**

The platform shall:

- Maintain a searchable, access-controlled, tamper-evident record of agent activity.
- Record user requests, prompts, model outputs, tool calls, tool outputs, evaluations, policy decisions, and administrative changes, subject to configured data-retention and privacy policies.
- Support search by application, agent, user, session, trace, owner, policy, decision, and time range.
- Apply RBAC and field-level masking to sensitive prompt, output, and tool data.
- Support governance policies with actions such as `Allow`, `Warn`, `Redact`, `Require Approval`, and `Block`.
- Record policy version, matched rule, decision, and evidence for each evaluated execution.
- Export an audit evidence package with manifest, selected records, hashes, policy versions, and export metadata.
- Record administrative user activity, including policy, ownership, and metadata changes.

#### **6.2 Governance Control Examples**

The policy interface shall support controls such as:

| Control | Example Action |
| :--- | :--- |
| PII detected in prompt | Redact or block |
| PCI data detected | Block |
| Unapproved model | Block |
| Unapproved tool or system | Require approval or block |
| Prompt or output logging | Enable, disable, or mask by policy |
| Retention | 30, 90, 365 days, or configured period |
| High-risk transaction | Require human approval |
| Agent without owner | Warn or prevent production promotion |

#### **6.3 User Interface & Workflow Specification**

**Screen Layout:** **Governance & Audit** workspace with Audit Trail, Policy Management, Findings, and Evidence Export tabs.

**Audit Workflow:**

1. The auditor opens **Governance & Audit**.
2. The auditor searches by `enduser.id`, `session_id`, agent, application, trace, or policy decision.
3. Selecting an execution opens the **Compliance Inspection Panel**.
4. The panel displays:
   - Prompt and output, subject to masking permissions
   - User, session, application, agent, version, and owner
   - Models, tools, and systems used
   - Guardrail and evaluation results
   - Policy decisions and matched policy version
   - Related logs and trace evidence
5. Credit card numbers and phone numbers are displayed as `[PCI_REDACTED]` or `[PII_REDACTED]` to users without the required permission.

**Policy Workflow:**

1. An authorized governance administrator opens **Policy Management**.
2. The administrator selects the policy `PCI Data Protection`.
3. The administrator changes the configured action from `Warn` to `Block` and publishes a new policy version.
4. A test execution is evaluated against the policy and returns `BLOCKED`.
5. The audit record captures the actor, policy version, prior value, new value, timestamp, and test outcome.

**Evidence Export Workflow:**

1. The auditor selects an application, date range, agents, and policy findings.
2. The auditor clicks **Export Audit Evidence Package**.
3. The platform produces a JSON/CSV package containing:
   - Export manifest
   - Selected audit records
   - Policy versions
   - Evaluation and decision evidence
   - SHA-256 hashes for included files
   - Export actor and timestamp
4. When configured, the manifest is digitally signed using an enterprise-managed key.

#### **6.4 Tamper-Evidence Requirements**

To support the claim of tamper evidence, the platform shall use one or more of the following configured controls:

- Append-only permissions for audit ingestion identities
- Separate read and write roles
- Hash chaining or signed batch manifests
- Periodic export to immutable or retention-locked object storage
- Administrative change audit records
- Verification tooling for exported evidence packages

The platform shall not describe ordinary mutable PostgreSQL records as inherently immutable.

#### **6.5 RFP Coverage**

| RFP Requirement | Platform Capability |
| :--- | :--- |
| Audit agent activity | Searchable execution and action history |
| Review prompts | Permission-aware prompt viewer |
| Review outputs | Permission-aware completion and tool-output viewer |
| Review user activity | User/session activity plus administrative-change audit |
| Apply governance controls | Versioned policy editor with allow, warn, redact, approval, and block actions |
| Export audit evidence | Manifested JSON/CSV package with hashes and optional digital signature |

---

### **Scenario 7: Executive Dashboard**

#### **7.1 Requirements**

The platform shall provide a single executive view answering:

> **How healthy is our AI ecosystem?**

The dashboard shall consolidate:

- Reliability
- Latency
- Cost
- Token usage
- Adoption
- Governance findings
- Optimization potential

Each category shall show current value, prior-period comparison, status, and drill-down capability.

#### **7.2 User Interface & Workflow Specification**

**Screen Layout:** Executive scorecard with category status, trends, and prioritized findings.

```text
┌────────────────────────────────────────────────────────────────────────────────────┐
│                       AI ECOSYSTEM HEALTH: HEALTHY                                  │
├──────────────────────────┬──────────────────────────┬───────────────────────────────┤
│ RELIABILITY & LATENCY    │ COST & TOKEN ECONOMICS   │ ADOPTION & VOLUME             │
│ Success Rate: 99.2%      │ MTD Cost: $14,250        │ Active Agents: 18             │
│ Change: +0.3%            │ Change: +12%             │ Active Stores: 7,420          │
│ P95 Latency: 1.1 sec     │ Total Tokens: 185M       │ Executions: 1.2M              │
│ Status: Healthy          │ Forecast: $18,900        │ Adoption Change: +14%         │
├──────────────────────────┴──────────────────────────┴───────────────────────────────┤
│ GOVERNANCE & OPTIMIZATION                                                          │
│ Guardrail Pass Rate: 99.9%       Open High-Risk Findings: 2                        │
│ Unowned Agents: 3                 Metadata-Incomplete Agents: 5                     │
│ Optimization Potential: $3,200/month                                               │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**UI Workflow:**

1. The executive opens **AI Ecosystem Health**.
2. The dashboard displays status and trends for reliability, cost, tokens, adoption, and governance.
3. The executive selects **Open High-Risk Findings** to drill into the associated applications, agents, owners, and policies.
4. The executive selects the cost trend to view applications and owners contributing to the increase.
5. The executive selects adoption to view active users, sessions, stores, agents, and execution growth.

#### **7.3 Health Status Calculation**

The primary status shall be expressed as `Healthy`, `Needs Attention`, or `Critical` rather than an unexplained composite percentage.

Category statuses shall be calculated from configurable thresholds. If a numeric composite score is displayed, the platform shall expose:

- Category weights
- Metric thresholds
- Calculation period
- Missing-data treatment
- The metrics that caused the status

#### **7.4 Adoption Definition**

Adoption reporting shall include configurable measures such as:

- Active agents
- Active applications
- Active users
- Active sessions
- Active stores or locations
- Total executions
- New agents or applications onboarded
- Repeat usage rate

---

## **4. Cross-Cutting Requirements**

### **4.1 Security and Access Control**

- Integrate with enterprise identity providers supported by Trillo AOS.
- Enforce role-based access to applications, agents, prompts, outputs, user identifiers, costs, policies, and exports.
- Support field-level masking for PII, PCI, secrets, and other configured sensitive data.
- Record access to sensitive audit content and evidence exports.

### **4.2 Data Retention and Privacy**

- Configure retention independently for traces, logs, prompts, outputs, metrics, and audit records.
- Support disabling or sampling prompt and output capture by application or policy.
- Support deletion or anonymization workflows when required by enterprise policy.
- Preserve aggregate metrics when detailed payloads expire, where permitted.

### **4.3 Data Quality**

- Identify missing mandatory telemetry attributes.
- Display instrumentation coverage by application, agent, and environment.
- Mark cost as estimated when provider-reported cost is unavailable.
- Mark dependencies as observed, registered, or inferred.
- Display the last-seen timestamp and evidence supporting each observed dependency.

### **4.4 Scalability and Performance**

- Support horizontal scaling of OTLP ingestion workers.
- Batch database writes using Arrow RecordBatches or equivalent internal representation.
- Use time-based partitioning for high-volume telemetry tables where appropriate.
- Maintain rollup tables or materialized views for executive, cost, token, reliability, and latency dashboards.
- Define service-level objectives after benchmark testing rather than embedding unverified latency claims in the specification.

---

# **Appendix A: PostgreSQL Data Model**

The following schema provides the core logical design. Production deployment may add partitioning, retention policies, row-level security, materialized views, and storage-specific optimizations.

```sql
-- =============================================================================
-- 1. APPLICATION CATALOG
-- =============================================================================
CREATE TABLE application (
    application_id VARCHAR(128) PRIMARY KEY,
    application_name VARCHAR(256) NOT NULL,
    owner_team VARCHAR(128),
    owner_email VARCHAR(256),
    cost_center VARCHAR(64),
    business_purpose TEXT,
    environment_scope JSONB DEFAULT '[]'::jsonb,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_applications_name ON application(application_name);
CREATE INDEX idx_applications_owner ON application(owner_team);
CREATE INDEX idx_applications_cost_center ON application(cost_center);

-- =============================================================================
-- 2. LOGICAL AGENT INVENTORY
-- =============================================================================
CREATE TABLE agent_inventory (
    agent_id VARCHAR(128) PRIMARY KEY,
    agent_name VARCHAR(256) NOT NULL,
    application_id VARCHAR(128) REFERENCES application(application_id),
    service_name VARCHAR(256),
    owner_team VARCHAR(128),
    owner_email VARCHAR(256),
    cost_center VARCHAR(64),
    business_purpose TEXT,
    governance_classification VARCHAR(64),
    registration_status VARCHAR(32) DEFAULT 'UNREGISTERED',
        -- REGISTERED, UNREGISTERED, METADATA_INCOMPLETE
    operational_status VARCHAR(32) DEFAULT 'INACTIVE',
        -- ACTIVE, DEGRADED, INACTIVE
    first_seen_at TIMESTAMPTZ,
    last_seen_at TIMESTAMPTZ,
    metadata_updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    metadata_source VARCHAR(64),
        -- TELEMETRY, ADMIN, CMDB, SERVICE_CATALOG, DEPLOYMENT
    attributes JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_agent_inventory_name ON agent_inventory(agent_name);
CREATE INDEX idx_agent_inventory_application ON agent_inventory(application_id);
CREATE INDEX idx_agent_inventory_owner ON agent_inventory(owner_team);
CREATE INDEX idx_agent_inventory_status ON agent_inventory(operational_status);
CREATE INDEX idx_agent_inventory_registration ON agent_inventory(registration_status);

-- =============================================================================
-- 3. AGENT RUNTIME INSTANCES
-- =============================================================================
CREATE TABLE agent_instance (
    agent_instance_id BIGSERIAL PRIMARY KEY,
    agent_id VARCHAR(128) NOT NULL REFERENCES agent_inventory(agent_id),
    service_instance_id VARCHAR(256) NOT NULL,
    service_version VARCHAR(128),
    environment VARCHAR(64),
    store_id VARCHAR(64),
    gcp_project_id VARCHAR(128),
    cluster_name VARCHAR(128),
    namespace_name VARCHAR(128),
    pod_name VARCHAR(256),
    operational_status VARCHAR(32) DEFAULT 'ACTIVE',
    first_seen_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    attributes JSONB DEFAULT '{}'::jsonb,
    UNIQUE (agent_id, service_instance_id)
);

CREATE INDEX idx_agent_instance_agent ON agent_instance(agent_id);
CREATE INDEX idx_agent_instance_store ON agent_instance(store_id);
CREATE INDEX idx_agent_instance_version ON agent_instance(service_version);
CREATE INDEX idx_agent_instance_last_seen ON agent_instance(last_seen_at DESC);

-- =============================================================================
-- 4. AGENT DEPENDENCIES
-- =============================================================================
CREATE TABLE agent_dependency (
    dependency_id BIGSERIAL PRIMARY KEY,
    agent_id VARCHAR(128) NOT NULL REFERENCES agent_inventory(agent_id),
    dependency_type VARCHAR(32) NOT NULL,
        -- MODEL, TOOL, SYSTEM, VECTOR_STORE, AGENT
    dependency_key VARCHAR(256) NOT NULL,
    dependency_name VARCHAR(256) NOT NULL,
    parent_dependency_id BIGINT REFERENCES agent_dependency(dependency_id),
    relationship_type VARCHAR(64) DEFAULT 'USES',
    discovery_source VARCHAR(32) NOT NULL,
        -- OBSERVED, REGISTERED, INFERRED
    confidence NUMERIC(5,4),
    first_seen_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    evidence_trace_id VARCHAR(64),
    attributes JSONB DEFAULT '{}'::jsonb,
    UNIQUE (agent_id, dependency_type, dependency_key)
);

CREATE INDEX idx_agent_dependency_agent ON agent_dependency(agent_id);
CREATE INDEX idx_agent_dependency_type ON agent_dependency(dependency_type);
CREATE INDEX idx_agent_dependency_name ON agent_dependency(dependency_name);

-- =============================================================================
-- 5. OTLP SPANS: EXECUTION, MODEL, TOOL, AND RETRIEVAL TRACES
-- =============================================================================
CREATE TABLE otlp_span (
    span_id VARCHAR(64) PRIMARY KEY,
    trace_id VARCHAR(64) NOT NULL,
    parent_span_id VARCHAR(64),
    execution_id VARCHAR(128),
    name VARCHAR(512) NOT NULL,
    span_kind VARCHAR(32),
    span_category VARCHAR(32),
        -- AGENT, MODEL, TOOL, RETRIEVAL, ORCHESTRATION, OTHER
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    duration_ms DOUBLE PRECISION NOT NULL,
    status_code VARCHAR(32) NOT NULL,
    status_message TEXT,

    -- Application, agent, instance, and ownership dimensions
    application_id VARCHAR(128),
    application_name VARCHAR(256),
    agent_id VARCHAR(128),
    agent_name VARCHAR(256),
    agent_version VARCHAR(128),
    service_name VARCHAR(256),
    service_instance_id VARCHAR(256),
    environment VARCHAR(64),
    store_id VARCHAR(64),
    owner_team VARCHAR(128),
    cost_center VARCHAR(64),

    -- User and session dimensions
    user_id VARCHAR(256),
    session_id VARCHAR(256),

    -- GenAI and dependency attributes
    operation_name VARCHAR(128),
    provider_name VARCHAR(128),
    request_model VARCHAR(256),
    response_model VARCHAR(256),
    tool_name VARCHAR(256),
    dependent_system VARCHAR(256),

    -- Token and cost values
    input_tokens BIGINT DEFAULT 0,
    output_tokens BIGINT DEFAULT 0,
    cached_tokens BIGINT DEFAULT 0,
    reasoning_tokens BIGINT DEFAULT 0,
    total_tokens BIGINT DEFAULT 0,
    estimated_cost_usd NUMERIC(18,8) DEFAULT 0,
    reported_cost_usd NUMERIC(18,8),
    cost_source VARCHAR(32),
        -- PROVIDER_REPORTED, PLATFORM_ESTIMATED
    pricing_record_id BIGINT,

    -- Payloads, subject to capture and masking policy
    prompt_text TEXT,
    completion_text TEXT,
    tool_arguments JSONB,
    tool_result JSONB,
    raw_attributes JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_spans_trace ON otlp_span(trace_id);
CREATE INDEX idx_spans_execution ON otlp_span(execution_id);
CREATE INDEX idx_spans_time ON otlp_span(start_time DESC);
CREATE INDEX idx_spans_application ON otlp_span(application_id);
CREATE INDEX idx_spans_agent ON otlp_span(agent_id);
CREATE INDEX idx_spans_status ON otlp_span(status_code);
CREATE INDEX idx_spans_model ON otlp_span(response_model);
CREATE INDEX idx_spans_tool ON otlp_span(tool_name);
CREATE INDEX idx_spans_store ON otlp_span(store_id);
CREATE INDEX idx_spans_user_session ON otlp_span(user_id, session_id);

-- =============================================================================
-- 6. OTLP METRICS: RAW OR AGGREGATED METRIC POINTS
-- =============================================================================
CREATE TABLE otlp_metric (
    metric_id BIGSERIAL PRIMARY KEY,
    metric_name VARCHAR(256) NOT NULL,
    metric_time TIMESTAMPTZ NOT NULL,
    aggregation_period VARCHAR(32),

    application_id VARCHAR(128),
    application_name VARCHAR(256),
    agent_id VARCHAR(128),
    agent_name VARCHAR(256),
    agent_version VARCHAR(128),
    model_name VARCHAR(256),
    tool_name VARCHAR(256),
    owner_team VARCHAR(128),
    cost_center VARCHAR(64),
    environment VARCHAR(64),
    store_id VARCHAR(64),
    token_type VARCHAR(32),

    metric_value DOUBLE PRECISION NOT NULL,
    sample_count BIGINT DEFAULT 1,
    dimensions JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_metrics_name_time ON otlp_metric(metric_name, metric_time DESC);
CREATE INDEX idx_metrics_application ON otlp_metric(application_id);
CREATE INDEX idx_metrics_agent ON otlp_metric(agent_id);
CREATE INDEX idx_metrics_model ON otlp_metric(model_name);
CREATE INDEX idx_metrics_owner ON otlp_metric(owner_team);

-- =============================================================================
-- 7. OTLP EVENTS: EXCEPTIONS, EVALUATIONS, AND EXECUTION EVENTS
-- =============================================================================
CREATE TABLE otlp_event (
    event_id BIGSERIAL PRIMARY KEY,
    trace_id VARCHAR(64),
    span_id VARCHAR(64),
    execution_id VARCHAR(128),
    event_name VARCHAR(256) NOT NULL,
    event_time TIMESTAMPTZ NOT NULL,

    application_id VARCHAR(128),
    agent_id VARCHAR(128),
    user_id VARCHAR(256),
    session_id VARCHAR(256),

    eval_metric_name VARCHAR(256),
    eval_score DOUBLE PRECISION,
    eval_label VARCHAR(32),
    policy_id VARCHAR(128),
    policy_version VARCHAR(64),
    policy_decision VARCHAR(32),

    event_body TEXT,
    attributes JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_events_name_time ON otlp_event(event_name, event_time DESC);
CREATE INDEX idx_events_trace ON otlp_event(trace_id);
CREATE INDEX idx_events_agent ON otlp_event(agent_id);
CREATE INDEX idx_events_user_session ON otlp_event(user_id, session_id);
CREATE INDEX idx_events_policy_decision ON otlp_event(policy_decision);

-- =============================================================================
-- 8. OTLP LOGS: APPLICATION AND CONTAINER DIAGNOSTICS
-- =============================================================================
CREATE TABLE otlp_log (
    log_id BIGSERIAL PRIMARY KEY,
    log_time TIMESTAMPTZ NOT NULL,
    trace_id VARCHAR(64),
    span_id VARCHAR(64),
    execution_id VARCHAR(128),

    severity_text VARCHAR(16) NOT NULL,
    severity_number INT,
    application_id VARCHAR(128),
    agent_id VARCHAR(128),
    agent_name VARCHAR(256),
    service_name VARCHAR(256),
    service_instance_id VARCHAR(256),
    environment VARCHAR(64),
    store_id VARCHAR(64),

    body TEXT NOT NULL,
    log_attributes JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_logs_time ON otlp_log(log_time DESC);
CREATE INDEX idx_logs_trace ON otlp_log(trace_id);
CREATE INDEX idx_logs_execution ON otlp_log(execution_id);
CREATE INDEX idx_logs_severity ON otlp_log(severity_text);
CREATE INDEX idx_logs_agent ON otlp_log(agent_id);

-- =============================================================================
-- 9. MODEL PRICING REGISTRY
-- =============================================================================
CREATE TABLE model_pricing (
    pricing_record_id BIGSERIAL PRIMARY KEY,
    provider_name VARCHAR(128) NOT NULL,
    model_name VARCHAR(256) NOT NULL,
    model_version VARCHAR(256),
    effective_from TIMESTAMPTZ NOT NULL,
    effective_to TIMESTAMPTZ,
    input_price_per_million NUMERIC(18,8),
    output_price_per_million NUMERIC(18,8),
    cached_input_price_per_million NUMERIC(18,8),
    reasoning_price_per_million NUMERIC(18,8),
    currency VARCHAR(8) DEFAULT 'USD',
    pricing_source TEXT,
    verified_at TIMESTAMPTZ,
    attributes JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_model_pricing_lookup
    ON model_pricing(provider_name, model_name, effective_from DESC);

-- =============================================================================
-- 10. GOVERNANCE POLICIES AND VERSIONS
-- =============================================================================
CREATE TABLE governance_policy (
    policy_id VARCHAR(128) PRIMARY KEY,
    policy_name VARCHAR(256) NOT NULL,
    description TEXT,
    policy_type VARCHAR(64) NOT NULL,
    status VARCHAR(32) DEFAULT 'DRAFT',
    owner_team VARCHAR(128),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE governance_policy_version (
    policy_id VARCHAR(128) NOT NULL REFERENCES governance_policy(policy_id),
    policy_version VARCHAR(64) NOT NULL,
    action VARCHAR(32) NOT NULL,
        -- ALLOW, WARN, REDACT, REQUIRE_APPROVAL, BLOCK
    rule_definition JSONB NOT NULL,
    published_by VARCHAR(256),
    published_at TIMESTAMPTZ,
    content_hash VARCHAR(128),
    PRIMARY KEY (policy_id, policy_version)
);

-- =============================================================================
-- 11. GOVERNANCE DECISIONS
-- =============================================================================
CREATE TABLE governance_decision (
    decision_id BIGSERIAL PRIMARY KEY,
    decision_time TIMESTAMPTZ NOT NULL,
    trace_id VARCHAR(64),
    span_id VARCHAR(64),
    execution_id VARCHAR(128),
    application_id VARCHAR(128),
    agent_id VARCHAR(128),
    user_id VARCHAR(256),
    session_id VARCHAR(256),
    policy_id VARCHAR(128) NOT NULL,
    policy_version VARCHAR(64) NOT NULL,
    decision VARCHAR(32) NOT NULL,
    matched_rule VARCHAR(256),
    evidence JSONB,
    explanation TEXT
);

CREATE INDEX idx_governance_decision_time ON governance_decision(decision_time DESC);
CREATE INDEX idx_governance_decision_trace ON governance_decision(trace_id);
CREATE INDEX idx_governance_decision_policy ON governance_decision(policy_id, policy_version);
CREATE INDEX idx_governance_decision_agent ON governance_decision(agent_id);

-- =============================================================================
-- 12. ADMINISTRATIVE AUDIT EVENTS
-- =============================================================================
CREATE TABLE administrative_audit_event (
    audit_event_id BIGSERIAL PRIMARY KEY,
    event_time TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    actor_user_id VARCHAR(256) NOT NULL,
    action VARCHAR(128) NOT NULL,
    resource_type VARCHAR(128) NOT NULL,
    resource_id VARCHAR(256) NOT NULL,
    prior_value JSONB,
    new_value JSONB,
    result VARCHAR(32) NOT NULL,
    source_ip VARCHAR(64),
    correlation_id VARCHAR(128),
    previous_hash VARCHAR(128),
    event_hash VARCHAR(128) NOT NULL
);

CREATE INDEX idx_admin_audit_time ON administrative_audit_event(event_time DESC);
CREATE INDEX idx_admin_audit_actor ON administrative_audit_event(actor_user_id);
CREATE INDEX idx_admin_audit_resource ON administrative_audit_event(resource_type, resource_id);

-- =============================================================================
-- 13. OPTIMIZATION RECOMMENDATIONS
-- =============================================================================
CREATE TABLE optimization_recommendation (
    recommendation_id BIGSERIAL PRIMARY KEY,
    generated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    application_id VARCHAR(128),
    agent_id VARCHAR(128),
    model_name VARCHAR(256),
    recommendation_type VARCHAR(64) NOT NULL,
    issue_summary TEXT NOT NULL,
    evidence JSONB NOT NULL,
    recommendation TEXT NOT NULL,
    estimated_monthly_savings_usd NUMERIC(18,2),
    estimated_tokens_saved BIGINT,
    confidence VARCHAR(32),
    analysis_start_time TIMESTAMPTZ,
    analysis_end_time TIMESTAMPTZ,
    execution_sample_count BIGINT,
    status VARCHAR(32) DEFAULT 'OPEN',
        -- OPEN, ACCEPTED, DISMISSED, IN_PROGRESS, VALIDATED
    disposition_notes TEXT,
    validated_savings_usd NUMERIC(18,2)
);

CREATE INDEX idx_optimization_agent ON optimization_recommendation(agent_id);
CREATE INDEX idx_optimization_status ON optimization_recommendation(status);
CREATE INDEX idx_optimization_savings ON optimization_recommendation(estimated_monthly_savings_usd DESC);
```

---

# **Appendix B: Suggested Demonstration Sequence**

To maximize RFP scoring, the demonstration should follow one connected story rather than seven disconnected screens:

1. Open the Executive Dashboard and identify one degraded agent and a cost increase.
2. Open Agent Inventory and locate `DriveThruOrderAgent`, its owner, purpose, instances, models, tools, and systems.
3. Open a failed execution from the agent details page.
4. Use the trace, payload, logs, and AI SRE summary to identify the root cause and impacted systems.
5. Switch to Latency Analytics and show that the same tool is also the primary P99 bottleneck.
6. Open Cost & Token Economics and show the application, agent, model, and owner contributing to the increase.
7. Open Token Optimization and show an evidence-backed model-right-sizing or prompt-caching recommendation.
8. Open Governance & Audit, review the execution, apply or demonstrate a policy, and export evidence.
9. Return to the Executive Dashboard to show how the issue appears in the consolidated ecosystem-health view.
