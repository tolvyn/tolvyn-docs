# AI Cost Index

The TOLVYN AI Cost Index is a public dataset of average AI request costs by provider, model, and date — sourced from anonymized aggregate data contributed by TOLVYN tenants who opted in. It is the only data source that reflects what AI actually costs in production, at the level of individual model versions, broken out by p50 and p95.

Free to query. No authentication. Apache 2.0 data license.

---

## What it is

When you call OpenAI's API, the published price is "$2.50 per million input tokens." That tells you the rate. It does not tell you:

- *What does a typical chat request actually cost?* — depends on prompt size, output length, caching, etc.
- *How does that compare across providers?*
- *Is `gpt-4o` getting more expensive in real-world usage even at a stable per-MTok rate?*

The Cost Index answers those questions with production data. Every night, TOLVYN computes the average, p50, p95, total request count, and tenant count for every (provider, model, date) bucket — but only if at least 3 tenants contributed to the bucket. Single-tenant data points are dropped to preserve anonymity.

---

## How data is collected

A background goroutine starts at server boot (`internal/aggregate/collector.go:13-36`):

```go
now := time.Now().UTC()
next := time.Date(now.Year(), now.Month(), now.Day(), 3, 0, 0, 0, time.UTC)
if !next.After(now) {
    next = next.Add(24 * time.Hour)
}
log.Printf("aggregate: next collection scheduled for %s", next.Format(time.RFC3339))
time.Sleep(time.Until(next))

yesterday := time.Now().UTC().AddDate(0, 0, -1)
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Minute)
n, err := RunCollection(ctx, db, yesterday)
```

| Property | Value |
|---|---|
| Schedule | **Daily at 03:00 UTC** |
| Period | The full UTC day immediately preceding the run (yesterday) |
| Timeout | 30 minutes |
| Output | Rows inserted/updated in `aggregate_metadata` |

The collection query aggregates the `requests` table grouped by `(provider, model_id, model_family)`, filtered to:

- Tenants with `status = 'active'`
- Tenants where `(settings->>'data_share_enabled') IS DISTINCT FROM 'false'` (i.e. not explicitly opted out)
- Buckets with `COUNT(DISTINCT tenant_id) >= 3` (k-anonymity floor)

---

## K-anonymity floor

From `collector.go:64`:

```sql
HAVING COUNT(DISTINCT r.tenant_id) >= 3
```

Every published data point reflects **at least 3 contributing tenants**. Buckets that don't meet this threshold are silently dropped at collection time. They never reach `aggregate_metadata`, never appear on the public index, and cannot be re-derived from the public surface.

The threshold is also re-applied at query time on `GET /v1/public/cost-index` (`internal/api/client.go:307`):

```sql
WHERE period_date >= now() - ($1 || ' days')::interval
  AND tenant_count >= 3
```

Two-layer enforcement. If a row somehow ended up with `tenant_count < 3` (e.g. via a manual SQL fix), the public endpoint still excludes it.

---

## What is collected

`aggregate_metadata` schema (migration 019_aggregate_metadata.sql:6-21):

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `period_date` | DATE | UTC date this row aggregates |
| `provider` | VARCHAR(50) | `openai` / `anthropic` / `google` |
| `model_id` | VARCHAR(100) | Provider's exact model ID |
| `model_family` | VARCHAR(100) | Normalized family |
| `total_requests` | BIGINT | Count of requests in the bucket |
| `total_cost_usd` | NUMERIC(14,6) | Sum of cost across contributing tenants |
| `avg_cost_usd` | NUMERIC(14,8) | Mean cost per request |
| `p50_cost_usd` | NUMERIC(14,8) | Median request cost |
| `p95_cost_usd` | NUMERIC(14,8) | 95th percentile request cost |
| `total_tokens_input` | BIGINT | Sum of input tokens |
| `total_tokens_output` | BIGINT | Sum of output tokens |
| `tenant_count` | INT | How many distinct tenants contributed (always ≥ 3) |
| `collected_at` | TIMESTAMPTZ | When this row was last updated |

## What is NOT collected

From migration 019's leading comment:

> *Anonymized aggregate metadata for TOLVYN AI Cost Index. No tenant_id, no PII, no request content. Tenants can opt out via settings.data_share_enabled = false.*

The migration is explicit: **no RLS on this table because it has no tenant data**. Concretely:

| Never collected | Note |
|---|---|
| `tenant_id` | The column does not exist in the table |
| Tenant name, email, or any identifier | Not in the source query |
| `service_name`, `team_id`, `agent_name` | Stripped at the aggregation level |
| `X-Tolvyn-User`, `X-Tolvyn-End-Customer` | Stripped |
| Prompt content, response content | TOLVYN never stores these to begin with |
| Per-request `request_id` | Not preserved through aggregation |
| Cost per individual tenant | Only the sum across ≥ 3 tenants is retained |

