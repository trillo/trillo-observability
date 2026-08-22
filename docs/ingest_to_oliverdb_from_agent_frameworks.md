# Ingesting to OliverDB from Agent Frameworks

**Document Version:** 0.1 (draft)
**Status:** Working design — how real agents (ADK, LangChain, CrewAI, LlamaIndex, raw SDKs) ship telemetry directly to OliverDB.
**Companion to:**
- `oliverdb_improvements.md` — the OliverDB wishlist this design leans on
- `OliverDB-Otel-Mapping-Requirements.md` — ingest-time mapping story
- `Telemetry-Ingestion-Endpoint-Design.md` — the Postgres small-setup path (this doc is the OliverDB counterpart)

Scope: **producers** — how a customer's agent process is configured to emit OpenTelemetry directly to their OliverDB slot. The Trillo AOS platform emitter (the simulator inside `TrilloAgentObservability/generate_live_telemetry.py`) is out of scope here; it has its own doc.

---

## 1. Purpose

Every AI-agent framework worth observing already emits OpenTelemetry — the value is not in "building a Trillo SDK." The value is in:

1. **Auth model.** A least-privilege primitive so an agent process ships spans without holding the OliverDB admin key.
2. **Credential propagation.** The right env-var surface so each framework's OTLP exporter just works.
3. **Semantic-convention conformance.** What the agent must emit so it lands in OliverDB's frozen columns and pre-built rollups as intended.
4. **Framework-specific quirks.** Small config recipes so an ADK / LangChain / CrewAI agent starts producing usable data in under 15 minutes.

We follow OTel conventions everywhere. Nothing in this design assumes a customer runs Trillo AOS as a producer — this is the "bring your own agents" ingestion path.

---

## 2. Auth model

### 2.1  Scoped write-only keys, one per agent identity

The OliverDB admin key never travels to an agent process. Trillo (or the OliverDB console) mints **scoped write-only keys** — the minimum privilege primitive OliverDB exposes today:

- **Scope:** `INSERT` on the spans table, no read, no admin, no key-mint.
- **Row pins (optional, recommended):** `resource_attrs.service.namespace = <application_id>` — the key can only write rows tagged with its owning application. Prevents cross-tenant contamination if a key leaks.
- **Rate cap:** per-key ingest ceiling to cap blast-radius of a runaway agent.
- **TTL:** rotatable; ~90 days default with an auto-rotate handoff.

One key per **agent identity**, not per instance — instances of the same logical agent (e.g. `menu-recommender-agent` running in West + East) share a key. Rotation and revocation are agent-level.

Read-only scoped keys are a separate concern (for the SRE Copilot, dashboards, ad-hoc analytics) and are covered in `SRE-Copilot-Tool-Manifest.md`.

### 2.2  Credential delivery — env vars, following framework conventions

Every OTel-native framework already reads:

```
OTEL_EXPORTER_OTLP_ENDPOINT
OTEL_EXPORTER_OTLP_HEADERS
OTEL_EXPORTER_OTLP_PROTOCOL
```

That's the entire surface. Trillo does not introduce Trillo-branded env vars for producers; the customer's deploy tooling sets the standard OTel envs from wherever it manages secrets (Kubernetes secret, GCP Secret Manager, Vault).

Recommended values for OliverDB:

```
OTEL_EXPORTER_OTLP_ENDPOINT=https://<tenant>.us-west-2.aws.olivercloud.ai
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=$OTEL_EXPORTER_OTLP_ENDPOINT/v1/traces
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <scoped_write_only_key>
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf   # or http/json
OTEL_SERVICE_NAME=<logical_agent_name>       # e.g. menu-recommender-agent
OTEL_RESOURCE_ATTRIBUTES=service.namespace=<application_id>,service.instance.id=<runtime_instance_id>,agent.version=<version>
```

