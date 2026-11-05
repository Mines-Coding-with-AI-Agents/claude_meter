# Existing Claude Code usage and limit trackers

Research date: 2026-11-05.
Star counts come from GitHub page fetches on that date and are approximate.
GitHub pages did not expose last-commit dates to the fetcher, so "last activity" is given as commit count only; verify recency before depending on any project.
Several projects are tiny and brand new, so treat them as design references rather than dependencies.

## Key platform fact: the official data source now exists

Claude Code's status line receives JSON on stdin that includes `rate_limits.five_hour` and `rate_limits.seven_day`, each with `used_percentage` (0 to 100) and `resets_at` (Unix epoch seconds).
Source: https://code.claude.com/docs/en/statusline
The field is present only for claude.ai Pro, Max, and Ultra subscribers, only after the first API response in a session, and each window may be absent independently.
Claude Code drops a window once its `resets_at` has passed.
Source: https://code.claude.com/docs/en/statusline
This means the real weekly percentage and reset time are available without any undocumented endpoint, but only while a Claude Code session is running and has made a request.
It is a push feed, so a tracker needs to persist snapshots itself to cover idle periods.

## Projects

### ccusage
- URL: https://github.com/ryoppippi/ccusage
- Stack: TypeScript/Node, with a Rust directory in the repo.
  Stars about 19.3k, 2,912 commits.
- Data: local JSONL parsing, with LiteLLM pricing for cost.
- Weekly limit: no.
  It offers weekly aggregation of tokens and cost and a 5-hour blocks report, but the page does not claim progress against the real limit.
- Alerts: none mentioned.
- UI: CLI, plus a beta `statusline` command.
- Strengths: the de facto standard, multi-agent support (Claude Code, Codex, OpenCode, Gemini CLI), JSON output, offline mode.
- Gaps: cost-centric, not limit-centric.
  No real percentage, no reset time, no alerts.

### Claude-Code-Usage-Monitor
- URL: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor
- Note: the owner is "Maciek-roboblog", not "Maciek-roboto".
- Stack: Python 3.9+. Stars about 8.8k, 170 commits.
- Data: local Claude config JSONL, plus official `rate_limits` via a statusline hook (`--statusline`).
  An experimental OAuth usage API is opt-in via `--api`.
- Weekly limit: seven-day percentages appear only when official statusline data is available.
  Otherwise it falls back to labeled estimates, with P90-of-history custom limits and fixed Pro estimates.
- Alerts: `--once` exit codes (10 near limit, 11 limit hit, 20 indeterminate, 30 error), terminal title updates, a plan auto-switch notification.
- UI: Rich TUI, one-shot snapshot, compact single line, JSON/text/CSV export, burn-rate and pace forecasting views.
- Strengths: the closest to what Claude Meter wants (forecasting plus official data plus provenance labels per number).
  Scriptable exit codes.
- Gaps: terminal only, GUIs are explicitly left to companion projects.
  Weekly pacing is not clearly a first-class feature.
  No push notifications.
- Source: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor and https://gittrend.io/repo/Maciek-roboblog/Claude-Code-Usage-Monitor (about 8.6k stars per that tracker).

