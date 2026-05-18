# TOLVYN

TOLVYN is the financial control plane for AI infrastructure. Every AI API call — metered, attributed, governed.

---

## What you get

- **Real-time cost metering.** Every request that goes through TOLVYN is recorded with model, token counts, latency, and exact cost in microdollars. Pricing is live, not estimated.
- **Budget enforcement.** Set hard or soft budgets per team, service, agent, user, or end-customer. Hard budgets block requests at the proxy before the provider is called.
- **Immutable audit ledger.** Every request appends to a SHA-256 hash-chained ledger with HMAC signatures. Verifiable at any time via `GET /v1/ledger/verify`. Designed for financial evidence, not just monitoring.
- **Drop-in SDKs and proxy.** Replace one import line in Python, Node.js, or Go. Or point any HTTP client at `proxy.tolvyn.io`. OpenAI, Anthropic, and Google supported.

---

## 60-second quickstart

1. Sign up at [app.tolvyn.io/signup](https://app.tolvyn.io/signup)
2. **Account → Provider Connections** — add your OpenAI / Anthropic / Google key (stored encrypted server-side)
3. **API Keys → Create** — copy the `tlv_live_...` key (shown once)
4. Use it:

   ```python
   from tolvyn import OpenAI
   client = OpenAI(tolvyn_api_key="tlv_live_...")
   client.chat.completions.create(model="gpt-4o", messages=[...])
   ```

5. Open the dashboard — the request appears within seconds with cost, tokens, and latency.

Full walkthrough: [Quickstart](getting-started/quickstart.md).

---

## Where to next

| If you want to... | Go to |
|---|---|
| Get to a working request in 5 minutes | [Quickstart](getting-started/quickstart.md) |
| Choose between SDK and proxy mode | [Integration Modes](integration-modes.md) |
| See SDK reference for your language | [Python](sdks/python.md) · [Node.js](sdks/nodejs.md) · [Go](sdks/go.md) |
| Move from Helicone, Portkey, or direct OpenAI | [Migration guides](migration/index.md) |

---

## Need help?

Email [founder@tolvyn.io](mailto:founder@tolvyn.io).
