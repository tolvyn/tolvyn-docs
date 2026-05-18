# Go SDK

The `tolvyn-go` module wraps the official `github.com/openai/openai-go`, `github.com/anthropics/anthropic-sdk-go`, and `github.com/google/generative-ai-go` clients so requests route through the TOLVYN proxy. Existing code stays the same; you change the constructor and add a TOLVYN API key.

Current published tag: **v0.1.1**.

---

## Installation

The module is split into per-provider sub-packages so you only pull in the official SDKs you actually use:

```bash
go get github.com/tolvyn/tolvyn-go/openai
go get github.com/tolvyn/tolvyn-go/anthropic
go get github.com/tolvyn/tolvyn-go/google
```

Go 1.21 or later is required (`go.mod` declares `go 1.21`).

The underlying provider SDKs pinned in `go.mod`:
- `github.com/openai/openai-go v0.1.0-alpha.62`
- `github.com/anthropics/anthropic-sdk-go v0.2.0-alpha.8`
- `github.com/google/generative-ai-go v0.18.0`

The OpenAI and Anthropic SDKs are alpha — their public APIs may change.

---

## Quick start

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
        TolvynAPIKey: "tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ",
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

The verbosity comes from the alpha `openai-go` SDK, not from TOLVYN. Once the SDK reaches a stable release this will simplify.

---

## OpenAI

```go
import (
    tolvyn "github.com/tolvyn/tolvyn-go"
    tolvynopenai "github.com/tolvyn/tolvyn-go/openai"
)

client := tolvynopenai.NewClient(tolvyn.ClientOptions{
    TolvynAPIKey:   "tlv_live_...",
    ProviderAPIKey: "sk-...",          // used for fail-open fallback
    Team:           "backend",
    Service:        "summarizer",
})
```

### Constructor signature

```go
func NewClient(opts tolvyn.ClientOptions) *Client

// Convenience: read TOLVYN_API_KEY and OPENAI_API_KEY from env.
func NewClientFromEnv() *Client
```

`Client` embeds `*openai.Client` from the alpha `openai-go` SDK, so every method on the official client is available unchanged.

---

## Anthropic

```go
import (
    tolvyn "github.com/tolvyn/tolvyn-go"
    tolvynanthropic "github.com/tolvyn/tolvyn-go/anthropic"
)

client := tolvynanthropic.NewClient(tolvyn.ClientOptions{
    TolvynAPIKey:   "tlv_live_...",
    ProviderAPIKey: "sk-ant-...",      // used for fail-open fallback
})
```

### Constructor signature

```go
func NewClient(opts tolvyn.ClientOptions) *Client
func NewClientFromEnv() *Client
```

`Client` embeds `*anthropic.Client` from `anthropic-sdk-go`. The SDK sets the required `anthropic-version: 2023-06-01` header automatically.

---

## Google

```go
import (
    "context"

    tolvyn "github.com/tolvyn/tolvyn-go"
    tolvyngoogle "github.com/tolvyn/tolvyn-go/google"
)

ctx := context.Background()
client, err := tolvyngoogle.NewClient(ctx, tolvyn.ClientOptions{
    TolvynAPIKey: "tlv_live_...",
})
if err != nil {
    panic(err)
}
defer client.Close()
```

### Constructor signature

Unlike OpenAI and Anthropic, Google requires a `context.Context` and returns an error (the official `genai.NewClient` does both):

```go
func NewClient(ctx context.Context, opts tolvyn.ClientOptions) (*Client, error)
func NewClientFromEnv(ctx context.Context) (*Client, error)
```

`Client` embeds `*genai.Client`. Use methods like `client.GenerativeModel("gemini-1.5-flash")` exactly as you would on the official SDK.

### Known Google limitations

- **No fail-open transport.** The Google sub-package uses a plain header-injection transport (`headerInjectTransport`). When the TOLVYN proxy is unreachable, Google requests fail — they do **not** fall back to `generativelanguage.googleapis.com`. `DisableFailOpen` and `ProviderAPIKey` are silently ignored for Google. Track the SDK roadmap on GitHub.

---

## ClientOptions

All three sub-packages share the same `tolvyn.ClientOptions` struct.

