# Language & Runtime Model

## The event loop, in one production sentence

JavaScript runs one call stack; anything slow must be I/O handed to the runtime, with continuations scheduled as micro/macrotasks. Server implication: **one stuck synchronous loop blocks every request on that instance.**

Where this repo respects it: the chat route returns a `Response` whose body is a stream — the request handler exits fast, and work continues as the stream pulls ([create-chat-stream-response.ts#L186](../../lib/streaming/create-chat-stream-response.ts#L186)). `result.consumeStream()` at [L184](../../lib/streaming/create-chat-stream-response.ts#L184) keeps the agent producing even if the client stops reading — so `onFinish` (persistence) still runs after an abort/close.

Where to watch it: [fetch.ts#L48-L62](../../lib/tools/fetch.ts#L48-L62) runs multiple regexes over up-to-full-page HTML strings *synchronously*. For a normal page it's fine; a pathological page makes this a CPU hog on the request path. That's the sentence above, live.

## Promises: parallel vs serial is a decision, not syntax

- **Parallel because independent:** five vendor model fetches in one `Promise.all` ([fetch-models.ts#L392-L398](../../lib/models/fetch-models.ts#L392-L398)); chat row + messages fetched together ([db/actions.ts#L162-L175](../../lib/db/actions.ts#L162-L175)); parallel file uploads ([use-file-dropzone.ts#L65-L103](../../hooks/use-file-dropzone.ts#L65-L103)).
- **Deliberately unawaited:** title generation starts early, is *held* as a promise, awaited only when needed ([create-chat-stream-response.ts#L161-L173](../../lib/streaming/create-chat-stream-response.ts#L161-L173) → [persist-stream-results.ts#L32-L33](../../lib/streaming/helpers/persist-stream-results.ts#L32-L33)). This is the "start now, need later" pattern — cheap concurrency without workers.
- **Fire-and-forget with containment:** analytics wrapped in an async IIFE with its own try/catch so it can never fail the request ([route.ts#L173-L206](../../app/api/chat/route.ts#L173-L206)). Note the `.catch` on the held title promise ([L169-L172](../../lib/streaming/create-chat-stream-response.ts#L169-L172)) — without it, a rejection before the await becomes an unhandled rejection.
- **Timeout by race:** `Promise.race([redis.incr(key), reject-after-3s])` ([adaptive-limit.ts#L81-L86](../../lib/rate-limit/adaptive-limit.ts#L81-L86)). Sharp edge: the loser isn't cancelled — the INCR still lands; only your *waiting* stops.
- **Serial where order matters:** `clearChats` deletes one chat at a time in a loop ([lib/actions/chat.ts#L229-L233](../../lib/actions/chat.ts#L229-L233)) — correct but N sequential round-trips; a `Promise.all` or one SQL delete is the obvious upgrade (ticket material).

Pitfall checklist: unhandled rejection on stored promises (attach `.catch` at creation); `race` doesn't cancel; `await` in a loop = serial by accident; `Promise.all` fails fast — one rejection loses the others' results (fine for model fetch since each fetcher catches internally, [fetch-models.ts#L186-L189](../../lib/models/fetch-models.ts#L186-L189)).

## Cancellation: AbortSignal end to end

The request's own signal ([route.ts#L22](../../app/api/chat/route.ts#L22)) is threaded into the agent stream ([create-chat-stream-response.ts#L179-L182](../../lib/streaming/create-chat-stream-response.ts#L179-L182)) and the title call ([title-generator.ts#L34](../../lib/agents/title-generator.ts#L34), abort handled gracefully at [L62-L69](../../lib/agents/title-generator.ts#L62-L69)). The fetch tool builds its own controller for a 10s timeout ([fetch.ts#L11-L23](../../lib/tools/fetch.ts#L11-L23)) — note it does *not* link to the request signal, so a user abort doesn't stop an in-flight page fetch (*investigate*: acceptable waste or worth linking with `AbortSignal.any`?).

## Generators

`async function*` powers streaming tools — yields become UI states ([search.ts#L24-L40](../../lib/tools/search.ts#L24-L40)). Consuming both shapes (iterable or promise) defensively: [search.ts#L203-L216](../../lib/tools/search.ts#L203-L216). Sharp edge: `for await` swallows nothing — a throw mid-iteration propagates and the earlier yields are already delivered; UI must tolerate "searching→error".

## Node vs Bun vs browser vs middleware

Bun runs scripts/deps (fast installs, [package.json#L107-L109](../../package.json#L107-L109)); the *server* code is still Node-API-compatible (`crypto.randomUUID` [create-chat-stream-response.ts#L7](../../lib/streaming/create-chat-stream-response.ts#L7), `Buffer` [upload/route.ts#L81](../../app/api/upload/route.ts#L81), `http` agents in [advanced-search/route.ts#L4-L7](../../app/api/advanced-search/route.ts#L4-L7)). Browser code lives under `'use client'` and uses `window`/`navigator` ([chat.tsx#L287](../../components/chat.tsx#L287), [L448-L450](../../components/chat.tsx#L448-L450)). Middleware ([proxy.ts](../../proxy.ts)) is the constrained runtime: no Node APIs, keep it header-level.

## Drills

1. In [persist-stream-results.ts](../../lib/streaming/helpers/persist-stream-results.ts), number every `await` and answer: which awaits could safely run in parallel? (Title + initial save are already parallel-started; message save must follow initial save — why? FK from messages.chatId, [schema.ts#L79-L81](../../lib/db/schema.ts#L79-L81).)
2. Predict the console order (fake, labeled):
   ```js
   // Illustrative fake code: not from this repo
   console.log(1); Promise.resolve().then(() => console.log(2));
   setTimeout(() => console.log(3)); console.log(4);
   ```
   (1,4,2,3 — microtasks before macrotasks.) Now find the real place ordering matters: `result.consumeStream()` is called *before* returning the response — what breaks if you move it after? Nothing visible locally, but abandoned clients would stall `onFinish`.
3. Self-grade — Basic: defines microtask. Solid: explains the held-promise title pattern. Strong: identifies the `race`-doesn't-cancel edge and the unlinked fetch-tool signal.

## Interview angle

- "Walk me through what happens when you `await`." → microtask continuation; cite the held title promise as a non-trivial usage. ([Q1](../08-interview-prep/01-js-ts-node-deep-dive.md))
- "How do you run things in parallel in Node?" → `Promise.all` + when *not* to (fail-fast semantics); cite [fetch-models.ts#L392-L398](../../lib/models/fetch-models.ts#L392-L398). (Q2)
- "How do you time out an operation?" → race vs AbortSignal, and that race leaks the loser; cite [adaptive-limit.ts#L81-L86](../../lib/rate-limit/adaptive-limit.ts#L81-L86). (Q3)
- "Generators — ever used one for real?" → streaming tool states; cite [search.ts#L24](../../lib/tools/search.ts#L24). (Q4)
