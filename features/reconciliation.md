# Reconciliation

Reconciliation compares the cost on a provider's invoice against TOLVYN's record of proxied requests for the same period. The output is a per-model gap and a flag for any line items the provider billed for that **never appeared in TOLVYN at all** — shadow AI.

---

## What reconciliation answers

- Is the provider's invoice consistent with what TOLVYN metered?
- Are there models or services that bypassed the TOLVYN proxy?
- Where is spend going that no team owns?

The output is one reconciliation run per (provider, invoice month) pair. Each run has a per-model breakdown plus a top-line gap.

---

## Supported providers and CSV formats

The matcher joins invoice lines to TOLVYN's `requests` table by **normalized model family** (`internal/reconciliation/matcher.go:59`). Each provider parser strips dated/regional suffixes via `pricing.NormalizeModelFamily()` (e.g. `gpt-4o-2024-08-06` → `gpt-4o`).

### OpenAI

Parser: `internal/reconciliation/openai_parser.go`.

Expected CSV columns (case-insensitive, required marked with †):

| Column | Required | Notes |
|---|---|---|
| `Model` † | yes | Model identifier (any suffix accepted) |
| `Cost` † | yes | Per-line dollar cost; leading `$` is stripped |
| `Description` | no | Free text; preserved per-family |
| `Requests` | no | Request count |
| `Context Tokens` | no | Input tokens |
| `Generated Tokens` | no | Output tokens |
| `Tokens` | no | Used when `Context Tokens` + `Generated Tokens` are not both present |

Rows are aggregated by normalized model family — multiple lines for `gpt-4o-2024-08-06`, `gpt-4o-2024-05-13`, etc. all roll up to `gpt-4o`.

#### How to export from OpenAI

