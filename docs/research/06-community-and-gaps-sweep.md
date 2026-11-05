# Community and gaps sweep

Research date: 2026-11-05.
Tools used: WebSearch, WebFetch, and authenticated `gh` (search and readme fetch only).
No third-party code was cloned, built, or run.
Fetched content is treated as untrusted data.
Time-boxed sweep, so depth is uneven.
Star counts and `pushedAt` dates come from `gh search repos` on 2026-11-05 (GitHub API, high confidence for the numbers, but `pushedAt` is last push, not last release).
Projects already covered in docs/research/02-existing-projects.md are not repeated.

## 1. Additional public tools

The space is crowded: `gh search repos` returns dozens of near-identical "Claude usage menu bar" repos, most with 0 to 3 stars.
Source: https://github.com/search?q=claude+usage+menubar&type=repositories (confidence: high).
Almost all read either the OAuth token from the macOS Keychain and call an undocumented usage endpoint, or the `statusLine` hook.
None found uses the `claude -p` `rate_limit_event` probe as its main source (confidence: medium, based on READMEs and search summaries only).

### Notable repos

| Repo | Stars | Last push | Platform | Data source | Notable features |
|---|---|---|---|---|---|
| richhickson/claudecodeusage | 340 | 2026-11-03 | macOS menu bar | Keychain OAuth token, internal usage API | Session, weekly, per-model, overage; green/yellow/red at 70/90 percent; multi-account via `CLAUDE_CONFIG_DIR`; 5 min refresh plus refresh on wake; opt-in session-needs-attention alerts with click-to-focus Terminal tab |
| SlavomirDurej/claude-usage-widget | 342 | 2026-11-03 | Windows, macOS, Linux (Electron-style widget) | claude.ai session credentials, encrypted storage | 7-day history graph, threshold notifications, compact mode, tray, taskbar percentage on Windows |
| JuanjoFuchs/ccburn | 97 | 2026-10-06 | TUI/CLI, npm, PyPI, WinGet | Claude Code usage limits | Burn-up charts for session, weekly, and monthly windows; compact mode for status bars; JSON output |
| gxjansen/claude-code-meter | 93 | 2026-05-19 | macOS Uebersicht widget | Keychain OAuth token | Bar color reflects pace (usage rate vs elapsed window time), not absolute fill; "remaining" vs "used" toggle |
| abhishekray07/claude-meter | 88 | 2026-04-01 | Local proxy plus web dashboard | Intercepts Claude Code traffic and captures rate-limit headers | Alpha research tool that estimates the hidden 5h and 7d budget bands; name collision with this project |
| Nihondo/AgentLimits | 70 | 2026-10-31 | macOS menu bar plus Notification Center widgets | Internal APIs (`claude.ai/api/organizations/{orgId}/usage`, ChatGPT wham usage) via in-app login | Codex and Claude together, custom usage scripts for other services, used/remaining toggle |
| mryll/claudebar | 63 | 2026-11-05 | Waybar and Omarchy (Linux) | Usage API | Pace indicators, per-model limits, extra usage spend, CSS classes `low/mid/high/critical`, JSON output |
| bozdemir/claude-usage-widget | 76 | 2026-10-29 | Cross-platform desktop widget plus CLI | Not verified | Opt-in Codex support |
| dependentsign/ClaudeUsageWidget | 56 | 2026-09-06 | macOS WidgetKit widget | Not verified | Native desktop widget with reset times |
| utaysi/claude-usage-widget | 35 | 2026-06-22 | Android home screen widget | Not verified | 5h and weekly on a phone home screen |
| efim0v/brow | 29 | 2026-10-21 | macOS notch | Not verified | Usage in the notch plus parallel session workspaces |
| bzvstudio/claude-usage-raycast-extension | 1 | 2026-11-02 | Raycast menu bar | Not verified | Raycast ring for 5h and weekly |

Sources for the table (confidence: high for numbers, medium for data-source column):
- https://github.com/richhickson/claudecodeusage
- https://github.com/SlavomirDurej/claude-usage-widget
- https://github.com/JuanjoFuchs/ccburn
- https://github.com/gxjansen/claude-code-meter
- https://github.com/abhishekray07/claude-meter
- https://github.com/Nihondo/AgentLimits
- https://github.com/mryll/claudebar
- https://github.com/bozdemir/claude-usage-widget
- https://github.com/dependentsign/ClaudeUsageWidget
- https://github.com/utaysi/claude-usage-widget
- https://github.com/efim0v/brow
- https://github.com/bzvstudio/claude-usage-raycast-extension

### Other categories

- SwiftBar: a Zenn write-up describes a single-file plugin combining 5h, weekly, and a design-quota line, polling the OAuth usage API, and reports HTTP 429 problems from aggressive polling.
  Source: https://zenn.dev/yktsnet/articles/202604-claude-usage-swiftbar?locale=en (confidence: medium, via search summary).
