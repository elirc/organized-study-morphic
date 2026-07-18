# Debugging & Code Review Rounds

Practical rounds are increasingly standard for mid-level loops: an interviewer hands you a bug or a PR and watches *how you think*. Method matters more than the answer. Time each round; narrate aloud.

---

## Debugging Round 1 (15 min): "Chats vanish after refresh"

Setup: read [debugging Scenario 1](../05-quality-engineering/03-systematic-debugging.md).
Interviewer's opening: "Users say their chat disappears after they refresh. Where do you start?"
What they're scoring: do you split write-vs-cache before touching code?
Your script: "First, reproduce and narrow to a layer. I'd query the DB for the message row. If it's there, it's a read/cache problem — Morphic caches chat loads with a 60s TTL and invalidates by tag, so I'd check whether the relevant write path calls `revalidateTag`. If the row's absent, it's persistence — which runs in the stream's `onFinish`, so I'd suspect the instance died before `maxDuration` and check the error logs that path emits."
Interviewer follow-ups: "It's in the DB." → cache/revalidation; compare a path that *does* revalidate ([shareChat](../../lib/actions/chat.ts#L265-L282)) with one that might not (title update). "It's not in the DB." → onFinish failure / instance death; propose the outbox.
Grading — Strong: you named the binary split and the *single cheapest probe* (one SQL query) that decides it, before proposing any fix.

## Debugging Round 2 (15 min): "Regenerate lost my answer"

Setup: [Scenario 2](../05-quality-engineering/03-systematic-debugging.md) / Flow 5.
Opening: "A user regenerated a response, the model errored, and now the original answer is gone. Bug?"
Scoring: can you distinguish a code bug from a *sequencing* flaw?
Script: "I'd confirm the ordering: does the delete fire before the new stream? In this design it does — the old assistant message is deleted in message-prep, then the stream runs. So this isn't a bug in delete or stream individually; it's that a destructive op has no compensation if the follow-up fails. Fix: delete after successful regeneration, or soft-delete, plus a regression test that forces the model to fail and asserts the old answer survives."
Follow-ups: "How would you test it?" → mock the model to reject, assert prior message exists. "Tradeoffs of soft-delete?" → storage + sweep vs simplicity.
Grading — Strong: "correct code, wrong order" framing + a concrete regression test.

## Debugging Round 3 (15 min): "Long chats get dumber"

Setup: [Scenario 3](../05-quality-engineering/03-systematic-debugging.md) / Pattern 10.
Opening: "Later answers in a long conversation ignore earlier context. Why?"
Script: "Is context being dropped or is the model just weak? I'd log message count before/after truncation, then check the model id against the context-window table. Current default models aren't in that table, so they fall to a 16k window and get over-truncated — a stale-config bug that silently hurts quality. Fix: source windows per model + a metric on truncation."
Follow-ups: "How did you find the stale table?" → reading two files together (default model + the window map). "Why is this hard to notice?" → it's a silent branch; you must log the decision point.
Grading — Strong: you instrument the *decision*, not the symptom.

---

## Review Round 1 (20 min): the delete-all endpoint

Setup: [Kata 1](../04-code-reading-gym/04-review-katas.md).
Opening: "Review this PR adding `DELETE /api/chats/[id]/messages`."
Scoring: do you catch the authz/RLS hole first, and phrase it kindly?
Your findings, in order: **Blocking** — bare `db` connection, no RLS, no ownership check → cross-tenant delete by id (IDOR). Fix: `withRLS` + ownership check like the existing delete actions. **Important** — no `revalidateTag`. **Optional** — return a count.
Model comment to deliver verbally:
> "The big one is blocking: this runs on the raw connection so RLS won't engage — a user could delete another user's messages by guessing the chat id. Wrap it in `withRLS(userId, …)` and add the ownership check `deleteMessagesFromIndex` uses. The rest are minor — missing cache invalidation, and maybe return a count for consistency."
Follow-ups: "How would you test the authz fix?" → the RLS integration test pattern (real Postgres, user A can't touch user B's rows).

## Review Round 2 (20 min): "cache forever" / "owner connection"

Setup: [Katas 2 & 6](../04-code-reading-gym/04-review-katas.md).
Opening: two small PRs — one sets the model cache TTL to Infinity, one switches to the DB owner connection "to fix RLS slowness."
Scoring: recognizing that a *cleanup-looking* PR is a security regression.
Findings: TTL→Infinity is **Important** (new models never appear; suggest single-flight instead) — not a security issue. Owner-connection is **Blocking** — table owners bypass RLS entirely, silently disabling all tenant isolation; the fix for "RLS is slow" is indexing the policy subquery (already done) and *measuring*, not removing the security boundary.
Grading — Strong: you sort "good idea, wrong default" (TTL) from "dangerous disguised as cleanup" (owner) and reserve Blocking for the security one.

---

## How to run any practical round

1. Restate the problem in your words (confirms understanding, buys thinking time).
2. State your method before acting ("narrow to a layer / review in severity order").
3. Name the cheapest next probe or the highest-severity finding first.
4. Propose a fix *and* a test.
5. Say the tradeoff.

Interviewers hire the candidate whose *process* they'd trust on a real incident or PR. The anchors let you sound like someone who's debugged this exact system — because you have.

Drill: have someone hand you one round cold and time you. Record yourself; listen for whether you narrowed before guessing.
