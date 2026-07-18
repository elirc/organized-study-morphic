# Mission Design Journal

## Design Rationale

The missions follow the way a real engineer grows in an unfamiliar full-stack codebase:

1. Find the heartbeat.
2. Build a folder map.
3. Learn contracts.
4. Trace UI.
5. Trace backend.
6. Trace state.
7. Trace data flow.
8. Practice review/debug/ownership.

The primary path is the chat feature because it touches React, Next.js routing, AI SDK streaming, auth, rate limits, search tools, Drizzle, PostgreSQL, and tests.

## Code Paths Chosen

- App entry: `app/layout.tsx:54-105`, `app/page.tsx:6-18`.
- Chat UI: `components/chat.tsx:38-586`, `components/chat-panel.tsx:30-420`.
- Routing: `app/search/page.tsx:11-33`, `app/search/[id]/page.tsx:30-59`.
- API: `app/api/chat/route.ts:20-229`, `app/api/upload/route.ts:15-115`, `app/api/feedback/route.ts:7-80`.
- Streaming: `lib/streaming/create-chat-stream-response.ts:33-235`.
- Agent/tools: `lib/agents/researcher.ts:67-163`, `lib/tools/search.ts:20-173`.
- Database: `lib/db/schema.ts:25-300`, `lib/db/actions.ts:23-193`.
- Tests: `lib/actions/__tests__/chat.test.ts:32-240`, `lib/db/__tests__/with-rls.test.ts:29-179`, `components/__tests__/render-message.test.tsx:37-82`.

## Tier Skills

- Junior missions build orientation and contract awareness.
- Mid-level missions build tracing, state ownership, side effect, and diff-review skills.
- Senior missions build risk prediction, performance/security review, missing-test design, and architectural narrative.

## Adapted Gaps

No Express, NestJS, Prisma, TypeORM, Redux, Zustand, React Query, SWR, or GraphQL patterns were found. Missions use Next.js API routes, server actions, React context, cookies, AI SDK hooks, Drizzle, and Vitest instead.

## Senior Engineer Difference

A senior does not only ask "where is the code?" They ask "what boundary is being crossed, who owns the data, what can fail, what should be tested, and what operational cost changes?"

