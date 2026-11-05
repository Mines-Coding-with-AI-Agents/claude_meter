# Decision Log

Critical decisions made during the project, newest at the bottom.
Each entry records the decision, the date, and the reasoning.

## D-001 - Subscription only, no API key

Date: 2026-11-05
Status: Accepted

Claude Meter must rely entirely on Zibo's Claude Code subscription.
No Anthropic API key is used anywhere in the project.
Reason: the goal is to track the subscription weekly limit, and the project should cost nothing beyond the existing plan.

## D-002 - Keep project records in the repo

Date: 2026-11-05
Status: Accepted

The conversation is logged in `docs/CONVERSATION.md` and critical decisions in `docs/DECISIONS.md`.
Research outputs live in `docs/research/`.
Reason: the project is a demonstration, so the reasoning trail should be visible alongside the code.

## D-003 - Research runs as parallel agents

Date: 2026-11-05
Status: Accepted

Discovery work is split into independent parallel agents (SDK and limits, existing projects, local data and ideas).
Reason: the topics do not depend on each other, and Zibo wants results in 3 to 5 minutes.

## D-004 - Primary data source is the status line `rate_limits` feed

Date: 2026-11-05
Status: Superseded by D-006 (the status line feed is kept only as a zero-cost supplement)

Superseded because the status line only updates while Claude Code runs on this Mac, and Zibo also uses Claude Code on Windows and a phone.

Claude Meter captures `rate_limits.five_hour` and `rate_limits.seven_day` (`used_percentage` and `resets_at`) from the Claude Code status line JSON.
A wrapper around the status line appends each reading, with a timestamp, to a local store.
Reasons:
- It is documented, needs no API key, and involves no token or cookie handling.
- It fits the Agent SDK legal guidance that tools should not collect or intermediate Claude.ai credentials.
- Three independent research agents converged on it.

Rejected as primary sources:
- The undocumented `api.anthropic.com/api/oauth/usage` endpoint: unstable, and it needs the OAuth token.
- Browser session cookies: fragile and sensitive.
- The Agent SDK `rate_limit_event`: it does not say which window it covers.

Known limits:
- The feed appears only while a session is running, and only after the first API response.
- A window can be absent or dropped after `resets_at`.
- So the store must keep its own history and label stale or missing data explicitly.

Sources: [01-agent-sdk-and-limits.md](research/01-agent-sdk-and-limits.md), [02-existing-projects.md](research/02-existing-projects.md).

## D-005 - MVP scope and stack

Date: 2026-11-05
Status: Accepted

MVP:
1. Native Swift (SwiftUI `MenuBarExtra`) app in the macOS top bar showing the weekly percentage, with no terminal window needed.
2. Popover with reset time, forecast ("limit hit on <day>" or "safe until reset"), and per-day budget.
3. Notifications at 50, 75, and 90 percent, and on projected exhaustion before reset.
4. Data source per D-006, with each number labeled by source and age.

Phase 2: status line supplement, transcript estimator, guard hook in warn mode, `/meter` plugin command, cost attribution.
Stack: Swift for the app (revised from TypeScript and SwiftBar after Zibo asked for a standalone menu bar tool).
Open: only the Command Line Tools are installed, so building and packaging a signed `.app` bundle (needed for notifications and launch at login) must be checked early.
Differentiator versus existing tools: projection and pacing (per-day budget, ideal-pace comparison, alerts on projected exhaustion) rather than static thresholds.
Source: [03-ideas-and-local-data.md](research/03-ideas-and-local-data.md).

## D-006 - Primary data source is a `claude -p` / Agent SDK probe of the weekly limit

Date: 2026-11-05
Status: Accepted (Zibo directed this approach after the check below)

Claude Meter polls the account-wide weekly limit by running a minimal `claude -p` query (or the equivalent Agent SDK call) and reading the `rate_limit_event` message.
Reasons:
- Zibo uses Claude Code on a Mac, a Windows desktop, and a phone, and the limit is account-wide.
- The status line feed only updates while Claude Code runs on this Mac, so it goes stale exactly when usage happens elsewhere.
- The probe works with the subscription login and no API key.

