# Side Effects, Async, and Reliability

A **side effect** is anything a function does beyond returning a value: writes, network calls, cache invalidations, logs. Reliability engineering is deciding, per effect: *can it fail? can it run twice? who notices?*

## The side-effect map

| Effect | Where triggered | Failure policy | Idempotent? |
| --- | --- | --- | --- |
| Chat/message/parts writes | [persist-stream-results.ts](../../lib/streaming/helpers/persist-stream-results.ts) | retry ([retry.ts](../../lib/utils/retry.ts) via [L89-L93](../../lib/streaming/helpers/persist-stream-results.ts#L89-L93)); duplicate-key = success ([L54-L68](../../lib/streaming/helpers/persist-stream-results.ts#L54-L68)); final failure = **log and drop** ([L94-L97](../../lib/streaming/helpers/persist-stream-results.ts#L94-L97)) | yes — client-generated ids + unique constraints |
| Destructive deletes (regenerate/edit) | [prepare-messages.ts#L57-L78](../../lib/streaming/helpers/prepare-messages.ts#L57-L78) | none — no compensation if the follow-up stream fails | re-running deletes nothing new (idempotent), but the *pairing* with regeneration is not atomic |
| Vendor search/fetch HTTP | [search.ts#L84-L142](../../lib/tools/search.ts#L84-L142), [fetch.ts](../../lib/tools/fetch.ts) | throw → AI SDK surfaces tool error state; fetch has a 10s abort ([fetch.ts#L11-L23](../../lib/tools/fetch.ts#L11-L23)); search has **no explicit timeout** (*investigate* — a hung vendor holds a stream step) | safe to retry (reads) |
| Redis INCR (rate limits) | [route.ts#L83](../../app/api/chat/route.ts#L83), [L137-L143](../../app/api/chat/route.ts#L136-L143) | 3s race-timeout; fail-open | INCR is atomic; INCR+EXPIRE pair is not |
| LLM calls (research, title) | [create-chat-stream-response.ts#L175-L184](../../lib/streaming/create-chat-stream-response.ts#L175-L184), [title-generator.ts](../../lib/agents/title-generator.ts) | research: error → stream error message ([L219-L221](../../lib/streaming/create-chat-stream-response.ts#L219-L221)); title: fallback string ([title-generator.ts#L61-L74](../../lib/agents/title-generator.ts#L61-L74)) | not idempotent (nondeterministic) but replaceable |
| Cache invalidation | `revalidateTag` after every mutation ([lib/actions/chat.ts#L92](../../lib/actions/chat.ts#L92) etc.) | none needed (worst case: 60s staleness) | yes |
| Analytics | fire-and-forget IIFE ([route.ts#L173-L206](../../app/api/chat/route.ts#L173-L206)); adaptive-limit events ([adaptive-limit.ts#L125-L132](../../lib/rate-limit/adaptive-limit.ts#L125-L132), `void`-ed) | swallow + log | n/a (best-effort) |
| Langfuse traces | [create-chat-stream-response.ts#L79-L94](../../lib/streaming/create-chat-stream-response.ts#L79-L94), flush in `finally` ([L213-L217](../../lib/streaming/create-chat-stream-response.ts#L213-L217)) | flush even on error paths ([L224-L227](../../lib/streaming/create-chat-stream-response.ts#L224-L227)) | n/a |
| R2 object write | [upload/route.ts#L84-L94](../../app/api/upload/route.ts#L84-L94) | error → 500; orphan objects possible if message never sent | re-upload creates a new key (timestamped) — not idempotent, harmless |

No queues, no cron, no webhooks in this repo — "background work" is *promise-shaped and request-scoped*. That's the headline architectural constraint: **if the serverless instance dies mid-stream, in-flight effects are lost silently.** (No outbox; nothing re-drives persistence.) Say this in the architecture-critique interview — then offer the upgrade path: durable queue or an outbox table written in the same transaction as the user's message.

## The reliability vocabulary, grounded

- **Idempotency**: run-twice-safe. Achieved here via client-generated ids + unique constraints + duplicate-key-as-success. Anchor: [persist-stream-results.ts#L54-L68](../../lib/streaming/helpers/persist-stream-results.ts#L54-L68).
- **Retry**: only around the final message save ([retryDatabaseOperation](../../lib/utils/retry.ts) at [persist-stream-results.ts#L89-L93](../../lib/streaming/helpers/persist-stream-results.ts#L89-L93)). Retrying is only safe *because* the write is idempotent — always say these two words together.
- **Timeout**: 10s fetch abort; 3s Redis race. Missing on: vendor search calls, Jina/Tavily-extract calls ([fetch.ts#L90-L133](../../lib/tools/fetch.ts#L90-L133) — no controller!). Inconsistent timeout discipline is one of the repo's honest weaknesses.
- **Backpressure**: `smoothStream` word-chunking ([create-chat-stream-response.ts#L182](../../lib/streaming/create-chat-stream-response.ts#L182)) + client render throttle ([chat.tsx#L215](../../components/chat.tsx#L215)) shape the flow; `consumeStream()` ([L184](../../lib/streaming/create-chat-stream-response.ts#L184)) deliberately *removes* backpressure from the client so persistence completes for abandoned tabs — a considered tradeoff (server pays for tokens nobody reads).
- **Compensation**: absent where it matters most (regeneration deletes before the replacement exists — Flow 5). The interview-grade fix: reorder (delete after success) or soft-delete.
- **Failure visibility**: every swallow is a `console.error`/`console.warn` — adequate for self-host, invisible at scale. "How would I know this broke?" per flow lives in [05-quality-engineering/06-observability-and-operations.md](../05-quality-engineering/06-observability-and-operations.md).

## Where effects sit in risky places (flagged)

1. **Effects inside `onFinish` of a stream** ([create-chat-stream-response.ts#L196-L218](../../lib/streaming/create-chat-stream-response.ts#L196-L218)) — correct for latency, but the process must survive to run it. Serverless timeout (`maxDuration: 300`, [route.ts#L18](../../app/api/chat/route.ts#L18)) is the real deadline for the *whole* pipeline including persistence.
2. **Analytics loads the full chat to count turns** ([route.ts#L179-L186](../../app/api/chat/route.ts#L179-L186)) — a DB read on every message *just for analytics*, post-response but pre-completion of the same instance. Cheap now; a counter column later.
3. **Delete-then-regenerate** (Flow 5) — the only place user data can be *lost* by design.
4. **EXPIRE after INCR** — the immortal-key edge (Pattern 5).

## Drills

1. Build the failure tree for "user sends message, instance killed at T": T ∈ {before insert promise resolves, during stream, inside onFinish before message save, after save before title}. For each: what does the user see now / after refresh? Verify against [persist-stream-results.ts](../../lib/streaming/helpers/persist-stream-results.ts). Self-grade — Strong: you identify the exact case producing an "Untitled" chat with the user message but no answer.
2. Design the outbox upgrade: table schema, what's written in the same transaction as the user message, who drains it, and what "at-least-once + idempotent consumer" means here. (Fake code, labeled.)
3. Add timeouts on paper to [fetchJinaReaderData](../../lib/tools/fetch.ts#L90-L121): copy the AbortController pattern from `fetchRegularData` — which constants would you share?

## Interview angle

- "How do you make writes safe to retry?" — idempotency via client ids; anchor and recite. ([Q31](../08-interview-prep/03-api-and-data-modeling-questions.md))
- "What happens if your server dies mid-request?" — the failure tree from drill 1 *is* the answer; then the outbox as the next rung. (Q36)
- "Design the async parts of a chat app" — queues vs request-scoped promises; know why Morphic gets away without queues (single-request effects, tolerable loss) and when that stops (billing, notifications, long jobs).
