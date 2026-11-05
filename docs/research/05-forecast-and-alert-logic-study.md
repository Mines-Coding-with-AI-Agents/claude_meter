# Forecast and alert logic study

Research date: 2026-11-05.
Method: read-only fetch of source files and issues through `gh api` and raw.githubusercontent.com.
Nothing was cloned, built, or run.
All fetched text was treated as untrusted data.

Time budget was about 4 minutes, so coverage is selective.
Not read in depth: ccusage source (code search of the repo was unavailable to the token, and prior study 02 says it has no limit awareness), and the issue trackers beyond title scans plus a handful of bodies.
Treat "not found" as "not looked at hard enough", not as "does not exist".

## Summary of what the field does

None of the studied tools implements a good weekly forecast.
The mainstream pattern is: percent used divided by percent of window elapsed, or a short-window slope extrapolated to 100%.
Alerting is almost absent: Claude-Code-Usage-Monitor has exit codes, claude-pulse has colour thresholds, nobody has hysteresis, cooldowns, or user-editable rules.
The most valuable lessons are about data hygiene (stale, absent, leaked, and expired windows), not about the maths.

## 1. Burn-rate and projection algorithms

### Claude-Code-Usage-Monitor (Maciek-roboblog)

Repo: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor

It projects on local token counts per 5-hour session block, not on official percentages.
The burn rate is a whole-block average, with no smoothing and no idle handling:

```python
# src/claude_monitor/core/calculations.py
tokens_per_minute = total_tokens / block.duration_minutes
cost_per_hour = (block.cost_usd / block.duration_minutes) * 60
...
projected_additional_tokens = burn_rate.tokens_per_minute * remaining_minutes
projected_total_tokens = current_tokens + projected_additional_tokens
```

URL: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor/blob/main/src/claude_monitor/core/calculations.py

Formula: `projected_total = used + (used / elapsed) * remaining`.
Lessons:
- A cumulative average since block start decays slowly after a burst and never reflects a pause, so after a long idle gap it over-projects.
- Returns `None` when `duration_minutes < 1` or tokens are zero.
  That is a sensible "too early to say" guard.
- A separate `calculate_hourly_burn_rate` sums tokens in the trailing 60 minutes and divides by 60.
  So it has two rates: block-average and trailing-hour.
- In `output/snapshots.py` the forecast deliberately uses an input+output-only burn rate so the basis matches `tokens_remaining`.
  The code comment says the cache-inclusive rate "would be wrong here".
  Lesson: the numerator and the limit must be in the same unit.
  Claude Meter avoids this by working directly in official percent points.
- When status is `limit_hit` the forecast is nulled out (`predicted_tokens_exhausted_at: None`) rather than showing a past time.

### Pace (used percent versus elapsed percent)

Claude-Code-Usage-Monitor `_pace_from_window`:

```python
elapsed_ratio = min(1.0, max(0.0, elapsed_seconds / _FIVE_HOUR_SECONDS))
delta = float(used_pct) - elapsed_pct
if delta > _PACE_TOLERANCE_POINTS:   # 10.0
    label = "slow down"
elif delta < -_PACE_TOLERANCE_POINTS:
    label = "speed up"
else:
    label = "on track"
```

