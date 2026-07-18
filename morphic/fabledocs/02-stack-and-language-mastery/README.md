# 02 — Stack and Language Mastery

Mental models for the stack, from first principles, always anchored to a real Morphic file. Each page: model → where the repo uses it well → sharp edges → pitfall checklist → drills → interview angle.

| File | Covers |
| --- | --- |
| [01-language-runtime-model.md](01-language-runtime-model.md) | Event loop, promises, parallel vs serial, generators, Node vs Bun vs browser |
| [02-framework-mental-models.md](02-framework-mental-models.md) | React 19 render/commit, hooks discipline, Next.js App Router (RSC, server actions, caching, middleware) |
| [03-type-system-and-contracts.md](03-type-system-and-contracts.md) | Inference, narrowing, generics, `unknown` vs `any`, Zod at boundaries |
| [04-tooling-and-build-system.md](04-tooling-and-build-system.md) | Bun, ESLint 9, Prettier-as-CI-gate, Vitest, Drizzle Kit |

Rule for this module: **no concept without an anchor**. If you can't point at a line where the event loop matters, you don't know it yet — you've only read about it.
