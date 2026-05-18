# Alerts

TOLVYN fires alerts on four conditions: budget threshold crossings, cost anomalies, model-family switches in a service, and operator-approved provider pricing changes. Alerts surface in the dashboard, by email (if SMTP is configured), and via webhooks (three of the four types).

---

## Alert types

| `alert_type` | Trigger | Severity | Delivery |
|---|---|---|---|
| `budget_threshold` | Budget utilization crosses 50/75/90/100% | `info` / `warning` / `critical` | In-app, email, webhook |
| `cost_anomaly` | A single request costs ≥10× the service's recent average | `critical` | In-app, email, webhook |
| `model_change` | A service switched model families with ≥2× cost change | `info` / `warning` / `critical` | In-app, email, webhook |
| `pricing_change` | Operator approves a provider pricing change | `info` / `warning` | In-app, email (no webhook) |

Every alert is inserted into the `alerts` table with a JSONB `metadata` field carrying type-specific detail.

---

## Budget threshold alerts

Source: `internal/alert/threshold.go`.

### Thresholds

```go
var thresholds = []int{50, 75, 90, 100}
```

| Utilization | Severity |
|---|---|
| ≥ 50% | `info` |
| ≥ 75% | `warning` |
| ≥ 90% | `warning` |
| ≥ 100% | `critical` |

The two `warning` levels (75% and 90%) are distinct dedup states — alerting at 75% does not preclude alerting again at 90%.

### When the check runs

`CheckAndFireThresholdAlerts` runs **inside the metering transaction** for every successful proxy request that updated at least one budget. This means the alert insert is atomic with the request log and the budget spend update — either all happen or none do.

### Dedup

Each budget tracks the highest threshold it has alerted for this period in `budgets.settings.last_alerted_threshold` (JSONB):

```go
if thr <= lastAlerted {
    continue // already alerted for this threshold this period
}
```

Once `90%` fires, `50%`, `75%`, and `90%` are skipped until the period rolls over. Only `100%` can still fire after `90%`. The setting resets to empty on period rollover.

### Metadata shape

```json
{
  "budget_id": "e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b",
  "scope_type": "team",
  "scope_id": "b3e2c8a4-1f6d-4a8c-9e0f-7d1c4b2a3e5f",
  "threshold": 75,
  "utilization": "76.4%",
  "spend": 764000000,
  "amount": 1000000000
}
```

---

## Cost anomaly alerts

Source: `internal/alert/anomaly.go`. Fires when a single request costs dramatically more than the service's recent average.

### Algorithm

```go
multiplier := float64(currentCostMicrodollars) / avgCost
if multiplier < 10.0 { return }
```

Parameters (verbatim from source):

| Parameter | Value | Description |
|---|---|---|
| Multiplier threshold | `10.0` | Current request cost must be ≥ 10× the average |
| Sample window | Last `100` requests | Rolling baseline |
| Minimum sample count | `10` requests | Below this, no alert (insufficient history) |
| Scope | Per service (`service_name` filter) | Each service has its own baseline |
| Severity | `critical` (always) | No tiered severity for anomalies |

The current request is excluded from the baseline (`WHERE id != $2::uuid`).

### When the check runs

`CheckCostAnomaly` is called **in a goroutine after the metering transaction commits**, so the new request is visible in the `requests` table. Failures (DB errors, email failures) are logged and never propagate to the proxied request.

### Metadata shape

```json
{
  "current_cost": 50000,
  "avg_cost": 4200,
  "multiplier": 11,
  "model_id": "gpt-4o",
  "service_name": "summarizer",
  "sample_count": 100,
  "request_id": "..."
}
```

---

## Model change alerts

Source: `internal/alert/model_change.go`. Fires when a service quietly switches model families with a significant cost impact.

### Algorithm

The function:

1. Pulls the last 200 requests for the tenant + service
2. Identifies the most recent request's model family (`current`)
3. Walks back until it finds a request using a different model family — that's the *transition point*
4. Computes the average cost of post-transition requests (`afterAvg`) vs. up to 50 pre-transition requests (`beforeAvg`)
5. Computes `multiplier := afterAvg / beforeAvg`

