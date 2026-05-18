# Team Insights

Team Insights breaks AI usage down by individual team member so you can identify high performers, spot people who would benefit from training, and share what works across the team. **This is not a surveillance tool.** TOLVYN never reads or stores prompt content — only metadata about *how much* and *what model*, never *what was said*.

From the dashboard:

> *TOLVYN surfaces aggregated team patterns. Individual prompt content is never read or stored.*

---

## How it works

Set the `X-Tolvyn-User` header on every request. The header value identifies the person who made the request — typically an email, employee ID, or display name.

### SDK mode (Python)

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    user="alice@company.com",
)
```

### SDK mode (Node.js)

```javascript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  user: 'alice@company.com',
});
```

### SDK mode (Go)

```go
client := tolvynopenai.NewClient(tolvyn.ClientOptions{
    TolvynAPIKey: "tlv_live_...",
    User:         "alice@company.com",
})
```

### Proxy mode

```python
client = OpenAI(
    api_key="tlv_live_...",
    base_url="https://proxy.tolvyn.io/v1/proxy/openai",
)

client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
    extra_headers={"X-Tolvyn-User": "alice@company.com"},
)
```

### curl

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "X-Tolvyn-User: alice@company.com" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
```

### Helicone migration

If you're migrating from Helicone, the proxy automatically reads `Helicone-User-Id` as a fallback when `X-Tolvyn-User` is absent. No code change required during migration.

---

## What is tracked

Per-user aggregation from `HandleUsageByUser` (`internal/api/client.go:1921`). One row per user with these fields:

| Field | Type | Meaning |
|---|---|---|
| `user` | string | The exact value sent in `X-Tolvyn-User` |
| `request_count` | int64 | Total proxied requests for this user in the period |
| `total_cost_usd` | string | Sum of `cost_microdollars`, formatted as USD |
| `avg_cost_usd` | string | Average cost per request |
| `model_count` | int64 | Distinct `model_id`s used by this user |
| `top_model` | string | Most-used model family (statistical mode) |
| `error_count` | int64 | Requests with `status_code >= 400` |
| `retry_rate_pct` | float64 | `error_count / request_count × 100` |

Default lookback is 30 days. Override with `?from=YYYY-MM-DD&to=YYYY-MM-DD`. Top 50 users by cost are returned.

---

## What is not tracked

- Prompt text — never read, never stored
- Response text — never read, never stored
- File attachments — never read, never stored
- Personal identifiers beyond the value you put in `X-Tolvyn-User`

The TOLVYN proxy reads the response body only to extract token counts (`usage` JSON in OpenAI/Anthropic responses). Nothing else from the response is persisted.

---

## Dashboard

**Team Insights** page. Columns:

| Column | Meaning |
|---|---|
| User | The value from `X-Tolvyn-User` |
| Requests | `request_count` |
| Spend | `total_cost_usd` |
| Avg Cost | `avg_cost_usd` |
| Top Model | `top_model` |
| Retry Rate | `retry_rate_pct` — color-coded |

### Retry rate color coding

From `dashboard-client/src/pages/UsersDashboard.jsx:23-27`:

| Retry rate | Color | Interpretation |
|---|---|---|
| `< 2%` | Green | Healthy |
| `2–10%` | Yellow | Worth checking |
| `> 10%` | Red | Likely prompt or model misuse |

High retry rates often mean the user is asking the wrong model, getting tool-call format errors, or feeding malformed inputs. A coaching opportunity, not a flag for HR.

---

## Per-user detail

Click any user on the dashboard, or call `GET /v1/usage/by-user/{user}` directly. Returns (`HandleUsageByUserDetail`, `client.go:2005`):

