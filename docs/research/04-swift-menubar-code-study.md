# Code study: four Swift macOS menu bar usage trackers

Research date: 2026-11-05.
Method: files read through `gh api` (raw contents) on the default branches.
Nothing was cloned, built or executed, and all fetched content was treated as untrusted data.
Time-boxed (about 4 minutes), so this covers the files named below and not every file in each repo.

## Projects at a glance

| Project | Stars | Build system | Data source | Notes |
|---|---|---|---|---|
| tddworks/ClaudeBar | about 1.6k | Tuist app target plus a root SwiftPM package of modules | Many providers: CLI probes, OAuth, local logs, JS mapping scripts | Largest and most mature, multi-provider, Windows and Linux ports |
| GordonBeeming/claude-bar | 4 | Pure SwiftPM plus Makefile that assembles the `.app` | OAuth usage endpoint `/api/oauth/usage` | Cleanest small reference, has pace logic and severity thresholds |
| pedroprates/claude-code-usage | 2 | Pure SwiftPM plus `build-app.sh` | Status line bridge script writing JSON files | Closest to our D-004 supplement design |
| bezlant/climeter | 0 | Xcode project | OAuth usage endpoint, with a status line file as an alternative source | Best backoff and stale handling reference |

URLs:
- https://github.com/tddworks/ClaudeBar
- https://github.com/GordonBeeming/claude-bar
- https://github.com/pedroprates/claude-code-usage
- https://github.com/bezlant/climeter

## 1. Layout and architecture

### GordonBeeming/claude-bar (best template for us)

Three SwiftPM targets: a pure `ClaudeBarCore` library, a `ClaudeBar` executable (SwiftUI, AppKit glue), and `ClaudeBarCoreTests`.
All logic that can be unit tested lives in Core: usage models, `UsageService` (an actor), pace math, severity thresholds, reset formatting, OAuth, telemetry.
The UI target holds only `UsageViewModel`, views, settings and the icon renderer.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Package.swift

```swift
targets: [
    .target(name: "ClaudeBarCore"),
    .executableTarget(name: "ClaudeBar", dependencies: ["ClaudeBarCore", ...]),
    .testTarget(name: "ClaudeBarCoreTests", dependencies: ["ClaudeBarCore"])
]
```

Testability trick: `UsageService` takes its dependencies as injected `@Sendable` closures (token loaders, fetch function, clear function), with a public convenience init wiring the real ones.
Tests then pass fakes without protocols or mocking libraries.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBarCore/UsageService.swift
There are 17 test files in Core, covering decoding, thresholds, pace, reset formatting, and service behavior.

### pedroprates/claude-code-usage

Same split: `CCUsageCore` (models, `Aggregator`) and `CCUsageTracker` executable.
Tests depend only on the Core target.

### tddworks/ClaudeBar

Strongly layered: `Domain`, `Infrastructure`, `App` targets in Tuist, plus a root `Package.swift` with one library per module (`Quotas`, `Monitoring`, `DataSources`, `Providers`).
Pure kernel types in `Quotas` (`SpendRate`, `Forecast`, `UsagePace`, `PaceLevel`), the refresh loop in `Monitoring` (`QuotaMonitor`, `AdaptiveCadence`, injectable `Clock`, `PowerState`), and alerting as ports and adapters (`AlertDestination`, `AlertSender`).
It uses Mockable and a `Clock` abstraction so cadence and alert logic are tested without real time.
Source: https://github.com/tddworks/ClaudeBar/blob/main/Package.swift
This is heavier than we need, but the idea of keeping the forecast kernel, the cadence logic and the alert decider as pure types is worth copying.

### bezlant/climeter

Flat Xcode project with one file per concern (`UsageRefreshCoordinator`, `ClaudeStatusLineUsageStore`, `LaunchAtLoginService`, `PowerStateMonitor`).
Tests exist for the coordinator, credential policy, and status line store, using injected `readData` closures.

## 2. Build and packaging without full Xcode

Two of the four prove that a SwiftPM-only build plus a hand-assembled bundle works, which fits our Command Line Tools constraint.

### Makefile recipe (GordonBeeming)

`swift build -c release`, then create `Contents/MacOS` and `Contents/Resources`, copy the binary, copy a checked-in `Packaging/Info.plist`, patch versions with PlistBuddy, then `codesign`.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Makefile

