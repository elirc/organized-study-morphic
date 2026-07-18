# Codex Learning Documentation for Morphic

This folder is a training guide for learning Morphic as a real codebase, not a replacement for the project README. It was created as a documentation-only learning layer. Existing docs, production code, tests, and configuration were intentionally left alone.

Morphic is a full-stack TypeScript answer engine built with Next.js App Router, React, Vercel AI SDK, Supabase auth, PostgreSQL through Drizzle ORM, Redis-backed advanced search caching, Tailwind/shadcn UI, and Vitest tests. The most important learning path is the chat flow: `app/page.tsx:6-18` renders `components/chat.tsx:38-586`, which sends messages to `app/api/chat/route.ts:20-229`, streams through `lib/streaming/create-chat-stream-response.ts:33-235`, and persists via `lib/streaming/helpers/persist-stream-results.ts:11-109`.

## The Four Suites

| Suite | Best Use |
| --- | --- |
| `architectural-cartographer/` | Top-down system understanding, feature traces, architecture critique, debugging, and ownership thinking. |
| `mission-learning-path/` | Active missions that make you read, trace, explain, review, and debug instead of passively consuming notes. |
| `user-story-build-path/` | Ten realistic feature tickets you can implement later, with implementation plans but no code. |
| `technology-best-practices/` | How this repo uses React, TypeScript, Next.js, API routes, Drizzle, Supabase, AI SDK, Zod, Tailwind, Docker, and Vitest in real files. |

## Recommended Reading Order

1. Read `architectural-cartographer/00-reading-map.md`.
2. Work through `architectural-cartographer/01-junior-engineer.md`.
3. Complete Missions 1-8 in `mission-learning-path/01-tier-1-junior-missions.md`.
4. Read `technology-best-practices/01-technology-map.md`.
5. Trace the full chat flow in `architectural-cartographer/02-mid-level-engineer.md`.
6. Pick Story 1 from `user-story-build-path/01-stories.md` and implement it yourself in a separate working session.
7. Use `architectural-cartographer/03-senior-engineer.md` when practicing review, debugging, and ownership judgment.

## How To Use These Docs With Codex

Ask Codex to help you think, not to replace your thinking. A strong learning prompt is:

```text
I am studying components/chat.tsx. Ask me questions about the useChat transport before explaining it. If I get stuck, give me one hint at a time.
```

For user stories, use:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z. Do not give me the solution immediately. Ask me questions that help me reason through the problem. Point me to the relevant files and patterns. Give hints in stages.
```

## How To Self-Study Over 30 Days

Use the 30-day plan in `technology-best-practices/04-practice-plan.md`. The arc is deliberate: first read entry points, then trace contracts, then follow state and APIs, then practice small changes, then write tests, then review diffs, then explain the system out loud.

For every checkpoint, write your answer first. Then read the "How to self-grade" section. If your answer only names files, keep going until you can explain responsibility, data shape, failure modes, and trade-offs.

