# Mid-Level Feature Tickets

12 cross-layer tickets. Each requires a **design note before code** (what changes at each layer, risks, rollback). These touch schema/API/UI/tests together — the mid-level daily reality.

Rule: write the design note (see [07-career-and-collaboration/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)) and get it agreed before implementing. Each ticket lists risk + rollback because that's the part juniors skip.

---

## Ticket M1: Zod-validate the `/api/chat` request body
Difficulty: Medium — 1 day. Layers: API + shared types.
Story: reject malformed chat requests with a 400 (not a 500) and stop trusting `message.parts`.
Design note must cover: schema definition (reuse the AI SDK UIMessage shape if exported, else structural), `.safeParse` → 400 with issues, forward-compat (non-strict / passthrough).
Anchors: [route.ts#L30-L52](../../app/api/chat/route.ts#L30-L52); the *safe* approach in [Kata 3](../04-code-reading-gym/04-review-katas.md).
Acceptance: - [ ] valid bodies unchanged - [ ] malformed → 400 + issues - [ ] table-tests of accepted/rejected bodies.
Risk: over-strict schema rejects legitimate newer fields → **characterize current accepted bodies first**, run log-only for a release if cautious. Rollback: feature-free — revert the PR.
Interview story potential: "Added input validation to the core endpoint without breaking existing clients by characterizing first."

## Ticket M2: Structured error channel from server to client
Difficulty: Medium–Hard — 1–2 days. Layers: API + client transport.
Story: replace regex-parsing JSON out of `error.message` ([chat.tsx#L135-L214](../../components/chat.tsx#L135-L214)) with a typed error shape.
Design note: how do errors survive the AI SDK transport? (custom error serialization / a data part / response headers.) Migration: support both old and new shapes during rollout.
Risk: touches every error branch; regressions in modal routing. Rollback: keep the string-parse fallback until the new path is proven.
Interview story: "Killed a stringly-typed error-parsing smell with a typed channel."

## Ticket M3: Per-model context windows from the model source
Difficulty: Medium — 1 day. Layers: config + utils.
Story: fix over-truncation on current default models by sourcing windows dynamically or defaulting generously.
Anchors: [context-window.ts#L12-L38](../../lib/utils/context-window.ts#L12-L38); defaults [default-model.ts](../../lib/config/default-model.ts), [cloud.json](../../config/models/cloud.json).
Acceptance: - [ ] known models unchanged - [ ] unknown models get a safe default + a warning log/metric - [ ] budget-math tests.
Risk: too-generous default → provider-side context errors; too-small → truncation. Rollback: data-only revert.
Interview story: "Diagnosed silent answer-quality degradation to a stale config table and fixed it." (Ties to debugging Scenario 3.)

## Ticket M4: Delete-after-success for regeneration
Difficulty: Medium–Hard — 1–2 days. Layers: streaming + actions.
Story: don't destroy the previous answer until the replacement succeeds.
Design note: reorder vs soft-delete; how the UI shows the transition; id stability.
Anchors: [prepare-messages.ts#L57-L78](../../lib/streaming/helpers/prepare-messages.ts#L57-L78).
Acceptance: - [ ] failed regen keeps old answer - [ ] successful regen replaces it - [ ] integration test forcing an LLM error.
Risk: id churn breaks UI keys; double answers if delete missed. Rollback: revert to delete-first.
Interview story: "Made a destructive operation safe by reordering + a regression test." (Strong STAR — conflict/mistake prompt.)

## Ticket M5: Unify the three rate limiters
Difficulty: Medium — 1 day. Layers: rate-limit + callers.
Story: one `checkDailyLimit(scope, id, limit)` returning a result; routes map results→responses.
Design note: shared core + INCR/EXPIRE atomicity (Lua); preserve 401-vs-429 + headers per caller.
Anchors: [guest-limit.ts](../../lib/rate-limit/guest-limit.ts), [chat-limits.ts](../../lib/rate-limit/chat-limits.ts), [adaptive-limit.ts](../../lib/rate-limit/adaptive-limit.ts).
Risk: subtle behavior drift (adaptive tracks analytics). Rollback: per-caller, adopt one at a time.
Interview story: "Consolidated triplicated logic behind one interface while preserving three distinct behaviors."

## Ticket M6: Message-mapping refactor to a table-driven mapper
Difficulty: Hard — 2 days. Layers: utils + types.
Story: replace the 700-line switch with a per-tool descriptor table.
Design note: descriptor shape (columns, state handling); type-safety for `tool_${name}_*` keys; must round-trip all existing part types.
Anchors: [message-mapping.ts](../../lib/utils/message-mapping.ts). Depends on Ticket 1's tests existing first.
Risk: high — this is the persistence heart. Rollback: keep old functions behind a flag; diff outputs on a fixture corpus.
Interview story: "Cut a 700-line file by N% behind a characterization test suite." (Strong technical-tradeoff STAR.)

## Ticket M7: Chat pinning
Difficulty: Medium — 1 day. Layers: schema + db + actions + UI.
Design note: boolean column vs join; index; migration (additive); RLS unchanged (same owner); cache invalidation points.
Anchors: schema [schema.ts#L26-L68](../../lib/db/schema.ts#L26-L68), sidebar [chat-history-client.tsx](../../components/sidebar/chat-history-client.tsx).
Risk: migration lock on large tables (nullable/defaulted column = cheap). Rollback: forward inverse migration.
Interview story: "Shipped a user-facing feature end to end: migration, API, UI, tests."

## Ticket M8: Feedback endpoint authorization
Difficulty: Medium — 1 day. Layers: API + actions.
Story: verify the caller may set feedback on that message (ownership via RLS-scoped read) before writing.
Anchors: [feedback/route.ts#L48-L68](../../app/api/feedback/route.ts#L48-L68), [actions/feedback.ts](../../lib/actions/feedback.ts); risk register entry.
Design note: how anonymous deployments still work (ownership check, not identity requirement).
Risk: breaking anonymous feedback. Rollback: revert.
Interview story: "Closed an authorization gap while keeping the anonymous mode working."

## Ticket M9: Dropped-citation observability
Difficulty: Medium — 1 day. Layers: utils + analytics.
Story: emit a metric each time `processCitations` drops a citation, labeled by reason.
Anchors: [citation.ts#L88-L100](../../lib/utils/citation.ts#L88-L100), analytics [lib/analytics/](../../lib/analytics/index.ts).
Risk: metric cardinality. Rollback: revert. Interview story: "Added observability to a silent soft-fail so we could measure model citation quality."

## Ticket M10: SSRF guard for the fetch tool
Difficulty: Medium–Hard — 1–2 days. Layers: tools + utils.
Story: block private/link-local/metadata IPs in the fetch tool + advanced-search.
Design note: DNS-resolve then check the *resolved* IP (not just the hostname) against denied ranges; allowlist option.
Anchors: [fetch.ts#L9-L21](../../lib/tools/fetch.ts#L9-L21), [search.ts#L84-L108](../../lib/tools/search.ts#L84-L108); risk register.
Risk: over-blocking legit URLs; DNS-rebinding subtlety. Rollback: config flag.
Interview story: "Investigated and mitigated an SSRF surface in a server-side fetch." (Strong security STAR.)

## Ticket M11: Keyset pagination for chat history
Difficulty: Medium — 1 day. Layers: db + API + UI.
Story: replace offset pagination with keyset on `(created_at, id)` for large histories.
Anchors: [db/actions.ts#L288-L314](../../lib/db/actions.ts#L288-L314), uses index [schema.ts#L45-L48](../../lib/db/schema.ts#L45-L48).
Risk: cursor encoding, API contract change (`nextOffset` → cursor). Rollback: keep offset endpoint during transition.
Interview story: "Moved to keyset pagination and explained why offset degrades at scale."

## Ticket M12: Persist-failure user feedback
Difficulty: Medium — 1 day. Layers: streaming + client.
Story: when `persistStreamResults` ultimately fails, tell the user their message wasn't saved (currently silent, [persist-stream-results.ts#L94-L97](../../lib/streaming/helpers/persist-stream-results.ts#L94-L97)).
Design note: how a post-stream server failure reaches the client (the structured-error channel of M2 helps).
Risk: false negatives (retry succeeded). Rollback: revert. Interview story: "Turned a silent data-loss window into a visible, recoverable state."

---

Sequencing note: M1 and M3 are the best first mid-level tickets (contained, high-value). M6 depends on Ticket 1. M2 unlocks M12. Do M4 when you want a *strong* behavioral story.