```make
bundle: build
	mkdir -p $(APP_BUNDLE)/Contents/MacOS $(APP_BUNDLE)/Contents/Resources
	cp .build/release/ClaudeBar $(APP_BUNDLE)/Contents/MacOS/
	cp Packaging/Info.plist $(APP_BUNDLE)/Contents/Info.plist
	IDENT="$(CODESIGN_IDENTITY)"; [ -n "$$IDENT" ] || IDENT="-"; \
	codesign --force --sign "$$IDENT" --identifier com.gordonbeeming.ClaudeBar $(CODESIGN_OPTS) $(APP_BUNDLE)
```

Key signing lesson, from a comment in that Makefile: ad-hoc signing (`-`) mints a new code identity on every build, so any Keychain "Always Allow" grant is lost after each rebuild.
A stable Apple Development identity keeps the designated requirement the same and the grant survives.
The Makefile prefers an `Apple Development` identity found via `security find-identity` and falls back to ad-hoc.
Release builds for that project are Developer ID signed and notarized (issue 17).

### Info.plist keys used

`CFBundleIdentifier`, `CFBundleExecutable`, `CFBundlePackageType=APPL`, version keys, `LSMinimumSystemVersion`, and `LSUIElement=true` (no Dock icon).
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Packaging/Info.plist
pedroprates inlines the same plist via a heredoc in `build-app.sh` and adds `NSHighResolutionCapable` and `CFBundleInfoDictionaryVersion`.
Source: https://github.com/pedroprates/claude-code-usage/blob/main/build-app.sh
pedroprates installs into `~/Applications` and does not sign at all, and tells users to add it to Login Items by hand.
That is a missed opportunity, since `SMAppService` works once the bundle exists.

### Launch at login

All that is needed is `SMAppService.mainApp`.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBar/LaunchAtLoginManager.swift

```swift
static var isEnabled: Bool { SMAppService.mainApp.status == .enabled }
static func setEnabled(_ enabled: Bool) throws {
    if enabled { try SMAppService.mainApp.register() }
    else { try SMAppService.mainApp.unregister() }
}
```

Pitfall noted in its doc comment: `register()` only succeeds from inside a real `.app` bundle and throws under bare `swift run`, so the UI must check availability first and disable the toggle.
climeter uses an identical wrapper (toggle based on `isEnabled`).

### Notarization and release (ClaudeBar, climeter)

ClaudeBar has `scripts/sign-app.sh` (Developer ID plus entitlements) and `scripts/notarize-app.sh` (`xcrun notarytool submit`, `xcrun stapler staple`, `stapler validate`).
climeter's release workflow imports a Developer ID cert into a temporary keychain on a `macos-15` runner and uses `xcodebuild`.
Both need a paid Developer ID for distribution, which we do not need for a personal local build.

## 3. Fetching usage data

### Approach A: OAuth usage endpoint (claude-bar, climeter, ClaudeBar)

`GET https://api.anthropic.com/api/oauth/usage` with a bearer token and two special headers.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBarCore/UsageClient.swift

```swift
request.timeoutInterval = 15
request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
request.setValue("oauth-2025-04-20", forHTTPHeaderField: "anthropic-beta")
// Anthropic routes requests without a `claude-code` User-Agent into a much
// stricter rate-limit bucket, so this app would otherwise see persistent 429s.
request.setValue("claude-code/2.1.114", forHTTPHeaderField: "User-Agent")
```

Credentials: read the Claude Code Keychain item (service `Claude Code-credentials`, JSON with `claudeAiOauth.accessToken`), with a fallback to `~/.claude/.credentials.json`.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBarCore/CredentialsProvider.swift
It is strictly read-only: no Keychain writes and no use of the refresh token, because refreshing would rotate Claude Code's single-use refresh token and log the CLI out.
The Keychain read must happen off the main actor because `SecItemCopyMatching` can block on a consent dialog.

Keychain prompt recurrence (claude-bar PR 6): Claude Code rewrites the Keychain item on each token rotation, which resets the trusted-app list, so macOS re-prompts even after "Always Allow".
The author chose to document this rather than work around it (self-refresh breaks the CLI, caching widens exposure).

Self-contained OAuth (claude-bar PR 11, 25, 33): an opt-in PKCE sign-in so the app has its own token and never touches the CLI item.
It surfaced a real race: two concurrent refreshes spend the same single-use refresh token, the second gets 400, and the sign-in looks revoked.
The fix is a `SingleFlight` actor that shares one in-flight refresh among callers.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBarCore/SingleFlight.swift
It also moved to least privilege scopes (`user:profile`, later `user:inference` again, issues 25 and 34).

