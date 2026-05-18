# Budgets

Budgets cap AI spend by team, service, agent, or organization. A budget has a scope, an amount in USD, a period, and one of two modes: `soft` (alert only) or `hard` (block at the proxy before the provider is called).

---

## Overview

A budget is a row in the `budgets` table with these key fields:

| Field | Description |
|---|---|
| `scope_type` | `organization`, `team`, `service`, or `agent` |
| `scope_id` | The team/service/agent identifier (or NULL for `organization`) |
| `amount_microdollars` | Limit in microdollars (int64) |
| `period` | `daily`, `weekly`, or `monthly` |
| `mode` | `soft` or `hard` |
| `current_spend_microdollars` | Running spend in this period |
| `period_start` / `period_end` | Inclusive bounds of the current period |
| `settings` | JSONB; stores `last_alerted_threshold` for dedup |

Multiple budgets can apply to a single request (e.g. an org budget plus a team budget plus an agent budget). All are tracked simultaneously; hard limits across *any* applicable budget block the request.

---

## Scope hierarchy

When the proxy receives a request, it resolves all budgets matching the request's tenant, team, service, and agent. The resolver returns them sorted **most-specific first**:

```
agent  >  service  >  team  >  organization
```

Source: `internal/budget/resolver.go` — `sortBudgets()` uses the explicit ordering `{"agent": 0, "service": 1, "team": 2, "organization": 3}`.

A request with team `backend`, service `summarizer`, and agent `claude-code` is checked against:

- Any `agent` budget where `scope_id = AgentNameToUUID("claude-code")`
- Any `service` budget where `scope_id = "summarizer"`
- Any `team` budget where `scope_id = "<backend team UUID>"`
- Any `organization` budget (no `scope_id`)

All four can exist simultaneously and all four spend counters tick on every request.

---

## Hard mode

When `mode = "hard"`, requests are rejected at the proxy **before the provider is called** if the budget is exceeded.

### What the proxy returns

The check is at `internal/proxy/proxy.go:306`:

```go
if b.Mode == "hard" && b.CurrentSpendMicrodollars >= b.AmountMicrodollars {
    // Hard limit exceeded — reject BEFORE calling provider.
    w.WriteHeader(http.StatusTooManyRequests)  // 429
    json.NewEncoder(w).Encode(map[string]string{
        "error":     "budget_exceeded",
        "budget_id": b.ID,
        "scope":     b.ScopeType,
        "limit_usd": formatUSD(b.AmountMicrodollars),
        "spent_usd": formatUSD(b.CurrentSpendMicrodollars),
        "message":   "Budget limit reached. Contact your administrator.",
    })
    return
}
```

Status: **HTTP 429 Too Many Requests**.

```json
{
  "error": "budget_exceeded",
  "budget_id": "e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b",
  "scope": "team",
  "limit_usd": "1000.00",
  "spent_usd": "1000.50",
  "message": "Budget limit reached. Contact your administrator."
}
```

### What does NOT count

- The blocked request **never reaches the provider** — no upstream cost is incurred
- The blocked request **is not metered** — no row appears in the request log
- No tokens, no provider response time, no cost — the request exists only as a 429 to the client

### A subtle behavior

The check is `current_spend >= amount`. Spend is checked **before** the current request's cost is added. The *first* request that pushes spend over the limit is therefore *allowed*; only the *next* request sees `current_spend >= amount` and is rejected. In practice this means:

- A hard budget of `$10.00` with `$9.99` spent will allow one more request, even if that request costs `$50.00` and pushes spend to `$59.99`
- The request *after* that will be rejected with `429`

To get strict pre-flight blocking, set the budget slightly lower than the true ceiling (e.g. `$9.50` to protect a true `$10.00` limit, assuming worst-case request costs around `$0.50`).

---

## Soft mode

When `mode = "soft"`, the budget tracks spend and fires alerts at threshold crossings but **never blocks requests**. The proxy continues to forward requests to the provider after a soft budget is exceeded.

### Threshold levels

From `internal/alert/threshold.go:16`:

```go
var thresholds = []int{50, 75, 90, 100}
```

| Threshold | Severity |
|---|---|
| 50% | `info` |
| 75% | `warning` |
| 90% | `warning` |
| 100% | `critical` |

When a request pushes utilization past a threshold, an alert is inserted into the `alerts` table, an email is sent to the tenant's address (if SMTP is configured), and a webhook is dispatched (event type `alert.budget_threshold`).

### Dedup

Each budget tracks the last threshold it alerted for in its `settings.last_alerted_threshold` JSONB field. Once `90%` has fired, `50%`, `75%`, and `90%` are skipped for the rest of the period; only `100%` can still fire. The field resets to empty when the period rolls over.

---

## Period types

