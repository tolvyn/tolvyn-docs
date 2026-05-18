# Savings Dashboard

The savings dashboard surfaces cost-reduction opportunities derived from your last 30 days of TOLVYN-metered traffic. Analysis runs nightly. **TOLVYN never auto-routes or modifies requests** — every finding is a suggestion you act on yourself.

---

## Overview

| Property | Value |
|---|---|
| Analysis schedule | Daily at **02:00 UTC** |
| Window | Rolling **30 days** ending at run time |
| Output | Findings inserted into `savings_findings`, surfaced on the dashboard |
| Action taken | None — TOLVYN does not change your routing |
| Atomic replace | Yes — non-dismissed findings deleted before each rerun |
| Dismissed findings | Preserved forever — survive subsequent runs |

Source: `internal/savings/analyzer.go` and `internal/savings/rules.go`.

---

## When analysis runs

A background goroutine started at server boot (`analyzer.go:111-127`):

```go
now := time.Now().UTC()
next := time.Date(now.Year(), now.Month(), now.Day(), 2, 0, 0, 0, time.UTC)
if !next.After(now) {
    next = next.Add(24 * time.Hour)
}
log.Printf("savings: next analysis scheduled for %s", next.Format(time.RFC3339))
time.Sleep(time.Until(next))
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Minute)
RunAllTenants(ctx, db)
```

The job runs at **02:00 UTC** every day with a **30-minute timeout**. Each active tenant is analyzed sequentially.

To trigger an immediate recompute outside the schedule, operators can call `POST /v1/operator/savings/recompute`.

---

## Four savings rules

All four run in order: `analyzeSmallTokenRequests`, `analyzeDuplicatePrompts`, `analyzeUnderutilizedCache`, `analyzeIdleModels` (`rules.go:13-18`). Findings from all rules are accumulated and inserted atomically.

### 1. `small_token_requests` — downgrade expensive models on short prompts

Source: `analyzeSmallTokenRequests`.

**Trigger criteria:**

| Criterion | Value |
|---|---|
| Token count (input + output) | < 500 |
| Token count | > 0 (excludes failed requests) |
| Request count for `(model_family, service_name)` pair | > 100 |
| Estimated monthly savings | ≥ $1.00 |
| Model must be in the downgrade map | see below |

**Hardcoded downgrade map** (`rules.go:28-39`):

| Expensive model | Suggested cheaper alternative | Savings ratio |
|---|---|---|
| `gpt-4o` | `gpt-4o-mini` | 93% |
| `gpt-4-turbo` | `gpt-4o-mini` | 90% |
| `gpt-4` | `gpt-4o-mini` | 90% |
| `claude-3-5-sonnet` | `claude-haiku-4-5` | 80% |
| `claude-sonnet-4-6` | `claude-haiku-4-5` | 75% |
| `claude-3-opus` | `claude-haiku-4-5` | 95% |
| `claude-3-sonnet` | `claude-haiku-4-5` | 70% |
| `claude-opus-4-6` | `claude-sonnet-4-6` | 70% |
| `claude-opus-4-7` | `claude-sonnet-4-6` | 70% |
| `o1` | `o3-mini` | 85% |
| `o1-pro` | `o1-mini` | 95% |
| `gemini-2.5-pro` | `gemini-2.5-flash` | 80% |
| `gemini-2.0-ultra` | `gemini-2.0-flash` | 85% |

If your most-used expensive model isn't in this map (e.g. `gpt-5` when released), it won't trigger a downgrade finding — the map needs updating in source.

Example finding text:

```
backend in this period used fewer than 500 tokens (avg 184 tokens).
claude-haiku-4-5 would cost ~$12.40 instead of ~$49.60 — saving ~$37.20.
```

### 2. `duplicate_prompts` — cache redundant requests

Source: `analyzeDuplicatePrompts`.

**Trigger criteria:**

| Criterion | Value |
|---|---|
| Detection method | `request_hash` column equality |
| Duplicate percentage | ≥ 5.0% of total requests |
| `request_hash` must be non-NULL | Required |

The `request_hash` column is populated by the proxy with a normalized hash of the request body (system prompt + messages + model + key parameters). A "duplicate" is two or more requests with the same hash.

**Savings estimate:** the full cost of every redundant request beyond the first per hash (`SUM(cost_microdollars)` of all but one duplicate per hash group).

Example finding text:

```
8.4% of requests this month are duplicate prompts (1247 redundant calls).
A prompt caching or request deduplication layer would save ~$42.80.
```

**Note on accuracy:** caching would only save the redundant portion if your application can tolerate cached responses (no temperature, no time-sensitive content). TOLVYN cannot tell which duplicates are intentional retries vs. avoidable cache misses.

### 3. `underutilized_cache` — enable provider prompt caching

Source: `analyzeUnderutilizedCache`.

**Trigger criteria:**

