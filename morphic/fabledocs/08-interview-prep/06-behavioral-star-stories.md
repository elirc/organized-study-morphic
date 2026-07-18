# Behavioral STAR Stories

Ten worksheets sourced from studying/contributing to Morphic. STAR = Situation, Task, Action, Result. The catch for an upskilling learner: you may not have shipped these *at a job* yet — so frame them honestly as "when I studied/contributed to an open-source codebase." Studying a real system deeply and contributing to it *is* a legitimate experience story; interviewers value it when it's specific and honest.

Each worksheet: prompts it answers → source → the STAR skeleton → evidence to cite → senior-signal detail → one-line resume bullet → rehearsal check (under 2 min? concrete? ends on impact?).

---

## Story 1: "Walk me through a complex codebase you learned."
Prompts: onboarding, ambiguity, learning speed.
Source: this entire curriculum.
S: Picked up Morphic, an AI answer engine (Next.js/React/Postgres/RLS), with no docs beyond a README. T: understand it well enough to contribute safely. A: found the entry point (`/api/chat`), traced one full flow end to end (request → auth → agent → stream → persist), mapped the layers, and identified the security boundary (RLS) before touching anything. R: could explain the whole request path and spot a real authorization gap in the feedback endpoint.
Evidence: the flow trace; the RLS mechanism (`set_config` per transaction).
Senior signal: "I read the security boundary first, not the UI."
Resume bullet: *Reverse-engineered a production AI app to a documented mental model, including its RLS-based tenant isolation.*

## Story 2: "Tell me about a bug you found by reading, not running."
Prompts: attention to detail, correctness.
Source: [architecture-critique #4](../03-architecture-and-patterns/06-architecture-critique.md).
S: reviewing context-window handling. T: figure out why long chats might degrade. A: cross-referenced the hardcoded model→window table against the current default models and found the defaults aren't listed, so they fall to a 16k window and over-truncate. R: identified a silent quality-degradation bug and a fix (per-model windows + a truncation metric).
Senior signal: found it by reading *two* files together; proposed observability so it can't silently recur.
Bullet: *Diagnosed silent answer-quality degradation to a stale config table via static analysis; proposed a metric-backed fix.*

## Story 3: "A time you made something safer / a mistake you prevented."
Prompts: risk, judgment, mistake-adjacent.
Source: Ticket M4 / Flow 5.
S: the regenerate flow deletes the old answer before the new one exists. T: prevent user-data loss. A: identified it as a sequencing flaw (not a code bug), proposed delete-after-success with a regression test that forces an LLM failure. R: a design that can't lose a committed answer.
Senior signal: "correct code, wrong order" — the fix was sequencing, not a patch.
Bullet: *Closed a data-loss window in a destructive regeneration flow by reordering and adding a failure-path regression test.*

## Story 4: "A technical tradeoff you weighed."
Prompts: tradeoffs, depth.
Source: [data-modeling](../03-architecture-and-patterns/02-data-model-and-persistence.md), Q23.
S: the message store uses a wide polymorphic table with per-tool columns. T: evaluate whether that's the right shape. A: laid out three options (wide table / jsonb / table-per-type) with their costs — DB constraints vs migration churn vs join complexity. R: concluded the wide table is defensible now but jsonb wins if tool types churn.
Senior signal: argued *both* sides and named the deciding variable (churn rate).
Bullet: *Evaluated a polymorphic-table vs jsonb data model, articulating the constraint/migration tradeoff.*

## Story 5: "A time you gave hard feedback in review."
Prompts: conflict, communication.
Source: [Review Round 1](05-debugging-and-code-review-rounds.md) / Kata 1.
S: a PR added a delete endpoint on the raw DB connection. T: block a cross-tenant vulnerability without discouraging the author. A: labeled it Blocking, explained the IDOR concretely, pointed at the exact pattern to copy, offered to pair. R: the fix used RLS + an ownership check; the author wasn't defensive.
Senior signal: the *register* — "want to pair on it?" keeps ego out so the code gets fixed.
Bullet: *Caught a cross-tenant authorization bug in review and coached the fix collaboratively.*

## Story 6: "A time you improved reliability."
Prompts: production thinking, systems.
Source: Senior Project 1 (outbox).
S: all persistence hangs off a live stream's completion, so instance death loses writes silently. T: raise the durability ceiling. A: designed an outbox written in the same transaction as the user message, drained idempotently (at-least-once + idempotent consumer). R: an RFC with dual-write rollout and a chaos test.
Senior signal: named the impossibility (exactly-once) and chose the right weaker guarantee.
Bullet: *Designed a transactional-outbox to make a best-effort persistence pipeline durable.*

## Story 7: "A time you dealt with ambiguity."
Prompts: ambiguity, initiative.
Source: SSRF investigation (Ticket M10 / Q34).
S: the fetch tool retrieves arbitrary URLs server-side — a possible SSRF surface, but exposure depends on infra I couldn't see. T: assess without over-claiming. A: labeled it "possible risk, verify exploitability", outlined the exploit (metadata endpoint) and the correct fix (check the *resolved* IP, not the hostname, against private ranges). R: a scoped, honest risk entry rather than a false alarm.
Senior signal: distinguished "possible risk" from "confirmed bug" — calibrated claims.
Bullet: *Investigated an SSRF surface, scoping the risk honestly and specifying a DNS-rebinding-safe fix.*

## Story 8: "A time you simplified / paid down tech debt."
Prompts: maintainability.
Source: Refactor Kata 2 (rate limiters) / Ticket 6.
S: three rate limiters shared ~80% code that had drifted. T: consolidate without changing behavior. A: extracted a shared core, added characterization tests first, adopted one caller at a time, and fixed the INCR/EXPIRE atomicity with a Lua script. R: one tested implementation, three preserved behaviors.
Senior signal: characterization tests *before* refactoring; incremental adoption.
Bullet: *Consolidated triplicated rate-limit logic behind one interface, preserving three behaviors via characterization tests.*

## Story 9: "A time you disagreed with a decision."
Prompts: conflict, influence.
Source: [maintainer-communication](../07-career-and-collaboration/03-maintainer-communication.md) / Kata 7 (persist-before-stream).
S: a proposal to persist synchronously before streaming "for reliability". T: push back on the latency cost. A: showed it taxes every message with a DB round-trip and that the existing onFinish already provides the durability, then proposed the outbox as the real lever — and offered to defer. R: made the case with evidence and committed to the maintainer's call.
Senior signal: disagreed with data, then deferred — influence without friction.
Bullet: *Argued against a latency-costing change with evidence, proposed the better lever, and deferred to the owner.*

## Story 10: "Something you're proud of / went beyond."
Prompts: ownership, passion.
Source: adding the first tests to an untested module (Ticket 1).
S: `message-mapping.ts` — 775 lines, zero tests, the persistence heart. T: make it safe to refactor. A: wrote round-trip tests for each part type, which surfaced the stringly-typed column-key fragility. R: coverage that enables the larger refactor and documents the bug class.
Senior signal: chose the highest-*leverage* test target, not the easiest.
Bullet: *Added the first test coverage to an untested 775-line persistence module, enabling a safe refactor.*

---

## Rehearsal rules

- Under 2 minutes. Time it.
- Concrete: a file, a metric, a specific fix — never "I improved things."
- End on impact/result, not on the problem.
- One "I" per story — your action, not the team's.
- Honesty: if it's a study/contribution story, say so; specificity makes it land regardless.

Drill: record three of these. Listen back for: did you narrate the *action* (what you specifically did) or drift into describing the *system*? Interviewers want the former.
