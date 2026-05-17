# Migrate from Portkey to TOLVYN

Two things change: the base URL and how you authenticate. Everything else stays the same.

---

## What changes

| | Portkey | TOLVYN |
|---|---|---|
| Base URL | `https://api.portkey.ai/v1` | `https://proxy.tolvyn.io/v1/proxy/openai` |
| Auth | `x-portkey-api-key: pk_...` | `Authorization: Bearer tlv_live_...` |
| Provider key | Virtual key (`x-portkey-virtual-key`) | Stored encrypted server-side (set once in dashboard) |

Your prompts, models, parameters, and response format are unchanged.

---

## Before / after

### Python (openai SDK)

**Before:**
```python
from openai import OpenAI

client = OpenAI(
    api_key="anything",  # ignored by Portkey
    base_url="https://api.portkey.ai/v1",
    default_headers={
        "x-portkey-api-key": "pk_...",
        "x-portkey-virtual-key": "openai-vk-...",
        "x-portkey-trace-id": "trace-abc123",
    },
)
```

**After:**
```python
from openai import OpenAI

client = OpenAI(
    api_key="tlv_live_...",   # your TOLVYN key
    base_url="https://proxy.tolvyn.io/v1/proxy/openai",
)
```

Add your real OpenAI key once in the TOLVYN dashboard under **Provider Connections** — it is stored encrypted server-side and never sent to the client.

---

### Node.js (openai SDK)

**Before:**
```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "anything",
  baseURL: "https://api.portkey.ai/v1",
  defaultHeaders: {
    "x-portkey-api-key": "pk_...",
    "x-portkey-virtual-key": "openai-vk-...",
  },
});
```

**After:**
```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "tlv_live_...",
  baseURL: "https://proxy.tolvyn.io/v1/proxy/openai",
});
```

---

### curl

**Before:**
```bash
curl https://api.portkey.ai/v1/chat/completions \
  -H "x-portkey-api-key: pk_..." \
  -H "x-portkey-virtual-key: openai-vk-..." \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
```

**After:**
```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
```

---

## Header mapping

TOLVYN reads Portkey metadata headers automatically during migration.

| Portkey Header | TOLVYN Equivalent | Notes |
|---|---|---|
| `x-portkey-api-key` | `Authorization: Bearer tlv_live_...` | Replace with your TOLVYN key |
| `x-portkey-virtual-key` | Provider key stored server-side | Set once in TOLVYN dashboard; remove from client |
| `x-portkey-trace-id` | stored in `tags.portkey_trace_id` | Auto-captured |
| `x-portkey-metadata` | merged into `tags.portkey_<key>` | JSON object values auto-captured |

All Portkey headers are stripped before the request reaches OpenAI — they will not appear in provider logs.

---

## Virtual keys vs. server-side provider keys

Portkey uses virtual keys to reference stored credentials from the client. TOLVYN stores the real provider key encrypted on the server — it never leaves.

1. Go to **Account → Provider Connections** in the TOLVYN dashboard.
2. Add your OpenAI (or Anthropic / Google) key.
3. Remove the `x-portkey-virtual-key` header from your client code.

Your client never holds a real provider key again.

---

## Feature comparison

| Feature | Portkey | TOLVYN |
|---|---|---|
| Request logging | ✅ | ✅ |
| Cost tracking | ✅ | ✅ |
| Multi-provider routing | ✅ | ✅ |
| Budget enforcement | limited | ✅ |
| Hard kill switch | ❌ | ✅ |
| Immutable audit ledger | ❌ | ✅ |
| Per-customer attribution | ❌ | ✅ |
| Invoice reconciliation | ❌ | ✅ |
| Savings analysis | ❌ | ✅ |

**On pricing:** Portkey Pro is $49/mo with basic observability and routing. TOLVYN Growth is $199/mo with budget enforcement, kill switches, a financial-grade audit trail, per-customer attribution, and invoice reconciliation. If you only need routing and logs, Portkey is cheaper. If you need financial control and accountability, TOLVYN is built for that.

---

## TOLVYN-native headers (optional)

Once migrated, adopt TOLVYN's attribution headers for richer breakdowns:

```
X-Tolvyn-Team: backend-team
X-Tolvyn-Service: summarization
X-Tolvyn-Feature: report-gen
X-Tolvyn-User: user-123
X-Tolvyn-End-Customer: acme-corp
```

These are all optional. The dashboard shows cost and usage breakdowns per team, service, user, and end customer automatically.
