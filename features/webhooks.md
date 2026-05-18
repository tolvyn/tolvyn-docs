# Webhooks

Webhooks deliver TOLVYN alert events to your HTTPS endpoint via signed POST requests. Use them for incident pipelines, ChatOps, or storing event history in your own system.

Up to **10 active endpoints per tenant**.

---

## When to use webhooks vs other channels

| | In-app | Email | Slack | Webhook |
|---|---|---|---|---|
| Pull or push | Pull (poll) | Push | Push | Push |
| Best for | Browsing | Stakeholder notify | Team chat | Automated downstream action |
| Reliable retries | n/a | SMTP retry | Slack retries 3x | TOLVYN retries 5x with backoff |
| Inspectable history | Yes | Mail logs | Slack channel | TOLVYN delivery log |

Use webhooks when you need a downstream system (PagerDuty, your own database, an internal pipeline) to receive every alert without polling.

---

## Creating a webhook

### Dashboard

**Webhooks → Add Endpoint**:

1. Enter a `https://` URL (HTTP is rejected at validation)
2. Pick one or more event types, or `alert.all` for everything
3. Click **Save**
4. **Copy the secret immediately** — it is shown once, then masked

### API

`POST /v1/webhooks`:

```bash
curl -X POST https://api.tolvyn.io/v1/webhooks \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://hooks.example.com/tolvyn",
    "event_types": ["alert.budget_threshold", "alert.cost_anomaly"]
  }'
```

Request fields (from `HandleCreateWebhook` in `internal/api/client.go`):

| Field | Type | Required |
|---|---|---|
| `url` | string | yes — must start with `https://` |
| `event_types` | string[] | yes — at least one |

Response (`201 Created`):

```json
{
  "id": "...",
  "url": "https://hooks.example.com/tolvyn",
  "event_types": ["alert.budget_threshold", "alert.cost_anomaly"],
  "active": true,
  "created_at": "2026-05-17T14:00:00Z",
  "updated_at": "2026-05-17T14:00:00Z",
  "secret": "wh_a1b2c3d4..."
}
```

The `secret` field is returned **only on creation**. Store it now — neither the dashboard nor the API will show it again.

Errors:

| Code | When |
|---|---|
| `400 invalid_url` | URL doesn't start with `https://` |
| `400 invalid_event_types` | `event_types` array is empty |
| `400 invalid_event_type` | Unknown event type passed |
| `422 max_webhooks` | Tenant already has 10 endpoints |

---

## Event types

Source: `internal/webhook/types.go`. Four real event types plus one synthetic test event:

| Event type | When dispatched |
|---|---|
| `alert.budget_threshold` | Budget utilization crosses 50/75/90/100% |
| `alert.cost_anomaly` | Single request costs ≥10× recent service average |
| `alert.model_change` | Service switches model family with ≥2× cost change |
| `alert.pricing_change` | A provider's published price for a model the tenant uses changes |
| `alert.all` | Wildcard subscription — receives all of the above |
| `webhook.test` | Sent by `POST /v1/webhooks/{id}/test` only — not a real alert |

A tenant subscribing to `alert.all` will be notified of any new event types added later. No code change required.

---

## Payload schema

Every delivery uses the same envelope:

```json
{
  "id": "<delivery UUID>",
  "event_type": "alert.budget_threshold",
  "tenant_id": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "timestamp": "2026-05-17T15:48:00Z",
  "data": { ... type-specific fields ... }
}
```

### `alert.budget_threshold`

Source: `BudgetThresholdData` struct.

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

### `alert.cost_anomaly`

Source: `CostAnomalyData` struct.

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

As of the BE-04 fix, the `provider` and `team_id` fields are populated at the dispatch site (`proxy.go meterAndRecord`).

### `alert.model_change`

Source: `ModelChangeData` struct.

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

As of the BE-05 fix, the `provider` field is populated at the dispatch site (`model_change.go`).

### `alert.pricing_change`

Source: `PricingChangeData` struct. Fires when a provider's published price for a model the tenant has used in the last 30 days changes.

