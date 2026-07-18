# 06 — Contribution Practice

Learning by shipping. Difficulty ladder from "locate and change one thing" to "design a subsystem a maintainer would actually accept."

| File | Level | Count |
| --- | --- | --- |
| [01-good-first-tickets.md](01-good-first-tickets.md) | Junior — locate, small change, test | 20 |
| [02-mid-level-feature-tickets.md](02-mid-level-feature-tickets.md) | Mid — cross-layer, design note first | 12 |
| [03-senior-build-projects.md](03-senior-build-projects.md) | Senior — subsystem, migration + rollout plan | 6 |
| [04-refactor-and-design-katas.md](04-refactor-and-design-katas.md) | Senior — critique & restructure | 10 |

**Contribution realism** — every item answers: why useful, why small blast radius, what pattern it follows, how it's tested, what makes a maintainer reject it. No drive-by refactors. Each also carries an **Interview story potential** line — the STAR story you'd get from completing it (feeds [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)).

Start with 2–3 junior tickets before anything else — they teach the pre-PR ritual ([bun lint && typecheck && format:check && build && test](../02-stack-and-language-mastery/04-tooling-and-build-system.md)) more than any doc can.
