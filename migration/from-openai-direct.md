# Migrate from OpenAI direct to TOLVYN

Two lines of code. That's it.

---

## What changes

1. Your API key (`sk-...` → `tlv_live_...`)
2. The base URL (`https://api.openai.com` → `https://proxy.tolvyn.io/v1/proxy/openai`)

Your prompts, models, parameters, and response format are completely unchanged. TOLVYN speaks OpenAI's API natively.

---

## Before / after

### Python

**Before:**
```python
from openai import OpenAI

client = OpenAI(api_key="sk-...")
```

**After:**
```python
from openai import OpenAI

client = OpenAI(
    api_key="tlv_live_...",
    base_url="https://proxy.tolvyn.io/v1/proxy/openai",
)
```

Add your real OpenAI key once in the TOLVYN dashboard (**Account → Provider Connections**). Your app code never holds a provider key again.

---

### Node.js

**Before:**
```js
import OpenAI from "openai";

const client = new OpenAI({ apiKey: "sk-..." });
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
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer sk-..." \
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

### Raw HTTP

**Before:**
```
POST /v1/chat/completions HTTP/1.1
Host: api.openai.com
Authorization: Bearer sk-...
Content-Type: application/json
```

**After:**
```
POST /v1/proxy/openai/v1/chat/completions HTTP/1.1
Host: proxy.tolvyn.io
Authorization: Bearer tlv_live_...
Content-Type: application/json
```

---

## What you get immediately

Once the two lines are changed, TOLVYN automatically:

- Logs every request with model, tokens, cost, and latency
- Calculates cost in real-time using live pricing data
- Stores an immutable audit ledger of all spend
- Lets you set budgets and get alerts when limits are approached

No configuration required. The dashboard is live the moment the first request goes through.

---

## Fail-open guarantee

If TOLVYN ever has an issue, requests automatically pass through to OpenAI. Your application never goes down because of TOLVYN. This is by design — TOLVYN is in your request path for observability, not for availability.

---

## Optional: add attribution headers

To see cost broken down by team, feature, or user in the dashboard:

```
X-Tolvyn-Team: backend-team
X-Tolvyn-Service: summarization
X-Tolvyn-Feature: report-gen
X-Tolvyn-User: user-123
X-Tolvyn-End-Customer: acme-corp
```

None of these are required. Add them incrementally as you need the visibility.
