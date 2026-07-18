# Performance Thinking

Rule zero: **measure first.** This repo ships the instruments — `perfLog`/`perfTime` behind `ENABLE_PERF_LOGGING` ([route.ts#L25-L27](../../app/api/chat/route.ts#L25-L27), timings sprinkled through [create-chat-stream-response.ts](../../lib/streaming/create-chat-stream-response.ts)) and Langfuse traces for LLM latency. Never optimize what you haven't timed.

## The performance domains here

| Domain | Where it lives | Likely hotspots |
| --- | --- | --- |
| LLM latency | agent stream | dominates everything; measured via traces |
| Server DB | lib/db | RLS subqueries, N+1 in `clearChats`, analytics chat-load |
| Network (vendor) | tools/providers, fetch | no timeout on search; sequential vendor model fetches |
| Render | components | per-token re-renders (throttled), part-fold |
| Bundle | client components | radix + streamdown + katex weight |
| Cache/memory | module caches | model cache, encoder cache, per-request Redis clients |

## Where to look first (anchored)

1. **N+1 delete loop** — [lib/actions/chat.ts#L229-L233](../../lib/actions/chat.ts#L229-L233): `clearChats` runs one transaction per chat, each re-checking ownership. O(chats) round-trips. Fix: single RLS-scoped `delete where userId` (cascades handle the rest). *Measure* first: it's invisible at 10 chats.
2. **Analytics loads the full chat per message** — [route.ts#L179-L186](../../app/api/chat/route.ts#L179-L186) loads all messages just to count the conversation turn, on every send. Fix: a counter column or derive from the metadata you already have.
3. **Serial vendor model fetches with no timeout** — [fetch-models.ts#L392-L398](../../lib/models/fetch-models.ts#L392-L398) is `Promise.all` (parallel, good) but each `fetchJson` has no timeout ([L150-L159](../../lib/models/fetch-models.ts#L150-L159)); one slow vendor stalls model selection, which sits on the chat critical path when no cookie/default resolves ([model-selection.ts#L140](../../lib/utils/model-selection.ts#L140)). Fix: per-fetch timeout; also single-flight the cache.
4. **Synchronous HTML processing** — [fetch.ts#L48-L67](../../lib/tools/fetch.ts#L48-L67) runs several regexes over up-to-50k-char strings on the request thread. Usually fine; a pathological page is a CPU spike. Fix: cap earlier, or offload.
5. **Expensive renders** — mitigated already: `useMemo` for sections ([chat.tsx#L224-L252](../../components/chat.tsx#L224-L252)), `experimental_throttle: 100` caps re-renders to ~10/s ([chat.tsx#L215](../../components/chat.tsx#L215)). Without the throttle, a fast stream re-renders the whole tree per token.
6. **Missing DB indexes?** — actually a *strength* here: every hot query has an index ([schema.ts#L43-L51](../../lib/db/schema.ts#L43-L51)), including one purpose-built for the RLS subquery. The lesson to internalize: RLS policies run a subquery per row-check; index it or pay.

## Finding each class of problem

- **N+1:** enable Drizzle query logging, count queries per request; a loop calling a per-item action is the tell (grep for `for (… of …) await`).
- **Serial async that should be parallel:** look for sequential `await`s with no data dependency (`clearChats`); confirm with timing.
- **Expensive renders:** React DevTools Profiler; look for components re-rendering on every token.
- **Oversized bundle:** `next build` output / bundle analyzer; katex + carousel + radix are the usual suspects — lazy-load heavy, rarely-used pieces (math rendering, video dialog).
- **Unbounded queries:** `/api/chats?limit=` isn't clamped ([chats/route.ts#L13-L14](../../app/api/chats/route.ts#L13-L14)) — a large limit is a self-inflicted slow query. Clamp it.

## The one number that dominates

LLM streaming time (seconds) dwarfs every DB/render microoptimization (milliseconds). So the highest-leverage perf work is: (a) start the stream ASAP — which is exactly why chat creation is optimistic and off the critical path (Flow 1); (b) don't add synchronous work *before* `agent.stream`. When reviewing a PR that adds an `await` before streaming (Kata 7), the perf lens says "you just added that latency to every message."

## Interview angle

- "How would you find a performance problem?" → measure-first, then the domain checklist; cite the built-in perf logging. ([Q41](../08-interview-prep/03-api-and-data-modeling-questions.md) area)
- "What's an N+1 and where would you look?" → `clearChats` loop, with the single-statement fix. (Q28 follow-up)
- "This endpoint got slow — walk me through it." → binary-search the domains (DB? vendor? render?), confirm with a timing probe before touching code.

Drill: instrument `clearChats` (add `perfTime` around the loop, conceptually), predict the shape of the curve as chat count grows, and write the one-line SQL replacement. Self-grade — Strong: you also note the replacement changes error semantics (no per-chat ownership message) and add a test pinning the new behavior.
