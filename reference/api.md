# REST API Reference

Every endpoint that TOLVYN exposes, sourced from `cmd/tolvyn-server/main.go` route registrations and the corresponding handlers in `internal/api/`.

---

## Overview

| Concern | Value |
|---|---|
| Management API base | `https://api.tolvyn.io` |
| Proxy base | `https://proxy.tolvyn.io/v1/proxy/{provider}/` (`openai`, `anthropic`, `google`) |
| Content type | `application/json` (unless noted) |
| Client auth | `Authorization: Bearer <JWT>` from `POST /v1/auth/login` |
| Operator auth | `Authorization: Bearer <TOLVYN_OPERATOR_TOKEN>` on `/v1/operator/*` |
| Public endpoints | `GET /health`, `GET /v1/public/cost-index` — no auth |
| Error format | `{"error":"<code>","message":"<human-readable description>"}` |
| Timestamps | RFC 3339 UTC (`2026-05-17T15:51:38Z`) |
| Pagination | `?limit=<n>&offset=<n>` on list endpoints; defaults vary |
| Date filters | `?from=2026-04-01&to=2026-05-01` on usage/ledger/audit endpoints |
| Money | All cost values dual-encoded as `cost_microdollars` (int64) and `cost_usd` (string, fixed-precision) |

JWTs expire after 24 hours. Use `POST /v1/auth/refresh` with a valid (non-expired) token to issue a new one without re-prompting credentials.

---

## Health

### `GET /health`

No auth. Returns service status and database reachability. Used by load balancers and uptime monitoring.

```bash
curl https://api.tolvyn.io/health
```

```json
{"status":"ok","db":"ok","version":"1.0.0","timestamp":"2026-05-17T15:52:39Z"}
```

Returns `503 Service Unavailable` with `{"status":"degraded","db":"error"}` when the database is unreachable.

---

## Authentication

### `POST /v1/auth/signup`

No auth. Creates a tenant on the `free` plan and returns a JWT. Rate-limited per IP.

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Display name |
| `email` | string | yes | Must match `^[^@\s]+@[^@\s]+\.[^@\s]+$` |
| `password` | string | yes | Minimum 8 characters |

```bash
curl -X POST https://api.tolvyn.io/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@acme.com","password":"hunter2hunter2"}'
```

```json
{
  "tenant_id": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "email": "alice@acme.com",
  "plan_tier": "free",
  "token": "eyJhbGciOiJIUzI1NiIsInR..."
}
```

Errors: `400 invalid_email`, `400 password_too_short`, `400 email_taken`, `429 rate_limited`.

### `POST /v1/auth/login`

No auth. Returns a JWT on success. Rate-limited per IP.

| Field | Type | Required |
|---|---|---|
| `email` | string | yes |
| `password` | string | yes |

```bash
curl -X POST https://api.tolvyn.io/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@acme.com","password":"hunter2hunter2"}'
```

```json
{
  "tenant_id": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "email": "alice@acme.com",
  "plan_tier": "free",
  "token": "eyJhbGciOiJIUzI1NiIsInR...",
  "expires_at": "2026-05-18T15:52:39Z"
}
```

Errors: `401 invalid_credentials` (also returned when the email is not registered, to resist enumeration), `429 rate_limited`.

### `POST /v1/auth/refresh`

Requires JWT. Issues a fresh JWT for the same tenant.

```bash
curl -X POST https://api.tolvyn.io/v1/auth/refresh \
  -H "Authorization: Bearer <jwt>"
```

```json
{"token":"eyJhbGciOiJIUzI1NiIsInR...","expires_at":"2026-05-18T15:52:39Z"}
```

### `POST /v1/auth/logout`

Requires JWT. Revokes the calling JWT's JTI server-side so it cannot be reused before expiry.

```bash
curl -X POST https://api.tolvyn.io/v1/auth/logout \
  -H "Authorization: Bearer <jwt>"
```

```json
{"message":"logged out"}
```

### `POST /v1/auth/forgot-password`

No auth. Sends a reset link if the email is registered. **Always** returns `200` to avoid leaking which addresses exist. Rate-limited per IP.

| Field | Type | Required |
|---|---|---|
| `email` | string | yes |

```bash
curl -X POST https://api.tolvyn.io/v1/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@acme.com"}'
```

```json
{"message":"If that email is registered, you will receive a reset link shortly."}
```