### tddworks/ClaudeBar
- URL: https://github.com/tddworks/ClaudeBar
- Stack: Swift 6.2, macOS 15+. Stars about 1.6k, 1,750 commits.
- Data: not documented on the README page.
  Another search summary says it shows session, weekly, and model-specific percentages (https://awesome.ecosyste.ms/projects/github.com%2Ftddworks%2FClaudeBar).
  Treat mechanism as unverified.
- Weekly limit: shows per-provider quotas with reset countdown.
- Alerts: the richest found.
  Notification before you run out, healthy/warning/critical/depleted levels, pace-aware colors, Slack or custom webhooks, an iPhone Lock Screen companion.
- UI: menu bar (up to three providers), notch, Touch Bar.
- Strengths: has a "forecast" showing whether a quota lasts until reset at the current pace.
  Multi-provider (Claude, Codex, Gemini, Copilot, Cursor, OpenRouter).
  Extensions via scripts.
- Gaps: broad and heavy, not focused on weekly planning.
  No per-day budget or "hit limit on Thursday" statement is documented.

### GordonBeeming/claude-bar
- URL: https://github.com/GordonBeeming/claude-bar
- Stack: Swift/SwiftUI MenuBarExtra, macOS 15+ Apple Silicon.
  Stars 4, 35 commits.
- Data: undocumented Anthropic usage endpoint, polled about once a minute with Claude Code's OAuth token read read-only from Keychain.
  Token counts from `~/.claude/projects` transcripts.
- Weekly limit: real limits with reset time in local timezone.
- Alerts: icon turns orange or red on API warning level, or on user thresholds.
  Optional OTLP JSON telemetry of limit resets and severity changes.
- UI: menu bar.
- Strengths: telemetry history, fuel-tank (remaining) mode, careful token handling.
- Gaps: Keychain prompts, depends on an undocumented endpoint, no projection.
  Source: https://github.com/GordonBeeming/claude-bar

### pedroprates/claude-code-usage (CC Usage Tracker)
- URL: https://github.com/pedroprates/claude-code-usage
- Stack: Swift/SwiftUI. Stars 2, 16 commits.
  Alpha.
- Data: statusline bridge writes `rate_limits` to `~/.claude/cc-usage-tracker/state.json`, which the app reads.
  Chains to any existing statusline command.
- Weekly limit: real, shown as a health dot (weekly) next to a 5-hour percent.
- Alerts: warn 60 percent and danger 85 percent defaults.
  Optional notification at the danger threshold, off by default.
- UI: menu bar.
- Strengths: the safest architecture (official feed, no credentials).
  Good reference for the bridge pattern.
- Gaps: static thresholds only, no pace or projection.
  README says the percentages are approximations.

### bezlant/climeter
- URL: https://github.com/bezlant/climeter
- Stack: Swift 5.9/SwiftUI. Stars 0, 142 commits.
- Data: statusline exporter script snapshot for Claude, no OAuth credential by default.
  Codex via its auth file and usage endpoint.
- Weekly limit: real 5-hour and 7-day.
- Alerts: none for usage, only update notices.
- UI: menu bar progress bar, peak-hours indicator with a countdown.
- Strengths: peak-hours indicator is a unique idea.
- Gaps: no alerting.
  Source: https://github.com/bezlant/climeter

### NoobyGains/claude-pulse
- URL: https://github.com/NoobyGains/claude-pulse
- Stack: single-file Python 3.8+, no dependencies.
  Stars about 471, 234 commits.
- Data: status line stdin for main bars (no network).
  Usage API optional for extra credits and per-model weekly caps.
- Weekly limit: real session and weekly bars with reset timers.
- Alerts: none.
- UI: status line, 48 widgets, themes.
- Strengths: popular, flexible, per-model weekly caps, multi-account rows.
- Gaps: display only, no projection or alerts.
  Source: https://github.com/NoobyGains/claude-pulse

### ncreasor/claude-usage (Claude Usage Monitor)
- URL: https://github.com/ncreasor/claude-usage
- Stack: not stated on the page.
  Stars 10, 62 commits.
- Data: reads the claude.ai session cookie from the browser cookie DB (Chrome, Brave, Arc, Safari) and calls `/api/organizations/{id}/usage`.
  This is scraping of a private web API.
- Weekly limit: real.
  Also has history charts and an extra-usage toggle.
- Alerts: none.
- UI: menu bar with themes.
- Gaps: cookie extraction is fragile and sensitive, no alerts.
  Source: https://github.com/ncreasor/claude-usage

### TylerGallenbeck/claude-code-limit-tracker
- URL: https://github.com/TylerGallenbeck/claude-code-limit-tracker
- Stack: Python (inferred from file names).
  Stars 22, 13 commits.
- Data: timestamps in `~/.claude/projects/` converted to session hours and prompt counts, with tier presets.
- Weekly limit: estimate only, with separate Sonnet and Opus weekly estimates.
- Alerts: none.
- UI: status line.
- Gaps: estimate, likely superseded by the official `rate_limits` field.
  Source: https://github.com/TylerGallenbeck/claude-code-limit-tracker

### ohugonnot/claude-code-statusline
- URL: https://github.com/ohugonnot/claude-code-statusline
- Stack: bash.
  Stars not checked.
- Data: reads rate limits from statusline stdin when available, usage API fallback.
- UI: status line.
  Alerts not verified.
- Source: https://github.com/ohugonnot/claude-code-statusline

### Others surfaced but not verified in depth
- ClaUse Bar, Mac App Store app, claude.ai login, 5-hour and 7-day percentages, threshold notifications: https://apps.apple.com/app/id6759294136
- claude-usage-bar on PyPI, Python menu bar using the existing Claude Code login: https://pypi.org/project/claude-usage-bar/
- ClaudeUsageBar on Product Hunt, threshold notifications: https://www.producthunt.com/products/claudeusagebar
- EtanHey/ClaudeBar (fork of tddworks with themes): https://github.com/EtanHey/ClaudeBar
- I did not find verified SwiftBar, Raycast, Waybar, tmux, or browser-extension projects in the time budget.
  The search query combined these terms and returned only menu bar apps.
  This is a coverage gap in this survey.
- I did not search Hacker News or Reddit successfully within the time budget.
  This is another gap.

## Comparison table

| Project | Stack | Stars (approx) | Data source | Real weekly % and reset | Alerts | UI |
|---|---|---|---|---|---|---|
| ccusage | TS/Node, Rust | 19.3k | Local JSONL | No (token and cost aggregates) | None | CLI, beta statusline |
| Claude-Code-Usage-Monitor | Python | 8.8k | JSONL, statusline, optional OAuth API | Yes when statusline data exists, else estimate | Exit codes, title, forecast | TUI, CLI |
| tddworks/ClaudeBar | Swift | 1.6k | Not documented | Yes (per README features) | Notifications, pace colors, webhooks | Menu bar, notch |
| claude-pulse | Python | 471 | Statusline stdin, optional API | Yes | None | Status line |
| TylerGallenbeck limit tracker | Python | 22 | Local JSONL | Estimate | None | Status line |
| ncreasor/claude-usage | unstated | 10 | Browser cookie, claude.ai API | Yes | None | Menu bar |
| GordonBeeming/claude-bar | Swift | 4 | OAuth token, usage endpoint | Yes | Thresholds, telemetry | Menu bar |
| pedroprates CC Usage Tracker | Swift | 2 | Statusline bridge | Yes | 60/85 percent, opt-in notify | Menu bar |
| bezlant/climeter | Swift | 0 | Statusline exporter | Yes | None | Menu bar |

## Gaps and opportunities for Claude Meter

1. Weekly pacing is mostly missing.
   Almost every tool shows "X percent used" and a static threshold.
   Only Claude-Code-Usage-Monitor and ClaudeBar mention forecasting, and neither documents a clear weekly "you run out on Thursday" message.
   Claude Meter can compute burn rate against time-to-reset and state a projected exhaustion day and time.
2. Ideal-pace comparison.
   With `seven_day.resets_at` known, the window start is reset minus 7 days.
   Compare actual percent against the linear expected percent at this moment, and show "ahead of pace by N points".
3. Per-day budget.
   Convert remaining percent and remaining days into "about N percent per day left", and flag today's overshoot.
   No surveyed tool does this.
4. Alerting beyond fixed thresholds.
   Fire on projected exhaustion before reset (for example "will hit limit in under 24 hours at current rate"), not only on crossing 85 percent.
   Offer native macOS notifications with de-duplication, and optionally a webhook as ClaudeBar does.
5. Use the official status line feed, not cookies or the OAuth endpoint.
   This avoids Keychain prompts and credential handling, as climeter and CC Usage Tracker show.
   Add a chained statusline wrapper so the user's existing status line keeps working.
6. Persist history.
   The feed only appears during an active session and after the first response.
   Store timestamped snapshots (SQLite or JSONL) so projections and "last known" values survive idle periods, and mark stale data clearly, as Claude-Code-Usage-Monitor does with provenance labels.
7. Handle window quirks.
   Windows may be absent or dropped after `resets_at`, so the tracker needs explicit states (unknown, stale, reset just occurred) rather than showing zero.
8. Weekly reset awareness.
   Combine reset time with usage patterns (for example, heavy Monday and light Friday) to suggest when to slow down.
9. Keep scope narrow. ccusage and ClaudeBar are broad multi-provider tools.
   A focused weekly-limit guardian for one subscription is a real niche.
10. Honest accuracy labels.
    Distinguish official percentages from estimates everywhere.
   The estimate-only tools (TylerGallenbeck, ccusage) are already stale next to the official field.
