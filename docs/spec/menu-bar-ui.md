# Menu Bar UI

Status: Draft.
Related decisions: D-005, D-007, D-008, D-009, D-010.

## Purpose

Show the weekly limit state at a glance and let the owner edit settings and rules.

## Menu bar item

- UI-1: The menu bar shows a compact label with the weekly utilization as a rounded percentage, for example `W 62%`.
- UI-2: When the latest reading is stale, the label must show a stale marker, for example `W 62%?`, and a dimmed style.
- UI-3: When there is no usable reading, the label shows `W --`.
- UI-4: The label color or symbol must reflect severity (calm, warning, critical) from the highest enabled checkpoint reached.
  Color must never be the only signal, because of accessibility.
- UI-5: The app has no Dock icon and no main window.
  It is a menu bar app only.

## Popover

- UI-6: The popover shows, from top to bottom: the large percentage, a progress bar, the reset time (local time and relative, for example "resets Thu 09:00, in 2d 4h"), and the source and age of the reading.
- UI-7: The popover shows the forecast when one exists: "At this pace the limit runs out on Thursday at 14:00" or "On pace to last until the reset".
  When the forecast is suppressed, it says why.
- UI-8: The popover shows the per-day budget (RUL-12) and today's usage compared with it.
- UI-9: The popover shows a chart of utilization over the current window, with the ideal-pace line from 0 at the window start to 100 percent at the reset time.
- UI-10: The popover lists the last 20 alerts (NTF-10).
- UI-11: The popover has a Refresh button (limited by PLL-7), a Settings button, and a Quit button.
- UI-12: The popover states must each have a distinct, explicit presentation: no data yet, probing, normal, stale, unavailable with reason, limit reached, and just reset.
  An unavailable state names the reason (for example "Claude Code not found") and offers an action to fix it.

## Settings and rules editor

- UI-13: Settings include: baseline interval, pause polling, launch at login, path to `claude`, and notification permission status.
- UI-14: The rules editor lists all rules with an enable toggle, and lets the owner edit each field of a rule, add a rule, duplicate a rule, and delete a rule.
- UI-15: The rules editor writes to `rules.json` (RUL-13, RUL-14) and shows validation errors next to the field (RUL-15).
- UI-16: The rules editor shows a replay summary for the current rule set: for each rule, how many times it would have fired in the stored history (RUL-17).
- UI-17: The rules editor has a "restore defaults" action that asks for confirmation.
- UI-18: A "show rules file in Finder" action must be available.

## Quality

- UI-19: All interactive elements have accessibility labels, and the layout works with larger text sizes.
- UI-20: The UI follows light and dark appearance automatically.
- UI-21: The UI reads only from view models backed by `Core`.
  It contains no business logic.

## Acceptance criteria

- Given a reading of 0.62 observed 2 minutes ago, when the menu bar renders, then it shows `W 62%` without a stale marker.
- Given the same reading observed 2 hours ago, when it renders, then the stale marker and dimmed style appear.
- Given `claude` is not found, when the popover opens, then it shows "Claude Code not found" and a way to set the path.
- Given an invalid rule, when the editor opens, then the error appears at the field and the other rules remain editable.
- Given the owner changes a threshold in the editor, when saved, then `rules.json` contains the new value.

## Open

- O-1: Exact label text and the severity thresholds for color.
  Needs a design pass.
- O-2: Does the owner want the 5-hour window shown as well?
  The first version shows only the weekly window.
- O-3: Minimum popover contents for the first usable version, if the full list is too large for the first milestone.