### `POST /v1/auth/reset-password`

No auth. Consumes a token from the forgot-password email and sets a new password.

| Field | Type | Required |
|---|---|---|
| `token` | string | yes — 64-hex string from the reset email |
| `password` | string | yes — minimum 8 chars |

```bash
curl -X POST https://api.tolvyn.io/v1/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{"token":"<hex token>","password":"newpassword123"}'
```

```json
{"message":"Password reset successfully"}
```

Errors: `400 invalid_token`, `400 token_expired`.

---

## Account

All require JWT.

### `GET /v1/account`

Returns the authenticated tenant, including subscription summary. `slack_webhook_url` is masked (first 40 chars + `...`).

```bash
curl https://api.tolvyn.io/v1/account -H "Authorization: Bearer <jwt>"
```

```json
{
  "tenant_id": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "name": "Alice",
  "email": "alice@acme.com",
  "plan_tier": "free",
  "status": "active",
  "created_at": "2026-05-01T12:00:00Z",
  "updated_at": "2026-05-17T15:52:39Z",
  "digest_enabled": true,
  "data_share_enabled": true,
  "subscription": {
    "plan_tier": "free",
    "included_requests": 10000,
    "status": "active",
    "current_period_start": "2026-05-01T12:00:00Z",
    "current_period_end": null
  }
}
```

### `PUT /v1/account`

Updates name, weekly-digest preference, and/or password.

| Field | Type | Notes |
|---|---|---|
| `name` | string | Optional |
| `digest_enabled` | bool (nullable) | Optional |
| `current_password` | string | Required if `new_password` is set |
| `new_password` | string | Minimum 8 chars |

```bash
curl -X PUT https://api.tolvyn.io/v1/account \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice K."}'
```

### `DELETE /v1/account`

Soft-deletes the tenant (`status = 'cancelled'`) and revokes the calling JWT. Tenant data is retained for audit but the account cannot log in.

### `PUT /v1/account/integrations/slack`

| Field | Type | Required |
|---|---|---|
| `webhook_url` | string | yes — must start with `https://hooks.slack.com/` |

### `DELETE /v1/account/integrations/slack`

Removes the Slack webhook from `settings`.

### `PUT /v1/account/settings/data-share`

Opt in/out of contributing anonymized aggregate metadata to the public AI Cost Index.

| Field | Type | Required |
|---|---|---|
| `enabled` | bool | yes |

### `PUT /v1/account/settings`

Alias of `PUT /v1/account` for spec compatibility.

---

## Provider keys

All require JWT. Provider keys are AES-256-GCM encrypted server-side with a tenant-scoped DEK. Plaintext keys are never returned after creation.

### `POST /v1/provider-keys`

| Field | Type | Required | Description |
|---|---|---|---|
| `provider` | string | yes | `openai`, `anthropic`, or `google` |
| `key` | string | yes | Provider API key (`sk-...`, `sk-ant-...`, etc.) |

```bash
curl -X POST https://api.tolvyn.io/v1/provider-keys \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"provider":"openai","key":"sk-proj-..."}'
```

```json
{"id":"5a7c1f3d-8b22-4e9f-b1c4-2a8d6e3c4f5a","provider":"openai"}
```

### `GET /v1/provider-keys`

Lists configured provider keys without revealing key material.

```json
[
  {
    "id": "5a7c1f3d-8b22-4e9f-b1c4-2a8d6e3c4f5a",
    "provider": "openai",
    "key_version": 1,
    "last_rotated_at": null,
    "created_at": "2026-05-17T14:00:00Z"
  }
]
```

### `DELETE /v1/provider-keys/{id}`

Revokes a provider key. Returns `204 No Content`.

---

## Teams

All require JWT.

### `POST /v1/teams`

| Field | Type | Required |
|---|---|---|
| `name` | string | yes |
| `slug` | string | no — auto-derived if omitted |

### `GET /v1/teams`

```json
[
  {
    "id": "b3e2c8a4-1f6d-4a8c-9e0f-7d1c4b2a3e5f",
    "name": "Backend",
    "slug": "backend",
    "created_at": "2026-05-17T14:00:00Z"
  }
]
```

### `GET /v1/teams/{id}`

Returns one team.

### `PUT /v1/teams/{id}`

Updates `name` and/or `slug`.

### `DELETE /v1/teams/{id}`

