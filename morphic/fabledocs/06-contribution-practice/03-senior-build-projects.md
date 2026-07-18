# Senior Build Projects

Six projects (2 days–4 weeks). Each is realistic — something a Morphic maintainer might actually accept — with the full senior packet: problem, value, design, likely files, migration, tests, security, performance, rollout/rollback, open questions, stretch. Write an RFC first ([07-career-and-collaboration/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)).

---

## Project 1: Durable persistence via an outbox (raise the reliability ceiling)
Estimate: 2–3 weeks. Interview story potential: *the* system-design story — "made a best-effort pipeline durable."

Problem: all persistence hangs off a live stream's `onFinish` ([create-chat-stream-response.ts#L196-L218](../../lib/streaming/create-chat-stream-response.ts#L196-L218)); instance death = silent loss (architecture-critique #1).
Product value: no lost conversations; foundation for anything billable/notifiable.
Design checklist: outbox table written **in the same transaction** as the user message; a drainer (cron or queue worker) that saves the assistant message idempotently; at-least-once delivery + idempotent consumer (client-generated ids already give this — Pattern 6).
Architecture decisions: platform queue (Upstash QStash / a real broker) vs DB-polling outbox; where the drainer runs (serverless cron vs worker); ordering guarantees.
Likely files: [lib/db/schema.ts](../../lib/db/schema.ts) (+ migration), new `lib/outbox/`, [persist-stream-results.ts](../../lib/streaming/helpers/persist-stream-results.ts), [create-chat-stream-response.ts](../../lib/streaming/create-chat-stream-response.ts).
Migration plan: additive table; dual-write (outbox + existing onFinish) during rollout; cut over once the drainer is proven.
Test plan: kill-the-worker chaos test; duplicate-delivery idempotency test; ordering under concurrency.
Security: outbox rows are per-user — RLS applies.
Performance: one extra insert per message (in the existing transaction); drainer batch size tuning.
Rollout/rollback: dual-write means rollback = stop the drainer, fall back to onFinish. Open questions: exactly-once is impossible — is at-least-once + idempotency acceptable? (Yes here.) Stretch: reuse the outbox for notifications/webhooks.

## Project 2: Real-time collaborative / multi-device chat sync
Estimate: 3–4 weeks. Story: "designed live sync with conflict handling."

Problem: a chat open on two devices doesn't update live; the model streams to one client only.
Value: multi-device continuity; shared/collaborative sessions.
Design: server broadcasts message events (SSE/WebSocket/Postgres LISTEN-NOTIFY) keyed by chatId; clients reconcile against `useChat` state; RLS must gate the subscription.
Decisions: transport (WS vs SSE vs a service like Ably); authz on the channel (a user may only subscribe to their own/public chats — mirror the RLS policy); last-writer-wins vs CRDT for edits.
Likely files: new realtime module, [components/chat.tsx](../../components/chat.tsx), the persistence path.
Migration: additive; feature-flagged.
Security: channel authorization is the crux — a naive pub/sub leaks across tenants; reuse the visibility model ([schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67)).
Rollout/rollback: flag per user; fall back to polling.
Open questions: how do live updates interact with regeneration's destructive deletes (Flow 5)? Stretch: presence indicators.

## Project 3: MCP tool integration surfaced end to end
Estimate: 2–3 weeks. Story: "extended an agent with a plugin protocol."

Problem: the persistence layer *already* anticipates dynamic/MCP tools ([schema.ts#L186-L190](../../lib/db/schema.ts#L186-L190), dynamic mapping in [message-mapping.ts#L255-L275](../../lib/utils/message-mapping.ts#L255-L275), [.mcp.json](../../.mcp.json)) but there's no first-class user-facing MCP tool config.
Value: users add their own tools without code.
Design: MCP client → dynamic tool registration into the researcher's tool set ([researcher.ts#L115-L146](../../lib/agents/researcher.ts#L115-L146)); UI to add/authorize servers; persistence already handles `tool-dynamic`.
Decisions: per-user vs global tools; sandboxing/authorization of third-party tools (capability boundary!); secret storage for tool credentials.
Security: **the crux** — an MCP tool runs with server privileges; treat adding one like granting a capability (compare the render catalog boundary). SSRF/exfiltration review required.
Migration: additive. Rollout: allowlist of vetted servers first. Open questions: how do dynamic tool schemas interact with per-model strict-schema requirements (Pattern 16)? Stretch: a tool marketplace.

## Project 4: Semantic search over the user's own chat history (RAG)
Estimate: 2–3 weeks. Story: "built retrieval over user data with tenant isolation."

Problem: no way to search past chats semantically.
Value: "find that thing I asked last month."
Design: embed message text on write; store vectors (pgvector) with the same RLS ownership; a retrieval tool the agent can call.
Decisions: embedding model/provider; pgvector vs external vector DB; index choice (HNSW/IVFFlat); when to embed (write-path vs batch).
Likely files: schema + migration (pgvector extension, embeddings table with RLS), new tool in [lib/tools/](../../lib/tools/search.ts), [researcher.ts](../../lib/agents/researcher.ts).
Security: embeddings are user data — RLS on the vector table; retrieval must run under the user's context (extend the RLS pattern, not bypass it).
Performance: embedding cost on write (async/outbox!); ANN index tuning. Rollout: backfill existing messages in batches. Open questions: cross-chat retrieval vs privacy expectations.

## Project 5: Provider-agnostic streaming eval + regression harness
Estimate: 2 weeks. Story: "built the eval harness that let us change prompts safely."

Problem: prompts are code ([search-mode-prompts.ts](../../lib/agents/prompts/search-mode-prompts.ts)) with no CI coverage — a prompt edit is an untested production change.
Value: catch citation-format/output-format regressions before merge.
Design: a golden-set of queries + assertions on structure (has `##` heading, valid citation format, spec blocks parse via the existing [parse-spec-block](../../lib/render/parse-spec-block.ts)); run against a cheap model in a nightly job (not PR CI — needs keys).
Decisions: deterministic structural checks vs LLM-graded quality; cost budget; where it runs.
Files: new `evals/`, reuse [citation.ts](../../lib/utils/citation.ts) + render parsers as oracles.
Rollout: nightly, non-blocking → promote key checks to required. Open questions: how to keep the golden set fresh. Stretch: A/B two prompt versions.

## Project 6: Admin analytics dashboard from existing events
Estimate: 2 weeks. Story: "turned raw events into an operator dashboard."

Problem: analytics events exist ([lib/analytics/](../../lib/analytics/index.ts)) but there's no aggregate view; ops questions ("how many adaptive chats today?") need ad-hoc queries.
Value: operator visibility; feeds capacity/limit tuning.
Design: aggregate the adaptive/chat/feedback events into materialized summaries; an admin-only page (authz!).
Decisions: query-time vs materialized; admin authorization model (there's no admin role today — design one).
Security: admin gating is net-new — do not reuse plain user auth; the dashboard reads across users, so RLS must be deliberately scoped.
Rollout: read-only, behind an admin flag. Open questions: privacy — aggregate only, never per-user content.

---

Each project's *first deliverable is the RFC*, not code. A senior is judged on the design doc and the rollback plan as much as the diff. Pick Project 1 if you want the strongest system-design interview narrative; Project 4 if you want to show data-modeling + isolation depth.
