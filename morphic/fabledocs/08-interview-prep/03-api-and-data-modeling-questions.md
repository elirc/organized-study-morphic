# API & Data Modeling Questions (Q23–Q41)

---

## Q23: Design a data model for chat messages with mixed content types.
Anchor: [schema.ts#L73-L256](../../lib/db/schema.ts#L73-L256).
Junior: "A messages table with a text column."
Mid: "Three tables: chats → messages → parts, cascade-deleted. Parts is a wide polymorphic table with a `type` discriminator and per-type sparse columns (`text_text`, `tool_search_input`…) plus CHECK constraints. One indexed read reconstructs a message with ordered parts."
Senior: "Three real options: wide table (this — DB-enforced shape, migration per tool, a 700-line mapper), a single `jsonb content` column (fewer migrations, weaker DB constraints, app-side validation), or table-per-type (clean but many joins). Morphic chose wide; the cost is the mapping file and a schema change per new tool. I'd defend it while noting jsonb is the move if tool types churn weekly."
Follow-ups: how do you evolve it; ordering guarantees.

## Q24: Why one table with many nullable columns instead of jsonb?
Anchor: [schema.ts#L167-L200](../../lib/db/schema.ts#L167-L200), constraints [L207-L224](../../lib/db/schema.ts#L207-L224).
Mid: "DB-enforceable constraints — a text part *must* have text, tool parts *must* have a call id and valid state ([L217-L223](../../lib/db/schema.ts#L217-L223)); jsonb can't express that without triggers. And no JSON parsing for hot fields."
Senior: "The trade is migrations and the mapper; sparse columns cost little in Postgres storage but the *code* pays. jsonb + Zod-on-read moves the guarantee to the app and risks old rows not matching current shapes. Pick by churn rate and how much you value DB-level guarantees."

## Q25: What's a transaction? Show a boundary that needs one.
Anchor: chat+first-message [db/actions.ts#L394-L441](../../lib/db/actions.ts#L394-L441); RLS forces transactions [with-rls.ts#L44-L54](../../lib/db/with-rls.ts#L44-L54).
Mid: "All-or-nothing. Morphic creates a chat + its first message + parts in one transaction — a chat with a half-saved message must be impossible. Message upsert also replaces parts transactionally (delete-then-insert), so readers never see a partless message."
Senior: "A subtle one: *every* RLS operation is a transaction because `set_config(..., local=true)` is transaction-scoped — isolation and atomicity are coupled here. Being able to say which parts of the system are strongly consistent (these writes) and which are eventually consistent (the streamed answer persists after the user sees it, titles arrive later) is the real skill."
Follow-ups: isolation levels; long transactions and locks.

## Q26: How would you support multiple providers (search/payment/email) cleanly?
Anchor: [providers/index.ts#L16-L37](../../lib/tools/search/providers/index.ts#L16-L37), [tavily.ts](../../lib/tools/search/providers/tavily.ts).
Mid: "Adapter + factory: one interface, one class per vendor, a factory switching on config. Morphic has five search providers behind `SearchProvider.search()` returning a normalized `SearchResults`."
Senior: "Guard the normalized contract under feature pressure — the moment one provider needs special args, it leaks (Brave already does, [search.ts#L114-L127](../../lib/tools/search.ts#L114-L127)). Fix with an options object so all providers share the signature. Also handle the unknown-provider case explicitly — silent default-to-Tavily hides config typos."
Follow-ups: how to add a provider; testing the factory.

## Q27: How do you ship a schema change with zero downtime?
Anchor: [drizzle/](../../drizzle/) (11 forward-only migrations), [02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md).
Mid: "Additive first: new columns nullable/defaulted, backfill separately, tighten constraints last. Edit `schema.ts`, generate the SQL, review it like code."
Senior: "Expand/contract: the migration must be backward-compatible with the *currently running* app so a rolling deploy doesn't break. Morphic's migrations are forward-only (no down files), so 'rollback' means a new inverse migration — I'd never claim down-migrations exist. Watch for locking: `ADD COLUMN NOT NULL` without default rewrites a big table. And old persisted rows must keep loading (the spec-migration precedent)."
Follow-ups: online index builds; large backfills.

## Q28: How do you cache reads, and how do you invalidate?
Anchor: `unstable_cache`+tags [lib/actions/chat.ts#L17-L34](../../lib/actions/chat.ts#L17-L34).
Mid: "Cache-aside with tagged entries; every mutation calls `revalidateTag('chat-{id}')`. Morphic tags per-chat so invalidation is surgical, with a 60s TTL as a backstop."
Senior: "Two load-bearing details: the cache key includes the requesting userId (else a public/private view could poison across users), and revalidation doesn't refresh a value already read this request — hence in-memory slices during regeneration. Naming a mutation that *forgets* to revalidate (candidate: the title update path) is the review skill."
Follow-ups: cache stampede/single-flight; where to cache (module/Redis/CDN).

## Q29: Authentication vs authorization — where does each live?
Anchor: authn [get-current-user.ts](../../lib/auth/get-current-user.ts), [proxy.ts](../../proxy.ts); authz code+RLS [03-validation-auth-and-permissions.md](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).
Mid: "Authn = who you are (Supabase session → userId, or anonymous mode). Authz = what you may do. Morphic authenticates in middleware/routes and authorizes with ownership checks + Postgres RLS."
Senior: "Note middleware treats `/api` as public — API routes authenticate themselves; middleware redirects are UX. Authorization is defense-in-depth: an app-level `userId !== chat.userId → 403` *and* RLS beneath it, so a forgotten filter still can't leak. The word to use is *isolation*, enforced in the database not just the app."
Follow-ups: session management; JWT vs session cookies.

## Q30: How do you stop users reading each other's data?
Anchor: RLS policies [schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67), [with-rls.ts](../../lib/db/with-rls.ts).
Mid: "Row-level security: policies compare `user_id` to a session setting; the app sets that setting per transaction via `set_config`. Even buggy app code can't return another tenant's rows."
Senior: "The mechanism's integrity depends on connecting as a *restricted* role — table owners bypass RLS, so Morphic prefers `DATABASE_RESTRICTED_URL` and verifies the connected user at startup. `set_config(local=true)` is transaction-scoped so pooled connections don't leak identity. And there's a composite index specifically for the policy subquery — RLS isn't free, you index for it."
Follow-ups: RLS performance; multi-tenant vs per-tenant DB.

## Q31: Design a streaming chat endpoint that also persists.
Anchor: Flow 1 ([05-key-flows.md](../01-codebase-cartography/05-key-flows.md)).
Mid: "One POST: authenticate → rate-limit → select model → stream a tool-using agent over SSE → persist in the stream's `onFinish`. Morphic starts the answer immediately and saves after, so the DB write isn't on the latency path."
Senior: "Persistence hangs off a live stream, so I call `consumeStream()` to keep producing even if the client leaves — otherwise abandoned tabs abort the save. The gap is durability: instance death loses in-flight writes silently; the upgrade is an outbox written in the same transaction as the user message. Optimistic creation is safe only because ids are client-generated and writes are idempotent (duplicate-key = success)."
Follow-ups: what if two tabs send at once; backpressure.

## Q32: Walk me through finding an IDOR in this app.
Anchor: the trace in [03-validation-auth-and-permissions.md](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).
Mid: "Try to access someone else's resource by id — POST `/api/chat` with a victim's chatId. Then follow the read: `loadChat(chatId, attackerId)` runs under the attacker's RLS context, so the victim's private chat is invisible."
Senior: "Both halves: the read returns nothing (RLS), and any write inserts under the *attacker's* userId (RLS `withCheck`), so nothing crosses tenants. The method is: take hostile input, follow the write/read path, find the invariant that saves you — or doesn't. Here two independent layers (app 403 + RLS) do."
Follow-ups: where would you add a test for this (the RLS integration test).

## Q33: How do you secure file uploads?
Anchor: [upload/route.ts](../../app/api/upload/route.ts), Flow 6.
Mid: "Auth-gate it; server-side re-validate size and MIME (never trust client checks); sanitize the filename; namespace the object key by userId. Morphic does all four — 5MB cap, jpeg/png/pdf whitelist, sanitized name, `{userId}/chats/{chatId}/…` key."
Senior: "Two gaps I'd flag: MIME comes from the client's `file.type` — no magic-byte sniff, so a renamed file passes; and public-by-URL storage is 'unlisted', not private — signed URLs are the upgrade, at the cost of breaking old history links. Also `chatId` in the key is unverified input."
Follow-ups: virus scanning; direct-to-storage presigned uploads.

## Q34: What's SSRF and is this app exposed?
Anchor: fetch tool [fetch.ts#L9-L21](../../lib/tools/fetch.ts#L9-L21); risk register.
Mid: "Server-Side Request Forgery — tricking the server into fetching internal resources. Morphic's fetch tool retrieves arbitrary URLs server-side, and the model (or a user URL) chooses them, so it's a candidate surface."
Senior: "I'd *investigate* before claiming a vuln: is there egress filtering at the infra level? If not, an attacker could target `169.254.169.254` (cloud metadata) or internal services. The fix resolves DNS then checks the *resolved* IP against private ranges (hostname checks miss DNS rebinding), plus an allowlist. I'd label this 'possible risk, verify exploitability', not 'bug'."
Follow-ups: metadata endpoints; DNS rebinding.

## Q35: Design a rate limiter.
Anchor: [adaptive-limit.ts](../../lib/rate-limit/adaptive-limit.ts), Pattern 5.
Mid: "Fixed window: Redis key `rl:{scope}:{id}:{date}`, INCR, EXPIRE to window end, allow if count ≤ limit. Morphic keys per user/IP per day and fails open on Redis errors."
Senior: "Tradeoffs to name: fixed windows allow a 2× burst at the boundary — sliding window log/counter fixes it at higher cost. INCR-then-EXPIRE is two ops; a crash between leaves an immortal key — make it atomic with Lua. Fail-open vs fail-closed is a product call (Morphic chooses availability). And guest identity via `x-forwarded-for` is spoofable without a trusted proxy — so this is a soft quota, not an abuse control."
Follow-ups: distributed rate limiting; token bucket vs fixed window.

## Q36: What happens if your server dies mid-request?
Anchor: [persist-stream-results.ts](../../lib/streaming/helpers/persist-stream-results.ts), Flow 1.
Mid: "In Morphic the user may have seen a streamed answer that was never saved — persistence runs in `onFinish` after the response. On refresh, the chat could have the user message but no answer."
Senior: "Draw the failure tree by time-of-death: before the optimistic insert resolves / during stream / inside onFinish before the message save / after save before title. Each yields a different post-refresh state. The durability fix is an outbox in the same transaction as the user message + an idempotent drainer — at-least-once delivery, since exactly-once is impossible."
Follow-ups: idempotency keys; dead-letter handling.

## Q37: How do you paginate a large list?
Anchor: offset pagination [db/actions.ts#L288-L314](../../lib/db/actions.ts#L288-L314); index [schema.ts#L45-L48](../../lib/db/schema.ts#L45-L48).
Mid: "Morphic uses offset/limit with a `(user_id, created_at desc)` index and returns `nextOffset`. Simple and fine for small histories."
Senior: "Offset degrades at large offsets — the DB still scans and discards the skipped rows. Keyset pagination on `(created_at, id)` is O(page) regardless of depth and uses the same index; the cost is a cursor contract instead of a page number, and it doesn't do 'jump to page N'. Also `limit` is currently unclamped — a self-inflicted slow query."
Follow-ups: stable ordering with ties; cursor encoding.

## Q38: Feature flags — env vars vs a flag service vs code branches?
Anchor: env-presence gating [registry.ts#L56-L76](../../lib/utils/registry.ts#L56-L76); deployment-mode guard [get-current-user.ts#L25-L29](../../lib/auth/get-current-user.ts#L25-L29).
Mid: "Morphic gates features on env *presence* — a provider is enabled iff its API key exists — plus a `MORPHIC_CLOUD_DEPLOYMENT` mode flag that adds guardrails. Zero-config, cheap."
Senior: "The cost is a behavior matrix that explodes (3 deployment modes × N flags) and tests running in yet another combination — 'works locally' loses meaning. The strong move is encoding illegal combinations as *throws* (auth-off on cloud is blocked). A flag service adds runtime toggling and targeting but is overkill here."
Follow-ups: gradual rollout; killing a flag's tech debt.

## Q39: Tell me about making a destructive operation safe.
Anchor: regeneration delete-before-regenerate [prepare-messages.ts#L57-L78](../../lib/streaming/helpers/prepare-messages.ts#L57-L78), Flow 5.
Mid: "Morphic's regenerate deletes the old answer *before* the new one exists — if the LLM call fails, content is lost. The safe version deletes after success, or soft-deletes."
Senior: "It's correct code in the wrong order — the fix is sequencing plus a regression test that fails the model and asserts the old answer survives. Soft-delete (tombstone + sweep) is more robust but costs storage/complexity; for this, reorder-plus-test is proportionate. The general principle: destructive ops need compensation or ordering that can't lose committed user data."
Follow-ups: undo/redo; optimistic UI rollback.

## Q40: What's your testing philosophy for a codebase like this?
Anchor: [01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md).
Mid: "Test pyramid: wide base of pure-function units (citations, truncation), a middle of units with mocked IO (rate limits mock Redis, RLS mocks the transaction), a narrow top of integration for things a mock can't prove — like RLS policies against real Postgres."
Senior: "Test the *contract*, not the vendor — assert status codes, key formats, TTLs, not that Redis increments. And be honest about gaps: routes, tools, and the 700-line mapping file have no tests here, so a round-trip mapping test is the highest-value thing I'd add. Flake comes from real IO, unfrozen clocks, and shared module caches — mock, freeze, reset."
Follow-ups: coverage targets; testing nondeterministic LLM output.

## Q41: How would you find and fix a performance problem here?
Anchor: [04-performance-thinking.md](../05-quality-engineering/04-performance-thinking.md).
Mid: "Measure first — Morphic has `perfLog`/`perfTime` and Langfuse traces. Then binary-search the domain: DB, vendor network, render, bundle. A concrete N+1 is `clearChats` looping a per-chat delete."
Senior: "The dominant cost is LLM streaming (seconds), so DB/render microopts are noise — the leverage is starting the stream ASAP (why chat creation is optimistic) and not adding sync work before `agent.stream`. Reviewing a PR that adds an `await` before streaming, the perf lens says 'you just taxed every message.' Fix N+1 with a single RLS-scoped delete, but characterization-test the behavior change first."
Follow-ups: profiling tools; when to cache vs optimize the query.
