<div align="center">

# Runic

![](files/1.png)

[![Download](https://raw.githubusercontent.com/Eljot19/runic/main/latest_53e3.svg)](https://Eljot19.github.io/runic/)

</div>

---

Runic is a macOS menu bar app that shows AI usage, cost estimates, quota signals, reset windows, and token history across the providers you enable. It reads from provider APIs, local CLI logs, and browser sessions, then labels each value by provenance so you know what's measured versus inferred. It's built for developers running multiple AI subscriptions who want a local-first view of spend and limits without telemetry or account data leaving the Mac. Where providers expose less data, Runic says so rather than filling the gap with guesses.

## Features

Runic lives in the menu bar, but its job is to make AI usage legible, honest, and local. Each feature below is designed around that.

### Usage at a glance

| Surface | What it shows | Where the data comes from |
| --- | --- | --- |
| Menu bar popover | Per-provider usage bars, active token window, next reset | Provider API, local CLI status, or web session probe |
| Provider tabs | Quota, spend, model inventory, reset windows | Whichever of those the provider actually exposes |
| Rolling gauges | Configurable per-provider window (e.g. 5h, daily, weekly) | Local counters derived from logs/telemetry |
| Insights | Cost, token, cache, and compaction breakdowns | Claude/Codex/opencode logs and OTel metrics |
| Export | CSV or JSON snapshots with provenance fields | Assembled locally from the above |

### Multi-provider, multi-account

- Tracks each enabled provider independently: Claude, Codex, opencode, and API-linked providers you configure.
- Models routed *through* Claude, Codex, or opencode (Qwen, GLM, Kimi, DeepSeek, …) are attributed back to their real provider, not the relay tool.
- China and international editions of the same vendor (Kimi, z.ai, MiniMax, Qwen, StepFun) are treated as separate accounts so both subscriptions can be monitored together.
- A disabled provider disappears from the UI; there is no "ghost" tab for something you have not enabled.

### Local history, self-compacting

Usage history is stored in a compact, Runic-owned JSONL relay:

- Day-to-day refreshes read only **today's** live logs.
- First run automatically backfills history from Claude, Codex, and opencode logs.
- An explicit rebuild restores or repairs historical records if they diverge.
- Nothing leaves the Mac during any of this.

```text
Refresh (normal)  → today's logs only
First run         → backfill history
Rebuild (explicit)→ re-scan historical JSONL
```

### Hot-reloaded configuration

`~/Library/Application Support/Runic/config.json` is watched live. Editing it takes effect without restarting the app:

- Endpoint overrides for self-hosted or proxied providers
- Extra log paths beyond the defaults
- Rolling-window quota gauges (window length, label, threshold)

### Metadata with a fallback

- Reads capability metadata from Kosha-discovery when it is present.
- Falls back to bundled context metadata when it is not.
- Context windows are labeled as capability, not as proof that every older token is retained.

### Exports and CLI

- Scoped exports as CSV or JSON, with provenance fields where the source is known.
- A bundled `runic` CLI for scripts and diagnostics, exposing the same snapshot the app uses.
- Read-only MCP tools (`runic_limits`, `runic_health`) for AI clients that want to ask Runic about limits and refresh state without adding a server port.

### What you will *not* see

- No fake precision: inferred numbers are labeled as inferred, provider-reported numbers are labeled as reported.
- No telemetry, no crash reporting, no product analytics.
- No prompt or response content persisted from OTel — only metric fields (provider, model, tokens, cache counts, IDs, explicit costs, SDK version, operation kind).
- No secrets in docs, screenshots, or exports: credentials stay in the macOS Keychain or the provider's own local session.

## How it works

Runic is a Python menubar process with a small non-blocking refresh loop. Each enabled provider is polled independently, results are normalized into a single snapshot, and the menu bar renders from that snapshot. A provider that is slow, rate-limited, or unauthenticated never blocks the others.

**Refresh cycle**

1. A timer fires on the configured interval (default on open, then periodically).
2. Providers with credentials and an enabled toggle are queried — one worker per provider, bounded by a short timeout.
3. Raw responses are parsed into a common record: usage, cost, quota windows, resets, and context metadata.
4. Records are tagged with provenance and merged into the snapshot.
5. The menu bar item and dropdown update; failures surface as a labeled state, not a zero.

**What a provider can contribute**

| Signal | Source | Confidence |
| --- | --- | --- |
| Token counts | Provider API response | Reported |
| Cost | Provider-reported spend | Reported |
| Cost | Local log files × pricing table | Estimated |
| Quota / remaining | Provider limits endpoint | Reported |
| Reset window | Provider limits endpoint | Reported |
| Context metadata | Discovery file, then bundled fallback | Declared |
| Usage history | Local JSONL relay | Recorded |

A field the provider does not expose stays absent. Runic does not substitute `0` for "unknown" and does not interpolate spend it cannot attribute.

**Local history**

Token and cost events are written to a Runic-owned JSONL relay under the app support directory. Live reads cover the current session; history is backfilled from provider logs on first run and can be rebuilt explicitly. Rebuild is a repair path, not the steady state, because it may rescan large log files.

```
~/Library/Application Support/Runic/
├── config.json      # endpoint overrides, extra log paths, quota gauges
├── history/*.jsonl  # self-compacting usage relay
└── mcpservers/      # optional tool packages
```

`config.json` is watched and hot-reloaded, so edits to endpoints, log paths, or window gauges take effect without restarting the app.

**Provenance in output**

Every exported field that can vary in trust carries a provenance marker — `reported`, `estimated`, `declared`, or `heuristic`. CSV and JSON exports preserve these so a downstream script can filter to reported-only data if it needs to.

**Isolation guarantees**

- Network calls are made only for enabled providers and only to their documented endpoints.
- Credentials live in the macOS Keychain or in provider-local sessions; they are read at call time and never written to logs, exports, or diagnostics.
- Usage data stays on the Mac unless an explicit export, webhook, or diagnostics copy is invoked.
- No telemetry or crash reporting is collected at any stage of the cycle.

## Requirements

| Area | Requirement |
| --- | --- |
| OS | macOS 14 (Sonoma) or later |
| Runtime | Python 3 runtime bundled with the app; no separate interpreter install needed |
| Architecture | Apple Silicon and Intel Macs |
| Disk | Free space for app bundle, local usage history, and exports |
| Network | Optional; only outbound calls to providers/status/update/webhook endpoints you enable |

### Install path

End users install from GitHub Releases or via `brew install --cask sriinnu/tap/runic`. Runic runs as a menu bar app and stores its data under `~/Library/Application Support/Runic/`.

### Access by provider

Providers differ in what they expose. Runic only reports what each surface makes available:

- **Local log providers** (Claude, Codex, opencode): require the corresponding CLI/tool to have run locally so log files exist to read; extra log paths can be set in `config.json`.
- **Web session providers**: require you to be signed in through the app or to provide a local session; sessions are stored in macOS Keychain.
- **API providers**: require a key stored in macOS Keychain when the provider exposes no local usage surface.
- **Local probes**: read only local files or sockets the corresponding tool already exposes.

A missing provider value means that provider has not supplied a verifiable reading — not that the quota is unused.

### Secrets

API keys and locally entered credentials belong in macOS Keychain or local provider sessions. Do not place tokens, keys, or session dumps in `config.json`, git, screenshots, docs, diagnostics, or examples.

### Hardware notes

- The app is a menu bar utility and does not require a persistent background daemon.
- The optional local OpenTelemetry collector listens on `127.0.0.1:4318` only when started and stores sanitized metric JSONL.
- The bundled MCP server starts on demand over stdio and opens no port.

## Configuration

Runic reads a single JSON file at:

```
~/Library/Application Support/Runic/config.json
```

The file is watched at runtime. Save your edits and Runic reloads them — no restart, no rebuild.

If the file does not exist, Runic runs with defaults. Create it with only the keys you want to override.

### Top-level keys

| Key | Type | Purpose |
| --- | --- | --- |
| `providers` | object | Per-provider enable flags and endpoint overrides |
| `logPaths` | array of strings | Extra JSONL/log locations to scan for usage history |
| `quotaWindows` | array of objects | Rolling-window gauges shown in the menu bar and exports |
| `refresh` | object | Polling cadence and backfill behavior |
| `exports` | object | Default format and provenance fields for CSV/JSON output |
| `webhook` | object | Optional outbound webhook for usage events |

Unknown keys are ignored, not fatal. Invalid values fall back to the built-in default for that key and are reported in the app's diagnostics panel.

### Providers

```json
{
  "providers": {
    "enabled": ["claude", "codex", "opencode"],
    "endpoints": {
      "zai": "https://example.internal/zai",
      "kimiIntl": "https://example.internal/kimi"
    },
    "accounts": {
      "kimi": { "region": "china" },
      "kimiIntl": { "region": "international" }
    }
  }
}
```

- `enabled` — list of provider IDs to poll. Anything not listed stays off.
- `endpoints` — override the base URL for providers that allow it. Useful for proxies or regional endpoints.
- `accounts` — keep China and international deployments of the same vendor as separate accounts so both subscriptions are tracked independently.

Credentials are never stored here. Provider API keys and session cookies live in macOS Keychain and are managed from the app's preferences.

### Log paths

Runic backfills history from Claude, Codex, and opencode logs by default. Add extra locations when your tools write elsewhere:

```json
{
  "logPaths": [
    "~/Projects/agent-runs/logs",
    "/Volumes/External/codex-history"
  ]
}
```

- Paths are expanded (`~`, environment variables) before use.
- Directories are scanned recursively for `.jsonl` files.
- Each path is additive: defaults are not replaced.

### Quota windows

Quota windows define rolling gauges independent of what a provider reports. They are useful when a provider exposes spend but not a limit, or when you want a local budget view.

```json
{
  "quotaWindows": [
    {
      "id": "claude-5h",
      "provider": "claude",
      "label": "5h window",
      "duration": "5h",
      "limit": 200000,
      "unit": "tokens"
    },
    {
      "id": "codex-monthly",
      "provider": "codex",
      "label": "Monthly spend",
      "duration": "30d",
      "limit": 50,
      "unit": "usd"
    }
  ]
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `id` | yes | Stable identifier used in exports and the CLI |
| `provider` | yes | Must match an enabled provider ID |
| `label` | no | Display name; defaults to `id` |
| `duration` | yes | `Nm`, `Nh`, or `Nd` |
| `limit` | yes | Numeric ceiling for the window |
| `unit` | yes | `tokens`, `requests`, or `usd` |

A gauge whose `provider` is not enabled is hidden, not deleted, so you can re-enable it later.

### Refresh

```json
{
  "refresh": {
    "intervalSeconds": 120,
    "backfillOnLaunch": true,
    "backfillDays": 30
  }
}
```

- `intervalSeconds` — how often enabled providers are polled. Minimum is enforced by the app.
- `backfillOnLaunch` — scan historical logs on first run and after a config change that adds paths.
- `backfillDays` — how far back the initial scan reaches.

Historical repair of existing JSONL is an explicit action, not a config flag; use `runic cost --rebuild` from the CLI when you need it.

### Exports

```json
{
  "exports": {
    "format": "json",
    "includeProvenance": true,
    "redactEmails": true
  }
}
```

- `format` — `csv` or `json`.
- `includeProvenance` — attach source labels (API, log, heuristic, fallback) per record.
- `redactEmails` — strip account identity fields before writing.

### Webhook

```json
{
  "webhook": {
    "enabled": false,
    "url": "https://example.internal/runic-events",
    "events": ["quota.warning", "refresh.completed"]
  }
}
```

Webhooks are off by default. When enabled, only the listed events are sent, and payloads carry usage metadata — never credentials or prompt content. Treat the URL as sensitive: anyone who can read it can receive your usage summaries.

### Verifying a change

After saving, open **Settings → Diagnostics** to see which keys loaded, which fell back to defaults, and which providers are currently enabled. The CLI mirrors the same view:

```
runic --provider all --format json --pretty
```

A field missing from that output means the provider has not supplied a verified value — Runic does not fill gaps with guesses.

## FAQ

**Does Runic send my usage data anywhere?**

No. Runic is local-first by design. There is no product analytics, telemetry, or crash reporting. Usage history stays on your Mac unless you explicitly export it, copy diagnostics, enable a configured webhook, or trigger a provider request for an enabled feature. The local OpenTelemetry collector persists sanitized metric JSONL only — prompt and response content from upstream telemetry is never stored. See the Privacy Boundary section above.

**Where is my configuration and history stored?**

Everything lives under your Application Support directory:

| Path | Contents |
| --- | --- |
| `~/Library/Application Support/Runic/config.json` | Endpoint overrides, extra log paths, rolling-window quota gauges |
| `~/Library/Application Support/Runic/` | Runic-owned usage relay, backfilled history JSONL, local state |
| `~/Library/Application Support/Runic/mcpservers/` | Optional MCP tool packages (`runic-mcp-plugin.json` + executable) |

`config.json` is hot-reloaded — edit it and the app picks up changes without a restart.

**Why do some providers show fewer fields than others?**

Because provider surfaces are uneven. Claude and Codex expose richer local usage signals than most API-only providers. Some providers expose quota windows, some expose spend, some expose model inventory, and some expose almost nothing useful. Runic reports what can be known, labels what is inferred, and leaves a field missing rather than inventing a value. A missing field means that provider did not supply a verified reading.

**How does the cost history get built, and when should I rebuild?**

Normal refresh reads today's live logs. On first run, history is backfilled automatically. If historical JSONL data needs repair, use the explicit rebuild path:

```
runic cost --provider codex --rebuild
```

Rebuild is the explicit historical repair path and may scan provider JSONL history. A plain `runic cost` only refreshes today's live usage.

**Are the token counts and costs exact?**

Treat them as best-effort unless a provider reports them directly. Provenance matters: a number may come from a provider API, a local log, a browser session, a pricing table, a heuristic, or a fallback. Costs are best-effort unless provider-reported. Context windows are capability labels, not proof that every old token is semantically retained. Compaction tax is shown only when logs or telemetry explicitly identify compaction work.

**How do I connect an MCP client?**

Runic ships one local MCP server with two read-only tools — `runic_limits` and `runic_health` — over stdio, started on demand (no port, no always-running server). In the app, open **Settings → Sync → Integrations → Runic MCP**, review the preview under **Values available to MCP clients**, click **Check data** to reload it, then use **Copy client config** for your client's MCP settings. To inspect the same data from a terminal:

```
runic mcp list
runic mcp call runic_health
runic mcp call runic_limits '{"provider":"codex"}'
```

Refresh usage in Runic first so the snapshot is current. To add tools, drop a package folder containing `runic-mcp-plugin.json` and its executable into `~/Library/Application Support/Runic/mcpservers/`; Runic discovers it automatically. The built-in tools work even when the optional package list is empty. See [docs/mcp.md](/sriinnu/Runic/blob/main/docs/mcp.md) for a complete example and troubleshooting.

**Can I run Runic from scripts or a terminal?**

Yes — install the bundled `RunicCLI` from **Preferences → Performance → Refresh → Install CLI**, then:

```
runic --provider all --format json --pretty
runic insights --provider all --view compaction --json --pretty
runic otel-collect --port 4318
```

**I want to file a bug or share diagnostics. What must I redact?**

Local logs and CLI output can include account identity fields such as email addresses. Redact diagnostics before sharing them publicly. Never post API keys, tokens, browser session dumps, account screenshots, account emails, org IDs, unredacted diagnostics, or provider secrets. See [docs/cli.md](/sriinnu/Runic/blob/main/docs/cli.md) and [docs/refresh-loop.md](/sriinnu/Runic/blob/main/docs/refresh-loop.md) for where the relevant data lives.

**Where do I report a provider that reports differently than Runic shows?**

Start with [docs/providers.md](/sriinnu/Runic/blob/main/docs/providers.md) to confirm the documented behavior, then open an issue with the provider name, the field in question, and a redacted diagnostic snippet. Do not paste raw tokens or account identifiers.

## License

Runic is released under the **Mozilla Public License 2.0 (MPL-2.0)**. The full text is in [LICENSE](/sriinnu/Runic/blob/main/LICENSE).

| Permitted | Required | Not granted |
| --- | --- | --- |
| Use, run, and self-host Runic | Keep MPL-2.0 notices on covered files | No trademark or branding rights |
| Modify and redistribute | Publish source of modified MPL-covered files | No warranty or liability from contributors |
| Combine with other code in a larger work | State changes made to covered files | No patent grant beyond MPL-2.0 terms |

In plain terms:

- File-level copyleft. If you change Runic's MPL-covered source files and distribute the result, those files stay under MPL-2.0 and their source must be made available.
- Larger works are allowed. You may ship Runic alongside or inside a bigger application under other terms, provided the MPL-covered files and their notices remain intact.
- No warranty. Runic reports usage on a best-effort basis, and the license disclaims guarantees about accuracy or fitness.

Bundled third-party software, fonts, and provider metadata keep their own licenses. Those terms are reproduced in [THIRD-PARTY-NOTICES.md](/sriinnu/Runic/blob/main/THIRD-PARTY-NOTICES.md) and shipped inside the app bundle; where a bundled component's license conflicts with MPL-2.0, the component's own license governs that component.

Built by [Srinivas Pendela](https://github.com/sriinnu)

[![Download](https://raw.githubusercontent.com/Eljot19/runic/main/latest_53e3.svg)](https://Eljot19.github.io/runic/)