- SwiftBar `claude-quota` reads the Keychain token read-only and warns that the endpoint is internal and may break.
  Source: https://github.com/grzegorz-raczek-unit8/claude-quota (confidence: medium, via search summary).
- VS Code: "Usage Bar for Claude Code" shows 5h and 7d in the status bar from the `statusLine` hook, and says it is unofficial.
  Source: https://marketplace.visualstudio.com/items?itemName=kino-poi.usage-bar-for-claude-code (confidence: medium).
- UsageBar (Product Hunt, macOS): 5h and weekly, reset timers, notifications; users asked for weekly in the menu bar instead of 5h.
  Source: https://www.producthunt.com/products/usagebar (confidence: medium).
  Its blog also publishes guides on weekly vs 5-hour behavior: https://usagebar.com/blog/claude-code-weekly-limit-vs-5-hour-lockout (not verified).
- Statusline-bridge macOS app "CC Usage Tracker" (alpha) uses a bridge invoked every assistant turn rather than log parsing.
  Source: https://github.com/pedroprates/claude-code-usage is a different project; the bridge app was only seen in a search summary, so confidence is low.
- Alfred and tmux/starship: no dedicated tool found in this sweep (confidence: low, absence of evidence only).
- Homebrew casks: Uebersicht is installed via `brew install --cask ubersicht` in gxjansen's README; no usage-tracker cask was verified.

### Gap read

- Pace-colored display (color from rate vs elapsed time, not raw fill) exists in gxjansen/claude-code-meter and mryll/claudebar, so it is validated but not novel.
- Almost every tool depends on an undocumented endpoint or the Keychain OAuth token, which several READMEs flag as fragile.
- Few tools persist history across idle periods; SlavomirDurej's 7-day graph and ccburn's burn-up charts are the main examples.
- Nobody found uses a dedicated probe; this is the genuine differentiator, with the caveats in section 3.

## 2. Community discussion

Official weekly caps were announced in mid-2025 and the Hacker News thread captured the reaction.
Source: https://hn.nuxt.dev/item/44713757 (confidence: high that these themes appear; only the first 100k of about 306k characters was read).

How people got burned:
- Weekly caps stacked on the 5-hour limit, so hitting one could mean no access for the rest of the week.
- Large refactors that read many files burned usage quickly.
- Compacting context near the end of a 5-hour window consumed the remaining allowance.
- A looping session burned a lot before the user noticed.
- Pro users reported hitting limits in 30 minutes to two hours.

Wished-for features, in the commenters' words:
- A usage meter showing closeness to each limit.
- Warnings before hitting a limit.
- Predictable, documented limits.
- Deferred or scheduled jobs for off-peak use.

Later guides report recurring problems:
- March 2026 Reddit snapshots complain that the meter "went haywire" and that long-time Pro users were wiped out after a handful of prompts.
  Source: https://reddit.sentinel-team.org/posts/1rpvvxa/snapshots/2026-03-14T00%3A11%3A38.395735Z (confidence: low, third-party snapshot, anecdote).
- A renewed subscription may keep the old weekly reset schedule (same low-confidence source family).
- A June 2026 bug reportedly showed an incorrect weekly limit for about 3 percent of Max and Pro users, and Anthropic reset limits.
  Source: https://pasqualepillitteri.it/en/news/5428/claude-code-weekly-usage-limit-bug-3-percent (confidence: low, secondary blog).
- A May 15 2026 global counter reset is reported by the same blog family: https://pasqualepillitteri.it/en/news/2614/claude-resets-rate-limits-5-hour-weekly-may-15-2026 (confidence: low).
- Agentic loops re-read the whole conversation each step, so long sessions drain faster.
  Source: https://oldeucryptoboi.substack.com/p/claude-code-usage-limits-guide (confidence: low).
- Guides disagree on whether the weekly window is rolling or fixed.
  Official docs say fixed per account (section 4), so treat the guides as wrong on that point.

Implication: users want in-the-moment pacing ("will I make it to reset at this rate"), early warnings, and trust that the number is correct.
A meter should show provenance and timestamp of each reading because the server-side numbers themselves have been buggy.

## 3. `rate_limit_event` and stream-json quirks

Only one detailed community write-up was found.
Source: https://littlebearapps.com/help/untether/claude-stream-json-cheatsheet/ (Untether project, CLI 2.1.283; confidence: medium, third party, version-pinned).

Reported shape:
- Top-level `type: "rate_limit_event"` with a `rate_limit_info` object.
- `status` is always present: `allowed`, `allowed_warning`, or `rejected`.
- `resetsAt` is integer epoch seconds for the window named by `rateLimitType`.
- `rateLimitType` is optional: `five_hour`, `seven_day`, `seven_day_opus`, `seven_day_sonnet`, `seven_day_overage_included`, or `overage`.
- Flat `utilization` is optional and is a 0 to 1 fraction for the warning window only.
- `unifiedWindows` is keyed by `five_hour`, `seven_day`, `seven_day_overage_included`, each with `{utilization, resetsAt}`.
- `unifiedWindows` is absent for API-key, Bedrock, and Vertex sessions.
- Extra-usage fields: `overageStatus`, `overageResetsAt`, `overageDisabledReason`, `isUsingOverage`.
- Example in the page: `five_hour` 0.09 and `seven_day` 0.15 (the 0 to 1 scale is the key point).

