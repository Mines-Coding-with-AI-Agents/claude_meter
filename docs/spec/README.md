# Specifications

Status: Draft.
These specs were drafted from the research in `docs/research/` and the decisions in `docs/DECISIONS.md`.
The owner has not yet reviewed them in detail.
Items marked "Open" need an owner decision.

## Index

- [product.md](product.md): problem, goals, non-goals, success criteria, scenarios.
- [glossary.md](glossary.md): the shared vocabulary.
- [probe.md](probe.md): reading the weekly limit through `claude -p` (IDs `PRB-*`).
- [polling.md](polling.md): when to probe (IDs `PLL-*`).
- [history-store.md](history-store.md): persisted readings and state (IDs `HST-*`).
- [rules-engine.md](rules-engine.md): rules, forecast, and alert evaluation (IDs `RUL-*`).
- [notifications.md](notifications.md): delivering alerts (IDs `NTF-*`).
- [menu-bar-ui.md](menu-bar-ui.md): menu bar item, popover, rules editor (IDs `UI-*`).
- [packaging.md](packaging.md): building, signing, and launching the app (IDs `PKG-*`).
- [EVALUATION.md](EVALUATION.md): how the specs are evaluated before planning starts.

## Conventions

- One sentence per line.
- Each requirement has a stable ID. IDs are never reused.
- "Must" is mandatory, "should" is a strong default, "may" is optional.
- Each feature spec ends with an "Open" section listing unresolved questions.
- Related decisions are cited as `D-NNN`.
