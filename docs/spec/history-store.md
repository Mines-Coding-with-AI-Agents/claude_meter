# History Store

Status: Draft.
Related decisions: D-008, D-009.

## Purpose

Persist readings and alert state, so forecasts, spike detection, and restart-safe alerts work.

## Files

All files live in `~/.claude-meter/`.
The directory name follows D-008 and may be renamed with the final app name.

- `history.jsonl`: one reading per line, append-only.
- `alert-state.json`: which alerts already fired.
- `rules.json`: the notification rules (see `rules-engine.md`).

## Requirements

- HST-1: Each probe attempt that yields a reading must append one line to `history.jsonl` with fields: `observedAt` (ISO 8601 UTC), `utilization` (fraction), `resetsAt` (epoch seconds), `source`.
- HST-2: Failed probes must not append a reading.
  They may be written to a separate in-memory status, shown in the UI.
- HST-3: Reads must skip lines that are not valid JSON and continue.
- HST-4: A single process, the app, writes the files.
  Writes to `alert-state.json` and `rules.json` must be atomic (write a temp file, then rename).
- HST-5: The app must create the directory with owner-only permissions (mode 0700) and files with mode 0600.
- HST-6: The store must provide the readings for the current window, defined as readings with `observedAt` at or after the window start.
- HST-7: A reset is detected when utilization drops by more than 25 points and the new value is below 0.10 (D-009).
  A change in `resetsAt` alone is not a reset, because `resetsAt` varies by sub-second amounts.
- HST-8: The store must keep at least the last 5 weeks of readings and may delete older lines on startup.
- HST-9: The store must never contain credentials, prompts, or command output other than the fields listed in HST-1.
- HST-10: The store must be reachable through an injected file system interface, so tests use a temporary directory.

## Acceptance criteria

- Given a history file with one corrupt line in the middle, when read, then all valid lines are returned.
- Given readings of 0.80 then 0.04, when checked, then a reset is detected.
- Given readings of 0.50 then 0.45, when checked, then no reset is detected.
- Given `resetsAt` values that differ by 1 second and similar utilization, when checked, then no reset is detected.
- Given a crash during a write of `alert-state.json`, when the app restarts, then the file is either the old or the new version, never partial.

## Open

- O-1: Should readings from `claude` status line data be accepted as a second source in a later version?
- O-2: Should history be exportable to CSV?
