# Agent Budgets

Agents — coding assistants, batch processors, customer-facing bots — fail in ways humans rarely do. A bad prompt loop runs overnight, a missing stop condition burns $10,000 before anyone wakes up, a forking task spawns a hundred sub-agents that each spawn ten more. Agent budgets are TOLVYN's answer: a hard spend cap tied to an agent name, enforced at the proxy.

---

## Why agents need their own budgets

Team or service budgets aren't enough when a single agent runs *within* a service. A team budget of `$2,000/month` doesn't help when one rogue agent burns the whole monthly allocation in an hour. You want the agent itself to have a ceiling that's tight to its expected workload.

Common scenarios:

| Agent | Risk | Why a per-agent budget helps |
|---|---|---|
| Claude Code | Runs unattended in a long session | Prevents one bad task from costing thousands |
| Cursor / Copilot inside a team | Heavy users dominate team budget | Caps individual contribution |
| Nightly batch processor | Loops if a query never terminates | Hard limit halts the loop before billing damage |
| Customer-facing support bot | Adversarial users craft expensive prompts | Per-bot cap survives prompt-injection attacks |

---

## The `X-Tolvyn-Agent` header

Set it on every request your agent makes:

### SDK

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    agent="claude-code",
)
```

```javascript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  agent: 'claude-code',
});
```

```go
client := tolvynopenai.NewClient(tolvyn.ClientOptions{
    TolvynAPIKey: "tlv_live_...",
    Agent:        "claude-code",
})
```

### Proxy / curl

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "X-Tolvyn-Agent: claude-code" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
```

### What value to use

The agent name should identify the *agent type*, not a specific run. Good values:

- `claude-code`
- `cursor`
- `nightly-report-bot`
- `support-bot-v2`

If you have multiple instances of the same agent type, use the same agent name for all of them — the budget applies to the type, not the instance. To track individual instances, set `X-Tolvyn-User` to the instance ID and combine with the agent header.

---

## How agent budgets work

An agent budget is a row in the `budgets` table with `scope_type = 'agent'` and a `scope_id` that is **derived from the agent name** via a deterministic UUIDv5 transform. The proxy resolves the budget on every request by hashing the value of `X-Tolvyn-Agent` and looking it up.

### Comparison with team and service budgets

| Property | Team budget | Service budget | Agent budget |
|---|---|---|---|
| Source field | Team ID from API key | `X-Tolvyn-Service` header | `X-Tolvyn-Agent` header |
| Scope ID | Real UUID from `teams` table | Plain string | UUIDv5 derived from name |
| Hierarchy | Outermost specific scope | Mid-level | **Most-specific** (resolved first) |
| Auto-reset on period | Yes | Yes | Yes |
| Hard mode | Yes (HTTP 429) | Yes | Yes |
| Soft mode | Yes (alerts) | Yes | Yes |

Agent budgets are checked **before** team and service budgets in the resolver's most-specific-first ordering (`internal/budget/resolver.go:283`). A request to a service inside a team will have agent → service → team → organization budgets all evaluated; the agent is the tightest scope and is most likely to trigger first.

---

## AgentNameToUUID — how the conversion works

The `scope_id` column is UUID-typed for foreign-key compatibility with teams and services. Agents don't have a table to draw real UUIDs from, so TOLVYN derives a deterministic UUID from the agent name.

From `internal/budget/resolver.go:17-36`:

```go
// AgentNameToUUID converts an agent name string to a deterministic UUID v5
// (SHA-1, DNS namespace) so it can be stored in the UUID-typed scope_id column.
func AgentNameToUUID(agentName string) string {
    // DNS namespace UUID: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
    ns := []byte{
        0x6b, 0xa7, 0xb8, 0x10,
        0x9d, 0xad,
        0x11, 0xd1,
        0x80, 0xb4,
        0x00, 0xc0, 0x4f, 0xd4, 0x30, 0xc8,
    }
    h := sha1.New()
    h.Write(ns)
    h.Write([]byte(agentName))
    sum := h.Sum(nil)
    // ... format as UUIDv5
}
```

This is the standard UUID v5 algorithm (SHA-1 hashed with the RFC 4122 DNS namespace). Properties:

- **Deterministic.** `AgentNameToUUID("claude-code")` always returns the same UUID, regardless of when or where it runs
- **No table required.** No insert needed to "register" an agent name
- **Cross-tenant isolation.** The UUID is the same across all tenants, but queries are scoped by `tenant_id`, so two different tenants using the same agent name see independent budgets
- **Case-sensitive.** `Claude-Code` and `claude-code` produce different UUIDs; pick a convention and stick to it
- **Trim whitespace.** A trailing space changes the hash. The proxy doesn't normalize — be consistent

Equivalent in Python for verification:

```python
import uuid
uuid.uuid5(uuid.NAMESPACE_DNS, "claude-code")
# UUID('d92a1f4d-...')
```

---

## Creating an agent budget

### Dashboard

**Budgets → Create Budget**:

1. Choose **Agent** from the **Scope** dropdown
2. Enter the agent name exactly as it appears in `X-Tolvyn-Agent` (case- and whitespace-sensitive)
3. Enter the amount in USD
4. Pick period (daily / weekly / monthly) and mode (soft / hard)
5. Save