```json
{
  "user": "alice@company.com",
  "request_count": 1843,
  "total_cost_usd": "124.50",
  "avg_cost_usd": "0.0675",
  "retry_rate_pct": 1.8,
  "model_breakdown": [
    {"model_id": "gpt-4o", "requests": 1200, "cost_usd": "98.20", "pct": 78.9},
    {"model_id": "gpt-4o-mini", "requests": 643, "cost_usd": "26.30", "pct": 21.1}
  ],
  "daily_spend": [
    {"date": "2026-05-17", "cost_usd": "12.40", "requests": 184},
    {"date": "2026-05-16", "cost_usd": "10.80", "requests": 162}
  ],
  "top_services": [
    {"service_name": "summarizer", "requests": 1200, "cost_usd": "84.10"},
    {"service_name": "embeddings", "requests": 543, "cost_usd": "12.20"}
  ]
}
```

- `model_breakdown` — all models the user touched, sorted by spend, with percentage share
- `daily_spend` — last 7 days, most recent first
- `top_services` — top 5 services by spend

---

## CLI

```bash
tolvyn cost --by user
tolvyn cost --by user --from 2026-05-01 --to 2026-05-17
```

Output (from `cmd_cost.go`):

```
User                                  Requests   Spend       Avg Cost    Top Model               Retry Rate
alice@company.com                     1,843      $124.50     $0.0675     gpt-4o                  1.8%
bob@company.com                       1,201      $89.10      $0.0742     gpt-4o                  3.2%
carol@company.com                     412        $12.40      $0.0301     gpt-4o-mini             0.5%
```

Retry rate is color-coded in the terminal (green / yellow / red) using the same thresholds as the dashboard.

---

## API

### `GET /v1/usage/by-user`

Query parameters:

| Param | Default | Description |
|---|---|---|
| `from` | `now - 30 days` | RFC 3339 or `YYYY-MM-DD` |
| `to` | `now` | RFC 3339 or `YYYY-MM-DD` |

Returns:

```json
{
  "users": [
    { "user": "alice@company.com", "request_count": 1843, "total_cost_usd": "124.50", ... }
  ],
  "total": 12,
  "period_start": "2026-04-17T00:00:00Z",
  "period_end": "2026-05-17T00:00:00Z"
}
```

Top 50 users by cost. Users with no `X-Tolvyn-User` set are excluded (`tolvyn_user IS NOT NULL AND tolvyn_user != ''`).

### `GET /v1/usage/by-user/{user}`

`{user}` is URL-encoded. Same date params as above. Returns the detail shape shown earlier.

---

## Empty state

If the page is empty or the API returns `"total": 0`:

1. Check that your application sets `X-Tolvyn-User` on requests
2. Check the date range — by default only the last 30 days are shown
3. Verify the value is non-empty — the query filters `tolvyn_user IS NOT NULL AND tolvyn_user != ''`
4. If you migrated from Helicone, the proxy falls back to `Helicone-User-Id` automatically — but if you removed that header without adding `X-Tolvyn-User`, recent requests will be unattributed
5. Run `tolvyn requests --limit 5` to confirm a few recent requests have non-null user attribution

---

## Privacy note

TOLVYN stores **metadata only**:

| What we store | What we never store |
|---|---|
| Model ID, token counts, cost, latency, status | Prompt text |
| `X-Tolvyn-User` value (e.g. email or ID) | Response text |
| `X-Tolvyn-Team`, `X-Tolvyn-Service`, `X-Tolvyn-Agent`, `X-Tolvyn-End-Customer` | Tool call arguments or results |
| Request timestamp, IP, user-agent | File / image attachment contents |

If you must avoid storing user identifiers, use opaque IDs (e.g. UUIDs from your auth system) instead of emails — they map back to the user only inside your own systems.

---

## See also

- [End Customers](end-customers.md) — for SaaS customer-level attribution
- [Agents](agents.md) — for per-agent budgets
- [API Reference: Usage](../reference/api.md#usage-and-analytics)
- [CLI Reference: `tolvyn cost`](../reference/cli.md#tolvyn-cost)
