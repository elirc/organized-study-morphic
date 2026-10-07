# Reference Suite

## Doc 1: Junior Onboarding Guide

Orient from outside in:

1. Read scripts in `package.json:6-18`.
2. Read env requirements in `docs/CONFIGURATION.md` (the upstream `.env.local.example` is not in this snapshot; `grep -rn "process.env" lib app` is the ground truth).
3. Read global shell in `app/layout.tsx:54-105`.
4. Read homepage route in `app/page.tsx:6-18`.
5. Read `components/chat.tsx:38-586` in chunks: state, `useChat`, effects, submit, render.

Setup:

```bash
bun install
cp .env.local.example .env.local
bun migrate
bun dev
bun run test
```

Assumptions: local PostgreSQL exists unless Docker Compose is used; no seed script was found.

## Doc 2: Mid-Level Architecture Guide

System design:

- Next.js App Router handles pages and API routes.
- React client components handle chat interaction.
- AI SDK handles transport and streaming.
- Server route enforces auth/rate/model/search boundaries.
- Researcher agent owns tool choice.
- Drizzle/PostgreSQL persists chats, messages, parts, feedback.

Primary contracts:

- Chat UI to API body: `components/chat.tsx:100-129`.
- API to stream config: `app/api/chat/route.ts:151-169`.
- Stream to DB persistence: `lib/streaming/create-chat-stream-response.ts:196-223`.
- DB message parts: `lib/db/schema.ts:122-258`.

## Doc 3: Senior Ownership Guide

Technical debt register:

- API body validation inconsistent.
- Advanced search route combines cache, crawl, parse, score, and HTTP in one file.
- Dynamic AI tool typing uses `any`.
- Upload chat ownership should be explicit.
- Main chat route needs focused tests.

Upgrade path:

1. Add validation schemas.
2. Add route tests.
3. Split advanced search into cache/client/scoring helpers.
4. Add concurrency and URL safety controls.
5. Tighten message/tool types where AI SDK allows.

## Doc 4: Code Review Guide

Review in this order:

1. Boundary: did request/props/data shape change?
2. Auth: does it preserve `getCurrentUserId`, guest behavior, and RLS assumptions?
3. State: does the state still have one clear owner?
4. Persistence: are cache tags revalidated after mutation?
5. Streaming: does error handling avoid breaking already-started streams?
6. Tests: do tests cover failure modes?

Examples:

- For chat API changes, inspect `app/api/chat/route.ts:37-52`, `app/api/chat/route.ts:68-144`, and `app/api/chat/route.ts:151-169`.
- For DB writes, inspect `lib/actions/chat.ts:167-200` and `lib/db/actions.ts:86-125`.
- For message rendering, inspect `components/__tests__/render-message.test.tsx:37-82`.

## Doc 5: Debugging Guide

Likely failure points:

- No model available: `app/api/chat/route.ts:96-113`.
- Guest blocked: `app/api/chat/route.ts:68-75`.
- Adaptive guest blocked in cloud: `app/api/chat/route.ts:115-134`.
- Search provider failure: `lib/tools/search.ts:84-142`.
- Persistence failure after stream: `lib/streaming/helpers/persist-stream-results.ts:81-97`.
- Upload storage missing: `app/api/upload/route.ts:22-31`.

Debugging path:

1. Reproduce in UI and note route/status.
2. Check whether the failing path is guest or authenticated.
3. Inspect request body from `components/chat.tsx:100-129`.
4. Check server route branch.
5. If streaming completes but history is missing, inspect `persistStreamResults`.
6. If DB access fails, inspect RLS helper tests for intended behavior.

## Doc 6: Change Playbook

To add a feature end-to-end:

1. Identify data owner: component state, cookie, API, or DB.
2. Add/adjust type first.
3. Update UI component.
4. Update API contract and validation.
5. Update server action or DB action.
6. Add cache invalidation if data is persisted.
7. Add tests around the narrowest risky boundary.
8. Run `bun lint`, `bun typecheck`, `bun format:check`, `bun run build`, `bun run test` before PR.

## Doc 7: Interview Walkthrough

Concise system explanation:

"Morphic is a Next.js full-stack AI answer engine. The homepage server component resolves user and model context, then renders a client chat. The chat component uses the AI SDK `useChat` hook with a custom transport body. `/api/chat` handles auth, guest mode, rate limits, search mode, and model selection. Authenticated requests stream through a researcher agent that can use search/fetch/todo tools, then saves message parts to PostgreSQL via Drizzle. Guest requests stream without persistence."

Deeper follow-ups:

- Trade-off: normalized message parts improve reconstruction and querying but make persistence mapping complex.
- Security: RLS is strong, but API validation and crawler/upload boundary checks should be tightened.
- Performance: streaming and pruning help responsiveness; advanced crawling needs concurrency controls.
- Testing: action and helper coverage exists, but main chat route tests should be expanded.

