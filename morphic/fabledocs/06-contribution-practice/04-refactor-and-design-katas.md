# Refactor and Design Katas

Ten senior katas. Unlike tickets, these have no "right" implementation — they train judgment. Each has self-grading criteria. Do them on paper or in a branch you throw away; the deliverable is the *reasoning*.

---

## Kata 1: Fix the boundary leak (Brave conditional)
Target: [search.ts#L114-L127](../../lib/tools/search.ts#L114-L127).
Task: redesign `SearchProvider.search` so the tool layer stops special-casing Brave. Options-object parameter? Per-provider capability flags?
Self-grade — Strong: your interface expresses `content_types`/`type` for *all* providers, the others ignore them, the tool loses its `if`, and you name the migration (touch all 5 provider classes) and a characterization test to pin current results first.

## Kata 2: Extract a rate-limit core (outbox for the immortal key)
Target: the three limiters.
Task: one `checkDailyLimit(scope, id, limit)` returning a typed result; fix INCR/EXPIRE atomicity.
Self-grade — Strong: shared core + a Lua/`MULTI` atomic increment + preserved per-caller response mapping (401 vs 429, headers, analytics), adopted one caller at a time.

## Kata 3: Split the mega-file (message-mapping)
Target: [message-mapping.ts](../../lib/utils/message-mapping.ts) (775 lines).
Task: table-driven descriptors replacing the parallel switches; type-safe column keys.
Self-grade — Strong: a descriptor type that makes `tool_${name}_input` keys checked, a round-trip test corpus gating the refactor, and an honest note on where type-safety still can't reach (dynamic/MCP names).

## Kata 4: Design the outbox (eventing change)
Target: the persistence path (Senior Project 1).
Task: write the RFC — table, same-transaction write, drainer, delivery semantics.
Self-grade — Strong: at-least-once + idempotent consumer justified against exactly-once impossibility, dual-write rollout, chaos test named.

## Kata 5: Remove the error-string parsing (structured errors)
Target: [chat.tsx#L135-L214](../../components/chat.tsx#L135-L214).
Task: design a typed error channel from server to client through the AI SDK transport.
Self-grade — Strong: you identify *how* structure survives the transport (data part / header / custom serialization), keep a fallback during migration, and enumerate every error branch that must move.

## Kata 6: Improve type safety at the DB-json boundary
Target: [schema.ts#L173-L198](../../lib/db/schema.ts#L173-L198) (`$type<any>()`).
Task: replace `any` json columns with validated shapes (Zod-on-read, or typed generics).
Self-grade — Strong: you weigh validate-on-read cost vs safety, note that old rows may not match current shapes (migration/tolerance), and pick a boundary to enforce at rather than sprinkling casts.

## Kata 7: Design a schema migration (chat folders/tags)
Task: users organize chats into folders. Design column vs join-table, indexes, RLS impact, backfill, expand/contract steps.
Self-grade — Strong: additive-first migration, no RLS change (same owner), cache-invalidation audit, and a keyset-friendly index; you state the rollback is a forward inverse migration (no down files exist — [drizzle/](../../drizzle/)).

## Kata 8: Reduce the N+1 in clearChats
Target: [lib/actions/chat.ts#L229-L233](../../lib/actions/chat.ts#L229-L233).
Task: single RLS-scoped delete; preserve error semantics.
Self-grade — Strong: you note the behavior change (no per-chat ownership message), add a characterization test first, and confirm cascades cover messages/parts.

## Kata 9: Write an RFC for per-model context windows
Target: [context-window.ts](../../lib/utils/context-window.ts) (Ticket M3).
Task: propose sourcing windows dynamically vs a generous default; how to detect the stale-table regression in the future.
Self-grade — Strong: you propose a metric/log on unknown-model fallback so this can't silently rot again, and budget-math tests.

## Kata 10: Review a flawed PR (pick a review kata, defend it)
Target: any kata in [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md).
Task: write the full review — blocking/important/optional, with kind specific comments and proposed fixes — then have a peer (or your future self) argue back.
Self-grade — Strong: you distinguish "wrong idea" from "good idea, wrong defaults" and your blocking findings are truly blocking (security/correctness), not taste.

---

## How to use these

Do one per week alongside tickets. For each, produce a one-page writeup: problem, chosen approach, alternative rejected + why, migration, tests, rollback. That page *is* the RFC and *is* the interview story. The senior signal is never the code — it's the decision log and the risk you retired. Cross-links: these feed [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) (design) and [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md) (STAR).
