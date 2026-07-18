# Validation, Auth, and Permissions

**Authentication** = who are you (Supabase session → userId). **Authorization** = what may you do (ownership checks + RLS). Keep the words separate; interviewers listen for it.

## The identity pipeline

1. Middleware refreshes the session cookie on every request ([proxy.ts#L24-L26](../../proxy.ts#L24-L26) → [supabase/middleware.ts#L39-L41](../../lib/supabase/middleware.ts#L39-L41)) and page-gates non-public paths ([L44-L60](../../lib/supabase/middleware.ts#L44-L60)). Note `/api` is public there — **API routes authenticate themselves**; middleware redirects are UX, not security.
2. Server code resolves identity per request: [getCurrentUserId](../../lib/auth/get-current-user.ts#L18-L44). Three outcomes: Supabase user id / `undefined` (guest) / the shared anonymous id when `ENABLE_AUTH=false` — with a hard guard that cloud deployments can't disable auth ([L25-L29](../../lib/auth/get-current-user.ts#L25-L29)) and a warning that always prints ([L31-L38](../../lib/auth/get-current-user.ts#L31-L38)).

## The validation map — every layer, what it catches

| Layer | Example | Catches | Misses |
| --- | --- | --- | --- |
| Client UX | file type/count filter ([use-file-dropzone.ts#L40-L55](../../hooks/use-file-dropzone.ts#L40-L55)); empty-input guard ([chat.tsx#L426](../../components/chat.tsx#L426)) | honest mistakes | everything hostile |
| Route presence checks | trigger/message/messageId ([route.ts#L38-L52](../../app/api/chat/route.ts#L38-L52)); score ∈ {1,-1} ([feedback/route.ts#L19-L24](../../app/api/feedback/route.ts#L19-L24)) | malformed requests | deep shape of `message.parts` (no Zod on the chat body — gap) |
| Server re-validation | upload size/MIME re-checked ([upload/route.ts#L48-L60](../../app/api/upload/route.ts#L48-L60)) | client bypass | MIME spoofing (`file.type` is client-asserted) |
| Tool-input Zod | search/todo schemas ([lib/schema/search.tsx](../../lib/schema/search.tsx), [todo.ts#L5-L27](../../lib/tools/todo.ts#L5-L27)) | malformed LLM output | semantic nonsense (a valid-shape query can still be garbage) |
| Spec catalog Zod | component whitelist ([catalog.ts](../../lib/render/catalog.ts)) | LLM injecting arbitrary UI | catalog-legal-but-weird UI |
| DB constraints | CHECKs per part type ([schema.ts#L207-L224](../../lib/db/schema.ts#L207-L224)); FKs; length limits | bugs in every layer above | business rules SQL can't see |
| RLS | policies ([schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67) etc.) | cross-tenant access even from buggy code | nothing *within* a tenant |

Transferable principle: validation layers are not redundant — each has a different **blast radius** when it's the last one standing.

## Authorization, three enforcement points (defense in depth)

1. **Explicit ownership checks in code**: `initialChat.userId !== userId → 403` ([create-chat-stream-response.ts#L64-L70](../../lib/streaming/create-chat-stream-response.ts#L64-L70)); delete/edit paths re-verify ([lib/actions/chat.ts#L296-L301](../../lib/actions/chat.ts#L296-L301), [db/actions.ts#L328-L334](../../lib/db/actions.ts#L328-L334)).
2. **RLS in Postgres**: even if (1) is forgotten, the row simply doesn't come back / the write throws — translated to `RLSViolationError` ([with-rls.ts#L55-L69](../../lib/db/with-rls.ts#L55-L69)).
3. **Visibility rules for sharing**: public chats readable by anyone ([schema.ts#L61-L66](../../lib/db/schema.ts#L61-L66)); the app mirrors it ([db/actions.ts#L69-L78](../../lib/db/actions.ts#L69-L78)); private chat + no user → login redirect ([search/[id]/page.tsx#L42-L44](../../app/search/[id]/page.tsx#L42-L44)).

**IDOR** (insecure direct object reference — guessing someone else's `chatId`): blocked by (1)+(2). Trace it: attacker POSTs `/api/chat` with a victim chatId and `isNewChat:false` → `loadChat(chatId, attackerId)` runs under the attacker's RLS context → the victim's private chat is invisible → `initialChat` null → treated as new/creatable, but writes insert with *attacker's* userId; the victim's data never crosses. Do this trace yourself once — it's the single best authz exercise in the repo.

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| `getChat` with no userId returns public chats only — looks safe | every `withOptionalRLS(null, …)` call site: is null *supposed* to mean public here? ([with-rls.ts#L80-L90](../../lib/db/with-rls.ts#L80-L90)) |
| feedback endpoint "works" | it accepts any `traceId`/`messageId` **without authenticating the caller**, and when Supabase is unconfigured the metadata update runs with no RLS ([feedback/route.ts#L48-L68](../../app/api/feedback/route.ts#L48-L68) → [actions/feedback.ts#L19](../../lib/actions/feedback.ts#L19)) — *possible risk*: feedback spoofing on others' messages; verify exploitability |
| `updateChatTitle` is internal | it's called with no userId from persist ([persist-stream-results.ts#L103](../../lib/streaming/helpers/persist-stream-results.ts#L103) → [db/actions.ts#L374-L388](../../lib/db/actions.ts#L374-L388)) — RLS bypassed; safe only because chatId comes from server context, an *implicit* invariant |
| the upload path has auth | `chatId` in the object key is unverified input ([upload/route.ts#L76-L78](../../app/api/upload/route.ts#L76-L78)); public-by-URL storage semantics |
| rate limits exist | they're fail-open, cloud-only, and keyed on a spoofable header for guests ([route.ts#L77-L82](../../app/api/chat/route.ts#L77-L82)) |
| the anonymous mode warning | anonymous mode means one shared user id — "auth disabled" is a *product* mode with real invariants, guarded against cloud ([get-current-user.ts#L25-L29](../../lib/auth/get-current-user.ts#L25-L29)) |

## Drills

1. **The IDOR trace** above, written out step by step with anchors. Self-grade — Strong: you covered both read (invisible) and write (attacker-owned insert) halves and named which layer saved each.
2. Enumerate all `withOptionalRLS` call sites (grep). For each, one line: why is null-userId acceptable here, or *investigate*. Compare with [09-reference/risk-register.md](../09-reference/risk-register.md).
3. Design the fix for the feedback gap: require auth? verify message ownership via RLS-scoped read first? What breaks for anonymous deployments? Write the 5-line design note.

## Interview angle

- "Authn vs authz?" — definitions + *where each lives in a real app* (middleware/route vs code+RLS). ([Q29](../08-interview-prep/03-api-and-data-modeling-questions.md))
- "How do you prevent users reading each other's data?" — the three enforcement points; RLS mechanics with the `set_config` detail. (Q30)
- "Walk me through finding an IDOR." — the trace in this file, verbatim. (Q32)
- "Where do you validate input?" — the seven-layer map; strongest close: "each layer has a different blast radius when it's the last one standing."