| Period | Bounds (UTC) |
|---|---|
| `daily` | 00:00:00 → 23:59:59 of the same calendar day |
| `weekly` | Monday 00:00:00 → Sunday 23:59:59 |
| `monthly` | 1st of month 00:00:00 → last day of month 23:59:59 |

Computed in `internal/budget/resolver.go` `periodStart()` / `periodEnd()`.

### Auto-reset

When the proxy resolves budgets for a request and finds one whose `period_end` is in the past, it resets that budget inline:

```sql
UPDATE budgets
SET    current_spend_microdollars = 0,
       period_start = <new start>,
       period_end   = <new end>,
       settings     = '{}'
WHERE  id = ?
  AND  period_end < now()
```

This means:

- A monthly budget rolls over at the start of each calendar month
- `current_spend_microdollars` resets to zero
- The `last_alerted_threshold` setting resets — so 50%/75%/90%/100% will fire again in the new period
- The reset is idempotent: if two requests arrive simultaneously, only the first wins the `UPDATE`; the second re-reads the updated row

The dashboard's budget list endpoint also calls `ResetExpiredForTenant()` so the UI reflects the current period even if no proxy request has triggered the reset yet.

---

## Creating a budget

### Dashboard

**Budgets → Create Budget**:

1. Pick a scope (Organization / Team / Service / Agent)
2. Enter the amount in USD
3. Pick a period (daily / weekly / monthly)
4. Pick a mode (soft / hard)
5. Save

For a team or service scope, the dashboard provides a picker for `scope_id`. For agent scope, the agent name is what the SDK / proxy sends in `X-Tolvyn-Agent`.

### API

`POST /v1/budgets` (see [API Reference](../reference/api.md#budgets) for the full schema):

```bash
curl -X POST https://api.tolvyn.io/v1/budgets \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_type":  "team",
    "scope_id":    "b3e2c8a4-1f6d-4a8c-9e0f-7d1c4b2a3e5f",
    "amount_usd":  1000,
    "period":      "monthly",
    "mode":        "hard"
  }'
```

### CLI

```bash
tolvyn budgets create \
  --scope team \
  --team backend \
  --amount 1000 \
  --period monthly \
  --mode hard
```

CLI flags from `cmd/tolvyn-cli/cmd_budgets.go`:

| Flag | Default | Notes |
|---|---|---|
| `--scope` | `org` | `org` → `organization` server-side |
| `--team` | — | Required when `--scope=team` |
| `--service` | — | Required when `--scope=service` |
| `--agent` | — | Required when `--scope=agent` |
| `--amount` | — | Required, > 0 |
| `--period` | `monthly` | `monthly`, `weekly`, `daily` |
| `--mode` | `soft` | `soft` or `hard` |

---

## Agent budgets

Agents are identified by the `X-Tolvyn-Agent` request header (or the `agent` SDK option). When you create an agent budget, the agent **name** is what you pass — the server derives a deterministic UUID and stores it in the `scope_id` column.

From `internal/budget/resolver.go:17-36`:

```go
// AgentNameToUUID converts an agent name string to a deterministic UUID v5
// (SHA-1, DNS namespace) so it can be stored in the UUID-typed scope_id column.
func AgentNameToUUID(agentName string) string {
    // DNS namespace UUID: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
    ns := []byte{0x6b, 0xa7, 0xb8, 0x10, ...}
    h := sha1.New()
    h.Write(ns)
    h.Write([]byte(agentName))
    // ... format as UUIDv5
}
```

Properties:

- Same agent name + same namespace always yields the same UUID
- No collision-resolution table needed
- Two agents with the same name across different tenants share the same UUID, but they are separated by `tenant_id` in every query
- Agent names are case-sensitive — `Claude-Code` and `claude-code` are different agents

At budget-resolution time, the proxy reads `X-Tolvyn-Agent`, runs the same UUID derivation, and queries `WHERE scope_id::text = $4`. No reverse-lookup is needed.

---

## Kill switch vs budget

| | Budget | Kill switch |
|---|---|---|
| Purpose | Spend cap with running counter | Unconditional block |
| What it tracks | `current_spend_microdollars` per period | Nothing — just a flag |
| When it blocks | `mode=hard` AND `spend >= amount` | Always (while active) |
| Order in proxy | After kill check (`proxy.go:296`) | First check (`proxy.go:261`) |
| HTTP response | `429 budget_exceeded` | `451 kill_switch_active` |
| Auto-reset | Yes, on period rollover | No — must be deactivated manually |
| Use for | Long-term cost control | Incident response, runaway agents |

Both can be active simultaneously. Kill switches are checked first.

---

## See also

- [Alerts](alerts.md) — budget threshold alert delivery
- [Kill Switch](kill-switch.md) — emergency unconditional block
- [API Reference: Budgets](../reference/api.md#budgets)
- [CLI Reference: `tolvyn budgets`](../reference/cli.md#tolvyn-budgets)
