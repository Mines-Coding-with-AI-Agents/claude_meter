# Glossary

Status: Draft.
Use these terms exactly in specs, code, and conversation.

- Weekly window: the 7-day period over which Claude Code weekly usage is counted.
  It ends at the reset time.
- Reset time (`resetsAt`): the end of the current weekly window, as epoch seconds from the probe.
  Display only after rounding to the minute.
- Window start: reset time minus 7 days.
- Utilization: the fraction of the weekly limit used, from 0.0 to 1.0, as reported by the probe.
  The app's internal unit is the fraction.
  The UI shows it as a percentage.
- Points: percentage points of the weekly limit.
  A rise from 40 percent to 70 percent is 30 points.
- Probe: one run of the installed `claude -p` that yields a rate limit event.
- Rate limit event: the `rate_limit_event` message in the `claude -p` stream-json output.
- Reading: one observation of utilization and reset time, with the time observed and its source.
- Source: where a reading came from.
  In the first version the only source is `probe`.
- Age: now minus the time a reading was observed.
- Stale: a reading whose age is over the stale threshold (default 45 minutes), or whose reset time has passed.
- Unavailable: the state when no usable reading exists, for example `claude` is missing or the probe failed.
- Rule: a user-editable definition of when to notify.
  Kinds are checkpoint, spike, projection, stale-data, and reset.
- Checkpoint: a rule that fires when utilization reaches a percentage while enough time remains in the window.
- Spike: a rule that fires when utilization rises by a number of points within a number of hours.
- Projection: a rule that fires when the forecast says the limit will run out well before the reset time.
- Forecast: the predicted time utilization reaches 100 percent at the current burn rate.
- Burn rate: points per hour.
  Defined in `rules-engine.md`.
- Per-day budget: remaining points divided by remaining days in the window.
- Alert: the result of a rule firing.
  An alert is delivered as a notification.
- Alert state: the persisted record of which alerts already fired, used to avoid repeats.
- Cooldown: the minimum time before the same rule may fire again.
- Re-arm: making a fired rule eligible to fire again.
- History: the persisted list of readings.
- Baseline interval: the default time between probes when nothing is happening (15 minutes).
