# CLI Reference

The `tolvyn` CLI manages TOLVYN from the terminal — authentication, cost summaries, live request tail, kill switches, budgets, reconciliation, audit log export.

Current version: **1.0.0**.

---

## Installation

```bash
curl -fsSL https://releases.tolvyn.io/install.sh | sh
```

The install script drops a single binary at `/usr/local/bin/tolvyn` (or `~/.local/bin/tolvyn` if root is not writable). To verify:

```bash
tolvyn version
```

```
tolvyn version 1.0.0
```

---

## Configuration

### Config file

`~/.tolvyn/config.json` is created on first `tolvyn init` / `tolvyn login`. File permissions are `0600`; directory permissions are `0700`.

```json
{
  "api_url": "https://api.tolvyn.io",
  "token": "eyJhbGciOiJIUzI1NiIsInR...",
  "default_environment": "production"
}
```

| Field | Type | Default | Description |
|---|---|---|---|
| `api_url` | string | `https://api.tolvyn.io` | The management API base URL |
| `token` | string | — | JWT received from `POST /v1/auth/login`. Omitted (empty string) when logged out |
| `default_environment` | string | `production` | Default `environment` for new API keys |

### Global flags

These flags apply to every command:

| Flag | Default | Description |
|---|---|---|
| `--api-url <url>` | from config | Override the API URL for one invocation |
| `--json` | `false` | Print raw JSON response instead of formatted tables |
| `--no-color` | `false` | Disable ANSI colors. Also auto-disabled when stdout is not a TTY |

---

## Commands

### `tolvyn init`

Interactive setup: prompts for API URL, email, and password, then logs in. Equivalent to `tolvyn login` plus a config write for the API URL.

```bash
tolvyn init
```

```
API URL [https://api.tolvyn.io]: 
Email: alice@acme.com
Password: 
Authenticated. Config saved to /home/alice/.tolvyn/config.json
```

### `tolvyn login`

Authenticate with email and password. Stores the resulting JWT in `~/.tolvyn/config.json`.

```bash
tolvyn login
```

```
Email: alice@acme.com
Password: 
Authenticated. Config saved to /home/alice/.tolvyn/config.json
```

### `tolvyn logout`

Clear the stored token. Does not revoke the JWT server-side — use `tolvyn keys revoke <id>` for that.

```bash
tolvyn logout
```

```
Logged out.
```

### `tolvyn status`

Check API connectivity, database status, server version, latency, and the local auth state.

```bash
tolvyn status
```

```
API:      https://api.tolvyn.io  [OK]
Database: ok
Version:  1.0.0
Latency:  42ms
Auth:     Authenticated as: alice@acme.com
```

Exits non-zero if the API returns a non-200 status.

---

### `tolvyn tail`

Stream live AI requests through a Server-Sent Events connection. Heartbeats every 15s. Reconnects up to 3 times on connection drop.

| Flag | Default | Description |
|---|---|---|
| `--team <name>` | — | Server-side filter by team name |
| `--service <name>` | — | Server-side filter by service name |
| `--model <substr>` | — | Filter by model (client-side substring match) |
| `--min-cost <usd>` | `0` | Hide requests below this USD threshold (client-side) |
| `--no-alerts` | `false` | Suppress `alert` events from the stream |

```bash
tolvyn tail --team backend --min-cost 0.01
```

```
TIME     | TEAM/SERVICE           | MODEL            |   TOKENS |     COST | LATENCY
─────────┼────────────────────────┼──────────────────┼──────────┼──────────┼─────────
15:48:01 | backend/summarizer     | gpt-4o           |      210 |  $0.0084 |   412ms
15:48:04 | backend/embeddings     | text-embedding-3 |    1,024 |  $0.0001 |    52ms
```

Press `Ctrl+C` to disconnect.

---

### `tolvyn cost`

Show spend summary. With no flags, returns an organization-wide summary with the top 10 models and top 10 teams.

