# Kill Switch

The kill switch unconditionally blocks AI requests at the TOLVYN proxy. Unlike a budget (which tracks spend), a kill switch is a flag that immediately rejects matching requests with `HTTP 451 Unavailable For Legal Reasons` until it is explicitly deactivated.

Use it for incident response: a runaway agent, a leaked API key, a service in a bad state, or any time you need to stop the bleeding fast.

---

## Overview

| Property | Value |
|---|---|
| Block status | `HTTP 451` |
| Order in proxy | First — runs before budget check |
| State location | In-memory map + `kill_switches` table |
| Latency | Sub-millisecond (no DB round-trip) |
| Persistence | Survives server restart (loaded from DB at startup) |
| Scope | `all`, `team`, `service`, `agent`, `api_key` |
| Auto-deactivation | None — must be deactivated manually |

---

## Scope types

| Scope | Matches when |
|---|---|
| `all` | Any request from this tenant — global kill |
| `team` | Request's team ID equals `scope_value` |
| `service` | Request's `X-Tolvyn-Service` equals `scope_value` |
| `agent` | Request's `X-Tolvyn-Agent` equals `scope_value` |
| `api_key` | Request's TOLVYN API key prefix (first 12 chars) equals `scope_value` |

Source: `internal/kill/state.go` `IsKilled()` and `internal/proxy/proxy.go:262-287`.

### Check order

`IsKilled` short-circuits on the first match. Scopes are checked in strict priority order: `all` → `team` → `service` → `agent` → `api_key`. A higher-priority scope match wins regardless of when each kill was activated:

```go
for i := range entries {
    e := &entries[i]
    switch e.ScopeType {
    case "all":
        return true, e
    case "team":
        if teamID != "" && e.ScopeValue == teamID { return true, e }
    case "service":
        if serviceName != "" && e.ScopeValue == serviceName { return true, e }
    case "agent":
        if agentName != "" && e.ScopeValue == agentName { return true, e }
    case "api_key":
        if apiKeyPrefix != "" && e.ScopeValue == apiKeyPrefix { return true, e }
    }
}
```

Priority is by scope type, not activation time. A team-scope kill activated at 09:00 beats an agent-scope kill activated at 08:30 if both would match the request — `team` is checked before `agent`. An `all`-scope kill always wins.

---

## Order in the proxy

The kill check is the **first** authorization gate after JWT validation and tenant-scope resolution. It runs **before** the budget check.

```
1. JWT validation
2. Tenant + scope extraction (team, service, agent, API key)
3. Kill switch check     ← rejects with 451
4. Provider key lookup
5. Budget check          ← rejects with 429
6. Forward to provider
```

Source: `internal/proxy/proxy.go:261` (kill) vs `proxy.go:306` (budget).

### HTTP 451 response body

When the kill matches, the proxy returns:

```json
{
  "error": "kill_switch_active",
  "message": "<reason or 'Kill switch active'>",
  "kill_id": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "scope_type": "team",
  "scope_value": "backend",
  "activated_at": "2026-05-17T15:55:00Z"
}
```

Status: `451 Unavailable For Legal Reasons`. Source: `proxy.go:275-285`.

---

## In-memory store

For sub-millisecond proxy-path latency, kill switches are kept in an in-memory map keyed by tenant ID. Source: `internal/kill/state.go`.

### Startup

At server boot, `kill.Init(db)` loads every active kill switch (`deactivated_at IS NULL`) from the database into the in-memory store. The proxy does not begin serving requests until this completes.

```go
rows, err := db.Query(`
    SELECT id::text, tenant_id::text, scope_type, scope_value,
           COALESCE(reason,''), activated_at
    FROM kill_switches
    WHERE deactivated_at IS NULL`)
```

### Live updates

The API handlers keep memory and DB in sync:

- `POST /v1/kill` → DB insert, then `kill.Store.Activate(tenantID, entry)` adds to memory
- `DELETE /v1/kill/{id}` → DB update (`deactivated_at = now()`), then `kill.Store.Deactivate(tenantID, killID)` removes from memory

### Restart behavior

Active kill switches **survive a server restart** because they are loaded from the DB at startup. Activations applied while the server was down (e.g. via direct SQL) are also picked up.

### Reconciliation loop

The store reloads from the database every 60 seconds via `StartReconciliationLoop`, so kill switches inserted directly via SQL (or restored from backup, or activated on a replica) eventually re-sync without a server restart. Worst-case staleness window is 60 seconds.

For tighter consistency, prefer the API — the in-memory store is updated synchronously on every `Activate` / `Deactivate` call from `POST /v1/kill` and `DELETE /v1/kill/{id}`. The reconciliation loop is a safety net, not the primary sync mechanism.

---

## Activating

### Dashboard

**Kill switch** page (or the red banner button on any page during an incident):