```json
{
  "model_id": "gpt-4o",
  "provider": "openai",
  "field_changed": "input",
  "previous_value": 2.5,
  "new_value": 2.0,
  "percent_change": -20.0,
  "impact_estimate_usd": "4.2000",
  "effective_date": "2026-06-01"
}
```

| Field | Type | Description |
|---|---|---|
| `model_id` | string | Provider's model identifier |
| `provider` | string | `openai` / `anthropic` / `google` |
| `field_changed` | string | `input`, `output`, or `cached` |
| `previous_value` | number | Old price (USD per million tokens) |
| `new_value` | number | New price (USD per million tokens) |
| `percent_change` | number | `(new - prev) / prev * 100`; negative means price dropped |
| `impact_estimate_usd` | string | Per-tenant monthly impact estimate based on last 30 days' usage; negative = savings |
| `effective_date` | string | ISO 8601 date the new price takes effect |

### `webhook.test`

Triggered only by the test endpoint. Minimal payload:

```json
{
  "id": "...",
  "event_type": "webhook.test",
  "timestamp": "2026-05-17T15:48:00Z",
  "data": { "message": "This is a test delivery from TOLVYN" }
}
```

---

## Headers

Three headers are set on every delivery (`dispatcher.go:215-217`):

| Header | Value |
|---|---|
| `X-Tolvyn-Signature` | `sha256=<hex>` — HMAC-SHA256 of raw payload bytes, keyed with your endpoint secret |
| `X-Tolvyn-Event` | The event type (e.g. `alert.budget_threshold`) |
| `X-Tolvyn-Delivery` | UUID of this specific delivery attempt |

`Content-Type: application/json` is also set.

---

## Signature verification

The signature is `sha256=<hex>` where `<hex>` is the lowercase hex encoding of `HMAC-SHA256(secret, payload_bytes)`. Use the **raw request body bytes**, not the parsed JSON — re-serializing alters whitespace and breaks the signature.

### Python

```python
import hmac, hashlib
from flask import Flask, request, abort

app = Flask(__name__)
SECRET = "wh_a1b2c3d4..."   # from webhook creation

@app.post("/tolvyn-webhook")
def handle():
    sig_header = request.headers.get("X-Tolvyn-Signature", "")
    if not sig_header.startswith("sha256="):
        abort(401)
    expected = "sha256=" + hmac.new(
        SECRET.encode(),
        request.get_data(),   # raw bytes
        hashlib.sha256,
    ).hexdigest()
    if not hmac.compare_digest(sig_header, expected):
        abort(401)

    event = request.get_json()
    print(f"received {event['event_type']}: {event['data']}")
    return "", 200
```

### Node.js

```javascript
import express from "express";
import crypto from "crypto";

const app = express();
const SECRET = "wh_a1b2c3d4...";

app.post("/tolvyn-webhook",
  express.raw({ type: "application/json" }),   // keep raw bytes
  (req, res) => {
    const sigHeader = req.header("X-Tolvyn-Signature") || "";
    if (!sigHeader.startsWith("sha256=")) return res.sendStatus(401);

    const expected = "sha256=" + crypto
      .createHmac("sha256", SECRET)
      .update(req.body)
      .digest("hex");

    const valid =
      sigHeader.length === expected.length &&
      crypto.timingSafeEqual(Buffer.from(sigHeader), Buffer.from(expected));

    if (!valid) return res.sendStatus(401);

    const event = JSON.parse(req.body.toString());
    console.log(`received ${event.event_type}:`, event.data);
    res.sendStatus(200);
  },
);
```

Use a **constant-time comparison** (`hmac.compare_digest` / `crypto.timingSafeEqual`) to avoid timing attacks. Do not use `==` or `===`.

---

## Retry policy

TOLVYN retries failed deliveries up to **5 attempts total** with the following intervals (from `dispatcher.go` `retryDelay()`):

| Failed attempt | Wait before next attempt |
|---|---|
| 1st | 1 minute |
| 2nd | 5 minutes |
| 3rd | 30 minutes |
| 4th | 2 hours |
| 5th | (no more retries — delivery marked permanently failed) |