| Flag | Default | Description |
|---|---|---|
| `--from <date>` | — | Start date `YYYY-MM-DD` |
| `--to <date>` | — | End date `YYYY-MM-DD` |
| `--team <id>` | — | Filter by team ID |
| `--model <id>` | — | Filter by model |
| `--by <dim>` | — | Group by `user`, `customer`, or `agent` |

```bash
tolvyn cost --from 2026-05-01 --to 2026-05-17
```

```
Period:              2026-05-01 to 2026-05-17
Total Spend:         $124.50
Total Requests:      18,432
Avg Cost/Req:        $0.0068

Top Models:
  gpt-4o               $98.20      78.9%  12,180 reqs
  gpt-4o-mini          $18.10      14.5%   5,400 reqs

Top Teams:
  backend              $84.10      67.6%
  data-platform        $32.40      26.0%
```

`--by model`, `--by team`, and `--by service` switch to single-dimension tables (model / team / service breakdown). `--by user` and `--by customer` switch to per-user / per-end-customer tables. `--by agent` currently prints a v1.4 notice and exits successfully.

---

### `tolvyn requests`

List recent proxied requests.

| Flag | Default | Description |
|---|---|---|
| `--team <id>` | — | Filter by team ID |
| `--model <id>` | — | Filter by model |
| `--from <date>` | — | Start date `YYYY-MM-DD` |
| `--to <date>` | — | End date `YYYY-MM-DD` |
| `--limit <n>` | `20` | Rows to return (max 100) |

```bash
tolvyn requests --model gpt-4o --limit 5
```

```
TIME              | TEAM/SERVICE     | MODEL          |  TOKENS |     COST | LATENCY
2026-05-17 15:48  | backend/summari… | gpt-4o         |     210 |  $0.0084 |   412ms
2026-05-17 15:47  | backend/summari… | gpt-4o         |     185 |  $0.0073 |   389ms
...
```

#### `tolvyn requests show <id>`

Full detail for a single request — every field plus headers, cost breakdown, and ledger linkage. Output is always JSON since the detail blob varies by provider.

```bash
tolvyn requests show c4d8e2f1-5b3a-4c7e-8f9d-1a2b3c4d5e6f
```

```json
{
  "id": "c4d8e2f1-...",
  "provider": "openai",
  "model_id": "gpt-4o-2024-08-06",
  "model_family": "gpt-4o",
  "tokens_input": 120,
  "tokens_output": 35,
  "cost_usd": "0.0084",
  "team_id": "b3e2c8a4-...",
  "service_name": "summarizer",
  "tolvyn_user": "alice@company.com",
  "ledger_sequence": 1843,
  "record_hash": "a1b2c3d4..."
}
```

---

### `tolvyn keys`

Manage TOLVYN API keys.

#### `tolvyn keys list`

```bash
tolvyn keys list
```

```
NAME             PREFIX               ENV            LAST USED
backend-prod     tlv_live_aB3x...     production     2026-05-17 15:50
ci-test          tlv_test_kQ7p...     test           2026-05-15 09:12
```

#### `tolvyn keys create`

| Flag | Default | Description |
|---|---|---|
| `--name <name>` | — (required) | Human label |
| `--env <env>` | `production` | `production` or `test` — controls the prefix |
| `--team <id>` | — | Scope to a team |

```bash
tolvyn keys create --name backend-prod
```

```
New API key created: backend-prod

tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ

Save this key — it will not be shown again.
```

#### `tolvyn keys revoke <id>`

Prompts for confirmation, then revokes the key.

```bash
tolvyn keys revoke c4d8e2f1-5b3a-4c7e-8f9d-1a2b3c4d5e6f
```

```
Revoke key c4d8e2f1-5b3a-4c7e-8f9d-1a2b3c4d5e6f? [y/N]: y
Key revoked.
```

---

### `tolvyn providers`