The dashboard displays the agent name in the budget list (not the derived UUID) — the `settings` JSONB column stores `{"agent_name": "<name>"}` for display purposes (`internal/api/client.go:1149-1159`).

### CLI

```bash
tolvyn budgets create \
  --scope agent \
  --agent claude-code \
  --amount 100 \
  --period daily \
  --mode hard
```

CLI flags from `cmd/tolvyn-cli/cmd_budgets.go`:

| Flag | Default | Notes |
|---|---|---|
| `--scope` | `org` | Must be `agent` for agent budgets |
| `--agent` | — | Required when `--scope=agent` |
| `--amount` | — | Required, > 0 |
| `--period` | `monthly` | `daily`, `weekly`, `monthly` |
| `--mode` | `soft` | `soft` or `hard` |

### API

`POST /v1/budgets`:

```bash
curl -X POST https://api.tolvyn.io/v1/budgets \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_type":  "agent",
    "agent_name":  "claude-code",
    "amount_usd":  100,
    "period":      "daily",
    "mode":        "hard"
  }'
```

Server-side: the handler validates that `agent_name` is non-empty (`client.go:1149-1153`), derives the UUID via `budget.AgentNameToUUID(*body.AgentName)`, stores it in `scope_id`, and writes `{"agent_name": "<name>"}` into `settings` (`client.go:1154-1158`).

Response:

```json
{
  "id": "e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b",
  "scope_type": "agent",
  "agent_name": "claude-code",
  "amount_usd": 100,
  "period": "daily",
  "mode": "hard",
  "period_start": "2026-05-17T00:00:00Z",
  "period_end":   "2026-05-17T23:59:59Z"
}
```

---

## Hard vs soft mode

Identical behavior to team and service budgets — see [Budgets → Hard mode](budgets.md#hard-mode) for the full details.

**Hard mode:** when `current_spend >= amount`, the proxy rejects matching requests with:

```
HTTP 429 Too Many Requests
```

```json
{
  "error": "budget_exceeded",
  "budget_id": "e5f8a1b2-...",
  "scope": "agent",
  "limit_usd": "100.00",
  "spent_usd": "100.50",
  "message": "Budget limit reached. Contact your administrator."
}
```

The request that *crosses* the threshold is allowed; the *next* one is blocked (same off-by-one as other hard budgets).

**Soft mode:** alerts fire at 50%, 75%, 90%, and 100% thresholds (see [Alerts](alerts.md#budget-threshold-alerts)). Requests are never blocked. Useful for monitoring an agent without disrupting it.

---

## Agent attribution in the request log

Every proxied request stores the raw header value in the `requests.agent_name` column. View it:

- **Dashboard:** Request log → click any row → detail panel shows `agent_name`
- **CLI:** `tolvyn requests --limit 5 --json | jq '.data[] | {agent: .agent_name, cost: .cost_usd}'`
- **API:** `GET /v1/usage/requests` — each row has `agent_name`

---

## Combining with kill switch

Sometimes a hard budget isn't enough — you want to stop an agent *immediately*, without waiting for spend to accumulate. Use a kill switch with `--scope agent`:

```bash
tolvyn kill --scope agent --target claude-code --reason "runaway loop"
```

While active, every request with `X-Tolvyn-Agent: claude-code` is rejected with `HTTP 451 kill_switch_active`. To re-enable:

```bash
tolvyn kill undo <kill-id>
```

See [Kill Switch](kill-switch.md) for the full pattern. Kill switch + budget is a defense in depth — the budget provides the long-term cap, the kill switch provides emergency stop.

---

## Common patterns

### Coding agents (Claude Code, Cursor)

```python
client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    agent="claude-code",
    user=os.environ["USER"],   # the developer running it
)
```

Pair with a per-agent daily budget of $20–100. Most days an agent uses $5–20; the cap protects against pathological runs.

### Batch processing agents

```python
client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    agent="nightly-report-bot",
    service="reporting",
)
```

Pair with a per-agent monthly budget close to the expected monthly spend × 1.5. If an unexpected loop pushes 2× the normal cost, hard mode catches it.

### Customer-facing agents

```python
client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    agent="support-bot",
    end_customer=customer_id,
    user=session_user_id,
)
```

Triple-attribution: agent type + the customer the conversation is for + the actual user. All three are searchable independently. Pair with both an agent budget *and* a per-customer budget if you bill end-customers for AI usage.

### Multiple agent versions

```python
client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    agent=f"support-bot-{model_version}",
)
```

Use a version suffix when running A/B tests of agent prompts or model choices. Each version gets its own budget and its own retry rate in the dashboard, so you can compare cost-per-resolved-ticket between versions.

---

## See also

- [Budgets](budgets.md) — full budget mechanics including period rollover
- [Kill Switch](kill-switch.md) — for immediate emergency stop
- [Team Insights](team-insights.md) · [End Customers](end-customers.md)
- [API Reference: Budgets](../reference/api.md#budgets)
- [CLI Reference: `tolvyn budgets`](../reference/cli.md#tolvyn-budgets)