This is exactly the credential handling D-001 and D-004 want to avoid, so approach A is a reference for what NOT to build, except for the headers and backoff knowledge.

### Approach B: status line bridge (pedroprates, climeter)

pedroprates installs `~/.claude/cc-usage-bridge.sh` as the status line command.
The script reads JSON from stdin, extracts `rate_limits`, writes one file per `session_id` under `~/.claude/cc-usage-tracker/sessions/`, and chains to the user's previous status line command.
The app aggregates by taking the max percentage per reset window and prunes stale session files.
Sources: https://github.com/pedroprates/claude-code-usage/blob/main/Sources/CCUsageTracker/BridgeInstaller.swift , `ClaudeUsageService.swift`, `scripts/test-bridge.sh`

Lessons from its history and code:
- Issue 1: a single shared `state.json` had a multi-writer race between concurrent sessions, fixed by per-session files plus aggregation.
- Issue 2: the bridge broke the status line under tmux.
- `isActivated` checks that `settings.json` `statusLine.command` actually routes through the bridge, because an earlier `isInstalled` (file exists) check left the app silently bypassed after the user changed the status line.
- It guards against chaining to no-op commands (`""`, `true`, `:`) that would blank the status line, and keeps a backup of the previous command, with recovery logic when that backup was lost.
- It requires `jq` (an `InstallError.jqMissing` case).
- Staleness: `isStale` is true at 5 minutes since `updated_at`, the UI dims, and the app watches the file with `DispatchSource` plus a 10 second timer fallback.

climeter reads a similar file with a 1 second poll and a 10 minute stale threshold, and requires both windows to be present (`incompleteWindows` error otherwise).
Source: https://github.com/bezlant/climeter/blob/main/Climeter/ClaudeStatusLineUsageStore.swift
Its test suite covers missing file, bad schema version, and incomplete windows, and it versions the file schema (`unsupportedSchema(Int)`).

### Polling, backoff, staleness

- claude-bar: fixed 60 second loop with `Task.sleep`, a wake-from-sleep observer (`NSWorkspace.didWakeNotification`), and a refresh-on-menu-open if the cache is older than 30 seconds.
  The menu renders cached data immediately and never blocks on the network.
  Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBar/UsageViewModel.swift
- climeter: base 180 seconds, doubling on 429 up to a 900 second cap, honoring `Retry-After`, with plus or minus 10 percent jitter and a `rateLimitRetryNotBefore` gate that defers manual refreshes too.
  Source: https://github.com/bezlant/climeter/blob/main/Climeter/UsageRefreshCoordinator.swift

```swift
// max backoff must stay high - /api/oauth/usage rate-limits aggressively
// (anthropics/claude-code#31637) and stays locked out for 30+ minutes
static let baseInterval: TimeInterval = 180.0
private let maxInterval: TimeInterval = 900.0
```

  It also re-reads the Keychain only every 15 minutes while stale, and retries once after a 401.
  It has a generation counter on the active task to drop results from superseded fetches.
