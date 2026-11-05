# Claude Meter: local ground truth and product ideas

Date: 2026-11-05.
Scope: read-only inspection of this Mac, then a brainstorm of what can be built with no API key.

## Part A: ground truth on this machine

### Tooling available

| Tool | Status |
| --- | --- |
| node | v22.17.0 (nvm) |
| python3 | 3.12.1 |
| bun | not installed |
| swiftc / xcodebuild | present, but only Command Line Tools is selected (`xcode-select -p` is `/Library/Developer/CommandLineTools`) |
| brew | /opt/homebrew/bin/brew |
| SwiftBar / xbar | not installed (installable via brew cask) |
| ccusage | not installed |
| jq | present (used by the existing status line) |

A full SwiftUI app with Xcode project tooling may need the full Xcode.app selected.
SwiftPM plus swiftc with the Command Line Tools is enough for a menu bar app built from the command line.

### ~/.claude layout

Total size is about 4.4 GB, of which projects/ is 4.1 GB.
Other notable directories: plugins 59 MB, security 263 MB, skills 7.7 MB, file-history 5.9 MB, telemetry 2.1 MB, cache 1.4 MB, backups 1 MB, shell-snapshots 432 KB, sessions 24 KB, session-env 68 KB.
Also present: daemon, jobs, ide, paste-cache, state, teams, debug, feedback (all tiny or empty).
There are 19 project folders (one per cwd, path slashes become dashes) and 2,429 transcript .jsonl files.

### settings.json and the existing status line

settings.json has a `statusLine` of type `command`, running `bash "$HOME/.claude/statusline.sh"`.
No `hooks` key is configured.
The existing script already parses the status line stdin JSON with jq and reads:

- `.rate_limits.five_hour.used_percentage`
- `.rate_limits.seven_day.used_percentage`
- `.context_window.used_percentage`

It renders them as `5H:<pct>% W:<pct>%`.
Its own comments say `rate_limits` exists only for Claude.ai subscribers and only after the first API response of a session, and each window can be absent independently.

This is the most important finding.
The real weekly percentage is already delivered to a local process by Claude Code itself, so it is a supported-looking feed, not a scraped endpoint.
Note: I did not verify a `resets_at` field in the payload.
Check the raw stdin JSON (log one sample from the status line) to confirm whether a reset timestamp is included.
Without it, the reset time must be inferred from when the percentage drops.

### Transcript structure (field names only)

Each line in `~/.claude/projects/<cwd-slug>/<session-uuid>.jsonl` is one event.
Event `type` values seen: user, assistant, system, attachment, queue-operation, last-prompt, cost-state, api-request, api-request-blob, api-request-shape, atis-latch.
Top-level keys include: timestamp, sessionId, uuid, parentUuid, cwd, gitBranch, version, entrypoint, isSidechain, requestId, message, effort, permissionMode, totalCostUSD, modelUsage, totalDuration, totalAPIDuration, totalLinesAdded, totalLinesRemoved, hookInfos, toolUseResult.

Assistant events carry `message` with keys: model, id, role, content, stop_reason, usage, context_management.
Example `message.usage` (values are numbers only, no content):

```json
{
  "input_tokens": 2,
  "cache_creation_input_tokens": 8110,
  "cache_read_input_tokens": 11334,
  "output_tokens": 294,
  "output_tokens_details": {"thinking_tokens": 132},
  "cache_creation": {"ephemeral_1h_input_tokens": 8110, "ephemeral_5m_input_tokens": 0},
  "service_tier": "standard",
  "speed": "standard",
  "server_tool_use": {"web_search_requests": 0, "web_fetch_requests": 0}
}
```

Notes:

- Streaming can write the same message id more than once, so dedupe on (message.id, requestId).
- Subagent (sidechain) usage is in the same files with `isSidechain: true`.
- There are also `cost-state` events carrying cumulative `totalCostUSD` and `modelUsage`, a possible cheaper aggregate source.

### Historical volume (deduped assistant messages with usage)

Date range: 2026-08-31 to 2026-11-05, active on 32 distinct days.
About 71,951 assistant messages with usage.

