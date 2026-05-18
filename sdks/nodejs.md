# Node.js SDK

The `tolvyn` npm package wraps the official `openai`, `@anthropic-ai/sdk`, and `@google/generative-ai` SDKs so requests route through the TOLVYN proxy. Existing application code stays the same; you change the import and add a TOLVYN API key.

Current version: **1.0.6**.

---

## Installation

```bash
npm install tolvyn
```

The package declares hard dependencies on `openai ^4.0.0` and `@anthropic-ai/sdk ^0.20.0`. As of v1.0.6, `@google/generative-ai ^0.21.0` is an **optional `peerDependency`** — install it alongside `tolvyn` only if you need the `Google` client; OpenAI/Anthropic-only consumers can skip it to reduce bundle size.

The package ships both ESM and CJS builds. The `exports` field in `package.json` resolves to the right format based on your module system:

```json
"exports": {
  ".": {
    "import": "./dist/esm/index.js",
    "require": "./dist/cjs/index.js",
    "types": "./dist/cjs/index.d.ts"
  }
}
```

Node.js 18 or later is required (the SDK uses the built-in `fetch`).

---

## Quick start

```javascript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
  tolvynApiKey: 'tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ',
});

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }],
});
console.log(response.choices[0].message.content);
```

Everything after construction is identical to the official `openai` package — streaming, tool use, batch all work unchanged.

---

## OpenAI

```javascript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  team: 'backend',
  service: 'summarizer',
  openAIApiKey: 'sk-...',   // used only for fail-open fallback
});

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }],
});
```

### TolvynOpenAIOptions

```typescript
interface TolvynOpenAIOptions extends Omit<ClientOptions, 'apiKey' | 'baseURL'> {
  tolvynApiKey?: string;
  proxyUrl?: string;
  team?: string;
  service?: string;
  feature?: string;
  agent?: string;
  user?: string;
  endCustomer?: string;
  failOpen?: boolean;
  openAIApiKey?: string;
}
```