Check result (one run, 2026-11-05, Haiku, `ANTHROPIC_API_KEY` unset):
- The event had `rateLimitType: seven_day`, `resetsAt` (epoch seconds, about a week out), and `unifiedWindows.seven_day.utilization: 0.02`.
- Only the weekly window was reported.
- Zibo states that 0.02 means 2 percent.
  This has not yet been cross-checked against `/usage`.
- Not yet known: whether the event appears on every call, and its behavior near the threshold.

Costs and risks accepted:
- Each probe spends a small amount of weekly quota, so poll sparingly (for example every 15 to 30 minutes) and make it switchable.
- No token is stored: per D-007 the app runs the installed `claude -p` as a subprocess and uses Claude Code's own login (amended 2026-11-05, replacing the earlier `setup-token` idea).
- The SDK legal page says SDK-built products should use API keys and not intermediate Claude.ai credentials.
  This is personal use of Zibo's own login, but it remains a gray area.
- The status line feed (D-004) stays as a zero-cost supplement.

## D-007 - Native Swift menu bar app with a testable core package

Date: 2026-11-05
Status: Accepted

The app is Swift, using SwiftUI `MenuBarExtra`.
Layout follows the claude-bar pattern: a pure Core library target with all probe parsing, forecast and rule logic, and a thin SwiftUI executable target on top.
Dependencies are injected as closures so the core can be unit tested.
The app runs the installed `claude -p` as a subprocess, so it uses Claude Code's own login and stores no token.
Build without full Xcode: `swift build -c release`, then a script assembles the `.app` with a checked-in `Info.plist` (`LSUIElement=true`) and signs it.
Ad-hoc signing changes the code identity on every build, so a stable Apple Development identity is preferred.
That needs an Apple ID set up for development, which Zibo may not have, so ad-hoc signing is the fallback.
Whether notifications and launch at login work with ad-hoc signing is tested early.
Launch at login uses `SMAppService.mainApp`, which only works from a real `.app`.
Open risks: the signed-bundle build path, and `claude` binary discovery from a GUI environment.
Sources: [04-swift-menubar-code-study.md](research/04-swift-menubar-code-study.md).

## D-008 - Data-driven notification rules

Date: 2026-11-05
Status: Accepted

Notification rules are data in `~/.claude-meter/rules.json`, with defaults written on first launch.
The app reloads the file on change, and an in-app editor writes the same file.
Evaluation is a pure function of (reading history, now, rules), so it is unit tested and can replay history to show what a rule would have fired.
Rule kinds: checkpoint (percent and minimum hours left), spike (points within hours, with cooldown), projection (exhaustion before reset), plus stale-data and reset notices.
Behavior borrowed from the code studies:
- Checkpoints are keyed per `resets_at` and fire once per week.
- Spike rules are edge-triggered with a cooldown.
- Projection alerts re-arm only after two consecutive safe evaluations.
- No alerts fire on stale, absent, or awaiting-new-week data.
- "Already notified" state is persisted, so a relaunch does not re-notify.
Projection is `max(week-average rate, 24h rate)` over wall-clock hours.
Sources: [05-forecast-and-alert-logic-study.md](research/05-forecast-and-alert-logic-study.md).

## D-009 - Data hygiene rules for the probe

Date: 2026-11-05
Status: Accepted

- `utilization` from `rate_limit_event` is a 0 to 1 fraction, and the status line feed uses 0 to 100.
  Each is normalized at the boundary and the two are never mixed.
- Reset time comes only from the `resetsAt` epoch, rounded for display and never compared for equality (it wobbles by sub-second amounts).
- A reset is detected by a usage drop of more than 25 points to under 10 percent, not by `resetsAt` jumps.
- The percentage is cleared once `now >= resetsAt`.
- The event fires only when the rounded value changes, so the app caches the last known value and labels it with its age.
- `claude -p` can exit 1 at a limit with no event, which the poller treats as "at limit" or "unavailable", never as zero.
- Every number shows its source and age.
Sources: [05](research/05-forecast-and-alert-logic-study.md), [06](research/06-community-and-gaps-sweep.md).