Conditions for alerting:

| Condition | Value | Notes |
|---|---|---|
| Switch must have happened within | `24 * time.Hour` | Older switches don't alert |
| Minimum pre-transition samples | `10` | `beforeRequests` size requirement |
| Cost-change trigger | `multiplier >= 2.0` (increase) **or** `multiplier < 0.5` (savings) | Between 0.5 and 2.0 = ignored |
| Dedup window | `24 hours` for same (`service`, `new_model_family`) | At most one alert per pair per day |

Severity mapping:

| Multiplier | Severity | Label |
|---|---|---|
| ≥ 10.0 | `critical` | `"<X.X>x more expensive"` |
| ≥ 2.0 | `warning` | `"<X.X>x more expensive"` |
| < 0.5 | `info` | `"<NN>% cheaper"` |

### Metadata shape

```json
{
  "service_name": "summarizer",
  "old_model_family": "gpt-4o",
  "new_model_family": "claude-opus-4-7",
  "cost_multiplier": 3.2,
  "avg_cost_before_usd": "0.0084",
  "avg_cost_after_usd":  "0.0269",
  "switched_at": "2026-05-17T15:48:00Z"
}
```

---

## Pricing change alerts

Source: `internal/alert/pricing_change.go`. Fires when an operator approves a pricing candidate (via `POST /v1/operator/pricing-candidates/{id}/approve`).

### Trigger

`NotifyPricingChange` is called from the operator's approval handler. It:

1. Queries `requests` for all tenants who used the affected `model_family` in the last 30 days
2. Computes a per-tenant cost impact estimate based on their token volume × price delta
3. Inserts a `pricing_change` alert for each affected tenant
4. Sends an email **only if** the impact exceeds $0.01 in either direction (`impactMicro < -10_000 || impactMicro > 10_000`)

### Severity

| Condition | Severity |
|---|---|
| `percentChange > 5%` | `warning` |
| Otherwise | `info` |

### Metadata shape

```json
{
  "model_id": "gpt-4o",
  "provider": "openai",
  "field_changed": "input",
  "previous_value": 2.5,
  "new_value": 2.0,
  "percent_change": -20.0,
  "impact_estimate_microdollars": -4200000
}
```

### No webhook for pricing changes

`internal/webhook/types.go` defines only `EventBudgetThreshold`, `EventCostAnomaly`, and `EventModelChange`. **There is no `pricing_change` webhook event** — these alerts surface only in-app and (above-threshold cases) by email.

---

## Delivery methods

### In-app

Every alert is inserted into the `alerts` table. The dashboard `/alerts` page polls `GET /v1/alerts` and displays unacknowledged alerts. CLI access: `tolvyn audit --last 7d` shows audit events but does not filter for alerts specifically; use the dashboard or `curl /v1/alerts`.

### Email

If the server has SMTP configured, emails go to the tenant's `email` address. SMTP environment variables (`internal/alert/email.go:27-39`):

| Variable | Required | Description |
|---|---|---|
| `TOLVYN_SMTP_HOST` | yes | SMTP server hostname |
| `TOLVYN_SMTP_PORT` | yes | `587` (STARTTLS) or `465` (implicit TLS / SMTPS) |
| `TOLVYN_SMTP_USER` | no | Username for PLAIN auth (omit for anonymous relay) |
| `TOLVYN_SMTP_PASS` | no | Password for PLAIN auth |
| `TOLVYN_ALERT_FROM_EMAIL` | yes | `From:` address |

**Port 465 uses implicit TLS** (TLS at connect time). All other ports (typically `587`) use STARTTLS via `smtp.SendMail`.

**Graceful degradation:** if `TOLVYN_SMTP_HOST`, `TOLVYN_SMTP_PORT`, or `TOLVYN_ALERT_FROM_EMAIL` is missing, `SendAlert` logs the message to stdout instead of sending — it never panics, returns an error, or blocks the caller. This lets the server run in development without SMTP configured.