| Field | Type | Default | Description |
|---|---|---|---|
| `TolvynAPIKey` | `string` | `""` (env: `TOLVYN_API_KEY`) | TOLVYN API key (`tlv_live_...` / `tlv_test_...`). Required — see panic note below. |
| `ProxyURL` | `string` | provider-specific default; env: `TOLVYN_PROXY_URL` | Override the proxy URL. Defaults: `https://proxy.tolvyn.io/v1/proxy/openai/`, `.../anthropic/`, `.../google/`. `Defaults()` automatically appends a trailing `/` if missing. |
| `Team` | `string` | `""` | Sent as `X-Tolvyn-Team`. |
| `Service` | `string` | `""` | Sent as `X-Tolvyn-Service`. |
| `Feature` | `string` | `""` | Sent as `X-Tolvyn-Feature`. |
| `Agent` | `string` | `""` | Sent as `X-Tolvyn-Agent`. |
| `User` | `string` | `""` | Sent as `X-Tolvyn-User`. |
| `EndCustomer` | `string` | `""` | Sent as `X-Tolvyn-End-Customer`. |
| `DisableFailOpen` | `bool` | `false` (fail-open enabled) | The zero value enables fail-open — matching Python and Node defaults. Set to `true` to force all requests through TOLVYN. |
| `ProviderAPIKey` | `string` | `""`; env: `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GOOGLE_API_KEY` (selected per sub-package) | Generic provider key used for fail-open fallback. **Note:** this is a single generic field, not separate `OpenAIAPIKey` / `AnthropicAPIKey` fields like other language SDKs. Each sub-package's `Defaults()` reads the appropriate env var if this field is empty. |

Header values are only sent when the corresponding field is non-empty.

### DisableFailOpen zero-value behavior

The boolean defaults to its Go zero value, `false`. That zero value means **fail-open is enabled** when `ProviderAPIKey` is also set. This was a deliberate rename in v0.1.1 — the previous field name `FailOpen bool` had the wrong zero-value semantic (a fresh `ClientOptions{}` would have `FailOpen: false`, disabling the feature).

| `DisableFailOpen` | `ProviderAPIKey` | Behavior |
|---|---|---|
| `false` (zero) | `"sk-..."` | Fail-open enabled |
| `false` (zero) | `""` | No fail-open (no key to fall back with); proxy errors surface |
| `true` | (any) | Fail-open disabled; proxy errors surface |

### `TolvynAPIKey` panic note

If neither the `TolvynAPIKey` field nor the `TOLVYN_API_KEY` environment variable is set, `ClientOptions.Defaults()` calls `panic("tolvyn: TolvynAPIKey is required (or set TOLVYN_API_KEY env var)")`. This is non-idiomatic Go — `NewClient` for OpenAI and Anthropic does not return an `error`, so missing-key configuration crashes the process. Always set the key before calling `NewClient`. The Google constructor returns an error from `genai.NewClient`, but the panic from `Defaults()` happens first.

---

## Attribution headers

Six headers identify who and what made the request:

```go
client := tolvynopenai.NewClient(tolvyn.ClientOptions{
    TolvynAPIKey: "tlv_live_...",
    Team:         "backend",
    Service:      "invoice-summarizer",
    Feature:      "summarize",
    Agent:        "claude-code",
    User:         "alice@company.com",
    EndCustomer:  "acme-corp",
})
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

The TOLVYN proxy strips all six before forwarding the request upstream.

---

## Fail-open behavior

Fail-open is enabled by default for OpenAI and Anthropic clients when `ProviderAPIKey` is set. When the proxy is unreachable, the SDK retries the request directly against the provider using that key.

### When fail-open triggers

Source: `tolvyn.IsProxyError()`. The SDK falls back when either:

- `err != nil` and the error message (lower-cased) contains any of:
  - `connection refused`
  - `no such host`
  - `timeout`
  - `dial tcp`
  - `eof`
  - `reset by peer`
- The response status code is `503 Service Unavailable`

### When fail-open does NOT trigger

Source: `tolvyn.ShouldNotFailOpen()`. The SDK does **not** fall back when the status code is in `[400, 500)` excluding `503`. Real provider errors (`400`, `401`, `403`, `404`, `422`, `429`) propagate as-is.

### Fallback URL composition

Unlike the Python and Node SDKs, the Go SDK **strips the `/v1/proxy/{provider}` prefix from the request path** before building the fallback URL (`tolvyn.go:184-197`):

```go
path := req.URL.Path
for _, prefix := range []string{"/v1/proxy/openai", "/v1/proxy/anthropic", "/v1/proxy/google"} {
    if strings.HasPrefix(path, prefix) {
        path = path[len(prefix):]
        break
    }
}
fallbackURL := strings.TrimSuffix(t.providerURL, "/") + "/" + strings.TrimPrefix(path, "/")
```

This produces correct fallback URLs for Anthropic (where the underlying SDK already builds paths like `/v1/messages`).

**Known issue for OpenAI:** the alpha `openai-go` SDK constructs request paths that, after the proxy-prefix strip, lack the `/v1/` segment OpenAI's API requires. The fallback URL becomes `https://api.openai.com/chat/completions` instead of `https://api.openai.com/v1/chat/completions`. Fail-open for OpenAI may return `404` from OpenAI's real API. Track and verify before relying on it in production; set `DisableFailOpen: true` and monitor TOLVYN uptime if your OpenAI requests must be guaranteed available.

