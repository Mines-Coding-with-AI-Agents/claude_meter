# Rules Engine

Status: Draft.
Related decisions: D-008, D-009, D-010.

## Purpose

Decide, from history and the current time, which alerts fire.
Rules are data that the owner can read and edit.

## Evaluation model

- RUL-1: Evaluation must be a pure function: `evaluate(history, now, rules, alertState) -> (alerts, newAlertState)`.
  It must not read the clock, files, or the network.
- RUL-2: Evaluation must produce no alerts when the latest reading is stale, when the window has ended, or when no reading exists.
- RUL-3: One evaluation may produce several alerts.
  Each alert names the rule that fired.

## Rule kinds

All rules have: `id`, `kind`, `enabled`, `message`, and optional `cooldownHours`.

- RUL-4, checkpoint: fields `percent` and `minHoursLeft`.
  It fires when utilization is at or above `percent` and the hours until reset are at least `minHoursLeft`.
  It fires once per window.
  The key is the rule `id` plus the window's reset time rounded to the hour.
- RUL-5, spike: fields `points` and `withinHours`.
  It fires when the latest utilization minus the lowest utilization among readings within `withinHours` is at least `points`.
  It is edge-triggered: after firing it cannot fire again until the cooldown has passed and the rise has fallen below the threshold.
- RUL-6, projection: fields `leadHours` and `minPercent`.
  It fires when the forecast exhaustion time is at least `leadHours` before the reset time and utilization is at least `minPercent`.
  It re-arms only after two consecutive evaluations in which the condition is false.
- RUL-7, stale-data: field `staleMinutes`.
  It fires when the latest reading is older than `staleMinutes` while polling is enabled.
  This is the only kind allowed to fire on stale data.
- RUL-8, reset: no extra fields.
  It fires once when a reset is detected (HST-7).

## Forecast

- RUL-9: Burn rate is the larger of the week-average rate and the last-24-hours rate, in points per hour.
  Week-average rate is utilization divided by hours elapsed since the window start.
  Last-24-hours rate is the utilization change over the readings in the last 24 hours divided by the hours they span.
- RUL-10: Forecast exhaustion time is now plus (1.0 minus utilization) divided by the burn rate.
  If the burn rate is zero or negative, there is no forecast.
- RUL-11: The forecast is suppressed when utilization is below 5 points, or when fewer than 2 readings exist in the last 24 hours and the week-average rate is the only basis, and the UI must say so.
- RUL-12: Per-day budget is the remaining points divided by the days left in the window, shown to one decimal.

## Rules file

- RUL-13: Rules are stored in `~/.claude-meter/rules.json` as JSON. The app writes defaults on first launch.
- RUL-14: The app must reload the file when it changes on disk and must apply edits made in the app to the same file.
- RUL-15: A rule that fails validation must be disabled with a visible error.
  The other rules must keep working.
- RUL-16: Message templates may use these placeholders: `{percent}`, `{hoursLeft}`, `{points}`, `{withinHours}`, `{exhaustAt}`, `{budgetPerDay}`.
  Unknown placeholders are left as written and flagged as a validation warning.
- RUL-17: The engine must be able to replay stored history against a candidate rule set and report when each rule would have fired.

## Default rules (draft)

```json
[
  { "id": "half-early",  "kind": "checkpoint", "percent": 50, "minHoursLeft": 72, "enabled": true,
    "message": "{percent}% of the weekly limit used with {hoursLeft}h left" },
  { "id": "eighty",      "kind": "checkpoint", "percent": 80, "minHoursLeft": 24, "enabled": true,
    "message": "{percent}% used, {hoursLeft}h until reset" },
  { "id": "burst",       "kind": "spike", "points": 30, "withinHours": 2, "cooldownHours": 6, "enabled": true,
    "message": "Usage rose {points} points in {withinHours}h" },
  { "id": "runway",      "kind": "projection", "leadHours": 12, "minPercent": 20, "enabled": true,
    "message": "At this pace the limit runs out {exhaustAt}, before the reset" },
  { "id": "gone-quiet",  "kind": "stale-data", "staleMinutes": 90, "cooldownHours": 6, "enabled": true,
    "message": "No fresh reading for {staleMinutes} minutes" },
  { "id": "new-week",    "kind": "reset", "enabled": true,
    "message": "Weekly limit reset" }
]
```

## Acceptance criteria

- Given utilization 0.52 with 100 hours left and rule `half-early`, when evaluated, then the alert fires once, and not again on the next evaluation.
- Given the same rule and 0.52 with 60 hours left, when evaluated, then nothing fires.
- Given readings 0.10 at 12:00 and 0.42 at 13:30 and rule `burst`, when evaluated, then the alert fires.
- Given `burst` has fired, when evaluated again within the cooldown, then it does not fire.
- Given the latest reading is 3 hours old, when evaluated, then only `stale-data` rules may fire.
- Given utilization 0.03, when forecasting, then no forecast is produced.
- Given a rule with `percent` as a string, when loaded, then it is disabled with an error and other rules still load.
- Given a new window after a reset, when evaluated, then checkpoint rules are eligible again.

## Open

- O-1: Should rules be able to combine conditions, such as "above 60 percent and rising fast"?
- O-2: Are the draft defaults right?
  In particular `minHoursLeft` for each checkpoint, and `leadHours` for projection.
- O-3: Should a rule have a severity that controls how the notification is shown?
- O-4: JSON has no comments.
  Is that acceptable for hand editing (D-011)?
