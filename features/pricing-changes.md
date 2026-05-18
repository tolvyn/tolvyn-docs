# Pricing Changes

Providers change AI prices without warning. OpenAI dropped GPT-4o input by 50% in May 2024. Anthropic introduced Sonnet 4.6 at a different price point than 3.5. A 20% input-price increase on a workload that processes 100M tokens/month is $200 in unexpected monthly cost — easy to miss in a dashboard, expensive at scale.

TOLVYN tracks pricing changes across all supported providers, estimates the impact on your specific usage, and notifies you when models you've used recently move. The same data flows into [Reconciliation](reconciliation.md) so historical cost calculations stay accurate even after a price change.

---

## Why pricing changes are hard to spot

Two failure modes:

1. **A model switch with no price change** is visible in [Model Change Alerts](alerts.md#model-change-alerts) — service `summarizer` quietly went from `gpt-4o` to `claude-opus-4-7`.
2. **A price change with no model switch** is invisible without explicit pricing tracking — `gpt-4o` got 30% more expensive, your spend went up 30%, but the dashboard "looks normal" because requests, tokens, and model usage are unchanged.

Pricing Changes is the answer to the second case.

---

## Three detection paths

### A. Automated scraper

A background goroutine runs once per day at **02:30 UTC** (`internal/pricing/scraper/scheduler.go:37-48`). It reads each provider's public pricing page, parses for known model identifiers, and compares against the prices currently in the `models` table.

When a price differs, a row is inserted into `pricing_candidates` with `status = 'pending'`:

| Column | Value |
|---|---|
| `provider` | `openai` / `anthropic` / `google` |
| `model_id` | The provider's official model ID |
| `field_changed` | `input`, `output`, `cached`, or `context_window` |
| `detected_value` | What the scraper found |
| `current_value` | What TOLVYN currently has |
| `source_url` | The URL the scraper read |
| `status` | `pending` until operator review |

Candidates never auto-apply. Operator review is mandatory.

The scraper can also be triggered on-demand via `POST /v1/operator/scraper/run`.

### B. Operator manual entry

When the scraper misses a change (the provider redesigned their pricing page, the model identifier doesn't match the regex, a price was announced before publication), an operator enters the change manually via `POST /v1/operator/models/changes`:

```bash
curl -X POST https://api.tolvyn.io/v1/operator/models/changes \
  -H "Authorization: Bearer <operator-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "model_id": "gpt-4o",
    "provider": "openai",
    "field_changed": "input",
    "previous_value": 2.5,
    "new_value": 2.0,
    "effective_date": "2026-06-01T00:00:00Z",
    "source_url": "https://openai.com/pricing",
    "notes": "Announced via OpenAI blog post"
  }'
```

This path **immediately notifies affected tenants** in a background goroutine (`operator.go:746-754`).

### C. Open-source pricing data

[`github.com/tolvyn/tolvyn-models`](https://github.com/tolvyn/tolvyn-models) is the open-source repository of model metadata: identifiers, families, context windows, modalities, and current prices. Apache 2.0 licensed. Community contributions accepted via pull request.

This repo serves two purposes:

- **Reference data for self-hosted TOLVYN deployments.** You can sync the repo into your own `models` table and pricing_history.
- **External validation.** Anyone can compare TOLVYN's prices against the public record in the repo.

The repo is the source of truth that the operator dashboard and the automated scraper both feed into. Pull requests with new models or detected price changes go through community review before being merged.

---

## Approval workflow

For scraper-detected candidates (path A):

1. **Scraper inserts** a `pricing_candidates` row with `status = 'pending'`.
2. **Operator dashboard** lists pending candidates at `GET /v1/operator/pricing-candidates?status=pending`.
3. **Operator reviews** the detected value against the source URL, then takes one of:
   - `POST /v1/operator/pricing-candidates/{id}/approve` — applies the change.
   - `POST /v1/operator/pricing-candidates/{id}/reject` — marks the candidate dismissed.

### What approval does

From `HandleOperatorApproveCandidate` (`operator.go:925-1047`), all inside a single transaction:

1. Marks the candidate row as `approved` with `approved_at` and `approved_by = 'operator'`.
2. Updates the `models` table: writes the new value into `pricing_input_per_mtok`, `pricing_output_per_mtok`, or `pricing_cached_per_mtok` depending on `field_changed`. Sets `pricing_updated_at = now()`.
3. **Closes the current active `pricing_history` row** for `(model_id, field_name)` by setting `effective_to = now() - interval '1 millisecond'`.
4. **Inserts a new active `pricing_history` row** with the new value, `effective_from = now()`, `effective_to = NULL`.
5. Commits the transaction.

Then, outside the transaction:

6. Inserts an audit row into `pricing_changes` for the operator dashboard's change history.
7. Calls `pricing.InvalidatePricingCache()` so subsequent requests use the new price immediately.

The `-1 millisecond` trick on `effective_to` ensures the old row closes strictly before the new row opens — no overlapping ranges, no double-pricing windows.

### Both paths notify customers

Both paths — manual operator entry and scraper-candidate approval — trigger customer notifications. `HandleOperatorApproveCandidate` calls `alert.NotifyPricingChange` in a goroutine after committing the price update (BE-01 fix), matching the manual-entry path's behavior.

---

## Customer notifications

When a pricing change is registered via the manual path (`POST /v1/operator/models/changes`), `alert.NotifyPricingChange` (`internal/alert/pricing_change.go`) runs asynchronously:

1. Queries the `requests` table for every tenant that used the affected `model_family` in the last 30 days, with their token volumes.
2. For each tenant, computes a per-tenant impact estimate:
   - For `input` changes: `tokens_input × (new_value − previous_value)`
   - For `output` changes: `tokens_output × (new_value − previous_value)`
3. Inserts a `pricing_change` row into `alerts` for each affected tenant.
4. Sends an email **only if** the absolute impact exceeds **$0.01** (`impactMicro < -10_000 || impactMicro > 10_000`).

Severity:

| Condition | Severity |
|---|---|
| `percent_change > 5%` | `warning` |
| Otherwise | `info` |

Pricing change notifications are also delivered via webhook using the `alert.pricing_change` event type. See the [Webhooks documentation](webhooks.md#alert-pricing_change) for the payload schema.

---

## Customer dashboard

The **Pricing Changes** page shows every pricing change that affected models the tenant has used recently.

Each card shows:

| Field | Source |
|---|---|
| Model | `model_id` |
| Provider | `provider` |
| What changed | `field_changed` (input / output / cached) |
| Before / After | `previous_value` / `new_value` in $/MTok |
| Percent change | Auto-computed |
| Effective date | When the new price began |
| Estimated impact | Per-tenant calculation based on the prior 30-day token volume |
| Direction | `increase` (red) / `savings` (green) / `neutral` (gray) |

Impact estimates are **approximations**: they assume your token mix in the next month matches the last 30 days. A team that scaled 10× or migrated workloads will see a different real impact.

API:

```bash
curl https://api.tolvyn.io/v1/models/changes \
  -H "Authorization: Bearer <jwt>"
```

Returns only changes for models the tenant has used. For the unfiltered history of every pricing change TOLVYN tracks, use `GET /v1/models/changes-history`.

---

## Pricing history table

`pricing_history` (migration 018) is the temporal store of every price the system has ever seen. Schema:

```sql
CREATE TABLE pricing_history (
    id             UUID PRIMARY KEY,
    model_id       VARCHAR(100) NOT NULL,
    field_name     VARCHAR(50)  NOT NULL,  -- input, output, cached
    value          NUMERIC(12,6) NOT NULL,
    effective_from TIMESTAMPTZ  NOT NULL,
    effective_to   TIMESTAMPTZ,            -- NULL = currently active
    source_url     TEXT,
    approved_by    TEXT,
    created_at     TIMESTAMPTZ DEFAULT now()
);
```

Each `(model_id, field_name)` pair has exactly one row with `effective_to IS NULL` at any moment — the current price. Older rows have `effective_to` set to the timestamp when their successor took over.

### Retroactive cost calculation

Cost-of-request at *any historical timestamp* is a temporal lookup:

```sql
SELECT value
FROM   pricing_history
WHERE  model_id  = $1
  AND  field_name = $2
  AND  effective_from <= $3
  AND  (effective_to IS NULL OR effective_to > $3)
LIMIT  1
```

Used by `internal/pricing/calculator.go` `CalculateCostAt(modelID, fieldName, timestamp)`. This is how Reconciliation can correctly compute a March 2024 cost using March 2024 prices, even after the price changed in April.

### Migration-time backfill

When migration 018 ran on existing databases, the script seeded `pricing_history` from the current snapshot in the `models` table — one `effective_from` row per `(model_id, field_name)` pair, set to `COALESCE(pricing_updated_at, now())`. So the temporal history starts at the migration date for pre-existing models. Newer models get a fresh history row on first approval.

---

## The `tolvyn-models` repo

[`github.com/tolvyn/tolvyn-models`](https://github.com/tolvyn/tolvyn-models) is Apache 2.0-licensed model metadata. Structure:

```
tolvyn-models/
├── openai/
│   ├── gpt-4o.json
│   ├── gpt-4o-mini.json
│   └── ...
├── anthropic/
│   ├── claude-opus-4-7.json
│   ├── claude-sonnet-4-6.json
│   └── ...
├── google/
│   └── ...
└── README.md
```

Each model file declares the canonical name, family, modality, context window, and current prices with source URL. The repo is the human-curated source of truth.

### How to contribute

1. Fork the repo
2. Edit the relevant `<provider>/<model>.json` file with the new price + source URL
3. Open a PR with a link to the provider's pricing announcement

Maintainers review, run automated sanity checks, and merge. The downstream `pricing_candidates` table can be re-populated from the repo by an operator.

This is how TOLVYN keeps pricing accurate at internet speed — through community contributions, validated by source URLs that anyone can audit.

---

## Operator API reference

| Endpoint | Purpose |
|---|---|
| `GET /v1/operator/pricing-candidates` | List candidates (filter by `status`) |
| `POST /v1/operator/pricing-candidates/{id}/approve` | Apply a detected change to `models` + `pricing_history` |
| `POST /v1/operator/pricing-candidates/{id}/reject` | Dismiss a candidate without applying |
| `POST /v1/operator/scraper/run` | Trigger an immediate scraper run |
| `POST /v1/operator/models/changes` | Manually create a pricing change (fires customer notifications) |
| `GET /v1/operator/models/changes` | List all pricing changes (operator view) |

All require `Authorization: Bearer <TOLVYN_OPERATOR_TOKEN>`. See [API Reference: Operator](../reference/api.md#operator-api).

---

## See also

- [Alerts: Pricing Change Alerts](alerts.md#pricing-change-alerts)
- [Reconciliation](reconciliation.md) — uses `pricing_history` for retroactive cost
- [API Reference: Models](../reference/api.md#models)
