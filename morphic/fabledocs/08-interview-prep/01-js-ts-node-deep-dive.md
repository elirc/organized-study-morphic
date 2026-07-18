# JS / TS / Node Deep Dive (Q1–Q12)

Each card: what it's really testing → repo anchor → junior/mid/senior answer → follow-ups → drill.

---

## Q1: Explain what happens when you `await` a promise.
Round: JS/TS. Testing: event-loop model, not memorized words.
Anchor: the held title promise — [create-chat-stream-response.ts#L161-L173](../../lib/streaming/create-chat-stream-response.ts#L161-L173) started, awaited later at [persist-stream-results.ts#L32-L33](../../lib/streaming/helpers/persist-stream-results.ts#L32-L33).
Junior: "It pauses the function until the promise resolves."
Mid adds: "It doesn't block the thread — the function suspends and its continuation is queued as a microtask; other work runs meanwhile. In Morphic, title generation is *started* before streaming and only awaited at persistence time, so it runs concurrently with the model."
Senior adds: "Microtasks drain before the next macrotask, so ordering matters; and a promise you hold but don't await needs a `.catch` or a rejection before the await becomes an unhandled rejection — which is exactly why that title promise has a `.catch` attached at creation ([L169-L172](../../lib/streaming/create-chat-stream-response.ts#L169-L172))."
Follow-ups: microtask vs macrotask order; what's an unhandled rejection.
Drill: predict output of a mixed `Promise.resolve().then`/`setTimeout` snippet, then find the held-promise pattern in the repo.

## Q2: How do you run async work in parallel, and when shouldn't you?
Anchor: [fetch-models.ts#L392-L398](../../lib/models/fetch-models.ts#L392-L398) (`Promise.all` of 5 vendors); serial counter-example [lib/actions/chat.ts#L229-L233](../../lib/actions/chat.ts#L229-L233).
Junior: "`Promise.all`."
Mid: "`Promise.all` for independent work — Morphic fetches five model providers in parallel. But `Promise.all` fails fast: one rejection loses the others, so each fetcher catches internally and returns `[]` ([fetch-models.ts#L186-L189](../../lib/models/fetch-models.ts#L186-L189))."
Senior: "Use `allSettled` when partial success is fine and you can't push the try/catch down; and `await` in a loop (`clearChats`) is accidental serialization — N round-trips where one statement would do. Parallel isn't free either: 5 concurrent fan-outs per cold request with no single-flight."
Follow-ups: `all` vs `allSettled` vs `race`; concurrency limits.

## Q3: How do you put a timeout on an async operation?
Anchor: `Promise.race` timeout [adaptive-limit.ts#L81-L86](../../lib/rate-limit/adaptive-limit.ts#L81-L86); AbortController [fetch.ts#L11-L23](../../lib/tools/fetch.ts#L11-L23).
Junior: "`Promise.race` with a setTimeout."
Mid: "Two ways: race against a timeout promise (Morphic does this for Redis), or an AbortController (the fetch tool aborts after 10s). Race is simpler; abort actually *cancels*."
Senior: "Race doesn't cancel the loser — the Redis INCR still lands, you just stop waiting; so a race-timeout on a *write* can double-count. AbortController is the right tool when the underlying API supports it. Note the fetch tool's API path has *no* timeout — inconsistent discipline (a ticket)."
Follow-ups: does your timeout leak the operation? `AbortSignal.any`.

## Q4: Have you used generators/iterators in real code?
Anchor: streaming search tool [search.ts#L24-L40](../../lib/tools/search.ts#L24-L40); dual-shape consumption [search.ts#L203-L216](../../lib/tools/search.ts#L203-L216).
Junior: "They produce values lazily with `yield`."
Mid: "`async function*` lets a tool yield progress before completing — Morphic's search tool yields `{state:'searching'}` immediately, then `{state:'complete', ...results}`, so the UI shows a spinner instantly."
Senior: "The consumer must handle both an async-iterable and a plain promise (the tool can be called either way — [search.ts#L182-L218](../../lib/tools/search.ts#L182-L218)); and a throw between yields still delivers earlier yields, so the UI must tolerate 'searching→error'."
Follow-ups: backpressure with generators; `for await`.

## Q5: `unknown` vs `any` — when each?
Anchor: `any` json columns [schema.ts#L173-L198](../../lib/db/schema.ts#L173-L198); disciplined `unknown` narrowing [create-chat-stream-response.ts#L219-L221](../../lib/streaming/create-chat-stream-response.ts#L219-L221).
Junior: "`any` turns off checking; `unknown` you must narrow."
Mid: "`any` disables the compiler for *everyone downstream* — Morphic's `$type<any>()` json columns push validation onto every reader. `unknown` forces the next person to prove the shape, e.g. `error instanceof Error ? error.message : String(error)`."
Senior: "Accept `any` only at generated/vendor edges and wrap it immediately; the real cost of the `any` columns here is a stringly-typed mapping layer where a mistyped column key compiles (`as keyof DBMessagePart`, [message-mapping.ts#L110-L121](../../lib/utils/message-mapping.ts#L110-L121)) — that's the blast radius of `any` made concrete."
Follow-ups: `never`; assertion functions.

## Q6: How do you type data crossing a trust boundary (HTTP body, external API)?
Anchor: Zod tool schemas [search.tsx](../../lib/schema/search.tsx), [todo.ts#L5-L27](../../lib/tools/todo.ts#L5-L27); the *gap* at [route.ts#L30-L52](../../app/api/chat/route.ts#L30-L52).
Junior: "Define an interface."
Mid: "Interfaces vanish at runtime — you need runtime validation. Zod schemas both validate and infer the type (`z.infer`), one source of truth. Morphic validates all LLM tool inputs with Zod."
Senior: "Interesting inversion here: the *LLM's* output is Zod-validated but the *HTTP body* to `/api/chat` gets only presence checks — equally untrusted, unequally guarded. I'd add a schema there, using `.safeParse` → 400, and non-strict for forward-compat (a too-strict schema rejecting new fields is its own bug)."
Follow-ups: parse-don't-validate; where NOT to validate (internal calls).

## Q7: How do you make LLM tool-calling reliable?
Anchor: tool schema per model [search.ts#L20-L23](../../lib/tools/search.ts#L20-L23), strict for o-models [search.tsx#L81-L94](../../lib/schema/search.tsx#L81-L94); citation contract.
Junior: "Give the model tools and hope."
Mid: "Constrain inputs with schemas so malformed calls are rejected; stream states so the UI reflects progress; and enforce policy server-side, not just in the prompt — Morphic's quick mode *wraps* the search tool to force `type:'optimized'` ([researcher.ts#L20-L63](../../lib/agents/researcher.ts#L20-L63)) rather than trusting the prompt."
Senior: "Some models need strict JSON schemas (OpenAI o-series) — pick the schema at construction time by model id. And the model↔UI contracts (citations, spec blocks) have no compiler between them, so they need end-to-end tests and soft-fail rendering; the prompt requests, code enforces, tests verify."
Follow-ups: constrained decoding vs prompt-format; retries on tool errors.

## Q8: Discriminated unions — real usage?
Anchor: message parts [message-mapping.ts#L347-L666](../../lib/utils/message-mapping.ts#L347-L666), tool_state narrowing.
Junior: "A union with a common tag field."
Mid: "Morphic's message parts are a union on `type`; the mapper switches on it and, for tool parts, further on `state`, so `output` only exists when `state==='output-available'` — invalid states are unrepresentable."
Senior: "That makes bad states uncompilable in the *UI* type, but the *DB* stores it in sparse columns typed `any`, so the guarantee stops at the boundary — you re-earn it with a round-trip test. The 700-line switch is also the cost of this design; a table-driven mapper trades some type-safety for size."
Follow-ups: exhaustiveness checking with `never`.

## Q9: When is a type cast (`as`) acceptable?
Anchor: `wrapSearchToolForQuickMode ... as T` [researcher.ts#L20-L62](../../lib/agents/researcher.ts#L20-L62).
Junior: "When you know better than the compiler."
Mid: "When you've *proven* something the compiler can't see — but each `as` is a skipped proof. Morphic's quick-mode wrapper casts to `T` because it reconstructs a tool with the same schema; it's documented and contained."
Senior: "Acceptable when contained, documented, and tested-around; dangerous when it hides a real mismatch — if the wrapped tool's yield shape drifts, that cast compiles and breaks at runtime. Prefer type guards or generics that make the proof real; reserve `as` for genuine compiler blind spots."
Follow-ups: `as const`; `satisfies` vs `as`.

## Q10: What does your CI run, and why in that order?
Anchor: [.github/workflows/ci.yml](../../.github/workflows/ci.yml).
Junior: "Tests."
Mid: "Lint, typecheck, format-check, and test run in parallel; build depends on all four ([ci.yml#L92-L95](../../.github/workflows/ci.yml#L92-L95)). Cheap fast checks gate the expensive build."
Senior: "Tests run with a *stubbed* `DATABASE_URL` ([L88-L90](../../.github/workflows/ci.yml#L88-L90)), so the RLS integration test is a no-op in CI and only means something locally — a real coverage gap to name. And format-check as a hard gate means you set up format-on-save or waste CI minutes."
Follow-ups: pre-commit hooks; what to gate vs warn.

## Q11: How do you handle conversations longer than the model's context window?
Anchor: prune+truncate [create-chat-stream-response.ts#L142-L159](../../lib/streaming/create-chat-stream-response.ts#L142-L159), [context-window.ts#L189-L285](../../lib/utils/context-window.ts#L189-L285).
Junior: "Cut old messages."
Mid: "Two tiers: cheap structural pruning (drop old reasoning/tool calls) then token-counted truncation that keeps the first user message and the recent tail. Morphic does exactly this, capping the first message at 30% of budget."
Senior: "The token counts are approximate (wrong tokenizer for non-OpenAI models) and the model→window table is *stale* — current default models fall to a 16k default and over-truncate silently. The real fixes are per-model windows and a metric on truncation events. Also: summarize-old-turns is the next rung beyond drop-old."
Follow-ups: summarization; RAG over history instead of truncation.

## Q12: Node vs browser vs edge — what runs where in a Next.js app?
Anchor: server [route.ts](../../app/api/chat/route.ts); client [chat.tsx#L1](../../components/chat.tsx#L1); middleware [proxy.ts](../../proxy.ts).
Junior: "Server and client."
Mid: "Server components/routes/actions run in Node — they read secrets and hit Postgres. `'use client'` = browser, only `NEXT_PUBLIC_*` env. Middleware runs per-request in a constrained runtime — Morphic keeps it to header/cookie work."
Senior: "The dangerous confusion is a secret imported into a client component — it either leaks or fails to bundle. And server *actions* are public POST endpoints, so 'it's server code' doesn't mean 'trusted caller' — every Morphic action re-authenticates ([lib/actions/chat.ts#L39-L45](../../lib/actions/chat.ts#L39-L45))."
Follow-ups: what's a server action really; RSC serialization boundary.

---

Interview angle recap: for each, the mid answer earns the offer by pairing a definition with *this repo's* concrete example and one tradeoff. Rehearse the anchors until you can cite file names without notes.
