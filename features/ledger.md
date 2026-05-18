# Immutable Ledger

The ledger is TOLVYN's answer to the question: **how do I prove what my AI costs were on a specific date, to an auditor who doesn't trust my dashboard?**

Every request that flows through TOLVYN is recorded in a SHA-256 hash-chained, HMAC-signed ledger. The chain is verifiable end-to-end at any time. Tampering with any historical record breaks the chain at that point — detectably, without external consensus, in milliseconds.

This is what separates TOLVYN from "AI usage analytics." Analytics dashboards can be edited. Audit ledgers cannot.

---

## Why hash-chaining matters for AI governance

Three categories of question that traditional logs cannot answer:

- *"Did your finance team alter the spend numbers before the audit?"* — Application logs are mutable. They were designed for debugging, not financial evidence.
- *"Prove your March AI cost is exactly the value you billed to the customer."* — Aggregating from mutable logs is a sticky note, not an audit trail.
- *"Show me the exact request that was blocked when our budget hit 100%."* — The ledger records the enforcement decision (`allow` / `block`) alongside the cost.

The TOLVYN ledger is built on the premise that AI spend has crossed the threshold where standard observability stops being adequate. When your AI bill is bigger than your CI/CD bill, you need financial-grade evidence.

---

## How it works

### Genesis block

Each tenant's chain begins with a synthetic predecessor: `previous_hash` of the **very first record** is `SHA-256(tenantID)` rendered as hex. The genesis hash anchors the chain — verifying from sequence 1 requires re-deriving it (`ledger.go:81-84`):

```go
func genesisHash(tenantID string) string {
    h := sha256.Sum256([]byte(tenantID))
    return hex.EncodeToString(h[:])
}
```

Two tenants that happened to start with the same first proxied request would still produce different chains, because the genesis hashes differ.

### Record structure

Every ledger row's `record_hash` is `SHA-256` of a **canonical JSON serialization** of this struct (`ledger.go:62-77`):

```json
{
  "tenant_id": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "sequence_number": 1843,
  "previous_hash": "<hex of previous record_hash>",
  "request_id": "<UUID of the metered request>",
  "cost_microdollars": 8400,
  "hierarchy_path": "/backend/summarizer",
  "provider": "openai",
  "model_id": "gpt-4o-2024-08-06",
  "model_family": "gpt-4o",
  "modality": "chat",
  "tokens_input": 120,
  "tokens_output": 35,
  "budget_status": "ok",
  "enforcement_action": "allow"
}
```

The JSON struct has fields in **declaration order**, so `json.Marshal` produces deterministic output — verification can always re-derive the same hash from the stored payload.

### Hash chain

Each record stores the hash of the previous one. The chain looks like:

```
genesisHash(tenant_id) → record[1].record_hash → record[2].record_hash → ... → record[N].record_hash
                          ↑                       ↑
                          stored as record[2].previous_hash
                          stored as record[3].previous_hash
```

Any tampering with record `K` produces a new `record_hash[K]`. That breaks the chain because `record[K+1].previous_hash` no longer matches. Verification immediately flags the seam.

### HMAC signing

`record_hash` is then signed with `HMAC-SHA256` keyed on `TOLVYN_HMAC_SECRET` (`ledger.go:97-101`):

```go
mac := hmac.New(sha256.New, secret)
mac.Write([]byte(recordHash))
hmacSignature := hex.EncodeToString(mac.Sum(nil))
```

The HMAC adds a second layer of defense: even an attacker with direct database write access cannot forge a valid record without also knowing the secret. The SHA-256 chain catches accidental or external tampering; the HMAC catches insider tampering. Different threat models, both addressed.

`TOLVYN_HMAC_SECRET` must be at least 32 bytes. The server refuses to start without it set.

### Advisory lock for sequence integrity

Sequence numbers must be gap-free per tenant for verification to work. Concurrent `AppendRecord` calls for the same tenant could otherwise race and produce duplicates or gaps.