Returns `204 No Content`.

---

## API keys

All require JWT.

### `POST /v1/api-keys`

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Human label |
| `environment` | string | no | `production` (default) or `test` — controls `tlv_live_` vs `tlv_test_` prefix |
| `team_id` | string | no | Scope the key to a team |
| `service_name` | string | no | Scope the key to a service |

```bash
curl -X POST https://api.tolvyn.io/v1/api-keys \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"name":"backend-prod","environment":"production"}'
```

```json
{
  "id": "c4d8e2f1-5b3a-4c7e-8f9d-1a2b3c4d5e6f",
  "key": "tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ",
  "prefix": "tlv_live_aB3x",
  "name": "backend-prod",
  "environment": "production"
}
```

The full `key` is shown **once**. The server stores `prefix` and a hash; recovery is not possible.

### `GET /v1/api-keys`

Lists keys without revealing key material.

```json
[
  {
    "id": "c4d8e2f1-5b3a-4c7e-8f9d-1a2b3c4d5e6f",
    "key_prefix": "tlv_live_aB3x",
    "name": "backend-prod",
    "environment": "production",
    "last_used_at": "2026-05-17T15:50:00Z",
    "revoked_at": null,
    "created_at": "2026-05-17T14:00:00Z"
  }
]
```

### `DELETE /v1/api-keys/{id}`

Revokes the key (sets `revoked_at`). Returns `204 No Content`.

---

## Budgets

All require JWT.

### `POST /v1/budgets`

| Field | Type | Required | Description |
|---|---|---|---|
| `scope_type` | string | yes | `organization`, `team`, `service`, `user`, `end_customer`, `agent`, or `all` |
| `scope_id` | string (nullable) | when applicable | Team ID, user ID, etc. |
| `agent_name` | string | when `scope_type=agent` | The agent identifier (UUID is derived from this) |
| `amount_usd` | number | yes | Must be > 0 |
| `period` | string | no | `daily`, `weekly`, `monthly` (default), `yearly` |
| `mode` | string | no | `soft` (default — alert only) or `hard` (block requests at the proxy) |

```bash
curl -X POST https://api.tolvyn.io/v1/budgets \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"scope_type":"team","scope_id":"b3e2c8a4-1f6d-4a8c-9e0f-7d1c4b2a3e5f","amount_usd":1000,"period":"monthly","mode":"hard"}'
```

```json
{
  "id": "e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b",
  "scope_type": "team",
  "amount_usd": 1000,
  "period": "monthly",
  "mode": "hard",
  "period_start": "2026-05-01T00:00:00Z",
  "period_end": "2026-06-01T00:00:00Z"
}
```

### `GET /v1/budgets`

Returns all budgets with current spend.

### `GET /v1/budgets/{id}`

Returns one budget.

### `PUT /v1/budgets/{id}`

Updates `amount_usd`, `period`, and/or `mode`.

### `DELETE /v1/budgets/{id}`

Returns `204 No Content`.

---

## Usage and analytics

All require JWT. All accept `?from=`, `?to=`, `?team_id=`, `?model=`, `?service=` filters (additive).

### `GET /v1/usage/summary`

```json
{
  "total_cost_microdollars": 12500000,
  "total_cost_usd": "12.50",
  "total_requests": 1843,
  "total_tokens_input": 215000,
  "total_tokens_output": 38000,
  "top_models": [
    {"model_id":"gpt-4o","requests":1200,"cost_microdollars":9800000,"cost_usd":"9.80"}
  ],
  "top_teams": [
    {"team_id":"b3e2c8a4-...","requests":1843,"cost_microdollars":12500000,"cost_usd":"12.50"}
  ]
}
```

### `GET /v1/usage/requests`

Paginated request log. Accepts `?limit`, `?offset`, `?status`. Returns `?format=csv` as a streaming CSV download.

```json
{
  "data": [
    {
      "id": "...",
      "created_at": "2026-05-17T15:48:00Z",
      "provider": "openai",
      "model_id": "gpt-4o",
      "tokens_input": 120,
      "tokens_output": 35,
      "cost_microdollars": 8400,
      "cost_usd": "0.0084",
      "latency_total_ms": 412,
      "status_code": 200,
      "team_id": "b3e2c8a4-...",
      "service_name": "summarizer"
    }
  ],
  "total": 1843,
  "limit": 50,
  "offset": 0
}
```

