# Ingest-time Attribute Normalization — Note for the OliverDB Team

**Document Version:** 0.1
**Audience:** OliverDB engineering team.
**From:** Trillo platform team.
**Related:**
- `oliverdb_improvements.md` §1.2 (semconv pull-outs on ingest) — the P0 item this note fleshes out.
- `oliverdb_onboarding.md` + `oliverdb_onboarding_admin.md` — the wire contracts we're building against.
- `oliverdb_entities/OtlpTelemetry.json` — the unified telemetry schema on our side that assumes this feature.

## 1. What we're asking for

An **ingest-time attribute promotion primitive** in OliverDB: a mechanism (either a Rust plugin in the ingest path, or a declarative rule set in the tenant config, or both) that reads a configured list of OTel semantic-convention attribute keys from every incoming record and copies each value into a dedicated first-class typed column, before the record lands on disk.

Concretely, when a tenant has this rule (declarative example):

```yaml
promote_semconv:
  # OTel Resource attributes
  - key: resource_attrs.service.name         → column: service_name         type: keyword
  - key: resource_attrs.service.namespace    → column: service_namespace    type: keyword
  - key: resource_attrs.service.instance.id  → column: service_instance_id  type: keyword
  - key: resource_attrs.service.version      → column: service_version      type: keyword
  - key: resource_attrs.deployment.environment → column: deployment_environment type: keyword
  - key: resource_attrs.cloud.provider       → column: cloud_provider       type: keyword
  - key: resource_attrs.cloud.region         → column: cloud_region         type: keyword
  - key: resource_attrs.k8s.namespace.name   → column: k8s_namespace        type: keyword
  - key: resource_attrs.k8s.pod.name         → column: k8s_pod_name         type: keyword
  - key: resource_attrs.k8s.deployment.name  → column: k8s_deployment_name  type: keyword
  - key: resource_attrs.host.name            → column: host_name            type: keyword

  # OTel GenAI semconv (specific to AI observability workloads)
  - key: attrs.gen_ai.system                 → column: genai_system         type: keyword
  - key: attrs.gen_ai.request.model          → column: genai_request_model  type: keyword
  - key: attrs.gen_ai.response.model         → column: genai_response_model type: keyword
  - key: attrs.gen_ai.response.finish_reasons[0] → column: genai_finish_reason type: keyword
```

…a producer POSTing standard OTLP with those keys inside the `attrs` / `resource_attrs` blobs would land in OliverDB with the values ALSO present in their dedicated typed columns. Same record, two access paths — the JSON blob preserves everything, the promoted columns speed the hot filter/aggregate paths.

## 2. Why this matters — the performance case

Arrow's columnar layout gives typed columns three advantages over embedded JSON that compound with volume:

1. **Vectorized filters.** A `WHERE service_name = 'web-frontend'` on a Dictionary-encoded string column runs at ~10 GB/s per core with SIMD. The equivalent `WHERE attrs.service.name = 'web-frontend'` on a JSON blob requires per-row parse-and-extract — 5-20× slower even with your label-sidecar index accelerating equality.

2. **Row-group pruning to storage.** Parquet-style row-group min/max stats let the file reader skip whole groups whose extremes prove no match. A `WHERE deployment_environment = 'prod'` on a 100M-row table with 1M-row groups can prune to ~10% of the data *before the CPU sees a byte*. JSON-embedded values can't participate in row-group stats.

3. **Aggregate speed.** `sum(int_value)` on a typed Int column runs contiguous 8-byte reads with SIMD adds. `sum(attrs.gen_ai.usage.input_tokens)` on a JSON blob has to JSON-parse per row before summation. Your label sidecar accelerates equality *filters* but doesn't speed *aggregation over values* the same way.

The label sidecar you've built is genuinely helpful and closes the gap on equality filters — but for range predicates, aggregate operations, and cross-column joins the plugin surface, typed columns win by a large margin. At scale (400K events/sec target, ~35B rows/day), this is the difference between snappy dashboards and painful ones.

## 3. The specific set — 15 attrs

The list above is the minimum viable set to promote, chosen because every one of them is filtered or grouped-by in real Trillo AI observability queries:

- **Service identity** (4): every dashboard filters by service; scoped-key row-pins already lean on `service.namespace`.
- **Deployment/Cloud** (3): env is a top-line filter across every UI panel; region drives per-geo rollups.
- **Kubernetes** (3): pod-level troubleshooting is a common drill-down; deployment name groups by rolling-update generation.
- **Host** (1): non-K8s deployments still exist.
- **GenAI** (4): "spend by model", "requests by provider", and "failed responses by finish reason" are three of the top-ten queries against the AI-agent workload.

