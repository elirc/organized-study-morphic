# Code Review Mindset

Review in layers, in order. Stop wasting a senior's time by nitpicking style on a PR that has a correctness bug.

1. **Does it work?** — does it do the thing? (run it / read the happy path)
2. **Is it correct?** — edge cases, error paths, concurrency, the invariant.
3. **Will it stay correct?** — tests, types, does it hold under a plausible future change?
4. **Does it fit?** — follows repo patterns, right layer, no boundary leak.
5. **Is it kind to the next maintainer?** — naming, comments-that-state-constraints, PR description.

Severity language: **Blocking** (won't merge), **Important** (should fix), **Optional/nit** (take or leave). Label every comment so the author can triage.

## Repo-specific review checklist

Reading a Morphic PR, check:

- [ ] **Authz:** new data path under RLS? mutation verifies ownership? (`withOptionalRLS(null,…)` justified?) — the #1 thing to catch, per [03-validation-auth-and-permissions.md](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).
- [ ] **Layer fit:** does `lib/db` code return `Response` objects, or does a component read DB columns? (boundary leaks, [01-boundaries-and-layers.md](../03-architecture-and-patterns/01-boundaries-and-layers.md))
- [ ] **Idempotency/retry:** new write — safe to run twice? retried only if idempotent?
- [ ] **The contracts:** did it touch the citation format, the spec catalog, the `/api/chat` body shape, or the DB schema? Those are multi-party — all sides + a test.
- [ ] **Silent failure:** new `catch {}` without a log? new soft-fail without a metric?
- [ ] **Tests:** pure logic → per-branch; IO → contract-level; the untested areas (routes, tools, mapping) especially want coverage.
- [ ] **Migrations:** additive? backward-compatible with the running app version? old rows still load?
- [ ] **CI ritual:** lint/typecheck/format/build/test all green (the author should have run [the pre-PR ritual](../02-stack-and-language-mastery/04-tooling-and-build-system.md)).
- [ ] **Security:** SSRF/XSS/upload items from [05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).

## Example comments (kind + specific + actionable)

Blocking, but generous:
> **Blocking:** this query runs on the bare `db` connection, so RLS won't apply and a user could read another user's chats by id. Wrap it in `withRLS(userId, …)` like `getChats` does ([db/actions.ts#L275-L283](../../lib/db/actions.ts#L275-L283)). Want to pair on it?

Important, with the why:
> **Important:** the empty `catch` here means a Redis outage looks like success and we'd never know. Mirror the analytics handler — `catch (e) { console.error('…', e) }` ([route.ts#L202-L205](../../app/api/chat/route.ts#L202-L205)) — and consider a metric.

Optional, clearly optional:
> **Nit (optional):** could pull `100` into a named `DAILY_CHAT_LIMIT`-style constant for readability. Not blocking.

Asking, not demanding:
> Curious about the ordering here — does the delete need to happen before the stream, or could we delete after success to avoid losing the old answer if regeneration fails? (See Flow 5.) Genuinely asking, might be a reason I'm missing.

## Anti-patterns in your own reviews

- Nitpicking style when CI's Prettier already enforces it — waste.
- "This is wrong" without the fix or the why — unkind and unactionable.
- Blocking on preference — reserve Blocking for correctness/security.
- Rewriting the PR in comments — suggest, don't dictate.
- Silence on the good parts — say what's well done; it calibrates trust.

## What "kind" buys you

Reviews are where teams build or burn trust. The register — "want to pair on it?", "might be a reason I'm missing" — keeps the author's ego out of the loop so the *code* gets fixed. Interviewers probe this directly: "how do you give hard feedback?"

## Drill

Take [Kata 1](../04-code-reading-gym/04-review-katas.md) (delete-all endpoint) and write all three severity levels of comments in this register. Self-grade — Strong: your Blocking comment names the vuln, the fix, the pattern to copy, and stays warm.

## Interview angle

- "How do you review code?" → the five layers + severity labels; cite the authz-first checklist. ([Q47](../08-interview-prep/06-behavioral-star-stories.md) area)
- "Tell me about hard feedback you gave." → a STAR from a kata; the register is the point.