You cannot recover a single tenant's spend from any combination of `aggregate_metadata` rows. The bucket grouping (`provider`, `model_id`, `model_family`) and the k-anonymity floor make tenant re-identification impossible.

---

## Opt-out

### Default: opted in

When a tenant signs up, `settings.data_share_enabled` is unset. The collector reads this as "not opted out" via:

```sql
(t.settings->>'data_share_enabled') IS DISTINCT FROM 'false'
```

`IS DISTINCT FROM 'false'` returns true for NULL and true for `'true'` — only explicit `'false'` excludes a tenant.

The account dashboard (`HandleGetAccount`, `client.go:70`) confirms this is the default:

```go
COALESCE((settings->>'data_share_enabled')::boolean, true)
```

The dashboard always shows `data_share_enabled: true` until you explicitly opt out.

### Opt out via dashboard

**Account → Settings → Data sharing** → toggle off.

### Opt out via API

```bash
curl -X PUT https://api.tolvyn.io/v1/account/settings/data-share \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}'
```

Response:

```json
{"data_share_enabled": false}
```

### Effect of opt-out

- Your requests are **immediately excluded** from the next nightly run.
- Existing `aggregate_metadata` rows are **not retroactively recomputed**. Historical data points that included you remain (they cannot be linked back to you anyway — see What is NOT collected above).
- You can re-enable at any time by setting `enabled: true`.

---

## The public Cost Index page

[`tolvyn.io/cost-index`](https://tolvyn.io/cost-index) (when deployed; until then the data is reachable directly via API).

The page shows:

- Cost per provider over time, with selectable model filters
- Top 10 most-used models by aggregate request count
- p50 vs p95 spread for each model (helps spot models where the long tail dominates cost)
- Period selector (7, 30, 90 days)

All data is reachable via the public API:

```bash
curl "https://api.tolvyn.io/v1/public/cost-index?provider=openai&days=30"
```

```json
{
  "data": [
    {
      "date": "2026-05-17",
      "provider": "openai",
      "model_id": "gpt-4o-2024-08-06",
      "model_family": "gpt-4o",
      "avg_cost_usd": "0.00842133",
      "p50_cost_usd": "0.00521000",
      "p95_cost_usd": "0.02410000",
      "total_requests": 184320,
      "tenant_count": 47
    }
  ]
}
```

No authentication. CORS open. `Access-Control-Allow-Origin: *`.

Query parameters:

| Param | Default | Description |
|---|---|---|
| `provider` | — | Filter by `openai`, `anthropic`, `google` |
| `model_id` | — | Specific model ID |
| `days` | `90` | Look-back window, 1–365 |

---

## Methodology

### Production data, not benchmarks

Every data point comes from real proxied traffic across multiple paying tenants. We do not synthesize prompts, run reference workloads, or normalize for any prompt distribution. The numbers reflect what real applications, with real prompts, with real token mixes, actually paid.

This has two implications:

- The data is **representative of how the industry actually uses these models**, including caching, retries, and the mix of short and long prompts.
- The data is **not directly comparable to provider quotes** — provider rates are per-MTok; this index is per-request. A model that gets used for one-shot translations will look cheaper than the same model used for 10K-token document summaries.

### Provider rate vs. effective cost

Use the per-MTok rates from each provider's pricing page when you're estimating budget for a new workload. Use the Cost Index when you want to know what AI actually costs at the request level for similar workloads.

### Why the floor is 3 tenants

A floor of 3 is the smallest number where averages cannot trivially reveal individual contributions. With 2 tenants in a bucket, the publication of the average + total + max would let either tenant infer the other's exact spend. With 3+, it's statistically infeasible without external information.

3 is a deliberate trade-off between utility (more data points published) and privacy (more contributors needed per bucket). Tightening to 10 would suppress nearly all niche-model data points. Loosening to 2 would weaken anonymity. 3 is the established minimum in differential privacy research for aggregate publishing.

### Update cadence

The index updates daily at 03:00 UTC for the previous UTC day. There is no rolling 24-hour window — each `period_date` is one full calendar day.

---

## See also

- [Account → Data Share Setting](../reference/api.md#account) — opt-out endpoint
- [API Reference: Public Cost Index](../reference/api.md#public)
- [The `tolvyn-models` repo](https://github.com/tolvyn/tolvyn-models) — open-source model metadata complementary to the Cost Index
