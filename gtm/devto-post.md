# How I built an immutable audit ledger for AI costs in Go

---

## The problem

Every engineering team I talked to had the same issue: AI features ship, the invoice arrives, and nobody can explain where the money went. Not which team. Not which service. Not which model change tripled costs overnight. Finance sees a number. Engineering sees logs. Neither can connect them. The invoice just says "$12,000" and everyone shrugs.

The deeper problem is governance. When you have ten services hitting the same OpenAI account, there is no accountability at the service level. A developer swaps `claude-haiku-4-5` for `claude-sonnet-4-6` in a staging config, that config accidentally ships to production, and you spend six weeks wondering why costs are 10x higher. I wanted something that would catch that in real time — and prove it, permanently, in a way that couldn't be edited after the fact.

## Why logs aren't enough

Application logs are mutable. You can delete them, rotate them, accidentally overwrite them. They also aren't typically designed for financial evidence — they're designed for debugging. If your CFO asks "did we really spend $12K in March?" and your answer is "yes, trust the log file," that's not an audit trail. That's a sticky note.

What you need for financial evidence is something that can't be silently modified. If a record changes, you need to know immediately. Blockchain gets thrown around for this, but the overhead is enormous and you don't need decentralization — you just need tamper evidence. A hash-chained ledger gives you exactly that: any modification to any historical record breaks all subsequent hashes, making the tampering detectable without requiring external consensus.

## The architecture

Every AI request that passes through the TOLVYN proxy appends a record to the ledger. Each record contains a SHA-256 hash of the previous record, creating a chain. Any modification to a historical record will produce a different hash for that record, which cascades — every subsequent record's `previous_hash` will no longer match the hash of the record before it.

Here's the simplified Go implementation:

```go
// Simplified — actual implementation in Go
type LedgerRecord struct {
    SequenceNumber   int64
    PreviousHash     string  // SHA-256 of prior record
    RecordHash       string  // SHA-256 of this record's payload
    HMACSignature    string  // HMAC-SHA256(record_hash, secret)
    RequestID        string
    CostMicrodollars int64
    // ... other fields
}

func appendRecord(db *sql.DB, req Request) error {
    // Get last record's hash
    var prevHash string
    db.QueryRow(`SELECT record_hash FROM ledger_records
        WHERE tenant_id=$1
        ORDER BY sequence_number DESC LIMIT 1`,
        req.TenantID).Scan(&prevHash)

    // Hash this record
    payload := canonicalJSON(req, prevHash)
    recordHash := sha256Hex(payload)
    hmac := hmacSHA256(recordHash, secret)

    // Append — never update
    db.Exec(`INSERT INTO ledger_records ...`,
        prevHash, recordHash, hmac, ...)
}
```

A few design decisions worth noting:

**Costs are stored in microdollars (int64), not floats.** Financial arithmetic with `float64` accumulates rounding errors. Storing everything as integer microdollars and using `shopspring/decimal` for intermediate calculation means your totals are exact.

**HMAC on top of the hash.** The SHA-256 chain catches accidental or external modification. The HMAC-SHA256 (keyed with a server secret) makes it harder for someone with direct DB access to recompute a valid hash after tampering. Both layers serve different threat models.

**Append-only at the application level.** The application never issues `UPDATE` or `DELETE` on `ledger_records`. PostgreSQL Row Level Security policies enforce this at the DB level, not just the application layer.

## The gzip bug

This one took me an hour to debug — and it's the kind of bug that produces no error messages, which makes it brutal.

Python's `httpx` library (used by the Anthropic SDK) sends `Accept-Encoding: gzip` by default on every request. Go's `net/http` transport normally handles gzip decompression transparently — but only when **it** sets the `Accept-Encoding` header itself. When the header is explicitly present on an outgoing request (forwarded verbatim from the client), Go assumes the caller wants to handle decompression and skips it.

The result: `json.Unmarshal` received raw gzip bytes. No error was returned — the JSON parser just couldn't find any valid fields. Every response had 0 input tokens, 0 output tokens, and $0.000000 cost. Silent, total failure.

The fix was one line: strip `Accept-Encoding` from the outgoing request before forwarding it to the provider.

```go
outbound.Header.Del("Accept-Encoding")
```

Go then sets its own `Accept-Encoding: gzip` on the request, handles decompression itself, and `json.Unmarshal` receives valid JSON. Token counts and costs populated correctly from that point.

## Verification

The chain is verifiable at any time via a single endpoint:

```bash
curl https://api.tolvyn.io/v1/ledger/verify \
  -H "Authorization: Bearer <token>"
# {"valid":true,"records_checked":64}
```

The verification endpoint walks every record in sequence, recomputes each hash from the stored payload, checks it against the stored `record_hash`, and verifies the HMAC. If any record has been modified, the chain breaks at that point and the response identifies which sequence number failed.

## What this enables

The practical outcomes are what make the architecture worth the complexity:

- **CFO asks "did we really spend $12K in March?"** — verify the chain, don't just look at a number. The ledger proves it.
- **Compliance audit** — export the full ledger for any date range, run verification, attach the output. Done.
- **Operational visibility** — every request is attributed to a team and service with exact token counts, so "who spent $3,400 this week" has a real answer.
- **Budget enforcement** — hard limits check against the running total before the provider request is dispatched. The provider is never called if the budget is exceeded. The ledger records that block as well.

The combination of real-time cost attribution and tamper-evident storage is what makes this useful for more than just dashboards. It's financial evidence, not just monitoring.

## Try it

Free trial at [tolvyn.io](https://tolvyn.io).

```bash
pip install tolvyn
# or
npm install tolvyn
```

One line change. Every AI call metered, attributed, and governed.

---

© 2026 TOLVYN. All rights reserved.