URL: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor/blob/main/src/claude_monitor/output/snapshots.py
Window start is derived as `resets_at - window_length`, which is the only anchor the official feed provides.
Pace is only computed for the five-hour window, and the weekly slot was explicitly deferred (issue #234 quotes the comment `seven_day is the OFFICIAL weekly slot, deferred`).
So nobody in this repo ships weekly pace yet.
The tolerance band is plus or minus 10 points, a symmetric dead band with no hysteresis.

### claude-pulse (NoobyGains)

Repo: https://github.com/NoobyGains/claude-pulse (single file `claude_status.py`, about 13.7k lines).

It has the only real time-series logic found.
Two estimators over a persisted sample list of `{t, s}` (timestamp, percent):

```python
# claude_status.py  _compute_velocity
recent = [s for s in samples if s.get("t", 0) > now - 300]   # last 5 min
dt = recent[-1]["t"] - recent[0]["t"]
if dt < 10: return None
return (dp / dt) * 60   # pct per minute
```

```python
# _estimate_runway: least squares over the last 10 minutes
slope = (n * sum_ts - sum_t * sum_s) / denom      # pct per second
if slope <= 0.001: return None                    # flat or declining
seconds_to_full = (100.0 - current_pct) / slope
if seconds_to_full > 86400: return None           # not useful
```

URL: https://github.com/NoobyGains/claude-pulse/blob/main/claude_status.py (search `_compute_velocity`, `_estimate_runway`).
Lessons:
- Windows of 5 and 10 minutes are tuned for a five-hour limit and a live status line.
  They are far too short for a weekly limit, where a coffee break would zero the slope.
- Idle handling is "show nothing": velocity at or below 0.5 pct per minute is hidden, and a flat slope returns `None`.
  This is honest but gives no projection at all during idle.
- Runway is suppressed beyond 24 hours.
  For a weekly limit that rule must be removed, since the whole point is a multi-day horizon.
- Needs at least 2 samples and at least 10 seconds span.
  The utilisation reading is integer percent, so short-window slopes are quantised and noisy.
  Integer quantisation is the key reason to use long windows.
- Pace widget `_calc_pace_pct(resets_at, 604800)` is `elapsed / window * 100`, clamped to 0..100, and returns `None` when `elapsed <= 0`.
  Arrows compare `pct - pace`, with double arrows for "well over" and "well under".

### claude-code-limit-tracker (TylerGallenbeck)

Repo: https://github.com/TylerGallenbeck/claude-code-limit-tracker
It estimates usage from local JSONL in hours of Sonnet and Opus against configured limits in `config/limits.json`.
Its week start is `now - weekday()` days at midnight (Monday), which is a calendar guess, not the real reset time.
No forecast.
Lesson: guessing the weekly boundary from the calendar is the failure mode the official `resets_at` removes.

### ccusage (ryoppippi)

https://github.com/ryoppippi/ccusage
Per study 02, it is cost-centric with 5-hour blocks and a burn rate in its blocks report, with no limit awareness.
Not source-reviewed here.
Its value for Claude Meter is limited to the transcript estimator (phase 2).

## 2. Weekly window modelling

- All tools that use official data derive window start as `resets_at - 7 days`.
  There is no official start field.
- Claude-Code-Usage-Monitor `official.py` treats a window as expired at or past reset:

```python
# src/claude_monitor/output/official.py  _window
if epoch is not None and now_epoch is not None and now_epoch >= epoch:
    pct = None
```

  URL: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor/blob/main/src/claude_monitor/output/official.py
  Lesson: after `resets_at` the stored percentage describes the previous week.
  Never display or project from it.
  Show "new week, no reading yet" until a fresh sample with a later `resets_at` arrives.
- The server may drop a window entirely.
  Claude Code issue 102689 reports `seven_day` missing everywhere (status line, `/usage`, desktop, web) while `five_hour` is present, apparently server side.
  URL: https://github.com/anthropics/claude-code/issues/102689
  Lesson: "weekly absent" is a normal state and needs its own UI state, not an error.
- Claude Code issue 92081: absent windows are omitted from the object, not `null`, so absent cannot be told from 0%.
  Example payload: `"rate_limits": { "seven_day": { "used_percentage": 47, ... } }` with no `five_hour` key.
  URL: https://github.com/anthropics/claude-code/issues/92081
  Lesson: treat a missing key as "unknown", never as 0.
- Per-model weekly caps exist.
  claude-pulse reads `seven_day_opus`, `seven_day_sonnet`, `seven_day_fable` and a `weekly_scoped` list from the OAuth usage payload.
  Lesson: the model should allow several weekly buckets, but the meter tracks the headline `seven_day`.
- Rollover detection: a new `resets_at` that is later than the stored one, or a percentage drop of more than a few points, means a new window.
  The reset time can also move earlier or later by seconds between samples, so compare with a tolerance (suggest 10 minutes) before calling it a rollover.
  This is an inference, not something found in the code.

## 3. Alert and threshold design

Claude-Code-Usage-Monitor `--once` exit codes (`output/snapshots.py` `_status`):

```python
if used_pct is None:
    return 20, "no_active_session" if active is None else "indeterminate"
if (active and active.get("limitMessages")) or used_pct >= 100.0:
    return 11, "limit_hit"
if used_pct >= Plans.LIMIT_DETECTION_THRESHOLD * 100:
    return 10, "near_limit"
return 0, "ok"
```

URL: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor/blob/main/src/claude_monitor/output/snapshots.py
Lessons:
- Four severity levels: ok, near_limit, limit_hit, plus an explicit unknown (20).
  The unknown state is a first-class result, which is the right call.
- Status is driven by the higher of the five-hour and weekly official percentages.
- It uses the real unclamped ratio for the threshold, with a comment that rounding must not false-fire.
- A real limit-hit message from the transcript overrides the numeric percentage.
- No hysteresis, cooldown, or dedup, because it is a pull tool run by the caller.
  Claude Meter is a long-running push tool, so it must add dedup state itself.

claude-pulse: colour escalation only, with staleness thresholds `STALENESS_WARN = 120`, `STALENESS_YELLOW = 300`, `STALENESS_RED = 600` seconds.
No notifications found.
Pace arrows are shown only when the user enables them (`"pace": False` by default).

Gap and opportunity: no studied project has checkpoint rules, spike rules, or per-rule cooldowns.
Claude Meter's rule model is therefore new ground, and the design below is original.

## 4. Stale, missing, estimated data and provenance

Claude-Code-Usage-Monitor has the best provenance model.
Constants: `_STATUSLINE`, `_API`, `_LOCAL`, with `confidence` values `official`, `experimental`, `unknown`, and a `source: {kind}` on every window.
Priority order: fresh official statusline, then opt-in OAuth API, then local estimate.
A stale capture must not drive status:

```python
# official.py
OFFICIAL_TTL_SECONDS = 600
stale = now_epoch is not None and captured is not None \
        and (now_epoch - captured) > OFFICIAL_TTL_SECONDS
```

```python
# snapshots.py
if official and not official.get("stale"):
    ...  # official drives status, headline_confidence = _OFFICIAL
```

URLs: the two files above.
The snapshot keeps a `stale` flag purely as a transparency signal and falls back to the local estimate.

Leaked-value defence: Claude Code bug 52326 made `used_percentage` carry the `resets_at` epoch when the 5h window had no data.
Monitor's `_clean_pct` drops values over 100 to `None`.
URL: https://github.com/anthropics/claude-code/issues/52326
Lesson: validate every sample (finite, 0 to 100, `resets_at` in a plausible range) before storing.

claude-pulse hardening patterns worth copying:
- A timestamp that is text, null, NaN, or absurdly far ahead is read as stale, never as an error.
  A corrupt cache entry becomes `+inf` age.
- Polled values older than 1800 s render with a `~` prefix (for example `~29%`).
- When a fallback re-reads an old cache, it must not re-stamp the timestamp, otherwise the staleness warning can never fire (source comment near `_read_stale_cache`).
- Single-flight lock files with a 30 s stale-lock break for the poller.
- Backoff using `Retry-After` on HTTP 429 from the usage endpoint.

## 5. Edge cases and bugs from issue trackers

- Timezone: Monitor issue 121 (Europe/Istanbul, no DST).
  Claude Code said reset at 18:00, the monitor said 19:00, and for an hour the tool showed pre-reset usage.
  Cause: the tool inferred the reset from a local timezone assumption instead of the epoch.
  URL: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor/issues/121
  Lesson: use only `resets_at` epoch seconds, keep all maths in UTC, and convert for display at the last step.
- Reset boundary: the monitor nulls the percentage when `now >= resets_at`.
  Without that, the old high percentage shows after the reset and fires false alerts.
- Weekly deferred: Monitor issue 234 shows users want weekly reset day and time, and a v4.0.0 release closed the request without shipping it.
  URL: https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor/issues/234
- Field availability (Claude Code tracker):
  - 95918: `rate_limits` never populated in statusLine or hook JSON on one version.
  - 96543: absent for enterprise login even though `/usage` shows credits.
  - 95711: not populated before the first API call of a session.
  - 101756: `statusLine.refreshInterval` ignored in the VS Code extension.
  - 84904 and 107376: not exposed to hooks, desktop app, or the VS Code extension.
  URLs: https://github.com/anthropics/claude-code/issues/95918 and the other numbers on the same path.
  Lesson: the feed is a best-effort supplement, so a monitor needs a second source or a graceful "no data" state.
- Clock skew: issue 103820 shows a server `date` header and a client-measured duration disagreeing by about 24 minutes after a stalled process.
  Lesson: timestamp samples with the local receive time, store `resets_at` separately, and never compare the two clocks for staleness beyond a generous tolerance.
- Quantisation: `used_percentage` is an integer in most payloads, but the SDK event shows fractions (`0.24`).
  Do not assume integer or percent versus fraction scale without a provenance-aware parser.

## 6. `claude -p` and `rate_limit_event`

Findings from Claude Code issues, not from tool source:

- Issue 91857 documents the shape of `rate_limit_event.rate_limit_info` at `status: "allowed"`:

```json
"unifiedWindows": {
  "five_hour": { "utilization": 0.24, "resetsAt": 1788465600 },
  "seven_day": { "utilization": 0.13, "resetsAt": 1789005600 }
}
```

  URL: https://github.com/anthropics/claude-code/issues/91857
  Important facts there:
  - `unifiedWindows` carries per-bucket utilisation as a fraction (0 to 1), with epoch `resetsAt`.
  - It is not declared in `SDKRateLimitInfo` types in SDK 0.3.257, so it is undocumented and may change.
  - Top-level `utilization` is absent at `allowed`.
  - It only arrives after a real model turn, and a `/usage`-only session emits none.
    So reading it costs tokens on every poll, which is the core problem for a meter.
  This revises the D-004 claim that the event "does not say which window it covers": on current builds `unifiedWindows` does, but it is undocumented.
- Issue 103253: `claude -p` exits 1 with "No messages returned from query" when a `rate_limit_event` arrives after the result.
  URL: https://github.com/anthropics/claude-code/issues/103253
- Issue 103820: print mode can exit 1 on a weekly limit with no `rate_limit_event`, no error, and no `result` on stdout.
  The message exists only in the transcript JSONL (`isApiErrorMessage: true`).
  Its last event before failing was `allowed_warning` at utilisation 0.97.
  URL: https://github.com/anthropics/claude-code/issues/103820
- claude-agent-sdk-typescript issue 403: the async iterator does not terminate after a `rate_limit_event` on a closed SSE stream.
  URL: https://github.com/anthropics/claude-agent-sdk-typescript/issues/403
- Status values seen: `allowed`, `allowed_warning`, `rejected`.
- Lesson: a `claude -p` probe is a spending read (one turn per sample), is fragile on exit code, and has no zero-token path.
  If used at all, poll rarely (for example once per hour while the Mac is active), use a minimal prompt, treat any non-zero exit as "no data", and tag samples `source: sdk_event`.

## Cross-cutting lessons

1. Percent in, percent out: forecast on the official percentage, never convert through tokens.
2. Never forecast from one point.
   Short windows (5 to 10 min) are too noisy for integer percentages.
3. Idle is not zero burn.
   Use a wall-clock window (hours to days) so idle time dilutes the rate honestly.
4. Window start is `resets_at - 7d`.
   Anything at or past `resets_at` is the previous week.
5. Absent and stale are states with labels, not errors and not zero.
6. Alerts need state: no studied tool has it, so edge-trigger plus cooldown plus reset-on-rollover is a differentiator.

## Recommended algorithm for Claude Meter

### Data model

Store samples as `(received_at_utc, used_pct, resets_at_epoch, source)`.
Source is one of `statusline`, `oauth_poll`, `sdk_event`, `estimate`.
Normalise `sdk_event` utilisation by multiplying by 100.
Reject samples outside 0..100, non-finite values, or with `resets_at` more than 8 days ahead or in the past.

Current week is defined by `reset = latest valid resets_at` and `start = reset - 7 days`.
A new week begins when a sample arrives whose `resets_at` exceeds the stored one by more than 10 minutes.
After `now >= reset` and before a new-week sample, state is `awaiting_new_week`: show no percentage and fire no alerts.
Data age is `now - received_at` of the newest sample.
States: `fresh` (age at most 10 min while a session runs), `stale` (older, shown as `~N%` with age), `absent` (no weekly window), `awaiting_new_week`.

### Projection

Compute two rates in percent points per hour and take the blend:

1. Week rate: `r_week = used_pct / hours_elapsed` where `hours_elapsed = (now - start)` in hours.
   Require `hours_elapsed >= 2`.
2. Recent rate: `r_recent = (used_now - used_at(now - 24h)) / 24`, using the nearest stored sample at or before `now - 24h`, and the week start as the floor.
   Require at least 3 samples spanning at least 6 hours, else skip.

Use `r = max(r_week, r_recent)` when both exist, otherwise whichever exists, otherwise no forecast.
Taking the max is deliberately conservative for a limit warning.
A burst inside the last 24 hours raises the projection, and idle time lowers both rates, because both use wall-clock denominators.

```
remaining_pts   = 100 - used_pct
hours_left      = (reset - now) in hours
projected_at_reset = used_pct + r * hours_left
exhaust_at      = now + remaining_pts / r      (only if r > 0)
```

Verdicts:
- `safe` if `projected_at_reset < 90`.
- `tight` if `90 <= projected_at_reset < 100`.
- `will_exhaust` if `projected_at_reset >= 100` and `exhaust_at < reset`; show the day and time and the hours before reset.
- `no_forecast` if the state is not `fresh` or `stale`, or there is not enough history.
Pace line for the UI: `delta = used_pct - elapsed_pct` with `elapsed_pct = hours_elapsed / 168 * 100`, shown as over or under pace.
Per-day budget: `(100 - used_pct) / max(days_left, 0.25)`.

### Spike detection

Rule: used points gained in a trailing window of `W` hours at least `P`, defaults `P = 30`, `W = 2`.
Compute `gain = used_now - used_at(now - W)` using the oldest sample inside the window, but require that sample to be no older than `W + 30 min`, else the gain is unreliable and the rule does not fire.
Reject the comparison across a rollover: restrict samples to the current week.
Because integer percentages quantise, never use windows shorter than 30 minutes.
Spike is edge-triggered, with a re-arm when `gain < 0.5 * P` or after the cooldown.

### Alert engine (applies to all user rules)

Each rule has: type (`checkpoint`, `spike`, `projection`), parameters, enabled flag, severity, and cooldown.
Persist per-rule state `{armed, last_fired_at, fired_for_reset_epoch}`.

- `checkpoint` (default 50 percent with at least 48 hours left, then 75 and 90): fires once per week per threshold, keyed on `(rule_id, reset_epoch)`.
  Hysteresis is unnecessary because usage in a week only goes up, and rollover clears it.
  Add a guard: if the first sample of a week arrives already above a threshold, fire only the highest crossed threshold, not all of them.
- `spike`: default 30 points in 2 hours, cooldown 3 hours, re-arm as above.
- `projection` (default: `will_exhaust` with at least 6 hours before reset): fires on the transition into `will_exhaust`.
  Re-arms only after the verdict has been `safe` or `tight` for 2 consecutive evaluations, which gives hysteresis against jitter.
  Re-notify if `exhaust_at` moves earlier by more than 12 hours, at most once per 6 hours.
- Never fire on `stale`, `absent`, or `awaiting_new_week` data.
  A single "data is stale" notice can be its own optional rule, fired once per outage.
- Severity: info (checkpoint under 75), warning (checkpoint at 75 or more, spike), critical (90 or more, projection to exhaust, limit hit).
  Map severity to notification sound or interruption level.

### Labels

Every number in the popover carries its source and age, for example `61% - statusline, 4 min ago`.
Estimates use a leading `~` and the word "estimate".
Forecast text names its basis: "based on 7-day average" or "based on last 24 hours".

### Test cases to write first

- Reset boundary: sample at `resets_at - 1s`, then at `resets_at + 1s` with no new window.
- Rollover with `resets_at` shifting by a few seconds (must not roll).
- Absent weekly key with present five-hour key.
- Leaked epoch in `used_percentage`.
- Burst then 12 hours idle: projection falls, spike does not re-fire.
- Mac asleep for 2 days: gaps longer than `W` suppress the spike rule.
- DST change inside the week: all maths in UTC, display only in local time.