Everything else (domain attrs, provider-tunable knobs, user-defined labels) stays in the JSON blob. Roughly 15 promoted columns × 8 bytes typical / row = ~120 bytes/row of extra typed data. On a Dictionary-encoded string layout with heavy repetition (service names, environments, model ids), the effective storage cost is dominated by dictionary entries, not the row-level references — well under 10% overhead in practice.

## 4. Suggested plugin implementation shape

The OliverDB onboarding admin doc mentions Rust plugins that can run in the ingest path. Our ask fits that model cleanly:

**Per-record transform:**
```
fn on_record(record: &mut ArrowRecordBatch, config: &PromoteConfig) {
    for rule in &config.rules {
        let value = extract_key(record, rule.source_key);   // walk JSON path
        if let Some(v) = value {
            record.set_column(rule.target_column, v);
        }
    }
}
```

Ideally batched (per RecordBatch, not per record) so the JSON parse can be amortized. If OliverDB's ingest already builds an intermediate representation of `attrs`/`resource_attrs` (for the label sidecar), that same parse can feed both the sidecar and the column promotion — no additional per-record work.

**Configuration surface:**
- Per-tenant `promote_semconv` list at the schema definition level, exposed on the same admin surface that mints keys and configures roles.
- Additive-only during tenant lifetime; changing an existing rule's target column would require a rewrite that we'd rather avoid.
- Well-known "OTel semconv v1.x" presets so tenants don't have to type the full list — subscribe to the preset and updates land at OliverDB-team pace.

**Data-plane behavior:**
- Presence of promoted columns doesn't change the JSON blob — both surfaces contain the value. Consumers can use either; queries on typed columns win the performance race.
- On a producer that doesn't emit a promoted key, the column is null (existing OliverDB nullability rules apply).
- Type mismatches (producer emits `gen_ai.request.model` as a number) → land the record with the column null and a per-record annotation for observability; do NOT reject the record.

## 5. Migration path for producers

We can start writing the promoted columns from the producer side *today* (dual-write into both JSON and typed columns) and switch to relying on the plugin once it ships:

- **Phase 1 (now):** Trillo producers set both `attrs.gen_ai.request.model` in the JSON AND `genai_request_model` in the typed column. Storage cost accepted for the interim.
- **Phase 2 (plugin ships):** Producers drop the typed writes; plugin populates from JSON. No consumer change (the schema is unchanged).

That means the ask on you is *not* blocking us — we can ship dual-write immediately. What the plugin unlocks is (a) producer simplicity (emit standard OTLP, get promoted columns for free) and (b) coordination-free evolution of the promotion set (change the config, all producers benefit).

## 6. Adjacent asks (context, not new)

Two related items already in `oliverdb_improvements.md` that this plugin feature composes with:

- **§1.2 semantic-convention pull-outs** — `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` into the frozen `input_tokens` / `output_tokens` int columns. Same mechanism as the promotion described here, just applied to numeric OTel semconv keys. If both land as one plugin, ~20 promoted columns total.
- **§5.2 sweep-friendly rollups** — declarative per-cadence materialization keyed on cube/sketch primitives. Composes with column promotion: if `genai_request_model` is a typed column, a `by_model` cube can pre-compute per-model aggregates that scheduled sweepers read in O(1).

## 7. Why we're asking now, not later

We have Slice AB→F' shipped on the Trillo side (integration, dispatcher rewrites, developer docs, unified `OtlpTelemetry` schema). The volume-scaling gap between "small demo" and "sustained production dashboard queries" is exactly where typed columns pay off. Getting the plugin design locked before we onboard the second real customer means we avoid dual-write coordination pain and downstream schema-drift risk.

If the plugin surface can land in the next 4-6 weeks, we'd sequence:
1. Trillo starts dual-write on the 15 promoted keys immediately (self-contained work on our side).
2. OliverDB ships the plugin with a preset for OTel semconv v1.x + a GenAI preset.
3. We flip Trillo producers to plugin-mode; dual-write disappears.

If it's longer than that, we're OK — the dual-write path stays working indefinitely; we just don't get the ergonomic wins of "emit standard OTLP, done."

## 8. Questions we'd appreciate answers to

1. Does the ingest-path plugin surface exist today (in any form) or is it a design item? If design, what's the timeline you'd hazard?
2. Is the promotion mechanism you'd build a plugin (Rust code) or declarative rules (config)? Preference on our side: **declarative rules**, so we can iterate on the promotion set without a plugin release.
3. Do the promoted columns need to be declared at schema-creation time, or can they be added incrementally as we discover new hot filters? (Answer here affects the "additive-only during tenant lifetime" constraint above.)
4. Any storage-cost visibility we can get per-column (dictionary size, RLE ratio, distinct-value count)? Would help us tune the promoted list over time.
5. Is there a plugin/rule-config test harness we could hit before deploying rules to prod tenants?

Happy to jump on a call to walk through the 15-attr list, any of the technical trade-offs, or the plugin design shape.

— Trillo platform team