1. Pick a scope
2. Enter the target (team name, service name, etc.) — or leave blank for `all`
3. Optional: enter a reason (recorded in audit log + included in 451 response body)
4. Click **Activate** — confirmation prompt
5. Matching requests start receiving 451 immediately

### CLI

```bash
tolvyn kill --scope team --target backend --reason "runaway agent"
```

Flags from `cmd/tolvyn-cli/cmd_kill.go`:

| Flag | Default | Description |
|---|---|---|
| `--scope` | — (required) | `team`, `service`, `agent`, `api_key`, or `all` |
| `--target` | — | Target value; required unless `--scope=all` |
| `--reason` | — | Recorded in audit log and shown in 451 response |

The CLI prompts for confirmation before activating. Output:

```
Kill team "backend"? This will immediately block all AI requests. [y/N]: y
✓ Kill switch activated. All matching AI requests are now blocked.
  ID:    a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
  Scope: team / backend
  Reason: runaway agent

To undo: tolvyn kill undo a1b2c3d4-...
```

### API

```bash
curl -X POST https://api.tolvyn.io/v1/kill \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_type":  "team",
    "scope_value": "backend",
    "reason":      "runaway agent loop"
  }'
```

Request fields:

| Field | Type | Required |
|---|---|---|
| `scope_type` | string | yes — `team`, `service`, `agent`, `api_key`, `all` |
| `scope_value` | string | yes (or `*` for `all`) |
| `reason` | string | no |

Response (201 Created):

```json
{
  "id": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "scope_type": "team",
  "scope_value": "backend",
  "reason": "runaway agent loop",
  "activated_by": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "activated_at": "2026-05-17T15:55:00Z"
}
```

Tenants are capped at **10 active kill switches** at any time. Attempting an 11th returns `400 limit_exceeded`.

---

## Listing active kills

### CLI

```bash
tolvyn kill list
```

Aliases: `tolvyn kill ls`.

```
ID         SCOPE    TARGET     REASON                 ACTIVATED
a1b2c3d…   team     backend    runaway agent loop     2026-05-17 15:55

1 active kill switch(es)
```

### API

```bash
curl https://api.tolvyn.io/v1/kill -H "Authorization: Bearer <jwt>"
```

Returns both active and recently-deactivated entries (filter on `deactivated_at` if you only want active ones).

---

## Deactivating

### CLI

```bash
tolvyn kill undo a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

Aliases: `tolvyn kill deactivate`, `tolvyn kill rm`.

```
✓ Kill switch a1b2c3d4-... deactivated. AI requests are flowing again.
```

### API

```bash
curl -X DELETE https://api.tolvyn.io/v1/kill/<kill-id> \
  -H "Authorization: Bearer <jwt>"
```

Returns `204 No Content`. The DB row is updated with `deactivated_at = now()` and `deactivated_by = <tenant id>` for audit. The row is not deleted — kill history is permanent.

The in-memory store is updated synchronously with the DB so subsequent requests stop receiving 451 within milliseconds.

---

## Kill vs budget — comparison

| | Kill switch | Budget (hard mode) |
|---|---|---|
| Purpose | Unconditional block, fast | Spend cap with running counter |
| Trigger | Active state | `current_spend >= amount` |
| HTTP status | `451 Unavailable For Legal Reasons` | `429 Too Many Requests` |
| Error code | `kill_switch_active` | `budget_exceeded` |
| Order in proxy | First — `proxy.go:261` | After kill — `proxy.go:306` |
| State | In-memory + DB | DB only (read per request) |
| Latency | Sub-ms | Single SELECT per request |
| Auto-reset | Never | On period rollover |
| Use case | Incident response | Long-term cost control |
| Limit per tenant | 10 active | No limit |
| Survives server restart | Yes (loaded at startup) | Yes (DB-backed) |

Both can be active simultaneously and they answer different questions: "should this request happen *at all*?" (kill) vs. "have we spent too much?" (budget).

---

## Operational pattern

Recommended kill-switch playbook during an incident:

1. **Scope as narrowly as possible.** Start with `--scope agent --target <agent name>` or `--scope api_key --target <prefix>` — these are surgical. Avoid `--scope all` unless you genuinely cannot identify the cause.
2. **Always record a reason.** It surfaces in the 451 response, gives downstream consumers (and on-call engineers) context, and lands in the audit log.
3. **Deactivate as soon as the cause is fixed.** Kill switches are not auto-deactivating. A forgotten `--scope all` will leave the entire org's AI traffic blocked indefinitely.
4. **Verify before deactivating.** Once you `kill undo`, the next request will hit the provider. Make sure the underlying issue (bug, leaked key, broken loop) is resolved first.

---

## See also

- [Budgets](budgets.md) — spend-based limits
- [Alerts](alerts.md) — incident detection
- [API Reference: Kill switches](../reference/api.md#kill-switches)
- [CLI Reference: `tolvyn kill`](../reference/cli.md#tolvyn-kill)