Manage provider API keys (OpenAI / Anthropic / Google).

#### `tolvyn providers list`

```bash
tolvyn providers list
```

```
PROVIDER      ID                                    VER    ADDED
openai        5a7c1f3d-8b22-4e9f-b1c4-2a8d6e3c4f5a  1      2026-05-17 14:00
anthropic     b6e9c2a3-7d11-4a8e-9c0f-3b7d2c4e5f6a  1      2026-05-17 14:02
```

#### `tolvyn providers add <provider>`

`<provider>` must be one of `openai`, `anthropic`, `google`. Prompts for the API key (hidden input).

```bash
tolvyn providers add openai
```

```
Enter openai API key (input hidden): 
✓ openai provider key stored (id: 5a7c1f3d-8b22-4e9f-b1c4-2a8d6e3c4f5a)
```

#### `tolvyn providers delete <id>` (alias `rm`)

Prompts for confirmation, then revokes the provider key.

```bash
tolvyn providers delete 5a7c1f3d-8b22-4e9f-b1c4-2a8d6e3c4f5a
```

```
Delete provider key 5a7c1f3d-8b22-4e9f-b1c4-2a8d6e3c4f5a? [y/N]: y
✓ Provider key 5a7c1f3d-... deleted.
```

---

### `tolvyn teams`

#### `tolvyn teams list`

```bash
tolvyn teams list
```

```
NAME                 COST CENTER    CREATED
Backend              CC-1042        2026-05-01
Data Platform        —              2026-05-03
```

#### `tolvyn teams create`

| Flag | Default | Description |
|---|---|---|
| `--name <name>` | — (required) | Team name |
| `--cost-center <code>` | — | Optional cost center code |

```bash
tolvyn teams create --name "Frontend" --cost-center CC-1099
```

---

### `tolvyn budgets`

#### `tolvyn budgets list` (alias `ls`)

```bash
tolvyn budgets list
```

```
SCOPE                  MODE   LIMIT        SPENT        UTIL     PERIOD
team (Backend)         hard   $1,000       $843.20      84.3%    monthly
service (chatbot-api)  soft   $500         $120.50      24.1%    monthly
agent (refactor-bot)   hard   $200         $182.00      91.0%    monthly
```

Utilization is color-coded: green (<75%), yellow (75–90%), red (≥90%).

#### `tolvyn budgets create` (alias `set`)

| Flag | Default | Description |
|---|---|---|
| `--scope <scope>` | `org` | `org`, `team`, `service`, or `agent` |
| `--team <name>` | — | Required when `--scope=team` |
| `--service <name>` | — | Required when `--scope=service` |
| `--agent <name>` | — | Required when `--scope=agent` (matches `X-Tolvyn-Agent`) |
| `--amount <usd>` | `0` (required) | Budget limit in USD; must be > 0 |
| `--period <period>` | `monthly` | `monthly`, `weekly`, or `daily` |
| `--mode <mode>` | `soft` | `soft` (alert only) or `hard` (block at proxy) |

```bash
tolvyn budgets create --scope team --team backend --amount 1000 --mode hard
```

```
Budget created: $1000.00 monthly hard limit on team backend
```

#### `tolvyn budgets delete <id>` (aliases `rm`, `remove`)

Prompts for confirmation.

```bash
tolvyn budgets delete e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b
```

---

### `tolvyn kill`

Emergency kill switch. With flags, activates a kill. Subcommands: `list`, `undo`. The bare `tolvyn kill` is equivalent to `tolvyn kill activate`.

#### Activation flags

| Flag | Default | Description |
|---|---|---|
| `--scope <scope>` | — (required) | `team`, `service`, `agent`, `api_key`, or `all` |
| `--target <value>` | — | Target name (team, service, agent, key); required unless `--scope=all` |
| `--reason <text>` | — | Free-text reason recorded in the audit log |

```bash
tolvyn kill --scope team --target backend --reason "runaway agent loop"
```