### `GET /v1/usage/by-model`

Aggregates by `model_id`.

### `GET /v1/usage/by-team`

Aggregates by `team_id`.

### `GET /v1/usage/by-service`

Aggregates by `service_name`.

### `GET /v1/usage/by-user`

Aggregates by `X-Tolvyn-User` header value.

### `GET /v1/usage/by-user/{user}`

Detail view for one user — all requests, breakdown by service/model.

### `GET /v1/usage/by-end-customer`

Aggregates by `X-Tolvyn-End-Customer`.

### `GET /v1/usage/by-end-customer/{customer}`

Detail view for one end customer.

### `GET /v1/usage/requests/{id}`

Single request with full metadata, ledger linkage, and tag map.

### `GET /v1/usage/timeseries`

Cost over time, bucketed by day. Accepts `?bucket=day|hour`.

```json
{
  "data": [
    {"bucket":"2026-05-17","cost_microdollars":1850000,"cost_usd":"1.85","requests":215}
  ]
}
```

### Spec-path aliases

These resolve to the same handlers for spec compatibility:

| Alias | Routes to |
|---|---|
| `GET /v1/cost/summary` | `GET /v1/usage/summary` |
| `GET /v1/cost/by-team` | `GET /v1/usage/by-team` |
| `GET /v1/cost/by-model` | `GET /v1/usage/by-model` |
| `GET /v1/cost/by-service` | `GET /v1/usage/by-service` |
| `GET /v1/cost/timeseries` | `GET /v1/usage/timeseries` |
| `GET /v1/keys` / `POST /v1/keys` / `DELETE /v1/keys/{id}` | `/v1/api-keys` |
| `GET /v1/providers` / `POST /v1/providers` / `DELETE /v1/providers/{id}` | `/v1/provider-keys` |

---

## Audit

### `GET /v1/audit`

JWT-required. Paginated audit log with optional `?actor_type=`, `?action=`, `?from=`, `?to=`, and `?format=csv`.

```json
{
  "data": [
    {
      "id": "...",
      "tenant_id": "...",
      "actor_type": "user",
      "actor_id": "...",
      "action": "user.login",
      "resource_type": null,
      "resource_id": null,
      "ip_address": "1.2.3.4",
      "details": {},
      "created_at": "2026-05-17T15:51:00Z"
    }
  ],
  "total": 142,
  "limit": 50,
  "offset": 0
}
```

---

## Ledger

### `GET /v1/ledger`

JWT-required. Paginated hash-chained audit ledger of every proxied request. Accepts `?from=`, `?to=`, and `?format=csv` for full export.

```json
{
  "data": [
    {
      "id": "...",
      "sequence_number": 1843,
      "request_id": "...",
      "cost_microdollars": 8400,
      "cost_usd": "0.0084",
      "provider": "openai",
      "model_id": "gpt-4o",
      "tokens_input": 120,
      "tokens_output": 35,
      "hierarchy_path": "/backend/summarizer",
      "record_hash": "a1b2c3d4...",
      "created_at": "2026-05-17T15:48:00Z"
    }
  ],
  "total": 1843,
  "limit": 50,
  "offset": 0
}
```

CSV export includes `previous_hash` and `hmac_signature` for offline chain verification.

### `GET /v1/ledger/verify`

JWT-required. Walks the chain from `?from_seq=` to `?to_seq=` (defaults: 1 to current max), recomputes every hash, and verifies HMACs.

```json
{"valid":true,"records_checked":1843,"first_seq":1,"last_seq":1843}
```

On failure, returns the sequence number that broke the chain.

---

## Alerts

All require JWT.

### `GET /v1/alerts`

Paginated. Filters: `?acknowledged=true|false`, `?severity=info|warning|critical`.

```json
{
  "data": [
    {
      "id": "...",
      "alert_type": "budget_threshold",
      "severity": "warning",
      "title": "Backend team at 80% of monthly budget",
      "acknowledged": false,
      "created_at": "2026-05-17T10:00:00Z",
      "metadata": {"budget_id":"...","threshold":0.8}
    }
  ],
  "total": 8,
  "limit": 50,
  "offset": 0
}
```

### `PUT /v1/alerts/{id}/acknowledge`

Marks an alert acknowledged. Aliased as `PUT /v1/alerts/{id}/ack`.

### `POST /v1/alerts/test`