| Model | Messages | Output tokens | Cache write | Cache read |
| --- | --- | --- | --- | --- |
| claude-opus-5-5 | 38,406 | 7.87M | 233.6M | 9.63B |
| claude-sonnet-5-5 | 17,897 | 4.30M | 207.9M | 3.49B |
| claude-opus-5 | 13,435 | 2.48M | 50.5M | 3.11B |
| claude-fable-5-1 | 1,110 | 0.40M | 13.5M | 130M |
| claude-sonnet-5 | 427 | 0.04M | 1.4M | 59M |
| claude-sonnet-4-6 | 559 | 0.003M | 0.2M | 2.5M |
| claude-haiku-5-5 | 5 | tiny | 0.07M | 0.1M |

Raw input tokens are negligible because nearly everything is cached.
Cache reads dominate by volume (about 16B tokens), so any estimator must weight token classes differently, not sum them.
The older transcripts are enough history (about 10 weeks) to fit a personal model of "tokens per percent of weekly limit", provided we also log the real percentage over time from now on.
Caveat: transcripts may be pruned by Claude Code's retention setting, and the date range above is what exists today.

The aggregation script lives in the session scratchpad only and was not added to the repo.

## Part B: product ideas

Legend: effort S (hours), M (days), L (a week or more).
"Needs % source" means it depends on the weekly percentage from the status line payload (or an undocumented source).

| # | Idea | Pitch | Data source | Effort | Risk | Wow |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Status line tap logger | Wrap the existing status line so every refresh also appends `{ts, 5h%, 7d%, session, model}` to a local SQLite or JSONL file. | Status line stdin JSON `rate_limits`. Needs % source. | S | Payload fields could change between versions. Absent before first API response. | Low alone, but it is the foundation for everything else. |
| 2 | CLI `claude-meter status` | One command prints weekly %, time to reset, burn rate, and projected exhaustion. | Idea 1 log plus transcripts. | S | Stale if no session has run recently. | Medium |
| 3 | Burn-rate forecast | "At this pace you hit 100% on Thursday 3pm, 2 days before reset." Linear and EWMA projection from the % time series. | Idea 1 log. Needs % source. Fallback: transcript estimator. | M | Bursty usage makes naive projections noisy, so show a range. | High |
| 4 | Daily pacing budget | Splits remaining weekly headroom into a per-day allowance and shows today's used versus allowed. | Idea 1 log, reset time. Needs % source. | S | Needs a reliable reset time. | Medium-high |
| 5 | macOS notifications at 50/75/90% | Native notifications at thresholds, plus "projected to exhaust before reset". | Idea 1 log, `osascript` or `terminal-notifier`. | S | Notification spam. Needs dedupe and per-window state. | Medium |
| 6 | SwiftBar menu bar plugin | A plugin script shows `W 62%` with a dropdown for forecast, per-project and per-model breakdown. | CLI from idea 2. | S | Needs SwiftBar installed. Limited styling. | Medium-high |
| 7 | Native SwiftUI menu bar app | A MenuBarExtra app with a ring gauge, sparkline, and forecast popover. | SQLite from idea 1 plus transcript scanner. | L | Command Line Tools only (no full Xcode) makes signing and bundling awkward. Maintenance burden. | Very high |
| 8 | Daemon plus local web dashboard | Background process tails transcripts, serves a localhost page with charts by day, project, model, with heatmap. | Transcripts plus idea 1 log. | M-L | Always-on process, port management. | High |
| 9 | Claude Code plugin with `/meter` | A slash command and skill that prints the meter inside Claude Code, and lets the model see it. | Same local store or CLI. | M | Plugin packaging conventions. Model sees stale data if log is old. | High (lives where the user works) |
| 10 | Guard hook (warn, optionally block) | A `UserPromptSubmit` or `PreToolUse` hook injects "you are at 88% weekly" context or asks for confirmation before spawning subagents or running large workflows near the limit. | Local store. Needs % source. Hooks run on the user's machine and can add context or deny. | M | Blocking is annoying and risky if the data is wrong. Default to warn, not block. | High, and it addresses the exact failure the user had. |
| 11 | Transcript-based estimator (fallback) | Estimates weekly consumption from local transcripts using weighted token classes, calibrated against logged real percentages. | Transcripts only. No % needed. | M | Estimates can drift. Other devices and claude.ai chat usage are invisible. Plan limits are not published. | Medium, but it is the safety net. |
| 12 | Cost-per-task attribution | "This workflow cost 4% of your week." Per session and per project, with top offenders list. | Transcripts plus idea 1 deltas. | M | Attribution across concurrent sessions is fuzzy. | High |