```
Kill team "backend"? This will immediately block all AI requests. [y/N]: y
✓ Kill switch activated. All matching AI requests are now blocked.
  ID:    a1b2c3d4-...
  Scope: team / backend
  Reason: runaway agent loop

To undo: tolvyn kill undo a1b2c3d4-...
```

While active, matching proxy requests return HTTP 451.

#### `tolvyn kill list` (alias `ls`)

```bash
tolvyn kill list
```

```
ID         SCOPE    TARGET     REASON                 ACTIVATED
a1b2c3d…   team     backend    runaway agent loop     2026-05-17 15:55

1 active kill switch(es)
```

#### `tolvyn kill undo <id>` (aliases `deactivate`, `rm`)

```bash
tolvyn kill undo a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

```
✓ Kill switch a1b2c3d4-... deactivated. AI requests are flowing again.
```

---

### `tolvyn models`

List available models with current pricing.

| Flag | Default | Description |
|---|---|---|
| `--provider <name>` | — | Filter by `openai`, `anthropic`, or `google` |

```bash
tolvyn models --provider openai
```

```
MODEL                                    PROVIDER     FAMILY             IN/MTok    OUT/MTok
gpt-4o                                   openai       gpt-4o             $2.5000    $10.0000
gpt-4o-mini                              openai       gpt-4o-mini        $0.1500    $0.6000
gpt-3.5-turbo                            openai       gpt-3.5-turbo      $0.5000    $1.5000

3 model(s)
```

#### `tolvyn models diff`

Show pricing changes for models the tenant has used.

| Flag | Default | Description |
|---|---|---|
| `--last <duration>` | `30d` | Look-back window (e.g. `7d`, `30d`, `90d`, `24h`) |
| `--provider <name>` | — | Filter by provider |

```bash
tolvyn models diff --last 30d --provider openai
```

```
MODEL                PROVIDER     FIELD    BEFORE       AFTER        CHANGE%  IMPACT       DATE
gpt-4o               openai       input    $5.0000      $2.5000      -50.00%  $42.10 saved 2026-05-01
o3-mini              openai       output   $4.4000      $4.4000      0.00%    $0.00        2026-04-20

Total impact: -$42.10
```

#### `tolvyn models changes`

Currently active pricing-change notifications for models the tenant has used. Distinct from `diff`: `changes` is the live "do I need to react?" view; `diff` is the historical audit log over a lookback window.

```bash
tolvyn models changes
```

```
MODEL                   PROVIDER    FIELD      BEFORE        AFTER         CHANGE%   EFFECTIVE
gpt-4o                  openai      input      $2.5000       $2.0000       -20.00%   2026-06-01
claude-haiku-4-5        anthropic   output     $1.2500       $1.5000       +20.00%   2026-05-25
```

Empty output (`No active pricing changes for models you use.`) means every model you currently use is at its latest published price.

---

### `tolvyn requests`

(See above — same flags as `tolvyn cost` minus `--by`, with `--limit`.)

---

### `tolvyn usage`

#### `tolvyn usage export`

Export request log as CSV. Server returns up to 10,000 rows.

| Flag | Default | Description |
|---|---|---|
| `--from <date>` | — | Start date `YYYY-MM-DD` |
| `--to <date>` | — | End date `YYYY-MM-DD` |
| `--team <id>` | — | Team ID filter |
| `--service <name>` | — | Service name filter |
| `--model <id>` | — | Model ID filter |
| `-o`, `--output <path>` | — | Write to file instead of stdout |

```bash
tolvyn usage export --from 2026-05-01 --to 2026-05-17 --output may-usage.csv
```

---

### `tolvyn reconcile`

Reconcile provider invoices against the TOLVYN ledger.

#### `tolvyn reconcile upload <file.csv>`

| Flag | Default | Description |
|---|---|---|
| `--month <YYYY-MM>` | — (required) | Invoice month |
| `--provider <name>` | `openai` | `openai`, `anthropic`, or `google` |

```bash
tolvyn reconcile upload openai-may-2026.csv --month 2026-05 --provider openai
```

```
RUN  2026-05 reconciliation

  Invoice total : $1240.5500
  Tolvyn total  : $1241.1000
  Gap           : -$0.5500
  Matched lines : 12  Unmatched: 0  Shadow AI: 0

