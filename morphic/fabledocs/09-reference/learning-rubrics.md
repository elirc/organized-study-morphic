# Learning Rubrics

Observable behaviors, not vague traits. Use these to place yourself honestly and to know what "the next level" looks like. The **Interview-ready** column is the bar a mid-level loop tests.

## Skill: Understanding this codebase

| Skill | Junior (locate) | Mid (modify safely) | Senior (own it) | Interview-ready |
| --- | --- | --- | --- | --- |
| Navigation | finds a file by feature name | traces a full flow across layers unaided | predicts blast radius of a change before making it | narrates the chat flow with anchors, no notes |
| The data model | reads the three tables | explains parts polymorphism + why | argues wide-table vs jsonb tradeoff | designs the model + one tradeoff in <5 min |
| Auth/isolation | knows RLS exists | explains `withRLS`/`set_config` mechanics | audits every `withOptionalRLS(null)` site | traces an IDOR to the invariant that stops it |
| Streaming/async | knows it streams | explains optimistic-create + onFinish persist | designs the outbox durability upgrade | answers "server dies mid-request" with the failure tree |

## Skill: Writing code

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Correctness | happy path works | handles edge cases + errors | identifies the invariant and protects it | states the invariant unprompted in review |
| Types | annotates | designs unions that make bad states unrepresentable | knows where types end and Zod begins | explains any-vs-unknown with blast-radius framing |
| Tests | writes a happy-path test | per-branch + mocked-IO contract tests | knows what NOT to test; kills flake at the source | states a testing philosophy + this repo's gaps |
| Async | uses async/await | chooses parallel vs serial deliberately | reasons about cancellation, timeouts, idempotency | "make a write retry-safe" → idempotency + anchor |

## Skill: Judgment (the mid→senior jump)

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Tradeoffs | "it works" | names one cost of an approach | argues both sides, names the deciding variable | every design answer includes a tradeoff |
| Risk | doesn't see it | flags obvious issues | prioritizes by risk×effort, labels confidence | "a risk I found" story, honestly calibrated |
| Boundaries | writes in one file | keeps code in the right layer | spots leaks + proposes the fix | names a real leak + its fix from a repo |
| Reliability | assumes success | handles failure per operation | designs compensation/idempotency/observability | "resilient but unobservable" as a named stage |

## Skill: Collaboration

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Code review | style nits | catches correctness/authz, labels severity | reviews for "will it stay correct", kindly | finds the authz bug in a kata + phrases it warmly |
| PRs/RFCs | "fixed stuff" | full PR description w/ risks | RFC-first for contracts, alternatives + rollback | describes the six-section PR + why rollback matters |
| Communication | asks vaguely | shows-work + asks a real decision | disagrees with evidence, then defers | STAR story of a disagreement resolved |

## Self-assessment checklist (score yourself 0–3 per row)

Understanding: [ ] trace chat flow from memory · [ ] explain RLS mechanics · [ ] sketch data model + tradeoff · [ ] explain optimistic persistence.
Code: [ ] any-vs-unknown with anchor · [ ] discriminated-union narrowing example · [ ] testing philosophy + repo gaps · [ ] parallel-vs-serial with a repo example.
Judgment: [ ] name 3 architecture risks + fixes · [ ] spot a boundary leak · [ ] argue a data-model tradeoff both ways · [ ] label a risk's confidence honestly.
Collaboration: [ ] review a kata in 3 severity levels · [ ] write a full PR description · [ ] 5 STAR stories under 2 min.

**Reading the score:** mostly 0–1 → you're at "locate", stay in modules 00–02 + drills. Mostly 2 → "modify safely", do mid tickets + interview cards. Consistent 3 → "own it", do a senior project + the critique, and you're answering above the mid-level bar.

## The one-line ladder (memorize it)

- Junior: *"How do I make it work?"*
- Mid: *"Is this the right pattern, and what does it cost?"*
- Senior: *"What does this commit us to, who pays, and how do we reduce the risk?"*

Mid-level interviews hire for the second question and reward the third. Every drill in this curriculum is practice at asking them.
