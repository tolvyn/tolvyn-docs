# Integration Modes

TOLVYN offers two integration modes. Pick one based on your reliability requirements.

## TL;DR

| Capability | SDK mode | Proxy mode |
|------------|----------|------------|
| Setup | Install `tolvyn` package | Set `OPENAI_BASE_URL` env var |
| Languages | Python, Node.js | Any |
| Fail-open (auto-fallback to provider direct) | ✅ Yes | ❌ No — request fails if TOLVYN is unreachable |
| Latency overhead | <50ms | <50ms |
| TOLVYN in critical path | No (fails open) | Yes |
| Recommended for | Production workloads | Prototyping, internal tools, non-critical batch |

**Rule of thumb:** Use SDK mode for production. Use proxy mode when you want the simplest possible integration and can tolerate rare TOLVYN downtime.

---

## SDK Mode

SDK mode is a drop-in replacement for the OpenAI or Anthropic SDK. Install the `tolvyn` package, change one import line, and your calls are metered.

The key advantage: **if TOLVYN is unreachable, the SDK automatically retries the request directly to OpenAI or Anthropic.** Your AI never stops working.

### Python

```python
# Before (standard OpenAI)
from openai import OpenAI
client = OpenAI(api_key="sk-...")

# After (TOLVYN SDK mode)
from tolvyn import OpenAI
client = OpenAI(
    tolvyn_api_key="tlv_live_...",   # your TOLVYN key
    openai_api_key="sk-...",          # your OpenAI key (used for fail-open fallback)
    team="engineering",
    service="chatbot-api"
)

# Everything else is identical
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}]
)
```

### Node.js

```javascript
// Before (standard OpenAI)
import OpenAI from 'openai'
const client = new OpenAI({ apiKey: 'sk-...' })

// After (TOLVYN SDK mode)
import { OpenAI } from 'tolvyn'
const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  openAIApiKey: 'sk-...',        // used for fail-open fallback
  team: 'engineering',
  service: 'chatbot-api'
})

// Everything else is identical
const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }]
})
```

### What happens when TOLVYN is down?

When the SDK cannot reach `proxy.tolvyn.io` (connection refused, timeout, or HTTP 503):

1. SDK detects the failure automatically — no error is thrown to your code
2. SDK retries the request directly to `api.openai.com` using your `openai_api_key`
3. Your AI call succeeds as normal
4. The request is **not** metered in TOLVYN for that call (it bypassed the proxy)
5. You may see a slightly higher latency on that one request (one retry)

This means TOLVYN is **never in your critical path** in SDK mode.

---

## Proxy Mode

Proxy mode requires no SDK installation. Point your existing OpenAI client at TOLVYN's proxy URL.

```bash
# Set these environment variables
export OPENAI_BASE_URL=https://proxy.tolvyn.io/v1/proxy/openai
export OPENAI_API_KEY=tlv_live_...   # your TOLVYN key, not your OpenAI key
```

### Python (proxy mode)

```python
from openai import OpenAI

client = OpenAI(
    api_key="tlv_live_...",                               # TOLVYN key
    base_url="https://proxy.tolvyn.io/v1/proxy/openai"   # TOLVYN proxy
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
    extra_headers={
        "X-Tolvyn-Team": "engineering",
        "X-Tolvyn-Service": "chatbot-api"
    }
)
```

### curl (proxy mode)

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -H "X-Tolvyn-Team: engineering" \
  -H "X-Tolvyn-Service: chatbot-api" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### Any language (raw HTTP)

```
POST https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions
Authorization: Bearer tlv_live_...
Content-Type: application/json
X-Tolvyn-Team: engineering
X-Tolvyn-Service: chatbot-api

{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}
```

### Important: TOLVYN is in your critical path

In proxy mode, if `proxy.tolvyn.io` is unreachable, your AI call fails. There is no automatic fallback.

To mitigate this:
- Set up uptime monitoring on `https://proxy.tolvyn.io/health`
- Configure your HTTP client with a reasonable timeout (recommended: 30s)
- Use SDK mode for any workload where AI downtime is unacceptable

---

## Switching from proxy mode to SDK mode

It is two lines:

**Python:**
```python
# Remove this:
base_url="https://proxy.tolvyn.io/v1/proxy/openai"

# Add this:
from tolvyn import OpenAI   # change import
openai_api_key="sk-..."     # add fallback key
```

**Node.js:**
```javascript
// Remove this:
baseURL: 'https://proxy.tolvyn.io/v1/proxy/openai'

// Add this:
import { OpenAI } from 'tolvyn'   // change import
openAIApiKey: 'sk-...'            // add fallback key
```

---

## Reliability Guarantees

**What TOLVYN guarantees:**
- Proxy latency overhead: <50ms p99
- SDK fail-open: automatic fallback within 5 seconds of detecting proxy unreachability
- No prompt modification: TOLVYN never modifies your prompts or messages
- No response modification: TOLVYN never modifies provider responses
- No content storage: TOLVYN stores metadata (model, tokens, cost, latency) only — never prompt text or response content

**What TOLVYN does not guarantee:**
- Provider availability (OpenAI, Anthropic, Google outages are outside TOLVYN's control)
- 100% metering coverage in SDK mode (fail-open requests bypass metering)

**Health endpoint:**
```
GET https://proxy.tolvyn.io/health
→ {"status":"ok","db":"ok","version":"1.0.0"}
```

Use this for uptime monitoring in proxy mode.

---

## Production Checklist

Before going to production with TOLVYN:

- [ ] **SDK mode**: Install `tolvyn` package, set both `tolvyn_api_key` and provider `api_key` for fail-open
- [ ] **Proxy mode**: Set up uptime monitoring on `proxy.tolvyn.io/health`
- [ ] Set `X-Tolvyn-Team` and `X-Tolvyn-Service` headers — required for cost attribution
- [ ] Create a budget with hard mode for each team or service
- [ ] Test the fail-open path: temporarily set `proxy_url` to an unreachable host and verify your code still works (SDK mode only)
- [ ] Add `X-Tolvyn-User` for per-user tracking (optional but recommended)
- [ ] Add `X-Tolvyn-End-Customer` if you are a SaaS company billing AI costs to customers

---

## See also

- [Migrate from Helicone](./migration/from-helicone.md)
- [Migrate from direct OpenAI](./migration/from-openai-direct.md)
- [Python SDK reference](./sdks/python.md)
- [Node.js SDK reference](./sdks/nodejs.md)