MODEL          INV REQ   INV COST     TLVN REQ   TLVN COST    GAP        STATUS
gpt-4o         12180     $980.4500    12180      $980.7000    -$0.2500   matched
...

Run ID: a8d2c4e6-...
```

#### `tolvyn reconcile list` (alias `ls`)

```bash
tolvyn reconcile list
```

```
MONTH    PROVIDER   INV TOTAL    GAP        SHADOW AI   FILE
2026-05  openai     $1240.5500   -$0.5500   0           openai-may-2026.csv
2026-04  openai     $1102.1000   balanced   0           openai-apr-2026.csv
```

#### `tolvyn reconcile delete <run-id>` (alias `rm`)

Prompts for confirmation, then deletes the run.

---

### `tolvyn audit`

View and export the audit log.

| Flag | Default | Description |
|---|---|---|
| `--from <date>` | — | Start date (RFC3339 or `YYYY-MM-DD`) |
| `--to <date>` | — | End date |
| `--last <duration>` | — | Show last N hours/days (`24h`, `7d`, `30d`) |
| `--format <fmt>` | `table` | `table` or `csv` |
| `--limit <n>` | `100` | Max entries |

```bash
tolvyn audit --last 7d
```

```
TIME                     ACTOR    ACTOR ID                              ACTION                           IP
2026-05-17 15:55:00      user     9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1  kill_switch.activated            1.2.3.4
2026-05-17 14:00:00      user     9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1  api_key.created                  1.2.3.4
...

142 entries
```

CSV export streams directly to stdout (use shell redirection to save):

```bash
tolvyn audit --last 30d --format csv > audit-may.csv
```

---

### `tolvyn ledger`

View, verify, and export the immutable hash-chained cost ledger.

#### `tolvyn ledger list` (alias `ls`)

| Flag | Default | Description |
|---|---|---|
| `--limit <n>` | `50` | Page size |
| `--offset <n>` | `0` | Page offset |
| `--from <date>` | — | Start date (RFC3339 or `YYYY-MM-DD`) |
| `--to <date>` | — | End date |

```bash
tolvyn ledger list --limit 5
```

```
SEQ      REQUEST ID                            PROVIDER   MODEL                   COST        HASH
1843     c4d8e2f1-5b3a-4c7e-8f9d-1a2b3c4d5e6f  openai     gpt-4o-2024-08-06       $0.0084     a1b2c3d4e5f6
1842     b3e2c8a4-1f6d-4a8c-9e0f-7d1c4b2a3e5f  openai     gpt-4o-2024-08-06       $0.0073     9c0d1e2f3a4b
...