`OpenAI` extends `OpenAIBase` (`openai`'s default export). All native `ClientOptions` except `apiKey` and `baseURL` pass through to the parent — `timeout`, `maxRetries`, `defaultQuery`, `fetch`, etc. all work.

---

## Anthropic

```javascript
import { Anthropic } from 'tolvyn';

const client = new Anthropic({
  tolvynApiKey: 'tlv_live_...',
  anthropicApiKey: 'sk-ant-...',   // used only for fail-open fallback
});

const response = await client.messages.create({
  model: 'claude-haiku-4-5',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Hello' }],
});
console.log(response.content[0].text);
```

### TolvynAnthropicOptions

```typescript
interface TolvynAnthropicOptions extends Omit<ClientOptions, 'apiKey' | 'baseURL'> {
  tolvynApiKey?: string;
  proxyUrl?: string;
  team?: string;
  service?: string;
  feature?: string;
  agent?: string;
  user?: string;
  endCustomer?: string;
  failOpen?: boolean;
  anthropicApiKey?: string;
}
```

`Anthropic` extends `AnthropicBase` from `@anthropic-ai/sdk`.

---

## Google

```javascript
import { Google } from 'tolvyn';

const ai = new Google({ tolvynApiKey: 'tlv_live_...' });
const model = ai.getGenerativeModel({ model: 'gemini-1.5-flash' });

const result = await model.generateContent('Hello');
console.log(result.response.text());
```

### TolvynGoogleOptions

```typescript
interface TolvynGoogleOptions {
  tolvynApiKey?: string;
  proxyUrl?: string;
  team?: string;
  service?: string;
  feature?: string;
  agent?: string;
  user?: string;
  endCustomer?: string;
  failOpen?: boolean;
  googleApiKey?: string;
}
```

As of v1.0.6, the class is named `Google` in source (was `TolvynGoogle` internally before — stack traces and `constructor.name` now show `Google`). It extends `GoogleGenerativeAI`. `getGenerativeModel()` is overridden to inject the TOLVYN proxy URL and attribution headers on every call.

### Google differences from OpenAI / Anthropic

- The interface is **not** `Omit<ClientOptions, ...>` — Google's SDK does not expose a constructor options interface to extend, so `TolvynGoogleOptions` stands alone. Pass-through fields like `timeout` are not available.
- The TOLVYN proxy URL is applied per-call by `getGenerativeModel()`, not stored as a base URL on the client. Multiple `Google` instances in the same process are safe — no global state.
- The TOLVYN API key is sent to Google's SDK and forwarded as `x-goog-api-key`. The TOLVYN proxy reads `x-goog-api-key` as a Bearer-auth fallback.

### Known Google limitations

- **Google fail-open works as of v1.0.6.** When `failOpen: true` and `googleApiKey` is set, the SDK wraps `generateContent` so proxy-unreachable errors retry against a direct `GoogleGenerativeAI` client. Unlike Python, the Node SDK is not constrained by process-global state — no warnings are emitted. Trigger conditions match OpenAI/Anthropic (see [Fail-open behavior](#fail-open-behavior)).

---

## Constructor parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `tolvynApiKey` | `string` | `undefined` (env: `TOLVYN_API_KEY`) | TOLVYN API key (`tlv_live_...` / `tlv_test_...`). Required — throws if neither argument nor env var is set. |
| `proxyUrl` | `string` | provider-specific default; env: `TOLVYN_PROXY_URL` | Override the proxy URL. Defaults: OpenAI `https://proxy.tolvyn.io/v1/proxy/openai/`, Anthropic `https://proxy.tolvyn.io/v1/proxy/anthropic/`, Google `https://proxy.tolvyn.io/v1/proxy/google` (no trailing slash). |
| `team` | `string` | `undefined` | Sent as `X-Tolvyn-Team`. |
| `service` | `string` | `undefined` | Sent as `X-Tolvyn-Service`. |
| `feature` | `string` | `undefined` | Sent as `X-Tolvyn-Feature`. |
| `agent` | `string` | `undefined` | Sent as `X-Tolvyn-Agent`. |
| `user` | `string` | `undefined` | Sent as `X-Tolvyn-User`. |
| `endCustomer` | `string` | `undefined` | Sent as `X-Tolvyn-End-Customer`. |
| `failOpen` | `boolean` | `true` | Enable automatic fallback to the provider direct when the proxy is unreachable. Requires a provider key. Non-functional on Google. |
| `openAIApiKey` | `string` | `undefined` (env: `OPENAI_API_KEY`) | Provider key for fail-open fallback. `OpenAI` only. Note the capitalization: `openAIApiKey` (camelCase with `AI` upper). |
| `anthropicApiKey` | `string` | `undefined` (env: `ANTHROPIC_API_KEY`) | Provider key for fail-open fallback. `Anthropic` only. |
| `googleApiKey` | `string` | `undefined` (env: `GOOGLE_API_KEY`) | Provider key — `Google` only. Currently stored but unused (see Google limitations). |
| Pass-through `ClientOptions` | various | various | For `OpenAI` and `Anthropic`, all native fields except `apiKey` and `baseURL` are forwarded to the parent constructor. Not supported on `Google`. |

Header values are only sent when the parameter is non-empty.

---

## Attribution headers

Six headers identify who and what made the request:

```javascript
const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  team: 'backend',
  service: 'invoice-summarizer',
  feature: 'summarize',
  agent: 'claude-code',
  user: 'alice@company.com',
  endCustomer: 'acme-corp',
});
```

Produces request headers:

```
X-Tolvyn-Team: backend
X-Tolvyn-Service: invoice-summarizer
X-Tolvyn-Feature: summarize
X-Tolvyn-Agent: claude-code
X-Tolvyn-User: alice@company.com
X-Tolvyn-End-Customer: acme-corp
```

All six are stripped by the TOLVYN proxy before the request is forwarded to OpenAI / Anthropic / Google.

---

## Fail-open behavior

Fail-open is enabled by default (`failOpen: true`). When the SDK cannot reach `proxy.tolvyn.io`, it retries the request directly against the provider using your provider key (for Google, requires v1.0.6+; earlier versions silently ignored `failOpen`). A message is written to `console.error`:

```
TOLVYN proxy unreachable — routing direct to OpenAI (fail-open)
```

### When fail-open triggers

Source: `failopen.ts` `isProxyError()`. Any of the following:

- Node.js connection errors with `code` property:
  - `ECONNREFUSED`
  - `ECONNRESET`
  - `ETIMEDOUT`
  - `ENOTFOUND`
  - Any code starting with `ERR_`
- HTTP `503 Service Unavailable` from the proxy (detected on `status` or `statusCode`)
- `fetch`-level error messages containing `ECONNREFUSED`, `ETIMEDOUT`, `fetch failed`, or `connect ECONNREFUSED`
- `cause`-wrapped errors (Node 18+ wraps low-level errors in a parent error with `cause`) — `isProxyError` recurses into `cause`

### When fail-open does NOT trigger

Source: `failopen.ts` `shouldNotFailOpen()`. The SDK does **not** fall back on:

- Any `4xx` response (`400`, `401`, `403`, `404`, `422`, `429`) — real API errors propagate as-is
- `5xx` other than `503` (`500`, `502`, `504`) — these surface to your code

### Configuration

Fail-open requires a provider key. Resolution order:

1. Explicit `openAIApiKey` / `anthropicApiKey` / `googleApiKey` option
2. Environment variable (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`)

If no provider key is configured, fail-open is a no-op and the original error propagates.

To disable fail-open and force every request through TOLVYN:

```javascript
const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  failOpen: false,
});
```

### Metering during fail-open

Requests that fall back to the provider directly **bypass the TOLVYN proxy**. They are not metered, not budget-checked, and not recorded in the ledger for that call. The dashboard shows no row for them. This is the trade-off: applications keep working during a TOLVYN outage at the cost of observability for affected requests.

### Note on Google

The `failOpen` option on the `Google` class is currently a no-op. There is no fallback transport when the proxy is unreachable.

---

## Environment variables

| Variable | Used for | Required |
|---|---|---|
| `TOLVYN_API_KEY` | Default `tolvynApiKey` when not passed explicitly | Yes (unless option is passed) |
| `TOLVYN_PROXY_URL` | Default `proxyUrl` override | No |
| `OPENAI_API_KEY` | Fail-open fallback for `OpenAI` | No |
| `ANTHROPIC_API_KEY` | Fail-open fallback for `Anthropic` | No |
| `GOOGLE_API_KEY` | Resolved but currently unused (Google fail-open not implemented) | No |

Sources: `client.ts`, `anthropic.ts`, `google.ts`.

---

## TypeScript usage

Types ship with the package. The exported option interfaces are importable:

```typescript
import { OpenAI, Anthropic, Google } from 'tolvyn';
import type {
  TolvynOpenAIOptions,
  TolvynAnthropicOptions,
  TolvynGoogleOptions,
} from 'tolvyn';

const opts: TolvynOpenAIOptions = {
  tolvynApiKey: process.env.TOLVYN_API_KEY,
  team: 'backend',
  service: 'summarizer',
};

const client = new OpenAI(opts);

const resp = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }],
});

