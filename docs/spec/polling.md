# Polling

Status: Draft.
Related decisions: D-009, D-010, D-011.

## Purpose

Decide when to run a probe, so spikes are caught quickly while spending little quota.

## Requirements

- PLL-1: The baseline interval is 15 minutes (D-011).
  It must be configurable in settings.
- PLL-2: The app must probe once at launch and once after the Mac wakes from sleep.
- PLL-3: The app must speed up to a fast interval (default 5 minutes) while usage is rising or a checkpoint is near.
  Rising means the last two readings differ by at least 2 points.
  Near means within 5 points below an enabled checkpoint rule's percentage.
- PLL-4: The app must return to the baseline interval after 3 consecutive readings with no rise.
- PLL-5: After a failed probe, the interval must double on each consecutive failure, up to a cap of 60 minutes.
  A successful probe resets it.
- PLL-6: The interval must never go below 2 minutes.
- PLL-7: A manual refresh must be allowed at most once per 60 seconds.
- PLL-8: The owner must be able to pause polling.
  While paused, the UI shows the last reading as stale after the threshold, and no alerts fire from it.
- PLL-9: Only one probe may run at a time.
- PLL-10: The scheduler must be driven by an injected clock so tests can run without waiting.

## Acceptance criteria

- Given a steady reading, when 15 minutes pass, then exactly one probe runs.
- Given two readings 3 points apart, when the next probe is scheduled, then it is 5 minutes later.
- Given 3 consecutive probe failures, when the next probe is scheduled, then the delay is 60 minutes.
- Given the Mac wakes from sleep, when the app notices, then a probe runs within 1 minute.
- Given a probe is running, when a manual refresh is requested, then no second probe starts.

## Open

- O-1: Should polling adapt to quiet hours, such as the owner's sleeping hours?
- O-2: Should the fast interval depend on the measured cost per probe (see `probe.md` O-2)?