The fix: a per-tenant PostgreSQL advisory lock (`ledger.go:127-130`):

```go
lockKey := advisoryLockKey(tenantID)  // first 8 bytes of SHA-256(tenantID) as int64
if _, err := tx.ExecContext(ctx, "SELECT pg_advisory_xact_lock($1)", lockKey); err != nil {
    return err
}
```

`pg_advisory_xact_lock` serializes all `AppendRecord` calls for the same tenant **within the scope of each call's transaction**. The lock releases automatically on transaction commit or rollback. Across tenants, no contention — each tenant's chain has its own lock key.

This means a single tenant making 1,000 concurrent requests will have those 1,000 ledger appends serialized; cross-tenant traffic is unaffected.

---

## What goes in every record

From `recordPayload`:

| Field | Meaning |
|---|---|
| `tenant_id` | Owner — RLS enforces visibility |
| `sequence_number` | Gap-free integer per tenant, starts at 1 |
| `previous_hash` | Hash of the prior record (or `genesisHash` for record 1) |
| `request_id` | UUID linking to the `requests` table |
| `cost_microdollars` | Exact cost as int64 — never a float |
| `hierarchy_path` | `/team/service[/agent]` for forensic drill-down |
| `provider` | `openai`, `anthropic`, `google` |
| `model_id` | Exact model used (e.g. `gpt-4o-2024-08-06`) |
| `model_family` | Normalized family (e.g. `gpt-4o`) |
| `modality` | `chat`, `embedding`, `audio`, `image` |
| `tokens_input` / `tokens_output` | Provider-reported counts |
| `budget_status` | `ok`, `warning`, `exceeded` at request time |
| `enforcement_action` | `allow` or `block` — proves whether the proxy let it through |

The record includes both the **cost** and the **enforcement decision**. A blocked request appears in the ledger with `cost_microdollars = 0` and `enforcement_action = "block"`, so even prevented spend leaves a footprint.

---

## Verification

`GET /v1/ledger/verify` walks the chain and re-derives every hash and HMAC. Parameters:

| Param | Default | Description |
|---|---|---|
| `from_seq` | `1` | Lowest sequence to check |
| `to_seq` | current max | Highest sequence to check |

### Successful verification

```bash
curl https://api.tolvyn.io/v1/ledger/verify \
  -H "Authorization: Bearer <jwt>"
```

```json
{
  "valid": true,
  "records_checked": 18432
}
```

### Failed verification

```json
{
  "valid": false,
  "records_checked": 1042,
  "first_invalid_sequence": 1043,
  "reason": "seq 1043: record_hash mismatch (stored a1b2c3..., derived d4e5f6...)"
}
```

`first_invalid_sequence` is the exact sequence where the chain broke. `reason` gives one of:

- `sequence gap: expected N, got M` — a record is missing
- `seq N: previous_hash mismatch` — record `N` claims a prior hash that doesn't match record `N-1`'s `record_hash`
- `seq N: record_hash mismatch` — re-deriving from the stored payload produces a different hash (the payload was edited after insertion)
- `seq N: HMAC mismatch` — record_hash is consistent but the HMAC doesn't verify (signing key changed, or HMAC was forged with the wrong secret)

Anything other than `valid: true` means the chain has been compromised at or after `first_invalid_sequence`. Investigate immediately — at minimum, restore from a verified backup and rotate `TOLVYN_HMAC_SECRET`.

---

## Exporting the ledger

`GET /v1/ledger?format=csv` streams every column to a CSV file. Includes `record_hash`, `previous_hash`, and `hmac_signature` so the export can be verified **offline** against the server's `TOLVYN_HMAC_SECRET` without ever talking to TOLVYN again.

```bash
curl "https://api.tolvyn.io/v1/ledger?format=csv&from=2026-05-01T00:00:00Z&to=2026-06-01T00:00:00Z" \
  -H "Authorization: Bearer <jwt>" \
  > tolvyn-ledger-2026-05.csv
```