const text: string = resp.choices[0].message.content ?? '';
console.log(text);
```

The TOLVYN classes inherit all type information from the underlying `openai` / `@anthropic-ai/sdk` types — autocomplete, parameter typing, response shapes all work identically.

---

## ESM vs CJS

The package ships both module formats and is wired so the right one resolves based on the consumer's module system.

### ESM (`"type": "module"` in your `package.json`, or `.mjs` file)

```javascript
import { OpenAI } from 'tolvyn';
```

### CJS (default Node, or `.cjs` file)

```javascript
const { OpenAI } = require('tolvyn');
```

TypeScript with `"module": "commonjs"` or `"node16"`/`"nodenext"` works without configuration changes. Type declarations resolve via the `types` field.

---

## Version and changelog

| Version | Notes |
|---|---|
| 1.0.6 | Current. Google fail-open functional (ND-01); class renamed `TolvynGoogle` → `Google` (ND-05); `@google/generative-ai` is now an optional peer dependency (ND-08). |
| 1.0.5 | Fail-open URL composition fix (ND-02); deduplicate `makeFailOpenFetch` so client.ts imports from failopen.ts (ND-03); trailing slash on Google proxy URL (ND-07). |
| 1.0.4 | OpenAI + Anthropic + Google. Node 18+. Dual ESM/CJS build. |

The Node SDK reached `1.0` ahead of the Python SDK (currently `0.1.5`) — the underlying provider SDKs (`openai`, `@anthropic-ai/sdk`) have stable APIs that gave Node a faster path to a stable public surface.

---

## See also

- [Integration Modes](../integration-modes.md) — SDK mode vs proxy mode trade-offs
- [Quickstart](../getting-started/quickstart.md) — end-to-end first request walkthrough
- [Migrate from OpenAI direct](../migration/from-openai-direct.md)
- [Python SDK](python.md) · [Go SDK](go.md)
