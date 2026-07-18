# Architecture Critique

The senior exercise: praise precisely, criticize with evidence, prioritize by risk×effort. This file doubles as system-design interview material — cross-linked from [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices (steal these)

1. **RLS as the authorization floor** with app checks above it and a restricted DB role beneath it ([schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67), [with-rls.ts](../../lib/db/with-rls.ts), [db/index.ts#L25-L28](../../lib/db/index.ts#L25-L28)). Even a badly-written new feature can't cross tenants. The composite index for policy subqueries ([schema.ts#L50-L51](../../lib/db/schema.ts#L50-L51)) shows the team paid the *whole* cost, not just the headline.
2. **Client-generated ids enabling optimistic persistence** ([chat.tsx#L58](../../components/chat.tsx#L58) → [prepare-messages.ts#L91-L116](../../lib/streaming/helpers/prepare-messages.ts#L91-L116) → duplicate-key-as-success). Latency win with a real consistency argument behind it.
3. **The capability-bounded generative UI** ([catalog.ts](../../lib/render/catalog.ts) + incremental parser [parse-spec-block.ts#L53-L92](../../lib/render/parse-spec-block.ts#L53-L92)): the LLM gets expressive power without HTML injection, streams render progressively, and legacy specs migrate ([L94-L99](../../lib/render/parse-spec-block.ts#L94-L99)). Best-tested corner of the repo, appropriately.
4. **Deployment-mode guardrails**: `ENABLE_AUTH=false` hard-blocked on cloud ([get-current-user.ts#L25-L29](../../lib/auth/get-current-user.ts#L25-L29)). Encoding "this config combination must never happen" as a throw is cheap and priceless.
5. **Provider plugin seams on both axes** (search vendors, model vendors) that have already absorbed growth: five search providers, six model providers, without upper-layer rewrites.

## Risks and tradeoffs (with evidence)

Confirmed by reading (severity-ordered):

1. **No durability for in-flight effects.** All persistence hangs off a live stream's `onFinish` ([create-chat-stream-response.ts#L196-L218](../../lib/streaming/create-chat-stream-response.ts#L196-L218)); instance death = silent loss. Acceptable for a search app; unacceptable the day this handles anything billable.
2. **Destructive regeneration without compensation** ([prepare-messages.ts#L57-L78](../../lib/streaming/helpers/prepare-messages.ts#L57-L78)) — the one flow where a *failure loses user content that existed*.
3. **Rate limiting is advisory in practice**: fail-open, cloud-gated, spoofable guest identity ([guest-limit.ts#L34-L42](../../lib/rate-limit/guest-limit.ts#L34-L42), [route.ts#L77-L82](../../app/api/chat/route.ts#L77-L82)); triplicated code drifting apart.
4. **Stale context-window table** ([context-window.ts#L12-L38](../../lib/utils/context-window.ts#L12-L38)): current default models (`gpt-5.4-mini` [default-model.ts#L3-L8](../../lib/config/default-model.ts#L3-L8), Gemini 3.x [cloud.json](../../config/models/cloud.json)) aren't listed → 16k fallback → likely over-truncation on long chats. Quietly degrades quality; nobody alerts.
5. **The 775-line mapping file** ([message-mapping.ts](../../lib/utils/message-mapping.ts)) with stringly-typed column access — every new tool grows it; `any`-typed json columns push validation to readers.
6. **`/api/chat` body unvalidated beyond presence checks** ([route.ts#L30-L52](../../app/api/chat/route.ts#L30-L52)) while everything the *LLM* sends gets Zod — inverted trust.

Hypotheses (*investigate* before acting):

- Feedback endpoint spoofing (unauthenticated `messageId`/`traceId` writes — [feedback/route.ts#L48-L68](../../app/api/feedback/route.ts#L48-L68)).
- Fetch-tool SSRF reach (server fetches arbitrary URLs, no private-range denylist — [fetch.ts#L14-L21](../../lib/tools/fetch.ts#L14-L21)); real exposure depends on network posture.
- `upsertMessage` conflict path dropping metadata updates ([db/actions.ts#L104-L107](../../lib/db/actions.ts#L104-L107)).
- Unbounded `/api/chats?limit=` ([chats/route.ts#L13-L14](../../app/api/chats/route.ts#L13-L14)).

## Prioritized improvements ("owning this for 3 months")

| # | Change | Why first | Migration path | Test strategy |
| --- | --- | --- | --- | --- |
| 1 | Zod-validate `/api/chat` + `/api/chats` inputs (clamp limit ≤50) | small, kills a class of bugs, zero product change | additive; reject-with-400 behind a log-only week if cautious | unit: schema table-tests; characterization test of current accepted bodies first |
| 2 | Regeneration: delete *after* successful regeneration (or soft-delete + sweep) | only user-data-loss flow | reorder inside [prepare-messages.ts](../../lib/streaming/helpers/prepare-messages.ts); keep ids stable for the UI | integration: fail the LLM call, assert old answer survives |
| 3 | Context-window config: per-model from the models source (or generous default + log) | silent quality degradation | data-only change in [context-window.ts](../../lib/utils/context-window.ts) | unit: budget math per model; log-assert unknown-model path |
| 4 | Unify rate limiters into one `checkDailyLimit(scope, id, limit)` returning a result object | 3× drift already visible | introduce shared core; adapt three callers one PR each | port existing tests; add INCR/EXPIRE atomicity via Lua |
| 5 | Outbox (or platform queue) for persistence | durability ceiling-raiser; do it before features that require it | outbox table + same-transaction write + drainer; dual-write period | kill-the-worker chaos test; duplicate-delivery idempotency test |
| 6 | Feedback endpoint: require auth (or verify message ownership under RLS) | possible integrity hole | tighten; anonymous deployments keep working via ownership check rather than identity | authz matrix test |

The ordering logic to say out loud: **correctness-of-inputs → user-data-loss → silent-quality → maintainability → durability → integrity-hardening**, adjusted for effort. Reasonable people reorder 5 and 6; being able to defend an ordering matters more than the ordering.

## What I would *not* change

- The wide parts table — costs are real but paid; jsonb rewrite risk isn't justified by pain yet.
- Fail-open limits — right default for this product; make it a config knob instead.
- No queue *today* — see #5; introduce with the first feature that needs it, not before. Premature outboxes rot.

Deferring is an architectural decision with reasons — interviewers score it.

## Drill

Write your own top-3 with different priorities and defend them in 5 sentences each against this list. Self-grade — Strong: at least one disagreement with this file, argued from evidence (anchors), not taste.
