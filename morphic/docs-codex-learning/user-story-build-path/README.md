# User Story Build Path

This suite gives you ten realistic tickets to implement later. They are grounded in Morphic's actual domain: chat, search mode, upload, feedback, history, streaming, persistence, and safety.

The stories progress from easy UI changes to expert architecture work. Do not ask Codex to implement the whole story first. Read the relevant files, write a short plan, make a small change, run checks, then ask for review.

## Difficulty Progression

- Stories 1-3: small UI/validation changes, usually 1-2 files.
- Stories 4-5: medium-easy UI features using existing data.
- Stories 6-7: API, state, or validation changes.
- Stories 8-9: full-stack persistence-oriented features.
- Story 10: architecture-sensitive safety/performance work.

## How To Know A Story Is Done

- Acceptance criteria are manually verifiable.
- Relevant tests are added or intentionally documented as not needed.
- No unrelated files are changed.
- The diff is explainable in five minutes.
- You can name the boundary you changed and the risk you tested.

## Use Codex Without Outsourcing Your Brain

Use this prompt:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z. Do not give me the solution immediately. Ask me questions that help me reason through the problem. Point me to the relevant files and patterns. Give hints in stages.
```

Ask Codex to explain existing patterns before generating code. Ask it to review your attempted implementation. Ask it to write tests after you have a working change. Do not blindly paste code; make yourself explain every line in the diff.