Email send happens **in a goroutine** after the alert is inserted. Email failures do not roll back the alert insert or affect the proxied request.

### Webhook

For `budget_threshold`, `cost_anomaly`, and `model_change`, a webhook is dispatched to every active webhook endpoint that subscribed to the matching event type.

Webhook event types (`internal/webhook/types.go:5-9`):

```go
const (
    EventBudgetThreshold = "alert.budget_threshold"
    EventCostAnomaly     = "alert.cost_anomaly"
    EventModelChange     = "alert.model_change"
)
```

Subscribe to all three (or use the special value `"alert.all"`) when creating a webhook via `POST /v1/webhooks`.

#### Webhook envelope

Every webhook delivery uses this envelope (`WebhookEvent`):

```json
{
  "id": "delivery UUID",
  "event_type": "alert.budget_threshold",
  "tenant_id": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "timestamp": "2026-05-17T15:48:00Z",
  "data": { ... }
}
```

#### `alert.budget_threshold` data

```json
{
  "budget_id": "e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b",
  "scope_type": "team",
  "scope_id": "b3e2c8a4-1f6d-4a8c-9e0f-7d1c4b2a3e5f",
  "service_name": "",
  "threshold_pct": 75,
  "spend_usd": "764.0000",
  "budget_usd": "1000.0000",
  "period": "monthly"
}
```

#### `alert.cost_anomaly` data

```json
{
  "request_id": "...",
  "provider": "",
  "model_id": "gpt-4o",
  "cost_usd": "0.0500",
  "avg_cost_usd": "0.0042",
  "multiplier_x": 11.9,
  "service_name": "summarizer",
  "team_id": ""
}
```

**Known gap:** `provider` and `team_id` fields exist in the struct but are not populated at the dispatch site (`anomaly.go:124-131`) — they will always be empty strings in the current implementation.

#### `alert.model_change` data

```json
{
  "model_id": "claude-opus-4-7",
  "provider": "",
  "field_changed": "model_family",
  "previous_value": "gpt-4o",
  "new_value": "claude-opus-4-7",
  "percent_change": "3.2x more expensive",
  "effective_date": "2026-05-17"
}
```

`provider` is similarly not populated by the dispatch site (`model_change.go:198-205`).

#### HMAC signature

Every webhook delivery is signed with HMAC-SHA256 using the secret returned at webhook creation. The signature is sent in an `X-Tolvyn-Signature` header. Verify before processing:

```python
import hmac, hashlib
expected = hmac.new(secret.encode(), payload_bytes, hashlib.sha256).hexdigest()
assert hmac.compare_digest(expected, request.headers["X-Tolvyn-Signature"])
```

---

## Acknowledging alerts

Alerts persist with `acknowledged = false` until you mark them handled. Acknowledged alerts move out of the active list but remain in the table for audit.

### Dashboard

The Alerts page shows unacknowledged alerts. Click an alert to expand, then click **Acknowledge**.

### API

```bash
curl -X PUT https://api.tolvyn.io/v1/alerts/<alert-id>/acknowledge \
  -H "Authorization: Bearer <jwt>"
```

Also aliased as `PUT /v1/alerts/{id}/ack`.

---

## Testing alerts

`POST /v1/alerts/test` triggers a synthetic alert through every configured delivery channel (Slack, email, all subscribed webhooks). Useful for verifying SMTP and webhook configuration without waiting for a real event.

```bash
curl -X POST https://api.tolvyn.io/v1/alerts/test \
  -H "Authorization: Bearer <jwt>"
```

```json
{"sent_to": ["slack", "email"]}
```

---

## See also

- [Budgets](budgets.md) — threshold alerts originate from budget utilization
- [Webhooks](webhooks.md) — endpoint setup and signature verification
- [Kill Switch](kill-switch.md) — for unconditional blocking
- [API Reference: Alerts](../reference/api.md#alerts) · [Webhooks](../reference/api.md#webhooks)