- ClaudeBar: `AdaptiveCadence` halves the wait when any quota rose by at least 1 point since the last tick (down to a minimum) and doubles it when nothing moved (up to the user's chosen interval), plus a battery stretch via `PowerState`.
  Windows are matched by `resetsAt` within 15 minutes of drift.
  Source: https://github.com/tddworks/ClaudeBar/blob/main/Modules/Monitoring/Sources/AdaptiveCadence.swift

### Timestamp noise (claude-bar issue 23)

`resets_at` wobbles by a fraction of a second around round minutes (for example `04:59:59.85` then `05:00:00.42`).
Rendering the raw minute made "Resets 5:03pm" flicker between adjacent minutes.
Fix: round to the nearest minute before display.
Never compare `resets_at` for equality to identify a window.

## 4. Notifications

claude-bar has no `UNUserNotificationCenter` use at all (zero matches).
It uses in-app "celebrations" instead.
Only ClaudeBar has real notifications, so it is the reference.

### ClaudeBar's design

- `SystemAlertSender` wraps `UNUserNotificationCenter`.
  It guards on `Bundle.main.bundleIdentifier != nil`, because notifications are unavailable from a bare binary.
  Source: https://github.com/tddworks/ClaudeBar/blob/main/Sources/Infrastructure/Alerting/SystemAlertSender.swift
- Permission flow: read `notificationSettings()` first, switch on `.authorized/.provisional` (proceed), `.denied` (log and point the user at System Settings), `.notDetermined` (call `requestAuthorization([.alert, .sound, .badge])`).
- Each notification gets a `UUID` identifier and a category identifier per kind (`QUOTA_ALERT_PERCENT`, `QUOTA_ALERT`, `SIGNED_OUT`).
- Decision logic is separated from delivery: `QuotaAlerts` decides, `AlertDestination.announce` delivers, so tests use a fake destination.

### Dedup and thresholds (QuotaAlerts)

User-editable list of up to 5 percentages (whole numbers 1 to 99), validated with typed refusals that Settings prints (`alreadyAlerted`, `alreadyListed`, `full`).
Hysteresis dedup: alert once when the value falls below a threshold, record it in a `told` set, and clear it only after the value climbs a full point above the threshold.
`told` is kept in memory only, so after a relaunch a login still below a threshold is told once more.
Source: https://github.com/tddworks/ClaudeBar/blob/main/Sources/Domain/Alerting/QuotaAlerts/QuotaAlerts.swift

```swift
if left < Double(percent) {
    if below.insert(percent).inserted { alerts.append(QuotaAlert(...)) }
} else if left >= Double(percent + 1) {
    below.remove(percent)
}
```

It also has a "signed out" alert, webhooks, and a settings pane (`NotifyPane`, `SyncAlertsPane`).
None of this includes a time-based cooldown, only hysteresis, and none of it forecasts: alerts are threshold-only.

### claude-bar settings

`SeverityThresholds` is a pure struct with warning (default 75) and critical (default 90) percentages, clamped in setters, with critical forced to be at least warning so a mid-edit settings field cannot create an unreachable band.
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBarCore/SeverityThresholds.swift

## 5. Forecast and pace logic

### ClaudeBar: `SpendRate`, `Forecast`, `UsagePace`, `PaceLevel`

Single source of truth so the UI never contradicts itself.
`SpendRate = percentUsed / percentTimeElapsed`, only defined once at least 3 percent of the window has elapsed (about 5 hours of a week) and before the window ends.
`runsOutBeforeReset` is true when `percentUsed - percentTimeElapsed > 5` points.
Forecast time to zero: `percentLeft * windowLength / (100 * rate)`, and "before reset" is `untilReset - runsOutIn`.
Sources: https://github.com/tddworks/ClaudeBar/blob/main/Modules/Quotas/Sources/SpendRate.swift and `Forecast.swift`

```swift
let runsOutIn = percentLeft * length / (100 * rate.value)
self = .runsOut(in: .seconds(runsOutIn),
                beforeReset: .seconds(max(0, untilReset - runsOutIn)))
```

`PaceLevel` grades projected end-of-period usage into six levels: under 50, 50 to 75, 75 to 90, 90 to 100, 100 to 120, over 120 percent.
Both pace and forecast read the same rate and the same 5 point margin, so they cannot disagree.

### claude-bar: steady pace line

The window start is derived as `resetsAt - duration` (5 hours for session, 7 days for weekly), and the fraction elapsed is clamped to 0...1.
Over pace means `percent > 5` and `percent - fraction*100 > margin + 0.0001` (the tolerance absorbs floating point error on the line).
Source: https://github.com/GordonBeeming/claude-bar/blob/main/Sources/ClaudeBarCore/UsageWindow.swift
Issue 15: the over-pace state is suppressed until usage exceeds 5 percent, because a fresh window reporting 1 percent produced a false flame and a false celebration.

Both are linear extrapolation of average burn since window start.
Neither uses recent-slope or day-of-week/active-hours weighting, so there is room for our differentiator (per-day budget, recent burn).

## 6. Bugs, pitfalls and open issues seen

- Keychain prompt reappears after each Claude Code token rotation (claude-bar PR 6).
- Concurrent OAuth refresh spends single-use refresh token and looks like revocation (claude-bar issue 33).
- Reset detection by `resetsAt` jump produced phantom weekly fireworks and hourly session confetti (claude-bar issue 10).
  The fix: a reset is a usage drop of more than 25 points to under 10 percent, evaluated per limit.
  It also catches ad-hoc server side resets.
- `resets_at` sub-second wobble flickers displayed minutes (issue 23).
- The usage endpoint is aggressively rate limited, with lockouts over 30 minutes, and requests without a `claude-code` User-Agent hit a stricter bucket (claude-bar comment, climeter comment citing anthropics/claude-code#31637).
- Status line bridge pitfalls (pedroprates): multi-writer race on one file, breaking the status line under tmux, silently bypassed when the user changes `statusLine.command`, hard dependency on `jq`.
- `SMAppService.register()` throws outside a real `.app`.
- ad-hoc signing changes identity on every build, invalidating Keychain grants.
- ClaudeBar open issues are mostly provider-specific parsing (DeepSeek multi-currency, leaderboard edge cases), and closed ones show cold-start CLI timeouts and "rate limited" on a sub-breakdown failing the whole read (issue 677, fixed so partial data still shows).
  Lesson: a failure of one optional sub-read must not discard the primary number.
- climeter and claude-bar are tiny projects (0 and 4 stars), so treat them as design references only.
- The repos' issue lists were short, so the pitfall evidence is mostly from PR and issue bodies by the maintainers, not user bug reports.

## Lessons to adopt / avoid

Adopt:
1. Split into a pure Core library target and a thin SwiftUI executable target, with tests only against Core (claude-bar layout).
2. Inject dependencies as closures or small protocols and inject a clock, so the forecast, cadence and alert logic run without real time or network.
3. Build with `swift build -c release` and a Makefile or script that assembles `Contents/MacOS`, `Contents/Info.plist` (with `LSUIElement=true`, a real `CFBundleIdentifier`), then `codesign`.
   This works with only the Command Line Tools.
4. Sign with a stable identity (a free Apple Development cert or a self-created local one) rather than ad-hoc, so any Keychain or notification permission state survives rebuilds.
5. Use `SMAppService.mainApp` for launch at login, and disable the toggle when not running from a `.app`.
6. Guard notification code on `Bundle.main.bundleIdentifier`, check `notificationSettings()` before requesting, handle `.denied` with a visible hint in the UI, and decide alerts in a pure type separate from the sender.
7. Alert dedup by hysteresis (fire once on crossing, re-arm only after dropping a margin below), and persist the "already told" state (ClaudeBar keeps it in memory, which re-notifies after relaunch).
   For us, key it by weekly window (`resets_at` rounded) and threshold.
8. Reuse the single-rate design: one `SpendRate`, with pace label and forecast both derived from it, a noise floor (3 percent of window or 5 percent usage), and a 5 point margin.
9. Show cached data instantly, refresh in the background, mark stale explicitly (5 to 10 minutes for push feeds), and never let one failed optional sub-read erase the main number.
10. Round `resets_at` to the minute for display and compare windows with a drift tolerance (ClaudeBar uses 15 minutes), never by equality.
11. Detect resets by a large usage drop (more than 25 points and under 10 percent) rather than by `resets_at` changes.
12. Status line bridge, if built: one file per session, aggregate by max per reset window, atomic writes, schema version field, verify `statusLine.command` still routes through the bridge, back up and chain the previous command, never chain to a no-op command, avoid a `jq` dependency (use a Swift helper or plain shell parsing).
13. Backoff pattern if we ever poll an endpoint: base interval of minutes, double on 429 to a cap, honor `Retry-After`, jitter, a not-before gate that also blocks manual refresh, and a generation counter to discard superseded results.

Avoid:
1. Do not call `/api/oauth/usage` or read the `Claude Code-credentials` Keychain item as the primary path.
   It conflicts with D-001 and D-004, triggers recurring consent prompts, is rate limited, and needs a spoofed `claude-code` User-Agent.
2. Do not refresh or write the Claude Code OAuth token, and never run two refreshes concurrently (single-use refresh token).
3. Do not use ad-hoc signing if any Keychain-backed feature is added.
4. Do not use one shared state file written by multiple sessions.
5. Do not use `resets_at` jumps to detect resets or as a window identity.
6. Do not alert on pace in the first few percent of a window (suppress below the noise floor) because the flame and notification will be false positives.
7. Do not copy ClaudeBar's multi-provider machinery (Tuist, QuickJS scripts, SQLite readers, webhooks).
   It is far beyond MVP scope.
8. Do not rely on `swift run` for testing launch at login or notifications, because both need a real bundle.
9. Do not depend on a time-less alert design: add an explicit cooldown or per-window dedup key so a reset followed by a fast climb re-arms correctly.
