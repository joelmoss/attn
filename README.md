# attn

A macOS app that shows what needs your attention. A live ring in the Dock, a menu bar panel and a small pill under the notch light up when something is waiting on you. Sources include Claude Code agents, GitHub review requests and failed CI, and any app or script that calls the `attn` command.

**Status:** planned, not built yet.

## The plan

- [`docs/designs/attn.md`](docs/designs/attn.md) — the design doc. It covers the problem, constraints, approved scope, the activity model, the provider/Sources architecture, the UI spec and tokens, and the decision ledgers from each review: office hours, CEO, eng (×3) and design.
- [`docs/designs/ceo-plan.md`](docs/designs/ceo-plan.md) — accepted and deferred scope.
- [`docs/designs/test-plan.md`](docs/designs/test-plan.md) — what has to be tested, with a value card for each test.
- [`docs/designs/spike-report.md`](docs/designs/spike-report.md) — results of the readability and platform spike.
- [`docs/designs/mockups/`](docs/designs/mockups/) — approved mockups: panel, live pill, first run and Settings › Sources. Where the mockups and the plan disagree, the plan wins.

## Where to start

1. The **readability and platform spike** (Next Steps #1, task C1). It answers the plan's open question OV-1: whether later activity on a GitHub thread you were mentioned in should light the ring again.
2. Then the tasks under "Implementation Tasks" in the design doc, in the order given by its parallelization notes.
