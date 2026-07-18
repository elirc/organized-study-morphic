# Trace Tables

Fill the table yourself first (cover the "What happens" column), then compare. Columns: step, file/line, value shape, owner, transformation, risk. Six tables spanning UI→API, persistence, auth, error, cache, and streaming.

---

## Table 1 (UI → API → model): a submitted question becomes model messages

| # | file:line | value shape | owner | transformation | risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [chat.tsx#L442](../../components/chat.tsx#L442) | `{role:'user',parts}` | client | user text + files → UIMessage | empty submit guarded at L426 |
| 2 | [chat.tsx#L110-L127](../../components/chat.tsx#L110-L127) | request body | client transport | adds trigger/chatId/isNewChat | client asserts isNewChat |
| 3 | [route.ts#L30-L31](../../app/api/chat/route.ts#L30-L31) | parsed body | route | JSON.parse | no schema validation |
| 4 | [prepare-messages.ts#L86-L133](../../lib/streaming/helpers/prepare-messages.ts#L86-L133) | `UIMessage[]` | orchestration | append to history (or optimistic) | stale cache avoided via in-memory |
| 5 | [create-chat-stream-response.ts#L133-L139](../../lib/streaming/create-chat-stream-response.ts#L133-L139) | `UIMessage[]` | orchestration | strip spec, strip reasoning (OpenAI) | OpenAI-only branch |
| 6 | [L139](../../lib/streaming/create-chat-stream-response.ts#L139) | `ModelMessage[]` | AI SDK | `convertToModelMessages` | — |
| 7 | [L142-L152](../../lib/streaming/create-chat-stream-response.ts#L142-L152) | `ModelMessage[]` | orchestration | prune → maybe truncate | truncation can empty on huge last msg |
| 8 | [researcher.ts#L124-L146](../../lib/agents/researcher.ts#L124-L146) | agent stream | agent | model + tools + stopWhen | up to 20/50 tool steps |

Risk to name aloud: the shape narrows from rich UIMessage to lossy ModelMessage across steps 5–7; anything the model needs must survive stripping.

---

## Table 2 (persistence): assistant `UIMessage` → `parts` rows

| # | file:line | value shape | owner | transformation | risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [persist-stream-results.ts#L24-L30](../../lib/streaming/helpers/persist-stream-results.ts#L24-L30) | UIMessage + metadata | orchestration | attach trace/mode/model | — |
| 2 | [lib/actions/chat.ts#L179-L200](../../lib/actions/chat.ts#L179-L200) | UIMessage | action | ensure id, revalidate | **no authz (caller contract)** |
| 3 | [db/actions.ts#L98-L108](../../lib/db/actions.ts#L98-L108) | messages row | db | insert / onConflict set role | metadata dropped on conflict |
| 4 | [db/actions.ts#L110-L118](../../lib/db/actions.ts#L110-L118) | delete + insert parts | db | replace parts (txn) | delete-then-insert |
| 5 | [message-mapping.ts#L131-L339](../../lib/utils/message-mapping.ts#L131-L339) | `DBMessagePart[]` | mapper | part → sparse columns | stringly-typed column keys |
| 6 | [schema.ts#L207-L224](../../lib/db/schema.ts#L207-L224) | rows | Postgres | CHECK constraints enforce shape | insert throws on violation |

---

## Table 3 (auth/isolation): request → RLS-scoped read

| # | file:line | value shape | owner | transformation | risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [route.ts#L58](../../app/api/chat/route.ts#L58) | `userId?` | route | Supabase session or anon | undefined = guest |
| 2 | [create-chat-stream-response.ts#L61](../../lib/streaming/create-chat-stream-response.ts#L61) | Chat+messages | orchestration | `loadChat(chatId,userId)` | cached per (chat,user) |
| 3 | [lib/actions/chat.ts#L17-L34](../../lib/actions/chat.ts#L17-L34) | cached fn | action | unstable_cache keyed w/ userId | userId-in-key prevents poisoning |
| 4 | [db/actions.ts#L162-L192](../../lib/db/actions.ts#L162-L192) | rows | db | `withOptionalRLS` | null userId ⇒ no RLS |
| 5 | [with-rls.ts#L48-L50](../../lib/db/with-rls.ts#L48-L50) | — | db | `set_config(user_id, local)` | isolation established here |
| 6 | [schema.ts#L96-L100](../../lib/db/schema.ts#L96-L100) | filtered rows | Postgres | policy WHERE via chat join | index-backed |
| 7 | [create-chat-stream-response.ts#L64-L70](../../lib/streaming/create-chat-stream-response.ts#L64-L70) | 403 or continue | orchestration | app-level ownership recheck | defense in depth |

The point: isolation is enforced at step 5/6 (DB) *and* re-checked at step 7 (app). Two independent layers.

---

## Table 4 (error): a 429 becomes a modal

| # | file:line | value shape | owner | transformation | risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [chat-limits.ts#L90-L106](../../lib/rate-limit/chat-limits.ts#L90-L106) | 429 Response, JSON body | limiter | build error + headers | HTTP shape made in lib/ |
| 2 | (network) | Error at client | AI SDK transport | flattens to `error.message` string | structure lost |
| 3 | [chat.tsx#L138-L143](../../components/chat.tsx#L138-L143) | booleans | client | substring/regex sniff `429`/`rate limit` | brittle string matching |
| 4 | [chat.tsx#L161-L184](../../components/chat.tsx#L161-L184) | parsed JSON | client | regex-extract `{…}` from message | double-encoding smell |
| 5 | [chat.tsx#L179-L184](../../components/chat.tsx#L179-L184) | modal state | client | set ErrorModal | — |
| 6 | [error-modal.tsx](../../components/error-modal.tsx) | UI | client | render | — |

Risk to name: the error *structure* is destroyed at step 2 and reconstructed by regex at step 4 — the whole chain argues for a structured error channel.

---

## Table 5 (cache): a title update and its visibility

| # | file:line | value shape | owner | transformation | risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [persist-stream-results.ts#L100-L108](../../lib/streaming/helpers/persist-stream-results.ts#L100-L108) | title string | orchestration | `updateChatTitle(chatId,title,userId)` | — |
| 2 | [db/actions.ts#L374-L388](../../lib/db/actions.ts#L374-L388) | Chat row | db | update via `withOptionalRLS` | — |
| 3 | ??? | — | ??? | **is there a `revalidateTag` here?** | *investigate* — unlike other writes, this path may not invalidate |
| 4 | [lib/actions/chat.ts#L29-L30](../../lib/actions/chat.ts#L29-L30) | cached read | action | 60s TTL | title may lag ≤60s in sidebar |

This table is deliberately incomplete at step 3 — your job is to determine whether the title write invalidates its cache. (Compare with [shareChat](../../lib/actions/chat.ts#L265-L282) which *does* call revalidateTag.)

---

## Table 6 (streaming): spec fence tokens → rendered component

| # | file:line | value shape | owner | transformation | risk |
| --- | --- | --- | --- | --- | --- |
| 1 | (LLM) | partial ```spec text | model | emits token by token | invalid mid-stream |
| 2 | [spec-fence.ts#L25-L44](../../lib/render/spec-fence.ts#L25-L44) | `SpecFenceResult` | evaluator | parse attempt; fail→pending | never throws to UI |
| 3 | [parse-spec-block.ts#L72-L86](../../lib/render/parse-spec-block.ts#L72-L86) | `Spec\|null` | parser | prefix-diff incremental compile | reset on non-prefix change |
| 4 | [parse-spec-block.ts#L19-L37](../../lib/render/parse-spec-block.ts#L19-L37) | pruned Spec | parser | drop dangling child refs | — |
| 5 | [catalog.ts#L7-L70](../../lib/render/catalog.ts#L7-L70) | validated Spec | catalog | Zod validate against whitelist | off-catalog throws→pending |
| 6 | [registry.tsx](../../lib/render/registry.tsx) | React nodes | renderer | Spec → components | — |

Self-grade for all tables — Basic: fills value shapes correctly. Solid: names each owner and transformation. Strong: articulates the risk column and, for Tables 4/5, names the design smell / open question rather than a mechanical step.
