# Architectural Cartographer

This suite is a top-down onboarding guide for Morphic. It teaches the system as a set of connected responsibilities: routes choose data, components hold interaction state, API routes enforce boundaries, agents perform search/reasoning, and database actions persist chat history.

Use it when you want to answer questions like:

- Where does a chat message go after I press submit?
- Which files own authentication, search mode, persistence, and rendering?
- What should I inspect before changing an API route or database schema?
- How would a senior engineer critique this architecture?

## Reading Order

1. `JOURNAL.md` for the inspection narrative.
2. `00-reading-map.md` for the file order and review checklist.
3. `01-junior-engineer.md` for setup, folders, entry points, and beginner anatomies.
4. `02-mid-level-engineer.md` for full-stack flow, contracts, and diff reading.
5. `03-senior-engineer.md` for critique, risks, performance, security, tests, and ownership.
6. `04-reference-suite.md` for compact guides you can reuse during real work.

## Checkpoints

Do not skip the Socratic checkpoints. Answer them before reading the self-grade rubric. Strong answers should name files, line ranges, data shapes, and risks. "It calls the backend" is a beginner answer. "The AI SDK transport prepares a body with `trigger`, `chatId`, and either `message` or `messages`, then `app/api/chat/route.ts` branches between authenticated persistence and guest ephemeral streaming" is a stronger answer.

## Difference From Other Suites

The mission suite trains by tasks. The user-story suite gives future implementation tickets. This suite is the map: it tells you what exists, why it is shaped this way, and where to be cautious.