Sends a test alert through the configured Slack webhook and email destinations.

```json
{"sent_to":["slack","email"]}
```

---

## Models

All require JWT.

### `GET /v1/models`

List of supported models with current pricing.

```json
[
  {
    "model_id": "gpt-4o",
    "provider": "openai",
    "model_family": "gpt-4o",
    "pricing_input_per_mtok": 2.5,
    "pricing_output_per_mtok": 10.0,
    "pricing_cached_per_mtok": 1.25,
    "context_window": 128000,
    "pricing_updated_at": "2026-05-01T00:00:00Z"
  }
]
```

### `GET /v1/models/changes`

Active pricing change notifications — models whose price changed in the last 30 days and which the tenant has used.

### `GET /v1/models/changes-history`

Full history of pricing changes the tenant has acknowledged or dismissed.

---

## Savings

All require JWT.

### `GET /v1/savings`

Nightly-computed savings findings (small-token-request batching, idle models, downgrade opportunities, etc.).

```json
[
  {
    "id": "...",
    "finding_type": "model_downgrade_opportunity",
    "model_id": "gpt-4o",
    "service_name": "summarizer",
    "period_start": "2026-05-10T00:00:00Z",
    "period_end": "2026-05-17T00:00:00Z",
    "current_cost_microdollars": 8500000,
    "optimized_cost_microdollars": 2100000,
    "savings_microdollars": 6400000,
    "recommendation_text": "Switching to gpt-4o-mini for summarizer requests under 500 tokens would save ~$6.40/week.",
    "dismissed": false,
    "created_at": "2026-05-17T02:00:00Z"
  }
]
```

### `POST /v1/savings/{id}/dismiss`

Marks a finding dismissed so it stops appearing in the dashboard.

---

## Webhooks

All require JWT. Maximum 10 endpoints per tenant.

### `GET /v1/webhooks`

```json
{
  "webhooks": [
    {
      "id": "...",
      "url": "https://hooks.example.com/tolvyn",
      "event_types": ["alert.budget_threshold","alert.kill_switch"],
      "active": true,
      "created_at": "2026-05-17T14:00:00Z",
      "updated_at": "2026-05-17T14:00:00Z",
      "last_delivery_at": "2026-05-17T15:45:00Z",
      "last_status": "success"
    }
  ],
  "total": 1
}
```

### `POST /v1/webhooks`

| Field | Type | Required |
|---|---|---|
| `url` | string | yes — must start with `https://` |
| `event_types` | string[] | yes — at least one (`alert.budget_threshold`, `alert.kill_switch`, `alert.all`, ...) |

The response includes the HMAC `secret` **once**. Save it — it is the signing key for verifying incoming webhook deliveries.

```json
{
  "id": "...",
  "url": "https://hooks.example.com/tolvyn",
  "event_types": ["alert.all"],
  "active": true,
  "created_at": "...",
  "updated_at": "...",
  "secret": "wh_a1b2c3d4..."
}
```

Errors: `422 max_webhooks` (limit of 10), `400 invalid_url`, `400 invalid_event_type`.

### `PUT /v1/webhooks/{id}`

Updates `url`, `event_types`, and/or `active`.

### `DELETE /v1/webhooks/{id}`

Returns `204 No Content`.

### `GET /v1/webhooks/{id}/deliveries`

Recent delivery attempts. Each row includes `status_code`, `attempt_count`, `next_retry_at`, `delivered_at`, `error_message`.

### `POST /v1/webhooks/{id}/test`

Triggers a synthetic `webhook.test` delivery.

---

## Kill switches

All require JWT. Maximum 10 active per tenant.

### `POST /v1/kill`

| Field | Type | Required |
|---|---|---|
| `scope_type` | string | yes — `team`, `service`, `agent`, `api_key`, or `all` |
| `scope_value` | string | yes (or `*` for `all`) |
| `reason` | string | no |

```bash
curl -X POST https://api.tolvyn.io/v1/kill \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"scope_type":"team","scope_value":"backend","reason":"Investigating runaway agent"}'
```

```json
{
  "id": "...",
  "scope_type": "team",
  "scope_value": "backend",
  "reason": "Investigating runaway agent",
  "activated_by": "9f8d6a4e-...",
  "activated_at": "2026-05-17T15:55:00Z"
}
```

When the kill switch is active, matching requests at the proxy receive **HTTP 451 Unavailable For Legal Reasons** without reaching the provider.