### Metering during fail-open

Requests that fall back to the provider directly **bypass the TOLVYN proxy**. They are not metered, not budget-checked, and not recorded in the ledger for that call. The dashboard shows no row.

### Disabling fail-open

```go
client := tolvynopenai.NewClient(tolvyn.ClientOptions{
    TolvynAPIKey:    "tlv_live_...",
    DisableFailOpen: true,
})
```

---

## Environment variables

`ClientOptions.Defaults()` reads these:

| Variable | Used for | Required |
|---|---|---|
| `TOLVYN_API_KEY` | Fills `TolvynAPIKey` if the field is empty | Yes — panics if also unset |
| `TOLVYN_PROXY_URL` | Fills `ProxyURL` if the field is empty | No |
| `OPENAI_API_KEY` | Fills `ProviderAPIKey` for the openai sub-package | No |
| `ANTHROPIC_API_KEY` | Fills `ProviderAPIKey` for the anthropic sub-package | No |
| `GOOGLE_API_KEY` | Fills `ProviderAPIKey` for the google sub-package (currently unused — see Google limitations) | No |

The selection of which env var maps to `ProviderAPIKey` is done by each sub-package: `openai.NewClient` calls `opts.Defaults("OPENAI_API_KEY", ...)`, anthropic uses `"ANTHROPIC_API_KEY"`, google uses `"GOOGLE_API_KEY"`.

---

## Context usage

The Go SDK follows standard Go conventions:

```go
ctx := context.Background()
// or with a deadline:
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

resp, err := client.Chat.Completions.New(ctx, params)
```

The TOLVYN wrapper does not consume or override the context — it is passed straight through to the underlying SDK. Cancellation, deadlines, and request-scoped values all propagate.

Google's `NewClient` requires a context at construction time because `genai.NewClient` does:

```go
ctx := context.Background()
client, err := tolvyngoogle.NewClient(ctx, tolvyn.ClientOptions{
    TolvynAPIKey: "tlv_live_...",
})
defer client.Close()
```

---

## `NewClientFromEnv` convenience constructors

Each sub-package exposes a zero-argument convenience that reads everything from environment variables:

```go
import (
    tolvynopenai "github.com/tolvyn/tolvyn-go/openai"
    tolvynanthropic "github.com/tolvyn/tolvyn-go/anthropic"
    tolvyngoogle "github.com/tolvyn/tolvyn-go/google"
)

oaiClient := tolvynopenai.NewClientFromEnv()        // reads TOLVYN_API_KEY, OPENAI_API_KEY
anthClient := tolvynanthropic.NewClientFromEnv()    // reads TOLVYN_API_KEY, ANTHROPIC_API_KEY
googClient, err := tolvyngoogle.NewClientFromEnv(context.Background())  // requires ctx
```

These are equivalent to passing the corresponding env vars into `NewClient`. Attribution headers (team, service, etc.) can't be set this way — use `NewClient` with explicit options when you need them.

---

## Google notes (vs. Python and Node)

The Go Google sub-package is closer to OpenAI/Anthropic in shape than the Python/Node Google clients:

- Each `Client` instance has its own HTTP transport — **no process-wide global state** (unlike the Python Google SDK, which calls `genai.configure()` globally).
- `Client` embeds `*genai.Client` directly. All `genai` methods (`GenerativeModel`, `ListModels`, etc.) are available without forwarding.
- `Close()` is exposed (from the embedded `*genai.Client`) — call `defer client.Close()` to release resources.

The fail-open gap is the same across all three language SDKs: no Google fail-open is implemented yet.

---

## Version and changelog

| Version | Notes |
|---|---|
| v0.1.1 | Current. Renamed `FailOpen bool` → `DisableFailOpen bool` so the zero value enables fail-open by default. Added `BoolPtr` helper (currently unused). |
| v0.1.0 | Initial release. OpenAI + Anthropic + Google. Google fail-open not implemented. |

**Note on the `Version` constant:** the source still declares `const Version = "0.1.0"` in `tolvyn.go`. The published tag is `v0.1.1`. The constant will be bumped in the next release.

The SDK is in early development. APIs may change between minor versions until 1.0.

---

## See also

- [Integration Modes](../integration-modes.md) — SDK mode vs proxy mode trade-offs
- [Quickstart](../getting-started/quickstart.md) — end-to-end first request walkthrough
- [Migrate from OpenAI direct](../migration/from-openai-direct.md)
- [Python SDK](python.md) · [Node.js SDK](nodejs.md)