Columns in the export:

```
sequence_number, created_at, request_id, provider, model_id, model_family,
cost_usd, hierarchy_path, budget_status, enforcement_action,
tokens_input, tokens_output, record_hash, previous_hash, hmac_signature
```

For auditors: hand them the CSV + the HMAC secret in a separate channel. They can run the verification themselves with any SHA-256 / HMAC library. They never need to trust TOLVYN's verify endpoint.

---

## What ledger integrity proves — and what it does NOT prove

### What it proves

- Every cost figure in the ledger is the value TOLVYN saw when the request was metered. It was not changed retroactively.
- The sequence of events is unbroken — no requests were retroactively inserted or deleted.
- Budget enforcement decisions (`allow` / `block`) are recorded contemporaneously with the request.
- The total cost over any time range can be computed from the verified ledger and is **mathematically equivalent** to the sum of `cost_microdollars` across those records.

### What it does NOT prove

- **The content of prompts or responses.** TOLVYN does not store prompt or response text. The ledger proves what model was called and what it cost, not what was said.
- **That the provider's invoice matches.** TOLVYN's view is the proxy's view. Requests that bypassed the proxy never reach the ledger. Use [Reconciliation](reconciliation.md) to spot-check against provider invoices.
- **The fairness of provider pricing.** The ledger records what TOLVYN was told the cost was at the time of metering, using prices in effect at that moment. It does not validate whether the provider charged correctly.
- **That `TOLVYN_HMAC_SECRET` was never leaked.** Rotation invalidates the HMAC of all prior records (they no longer verify with the new secret). Don't rotate the HMAC secret without an explicit re-signing plan, or you destroy your own audit trail.

---

## Compliance use cases

### Audit evidence

For an external auditor (SOC 2, ISO 27001, financial audit): export the ledger CSV for the audit period, provide the HMAC secret over a separate channel, point them at the open-source verification logic in `internal/ledger/ledger.go`. The auditor produces independent verification without trusting your dashboard.

### CFO reporting

When the CFO asks "did we really spend $12K on AI in March?", run `GET /v1/ledger/verify` for March's sequence range. If `valid: true`, the sum of `cost_microdollars` in that range *is* the answer — verifiably, mathematically.

### Incident response

When something looks wrong on the dashboard, run a chain verification first. If the chain is valid, the dashboard is reflecting the truth and the issue is in upstream metering or pricing. If the chain is invalid, the issue is in your database — restore from backup.

### Customer audit

For SaaS customers who need a verified report of AI usage on their behalf, filter the ledger by `hierarchy_path` (which encodes attribution) or `X-Tolvyn-End-Customer` (joined via `request_id` → `requests` table). Export the slice + HMAC for the customer to verify independently.

---

## Operational notes

- **The ledger is not a queue.** Records are inserted synchronously inside the request's metering transaction. There is no separate "ledger lag" — if the request was metered, the ledger row exists.
- **The chain survives RLS** because `AppendRecord` runs inside `withTenant` which sets `app.current_tenant`. Cross-tenant reads are still blocked.
- **Backups must be physical, point-in-time, or logical with `pg_dump --no-data --schema-only` plus `pg_dump --data-only --table=ledger_records`.** Avoid tools that re-order rows during export — the SQL primary key on `sequence_number` is unique but verification walks in `sequence_number ASC` order, so any inserter that doesn't preserve order can break verification.
- **Don't manually edit the `ledger_records` table.** The application enforces RLS, advisory locks, and atomic inserts. Direct SQL bypasses all of that and almost certainly breaks the chain.

---

## See also

- [Reconciliation](reconciliation.md) — spot-check the ledger against provider invoices
- [Budgets](budgets.md) — what `budget_status` and `enforcement_action` reflect
- [API Reference: Ledger](../reference/api.md#ledger)
