# Fast Track — One Weekend

Goal: by Sunday night you can run Morphic, trace two end-to-end flows without notes, make one safe change, and explain the chat flow aloud as if an interviewer asked.

## Saturday morning — run it

All commands are __inferred__ from [package.json](../package.json#L6-L18) and [AGENTS.md](../AGENTS.md) (not executed on the authoring machine — no Bun installed there).

**Option A: Docker (fastest, includes Postgres + Redis + SearXNG):**

```bash
cp .env.local.example .env.local   # set at least OPENAI_API_KEY
docker compose up -d               # inferred — starts postgres, redis, searxng, morphic
# visit http://localhost:3000
```

**Option B: local dev (needs your own Postgres + a search API key):**

```bash
bun install                        # inferred; engines pin bun 1.2.12 (package.json#L107-L109)
cp .env.local.example .env.local   # set OPENAI_API_KEY, TAVILY_API_KEY, DATABASE_URL
bun migrate                        # inferred — runs lib/db/migrate.ts
bun dev                            # inferred — Next.js dev server on :3000
```

**Sanity checks:**

```bash
bun typecheck                      # inferred — tsc --noEmit
bun run test                       # inferred — Vitest; note: NOT `bun test` (AGENTS.md#L17)
```

Run one test file to prove the harness works: `bun run test lib/utils/__tests__/citation.test.ts` (__inferred__).

## Saturday afternoon — the first 10 files, in order

Open these in order. Predict before you scroll (questions inline).

1. [package.json](../package.json) — scripts, deps. *Predict: which package is the AI engine?* (`ai` — the Vercel AI SDK.)
2. [app/api/chat/route.ts](../app/api/chat/route.ts) — the one endpoint that matters. Read all 229 lines. Note the order: parse → auth ([L58](../app/api/chat/route.ts#L58)) → guest gate ([L68-L85](../app/api/chat/route.ts#L68-L85)) → mode/model ([L87-L103](../app/api/chat/route.ts#L87-L103)) → rate limits ([L136-L144](../app/api/chat/route.ts#L136-L144)) → stream ([L151-L169](../app/api/chat/route.ts#L151-L169)).
3. [lib/streaming/create-chat-stream-response.ts](../lib/streaming/create-chat-stream-response.ts) — ownership check ([L64-L70](../lib/streaming/create-chat-stream-response.ts#L64-L70)), message prep, agent stream, `onFinish` persistence ([L196-L218](../lib/streaming/create-chat-stream-response.ts#L196-L218)).
4. [lib/agents/researcher.ts](../lib/agents/researcher.ts) — the agent. Quick vs adaptive mode switch at [L92-L113](../lib/agents/researcher.ts#L92-L113).
5. [lib/tools/search.ts](../lib/tools/search.ts) — a streaming tool (`async *execute`, [L24](../lib/tools/search.ts#L24)) and the provider switch ([L54-L72](../lib/tools/search.ts#L54-L72)).
6. [lib/db/schema.ts](../lib/db/schema.ts) — three tables: `chats`, `messages`, `parts`. Read the RLS policies ([L53-L67](../lib/db/schema.ts#L53-L67)).
7. [lib/db/with-rls.ts](../lib/db/with-rls.ts) — how a userId becomes a Postgres session setting ([L44-L54](../lib/db/with-rls.ts#L44-L54)).
8. [components/chat.tsx](../components/chat.tsx) — the client. `useChat` transport at [L89-L130](../components/chat.tsx#L89-L130); note how `isNewChat` is computed on the client ([L123-L126](../components/chat.tsx#L123-L126)).
9. [lib/auth/get-current-user.ts](../lib/auth/get-current-user.ts) — auth in 44 lines, including the `ENABLE_AUTH=false` anonymous mode ([L23-L40](../lib/auth/get-current-user.ts#L23-L40)).
10. [lib/agents/prompts/search-mode-prompts.ts](../lib/agents/prompts/search-mode-prompts.ts) — the system prompt is a product artifact. Read the citation contract ([L72-L101](../lib/agents/prompts/search-mode-prompts.ts#L72-L101)).

## Saturday evening — trace flow #1: "user asks a question"

With the app running, ask "why is the sky blue?" and watch the network tab. Then confirm each hop in code:

`Chat.onSubmit` ([components/chat.tsx#L421-L452](../components/chat.tsx#L421-L452)) → POST `/api/chat` → `getCurrentUserId` → `createChatStreamResponse` → `prepareMessages` ([lib/streaming/helpers/prepare-messages.ts#L91-L116](../lib/streaming/helpers/prepare-messages.ts#L91-L116) — note the chat row is created *in the background* while the LLM starts) → `researcher(...).stream(...)` → search tool yields `searching` then `complete` ([lib/tools/search.ts#L37-L164](../lib/tools/search.ts#L37-L164)) → SSE stream to browser → `onFinish` → `persistStreamResults` ([lib/streaming/helpers/persist-stream-results.ts#L11-L109](../lib/streaming/helpers/persist-stream-results.ts#L11-L109)).

Pause-and-predict: *what happens if the LLM finishes before the background chat-row insert does?* Answer at [persist-stream-results.ts#L35-L79](../lib/streaming/helpers/persist-stream-results.ts#L35-L79) — it awaits the pending insert, retries on failure, and treats a duplicate-key error as success (idempotency).

## Sunday morning — trace flow #2: citations

The model writes `[1](#toolCallId)`. Find where that becomes a real link: prompt contract ([search-mode-prompts.ts#L72-L101](../lib/agents/prompts/search-mode-prompts.ts#L72-L101)) → `citationMap` built in the tool ([search.ts#L145-L151](../lib/tools/search.ts#L145-L151)) → extracted per message ([lib/utils/citation.ts#L21-L45](../lib/utils/citation.ts#L21-L45)) → regex-replaced into `[domain](url)` ([citation.ts#L72-L110](../lib/utils/citation.ts#L72-L110)).

## Sunday afternoon — one small safe change + one test

Change: in [lib/agents/title-generator.ts#L26](../lib/agents/title-generator.ts#L26), the fallback title truncates at 75 chars. Make it configurable via a constant, or change the [DEFAULT_CHAT_TITLE](../lib/streaming/create-chat-stream-response.ts#L31) string, run the app, and confirm nothing breaks. Revert after. The point is the loop: edit → typecheck → test → run.

Then run the existing test suite once more and read one test file end-to-end: [lib/db/__tests__/with-rls.test.ts](../lib/db/__tests__/with-rls.test.ts) — see how `db.transaction` is mocked so the RLS wrapper is testable without Postgres.

## Sunday evening — teach-back (interview rehearsal)

Say aloud, in under 3 minutes, as if answering "walk me through a codebase you know well":

> "Morphic is an AI answer engine on Next.js. One POST endpoint handles chat: it authenticates via Supabase, applies Redis rate limits, picks a model, and streams a tool-using agent's output over SSE. The agent loops search/fetch tools; results carry a citation map the UI resolves into links. Persistence is Postgres via Drizzle with row-level security — every query runs in a transaction that sets the user ID as a session variable, so isolation is enforced in the database, not just the app. The interesting tradeoffs are X and Y…"

Fill X and Y yourself (candidates: optimistic chat creation vs consistency; fail-open rate limiting; guest mode sending full history from the client). If you can't fill them, reread [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).

## What this fast path skips

Generative UI spec blocks, the model-selection stack, file upload, feedback/observability, the regeneration/edit flow, all drills, and everything in modules 03–08. The map is [01-codebase-cartography/01-system-map.md](01-codebase-cartography/01-system-map.md) when you're ready.
