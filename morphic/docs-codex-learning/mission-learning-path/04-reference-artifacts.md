# Reference Artifacts

### Mission 25: Write the Docs That Don't Exist

**Tier:** Senior

**Time Estimate:** 90 minutes

**Goal:** Practice turning code understanding into permanent team knowledge.

**The Concept:** Documentation is cached engineering judgment.

**Design Intent Before You Read the Code:** Choose a feature path, cite real files, explain ownership and risks, and write for the next engineer under time pressure.

**Find It In The Code:** Use the chat path: `components/chat.tsx:421-452`, `app/api/chat/route.ts:20-229`, `lib/streaming/create-chat-stream-response.ts:33-235`, `lib/db/schema.ts:25-300`.

```md
<!-- Good documentation cites the boundary and the reason. -->
The chat route branches between guest and authenticated streams at app/api/chat/route.ts:151-169.
```

**The Aha Moment:** The best docs reduce future debugging time.

**Socratic Checkpoint:** 5 questions.

1. What exact behavior are you documenting?
2. Which files prove it?
3. What would a junior misunderstand?
4. What would a senior worry about?
5. What test or log would help verify it?

**How to Self-Grade:** Strong docs include path, data shape, reason, risks, and verification.

**Connects To:** All previous missions.

## Doc 1: Junior Onboarding Checklist

- Run `bun install`.
- Copy `.env.local.example` to `.env.local`.
- Fill AI provider, database, and search settings.
- Run `bun migrate`.
- Run `bun dev`.
- Read `app/page.tsx`, `components/chat.tsx`, `app/api/chat/route.ts`.
- First PR checklist: small scope, no unrelated refactor, tests where risk exists, `bun lint`, `bun typecheck`, `bun format:check`, `bun run test`.
- Questions to ask: What owns the state? What validates the input? What persists the result?

## Doc 2: Architecture Guide For New Engineers

Navigate by runtime boundary:

- Pages: `app/**/page.tsx`.
- HTTP routes: `app/api/**/route.ts`.
- Client UI: components with `'use client'`.
- Server/domain actions: `lib/actions`.
- DB: `lib/db`.
- AI tools and agents: `lib/agents`, `lib/tools`.

Avoid getting lost by tracing one user action at a time.

## Doc 3: Code Review Checklist

- Check public boundary validation first.
- Check auth and ownership second.
- Check state owner third.
- Check data persistence and cache invalidation fourth.
- Check tests last, after you know the risk.

Examples: `/api/chat` validation in `app/api/chat/route.ts:37-52`; cache invalidation in `app/api/chat/route.ts:208-212`.

## Doc 4: Debugging Playbook

1. Reproduce and capture status/message.
2. Identify route or component.
3. Trace request body or props.
4. Inspect auth mode.
5. Inspect provider/model/search config.
6. Inspect DB persistence if UI streamed but history is missing.
7. Write the smallest failing test once cause is known.

## Doc 5: Change Playbook

Branch -> read route/component -> write plan -> edit smallest surface -> add test -> run local checks -> review diff -> open PR -> address review -> merge.

For this repo, always consider `bun run test` rather than `bun test`, because the package script invokes Vitest (`package.json:15`).

## Doc 6: Senior Ownership Notes

Monitor:

- Streaming failures.
- Search provider latency/errors.
- DB/RLS errors.
- Advanced crawl fan-out.
- Auth mode configuration.
- Upload failures and object storage config.

Improve:

- Zod validation at API boundaries.
- Chat route tests.
- Upload ownership checks.
- Advanced search decomposition and safety.

Leave alone unless necessary:

- Existing provider abstraction.
- AI SDK stream path.
- Drizzle table names and migrations without a migration plan.

## Doc 7: Interview Walkthrough

Practice answer:

"I worked through a full-stack Next.js answer engine. The main flow starts in a server component, passes auth and model context to a client chat, sends structured AI SDK message parts to `/api/chat`, then streams through a researcher agent with search tools. Authenticated responses persist to PostgreSQL through Drizzle with message parts normalized for reconstruction. The trade-off is powerful rich-message persistence at the cost of complex mapping and dynamic types."

Follow-up trade-offs:

- Why not Redux? State is local, contextual, or cookie-backed.
- Why Drizzle? Type-safe SQL and schema co-location.
- Biggest risk? Public API validation and advanced crawler safety.
- Best improvement? Add route tests and validation schemas before broad feature work.

