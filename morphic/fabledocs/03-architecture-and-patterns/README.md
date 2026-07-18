# 03 — Architecture and Patterns

Where the mid-level → senior jump happens: not "does it work" but "is this the right shape, what does it cost, and what breaks first".

| File | Covers |
| --- | --- |
| [01-boundaries-and-layers.md](01-boundaries-and-layers.md) | The layer stack, what each owns, real leaks |
| [02-data-model-and-persistence.md](02-data-model-and-persistence.md) | Schema, relationships, indexes, transactions, safe schema change |
| [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) | Every validation layer; authn vs authz; isolation; junior-miss vs senior-check |
| [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) | Side-effect map; idempotency, retries, timeouts, failure visibility |
| [05-pattern-catalog.md](05-pattern-catalog.md) | 16 pattern cards (recognition training) |
| [06-architecture-critique.md](06-architecture-critique.md) | Strengths, risks, prioritized improvements — doubles as system-design interview prep |

Vocabulary this module makes precise: *boundary, contract, invariant, ownership, isolation, idempotency, consistency, blast radius*. Use them only where they earn their keep — interviewers can smell decoration.
