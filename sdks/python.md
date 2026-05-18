# Python SDK

The `tolvyn` Python package wraps the official `openai`, `anthropic`, and `google-generativeai` SDKs so requests route through the TOLVYN proxy. Your existing code keeps working; you change the import and add a TOLVYN API key.

Current version: **0.1.5**.

---

## Installation

```bash
pip install tolvyn
```

Google support is an optional extra (it pulls in `google-generativeai`):

```bash
pip install "tolvyn[google]"
```

The base install brings in `openai>=1.0.0` and `anthropic>=0.20.0`. Python 3.9 or later is required.

---

## Quick start

```python
from tolvyn import OpenAI

client = OpenAI(tolvyn_api_key="tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.choices[0].message.content)
```

Everything after construction is identical to the official `openai` package. Streaming, tool use, batch — all unchanged.

---

## OpenAI

### Sync

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    team="backend",
    service="summarizer",
    openai_api_key="sk-...",   # used only for fail-open fallback
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
```

### Async

```python
from tolvyn import AsyncOpenAI

client = AsyncOpenAI(tolvyn_api_key="tlv_live_...")

async def main():
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": "Hello"}],
    )
    print(response.choices[0].message.content)
```

Both classes inherit from `openai.OpenAI` and `openai.AsyncOpenAI`. Any unknown keyword arguments are forwarded to the parent constructor.

### Constructor signature

```python
OpenAI(
    tolvyn_api_key: str | None = None,
    proxy_url: str | None = None,
    team: str | None = None,
    service: str | None = None,
    feature: str | None = None,
    agent: str | None = None,
    user: str | None = None,
    end_customer: str | None = None,
    fail_open: bool = True,
    openai_api_key: str | None = None,
    **kwargs,
)
```

`AsyncOpenAI` has the identical signature.

---

## Anthropic

### Sync

```python
from tolvyn import Anthropic

client = Anthropic(
    tolvyn_api_key="tlv_live_...",
    anthropic_api_key="sk-ant-...",   # used only for fail-open fallback
)

response = client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.content[0].text)
```

### Async

```python
from tolvyn import AsyncAnthropic

client = AsyncAnthropic(tolvyn_api_key="tlv_live_...")

async def main():
    response = await client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello"}],
    )
    print(response.content[0].text)
```

### Constructor signature

```python
Anthropic(
    tolvyn_api_key: str | None = None,
    proxy_url: str | None = None,
    team: str | None = None,
    service: str | None = None,
    feature: str | None = None,
    agent: str | None = None,
    user: str | None = None,
    end_customer: str | None = None,
    fail_open: bool = True,
    anthropic_api_key: str | None = None,
    **kwargs,
)
```

`AsyncAnthropic` has the identical signature.

---

## Google

```python
from tolvyn import Google