1. Go to [platform.openai.com/usage](https://platform.openai.com/usage)
2. Select the month you want to reconcile
3. Click **Export** → **CSV**

### Anthropic

Parser: `internal/reconciliation/anthropic_parser.go`.

Expected CSV columns:

| Column | Required | Notes |
|---|---|---|
| `Model` † | yes | Claude model identifier |
| `Cost (USD)` † | yes | Per-line dollar cost; leading `$` stripped |
| `Input Tokens` | no | Summed into `Tokens` |
| `Output Tokens` | no | Summed into `Tokens` |
| `Cache Read Tokens` | no | Summed into `Tokens` |
| `Cache Write Tokens` | no | Summed into `Tokens` |

Rows with `Cost (USD) = 0` are skipped. Token columns are summed into a single tokens total per family.

#### How to export from Anthropic

1. Go to [console.anthropic.com/settings/usage](https://console.anthropic.com/settings/usage)
2. Select the month
3. Click **Export usage** → **CSV**

### Google

Parser: `internal/reconciliation/google_parser.go`. Two formats are auto-detected based on the header row.

**Format A — AI Studio export:**

| Column | Required |
|---|---|
| `Model` † | yes |
| `Cost` † | yes |
| `Prompt Token Count` | no |
| `Candidates Token Count` | no |
| `Total Tokens` | no |

**Format B — Cloud Console billing export:**

| Column | Required |
|---|---|
| `SKU Description` † | yes — model extracted by parsing words after "gemini" |
| `Cost (USD)` † | yes |
| `Usage` | no |

The `models/` prefix from AI Studio API responses is stripped automatically.

#### How to export from Google

- **AI Studio:** [aistudio.google.com/usage](https://aistudio.google.com/usage) → Export CSV
- **Cloud Console:** Cloud Console → **Billing** → **Reports** → filter by Generative Language API SKU → export CSV

---

## Running a reconciliation

### Dashboard

**Reconciliation** page → **Upload Invoice**:

1. Drag/drop or select a CSV file
2. Pick the provider (OpenAI / Anthropic / Google)
3. Pick the invoice month (YYYY-MM)
4. Click **Run**

Results appear within seconds. The run is saved and listed on the same page for future reference.

### CLI

```bash
tolvyn reconcile upload openai-may-2026.csv \
  --month 2026-05 \
  --provider openai
```

CLI flags from `cmd_reconcile.go`:

| Flag | Default | Notes |
|---|---|---|
| `--month` | — (required) | `YYYY-MM` |
| `--provider` | `openai` | `openai`, `anthropic`, `google` |

Output:

```
RUN  2026-05 reconciliation

  Invoice total : $1240.5500
  Tolvyn total  : $1241.1000
  Gap           : -$0.5500
  Matched lines : 12  Unmatched: 0  Shadow AI: 0

MODEL          INV REQ   INV COST     TLVN REQ   TLVN COST    GAP        STATUS
gpt-4o         12180     $980.4500    12180      $980.7000    -$0.2500   matched
...

Run ID: a8d2c4e6-...
```

### API

`POST /v1/reconciliation/upload` (multipart):

```bash
curl -X POST https://api.tolvyn.io/v1/reconciliation/upload \
  -H "Authorization: Bearer <jwt>" \
  -F "file=@openai-may-2026.csv" \
  -F "provider=openai" \
  -F "invoice_month=2026-05"
```

---

## Understanding results

### Gap formula

From `matcher.go:106`:

```
Gap (USD) = Invoice cost (USD) − Tolvyn cost (USD)
```

| Sign | Meaning |
|---|---|
| `Gap > 0` | Invoice is higher than what TOLVYN saw. Some spend bypassed the proxy (or model families don't match — see Limitations) |
| `Gap ≈ 0` | All provider spend went through TOLVYN |
| `Gap < 0` | TOLVYN tracked more spend than the invoice. Unusual — possible causes are listed below |

A small negative gap is normal: provider invoices often round per-line costs to the nearest cent, while TOLVYN records exact microdollar values. A large negative gap usually indicates a pricing mismatch (TOLVYN's cached price is out of date) — file a [pricing change](../features/pricing-changes.md) request.

### Per-line status

Each model family in the invoice gets one of three statuses:

| Status | Meaning |
|---|---|
| `matched` | TOLVYN has requests for this model family in this month |
| `unmatched` | Invoice has the line, but TOLVYN has zero matching requests *and* zero cost (rare — usually one-time billing adjustments) |
| `SHADOW AI` | Invoice cost > $0, TOLVYN cost = $0 — the provider was billed for usage that never touched the proxy |

---

## Shadow AI detection

From `matcher.go:107-110`:

```go
if lr.InvoiceCostUSD > 0 && lr.TolvynCostUSD == 0 {
    lr.ShadowAI = true
    result.ShadowAILines++
}
```

A model family is flagged as **Shadow AI** when the provider billed for it but TOLVYN saw zero requests for that family in the same month for this tenant. Concrete causes:

- A team is calling the provider directly without using a TOLVYN key
- A leaked OpenAI/Anthropic key is being used by someone outside your organization
- An older system that pre-dates your TOLVYN deployment still has the raw provider key
- A new service shipped without going through the TOLVYN proxy

Investigating: open the provider's dashboard (OpenAI Usage, Anthropic Console, Cloud Console) and look for API keys with traffic in the same month that aren't your TOLVYN-stored provider key.

---

## Gap thresholds — what's concerning

There is **no automatic threshold** in code. Use the following as a rule of thumb:

| Gap | Interpretation |
|---|---|
| < 0.5% | Normal rounding |
| 0.5–2% | Worth investigating — pricing drift or a small bypass |
| 2–10% | Notable shadow AI or pricing mismatch — track it down |
| > 10% | Significant bypass — somebody is calling providers outside your governance |

Run a fresh reconciliation each month when the invoice arrives. Trends matter more than absolute gaps: a 1% gap that grows monthly is a leak.

---

## Past runs

### Dashboard

**Reconciliation** page shows every prior run with month, provider, gap, and shadow AI count. Click any row to expand the line-item breakdown.

### CLI

```bash
tolvyn reconcile list
```

```
MONTH    PROVIDER   INV TOTAL    GAP        SHADOW AI   FILE
2026-05  openai     $1240.5500   -$0.5500   0           openai-may-2026.csv
2026-04  openai     $1102.1000   balanced   0           openai-apr-2026.csv
```

### API

`GET /v1/reconciliation` (list), `GET /v1/reconciliation/{id}` (detail).

---

## Deleting a run

`DELETE /v1/reconciliation/{id}` — also `tolvyn reconcile delete <run-id>`. Both prompt for confirmation. Past runs do not affect ongoing metering; delete them only to reduce clutter.

---

## Limitations

### Match by model family, not exact model ID

The matcher joins on `model_family`, not `model_id` (`matcher.go:59`). This means:

- Invoice lines `gpt-4o-2024-08-06` and `gpt-4o-2024-05-13` collapse into one tenant-side row for `gpt-4o`
- TOLVYN requests with `model_family = gpt-4o` but a different exact `model_id` still match

This is intentional — providers change date suffixes between billing cycles and matching exactly would create noise. But it means a tenant who switched between two minor versions of the same family within a month won't see a per-version breakdown in the reconciliation report.

### Model normalization edge cases

Reconciliation depends on `pricing.NormalizeModelFamily()` producing the same family name for both the invoice CSV row and the tenant's request rows. Anomalies happen when:

- A provider releases a new model not yet in TOLVYN's normalization regex (`gpt-5-2026-06-15` would normalize differently from any historical request)
- A provider changes the format of a model identifier (e.g. drops the date suffix entirely)

If you see `unmatched` lines for models you definitely used, file a [pricing change](../features/pricing-changes.md) or check the model normalization in `internal/pricing/models.go`.

### One reconciliation per (provider, month)

Uploading a second CSV for the same provider+month creates a separate run; the older run is not replaced. Use the dashboard or API to delete prior runs if you re-uploaded a corrected invoice.

### CSV-only — no API ingestion of provider invoices

TOLVYN does not pull invoices from provider APIs automatically. You upload CSVs manually each month. This is a deliberate choice (provider invoice APIs are minimal or non-existent) but means a missed upload = no reconciliation that month.

---

## See also

- [Integration Modes](../integration-modes.md) — why proxy-mode requests are what TOLVYN sees
- [Pricing Changes](pricing-changes.md) — how provider price changes affect reconciliation accuracy
- [API Reference: Reconciliation](../reference/api.md#reconciliation)
- [CLI Reference: `tolvyn reconcile`](../reference/cli.md#tolvyn-reconcile)
