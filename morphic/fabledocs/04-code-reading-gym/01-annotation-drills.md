# Annotation Drills

For each excerpt, write five headers before peeking at the notes: **Inputs** (and who controls them), **Outputs**, **Dependencies** (imports/env/IO), **Invariants** (what must hold), **Failure modes** (how it breaks, who notices). Then compare.

Self-grading rubric (applies to all drills):
- **Basic:** correct inputs/outputs; names the obvious error path.
- **Solid:** identifies the invariant and every side effect, including logs/caches.
- **Strong:** identifies at least one *non-local* failure mode (something that breaks a different file) and one untested behavior.

---

## Drill 1 — [with-rls.ts#L39-L70](../../lib/db/with-rls.ts#L39-L70) (`withRLS`)

Notes: Inputs: `userId` (server-derived — never client-passed directly; verify by callers), callback. Invariant: **every statement in the callback sees `app.current_user_id = userId`, and nothing outside the transaction does** (`set_config` third arg `true` = transaction-local). Failure modes: error-message sniffing for RLS violations ([L58-L61](../../lib/db/with-rls.ts#L58-L61)) breaks if Postgres wording changes; a callback that *catches and swallows* DB errors defeats the translation. Non-local: if the pool connects as table owner, policies silently don't apply — the invariant depends on [db/index.ts#L25-L28](../../lib/db/index.ts#L25-L28).

## Drill 2 — [persist-stream-results.ts#L35-L79](../../lib/streaming/helpers/persist-stream-results.ts#L35-L79) (initial-save settlement)

Notes: Inputs include two *promises created elsewhere* — the ambient state is the drill's point. Invariant: the assistant message is only saved after a chat row exists (FK). Failure modes: fallback duplicate-key path treats *any* "duplicate key"/"unique constraint" substring as success ([L56-L58](../../lib/streaming/helpers/persist-stream-results.ts#L56-L58)) — a unique violation from a *different* constraint would be mistaken for success (non-local, subtle). Untested: no test simulates initial-save rejection + fallback duplicate (check [prepare-messages.test.ts](../../lib/streaming/helpers/__tests__/prepare-messages.test.ts) — prep only).

## Drill 3 — [search.ts#L54-L108](../../lib/tools/search.ts#L54-L108) (provider selection + searxng self-call)

Notes: Inputs: LLM-controlled args + env. Dependency surprise: `getBaseUrlString()` — request headers reach into a *tool*. Invariant: `searchResult` is always assigned or the function throws. Failure modes: env typo in `SEARCH_API` silently falls to Tavily (factory default, [providers/index.ts#L33-L35](../../lib/tools/search/providers/index.ts#L33-L35)); the self-fetch has no timeout; a non-OK advanced-search response throws with status text but loses the body.

## Drill 4 — [prepare-messages.ts#L25-L79](../../lib/streaming/helpers/prepare-messages.ts#L25-L79) (regeneration branch)

Notes: The richest drill. Invariant *attempted*: messages sent to the model = exactly the prefix before the regenerated point. Failure modes: positional fallback ([L39-L55](../../lib/streaming/helpers/prepare-messages.ts#L39-L55)) can pick the wrong message when ids desync; deletes commit before the new stream succeeds (compensation gap); in-memory slice vs cache comment ([L61-L63](../../lib/streaming/helpers/prepare-messages.ts#L61-L63)) reveals a systemic staleness trap others will hit.

## Drill 5 — [context-window.ts#L189-L285](../../lib/utils/context-window.ts#L189-L285) (`truncateMessages`)

Notes: Invariants: result starts with a user message ([L280-L283](../../lib/utils/context-window.ts#L280-L283)); first user message kept only if ≤30% of budget. Failure modes: token counts are estimates (wrong tokenizer for non-OpenAI, [L53-L62](../../lib/utils/context-window.ts#L53-L62)); the while-shift loop ([L253-L263](../../lib/utils/context-window.ts#L253-L263)) can evict everything and still not fit; pathological case — a huge *last* user message yields an empty result (does the caller handle `[]`? check [create-chat-stream-response.ts#L149-L159](../../lib/streaming/create-chat-stream-response.ts#L149-L159) — no; *investigate*).

## Drill 6 — [upload/route.ts#L15-L70](../../app/api/upload/route.ts#L15-L70) (upload handler)

Notes: Inputs: multipart form — `file.type` and `chatId` are client-controlled. Invariant: object keys always namespaced under the *authenticated* userId ([L78](../../app/api/upload/route.ts#L78)). Failure modes: MIME whitelist checks a client-asserted string; 500 leaks `err.message` to the client ([L63-L69](../../app/api/upload/route.ts#L63-L69)) — information disclosure smell.

## Drill 7 — [adaptive-limit.ts#L42-L111](../../lib/rate-limit/adaptive-limit.ts#L42-L111) (`checkAdaptiveLimit`)

Notes: Outputs: a rich result object with `enforced` flag — consumers *can* distinguish real checks from no-ops (used to gate analytics, [L125-L132](../../lib/rate-limit/adaptive-limit.ts#L125-L132)); good design detail. Failure modes: new Redis client per call ([L73-L76](../../lib/rate-limit/adaptive-limit.ts#L73-L76)) — fine for Upstash REST, a bug if copied to TCP Redis; count incremented even when the request later fails elsewhere (a blocked-then-500 request still consumed quota).

## Drill 8 — [citation.ts#L72-L110](../../lib/utils/citation.ts#L72-L110) (`processCitations`)

Notes: Inputs: LLM-authored markdown + maps derived from tool outputs. Invariant: output contains no `[n](#id)` pattern that could render as a broken anchor. Failure modes: silent empty-string replacement destroys evidence of model misbehavior; regex allows spaces in `[ 1 ]` but not in the id; `encodeURI` guards the href but the *domain label* comes from the cited page's URL — displayed text is derived data, safe, but worth checking `displayUrlName` ([domain.ts](../../lib/utils/domain.ts)).

## Drill 9 — [chat.tsx#L254-L297](../../components/chat.tsx#L254-L297) (copy-shortcut effect)

Notes: Dependencies: window event bus + `messagesRef` + `offsetParent` visibility probe. Invariant: exactly one visible Chat instance handles the event (`defaultPrevented` handshake). Failure modes: `offsetParent` is null for `position: fixed` ancestors too — visibility probe has CSS-dependent edge cases; clipboard API needs secure context.

## Drill 10 — [fetch-models.ts#L382-L418](../../lib/models/fetch-models.ts#L382-L418) (`fetchAvailableModels`)

Notes: Invariant: returns *something* within TTL semantics; each vendor fetcher already degrades to `[]`. Failure modes: no single-flight — concurrent cold requests each fan out to 5 vendors; a vendor hanging (no timeout in [fetchJson](../../lib/models/fetch-models.ts#L150-L159)) delays `Promise.all` indefinitely — and this runs during *model selection* on the chat path when no cookie/default applies ([model-selection.ts#L140](../../lib/utils/model-selection.ts#L140)). Non-local latency risk found by reading two files together — that's the Strong bar.