Showing 5 of 1843 records (limit=5 offset=0)
```

#### `tolvyn ledger verify`

Walks the chain, re-derives every hash, and validates HMACs. **Exits with code `1` if the chain is broken** — important for CI/audit scripts that need a hard failure signal.

| Flag | Default | Description |
|---|---|---|
| `--from-seq <n>` | `1` | Starting sequence |
| `--to-seq <n>` | current max | Ending sequence |

```bash
tolvyn ledger verify
```

```
✓ Chain verified — 1843 records, all hashes valid
```

On failure:

```
✗ Chain broken at sequence 1043 — seq 1043: record_hash mismatch (stored a1b2c3..., derived d4e5f6...)
Records checked before break: 1042
```

(Exit code: `1`.)

Use in CI / audit scripts:

```bash
tolvyn ledger verify || { echo "Ledger broken — investigate"; exit 1; }
```

#### `tolvyn ledger export`

Stream the full ledger as CSV — includes `record_hash`, `previous_hash`, and `hmac_signature` so the export can be verified offline. The fundamental audit-evidence flow: export, hand to auditor with the HMAC secret, auditor verifies independently.

| Flag | Default | Description |
|---|---|---|
| `--from <date>` | — | Start date |
| `--to <date>` | — | End date |
| `-o`, `--output <path>` | — | Write to file (default: stdout) |

```bash
tolvyn ledger export --from 2026-05-01 --to 2026-06-01 -o tolvyn-ledger-may.csv
```

```
✓ Wrote 482113 bytes to tolvyn-ledger-may.csv
```

---

### `tolvyn webhooks` (alias `wh`)

Manage webhook endpoints for alert event delivery.

#### `tolvyn webhooks list` (alias `ls`)

```bash
tolvyn webhooks list
```

```
ID                                    URL                                      EVENTS                          STATUS     LAST DELIVERY
a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d  https://hooks.example.com/tolvyn         alert.all                       success    2026-05-17 15:45
b6e9c2a3-7d11-4a8e-9c0f-3b7d2c4e5f6a  https://pagerduty.example.com/v2         alert.budget_threshold,alert…   failed     2026-05-16 11:20 (disabled)
```

Status is color-coded: green `success`, red `failed`, dash `—` for never-delivered.

#### `tolvyn webhooks create`

| Flag | Default | Description |
|---|---|---|
| `--url <url>` | — (required) | Webhook URL — must start with `https://` |
| `--events <list>` | `alert.all` | Comma-separated event types (`alert.budget_threshold`, `alert.cost_anomaly`, `alert.model_change`, `alert.pricing_change`, `alert.all`) |

```bash
tolvyn webhooks create \
  --url https://hooks.example.com/tolvyn \
  --events alert.budget_threshold,alert.cost_anomaly
```

```
✓ Webhook created
  ID:     a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
  URL:    https://hooks.example.com/tolvyn
  Events: alert.budget_threshold,alert.cost_anomaly

Signing secret (shown ONCE — save it now):
wh_a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0
```

The HMAC secret is the signing key for verifying incoming webhook deliveries. **It is shown only on creation** — store it in your secret manager immediately. There is no retrieve-secret endpoint; if lost, you must delete and recreate the webhook.

#### `tolvyn webhooks delete <id>` (alias `rm`)

Prompts for confirmation, then deletes the endpoint. Pending retries are abandoned.

```bash
tolvyn webhooks delete a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

#### `tolvyn webhooks test <id>`

Trigger a synthetic `webhook.test` delivery against the configured URL. Exits with code `1` if the receiver returns non-2xx or errors.

```bash
tolvyn webhooks test a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

```
✓ Delivered (HTTP 200)
```

#### `tolvyn webhooks deliveries <id>`

Recent (last 50) delivery attempts for the endpoint — status code, attempt count, timestamps, error messages.

```bash
tolvyn webhooks deliveries a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

```
DELIVERY ID                           EVENT                       STATUS   ATTEMPTS  DELIVERED         ERROR
d8e9f0a1-2b3c-4d5e-6f7a-8b9c0d1e2f3a  alert.budget_threshold      200      1         2026-05-17 15:45
c9d0e1f2-3a4b-5c6d-7e8f-9a0b1c2d3e4f  alert.cost_anomaly          503      3         —                 fetch failed
```

---

### `tolvyn alerts` (alias `al`)

Manage alerts (budget thresholds, cost anomalies, model changes, pricing changes).

#### `tolvyn alerts list` (alias `ls`)

| Flag | Default | Description |
|---|---|---|
| `--unread` | `false` | Show only unacknowledged alerts |
| `--type <type>` | — | Filter by alert type (`budget_threshold`, `cost_anomaly`, `model_change`, `pricing_change`) |
| `--severity <sev>` | — | Filter by severity (`info`, `warning`, `critical`) |
| `--limit <n>` | `50` | Page size |

```bash
tolvyn alerts list --unread --severity critical
```

```
ID                                    TYPE                    SEVERITY    TITLE                                              CREATED           STATUS
e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b  budget_threshold        critical    Budget team/backend at 100% utilization            2026-05-17 15:50  unread
f6a9b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c  cost_anomaly            critical    Cost anomaly: summarizer — request 14x average     2026-05-17 14:22  unread