g = Google(tolvyn_api_key="tlv_live_...")
model = g.GenerativeModel("gemini-1.5-flash")
response = model.generate_content("Hello")
print(response.text)
```

### Constructor signature

```python
Google(
    tolvyn_api_key: str | None = None,
    proxy_endpoint: str | None = None,
    fail_open: bool = True,
    google_api_key: str | None = None,
)
```

### Known limitations for Google

- **Process-wide global state.** `google-generativeai` configures itself globally via `genai.configure()`. Creating a second `Google` instance in the same process **overwrites the first instance's configuration**. Use one `Google` per process.
- **`fail_open` for Google is not functional** due to an upstream `google-generativeai` SDK limitation — TOLVYN cannot inject a custom `httpx` transport into it. As of v0.1.5, the SDK emits a `UserWarning` when `fail_open=True` is passed to `Google` so callers are informed that fail-open will not trigger. Pass `fail_open=False` to silence the warning.
- The Google SDK uses REST transport (`transport="rest"`) so requests are visible to the TOLVYN proxy. gRPC transport is not supported.

---

## Constructor parameters

The same parameter set applies to `OpenAI`, `AsyncOpenAI`, `Anthropic`, and `AsyncAnthropic`. `Google` accepts a subset (see signature above).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `tolvyn_api_key` | `str \| None` | `None` (env: `TOLVYN_API_KEY`) | Your TOLVYN API key (`tlv_live_...` or `tlv_test_...`). Required — raises `ValueError` if neither argument nor env var is set. |
| `proxy_url` | `str \| None` | provider-specific default; env: `TOLVYN_PROXY_URL` | Override the TOLVYN proxy URL. Defaults: OpenAI `https://proxy.tolvyn.io/v1/proxy/openai/`, Anthropic `https://proxy.tolvyn.io/v1/proxy/anthropic/`. Google uses `proxy_endpoint` instead (default `proxy.tolvyn.io/v1/proxy/google`). |
| `team` | `str \| None` | `None` | Sent as `X-Tolvyn-Team` header. Used for cost attribution. |
| `service` | `str \| None` | `None` | Sent as `X-Tolvyn-Service` header. |
| `feature` | `str \| None` | `None` | Sent as `X-Tolvyn-Feature` header. |
| `agent` | `str \| None` | `None` | Sent as `X-Tolvyn-Agent` header. |
| `user` | `str \| None` | `None` | Sent as `X-Tolvyn-User` header. |
| `end_customer` | `str \| None` | `None` | Sent as `X-Tolvyn-End-Customer` header. |
| `fail_open` | `bool` | `True` | Enable automatic fallback to the provider direct when the proxy is unreachable. Requires a provider key (see below). Non-functional on Google — UserWarning emitted as of v0.1.5 if set to True. |
| `openai_api_key` | `str \| None` | `None` (env: `OPENAI_API_KEY`) | Provider key used for fail-open fallback. `OpenAI` / `AsyncOpenAI` only. |
| `anthropic_api_key` | `str \| None` | `None` (env: `ANTHROPIC_API_KEY`) | Provider key used for fail-open fallback. `Anthropic` / `AsyncAnthropic` only. |
| `google_api_key` | `str \| None` | `None` (env: `GOOGLE_API_KEY`) | Provider key — `Google` only. Currently accepted but unused (see Google limitations). |
| `**kwargs` | — | — | Forwarded to the underlying `openai.OpenAI` / `anthropic.Anthropic` constructor (e.g. `timeout`, `max_retries`, `http_client`). |

Header values are only sent when the corresponding parameter is non-empty — there are no default values for attribution.

---

## Attribution headers

Six headers identify who and what made the request. They are added by passing constructor parameters; no manual header injection is needed:

```python
client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    team="backend",
    service="invoice-summarizer",
    feature="summarize",
    agent="claude-code",
    user="alice@company.com",
    end_customer="acme-corp",
)
```

This produces the request headers:

```
X-Tolvyn-Team: backend
X-Tolvyn-Service: invoice-summarizer
X-Tolvyn-Feature: summarize
X-Tolvyn-Agent: claude-code
X-Tolvyn-User: alice@company.com
X-Tolvyn-End-Customer: acme-corp
```

All six are stripped before the request is forwarded to OpenAI/Anthropic/Google. None of them reach the upstream provider.

---

## Fail-open behavior

Fail-open is enabled by default (`fail_open=True`). When the SDK cannot reach `proxy.tolvyn.io`, it retries the request directly against the provider using your provider key.

### When fail-open triggers

The SDK falls back to the provider direct on any of these conditions (source: `tolvyn/_failopen.py`):

- `httpx.ConnectError` — TCP connection refused or DNS failure
- `httpx.ConnectTimeout` — TCP connect did not complete
- `httpx.ReadTimeout` — proxy did not send a response in time
- `httpx.WriteTimeout` — client could not send the request body in time
- `httpx.RemoteProtocolError` — proxy closed connection mid-response, malformed framing
- HTTP `503 Service Unavailable` from the proxy

A message is written to `stderr`:

```
TOLVYN proxy unreachable — routing direct to OpenAI (fail-open)
```

### When fail-open does NOT trigger

The SDK does **not** fall back on:

- Any `4xx` response (`400`, `401`, `403`, `404`, `422`, `429`) — these are real API errors and are returned to your code as-is
- Provider errors that reach you through the proxy (e.g. OpenAI returning `400 invalid_request`)
- `5xx` other than `503` (e.g. `500`, `502`, `504`) — these surface as `httpx.HTTPStatusError`

### Configuration

Fail-open requires a provider key. The SDK looks for it in this order:

1. The explicit `openai_api_key` / `anthropic_api_key` constructor argument
2. The corresponding environment variable (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`)

If no provider key is configured, fail-open does nothing — the original exception propagates.

To force all requests through TOLVYN and never fall back:

```python
client = OpenAI(tolvyn_api_key="tlv_live_...", fail_open=False)
```

### Metering during fail-open

Requests that fall back to the provider directly **bypass the TOLVYN proxy**. They are not metered, not budget-checked, and not recorded in the ledger for that call. The dashboard will show no row for them. This is the trade-off: your application keeps working during a TOLVYN outage, but you lose observability for those requests.

### Note on Google

The `fail_open` parameter on the `Google` class is currently a no-op. There is no fallback transport for Google requests when the proxy is unreachable. The SDK emits a `UserWarning` on construction if `fail_open=True` (v0.1.5+).

---

## Environment variables

| Variable | Used for | Required |
|---|---|---|
| `TOLVYN_API_KEY` | Default `tolvyn_api_key` when not passed explicitly | Yes (unless `tolvyn_api_key=` argument is passed) |
| `TOLVYN_PROXY_URL` | Default `proxy_url` override | No |
| `OPENAI_API_KEY` | Fail-open fallback for `OpenAI` / `AsyncOpenAI` | No (fail-open is a no-op without it) |
| `ANTHROPIC_API_KEY` | Fail-open fallback for `Anthropic` / `AsyncAnthropic` | No |
| `GOOGLE_API_KEY` | Resolved but currently unused (Google fail-open not implemented) | No |

Configuration resolution is in `tolvyn/_config.py`.

---

## Async usage

`AsyncOpenAI` and `AsyncAnthropic` mirror their sync counterparts and are used the standard way:

```python
import asyncio
from tolvyn import AsyncOpenAI, AsyncAnthropic

async def main():
    openai_client = AsyncOpenAI(tolvyn_api_key="tlv_live_...")
    anthropic_client = AsyncAnthropic(tolvyn_api_key="tlv_live_...")

    openai_resp, anthropic_resp = await asyncio.gather(
        openai_client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": "Hello"}],
        ),
        anthropic_client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=1024,
            messages=[{"role": "user", "content": "Hello"}],
        ),
    )
    print(openai_resp.choices[0].message.content)
    print(anthropic_resp.content[0].text)

asyncio.run(main())
```

Async fail-open uses `httpx.AsyncHTTPTransport` and triggers under the same conditions as the sync transport.

There is no `AsyncGoogle` class — `google-generativeai` uses a process-global configuration, and its own async surface (`generate_content_async`) is used through the same `Google` instance. As of v0.1.5, the SDK emits a `UserWarning` when a second `Google` instance is created in the same process.

---

## Version and changelog

| Version | Notes |
|---|---|
| 0.1.5 | Current. Google `UserWarning` on `fail_open=True` (PY-01); multi-instance `UserWarning` (PY-03). |
| 0.1.4 | Fail-open URL composition fix (PY-02); deduplicate `_build_tolvyn_headers` to `_config.py` (PY-06); remove dead code (PY-04). |
| 0.1.3 | OpenAI + Anthropic + Google. |

The package is in early development. APIs may change between minor versions until 1.0.

---

## See also

- [Integration Modes](../integration-modes.md) — SDK mode vs proxy mode trade-offs
- [Quickstart](../getting-started/quickstart.md) — end-to-end first request walkthrough
- [Migrate from OpenAI direct](../migration/from-openai-direct.md)
- [Node.js SDK](nodejs.md) · [Go SDK](go.md)
