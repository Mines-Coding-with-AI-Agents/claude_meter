# Probe

Status: Draft.
Related decisions: D-001, D-006, D-007, D-009.

## Purpose

Obtain the current weekly utilization and reset time from the owner's Claude Code subscription, with no API key and no stored credentials.

## Background

A one-time check on 2026-11-05 ran `claude -p` with a trivial prompt and the API key unset.
The stream-json output contained a rate limit event with this shape (identifiers removed):

```json
{
  "type": "rate_limit_event",
  "rate_limit_info": {
    "status": "allowed",
    "resetsAt": 1794384000,
    "rateLimitType": "seven_day",
    "overageStatus": "rejected",
    "unifiedWindows": {
      "seven_day": { "utilization": 0.02, "resetsAt": 1794384000, "status": "allowed" }
    }
  }
}
```

The event is undocumented in its `unifiedWindows` detail.
Community reports say it fires only when the rounded value changes, and only after a paid model turn.

## Requirements

- PRB-1: The probe must run the installed `claude` binary as a subprocess with `-p`, a minimal prompt, `--output-format stream-json`, `--verbose`, the smallest available model, no tools, and no session persistence.
- PRB-2: The probe must remove `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN` from the child environment, so the subscription login is always used.
- PRB-3: The app must never read, store, or log any credential, token, or the full child environment.
- PRB-4: The probe must locate `claude` in this order: the path set in app settings, well-known install locations, then the owner's login shell (`command -v claude`).
  A GUI app does not inherit the terminal `PATH`, so the lookup must not rely on it.
- PRB-5: The probe must kill the process after a timeout (default 60 seconds) and report a failure.
- PRB-6: The probe must parse the stream line by line and ignore lines that are not valid JSON.
- PRB-7: The parser must extract the weekly utilization from `rate_limit_info.unifiedWindows.seven_day.utilization` and the reset time from the same object, falling back to `rate_limit_info.resetsAt` when `rateLimitType` is `seven_day`.
- PRB-8: The parser must treat utilization as a fraction from 0.0 to 1.0.
  A value outside that range must be rejected as invalid, not clamped.
- PRB-9: The parser must tolerate key casing differences (`resetsAt` and `resets_at`) and unknown extra keys.
- PRB-10: The probe result must be one of: a reading, or a failure with a reason.
  Failure reasons: `claudeNotFound`, `launchFailed`, `timedOut`, `nonZeroExit`, `noWeeklyData`, `invalidData`.
- PRB-11: A failure must never be converted into a reading of zero or any other number.
- PRB-12: If the process exits with a non-zero status and no rate limit event, the failure reason is `nonZeroExit`, and the captured stderr must be checked for limit-reached wording.
  A recognized limit-reached message is reported as `limitReached` with utilization 1.0 treated as unknown-but-at-limit, shown in the UI as "limit reached".
- PRB-13: The probe must not write anything outside the history store and must not touch `claude`'s own files.
- PRB-14: A reading has the fields: utilization (fraction), resetsAt (epoch seconds), observedAt, source (`probe`).

## Acceptance criteria

- Given the sample event above, when parsed, then the reading has utilization 0.02 and the given reset time.
- Given an event with utilization 1.4, when parsed, then the result is `invalidData`.
- Given an event with only a `five_hour` window, when parsed, then the result is `noWeeklyData`.
- Given output with no rate limit event and exit code 0, when parsed, then the result is `noWeeklyData`.
- Given `ANTHROPIC_API_KEY` set in the parent, when the probe runs, then the child environment does not contain it.
- Given `claude` is not at any searched location, when the probe runs, then the result is `claudeNotFound`.
- Given a process that never exits, when 60 seconds pass, then the process is killed and the result is `timedOut`.

## Open

- O-1: Does the event fire on every probe, or only when the rounded value changes?
  If it can be absent on a healthy probe, how should the app treat it: as unchanged since the last reading, or as `noWeeklyData`?
  Measure with repeated probes before deciding.
- O-2: What is the quota cost of one probe?
  Measure and record it.
- O-3: Exact wording of the limit-reached message for PRB-12.
  Capture a real sample.
- O-4: Which smallest model name is valid for this account at implementation time.