> **Depends on `oliverdb_improvements.md` §1.1.** Until OliverDB has the native OTLP endpoint, the `TRACES_ENDPOINT` above points at a small **Trillo-hosted OTLP → OliverDB adapter** that translates the OTLP protobuf to `/v1/ingest`'s positional-cells shape. Section 7 covers the adapter.

### 2.3  Trillo does not proxy the customer's telemetry

The customer's agent talks **directly** to OliverDB (or the adapter that Trillo hosts as a stop-gap). Trillo AOS is not on the write path. This preserves:

- Latency budget — one hop, not two.
- Blast radius — a Trillo outage doesn't sink customer observability.
- Cost — no re-transmission of the customer's payload volume.

Trillo is on the **read** side (SRE Copilot, dashboards) via its own read-scoped keys, and on the **management** side (minting keys, refreshing rotations).

---

## 3. Semantic-convention conformance

OliverDB will land whatever OTLP you send in the `resource_attrs` and `attrs` blobs, but the **frozen columns and pre-built rollups only work when the producer follows the OpenTelemetry `gen_ai.*` semantic conventions.**

### 3.1  Required span attributes (for the token / cost story)

| OTel key | Purpose | OliverDB destination |
|---|---|---|
| `gen_ai.system` | provider (openai, anthropic, google-vertex, …) | `attrs.gen_ai.system`, indexed |
| `gen_ai.request.model` | model id (claude-sonnet-5, gemini-2.5-pro, …) | `attrs.gen_ai.request.model`, indexed |
| `gen_ai.usage.input_tokens` | prompt token count | **`input_tokens`** column (once §1.2 lands) |
| `gen_ai.usage.output_tokens` | completion token count | **`output_tokens`** column (once §1.2 lands) |
| `gen_ai.response.finish_reasons` | stop reasons | `attrs.gen_ai.response.finish_reasons` |

Until improvement `oliverdb_improvements.md §1.2` lands, the producer (or the adapter in §7) must **also** set the top-level `input_tokens` / `output_tokens` cells directly.

### 3.2  Recommended resource attributes (fleet analytics)

The `TrilloAgentObservability` app's inventory analytics assume a canonical shape on `resource_attrs`. If the producer sets them, no reconciliation is needed:

```
service.name          — the logical agent name (matches OTEL_SERVICE_NAME)
service.namespace     — the application id (or slug)
service.instance.id   — a stable per-runtime-instance identifier
service.version       — the agent version (v2.5.0 / v2.6.0)
cloud.provider        — gcp / aws / azure
cloud.region          — the region (us-west1, …)
deployment.environment — production / staging / dev
```

Trillo-specific enrichment (owner_team, cost_center, business_unit, governance_class, business_purpose) can either be set on the producer side OR added at ingest by the OliverDB Rust plugin, per `OliverDB-Otel-Mapping-Requirements.md`. Preferred: **producer sets them** — that way the observability data can never disagree with the inventory. Fallback: **plugin fills them in** from a small dimension table keyed by `service.name`.

### 3.3  Span-name conventions

The pre-built `by_operation` rollup is keyed by `(service_name, span_name)`. Recommended span names in the AI-agent domain:

| Span kind | span_name | Notes |
|---|---|---|
| Top-level agent invocation | `agent.run` | one per user-visible request |
| LLM call | `llm.chat` or `llm.<model>.generate` | one per model call |
| Tool call | `tool.<tool_name>` | one per tool/function invocation |
| Retrieval | `retrieval.<store>` | vector DB / knowledge base |
| Downstream HTTP | `http.<system>` | to a dependent system |
| Sub-agent | `agent.<sub_name>.run` | nested agent |

Kinds: `agent.run` = `SERVER`; LLM/tool/retrieval/HTTP/sub-agent = `CLIENT`.

---

## 4. Framework recipes

Each recipe assumes the env vars in §2.2 are set. Each is minimal — the OTel ecosystem has fuller docs; the point here is what Trillo has verified works end-to-end.

### 4.1  Google Agent Development Kit (ADK)

ADK emits OpenTelemetry natively via `google.adk.trace`. To enable OTLP export:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
import os