## D-010 - Three lessons from other projects are MVP requirements

Date: 2026-11-05
Status: Accepted (Zibo approved adding these to the MVP)

The MVP includes three behaviors learned from existing projects:
1. Trustworthy numbers: every reading shows its source and age, stale readings are marked (older than about 10 minutes), the percentage is cleared after the reset time, and a reset is detected by a usage drop of more than 25 points to under 10 percent (see D-009).
2. Quiet, reliable alerts: checkpoints fire once per week, spike alerts are edge-triggered with a cooldown, projection alerts re-arm after two consecutive safe readings, and "already notified" state is saved to disk so a relaunch does not re-notify (see D-008).
3. Adaptive polling: poll slowly when usage is flat and far from any threshold, poll faster when usage is rising or near a checkpoint, and back off on failures, because each probe spends real quota.

Sources: [04](research/04-swift-menubar-code-study.md), [05](research/05-forecast-and-alert-logic-study.md).
Note: D-005 to D-009 were accepted later the same day.

## D-011 - Working name and two defaults

Date: 2026-11-05
Status: Accepted

- Working name: Weekwatch.
  The name avoids "Claude" so it does not collide with the existing claude-meter project (abhishekray07/claude-meter) or imply an official Anthropic product.
  It is a working title and can be renamed cheaply.
  The repository and folder keep their current names until a final name is chosen.
- Baseline poll interval: 15 minutes, with the adaptive speed-up and backoff from D-010.
- Rules file format: JSON, because Swift's `Codable` supports it natively with no extra dependency.

## D-012 - Agent environment: AGENTS.md, per-feature specs, no skills, guardrails in settings

Date: 2026-11-05
Status: Accepted

- `AGENTS.md` is the canonical agent instruction file, and `CLAUDE.md` only imports it with `@AGENTS.md`.
- Specs are one file per feature in `docs/spec/`, with numbered requirement IDs, acceptance criteria, and an Open section.
- The project uses plain Claude Code only: no skills, plugins, or MCP servers, because this is a demonstration and not everyone has the same skill set.
- Planning has not started. No plan is written and no production code is written until Zibo says so.
- The hard rules in `AGENTS.md` (no API key, no credential access, no direct API calls, no commits or pushes without being asked) are enforced mechanically in `.claude/settings.json` with deny and ask rules.
- The first spec drafts were written from the research and decisions, before a requirements interview. They are marked Draft and each has open questions.

## D-013 - Specs are evaluated in layers before planning

Date: 2026-11-05
Status: Accepted

The specs are evaluated before planning starts, using five layers defined in `docs/spec/EVALUATION.md`: spec lint, scenario walkthroughs, measurements of gating unknowns, a blind-reader review, and owner sign-off.
For now the evaluation is a document only, and none of the layers has been run.
When planning starts, the evaluation becomes part of the plan.
Open: where the lint lives, whether the Draft, Reviewed, Approved gate becomes a hard rule in `AGENTS.md`, and approval to run the measurement probes.

## D-014 - Development plan accepted and paused

Date: 2026-11-05
Status: Accepted (paused)

Zibo accepted the development plan and asked to pause it.
The plan has six milestones: M0 spec readiness (the evaluation in `docs/spec/EVALUATION.md`), M1 foundation and risk spike, M2 Core domain, M3 platform integration, M4 UI, M5 hardening and demo.
The app is a Swift package with `Core` (pure logic), `Platform` (macOS adapters), and `WeekwatchApp` (thin SwiftUI shell).
Development uses plain Claude Code: an orchestrator session, test-first implementation, a fresh read-only reviewer given only the spec and the diff, and a single gate script (`scripts/check.sh`).
The plan file lives outside the repository at `/Users/zibo/.claude/plans/encapsulated-booping-fog.md`.
Until Zibo says to resume: no `docs/plan/` files, no `AGENTS.md` status change, and no production code.
On resume the first steps are rendering the plan as `docs/plan/plan.html` and starting M0.
