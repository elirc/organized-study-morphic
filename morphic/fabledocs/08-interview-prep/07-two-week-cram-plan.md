# Two-Week Cram Plan

For a candidate with a mid-level fullstack interview in two weeks who has this repo as their study lab. ~2–3 focused hours/day. Adjust to your gaps — if you're weak on React, weight week 1 toward module 02; strong on frontend, weight toward system design.

Ground rule: **say answers aloud.** Silent reading hides the gaps a live interview finds. Trace every anchor before answering an anchored question.

## Week 1 — build the mental model + core answers

**Day 1 — Orientation.** Do [00-fast-track.md](../00-fast-track.md) (run it or read it fully). Read [01-system-map.md](../01-codebase-cartography/01-system-map.md) + [03-domain-glossary.md](../01-codebase-cartography/03-domain-glossary.md). Output: draw the flow diagram from memory.

**Day 2 — The core flow.** [05-key-flows.md](../01-codebase-cartography/05-key-flows.md) Flows 1–3. Do the Flow 1 drill (write 14 steps from memory). This is your "walk me through the codebase" story.

**Day 3 — More flows + JS/TS.** Key Flows 4–7. Then [08 Q1–Q6](01-js-ts-node-deep-dive.md) — say each aloud, trace anchors.

**Day 4 — JS/TS + language model.** [08 Q7–Q12](01-js-ts-node-deep-dive.md) + [02-stack/01-language-runtime-model.md](../02-stack-and-language-mastery/01-language-runtime-model.md). Do the event-loop and parallel-vs-serial drills.

**Day 5 — Frontend.** [02-stack/02-framework-mental-models.md](../02-stack-and-language-mastery/02-framework-mental-models.md) + [08 Q13–Q22](02-frontend-framework-questions.md). Focus: server/client split, stale closures, streaming render.

**Day 6 — API & data.** [03-architecture/02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md) + [08 Q23–Q31](03-api-and-data-modeling-questions.md). Focus: the message data model + transactions + streaming endpoint.

**Day 7 — Checkpoint #1.** Self-assess: can you, without notes, (a) trace the chat flow, (b) explain RLS, (c) answer Q1/Q5/Q13/Q23/Q31? If any is shaky, reread that anchor today. **Mock system design** (30 min, whiteboard): [04-system-design](04-system-design-from-this-repo.md), steps 1–8 cold, then grade.

## Week 2 — depth, practice, polish

**Day 8 — Auth/security.** [03-architecture/03-validation-auth](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) + [05-quality/05-security-checklist](../05-quality-engineering/05-security-checklist.md) + [08 Q32–Q35](03-api-and-data-modeling-questions.md). Do the IDOR trace aloud.

**Day 9 — Patterns + critique.** [03-architecture/05-pattern-catalog](../03-architecture-and-patterns/05-pattern-catalog.md) (recognition drill) + [06-architecture-critique](../03-architecture-and-patterns/06-architecture-critique.md). These power system-design and "tradeoff" answers.

**Day 10 — Debugging round.** [05-debugging](05-debugging-and-code-review-rounds.md) Debugging Rounds 1–3, timed. Also [05-quality/03-systematic-debugging](../05-quality-engineering/03-systematic-debugging.md). Record yourself; check you narrow before guessing.

**Day 11 — Review round.** [05-debugging](05-debugging-and-code-review-rounds.md) Review Rounds 1–2, timed. [04-code-reading-gym/04-review-katas](../04-code-reading-gym/04-review-katas.md) — do 4 more katas, sorting "wrong idea" vs "wrong default".

**Day 12 — Behavioral.** [06-behavioral-star-stories](06-behavioral-star-stories.md) — write and rehearse your top 5 (aim to cover: complex codebase, tradeoff, hard feedback, mistake/data-loss, disagreement). Time each under 2 min.

**Day 13 — Second mock + weak spots.** Full 30-min system design again (try variation prompt "add multi-tenancy"). Redo any question card you flubbed. One more debugging round cold.

**Day 14 — Checkpoint #2 + light review.** Self-assess against the checklist below. Skim, don't cram — reread your own flow diagram, your 5 STAR bullets, and the pattern catalog one-liners. Rest before the interview.

## Self-assessment checklists

**Day 7 (must-haves before week 2):**
- [ ] Trace the authenticated chat flow, 10+ steps, from memory.
- [ ] Explain RLS + `withRLS` in 60 seconds with the `set_config` detail.
- [ ] Answer Q1 (await/event loop) and Q5 (any vs unknown) with a repo anchor.
- [ ] Sketch the chat data model (3 tables) and name one tradeoff.

**Day 14 (interview-ready):**
- [ ] System design: design the answer engine end to end in 30 min, raising citations-as-contract and durability unprompted.
- [ ] Debugging: narrate the reproduce→narrow→fix method on a cold prompt.
- [ ] Review: find the authz bug in Kata 1 and phrase it kindly.
- [ ] 5 STAR stories, each under 2 min, ending on impact.
- [ ] Answer any card Q1–Q41 at the mid-level bar (definition + repo example + tradeoff).
- [ ] Name 3 real weaknesses of the architecture and a fix for each.

If you can check every Day-14 box, you're answering at the level the loop is testing for. The edge you'll have over other candidates: every answer is backed by production code you actually read — lean into that ("in a codebase I studied deeply, they handle this by…").

## If you have less than two weeks

3-day version: Day 1 → fast-track + key flows 1–4 + Q1–Q12. Day 2 → Q13–Q35 + system design walkthrough. Day 3 → one debugging round, one review round, 3 STAR stories, checkpoint #2. You'll be thinner on depth but have the anchors that matter.