### `GET /v1/kill`

Returns active and inactive kill switches.

### `DELETE /v1/kill/{id}`

Deactivates a kill switch (sets `deactivated_at`). Past activations are retained for audit. Returns `204 No Content`.

---

## Reconciliation

All require JWT. Upload provider invoice CSVs and the server matches them against the ledger.

### `POST /v1/reconciliation/upload`

`multipart/form-data` request.

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | yes | CSV export from the provider's billing console |
| `provider` | string | yes | `openai`, `anthropic`, or `google` |
| `invoice_month` | string | yes | `YYYY-MM` |

```bash
curl -X POST https://api.tolvyn.io/v1/reconciliation/upload \
  -H "Authorization: Bearer <jwt>" \
  -F "file=@openai-may-2026.csv" \
  -F "provider=openai" \
  -F "invoice_month=2026-05"
```

```json
{
  "id": "...",
  "provider": "openai",
  "invoice_month": "2026-05",
  "status": "completed",
  "invoice_total_usd": "1240.55",
  "ledger_total_usd": "1241.10",
  "delta_usd": "-0.55",
  "matched_lines": 12,
  "unmatched_lines": 0
}
```

Errors: `400 bad_request` (missing field), `400 invalid_provider`, `422 parse_error` (CSV malformed).

### `GET /v1/reconciliation`

Lists prior reconciliation runs.

### `GET /v1/reconciliation/{id}`

Detail view including per-model delta breakdown.

### `DELETE /v1/reconciliation/{id}`

Returns `204 No Content`.

---

## SSE — live tail

### `GET /v1/stream/tail`

JWT-required. Server-Sent Events stream of proxied requests in real time. Used by the dashboard's live-tail view and by `tolvyn tail` CLI.

Query parameters (all optional, additive):

| Param | Type | Description |
|---|---|---|
| `team` | string | Filter by team name |
| `service` | string | Filter by service name |
| `model` | string | Filter by model ID |
| `min_cost` | int64 | Filter by cost in microdollars |

```bash
curl -N https://api.tolvyn.io/v1/stream/tail \
  -H "Authorization: Bearer <jwt>"
```

```
data: {"type":"request","id":"...","provider":"openai","model_id":"gpt-4o","cost_microdollars":8400,"team":"backend","service":"summarizer","status_code":200,"latency_ms":412,"created_at":"2026-05-17T15:48:00Z"}

data: {"type":"heartbeat"}
```

A `heartbeat` event is emitted every 15 seconds so intermediate proxies do not close the connection.

---

## Proxy

Proxy endpoints accept the full OpenAI / Anthropic / Google API on the corresponding path prefix.

### `/v1/proxy/openai/*`

