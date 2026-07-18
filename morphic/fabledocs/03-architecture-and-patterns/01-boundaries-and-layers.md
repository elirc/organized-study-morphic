# Boundaries and Layers

A **layer** owns a concern; a **boundary** is the contract between layers. Layers exist so a change in one place (swap Tavily for Brave) doesn't ripple everywhere. The test of a boundary: *can you describe what crosses it in one sentence?*

## The stack, top to bottom

```
Browser (components/, hooks/)          — render, local state, optimistic UX
   │  HTTP: JSON bodies, SSE streams
API routes (app/api/)                  — parse, authenticate, gate, status codes
   │  typed function calls
Orchestration (lib/streaming/)         — sequence the work: prep → agent → persist
   │
Domain services (lib/agents, lib/tools, lib/rate-limit, lib/config)
   │
Application actions (lib/actions/)     — auth-aware, cache-aware use-cases
   │
Persistence (lib/db/)                  — schema, RLS, SQL
   │
Postgres + Redis + object storage + vendor APIs
```

## What each layer owns (and must not own)

| Layer | Owns | Must not own | Evidence of discipline |
| --- | --- | --- | --- |
| API route | status codes, header/cookie reads, gate *ordering* | SQL, prompts | [route.ts](../../app/api/chat/route.ts) contains zero SQL, zero prompt text |
| Orchestration | the sequence + failure policy | vendor details | [create-chat-stream-response.ts](../../lib/streaming/create-chat-stream-response.ts) imports actions/agents, never `drizzle-orm` |
| Tools | vendor normalization | UI shapes | search tool returns `SearchResults`, not component props |
| lib/actions | auth + cache lifecycle | raw SQL | [lib/actions/chat.ts](../../lib/actions/chat.ts) delegates every query to `dbActions` |
| lib/db | SQL + RLS | HTTP anything | [db/actions.ts](../../lib/db/actions.ts) returns rows/counts, never `Response` |

## Good boundaries worth imitating

1. **`SearchProvider`** — five vendors, one signature ([providers/index.ts#L16-L37](../../lib/tools/search/providers/index.ts#L16-L37)). Adding Firecrawl touched the factory + one new file, nothing above.
2. **`withRLS`** — "give me a userId and a callback; I guarantee isolation" ([with-rls.ts#L39-L70](../../lib/db/with-rls.ts#L39-L70)). Callers can't forget to clear the setting because transaction scope does it.
3. **The two-actions split** — [lib/actions/chat.ts](../../lib/actions/chat.ts) (checks `getCurrentUserId`) vs [lib/db/actions.ts](../../lib/db/actions.ts) (takes `userId` as a parameter). Auth decisions live in exactly one layer; persistence is testable without mocking auth.
4. **The render catalog** — the LLM's capability boundary ([catalog.ts#L7-L70](../../lib/render/catalog.ts#L7-L70)): what crosses it is "a tree of these five components with these validated props", nothing else.

## Boundary leaks (real, learn to spot the smell)

1. **Vendor conditional above the adapter:** `if (searchAPI === 'brave')` in the tool layer ([search.ts#L114-L127](../../lib/tools/search.ts#L114-L127)) — the adapter interface couldn't express Brave's options, so vendor knowledge leaked up. Fix: options-object parameter on the interface. Smell name: *switch-on-type above an abstraction*.
2. **SearXNG special-casing, twice:** the tool checks `searchAPI === 'searxng' && depth === 'advanced'` and calls its own HTTP route ([search.ts#L84-L108](../../lib/tools/search.ts#L84-L108)) — provider selection logic *and* transport logic in the caller. The provider should decide its own depth handling.
3. **HTTP status shapes produced in lib/:** rate limiters return `Response` objects ([guest-limit.ts#L81-L109](../../lib/rate-limit/guest-limit.ts#L81-L109)) — persistence-layer-adjacent code owning HTTP. Convenient, but now you can't reuse the limiter in a non-HTTP context (CLI, cron) without faking responses. Alternative: return a result object; let the route build the `Response`.
4. **Client-computed server facts:** `isNewChat` decided by the browser ([chat.tsx#L123-L126](../../components/chat.tsx#L123-L126)) and trusted by the server for write-path branching ([prepare-messages.ts#L92](../../lib/streaming/helpers/prepare-messages.ts#L92)). It's *safe* here (RLS + duplicate-key tolerance) but it moves an invariant ("is this chat new?") to the least trustworthy layer — a smell to name in review even when harmless.
5. **`getBaseUrlString()` inside a tool** ([search.ts#L90](../../lib/tools/search.ts#L90)) — a domain tool reaching for HTTP-request context (headers) to call *itself*. Layering would say: inject the advanced-search capability, don't have the tool rediscover the network.

## Ownership rule of thumb

When you can't decide where code goes, ask: **who must change when this changes?** If a new search vendor forces edits above `lib/tools/search/providers/`, the boundary failed. If a new HTTP status code forces edits below `app/api/`, same.

## Drill

Take leak #3. Sketch the refactor: `checkGuestLimit(ip): LimitResult`, route maps `LimitResult → Response`. List every file touched and what test you'd write first. Self-grade — Basic: the two files. Solid: notices all three limiters should share the mapping. Strong: keeps the exact 401-vs-429 + header behavior pinned by a test *before* refactoring (characterization test).

## Interview angle

- "Describe the layers of an app you know." — recite this stack with the two-actions split as the highlight. ([Q22](../08-interview-prep/03-api-and-data-modeling-questions.md))
- "Tell me about a leaky abstraction you've seen." — the Brave conditional, plus the fix. Naming *both the smell and the fix* is what makes it a mid-level answer.
- "Where should validation/authorization live?" — next file: [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md).