Each interval has **±10% random jitter** so a fleet of tenants doesn't all retry at the same wall-clock instant after a receiver recovers.

After 5 attempts the delivery is marked as `last_status = 'failed'` on the endpoint and `next_retry_at` is set to `NULL`. No further attempts.

A delivery is treated as **failed** when the HTTP response status is outside `200-299`, the connection times out (5-second client timeout), or the request errors before getting a response.

### Retry tick

A background goroutine started by `main.go` calls `RetryPendingDeliveries(db)` every 5 minutes. It scans for deliveries where:

```sql
delivered_at IS NULL
AND attempt_count < 5
AND next_retry_at <= now()
```

So the actual time between attempts is *retry-delay rounded up to the next 5-minute tick*. A 1-minute retry can land up to 4 minutes late; a 2-hour retry can land up to 4 minutes late.

### Idempotency

TOLVYN claims each delivery row at the start of a retry pass:

```sql
UPDATE webhook_deliveries SET next_retry_at = NULL
WHERE id = $1 AND next_retry_at IS NOT NULL
```

If two ticks race (unusual but possible), only one wins the row. The losing tick exits without re-delivering. Each event delivers **at-least-once** — your endpoint should be idempotent on `X-Tolvyn-Delivery`.

---

## Delivery log

Every attempt is recorded in `webhook_deliveries` with:

- `status_code` (HTTP status from your endpoint, or `0` if connection error)
- `attempt_count` (how many times this delivery has been tried)
- `next_retry_at` (when the next attempt will happen, `NULL` if delivered or permanently failed)
- `delivered_at` (timestamp of the successful 2xx, `NULL` if not yet)
- `error_message` (error string on failure)

### Dashboard

**Webhooks** page → click an endpoint → **Recent deliveries** tab. Shows the last 50 attempts with timestamps, status codes, and the raw payload.

### API

```bash
curl https://api.tolvyn.io/v1/webhooks/<webhook-id>/deliveries \
  -H "Authorization: Bearer <jwt>"
```

---

## Testing a webhook

Sends a synthetic `webhook.test` event through your real endpoint. Does **not** generate or consume a real alert; useful for verifying URL reachability and signature validation without waiting for live traffic.

### Dashboard

**Webhooks** page → click the endpoint → **Send Test**.

### API

```bash
curl -X POST https://api.tolvyn.io/v1/webhooks/<webhook-id>/test \
  -H "Authorization: Bearer <jwt>"
```

Returns the HTTP status your endpoint replied with:

```json
{"status_code": 200, "delivery_id": "..."}
```

The test still creates a `webhook_deliveries` row, so you can inspect it in the delivery log.

---

## Updating and deleting

### Update

`PUT /v1/webhooks/{id}` — change `url`, `event_types`, or `active`:

```bash
curl -X PUT https://api.tolvyn.io/v1/webhooks/<webhook-id> \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"event_types": ["alert.all"], "active": true}'
```

Updating the URL or event types does **not** rotate the secret. The signing key remains the same.

### Delete

`DELETE /v1/webhooks/{id}` — returns `204 No Content`. Pending retries for this endpoint will not be attempted further.

---

## Rotating the secret

There is no in-place secret rotation. To rotate:

1. Create a new webhook endpoint with the same URL and event types
2. Update your endpoint code to accept either the old or new signature (a dual-verification window)
3. Delete the old webhook
4. After traffic has shifted, drop the old-secret branch from your code

Plan rotation as a deliberate operation; an event in flight when you delete the old endpoint will not be delivered.

---

## Limits and quotas

| Limit | Value |
|---|---|
| Active endpoints per tenant | 10 |
| Max attempts per delivery | 5 |
| Delivery timeout | 5 seconds |
| Event types currently supported | 4 (`alert.budget_threshold`, `alert.cost_anomaly`, `alert.model_change`, `alert.pricing_change`) |

---

## See also

- [Alerts](alerts.md) — what generates webhook events
- [Budgets](budgets.md) · [Kill Switch](kill-switch.md)
- [API Reference: Webhooks](../reference/api.md#webhooks)