Proxies everything matching the OpenAI API surface. Authentication is `Authorization: Bearer tlv_live_...` (a TOLVYN API key, not an OpenAI key).

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
```

### `/v1/proxy/anthropic/*`

```bash
curl https://proxy.tolvyn.io/v1/proxy/anthropic/v1/messages \
  -H "Authorization: Bearer tlv_live_..." \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-haiku-4-5","max_tokens":1024,"messages":[{"role":"user","content":"Hello"}]}'
```

### `/v1/proxy/google/*`

Same pattern. The TOLVYN API key may be passed as `Authorization: Bearer` or `x-goog-api-key`.

### Proxy-specific status codes

| Status | Meaning |
|---|---|
| `401 invalid_token` | TOLVYN key invalid, revoked, or missing |
| `429 budget_exceeded` | A hard budget would be exceeded by this request |
| `451 killed` | A kill switch is active for the request's scope |
| `502 provider_key_missing` | No provider key configured for the requested provider |
| `503 Service Unavailable` | Proxy unreachable — SDK fail-open path triggers on this |

All other responses pass through from the provider unchanged.

---

## Public

### `GET /v1/public/cost-index`

No auth, CORS open. Aggregate AI cost data contributed by tenants who opted in to data sharing (k-anonymity threshold: 3 tenants).

Query parameters (all optional):

| Param | Type | Description |
|---|---|---|
| `provider` | string | Filter by `openai`, `anthropic`, `google` |
| `model_id` | string | Filter by specific model |
| `days` | int | Look-back window, 1–365 (default 90) |

```bash
curl "https://api.tolvyn.io/v1/public/cost-index?provider=openai&days=30"
```

```json
{
  "data": [
    {
      "date": "2026-05-17",
      "provider": "openai",
      "model_id": "gpt-4o",
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

Only buckets with `tenant_count >= 3` are returned.

---

## Operator API

All require `Authorization: Bearer <TOLVYN_OPERATOR_TOKEN>` (a server-side static token, not a JWT). These endpoints back the operator dashboard at `admin.tolvyn.io` and are not intended for tenant-facing use.

### `POST /v1/operator/auth/verify`

Verifies the operator token is valid. Returns `200 OK` if so, `401 unauthorized` otherwise.

### `GET /v1/operator/tenants`

Paginated list of all tenants.

### `GET /v1/operator/tenants/{id}`

Tenant detail: plan, subscription, usage rollup, provider key inventory, recent audit events.

### `PUT /v1/operator/tenants/{id}`

Operator update of tenant fields (`plan_tier`, `status`, `included_requests`, manual notes).

### `GET /v1/operator/stats`

Platform-wide stats: total tenants, total requests, total cost, MRR, churn signals.

### `GET /v1/operator/models`

All models across all tenants (operator-visible, includes inactive).

### `POST /v1/operator/models`

Adds a new model row.

### `PUT /v1/operator/models/{id}`

Updates a model (pricing, context window, deprecation status).

### `GET /v1/operator/models/changes`

Pricing change records — both auto-detected (via scraper) and operator-curated.

### `POST /v1/operator/models/changes`

Manually create a pricing change announcement.

### `POST /v1/operator/email/test`

Sends a test email through the operator's mail integration.

### `GET /v1/operator/health`

Operator-side health probe — returns DB connection pool stats, background goroutine status, queue depths.

### `GET /v1/operator/alerts/stats`

Counts of alerts emitted by type and severity over the last 30 days.

### `POST /v1/operator/savings/recompute`

Forces an immediate run of the nightly savings analyzer (normally scheduled at 02:00 UTC).

### `GET /v1/operator/pricing-candidates`

Pricing candidates surfaced by the daily scraper that have not yet been approved or rejected.

### `POST /v1/operator/pricing-candidates/{id}/approve`

Approves a candidate. The price is applied to the `models` table and appended to `pricing_history`.

### `POST /v1/operator/pricing-candidates/{id}/reject`

Rejects a candidate.

### `POST /v1/operator/scraper/run`

Triggers an immediate scrape of provider pricing pages (normally scheduled daily).

---

## Error reference

| Code | HTTP | Common cause |
|---|---|---|
| `invalid_body` | 400 | Request body is not valid JSON |
| `invalid_email` | 400 | Email format rejected by regex |
| `password_too_short` | 400 | Password less than 8 characters |
| `email_taken` | 400 | Email already registered (signup) |
| `name_required` | 400 | Required field omitted |
| `amount_required` | 400 | `amount_usd <= 0` on budget create |
| `invalid_scope_type` | 400 | `scope_type` not in allowed set |
| `scope_value_required` | 400 | Missing scope value when required |
| `limit_exceeded` | 400 | Tenant resource limit hit (10 kills, 10 webhooks, etc.) |
| `provider_required` / `key_required` | 400 | Provider key create — missing fields |
| `invalid_url` | 400 | Webhook URL not `https://` |
| `invalid_event_type` | 400 | Webhook event type not in registry |
| `invalid_provider` | 400 | Reconciliation provider not in `{openai,anthropic,google}` |
| `parse_error` | 422 | CSV body could not be parsed |
| `max_webhooks` | 422 | More than 10 webhook endpoints |
| `missing_auth` | 401 | No or malformed `Authorization` header |
| `invalid_token` | 401 | JWT signature failed or token revoked |
| `invalid_credentials` | 401 | Login email/password mismatch (also returned for unknown email) |
| `current_password_required` | 400 | Account update with `new_password` but no `current_password` |
| `rate_limited` | 429 | Per-IP rate limit on signup / login / forgot-password |
| `internal_error` | 500 | Server-side fault — check `/health`, contact support if persistent |

---

## See also

- [Integration Modes](../integration-modes.md)
- [Quickstart](../getting-started/quickstart.md)
- [CLI Reference](cli.md)
- [Python SDK](../sdks/python.md) · [Node.js SDK](../sdks/nodejs.md) · [Go SDK](../sdks/go.md)