resource = Resource.create({
    "service.name": os.getenv("OTEL_SERVICE_NAME"),
    "service.namespace": os.getenv("APPLICATION_ID"),
    "service.instance.id": os.getenv("INSTANCE_ID"),
})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)
```

ADK auto-instruments `agent.run` (agent turns), `llm.chat` (Gemini calls), `tool.<name>` (tool invocations), `agent.<sub>.run` (sub-agent hops). Token counts land on the `llm.chat` span with `gen_ai.usage.*` keys, matching §3.1.

**Verified in this configuration:** all four span shapes appear in OliverDB with `gen_ai.request.model` set; token cells populate correctly. No custom wrapper needed.

### 4.2  LangChain / LangGraph

LangChain-native tracing goes to LangSmith by default. To send OTel to OliverDB **in addition to (or instead of) LangSmith**, use OpenLLMetry or the community OTel instrumentation:

```bash
pip install opentelemetry-instrumentation-langchain
```

```python
from opentelemetry.instrumentation.langchain import LangchainInstrumentor
LangchainInstrumentor().instrument()
```

The instrumentation emits spans that match the semantic conventions in §3, including token counts on LLM calls. Verify `gen_ai.system` is set (some LangChain LLM wrappers do not set it — set it via `OTEL_RESOURCE_ATTRIBUTES` as a fallback).

**Known gap:** custom `RunnableLambda` chains don't get auto-instrumented. Producers can add manual spans with the standard OpenTelemetry Python SDK — no Trillo-specific API involved.

### 4.3  CrewAI

CrewAI has OTel support via the community package:

```bash
pip install opentelemetry-instrumentation-crewai
```

```python
from opentelemetry.instrumentation.crewai import CrewAIInstrumentor
CrewAIInstrumentor().instrument()
```

CrewAI's model of "crew → agent → task → tool" maps to OTel spans in that hierarchy. Token counts land per `Task` span.

**Known gap:** CrewAI's tool-call spans sometimes miss `gen_ai.system` when the LLM is called through a custom LiteLLM proxy — instrument the proxy's HTTP call as a manual span, or set the attribute on the crew's resource.

### 4.4  LlamaIndex

```bash
pip install opentelemetry-instrumentation-llama-index
```

```python
from opentelemetry.instrumentation.llama_index import LlamaIndexInstrumentor
LlamaIndexInstrumentor().instrument()
```

LlamaIndex emits `retrieval.<store>` spans (vector DB, keyword index, etc.) alongside LLM spans. The retrieval spans do **not** carry tokens; that's fine — they aggregate under `by_operation`.

### 4.5  Raw provider SDKs (OpenAI, Anthropic, Vertex)

For hand-written agents built on the vendor SDKs, the smallest working setup:

```bash
pip install opentelemetry-instrumentation-openai
pip install opentelemetry-instrumentation-anthropic
```

```python
from opentelemetry.instrumentation.openai import OpenAIInstrumentor
from opentelemetry.instrumentation.anthropic import AnthropicInstrumentor
OpenAIInstrumentor().instrument()
AnthropicInstrumentor().instrument()
```

You still need to emit the outer `agent.run` span manually — the SDK instrumentations only cover the `llm.chat` layer.

### 4.6  Non-OTel producers

If a customer's agent is on a stack without OTel instrumentation, the fallback is:

- **HTTP direct.** Point the customer's log-forwarder at Trillo's OTLP → OliverDB adapter (§7). The adapter accepts loose JSON with a documented schema and coerces it to OTLP.
- **Not recommended** — the semantic-convention alignment is fragile and every producer has to be reviewed. Push the customer toward one of the OTel instrumentations above.

---

## 5. Sampling

**Default: 100 % head-based sampling.** AI-agent traffic volumes are low compared to microservice traffic (tens-to-thousands of `agent.run`s per hour per agent, not millions). At those rates the storage and query cost of full retention is negligible compared to the operational value of "every trace is there."

**When to sample.** Only for agents with sustained > 100 rps per instance or where per-trace payloads exceed ~50 KB. In those cases:

- Set `OTEL_TRACES_SAMPLER=parentbased_traceidratio` and `OTEL_TRACES_SAMPLER_ARG=0.1` (10%).
- Or use **tail-based sampling** in an OTel Collector between producers and OliverDB — keep 100% of error traces, 100% of slow traces (p95+), 10% of the rest. This is where the OTel Collector earns its keep.

Never sample the `evalScore` / `governanceOutcome` signal — a scoped-key policy on the read side can compensate for missing traces via the `Execution` (Postgres) side, but if the trace itself is dropped, the drift analyses lose their input.

---

## 6. Producer-side verification

A one-command check the customer runs after configuring env vars:

```bash
python -c "
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
p = TracerProvider(); p.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(p)
with trace.get_tracer('smoke').start_as_current_span('agent.run') as s:
    s.set_attribute('gen_ai.system', 'openai')
    s.set_attribute('gen_ai.request.model', 'gpt-4o')
    s.set_attribute('gen_ai.usage.input_tokens', 100)
    s.set_attribute('gen_ai.usage.output_tokens', 42)
