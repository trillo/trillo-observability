# OliverDB — trillo onboarding (AI observability)

Your dedicated OliverDB instance is live, shaped for AI-agent observability over OTel traces: one row per span, token accounting as first-class columns. Every example below was run against your instance before this doc was sent.

| | |
|---|---|
| Endpoint | `https://trillo.us-west-2.aws.olivercloud.ai` |
| Capacity | 2 cells × 2 cores — 4 vCPU · 50 GB storage per cell |
| Retention | 3-day hot window on SSD, 180-day total in object storage |
| API key | Sent separately (shown once at mint; we do not retain it). `export OLIVER_KEY=<key>` |

## Point your AI at it first

Your instance publishes a machine-readable capability manifest — the live schema with effective indexes, the SQL dialect, your rollups, and runnable examples, scoped to whatever key fetches it:

```bash
curl https://trillo.us-west-2.aws.olivercloud.ai/onboard -H "Authorization: Bearer $OLIVER_KEY"
```

One fetch replaces a schema walkthrough — hand that URL + a key to your agent and it knows exactly what it can do (and, with a scoped key, only what that key allows). MCP-speaking agents (Claude Desktop / Claude Code) can instead use `oliverdb-tenant-mcp` with `OLIVER_ENDPOINT` + `OLIVER_API_KEY` — four read-only tools: `oliver_sql`, `oliver_schema`, `oliver_stats`, `oliver_onboard`.

## Your table — one row per span

| Column | Type | Notes |
|---|---|---|
| `trace_id`, `span_id`, `parent_span_id` | Id | exact needle lookup (sparse + bloom) |
| `service_name`, `span_name`, `span_kind`, `status_code` | Keyword | group by · filter · bitmap index |
| `duration_us` | Int | latency measure |
| `input_tokens`, `output_tokens` | Int | **token accounting — summable** (attrs keys can be filtered/grouped but never summed, so these are real columns) |
| `resource_attrs`, `attrs` | Attrs | the OTel long-tail — any key filterable as `attrs.<key>`, no schema change ever (e.g. `gen_ai.request.model`, `stop_reason`, `session.id`) |

Implicit `ts` in **epoch milliseconds** (span start). Columns are frozen; rollups evolve any time. Pre-built rollups: cube `by_operation` (service × span_name → duration, tokens) and sketch `latency` (p50/p95/p99 duration by service) — dashboards are fast from the first row.

## Sending spans

`POST /v1/ingest`, JSON array, positional `cells` in the column order above. `{"Str":…}` for Id/Keyword/Attrs (attrs = a JSON object as a string), `{"I64":…}` for Int:

```bash
curl "https://trillo.us-west-2.aws.olivercloud.ai/v1/ingest" \
  -H "Authorization: Bearer $OLIVER_KEY" -H 'content-type: application/json' \
  -d '[{"ts":1787259115586,"cells":[
    {"Str":"tr-4fa1"},{"Str":"sp-9c22"},{"Str":"sp-root1"},
    {"Str":"llm-gateway"},{"Str":"llm.chat"},{"Str":"CLIENT"},{"Str":"OK"},
    {"I64":307319},{"I64":4210},{"I64":512},
    {"Str":"{\"deployment.environment\":\"prod\"}"},
    {"Str":"{\"gen_ai.request.model\":\"claude-sonnet-5\",\"stop_reason\":\"end_turn\"}"}]}]'
```

Ack semantics: default `202 {"buffered": n}` (flushes within 5 s; check drain via `GET /v1/status` — `buffered` back to 0, `flushed_rows` up). Must-land writes: append `?sync=1` for the engine's own `200 {"ingested": n}`. Batch hundreds-to-thousands of spans per request (our 772-span smoke: 0.3 s).

## Querying (`POST /v1/query`, `{"sql":"…"}`, table `t`) — all verified live

```sql
-- Span volume + mean latency by service (cube-served, sub-ms)
SELECT service_name, count(*), avg(duration_us) FROM t GROUP BY service_name

-- p95 latency by service (sketch-served)
SELECT service_name, percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_us)
FROM t GROUP BY service_name

-- Token spend by model — group by an attrs key, sum real columns
SELECT attrs.gen_ai.request.model, sum(input_tokens), sum(output_tokens), count(*)
FROM t GROUP BY attrs.gen_ai.request.model

-- Error rate by operation
SELECT span_name, count(*) FILTER (WHERE status_code = 'ERROR'), count(*)
FROM t GROUP BY span_name ORDER BY 3 DESC

-- Reconstruct one trace (index-served needle)
SELECT ts, span_name, service_name, status_code, duration_us
FROM t WHERE trace_id = 'tr-…' ORDER BY ts

-- Why did LLM calls stop?
SELECT attrs.stop_reason, count(*) FROM t WHERE span_name = 'llm.chat'
GROUP BY attrs.stop_reason

-- Token burn per 15 minutes
SELECT time_bucket('15m', ts), sum(input_tokens), sum(output_tokens)
FROM t GROUP BY time_bucket('15m', ts) ORDER BY 1

-- Slowest agent sessions (exact top-N, single pass)
SELECT attrs.session.id, max(duration_us) FROM t
WHERE span_name = 'agent.run' GROUP BY attrs.session.id ORDER BY 2 DESC LIMIT 5
```

Dialect notes: `text_match()` for FTS (if you add a Text column — none in this schema), `LIKE 'prefix%'` index-accelerated, datetime literals accepted in `WHERE ts` bounds, `time_bucket('5m'|'1h'|…)`. No JOINs/subqueries/CTEs. Many questions over one window → `POST /v1/query_batch` answers a whole dashboard from one shared scan, including anomaly screening across series.

## Scoped keys for your agents

Your admin key mints least-privilege keys — each carries a policy the engine enforces on every query (row pins, column redaction, window caps; the policy *rewrites* out-of-scope queries rather than trusting the caller). The authoring guide is itself agent-ready:

```bash
curl https://trillo.us-west-2.aws.olivercloud.ai/onboard/admin -H "Authorization: Bearer $OLIVER_ADMIN_KEY"
```

Typical split: a write-only key for your OTel exporter, read-only scoped keys for agents and dashboards. Network allow-listing (IP ranges) is available on the console's Roles & keys page.

## Questions

Reply to this thread — or ask your agent after it fetches `/onboard`; that manifest is the same document we'd walk you through.