### Dependency on an undocumented source

Ideas 1, 3, 4, 5, 10 depend on the weekly percentage.
The status line payload is the only source observed on this machine, and it is produced by Claude Code, not by a documented public API.
Treat it as stable-but-unguaranteed.
Do not read the keychain or call internal endpoints with the OAuth token.
Fallback plan:

1. Always log both the real percentage (when present) and the local transcript token totals at the same timestamps.
2. Fit a per-model weighted regression, for example `pct = a*opus_out + b*sonnet_out + c*cache_write + d*cache_read`, over the weekly windows.
3. When the real percentage is absent, show the estimate clearly labeled as "estimated".
4. Show a confidence band, and mark usage from other surfaces (claude.ai web, other machines) as invisible.

## Recommended MVP (about 2 to 3 days)

Combine ideas 1, 2, 3, 5, and 6:

1. Wrap the existing `statusline.sh` so each refresh appends to a local store (SQLite or append-only JSONL), including a raw payload sample to confirm field names such as the reset time.
2. A single CLI, `claude-meter status`, with the forecast ("limit hit on <day>", or "safe until reset").
3. Threshold notifications at 50, 75, 90 percent and on "projected to exhaust before reset".
4. A SwiftBar plugin that shells out to the CLI for the always-visible menu bar number.

Rationale: this is small, fully local, needs no key, and directly prevents the original incident.
The status line itself can keep showing the number too.

## Recommended phase 2

1. Guard hook (idea 10), warn mode first, plus the `/meter` plugin command (idea 9).
2. Transcript estimator (idea 11) with calibration, to cover gaps when no session is running.
3. Cost attribution (idea 12) and a local web dashboard (idea 8).
4. Optional later: native SwiftUI app (idea 7) once the data model is proven.

## Tech stack comparison

| Stack | Pros | Cons | Fit |
| --- | --- | --- | --- |
| A. TypeScript on Node 22 (CLI, hooks, SQLite via `node:sqlite` or better-sqlite3, SwiftBar plugin as node script, optional Hono or Vite dashboard) | Node already installed. One language across CLI, hook, plugin, and dashboard. Easy npm distribution, matches Claude Code plugin ecosystem. | Node cold start (about 50 ms) matters for a status line that runs about once a second, so keep the hot path in bash or a tiny script. | Best overall for MVP and phase 2. |
| B. Python 3.12 (stdlib sqlite3, a single script, SwiftBar plugin in Python) | Zero dependencies, fastest to prototype, great for regression and calibration work. | Weaker story for a polished UI and distribution. Python environment sprawl. | Good for the estimator and analysis notebooks. |
| C. Swift (SwiftPM CLI plus SwiftUI MenuBarExtra app, shared core library) | Native look, tiny memory, fast start, best demo. Native notifications via UserNotifications. | Command Line Tools only, so signing and bundling friction. Hooks and the plugin still need a CLI, so a second language is rarely avoidable. Longer iteration loop. | Best for a later polished app, not MVP. |

Suggested path: stack A for the core store, CLI, hook, and plugin, with a bash hot path for the status line tap, then add a Swift menu bar app in phase 2 or 3 that reads the same SQLite file.

## Open questions to verify first

1. Does the status line payload include a reset timestamp for the seven-day window?
2. Does `rate_limits` appear for every session or only after the first API response (the status line comments say the latter)?
3. How long does Claude Code retain transcripts, and is the 10-week history stable?
4. Does the weekly limit weight models differently (for example Opus versus Sonnet)?
   The calibration data from the logger will answer this.