Quirks and caveats:
- The event is a snapshot, "not a throttle notice", and never ends the session.
- The CLI emits one only when rounded utilization or reset time changes, so a probe may not see a per-window event on every call.
- The warning threshold is 0.7.
- `rejected` with `isUsingOverage` is not a block, because extra usage covers it.
- Older schema fields (`requests_limit`, `retry_after_ms`) are reportedly never seen in real CLI output.
- Key naming differs between sources: camelCase in the cheatsheet and Rust crate, snake_case in an Elixir SDK, and one Rust test normalizes percent-style values such as 42.
  Sources: https://docs.rs/crate/ironflow-core/4.5.0/source/src/providers/claude/rate_limit_event.rs and https://hexdocs.pm/claude_code/ClaudeCode.Message.RateLimitEvent.html (confidence: low to medium, from search summaries; not fetched in full).
- The enums can drift across CLI versions; Untether checks them with a zero-token test that reads them from the installed CLI.

Not found in this sweep:
- Any public report of the cost per probe in tokens or quota.
- Any report on whether a probe itself moves the weekly number.
- Whether `unifiedWindows` appears on the very first event of a fresh session or only after a response.
Treat all three as open questions to verify empirically.

Also relevant: the official statusline docs expose `rate_limits.five_hour` and `rate_limits.seven_day` with `used_percentage` 0 to 100 and `resets_at`, which is on a different scale (percent) from the stream-json `utilization` (fraction).
Source: https://code.claude.com/docs/en/statusline (cited in 02-existing-projects.md; confidence: high).

## 4. Official statements (Anthropic docs only)

From https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work (confidence: high):
- Weekly limits apply across all models, and some plans have a separate weekly limit for one additional model (named "Fable" in the fetched text).
- Each account has a fixed weekly reset time, visible under Settings > Usage.
- The weekly reset stays the same regardless of when you start using Claude or when your subscription begins.
- The 5-hour session limit starts with your first message and resets five hours later.
- Usage across claude.ai, Claude Code, and Claude Desktop counts toward the same limit.
- Mobile apps are not explicitly mentioned on that page.
- Usage depends on conversation length and complexity, features used, model, and effort level.
- Connectors and tools are token-intensive, and long conversations with automatic context management use more.
- No exact counting formula is given.

From https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan (confidence: high):
- Limits are shared across Claude and Claude Code.
- Pro and Max have a 5-hour limit and a weekly limit; Ultra has a weekly limit, and verified Ultra accounts have no 5-hour limit.
- Max and Ultra have a separate weekly limit for the extra model.
- Wrap-up usage after hitting the 5-hour limit counts toward the weekly limit.
- Users can run `/status` for remaining allocation (fetched text; verify, since other docs and guides say `/usage`).
- The page does not state reset timing for the weekly limit.

Answers to the user's questions:
- Reset behavior: fixed per-account time, not rolling from first use (high).
- Web and mobile share the limit: web and desktop yes, mobile not explicitly stated in the fetched pages (medium).

## What to steal, what to skip

Steal:
- Pace-relative coloring (rate vs elapsed time in the window) from gxjansen/claude-code-meter and mryll/claudebar, as the primary signal for weekly pacing.
- Menu bar metric choice (session, weekly, model, spend, icon only) and a native monochrome-until-hot style from richhickson/claudecodeusage.
- Refresh on wake from sleep and retry on transient failure (richhickson).
- Local history and burn-up chart so the weekly line survives idle periods (ccburn, SlavomirDurej).
- Threshold notifications once per window per reset, the way Untether latches warnings, to avoid repeated alerts.
- Per-window provenance and timestamp on every number, given the bug and haywire-meter reports.
- Parse `rate_limit_event` defensively: accept camelCase and snake_case, treat `utilization` as 0 to 1, tolerate a missing `unifiedWindows` and unknown enum values.
- Pin and log the CLI version with each reading, since schemas drift.
- Show the official reset time as a fixed weekly anchor, with a countdown.

Skip:
- Undocumented OAuth or claude.ai internal endpoints with Keychain token reads; they are the fragile, 429-prone approach nearly everyone uses.
- Local-proxy interception of Claude Code traffic (abhishekray07/claude-meter), which is invasive and an estimator rather than a reading.
- Token-and-cost estimation as the headline number; the official percentage is available.
- Multi-provider scope creep (Codex, Copilot, Cursor) and notch, Touch Bar, or Android surfaces.
- Reusing the name "Claude Meter" without checking: https://github.com/abhishekray07/claude-meter already exists with 88 stars.
- Guide claims about rolling weekly windows or exact plan token caps; official docs say fixed reset and give no formula.
