# Product

Status: Draft.

## Problem

The owner uses Claude Code on a Mac, a Windows desktop, and a phone.
The weekly usage limit is shared across them.
In the past the owner burned the whole weekly limit without noticing and had to wait days before using Claude Code again.
Nothing warns the owner while this is happening.

## Goal

Show the weekly limit state at a glance in the macOS menu bar, and warn the owner early enough to change behavior.

## Goals

- G1: The menu bar shows the current weekly usage percentage and how fresh it is.
- G2: The popover shows time until reset, a forecast, and a per-day budget.
- G3: The owner receives notifications from rules that the owner can read and edit, in a file or in the app.
- G4: The app needs no API key and stores no credentials.
  It works with the owner's Claude Code subscription only (D-001, D-006).
- G5: The app costs little weekly quota to run.
- G6: Stale, missing, or uncertain data is always shown as such, never as a confident number (D-009, D-010).

## Non-goals

- N1: No Windows, Linux, iOS, or Android app.
- N2: No tracking of cost in dollars or per-project attribution.
- N3: No multi-user or cloud sync.
- N4: No reading of Claude Code transcripts in the first version.
- N5: No blocking or throttling of Claude Code.
  The app only informs.
- N6: No distribution to other users, no notarization.

## Success criteria (draft)

- SC1: The owner is warned before the weekly limit runs out in a heavy-use week, with time left to change behavior.
- SC2: The menu bar number is never silently wrong.
  If it is not current, the app says so.
- SC3: A notification for the same condition is shown once, including across app restarts.
- SC4: Running the app for a full week uses a small, known amount of weekly quota (to be measured, see `probe.md` Open).

## Scenarios

- S1, big refactor: on Monday the owner starts a heavy session.
  Usage rises 30 points in 2 hours.
  The owner gets a spike warning while there are days left in the week.
- S2, checkpoint early: the owner reaches 50 percent with more than 3 days left.
  The owner gets a notice that the pace is too high.
- S3, other devices: the owner works on the Windows desktop all day.
  The Mac is idle.
  The app polls, picks up the higher usage, and notifies.
- S4, quiet week: usage is low and steady.
  The app shows a calm state and sends nothing.
- S5, after reset: the weekly window resets.
  The percentage drops, old alerts re-arm, and a reset notice may appear.
- S6, no data: `claude` is missing or logged out.
  The app shows an explicit unavailable state.

## Constraints

- macOS only.
  Subscription login only.
  No full Xcode is assumed (D-007).
- The only data source is the installed `claude` binary (D-006, D-007).

## Related decisions

D-001, D-005, D-006, D-007, D-008, D-009, D-010, D-011.

## Open

- O-1: Confirm the goals, non-goals, and success criteria with the owner.
- O-2: Does phone and web usage count toward the same limit?
  Official docs do not name mobile.
  The probe will show it over time.
- O-3: Which minimum macOS version to support (see `packaging.md`).