2 alert(s) shown (total 8)
```

Severity is color-coded: red `critical`, yellow `warning`, green `info`.

#### `tolvyn alerts ack <id>` (alias `acknowledge`)

Mark an alert acknowledged. Acknowledged alerts move out of the active list.

```bash
tolvyn alerts ack e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b
```

```
✓ Alert e5f8a1b2-... acknowledged.
```

#### `tolvyn alerts test`

Create a synthetic test alert — verifies the alert pipeline (dashboard visibility, email, webhooks) without waiting for a real threshold event.

```bash
tolvyn alerts test
```

```
✓ Test alert created (id: g7a0b3c4-5d6e-7f8a-9b0c-1d2e3f4a5b6c)
  TOLVYN Alert System Test
```

---

### `tolvyn savings`

View and dismiss savings findings produced by the nightly analyzer.

#### `tolvyn savings list` (alias `ls`)

| Flag | Default | Description |
|---|---|---|
| `--min-savings <usd>` | `0` | Hide findings below this monthly USD threshold |

```bash
tolvyn savings list --min-savings 5
```

```
ID                                    TYPE                        MODEL                   SERVICE             SAVINGS/MO    RECOMMENDATION
e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b  small_token_requests        gpt-4o                  summarizer          $42.20        backend in this period used fewer than 500 tokens (avg 184 tokens). c…
f6a9b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c  underutilized_cache         —                       —                   $84.20        You're paying full input price for repeated prompts on anthropic. Cur…
g7a0b3c4-5d6e-7f8a-9b0c-1d2e3f4a5b6c  duplicate_prompts           —                       —                   $12.10        8.4% of requests this month are duplicate prompts (1247 redundant ca…

Total potential savings: $138.50
```

#### `tolvyn savings dismiss <id>`

Mark a finding dismissed. Dismissed findings stay dismissed forever — they are not regenerated on subsequent nightly runs even if the underlying condition persists.

```bash
tolvyn savings dismiss e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b
```

```
✓ Finding e5f8a1b2-... dismissed.
```

---

### `tolvyn version`

Print the CLI version.

```bash
tolvyn version
```

```
tolvyn version 1.0.0
```

`tolvyn --version` is equivalent.

---

## Output formats

### Default — formatted tables

Color-coded, fixed-width columns. Auto-disables colors when stdout is not a TTY (e.g. piping to `less` or `grep`).

### `--json`

Raw JSON response from the API. Useful for scripting:

```bash
tolvyn cost --from 2026-05-01 --json | jq '.total_cost_usd'
```

### `--no-color`

Disables ANSI color codes. Combine with `--json` only if you want to see ANSI escapes stripped from JSON values (rare).

---

## Environment variables

The CLI itself reads no environment variables directly — all configuration lives in `~/.tolvyn/config.json`. (Compare to the SDKs, which honor `TOLVYN_API_KEY` and `TOLVYN_PROXY_URL`.)

Override the API URL for one invocation with `--api-url`:

```bash
tolvyn --api-url https://api-staging.tolvyn.io status
```

---

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Generic error (network, auth, validation, server 4xx/5xx) |

The CLI does not yet distinguish error categories via exit codes. Parse the error message printed to stderr if you need to act on specific cases in scripts.

---

## See also

- [Quickstart](../getting-started/quickstart.md)
- [REST API Reference](api.md)
- [Integration Modes](../integration-modes.md)
