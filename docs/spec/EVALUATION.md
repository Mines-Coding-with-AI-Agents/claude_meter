# Spec Evaluation

Status: Draft.
Nothing in this file has been run yet.
This file defines how the specs in `docs/spec/` are evaluated before planning starts.
When planning starts, this evaluation becomes part of the plan.

## Purpose

Find defects in the specs while they are cheap to fix.
The defects to find are: ambiguity, contradiction, missing cases, untestable requirements, and defaults that behave badly.

## Layers

The layers run from cheap and automatic to expensive and human.
Each layer catches a different kind of defect.

### L1: Spec lint

A mechanical check that can run at any time.
Checks:

- Requirement IDs are unique and never reused.
- Every cross-reference resolves, such as `RUL-12`, `PRB-4`, and `D-009`.
- Every spec has Status, Related decisions, and Open sections.
- Every requirement is covered by at least one acceptance criterion.
- Vague words are flagged, such as "fast", "quickly", "appropriate", "etc.", and "should work".
- Glossary terms are used consistently.
- Formatting rules hold: no em dash and one sentence per line.

### L2: Scenario walkthroughs

Replay the scenarios in `product.md` against the draft default rules and polling numbers, using synthetic readings.
Each walkthrough is a table of time, reading, and expected alerts.
Each table later becomes a test fixture.
A walkthrough passes when the alerts match what the owner would want in that scenario.

Walkthroughs to write:

| ID | Scenario | What it checks |
|---|---|---|
| W1 | S1: 30 points in 2 hours on Monday | The spike rule fires once, with the right cooldown. A rise that falls between two polls is still caught. |
| W2 | S2: 50 percent with more than 3 days left | The `half-early` checkpoint fires once and does not repeat after a restart. |
| W3 | S3: heavy use on another device, Mac idle | Polling picks up the rise. The stale-data rule behaves correctly in between. |
| W4 | S4: quiet, steady week | No alert fires. The forecast is suppressed or calm. |
| W5 | S5: reset | Reset is detected from the usage drop. Checkpoints re-arm. The percentage is cleared at the reset time. |
| W6 | S6: `claude` missing or logged out | The UI shows an explicit unavailable state. A failure is never shown as zero. |
| W7 | Slow steady climb to 90 percent over 6 days | The projection and checkpoint rules agree and do not double-notify excessively. |
| W8 | Clock and window edges | Behavior across the reset boundary, daylight saving changes, and a Mac asleep through a reset. |

### L3: Measurements for gating unknowns

Some specs are only correct if these facts are known.
Each is a measurement, not an implementation.
Results are written into the spec and the matching Open item is closed.

| ID | Question | Spec to update |
|---|---|---|
| M1 | Does the rate limit event fire on every probe, or only when the rounded value changes? | `probe.md` O-1 |
| M2 | What quota does one probe cost? | `probe.md` O-2, `polling.md` O-2 |
| M3 | What does a limit-reached failure look like, in output and exit code? | `probe.md` O-3 |
| M4 | Which smallest model name is valid for this account? | `probe.md` O-4 |

Each probe spends a little weekly quota, so the owner approves before they run.

### L4: Blind-reader review

A fresh reviewer, with no conversation history and no research, reads only `docs/spec/`.
The reviewer is asked to:

- List ambiguities, contradictions, and missing cases.
- Answer specific "what happens when" questions, such as "the window resets while an alert is cooling down".
- Draft test cases from the acceptance criteria.

If the reviewer's answers differ from the owner's intent, the spec is ambiguous.
Reviewer output is a list of suggestions.
The owner decides what to change.

### L5: Owner review and sign-off

The owner reviews `product.md` first, because goals and success criteria drive everything else.
Then the owner reviews the draft default values and closes the Open items.
Each spec moves from Draft to Reviewed to Approved.

## Ready for planning

Planning may start when all of these are true:

- L1 passes with no errors.
- L2 walkthroughs W1 to W8 pass.
- L3 measurements M1 to M4 are recorded.
- L4 findings are resolved or consciously deferred, with the deferral written in the spec.
- Every spec is Approved by the owner, or has explicit deferred items the owner accepted.
- The owner says planning may start.

## Status

| Spec | Lint | Walkthroughs | Blind review | Owner approval | Spec status |
|---|---|---|---|---|---|
| product.md | not run | n/a | not run | not reviewed | Draft |
| glossary.md | not run | n/a | not run | not reviewed | Draft |
| probe.md | not run | W6 | not run | not reviewed | Draft |
| polling.md | not run | W1, W3 | not run | not reviewed | Draft |
| history-store.md | not run | W5, W8 | not run | not reviewed | Draft |
| rules-engine.md | not run | W1, W2, W4, W5, W7 | not run | not reviewed | Draft |
| notifications.md | not run | W2 | not run | not reviewed | Draft |
| menu-bar-ui.md | not run | W3, W6 | not run | not reviewed | Draft |
| packaging.md | not run | n/a | not run | not reviewed | Draft |

| Measurement | Result |
|---|---|
| M1 | not measured |
| M2 | not measured |
| M3 | not measured |
| M4 | not measured |

## Open

- O-1: Where the lint lives, for example a plain script in the repository, and whether `AGENTS.md` should mention it.
- O-2: Whether the Draft, Reviewed, Approved gate becomes a hard rule in `AGENTS.md`.
- O-3: Owner approval to run the L3 probes.
- O-4: Whether the walkthrough tables are stored as machine-readable fixtures from the start.
