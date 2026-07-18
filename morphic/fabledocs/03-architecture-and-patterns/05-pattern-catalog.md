# Pattern Catalog

Sixteen pattern cards. The goal is **recognition** — seeing the shape in any repo and knowing when it's the wrong shape — not name-dropping. Each card: problem → shape → real anchors → why it works → failure modes → interview angle → drill.

---

## Pattern 1: Adapter + Factory (search providers)

Problem it solves: five search vendors with different APIs must look identical to the tool layer.
General shape: an interface (`search(query, …): Promise<SearchResults>`), one class per vendor, a factory switching on config.
Real example: [lib/tools/search/providers/index.ts#L16-L37](../../lib/tools/search/providers/index.ts#L16-L37); interface via [base.ts](../../lib/tools/search/providers/base.ts); concrete: [tavily.ts#L11-L104](../../lib/tools/search/providers/tavily.ts#L11-L104).
Second example: model providers behind `createProviderRegistry` ([lib/utils/registry.ts#L9-L31](../../lib/utils/registry.ts#L9-L31)).
Why this implementation works: the normalized `SearchResults` type is the contract; callers never see vendor JSON.
Failure modes: leaks — Brave gets extra positional args at the call site ([search.ts#L114-L127](../../lib/tools/search.ts#L114-L127)), so "identical" is already false; unknown provider silently defaults to Tavily ([index.ts#L33-L35](../../lib/tools/search/providers/index.ts#L33-L35)) which hides typos in `SEARCH_API`.
Use it when: ≥2 interchangeable vendors. Avoid it when: one vendor and no concrete second candidate — the abstraction tax buys nothing.
Interview angle: "How would you support multiple payment/search/email providers?" Contrast options-object vs positional params to fix the Brave leak.
Drill: sketch the options-object refactor of `SearchProvider.search` and list every file it touches.

## Pattern 2: Tenant isolation via Postgres RLS + transaction-scoped identity

Problem it solves: enforcing "users only see their rows" in one place, immune to a forgotten `WHERE user_id = ?`.
General shape: policies compare row columns to `current_setting('app.current_user_id')`; app wraps every query in a transaction that `set_config`s the id.
Real example: policies [schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67) (and via-join for messages/parts, [L92-L117](../../lib/db/schema.ts#L92-L117), [L226-L254](../../lib/db/schema.ts#L226-L254)); wrapper [with-rls.ts#L39-L70](../../lib/db/with-rls.ts#L39-L70).
Second example: composite index specifically to keep the RLS subquery fast — [schema.ts#L50-L51](../../lib/db/schema.ts#L50-L51).
Why this implementation works: `set_config(..., true)` is transaction-local, so pooled connections can't bleed identity; parameterized SQL blocks injection (tested at [with-rls.test.ts#L56-L70](../../lib/db/__tests__/with-rls.test.ts#L56-L70)).
Failure modes: (1) connecting as a table **owner bypasses RLS** — hence `DATABASE_RESTRICTED_URL` preference ([db/index.ts#L25-L28](../../lib/db/index.ts#L25-L28)) and the startup user check ([L68-L100](../../lib/db/index.ts#L68-L100)); (2) `withOptionalRLS(null, …)` runs with **no isolation** ([with-rls.ts#L80-L90](../../lib/db/with-rls.ts#L80-L90)) — every call site is a place to audit; (3) policy joins cost performance on big tables.
Use it when: multi-tenant data where a missed filter is catastrophic. Avoid when: single-tenant, or your team can't operate two DB roles.
Interview angle: "App-level vs DB-level authorization" — the strongest answer names *defense in depth*: this repo checks ownership in code ([create-chat-stream-response.ts#L64-L70](../../lib/streaming/create-chat-stream-response.ts#L64-L70)) *and* in the DB.
Drill: find every caller of `withOptionalRLS` and classify each null-userId path as intended-public or *investigate*.

## Pattern 3: Streaming tool as async generator

Problem it solves: tools that take seconds must show progress before completing.
General shape: `async *execute()` yields `{state:'searching'}` → work → yields `{state:'complete', ...}`.
Real example: [search.ts#L24-L165](../../lib/tools/search.ts#L24-L165); second: [fetch.ts#L164-L191](../../lib/tools/fetch.ts#L164-L191).
Why it works: the AI SDK forwards each yield as a UI part state (`input-streaming` → `output-available`), which the DB also models ([schema.ts#L217-L220](../../lib/db/schema.ts#L217-L220)).
Failure modes: consumers must handle both iterator and promise shapes — see the defensive dance in [search.ts#L203-L216](../../lib/tools/search.ts#L203-L216); throwing between yields leaves the UI on a spinner unless error states are mapped.
Use when: any multi-second tool. Avoid when: sub-100ms operations — states add noise.
Interview angle: generators/iterators in production JS — Q4 in [01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md).
Drill: write (fake, labeled) a `weather` tool with three yields including an error state.

## Pattern 4: Decorator/wrapper to constrain a tool per mode

Problem it solves: quick mode must force `type:'optimized'` without forking the search tool.
General shape: wrap the tool, override params, delegate and re-yield.
Real example: [researcher.ts#L20-L63](../../lib/agents/researcher.ts#L20-L63) (`wrapSearchToolForQuickMode`).
Second example: No second example found.
Why it works: same description/schema so the model's view is unchanged; only server behavior differs.
Failure modes: the cast `as T` lies to the compiler; if the wrapped tool's yield shape drifts, this compiles but breaks at runtime.
Use when: policy differences per context. Avoid when: the policy belongs in the prompt (cheaper) or in the tool itself (single source).
Interview angle: decorator pattern in TS; also "how do you constrain LLM behavior *server-side* instead of trusting the prompt?" — strong answer: prompts request, code enforces.
Drill: find what else quick mode changes besides the wrapper (prompt, `activeTools`, `maxSteps` — [researcher.ts#L92-L101](../../lib/agents/researcher.ts#L92-L101)).

## Pattern 5: Fixed-window rate limit with Redis INCR+EXPIRE, fail-open

Problem it solves: cheap daily caps without a limits service.
General shape: key `rl:{scope}:{id}:{yyyy-mm-dd}`; INCR; on first hit EXPIRE to midnight; allow if count ≤ limit; on any Redis failure, allow.
Real example: [adaptive-limit.ts#L72-L110](../../lib/rate-limit/adaptive-limit.ts#L72-L110); seconds: [guest-limit.ts#L45-L78](../../lib/rate-limit/guest-limit.ts#L45-L78), [chat-limits.ts#L45-L78](../../lib/rate-limit/chat-limits.ts#L45-L78).
Why it works: atomic INCR means no lost counts under concurrency; date-keyed windows self-partition; a 3s `Promise.race` timeout ([adaptive-limit.ts#L81-L86](../../lib/rate-limit/adaptive-limit.ts#L81-L86)) keeps Redis latency off the critical path.
Failure modes: INCR-then-EXPIRE is two calls — crash between them leaves an immortal key; fixed windows allow 2× burst at midnight; fail-open means outage = no limits (deliberate here); `x-forwarded-for` identity is spoofable without a trusted proxy ([route.ts#L77-L82](../../app/api/chat/route.ts#L77-L82)).
Use when: soft product quotas. Avoid when: billing/abuse-critical caps — use atomic scripts + sliding windows and fail-closed.
Interview angle: the canonical "design a rate limiter" — Q35. Naming fixed-vs-sliding window and fail-open-vs-closed *from real code you can cite* is a standout answer.
Drill: write the Lua (or `MULTI`) version that makes INCR+EXPIRE atomic (fake code, labeled).

## Pattern 6: Optimistic write with awaited settlement + idempotent fallback

Problem it solves: don't make the user wait for a DB insert before the LLM starts answering.
General shape: fire the insert, keep the promise, stream; at finish, await the promise; on failure retry; treat duplicate-key as success.
Real example: fire [prepare-messages.ts#L91-L116](../../lib/streaming/helpers/prepare-messages.ts#L91-L116); settle [persist-stream-results.ts#L35-L79](../../lib/streaming/helpers/persist-stream-results.ts#L35-L79) (duplicate-key-as-success at [L54-L68](../../lib/streaming/helpers/persist-stream-results.ts#L54-L68)).
Second example: title generated in parallel and awaited at persist time ([create-chat-stream-response.ts#L161-L173](../../lib/streaming/create-chat-stream-response.ts#L161-L173), [persist-stream-results.ts#L32-L33](../../lib/streaming/helpers/persist-stream-results.ts#L32-L33)).
Why it works: **idempotency** — client-generated chatId means "create chat" can run twice safely; duplicate key = the other attempt won, proceed.
Failure modes: if the process dies mid-stream, the user saw an answer that was never saved (window of unpersisted truth); error detection by message-string matching ([L56-L58](../../lib/streaming/helpers/persist-stream-results.ts#L56-L58)) is brittle across drivers.
Use when: latency-critical paths with idempotent writes. Avoid when: the write must gate the response (payments!).
Interview angle: "How do you make an API idempotent?" — client-generated ids + unique constraints is the textbook answer, and you can cite this file.
Drill: list the three places client-generated ids appear (chatId [chat.tsx#L58](../../components/chat.tsx#L58), messageId [prepare-messages.ts#L86-L89](../../lib/streaming/helpers/prepare-messages.ts#L86-L89), cuid2 default [schema.ts#L23](../../lib/db/schema.ts#L23)).

## Pattern 7: Cache-aside with tag invalidation (`unstable_cache` + `revalidateTag`)

Problem it solves: chat loads happen on every page view and every stream start; DB reads are the hotspot.
General shape: wrap the read in a keyed cache with tags; every mutation revalidates the tag.
Real example: [lib/actions/chat.ts#L17-L34](../../lib/actions/chat.ts#L17-L34) (keys include chatId *and* requesting user; tags `chat-{id}` + `chat`; 60s TTL); invalidations sprinkled at [L92](../../lib/actions/chat.ts#L92), [L132](../../lib/actions/chat.ts#L132), [L214](../../lib/actions/chat.ts#L214), [L236](../../lib/actions/chat.ts#L236), [route.ts#L208-L212](../../app/api/chat/route.ts#L208-L212).
Why it works: per-chat tags make invalidation surgical; the `userId` in the cache key prevents cross-user cache poisoning — a subtle, load-bearing detail.
Failure modes: **stale-within-request** — revalidation doesn't refresh an already-read value, which is why regeneration uses in-memory slices instead of re-reading ([prepare-messages.ts#L61-L63](../../lib/streaming/helpers/prepare-messages.ts#L61-L63)); forgetting one mutation path leaves ghosts for up to 60s.
Use when: read-heavy, tolerance for seconds of staleness. Avoid when: read-after-write must be exact.
Interview angle: "How do you invalidate caches?" — Q28; the userId-in-key detail is a great cache-poisoning answer.
Drill: grep every write to chats/messages and match each to its `revalidateTag` — is any missing? (Check [updateChatTitle](../../lib/db/actions.ts#L374-L388) called from [persist-stream-results.ts#L100-L108](../../lib/streaming/helpers/persist-stream-results.ts#L100-L108) — *investigate*.)

## Pattern 8: Wide-table polymorphism (parts)

Problem it solves: heterogeneous message parts in one queryable table.
General shape: `type` discriminator + sparse per-type columns + CHECK constraints per type.
Real example: [schema.ts#L123-L256](../../lib/db/schema.ts#L123-L256); checks [L207-L224](../../lib/db/schema.ts#L207-L224); mapping layer [message-mapping.ts](../../lib/utils/message-mapping.ts).
Second example: No second example found.
Why it works: one indexed read per message; DB-level shape guarantees; no JSON parsing for common fields.
Failure modes: schema migration per new tool; 700-line mapping file; sparse rows waste little in Postgres but the *code* pays; alternatives (jsonb column, or table-per-type) each trade differently.
Use when: bounded, stable set of variants + need for DB constraints. Avoid when: variants churn weekly — use jsonb + app validation.
Interview angle: classic data-modeling question ("store polymorphic events") — Q23. Know all three options and this repo's choice.
Drill: write the jsonb alternative schema (fake, labeled) and name two constraints you lose.

## Pattern 9: Env-presence feature flags with deployment-mode guardrails

Problem it solves: one codebase serving personal Docker, self-host, and managed cloud.
General shape: features switch on env presence; a mode flag (`MORPHIC_CLOUD_DEPLOYMENT`) adds hard guardrails.
Real example: provider enablement [registry.ts#L56-L76](../../lib/utils/registry.ts#L56-L76); anonymous-mode ban on cloud [get-current-user.ts#L25-L29](../../lib/auth/get-current-user.ts#L25-L29); limits only on cloud [adaptive-limit.ts#L47-L56](../../lib/rate-limit/adaptive-limit.ts#L47-L56); pinned cloud models [load-models-config.ts#L46-L63](../../lib/config/load-models-config.ts#L46-L63).
Why it works: zero-config personal use; the cloud guard turns a foot-gun (auth off in prod) into a thrown error.
Failure modes: behavior matrix explodes (3 modes × N flags); tests run in yet another mode; "works locally" stops meaning anything. Mitigation: guardrail asserts like the cloud/auth one.
Interview angle: feature flags vs config vs code branches — Q38.
Drill: build the truth table for {ENABLE_AUTH, ENABLE_GUEST_CHAT, MORPHIC_CLOUD_DEPLOYMENT} → who can chat, what persists.

## Pattern 10: Model-context budget management (prune → truncate)

Problem it solves: conversations outgrow model context windows.
General shape: cheap structural pruning first (drop old reasoning/tool calls), then token-counted truncation keeping first user message + recent tail.
Real example: prune [create-chat-stream-response.ts#L142-L147](../../lib/streaming/create-chat-stream-response.ts#L142-L147); truncate [context-window.ts#L189-L285](../../lib/utils/context-window.ts#L189-L285) (first-message ≤30% budget rule [L227-L237](../../lib/utils/context-window.ts#L227-L237); must-start-with-user [L280-L283](../../lib/utils/context-window.ts#L280-L283)).
Second example: same pipeline in the guest path ([create-ephemeral-chat-stream-response.ts#L71-L83](../../lib/streaming/create-ephemeral-chat-stream-response.ts#L71-L83)).
Why it works: tiered cost — pruning is free, tiktoken counting is paid only when needed ([context-window.ts#L290-L302](../../lib/utils/context-window.ts#L290-L302)).
Failure modes: the model table is hardcoded and **stale** — unknown models (including the current defaults) fall to a 16,384-token window ([L37-L38](../../lib/utils/context-window.ts#L37-L38)), likely over-truncating (*confirmed by reading; runtime impact: investigate*); tokenizer approximations for non-OpenAI models ([L53-L62](../../lib/utils/context-window.ts#L53-L62)).
Interview angle: LLM app design staple ("how do you handle long conversations?") — Q11.
Drill: compute the max input budget for `gpt-4.1` from the code (128000 − 16384 − 12800 = 98816) and for an unknown model.

## Pattern 11: Prompt-as-contract with UI

Problem it solves: the UI needs machine-readable structure from free-text LLM output.
General shape: prompt specifies an exact microformat; tool output carries the lookup table; UI parses with a regex and fails soft.
Real example: format spec [search-mode-prompts.ts#L72-L101](../../lib/agents/prompts/search-mode-prompts.ts#L72-L101); data [search.ts#L145-L151](../../lib/tools/search.ts#L145-L151); parser [citation.ts#L82-L109](../../lib/utils/citation.ts#L82-L109).
Second example: spec fences (Flow 7) — same idea with a real parser instead of a regex.
Why it works: fails soft — invalid citations render as nothing, not as broken UI.
Failure modes: three-party contract with no type checking; silent drops hide model regressions (a metric on dropped citations would fix observability); prompt edits are production changes without CI coverage.
Interview angle: "How do you get structured output from LLMs?" — spectrum: prompt-format → tool-calls → constrained decoding. You can cite both ends from this repo.
Drill: find what happens to a citation whose number exceeds 100 ([citation.ts#L88-L90](../../lib/utils/citation.ts#L88-L90)) and defend that magic bound.

## Pattern 12: Module-scoped TTL cache

Problem it solves: model discovery hits 5 vendor APIs; you can't do that per request.
General shape: module-level `{ value, expiresAt }`; check-before-fetch; overwrite after.
Real example: [fetch-models.ts#L39-L44](../../lib/models/fetch-models.ts#L39-L44), [L382-L418](../../lib/models/fetch-models.ts#L382-L418) (2-min TTL).
Second example: encoder cache [context-window.ts#L44](../../lib/utils/context-window.ts#L44), [L125-L145](../../lib/utils/context-window.ts#L125-L145); config memo [load-models-config.ts#L14-L15](../../lib/config/load-models-config.ts#L14-L15).
Why it works: serverless-friendly (per-instance, no coordination), trivially correct for read-only data.
Failure modes: N instances = N caches = N× vendor calls; no stampede protection (concurrent misses all fetch); stale across deploys is fine here but the pattern gets copy-pasted into places it isn't.
Interview angle: cache placement (module vs Redis vs CDN) — Q28 follow-up.
Drill: add (on paper) single-flight de-duplication to `fetchAvailableModels` — store the in-flight promise, not just the value.

## Pattern 13: Graceful-degradation fallback chains

Problem it solves: many optional dependencies (providers, Redis, title model) must not take the product down.
General shape: try preferred → log → try fallback → finally a safe default.
Real example: model selection cookie → default → first-fetched ([model-selection.ts#L107-L164](../../lib/utils/model-selection.ts#L107-L164)); title LLM → 75-char prefix ([title-generator.ts#L26](../../lib/agents/title-generator.ts#L26), [L61-L74](../../lib/agents/title-generator.ts#L61-L74)); fetch tool Jina → Tavily extract ([fetch.ts#L176-L184](../../lib/tools/fetch.ts#L176-L184)); regenerate-by-id → last-message fallback ([prepare-messages.ts#L39-L55](../../lib/streaming/helpers/prepare-messages.ts#L39-L55)).
Why it works: each fallback is *cheaper and dumber* than the primary — the right direction.
Failure modes: silent fallbacks mask outages (you ship "Untitled" chats for a week before noticing); fallbacks that can be *wrong* (the regenerate positional fallback) are worse than errors.
Interview angle: resilience question — "what should happen when your Redis/LLM/vendor is down?" Distinguish degrade-safely from fail-loudly, per operation.
Drill: classify each of the four examples above as "safe to silently degrade" or "should alert" and justify.

## Pattern 14: Middleware-derived absolute base URL

Problem it solves: server code sometimes needs its own public origin (self-calls, redirects) behind proxies.
General shape: middleware computes origin from `x-forwarded-proto/host` and stamps headers; helpers read them.
Real example: [proxy.ts#L6-L38](../../proxy.ts#L6-L38); consumed for the advanced-search self-call ([search.ts#L88-L101](../../lib/tools/search.ts#L88-L101)) via [lib/utils/url.ts](../../lib/utils/url.ts).
Second example: No second example found.
Why it works: one place understands proxies; everyone else reads a header.
Failure modes: forwarded headers are attacker-controlled if no trusted proxy sits in front — an absolute-URL builder fed by them can become an SSRF/poisoning primitive (*possible risk*; see risk register); self-HTTP-calls add a network hop where a function call would do.
Interview angle: "Why is `Host` untrustworthy?" — classic web-security probe.
Drill: propose replacing the self-call with a direct function import; list what you gain (no hop, no header trust) and lose (route-level caching in [advanced-search/route.ts](../../app/api/advanced-search/route.ts)).

## Pattern 15: Human-in-the-loop tool (no execute)

Problem it solves: some "tool calls" are actually questions for the user.
General shape: define the tool schema but omit `execute`; the UI renders the pending call and submits the user's answer as the tool result.
Real example: [question.ts#L8-L15](../../lib/tools/question.ts#L8-L15) (comment: "execute function removed to enable frontend confirmation"); the client resolves it via `addToolResult` ([chat.tsx#L493-L528](../../components/chat.tsx#L493-L528), rendered by [question-confirmation.tsx](../../components/question-confirmation.tsx)).
Second example: No second example found.
Why it works: reuses the entire tool-call plumbing (persistence, states, streaming) for a UX interaction.
Failure modes: a pending question part persists in `input-available` state forever if the user never answers — check how history handles it (*investigate*).
Interview angle: agent design — "how do you pause an agent for user input?" This is the cleanest published answer.
Drill: trace one `askQuestion` round-trip through the part states in the DB ([schema.ts#L217-L220](../../lib/db/schema.ts#L217-L220)).

## Pattern 16: Schema-per-model capability adaptation

Problem it solves: some models (OpenAI o-series) need strict JSON schemas for tool calls.
General shape: pick the Zod schema variant at tool-creation time based on the model id.
Real example: [lib/schema/search.tsx#L81-L94](../../lib/schema/search.tsx#L81-L94); tool built per model at [search.ts#L20-L23](../../lib/tools/search.ts#L20-L23) and [researcher.ts#L82](../../lib/agents/researcher.ts#L82).
Second example: reasoning-part stripping only for OpenAI ([create-chat-stream-response.ts#L129-L136](../../lib/streaming/create-chat-stream-response.ts#L129-L136), with the upstream issue linked in the comment).
Why it works: capability differences are isolated at construction, not scattered through execution.
Failure modes: string-prefix sniffing (`modelName?.startsWith('o')`) is fragile as vendors rename; every new special case is another axis in the test matrix.
Interview angle: "How do you handle provider quirks without if-statements everywhere?" — construction-time adaptation vs runtime branching.
Drill: find the third model-specific branch in the codebase (Ollama's `supportedUrls` override — [registry.ts#L36-L49](../../lib/utils/registry.ts#L36-L49)).

---

## Recognition drill (whole catalog)

Pick any other repo you know (e.g., your last job's). For each of the 16 patterns, write one line: *present / absent / present-but-worse*, with the file if present. This inventory is exactly the "walk me through a codebase you know" interview answer — see [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md), story 1.
