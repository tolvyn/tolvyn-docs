# Migrate from Helicone to TOLVYN

Two things change: the base URL and your API key. Everything else stays the same.

---

## What changes

| | Helicone | TOLVYN |
|---|---|---|
| Base URL | `https://oai.helicone.ai/v1` | `https://proxy.tolvyn.io/v1/proxy/openai` |
| API key header | `Helicone-Auth: Bearer hcsk_...` | `Authorization: Bearer tlv_live_...` |

Your prompts, models, parameters, and response format are unchanged.

---

## Before / after

### Python (openai SDK)

**Before:**
```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-...",
    base_url="https://oai.helicone.ai/v1",
    default_headers={
        "Helicone-Auth": "Bearer hcsk_...",
        "Helicone-User-Id": "user-123",
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

User attribution works automatically — TOLVYN reads `Helicone-User-Id` if `X-Tolvyn-User` is absent. You can keep the header or remove it.

---

### Node.js (openai SDK)

**Before:**
```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "sk-...",
  baseURL: "https://oai.helicone.ai/v1",
  defaultHeaders: {
    "Helicone-Auth": "Bearer hcsk_...",
    "Helicone-User-Id": "user-123",
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
curl https://oai.helicone.ai/v1/chat/completions \
  -H "Authorization: Bearer sk-..." \
  -H "Helicone-Auth: Bearer hcsk_..." \
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

TOLVYN reads Helicone headers automatically during migration — you don't need to change them immediately.

| Helicone Header | TOLVYN Equivalent | Notes |
|---|---|---|
| `Helicone-Auth` | `Authorization: Bearer tlv_live_...` | Replace with your TOLVYN key |
| `Helicone-User-Id` | `X-Tolvyn-User` | Auto-mapped — works without changes |
| `Helicone-Session-Id` | stored in `tags.helicone_session_id` | Auto-captured |
| `Helicone-Property-*` | stored in `tags.helicone_<name>` | Auto-captured with `helicone_` prefix |

All Helicone headers are stripped before the request reaches OpenAI — they will not appear in provider logs.

---

## Feature comparison

| Feature | Helicone | TOLVYN |
|---|---|---|
| Request logging | ✅ | ✅ |
| Cost tracking | ✅ | ✅ |
| Per-user attribution | ✅ | ✅ |
| Budget enforcement | ❌ | ✅ |
| Hard kill switch | ❌ | ✅ |
| Immutable audit ledger | ❌ | ✅ |
| Per-customer attribution | ❌ | ✅ |
| Invoice reconciliation | ❌ | ✅ |
| Savings analysis | ❌ | ✅ |
| Multi-provider (OpenAI + Anthropic + Google) | ✅ | ✅ |

**On pricing:** Helicone Pro is $79/mo with basic observability. TOLVYN Growth is $199/mo with budget enforcement, kill switches, a financial-grade audit trail, per-customer attribution, and invoice reconciliation. If you only need logs, Helicone is cheaper. If you need control, TOLVYN pays for itself the first time a runaway agent gets blocked.

---

## TOLVYN-native headers (optional)

Once migrated, you can adopt TOLVYN's own attribution headers for richer data:

```
X-Tolvyn-Team: backend-team
X-Tolvyn-Service: summarization
X-Tolvyn-Feature: report-gen
X-Tolvyn-Agent: gpt-pipeline
X-Tolvyn-User: user-123
X-Tolvyn-End-Customer: acme-corp
```

These are all optional — use none, some, or all.