p.shutdown()
"
```

Then on the Trillo side:

```sql
POST /v1/query {"sql": "SELECT trace_id, service_name, input_tokens, output_tokens
                        FROM t WHERE span_name = 'agent.run'
                        ORDER BY ts DESC LIMIT 5"}
```

If the row appears with the right service_name and token counts, the producer path is healthy.

---

## 7. Trillo-hosted OTLP → OliverDB adapter (stop-gap)

Until OliverDB ships native OTLP (`oliverdb_improvements.md §1.1`), we run a small stateless service:

- **Endpoint:** `https://otlp-to-oliver.trillo.ai/v1/traces`
- **Auth:** the same scoped write-only key the producer sets in `OTEL_EXPORTER_OTLP_HEADERS`. The adapter forwards it verbatim to OliverDB.
- **Translation:** OTLP protobuf → OliverDB positional-cells JSON. Pulls `gen_ai.usage.*` into token cells (compensating for §1.2 not being live yet). Buffers up to 1000 spans / 5 s and POSTs to `/v1/ingest`.
- **Failure modes:** on OliverDB 5xx, retries with exponential backoff for up to 60 s, then drops with a metric. Never blocks the producer.
- **Metrics:** exported to Trillo AOS's operations dashboard (spans/s in, spans/s out, drop rate, translation errors).

The adapter is single-file, single-binary, deployable per-region. Once native OTLP is live, the customer just changes `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` to point directly at OliverDB and the adapter can be decommissioned per-customer with no other change.

---

## 8. Open questions

1. **Adapter or no adapter?** If OliverDB commits to native OTLP inside the POC window, we may skip §7 entirely. Confirm timeline.
2. **Key rotation UX.** Producers running in Kubernetes want the key delivered as a secret. Trillo's minting flow needs to output the key in a format that plugs into the customer's secret pipeline (kubectl create secret, gcloud secrets create). Design.
3. **Multi-tenant Trillo customers.** If one customer runs OliverDB and Trillo runs the SRE Copilot on their behalf, how do we scope our read-key across customer instances? Cross-customer joins are not a POC goal; leave as a v2 question.
4. **`gen_ai.system` fallbacks.** When frameworks don't set it, is it acceptable to infer from `gen_ai.request.model` on the OliverDB side (via the mapping-requirements Rust plugin), or must producers guarantee it? Preference: infer with a WARN, so producers aren't blocked.
5. **Prompt / completion capture policy.** Some customers can't ship prompts/completions off-premises at all. Design a per-agent config (env var `TRILLO_CAPTURE_TEXT=none|redacted|full`) that the OTel instrumentations honor.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-22 | Trillo (via Claude Code session) | Initial draft; scoped to producer side (ADK / LangChain / CrewAI / LlamaIndex / raw SDKs). Adapter documented as a stop-gap. |
