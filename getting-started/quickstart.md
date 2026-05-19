# Quickstart

Get a request flowing through TOLVYN and visible in the dashboard in under 5 minutes.

---

## Prerequisites

- An API key from OpenAI, Anthropic, or Google
- One of: Python 3.9+, Node.js 18+, Go 1.21+, or any HTTP client (e.g. curl)
- A few minutes

---

## Step 1 — Sign up

Go to [app.tolvyn.io/signup](https://app.tolvyn.io/signup).

You will need:
- Your name
- A work email
- A password (minimum 8 characters)

You start on the **Free** plan: 10,000 included requests per month, no card required, no time limit. You are logged in immediately — no email verification step.

---

## Step 2 — Add your provider key

Navigate to **Account → Provider Connections** in the dashboard.

Pick a provider and paste the corresponding key:

| Provider | Key format | Where to get one |
|---|---|---|
| OpenAI | `sk-...` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| Anthropic | `sk-ant-...` | [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys) |
| Google | API key for Generative Language API | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) |

The provider key is encrypted with envelope encryption (AES-256-GCM, tenant-scoped DEK) and stored server-side. Your application code never holds it again. TOLVYN uses it to authenticate to the provider when proxying your requests.

You can add keys for all three providers from the same dashboard.

---

## Step 3 — Get your TOLVYN API key

Navigate to **API Keys → Create**.

Give the key a name (e.g. `local-dev` or `backend-prod`) and click **Create**.

The full key is shown **once** and looks like:

```
tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ
```

- Production keys are prefixed `tlv_live_`
- Test keys (for sandboxed runs) are prefixed `tlv_test_`

Save it now. The dashboard stores only the prefix and a hash — there is no recovery if you lose the full key. Create a new one if that happens.

---

## Step 4 — Make your first request

Pick the integration style that fits your stack. All four options below produce the same metered request.

### Python SDK

```bash
pip install tolvyn
```

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ",
    openai_api_key="sk-...",        # recommended: enables automatic fail-open
    team="engineering",
    service="my-app",
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.choices[0].message.content)
```

### Node.js SDK

```bash
npm install tolvyn
```

```javascript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
  tolvynApiKey: 'tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ',
  openAIApiKey: 'sk-...',           // recommended: enables automatic fail-open
  team: 'engineering',
  service: 'my-app',
});

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }],
});
console.log(response.choices[0].message.content);
```

### Go SDK

```bash
go get github.com/tolvyn/tolvyn-go@latest
```

```go
package main

import (
    "context"
    "fmt"

    oai "github.com/openai/openai-go"
    tolvyn "github.com/tolvyn/tolvyn-go"
    tolvynopenai "github.com/tolvyn/tolvyn-go/openai"
)

func main() {
    client := tolvynopenai.NewClient(tolvyn.ClientOptions{
        TolvynAPIKey:   "tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ",
        ProviderAPIKey: "sk-...",        // recommended: enables automatic fail-open
        Team:           "engineering",
        Service:        "my-app",
    })

    resp, err := client.Chat.Completions.New(context.Background(), oai.ChatCompletionNewParams{
        Model: oai.F(oai.ChatModelGPT4o),
        Messages: oai.F([]oai.ChatCompletionMessageParamUnion{
            oai.ChatCompletionUserMessageParam{
                Role: oai.F(oai.ChatCompletionUserMessageParamRoleUser),
                Content: oai.F([]oai.ChatCompletionContentPartUnionParam{
                    oai.ChatCompletionContentPartTextParam{
                        Text: oai.F("Hello"),
                        Type: oai.F(oai.ChatCompletionContentPartTextTypeText),
                    },
                }),
            },
        }),
    })
    if err != nil {
        panic(err)
    }
    fmt.Println(resp.Choices[0].Message.Content)
}
```

### curl (proxy mode)

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

> **Why add your provider key?** If TOLVYN's proxy is unreachable, the SDK
> automatically retries the request directly against the provider. Your AI
> never stops working. Omit it only if you want hard failure on TOLVYN outages.

### Using Anthropic or Google instead?

Replace the import and constructor.

**Anthropic (Python):**

```python
from tolvyn import Anthropic

client = Anthropic(
    tolvyn_api_key="tlv_live_...",
    anthropic_api_key="sk-ant-...",   # recommended: enables automatic fail-open
    team="engineering",
    service="my-app",
)
```

**Google (Python, requires the `[google]` extra):**

```python
from tolvyn import Google

goog = Google(tolvyn_api_key="tlv_live_...")
model = goog.GenerativeModel("gemini-1.5-flash")
```

**Proxy mode (any language, any provider):**

```bash
# OpenAI
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions ...

# Anthropic
curl https://proxy.tolvyn.io/v1/proxy/anthropic/v1/messages ...

# Google
curl https://proxy.tolvyn.io/v1/proxy/google/v1beta/models/...
```

---

## Step 5 — See it in the dashboard

Open [app.tolvyn.io/requests](https://app.tolvyn.io/requests).

Within a few seconds of the request completing, you will see a row with:

- Timestamp
- Model (`gpt-4o`)
- Tokens in / out
- Exact cost in USD
- Latency
- HTTP status

Click the row to see the full metadata: attribution headers, provider response time, ledger sequence number, and any tags captured from the request headers.

On the **Dashboard** page you will see the request reflected in the cost-over-time chart and the model breakdown within a minute.

---

## Next steps

You now have a working metered request. The next things to set up depend on your goal:

| Goal | Read this |
|---|---|
| Pick the right integration for production reliability | [Integration Modes](../integration-modes.md) |
| Block runaway spend before the bill arrives | [Budgets & Enforcement](../features/budgets.md) |
| Break costs down by team, service, user, or customer | [Team Insights](../features/team-insights.md) · [End Customers](../features/end-customers.md) |
| Migrate from an existing AI gateway | [Migration guides](../migration/index.md) |

---

## Troubleshooting

**Request returns 401 with `invalid_token`** — your TOLVYN key is wrong, expired, or revoked. Check the prefix in **API Keys** matches the key you're using.

**Request returns 502 with `provider_key_missing`** — you have not added a provider key for the provider you're calling. Go back to Step 2.

**Request returns 503** — TOLVYN proxy is unreachable. In SDK mode this triggers the fail-open path (the request continues directly to the provider, but is not metered). In proxy mode the request fails.

**Request succeeds but does not appear in the dashboard** — the SDK fell back direct to the provider. Check your network can reach `proxy.tolvyn.io` and that you set `tolvyn_api_key` (not `openai_api_key`) as the TOLVYN key.

Email [founder@tolvyn.io](mailto:founder@tolvyn.io) if anything else looks wrong.