| Criterion | Value |
|---|---|
| Providers checked | `openai`, `anthropic` only |
| Minimum input tokens in window | ≥ 100,000 |
| Cache hit rate threshold | < 5% (`cached / input < 0.05`) |
| Estimated monthly savings | ≥ $0.50 |

**Savings estimate** (`rules.go:216-217`):

```go
estimatedInputCost := int64(float64(totalCost) * 0.65)
savings := int64(float64(estimatedInputCost) * 0.30 * 0.90)
```

Assumes:

- 65% of total cost is input cost (typical ratio for chat workloads)
- 30% of input tokens are cacheable (the system prompt, few-shot examples, etc.)
- 90% discount on cached tokens (standard provider rate)

These constants are not parameterized — they are sensible defaults for typical chat applications but won't match every workload.

Example finding text:

```
You're paying full input price for repeated prompts on anthropic.
Current cache hit rate: 0.8% (target ≥5%).
Enabling prompt caching could save ~$84.20 (estimated assuming 30% of tokens cacheable).
```

Google is excluded from this rule — its prompt caching mechanism is different and TOLVYN does not yet model it.

### 4. `idle_model` — surface unused model allocations

Source: `analyzeIdleModels`.

**Trigger criteria:**

| Criterion | Value |
|---|---|
| Model had activity in the last 90 days | Required |
| Model has had 0 requests in the last 30 days | Required |
| Savings amount | `$0.00` |

Idle-model findings carry **no dollar estimate** — the previous budget-share proxy metric was removed (BE-15) because it presented a fabricated number. The finding is still surfaced with the model name and idle period; only the misleading savings figure is gone.

Example finding text:

```
gpt-4o-2024-05-13 has had 0 requests in the last 30 days (last used 47 days ago,
2104 total historical requests).
If you have active budgets tied to services that relied on this model,
consider removing them to reduce management overhead.
```

---

## Dismissing findings

Findings have a `dismissed` boolean. Once dismissed, a finding:

- Disappears from the active dashboard
- Is **not deleted** — it stays in the `savings_findings` table
- **Survives subsequent nightly runs** — even if the underlying condition (e.g. low cache hit rate) persists, dismissed findings are not regenerated

### Dashboard

**Savings** page → click a finding → **Dismiss**. The finding moves to the dismissed list (visible via a filter toggle).

### API

`POST /v1/savings/{id}/dismiss` — sets `dismissed = true` and `dismissed_at = now()`.

```bash
curl -X POST https://api.tolvyn.io/v1/savings/<finding-id>/dismiss \
  -H "Authorization: Bearer <jwt>"
```

To un-dismiss, currently you must wait for the next nightly rerun to regenerate a non-dismissed copy — there is no `un-dismiss` endpoint. (Dismissing is intended to be one-way.)

---

## Triggering a recompute

Tenant-facing endpoints cannot trigger a rerun on demand. Operators can:

```bash
curl -X POST https://api.tolvyn.io/v1/operator/savings/recompute \
  -H "Authorization: Bearer <operator-token>"
```

This runs `RunAllTenants` immediately. Useful after a major code change in `rules.go` or after a backfill of `request_hash`.

---

## Atomic-replace behavior

At the start of each rerun, the analyzer (`analyzer.go:51-56`):

```sql
DELETE FROM savings_findings
WHERE tenant_id = $1 AND dismissed = false
```

Then it inserts fresh findings from all four rules. This means:

- A finding that no longer applies (e.g. you switched models and the small-token-request opportunity is gone) **disappears** at the next run
- A finding that **still applies** gets a **new row** with a fresh timestamp — IDs change between runs
- **Dismissed findings persist** — the `DELETE` excludes them — so dismissing a finding once dismisses it forever

If you need stable IDs for tracking outside TOLVYN, use the `(finding_type, model_id, service_name)` tuple as the natural key.

---

## Limitations

### Estimates are approximate

Every dollar figure in a savings finding is an **estimate**. Sources of inaccuracy:

- **Downgrade ratios are hardcoded** per family (not based on your actual prompt complexity)
- **Cache savings assume** 30% of input tokens are cacheable and 65% of cost is input — true for some workloads, off by 2× for others
- **Pricing drift** — savings are computed at the prices in effect during the 30-day window. A pricing change mid-window can skew estimates
- **No prompt analysis** — TOLVYN doesn't look at prompt content, so it can't tell you whether a cheaper model would actually produce acceptable output

Treat findings as a hit-list to investigate, not a guaranteed savings amount.

### Rolling 30-day window only

The window is always "last 30 days". You can't run analysis for a custom date range. To get last-quarter findings, wait until the analyzer has run for 30 days against the period of interest.

### Hardcoded model lists

The downgrade map covers 13 model families. Models released after the last update still need to be added in source — file an issue if you see expensive usage of an unmapped model.

---

## See also

- [Budgets](budgets.md) — set hard caps on the spend savings analysis surfaces
- [Pricing Changes](pricing-changes.md) — what triggers price drift
- [API Reference: Savings](../reference/api.md#savings)
