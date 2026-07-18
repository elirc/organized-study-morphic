# Key Flows

Seven end-to-end traces. These are the spine of the curriculum — later modules rotate through them rather than reusing one favorite. Format: why it matters → files → trace table → validation/authz → persistence/side effects → tests → what juniors miss / seniors notice → interview angle → drill.

---

## Flow 1: Authenticated chat submission (UI → API → agent → stream → persist)

Why this flow matters: it's the product. Everything else decorates it. It also demonstrates streaming, optimistic writes, and parallel work — three mid-level interview staples.

Open these files first:
- [components/chat.tsx#L89-L130](../../components/chat.tsx#L89-L130) — how the request body is shaped
- [app/api/chat/route.ts#L20-L169](../../app/api/chat/route.ts#L20-L169) — the gate sequence
- [lib/streaming/create-chat-stream-response.ts](../../lib/streaming/create-chat-stream-response.ts) — orchestration
- [lib/streaming/helpers/prepare-messages.ts#L91-L116](../../lib/streaming/helpers/prepare-messages.ts#L91-L116) — optimistic new-chat write

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Client | [chat.tsx#L421-L452](../../components/chat.tsx#L421-L452) | `onSubmit` builds parts (text + uploaded files), calls `sendMessage`, pushes `/search/{chatId}` into history | `{role:'user', parts:[…]}` | chatId generated client-side ([L58](../../components/chat.tsx#L58)) |
| 2 | Client | [chat.tsx#L102-L129](../../components/chat.tsx#L102-L129) | Transport shapes body; computes `isNewChat` from local message count | `{trigger, chatId, message, isNewChat}` | client-computed flag drives server writes |
| 3 | API | [route.ts#L30-L52](../../app/api/chat/route.ts#L30-L52) | Parse + trigger validation (400s) | body JSON | no schema validation — hand-rolled checks |
| 4 | API | [route.ts#L58](../../app/api/chat/route.ts#L58), [L68-L85](../../app/api/chat/route.ts#L68-L85) | Auth; guest gate + IP limit | `userId: string \| undefined` | — |
| 5 | API | [route.ts#L87-L103](../../app/api/chat/route.ts#L87-L103) | Search mode from cookie; `selectModel` | `Model` | cookie is user-controlled input |
| 6 | API | [route.ts#L136-L144](../../app/api/chat/route.ts#L136-L144) | Daily overall (100) + adaptive (30) limits | 429 JSON on block | fail-open on Redis errors |
| 7 | Stream | [create-chat-stream-response.ts#L56-L70](../../lib/streaming/create-chat-stream-response.ts#L56-L70) | For existing chats: load + **ownership check** (403) | `Chat & {messages}` | skipped when `isNewChat` |
| 8 | Stream | [prepare-messages.ts#L91-L116](../../lib/streaming/helpers/prepare-messages.ts#L91-L116) | New chat: fire-and-hold `createChatWithFirstMessage` promise; return `[message]` immediately | `UIMessage[]` | write may still be in flight when stream ends |
| 9 | Stream | [create-chat-stream-response.ts#L129-L159](../../lib/streaming/create-chat-stream-response.ts#L129-L159) | Strip spec/reasoning parts, convert, prune, truncate to context window | `ModelMessage[]` | token estimates approximate |
| 10 | Stream | [L161-L173](../../lib/streaming/create-chat-stream-response.ts#L161-L173) | Title generation kicked off **in parallel** | `Promise<string>` | errors swallowed → 'Untitled' |
| 11 | Agent | [researcher.ts#L124-L146](../../lib/agents/researcher.ts#L124-L146) | `ToolLoopAgent.stream` with `stopWhen: stepCountIs(maxSteps)` | SSE UI-message stream | model may loop tools up to 20/50 steps |
| 12 | Stream | [L186-L218](../../lib/streaming/create-chat-stream-response.ts#L186-L218) | Response returned; `onFinish` → `persistStreamResults` | response `UIMessage` | persistence happens **after** user got their answer |
| 13 | Persist | [persist-stream-results.ts#L35-L108](../../lib/streaming/helpers/persist-stream-results.ts#L35-L108) | Await pending insert (fallback + duplicate-key tolerated), save assistant message w/ retry, update title | rows in `chats/messages/parts` | failure here = answer shown but not saved |
| 14 | API | [route.ts#L208-L212](../../app/api/chat/route.ts#L208-L212) | `revalidateTag('chat-{id}')` | — | fires when response object is created, not when stream finishes |

Validation and authorization: trigger/message presence ([route.ts#L38-L52](../../app/api/chat/route.ts#L38-L52)); share-page referer block ([L61-L66](../../app/api/chat/route.ts#L61-L66)); ownership at [create-chat-stream-response.ts#L64-L70](../../lib/streaming/create-chat-stream-response.ts#L64-L70); RLS underneath everything ([schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67)).

Persistence and side effects: chats/messages/parts inserts via [db/actions.ts#L394-L441](../../lib/db/actions.ts#L394-L441); Langfuse trace ([create-chat-stream-response.ts#L79-L94](../../lib/streaming/create-chat-stream-response.ts#L79-L94)); analytics fire-and-forget IIFE ([route.ts#L173-L206](../../app/api/chat/route.ts#L173-L206)).

Tests that cover it: [lib/streaming/helpers/__tests__/prepare-messages.test.ts](../../lib/streaming/helpers/__tests__/prepare-messages.test.ts), [lib/streaming/__tests__/prune-messages-integration.test.ts](../../lib/streaming/__tests__/prune-messages-integration.test.ts). No test covers the route handler itself — evidence: no `__tests__` dir under `app/api/chat/`.

What juniors usually miss: the answer streams to the user *before* it's saved; steps 13–14 are where consistency lives, and they're off the request's critical path on purpose (*latency* beats durability here — a product decision).

What seniors notice: `isNewChat` is client-asserted (step 2). A malicious client could send `isNewChat: true` for an existing chatId — trace what happens: `createChatWithFirstMessageTransaction` inserts a chat with the caller's own `userId` under RLS, so a duplicate-key error fires on the existing id; can it ever *steal* a chat? RLS `withCheck` says no. That chain of reasoning — hostile input → follow the write path → find the invariant that saves you — is the senior skill.

Interview angle: "Design a streaming chat endpoint" / "How do you persist data you're streaming to the user?" → [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md), and Q13/Q31 in the question banks.

Drill: close the code. Write the 14 steps from memory with at least 8 file names. Self-grade — Basic: 8 steps, right order. Solid: gates in exact order + where authz happens twice. Strong: also explain why persistence is in `onFinish` and what's lost if the process dies mid-stream.

---

## Flow 2: Message persistence — UIMessage → rows and back

Why this flow matters: it's a nontrivial data-modeling decision (one wide `parts` table with per-tool columns) that you can critique in interviews.

Open these files first:
- [lib/db/schema.ts#L123-L256](../../lib/db/schema.ts#L123-L256) — the `parts` table
- [lib/utils/message-mapping.ts#L127-L339](../../lib/utils/message-mapping.ts#L127-L339) — UI→DB
- [lib/db/actions.ts#L86-L125](../../lib/db/actions.ts#L86-L125) — upsert

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Stream | [persist-stream-results.ts#L84](../../lib/streaming/helpers/persist-stream-results.ts#L84) | `upsertMessage(chatId, responseMessage, userId)` | `UIMessage` with parts | — |
| 2 | Action | [lib/actions/chat.ts#L179-L200](../../lib/actions/chat.ts#L179-L200) | Ensures id; **no authz check by design** (documented caller contract, [L171-L178](../../lib/actions/chat.ts#L171-L178)) | — | contract enforced by comment only |
| 3 | DB | [db/actions.ts#L98-L107](../../lib/db/actions.ts#L98-L107) | Insert message, `onConflictDoUpdate` sets **only `role`** | messages row | metadata changes on conflict are dropped — *confirmed by reading; investigate impact* |
| 4 | DB | [db/actions.ts#L110-L118](../../lib/db/actions.ts#L110-L118) | Delete all existing parts, insert mapped parts — inside one transaction | parts rows | delete+insert = simple, loses part ids |
| 5 | Mapper | [message-mapping.ts#L131-L339](../../lib/utils/message-mapping.ts#L131-L339) | Each part type → sparse columns (`tool_search_input`, `text_text`, …) | `DBMessagePart` | unknown types fall through to `data_*` columns ([L326-L331](../../lib/utils/message-mapping.ts#L326-L331)) |
| 6 | Read path | [db/actions.ts#L155-L193](../../lib/db/actions.ts#L155-L193) | chat + messages + ordered parts in parallel queries | `Chat & {messages: UIMessage[]}` | — |
| 7 | Mapper | [message-mapping.ts#L344-L666](../../lib/utils/message-mapping.ts#L344-L666) | Columns → typed UI parts; throws on unknown tool_state | `UIMessage` | a bad row can make a whole chat unloadable |

Validation and authorization: DB check constraints guarantee shape (e.g. text parts must have text, [schema.ts#L207-L224](../../lib/db/schema.ts#L207-L224)); RLS policies on all three tables gate row access via the chat's `user_id` ([schema.ts#L226-L254](../../lib/db/schema.ts#L226-L254)); `withRLS` sets the transaction-local user ([with-rls.ts#L44-L54](../../lib/db/with-rls.ts#L44-L54)).

Persistence and side effects: cache invalidation after every write via `revalidateTag('chat-{id}')` ([lib/actions/chat.ts#L196-L199](../../lib/actions/chat.ts#L196-L199)).

Tests that cover it: [with-rls.test.ts](../../lib/db/__tests__/with-rls.test.ts) (unit, mocked), [rls-policies.integration.test.ts](../../lib/db/__tests__/rls-policies.integration.test.ts) (needs real Postgres). The mapping layer itself has **no direct test file** — evidence: no `message-mapping` under `lib/utils/__tests__/`.

What juniors usually miss: the "upsert" for parts is *delete-then-insert* — safe only because it's transactional; copy the pattern outside a transaction and readers can observe a partless message.

What seniors notice: the wide-table design trades normalization for one-query reads and DB-enforceable checks; the cost is [message-mapping.ts](../../lib/utils/message-mapping.ts) — 775 lines of near-duplicated switches, and a schema migration every time a tool is added. Alternative: single `jsonb content` column + Zod at the boundary (fewer migrations, weaker DB constraints). Being able to argue both sides is exactly a mid-level→senior data-modeling interview answer.

Interview angle: "How would you store chat messages with heterogeneous content?" — Q23/Q24 in [03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: add a hypothetical `tool-weather` part on paper: list every file/column you'd touch. Self-grade — Solid: schema column pair + both mapping switches + type. Strong: you also flag the migration + backward-compat need for already-persisted rows, and propose the jsonb alternative.

---

## Flow 3: Guest chat + rate limiting (auth/security boundary)

Why this flow matters: it's the repo's clearest authorization boundary and its most interview-ready "how do you rate limit?" material.

Open these files first:
- [route.ts#L68-L85](../../app/api/chat/route.ts#L68-L85), [L115-L144](../../app/api/chat/route.ts#L115-L144)
- [lib/rate-limit/guest-limit.ts](../../lib/rate-limit/guest-limit.ts)
- [create-ephemeral-chat-stream-response.ts](../../lib/streaming/create-ephemeral-chat-stream-response.ts)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API | [route.ts#L68-L75](../../app/api/chat/route.ts#L68-L75) | No userId + `ENABLE_GUEST_CHAT!=='true'` → 401 | — | — |
| 2 | API | [route.ts#L77-L85](../../app/api/chat/route.ts#L77-L85) | IP from `x-forwarded-for` (first hop) → guest limit | ip string | header is client-influenceable unless a trusted proxy strips it |
| 3 | Limit | [guest-limit.ts#L51-L63](../../lib/rate-limit/guest-limit.ts#L51-L63) | `INCR rl:guest:chat:{ip}:{date}`; `EXPIRE` to midnight UTC on first hit | count | INCR then EXPIRE = two ops; if EXPIRE fails, key lives forever (*investigate*) |
| 4 | Limit | [guest-limit.ts#L34-L42](../../lib/rate-limit/guest-limit.ts#L34-L42), [L75-L78](../../lib/rate-limit/guest-limit.ts#L75-L78) | Not cloud / no Upstash / Redis error → **allowed** (fail-open) | — | availability chosen over enforcement |
| 5 | API | [route.ts#L115-L134](../../app/api/chat/route.ts#L115-L134) | Guests requesting adaptive mode on cloud → 401 with `authRequired` JSON (nudge, not silent downgrade) | JSON error | — |
| 6 | Stream | [create-ephemeral-chat-stream-response.ts#L32-L97](../../lib/streaming/create-ephemeral-chat-stream-response.ts#L32-L97) | Client-supplied **full history** → same prune/truncate → agent stream; nothing persisted | `UIMessage[]` from client | trust boundary: guests control their entire "history" |
| 7 | Client | [chat.tsx#L115](../../components/chat.tsx#L115) | `...(isGuest ? { messages } : {})` — only guests send full history | — | authenticated path deliberately re-reads history server-side |

Validation and authorization: the interesting part is what's *absent* — guest messages are not persisted, so RLS never engages; the security model is "ephemeral + rate-limited".

Persistence and side effects: Redis key writes only. 401 response bodies include `X-RateLimit-*` headers ([guest-limit.ts#L96-L104](../../lib/rate-limit/guest-limit.ts#L96-L104)).

Tests that cover it: [lib/rate-limit/__tests__/guest-limit.test.ts](../../lib/rate-limit/__tests__/guest-limit.test.ts), [adaptive-limit.test.ts](../../lib/rate-limit/__tests__/adaptive-limit.test.ts).

What juniors usually miss: fail-open is a *decision*, not a bug — Morphic prefers serving users during a Redis outage over strict enforcement. Say the word "fail-open" in a review and justify it.

What seniors notice: three rate limiters ([guest](../../lib/rate-limit/guest-limit.ts), [overall](../../lib/rate-limit/chat-limits.ts), [adaptive](../../lib/rate-limit/adaptive-limit.ts)) share ~80% code with drifted details (adaptive tracks analytics, returns 429; guest returns 401). That's a refactor kata ([06-contribution-practice/04](../06-contribution-practice/04-refactor-and-design-katas.md)) and a reliability question: INCR/EXPIRE non-atomicity → fix with a Lua script or `SET NX EX`+`INCR` pattern.

Interview angle: "Design rate limiting" is a top-5 system design warm-up — Q35 in [03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) walks the fixed-window-vs-sliding-window tradeoff using this exact code.

Drill: what happens if two requests from the same new IP race step 3? Walk both interleavings. Self-grade — Strong: you notice both INCR, one sets EXPIRE (count==1 exactly once), so the race is benign *except* when the EXPIRE call itself fails.

---

## Flow 4: Search tool → provider → citations (server/tool flow)

Why this flow matters: plugin architecture + prompt/UI contract in one trace.

Open these files first:
- [lib/tools/search.ts#L20-L167](../../lib/tools/search.ts#L20-L167)
- [providers/index.ts#L16-L37](../../lib/tools/search/providers/index.ts#L16-L37), [tavily.ts](../../lib/tools/search/providers/tavily.ts)
- [lib/utils/citation.ts](../../lib/utils/citation.ts)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Model | (LLM output) | Emits tool call `search({query, type, …})` per schema | Zod-validated args | schema differs per model — strict for OpenAI o-models ([lib/schema/search.tsx#L81-L94](../../lib/schema/search.tsx#L81-L94)) |
| 2 | Tool | [search.ts#L37-L40](../../lib/tools/search.ts#L37-L40) | `yield {state:'searching', query}` → UI shows spinner immediately | streamed partial | — |
| 3 | Tool | [search.ts#L54-L72](../../lib/tools/search.ts#L54-L72) | Pick provider: `type==='general'` → dedicated general provider else `SEARCH_API` env | provider type | env read at call time |
| 4 | Provider | [tavily.ts#L19-L48](../../lib/tools/search/providers/tavily.ts#L19-L48) | API-key check, ≥5-char query pad, cloud domain exclusions, POST | vendor JSON | key validated only at execution |
| 5 | Provider | [tavily.ts#L62-L102](../../lib/tools/search/providers/tavily.ts#L62-L102) | Match images to results by title; sanitize URLs | `SearchResults` | — |
| 6 | Tool | [search.ts#L145-L156](../../lib/tools/search.ts#L145-L156) | Build `citationMap {1: item…}`; attach `toolCallId` | augmented results | 1-based keys — contract with prompt |
| 7 | UI | [citation.ts#L21-L45](../../lib/utils/citation.ts#L21-L45), [L72-L110](../../lib/utils/citation.ts#L72-L110) | Extract maps per toolCallId; regex-replace `[n](#id)` → `[domain](url)`; invalid citations become **empty string** | markdown | silent drop hides model citation errors |

Validation and authorization: tool input via Zod ([lib/schema/search.tsx#L6-L46](../../lib/schema/search.tsx#L6-L46)); no user-level authz — tools run with server privileges (important: whatever the model asks to search, the server searches).

Persistence and side effects: outbound HTTP to vendor; results persisted later as `tool_search_output` json ([schema.ts#L173-L174](../../lib/db/schema.ts#L173-L174)). SearXNG advanced mode makes a **self-call** to `/api/advanced-search` using the middleware-derived base URL ([search.ts#L88-L101](../../lib/tools/search.ts#L88-L101)) with Redis caching ([advanced-search/route.ts#L27-L113](../../app/api/advanced-search/route.ts#L27-L113)).

Tests that cover it: [lib/utils/__tests__/citation.test.ts](../../lib/utils/__tests__/citation.test.ts). The tool and providers have no direct unit tests (evidence: no `__tests__` under `lib/tools/`).

What juniors usually miss: `async *execute` is a generator — the UI gets `searching` before results exist. That's why search feels instant.

What seniors notice: the citation pipeline is a three-party contract (prompt text ↔ tool output ↔ UI regex). Nothing type-checks it; only the prompt's ALL-CAPS pleading holds it together. A senior asks: "what validates that the model's citation ids exist?" — answer: `processCitations` silently deletes bad ones ([citation.ts#L88-L100](../../lib/utils/citation.ts#L88-L100)).

Interview angle: "How do you build reliable LLM tool-calling?" — Q7/Q12 in [01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md); plugin/factory questions in Q26.

Drill: add a fake provider `dummy` on paper: which 3 places must change? Verify: type union + factory case ([providers/index.ts#L8-L37](../../lib/tools/search/providers/index.ts#L8-L37)) + class file extending [BaseSearchProvider](../../lib/tools/search/providers/base.ts). Strong answer also mentions `.env.local.example` and the `SEARCH_API` docs.

---

## Flow 5: Regenerate / edit message (cross-layer mutation)

Why this flow matters: destructive mutation driven by UI state — the hardest correctness surface in the repo, and a goldmine of edge cases.

Open these files first:
- [chat.tsx#L366-L419](../../components/chat.tsx#L366-L419) — edit & reload handlers
- [prepare-messages.ts#L25-L79](../../lib/streaming/helpers/prepare-messages.ts#L25-L79)
- [db/actions.ts#L240-L270](../../lib/db/actions.ts#L240-L270) — `deleteMessagesFromIndex`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Client | [chat.tsx#L366-L401](../../components/chat.tsx#L366-L401) | Edit: locally replace message parts, then `regenerate({messageId})` | local state mutation | UI state and DB now differ until server confirms |
| 2 | Client | [chat.tsx#L106-L121](../../components/chat.tsx#L106-L121) | Transport sends `trigger:'regenerate-message'`, includes the edited user message when applicable | body | — |
| 3 | Server | [prepare-messages.ts#L27-L34](../../lib/streaming/helpers/prepare-messages.ts#L27-L34) | Load chat (or reuse preloaded); throw if empty | chat + messages | — |
| 4 | Server | [prepare-messages.ts#L35-L55](../../lib/streaming/helpers/prepare-messages.ts#L35-L55) | Find target by id; **fallback**: last assistant/user message if id missing | index | fallback can regenerate the *wrong* message if ids desync |
| 5 | Server | [prepare-messages.ts#L57-L64](../../lib/streaming/helpers/prepare-messages.ts#L57-L64) | Assistant target: delete it + everything after; return in-memory slice (comment explains `unstable_cache` staleness, [L61-L63](../../lib/streaming/helpers/prepare-messages.ts#L61-L63)) | `UIMessage[]` | delete happens **before** the new answer exists |
| 6 | Server | [prepare-messages.ts#L66-L78](../../lib/streaming/helpers/prepare-messages.ts#L66-L78) | User-edit target: upsert edited message, delete followers | — | same |
| 7 | DB | [db/actions.ts#L245-L269](../../lib/db/actions.ts#L245-L269) | Load all chat messages ordered, slice from index, `IN`-delete; parts cascade | count | createdAt-ordered — equal timestamps could misorder (*investigate*) |

Validation and authorization: [lib/actions/chat.ts#L296-L301](../../lib/actions/chat.ts#L296-L301) verifies chat ownership before deletion; RLS backs it.

Persistence and side effects: destructive deletes; `revalidateTag` per mutation ([lib/actions/chat.ts#L308-L309](../../lib/actions/chat.ts#L308-L309)).

Tests that cover it: [prepare-messages.test.ts](../../lib/streaming/helpers/__tests__/prepare-messages.test.ts).

What juniors usually miss: if the regeneration LLM call *fails*, the old answer was already deleted — the user lost content. There's no compensation step (no undo, no soft delete).

What seniors notice: this is where you'd argue for delete-after-success or soft-deletion (tombstones), and weigh the cost (complexity, storage) — a compact *compensation/rollback* interview story.

Interview angle: "Tell me about a destructive operation and how you'd make it safe" — Q39, and debugging round #2 in [05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).

Drill: write the failure matrix — {delete succeeds, stream fails} × {assistant regen, user edit}. What does the user see after refresh in each cell? Verify against the code. Strong = you catch that the edited user message *was* upserted (step 6), so a failed stream keeps the edit but loses the answer.

---

## Flow 6: File upload → R2 (security boundary)

Why this flow matters: a compact, complete example of input hardening — the checklist interviewers want when they ask "how do you handle file uploads?"

Open these files first:
- [hooks/use-file-dropzone.ts#L35-L106](../../hooks/use-file-dropzone.ts#L35-L106)
- [app/api/upload/route.ts](../../app/api/upload/route.ts)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Client | [use-file-dropzone.ts#L40-L63](../../hooks/use-file-dropzone.ts#L40-L63) | Client-side type filter + max 3 files; optimistic `uploading` state | `UploadedFile[]` | client checks are UX only |
| 2 | Client | [use-file-dropzone.ts#L65-L79](../../hooks/use-file-dropzone.ts#L65-L79) | `POST /api/upload` multipart per file, parallel | FormData | — |
| 3 | Server | [upload/route.ts#L17-L20](../../app/api/upload/route.ts#L17-L20) | **Auth required** → 401 for guests | — | — |
| 4 | Server | [upload/route.ts#L22-L31](../../app/api/upload/route.ts#L22-L31) | Storage-config gate → 400 with setup message | — | feature = config presence |
| 5 | Server | [upload/route.ts#L44-L60](../../app/api/upload/route.ts#L44-L60) | Server re-validates: size ≤5MB, MIME ∈ {jpeg, png, pdf} | — | MIME comes from client `file.type` (*possible risk*: not content-sniffed) |
| 6 | Server | [upload/route.ts#L72-L78](../../app/api/upload/route.ts#L72-L78) | Filename sanitized `[^a-z0-9.\-_] → _`; key = `{userId}/chats/{chatId}/{ts}-{name}` | object key | userId prefix = ownership by pathing |
| 7 | Server | [upload/route.ts#L84-L94](../../app/api/upload/route.ts#L84-L94) | `PutObjectCommand` to R2/S3; public URL constructed | `{filename,url,mediaType}` | bucket objects are publicly readable by URL |
| 8 | Client | [chat.tsx#L433-L440](../../components/chat.tsx#L433-L440) | URL becomes a `file` part in the next message | message part | URL persisted into chat history |

Validation and authorization: dual validation (client for UX, server for safety) — the transferable rule: **the server never trusts step 1**.

Persistence and side effects: external object store write; no DB row for the upload itself (the message part is the record).

Tests that cover it: none found for the route (evidence: no `__tests__` under `app/api/upload/`).

What juniors usually miss: `chatId` in the key is client-supplied and unverified — any string lands in your own userId namespace. Harmless? Mostly, but a senior asks anyway.

What seniors notice: public-by-URL storage means "unlisted", not "private" — anyone with the URL reads the file. Signed URLs would be the upgrade; cost: URL expiry breaks old chat history. A perfect tradeoff story.

Interview angle: uploads + SSRF + validation are Q33/Q34 in [03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md); checklist in [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).

Drill: list the 5 server-side checks in order from memory; then name the one check the server *cannot* do from `file.type` alone and how you'd fix it (magic-byte sniffing).

---

## Flow 7: Generative UI spec blocks (streaming client rendering)

Why this flow matters: the repo's most original engineering; also your best "tell me about complex frontend streaming" story.

Open these files first:
- [lib/render/catalog.ts](../../lib/render/catalog.ts) — the component whitelist
- [lib/render/spec-fence.ts#L21-L55](../../lib/render/spec-fence.ts#L21-L55)
- [lib/render/parse-spec-block.ts#L53-L92](../../lib/render/parse-spec-block.ts#L53-L92)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Prompt | [render/prompt.ts](../../lib/render/prompt.ts) via [search-mode-prompts.ts#L145-L147](../../lib/agents/prompts/search-mode-prompts.ts#L145-L147) | Model is told to emit ```spec fences for images/related questions | prompt text | model may emit garbage |
| 2 | Model | (stream) | Emits partial JSON spec, token by token | growing string | mid-stream = always invalid JSON |
| 3 | UI | [spec-fence.ts#L25-L50](../../lib/render/spec-fence.ts#L25-L50) | Evaluator: parse attempt per chunk; failure → `pending` (render nothing, never crash) | `SpecFenceResult` | — |
| 4 | Parser | [parse-spec-block.ts#L72-L79](../../lib/render/parse-spec-block.ts#L72-L79) | **Prefix-diff incremental compile**: if new source startsWith last, push only the delta; else reset compiler | `Spec` | O(delta) per chunk instead of O(n²) |
| 5 | Parser | [parse-spec-block.ts#L19-L37](../../lib/render/parse-spec-block.ts#L19-L37) | Prune children referencing not-yet-streamed elements | pruned Spec | dangling refs during stream |
| 6 | Migration | [parse-spec-block.ts#L94-L99](../../lib/render/parse-spec-block.ts#L94-L99) | `migrateSpec` upgrades legacy persisted specs before validating | Spec | old chat history must keep rendering — schema-versioning discipline |
| 7 | Catalog | [catalog.ts#L7-L70](../../lib/render/catalog.ts#L7-L70) | Validate against whitelist: Heading/Stack/Button/Grid/Image + `submitQuery` action, all Zod-typed | validated Spec | anything off-catalog throws ([parse-spec-block.ts#L99-L106](../../lib/render/parse-spec-block.ts#L99-L106)) |
| 8 | Server | [strip-spec-from-messages.ts](../../lib/streaming/helpers/strip-spec-from-messages.ts) (called at [create-chat-stream-response.ts#L133](../../lib/streaming/create-chat-stream-response.ts#L133)) | Specs are stripped before history goes back to the model | — | prevents prompt-history bloat/echo |

Validation and authorization: this is **XSS defense by construction** — the model can only instantiate whitelisted components with Zod-validated props; it never writes HTML. Compare with `dangerouslySetInnerHTML` approaches.

Persistence and side effects: spec text persists inside message text parts; copy-to-clipboard strips them ([chat.tsx#L287](../../components/chat.tsx#L287)).

Tests that cover it: the best-tested area — [parse-spec-block.test.ts](../../lib/render/__tests__/parse-spec-block.test.ts), [spec-fence.test.ts](../../lib/render/__tests__/spec-fence.test.ts), [migrations.test.ts](../../lib/render/__tests__/migrations.test.ts), [strip-spec-blocks.test.ts](../../lib/render/__tests__/strip-spec-blocks.test.ts), plus an e2e ([llm-image-output.e2e.test.ts](../../lib/render/__tests__/llm-image-output.e2e.test.ts)).

What juniors usually miss: why parsing can't wait for the fence to close — users would stare at raw JSON for seconds. Incremental parsing is a UX requirement, not cleverness.

What seniors notice: the catalog is a *capability boundary* for the model. Adding a component to the catalog = granting the model a new capability; review accordingly (what can `Image src` point at? any URL — tracked in the risk register).

Interview angle: "How would you let an LLM render UI safely?" — Q19 in [02-frontend-framework-questions.md](../08-interview-prep/02-frontend-framework-questions.md); also a system-design variation prompt.

Drill: simulate the parser by hand on the stream `{"root":"a","elements":{"a":{"type":"Stack","children":["b"]}` arriving in 3 chunks — what renders after each chunk? Strong = you apply pruning correctly (child `b` dropped until defined).
