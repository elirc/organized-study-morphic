# System Design From This Repo

Reverse-engineer Morphic into a whiteboard exercise. Prompt: **"Design an AI answer engine (Perplexity-style): a user asks a question, and the system searches the web and streams back a cited answer, with saved history."**

For each step: what a junior/mid/senior answer sounds like, then what Morphic *actually* chose (with anchors) and a stronger/simpler alternative. Practice by whiteboarding it cold first, then reading.

---

## Step 1: Requirements & scope

- Junior: jumps to "React frontend, Node backend, database."
- Mid: clarifies first — auth or anonymous? persist history? which search provider? real-time streaming? expected scale? read/write ratio (this is read-heavy: many history loads, fewer sends).
- Senior: also surfaces the *hard* requirements hiding in "cited answer" (a machine-readable link between claim and source) and "streamed" (persistence can't block the response) — the two constraints that shape everything.

Morphic's answers: optional auth + anonymous/guest modes; PostgreSQL history; pluggable providers; SSE streaming.

## Step 2: API surface

- Junior: many endpoints (`/send`, `/getMessages`, `/search`…).
- Mid: one streaming `POST /api/chat` for the conversation + a few REST endpoints (`GET /api/chats`, `/api/upload`, `/api/feedback`). Body carries `trigger` (submit vs regenerate) so one endpoint handles both.
- Senior: notes the endpoint returns a *stream*, so status/errors are awkward (you can't 500 mid-stream cleanly) — a real design tension.

Morphic: exactly this — [route.ts](../../app/api/chat/route.ts), body shape at [L30-L31](../../app/api/chat/route.ts#L30-L31). Alternative: separate "start" + "subscribe" endpoints (cleaner errors, more round-trips).

## Step 3: The agent / search loop

- Junior: "call the LLM with the question."
- Mid: a tool-loop — the model calls a search tool, reads results, maybe fetches pages, then writes the answer; cap the loop (step limit) so it can't run forever.
- Senior: two modes (fast vs thorough) with different step budgets and tools; force cheap defaults server-side rather than trusting the prompt; make search a pluggable provider.

Morphic: `ToolLoopAgent`, `stopWhen: stepCountIs(20|50)`, quick vs adaptive modes ([researcher.ts#L92-L146](../../lib/agents/researcher.ts#L92-L146)); quick mode wraps the tool to force `optimized` search. Alternative: a fixed pipeline (search→answer, no loop) — cheaper, less capable.

## Step 4: Citations (the sneaky-hard requirement)

- Junior: "the model includes URLs."
- Mid: the search tool returns numbered results; the model cites `[1]`; the UI maps numbers back to URLs.
- Senior: this is a *contract* across three parties — prompt format, tool output (a citation map keyed by toolCallId), UI parser — with nothing type-checking it, so it needs soft-fail rendering and end-to-end tests.

Morphic: `[n](#toolCallId)` format ([search-mode-prompts.ts#L72-L101](../../lib/agents/prompts/search-mode-prompts.ts#L72-L101)), map built in the tool ([search.ts#L145-L151](../../lib/tools/search.ts#L145-L151)), regex-resolved with invalid citations dropped ([citation.ts#L82-L109](../../lib/utils/citation.ts#L82-L109)). Stronger alternative: structured citation parts instead of in-text markers — type-safe, but changes the streaming/render model.

## Step 5: Data model & persistence

- Junior: "a messages table."
- Mid: chats → messages → parts (for mixed content), cascade deletes, indexes on the hot queries.
- Senior: parts as a polymorphic table with DB constraints; ids client-generated to enable optimistic writes; persist *after* the stream so latency isn't taxed.

Morphic: [schema.ts#L26-L256](../../lib/db/schema.ts#L26-L256), optimistic create ([prepare-messages.ts#L91-L116](../../lib/streaming/helpers/prepare-messages.ts#L91-L116)). Alternative: jsonb content column (fewer migrations, weaker constraints).

## Step 6: Auth & isolation

- Junior: "check a token."
- Mid: sessions via Supabase; each user sees only their chats; public chats shareable.
- Senior: enforce isolation in the *database* (RLS) as well as the app, using a restricted DB role so owners can't bypass it; support anonymous mode but block it in managed deployments.

Morphic: RLS ([schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67), [with-rls.ts](../../lib/db/with-rls.ts)), restricted-user preference ([db/index.ts#L25-L28](../../lib/db/index.ts#L25-L28)), cloud guard ([get-current-user.ts#L25-L29](../../lib/auth/get-current-user.ts#L25-L29)).

## Step 7: Scaling & reliability concerns

- Junior: "add servers."
- Mid: rate-limit per user/IP (Redis); cache history reads; the DB is the shared state, app servers are stateless.
- Senior: LLM latency dominates, so optimize time-to-first-token, not the DB; persistence hanging off a live stream is a durability risk (instance death loses writes) → outbox/queue; rate limits should be atomic and consider fail-open vs fail-closed.

Morphic: Redis limits ([rate-limit/](../../lib/rate-limit/adaptive-limit.ts)), `unstable_cache` history ([lib/actions/chat.ts#L17-L34](../../lib/actions/chat.ts#L17-L34)), stateless app + Postgres. Gap: no outbox (architecture-critique #1).

## Step 8: Observability

- Mid: log errors, trace LLM calls.
- Senior: trace each research run (Langfuse), metric the silent soft-fails (dropped citations, truncation, persist failures), health-check the DB connection *and* that it's the RLS-binding role.

Morphic: Langfuse traces, perf logging, DB startup check — but many swallow-and-log gaps ([06-observability-and-operations.md](../05-quality-engineering/06-observability-and-operations.md)).

---

## Variation prompts (senior stretch)

1. **"Add multi-tenancy (organizations)."** — a chat belongs to an org, not just a user; RLS keys on org membership; the composite index and policies change; how do you migrate existing single-owner chats? (Additive `org_id`, backfill to a personal org.)
2. **"Make it real-time across a user's devices."** — broadcast message events (Senior Project 2); the crux is authorizing the subscription channel with the same visibility rules — a naive pub/sub leaks across tenants.
3. **"Handle 10× traffic."** — LLM cost/latency dominates: queue + backpressure, cache identical queries, move persistence to an outbox drained by workers, shard Redis; the DB is fine (indexed, stateless app).
4. **"Add semantic search over history."** — pgvector with the *same* RLS ownership; embed on write (async via the outbox); retrieval runs under the user's context (Senior Project 4).
5. **"Guarantee no lost messages."** — outbox in the same transaction as the user message; at-least-once + idempotent consumer; exactly-once is impossible, say so.

## How to run this in the room

State assumptions, sketch the API + data model first (interviewers anchor on those), then walk the request path, *then* discuss scale. At each step, say the tradeoff out loud ("optimistic create wins latency but risks a durability window") — that sentence is what separates mid from senior. Cite Morphic where it helps ("in a codebase I studied, they…") — concrete beats abstract every time.

Drill: whiteboard the whole thing in 30 minutes on paper, then grade against this file. Strong = you raised citations-as-a-contract and persistence-durability *unprompted*.
