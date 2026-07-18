# Review Katas

Ten fake PRs. For each: read the author's intent + diff summary, then write findings graded **Blocking** (must fix before merge), **Important** (should fix), **Optional** (nit/preference). Compare with the expected findings. Practice *kind, specific* language — the model comments show the register.

Grading: name the finding *and* its severity. Strong reviewers also propose the fix and cite the pattern the code should follow.

---

## Kata 1: "Add a delete-all-messages endpoint"

Author intent: let users clear a chat's messages.
Fake diff summary: new route `DELETE /api/chats/[id]/messages` calling `db.delete(messages).where(eq(messages.chatId, id))` directly.
Files this resembles: [db/actions.ts#L240-L270](../../lib/db/actions.ts#L240-L270).
Expected findings — **Blocking:** no auth and no RLS wrapper — uses bare `db`, so it deletes *any* chat's messages by id (IDOR + cross-tenant). Must go through `withRLS(userId, …)` and verify ownership like [deleteMessagesFromIndex](../../lib/actions/chat.ts#L287-L312). **Important:** no `revalidateTag` → stale sidebar. **Optional:** return a count for symmetry with existing delete actions.
Model comment:
> Blocking: this runs on the raw `db` connection, so RLS never engages and any authenticated user can delete another user's messages by guessing the chat id. Wrap it in `withRLS(userId, …)` and add the same ownership check `deleteMessagesFromIndex` uses. Happy to pair on it.

---

## Kata 2: "Speed up model list by caching forever"

Author intent: stop refetching vendor models.
Fake diff summary: changes `MODEL_CACHE_TTL_MS` from 2 min to `Infinity`.
Files: [fetch-models.ts#L8](../../lib/models/fetch-models.ts#L8).
Expected — **Important:** infinite TTL means newly-enabled providers/models never appear until redeploy; the 2-min TTL is a deliberate freshness/latency balance. **Optional:** if the goal is fewer fan-outs, add single-flight de-duplication instead. **Not blocking** (per-instance cache, no correctness risk).
Model comment:
> The 2-minute TTL is doing real work — it's how a newly-added API key shows up without a redeploy. If the concern is fan-out cost, single-flight (dedupe concurrent misses) fixes that without freezing the list. Want me to sketch it?

---

## Kata 3: "Validate chat body with Zod"

Author intent: add input validation to `/api/chat`.
Fake diff summary: adds `bodySchema.parse(body)` that throws on unknown keys (`.strict()`).
Files: [route.ts#L30-L52](../../app/api/chat/route.ts#L30-L52).
Expected — **Blocking:** `.strict()` + throwing = a client sending a newer optional field gets a 500; also an uncaught `ZodError` becomes the generic 500 at [L222-L227](../../app/api/chat/route.ts#L222-L227), losing the 400. Use `.safeParse`, return 400 with issues, and prefer `.passthrough()`/non-strict for forward-compat. **Important:** validate but don't over-constrain `message.parts` (reuse the SDK's UIMessage shape). **Optional:** share the schema with the client transport.
This kata teaches: *good idea, dangerous defaults.* Validation that's stricter than the contract is its own bug.

---

## Kata 4: "Add retry to the search tool"

Author intent: make search resilient.
Fake diff summary: wraps the vendor `fetch` in a 3× retry loop, no backoff, no timeout.
Files: [tavily.ts#L32-L55](../../lib/tools/search/providers/tavily.ts#L32-L55).
Expected — **Important:** retry without a timeout can *lengthen* a hang to 3×; add a per-attempt AbortController first (the tool has no timeout today). **Important:** retry only idempotent/safe failures (network, 5xx), not 4xx (bad key → fail fast). **Optional:** exponential backoff + jitter. Note: search is a *read*, so retry is safe *if* bounded.
Model comment:
> Retries are right for a read like search, but without a per-attempt timeout this could turn one 30s hang into 90s. Add an AbortController (steal the pattern from `fetchRegularData`), and skip retry on 4xx — a bad API key won't fix itself.

---

## Kata 5: "Store feedback score directly from client"

Author intent: record thumbs up/down.
Fake diff summary: client POSTs `{messageId, score}`; route updates message metadata with no auth check.
Files: [feedback/route.ts#L48-L68](../../app/api/feedback/route.ts#L48-L68), [actions/feedback.ts#L19](../../lib/actions/feedback.ts#L19).
Expected — **Blocking (or Important, argue it):** any client can set feedback on *any* messageId, and with Supabase unconfigured the write runs with no RLS. Verify ownership (RLS-scoped read of the message's chat) before writing. This mirrors a *real* open question in the repo — see [risk-register.md](../09-reference/risk-register.md). **Important:** score isn't range-checked in the action (route checks {1,-1}, action trusts). 
Teaches: reviewing *matching existing (imperfect) code* — "the current endpoint does this too" is not a merge argument.

---

## Kata 6: "Simplify RLS by using the owner connection"

Author intent: fix "RLS is slow" by connecting as the DB owner.
Fake diff summary: sets `DATABASE_URL` to the owner role and drops `DATABASE_RESTRICTED_URL`.
Files: [db/index.ts#L25-L28](../../lib/db/index.ts#L25-L28).
Expected — **Blocking:** table owners **bypass RLS entirely** — this silently disables all tenant isolation while every policy still *looks* present. The startup check ([L82-L91](../../lib/db/index.ts#L82-L91)) would warn, but nobody reads warnings. This is a security regression disguised as a perf fix. **Alternative:** index the policy subqueries (already done, [schema.ts#L50-L51](../../lib/db/schema.ts#L50-L51)); measure before assuming RLS is the bottleneck.
Teaches: the most dangerous PRs *look* like cleanups.

---

## Kata 7: "Persist message before streaming for reliability"

Author intent: don't lose messages if the stream dies.
Fake diff summary: `await upsertMessage(...)` before `agent.stream(...)` on the new-chat path, removing the optimistic promise.
Files: [prepare-messages.ts#L91-L116](../../lib/streaming/helpers/prepare-messages.ts#L91-L116).
Expected — **Important:** this trades latency (user now waits for a DB round-trip before the model starts) for a reliability gain that the existing code *already provides* via awaiting the pending promise in `onFinish`. Measure the latency cost; the current design is intentional. **Optional:** if durability is the real goal, the outbox (architecture-critique #5) is the right lever, not blocking the stream.
Teaches: "more reliable" isn't free; name the cost.

---

## Kata 8: "Add a Notion search provider"

Author intent: new provider.
Fake diff summary: new `NotionProvider` class; adds it to the factory; changes the tool's return type to include Notion-specific fields.
Files: [providers/index.ts](../../lib/tools/search/providers/index.ts), [search.ts](../../lib/tools/search.ts).
Expected — **Blocking:** widening the shared `SearchResults` type for one provider leaks vendor specifics into every consumer (the exact leak the Brave case already shows — [search.ts#L114-L127](../../lib/tools/search.ts#L114-L127)). Keep the normalized contract; map Notion fields into it. **Important:** add to the `SearchProviderType` union + `.env.local.example` + docs. **Optional:** a test — providers currently have none.
Teaches: defending an abstraction's contract under feature pressure.

---

## Kata 9: "Fix flaky title test by increasing timeout"

Author intent: a title-generation test intermittently fails.
Fake diff summary: bumps the test timeout from 5s to 30s.
Files: pattern of [title-generator.ts](../../lib/agents/title-generator.ts).
Expected — **Important:** raising a timeout hides flake, doesn't fix it. A title test that calls a real model is non-deterministic and slow — mock `generateText` and assert the *fallback* and *quote-stripping* logic ([title-generator.ts#L51-L60](../../lib/agents/title-generator.ts#L51-L60)) deterministically. **Optional:** if it must hit a model, mark it e2e and exclude from CI (CI has no keys). 
Teaches: flake is a design signal, not a timeout problem.

---

## Kata 10: "Render model output as HTML for rich formatting"

Author intent: prettier answers.
Fake diff summary: replaces the spec-block renderer with `dangerouslySetInnerHTML={{__html: marked(content)}}`.
Files: contrast with [lib/render/](../../lib/render/catalog.ts), [render-message.tsx](../../components/render-message.tsx).
Expected — **Blocking:** this is XSS — LLM output (influenced by fetched web pages!) rendered as raw HTML. The whole spec-catalog design exists to give rich UI *without* this. If Markdown rendering is the goal, use the sanitizing markdown pipeline already in use (streamdown), never raw HTML. 
Teaches: recognizing a security regression that's framed as a UX improvement — the highest-value review skill.

---

Meta-drill: which katas are "good idea, wrong defaults" (3, 4, 7, 9) vs "wrong idea" (1, 6, 10) vs "matches bad precedent" (5, 8)? Sorting them trains the judgment interviewers probe in code-review rounds. Timed version in [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).
