# Writing PRs and RFCs

## The PR description that gets merged fast

Reviewers approve what they understand. Structure:

```
## What
One sentence: the change.

## Why
The problem/ticket. Link it.

## How
The approach in 2–4 bullets. Note anything non-obvious.

## Testing
What you ran (bun run test, manual steps) and what you added.

## Risks / rollback
Blast radius; how to revert; anything reviewers should scrutinize.

## Follow-ups
Deliberately-out-of-scope items (so reviewers don't ask).
```

Worked example (for Ticket 2, clamp pagination):

> **What:** Clamp `/api/chats` `limit` to `[1,50]` and `offset` to `≥0`.
> **Why:** `limit` was unbounded ([chats/route.ts#L13-L14](../../app/api/chats/route.ts#L13-L14)); a large value forces a slow query. Ticket #2.
> **How:** `Math.min(50, Math.max(1, parsed))`, `NaN`→default. Max lives in a `MAX_PAGE_SIZE` constant.
> **Testing:** added a test asserting `limit=100000`→50 and `limit=abc`→20; `bun run test app/api/chats` green; full ritual green.
> **Risks/rollback:** none behind a flag — revert the commit. Normal values unchanged (regression test proves it).
> **Follow-ups:** keyset pagination is a separate ticket (M11).

Why this works: the reviewer knows the blast radius (one route), that behavior for valid inputs is unchanged (the test), and that you're not sneaking in scope.

## Commit messages

- Imperative subject ≤ ~72 chars: "Clamp chat pagination limit to 50".
- Body: *why*, not what the diff already shows.
- One logical change per commit; don't mix a refactor with a fix (reviewers can't separate them).

## When to write an RFC instead of just a PR

Write an RFC (design doc reviewed *before* code) when the change:
- alters a **contract** (DB schema, `/api/chat` body, citation format, spec catalog);
- has a **migration** or rollback complexity;
- touches **security/authz** (RLS, new endpoints, SSRF surface);
- is a **senior project** (any of [03-senior-build-projects.md](../06-contribution-practice/03-senior-build-projects.md)).

Everything smaller: a good PR description is enough. Over-RFC-ing small changes wastes the team; under-RFC-ing a migration wastes a rollback.

## RFC template (tailored to this repo)

```
# RFC: <title>

## Problem
What's broken/missing. Evidence with anchors.

## Goals / Non-goals
What this does and explicitly does not do.

## Proposed design
The approach. Diagrams if it helps.

## Layer-by-layer impact
- Schema / migration:
- API / actions:
- Streaming / agent:
- UI:
- Tests:

## Data & migration plan
Additive first? Backfill? Backward-compatible with the running app? (No down-migrations exist — plan expand/contract.)

## Authorization & security
RLS impact; new endpoints' auth; SSRF/XSS considerations.

## Reliability
Idempotency, retries, failure visibility, blast radius.

## Rollout / rollback
Flag? Dual-write? How to revert safely.

## Alternatives considered
At least one, and why rejected.

## Open questions
```

The two sections that separate senior from mid: **Alternatives considered** (shows you didn't tunnel-vision) and **Rollout/rollback** (shows you own it in production). An RFC without a rollback plan reads as junior.

## Interview angle

- "How do you propose a big change?" → RFC-first for contracts/migrations, with alternatives + rollback; PR-description otherwise. ([Q48](../08-interview-prep/06-behavioral-star-stories.md) area)
- "What's in a good PR?" → the six sections; the *risks/rollback* line is what reviewers value most.

Drill: write the full RFC for one senior project (Project 1 outbox is ideal — [03-senior-build-projects.md](../06-contribution-practice/03-senior-build-projects.md)). Self-grade — Strong: your rollback plan is concrete (dual-write, stop the drainer) and you rejected at least one real alternative (queue vs DB-polling outbox) with a reason.
