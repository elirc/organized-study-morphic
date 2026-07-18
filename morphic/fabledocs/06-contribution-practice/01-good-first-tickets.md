# Good First Tickets

20 junior tickets, spread across UI, API, persistence, tools, config, tests, docs. Each is small, low-blast-radius, and follows an existing pattern. Format is compressed after the first two full examples; every one carries acceptance criteria, anchors, and an interview-story line.

Pre-work for all: run the pre-PR ritual before pushing ([04-tooling-and-build-system.md](../02-stack-and-language-mastery/04-tooling-and-build-system.md)).

---

## Ticket 1: Add a round-trip test for message-mapping (search part)
Difficulty: Easy — 1–2h. Skills: Vitest, discriminated unions.
Story: As a maintainer, I want `message-mapping.ts` covered so refactors are safe.
Why good: 775 lines, zero tests; a round-trip test catches the whole column-key bug class. Blast radius: test-only.
Acceptance: - [ ] test file at `lib/utils/__tests__/message-mapping.test.ts` - [ ] round-trips text, file, and `tool-search` (output-available) parts through `mapUIMessagePartsToDBParts` → `mapDBPartToUIMessagePart` - [ ] passes `bun run test`.
Read first: [message-mapping.ts#L127-L339](../../lib/utils/message-mapping.ts#L127-L339), [L344-L666](../../lib/utils/message-mapping.ts#L344-L666); pattern from [citation.test.ts](../../lib/utils/__tests__/citation.test.ts).
Files touched: new test file only.
Plan: 1. Build a sample UI part. 2. Map to DB, back to UI. 3. `toEqual`.
What could go wrong: the DB shape adds `order`/`messageId` — assert on the round-tripped *UI* shape, not the DB intermediate.
Review Qs: does it cover an error-state part? Reject if: it asserts on private column names instead of behavior.
Interview story potential: "I added the first tests to an untested 700-line module and found a latent bug."

## Ticket 2: Clamp `/api/chats` pagination limit
Difficulty: Easy — 1h. Skills: input validation, HTTP.
Story: As an operator, I want `limit` bounded so a huge value can't trigger a slow query.
Why good: one-line-ish hardening following existing parse logic; low risk. Blast radius: one route.
Acceptance: - [ ] `limit` clamped to `[1,50]`, `offset` ≥ 0 - [ ] existing behavior for normal values unchanged - [ ] a test asserting `limit=100000` → 50.
Read first: [chats/route.ts#L11-L26](../../app/api/chats/route.ts#L11-L26).
Files: [app/api/chats/route.ts](../../app/api/chats/route.ts) (+ test).
Plan: `Math.min(50, Math.max(1, parsed))`; guard `NaN`.
What could go wrong: `parseInt('abc')` → NaN; handle it.
Review Qs: is the max a named constant? Reject if: it changes the default page size (contract change).
Interview story potential: "Small input-hardening PR that closed a self-inflicted-DoS vector."

## Ticket 3: Add a per-attempt timeout to the fetch tool's API path
Difficulty: Easy–Medium — 2h. Reliability.
Acceptance: Jina/Tavily-extract fetches get an AbortController+timeout like `fetchRegularData` does; timeout → clear error.
Anchors: copy [fetch.ts#L11-L23](../../lib/tools/fetch.ts#L11-L23) into [fetchJinaReaderData](../../lib/tools/fetch.ts#L90-L121) and [fetchTavilyExtractData](../../lib/tools/fetch.ts#L123-L158). Reject if: magic-number timeout not shared with the existing 10s constant.
Interview story: "Made an external dependency fail fast instead of hanging a request."

## Ticket 4: Surface a friendly message when no model is available
Difficulty: Easy — 1–2h. UX.
Acceptance: the 503 "No enabled model" ([route.ts#L98-L103](../../app/api/chat/route.ts#L98-L103)) shows a helpful modal (config hint) instead of a generic toast.
Anchors: error parsing in [chat.tsx#L135-L214](../../components/chat.tsx#L135-L214) — add a `503`/`no enabled model` branch. Reject if: it swallows other 5xx into the same modal.
Interview story: "Turned an opaque server error into an actionable user message."

## Ticket 5: Add `bun run test` names/docs for targeted runs to scripts/README
Difficulty: Easy — 45m. Docs.
Acceptance: [scripts/README.md](../../scripts/README.md) documents targeted test/watch commands and the `bun test` vs `bun run test` trap. Reject if: contradicts [AGENTS.md](../../AGENTS.md).
Interview story: (minor) "Reduced onboarding friction."

## Ticket 6: Extract the duplicated midnight-UTC helpers
Difficulty: Easy — 1–2h. DRY.
Acceptance: `getSecondsUntilMidnight`/`getNextMidnightTimestamp` (identical in [guest-limit.ts#L14-L26](../../lib/rate-limit/guest-limit.ts#L14-L26), [chat-limits.ts#L10-L25](../../lib/rate-limit/chat-limits.ts#L10-L25), [adaptive-limit.ts#L17-L29](../../lib/rate-limit/adaptive-limit.ts#L17-L29)) live in one util; all three import it; tests still pass.
Reject if: behavior changes, or the util isn't tested. Interview story: "De-duplicated triplicated time logic without changing behavior (characterization-tested first)."

## Ticket 7: Add a test for the empty-input guard on submit
Difficulty: Easy — 1h. RTL.
Acceptance: submitting empty input + no files doesn't call `sendMessage` ([chat.tsx#L421-L452](../../components/chat.tsx#L421-L452)).
Interview story: "Pinned a UI invariant with a test."

## Ticket 8: Return a count from `clearChats`
Difficulty: Easy — 1h. Consistency with sibling actions.
Acceptance: `clearChats` returns `{success, count}` like `deleteMessagesAfter`; caller unaffected. Anchors: [lib/actions/chat.ts#L223-L238](../../lib/actions/chat.ts#L223-L238). Reject if: it doesn't handle the empty-list case.

## Ticket 9: Add `Image` alt/description fallback in the render catalog docs
Difficulty: Easy — 1h. Accessibility/docs.
Acceptance: document that catalog `Image` should carry `description` for a11y; note where props are defined ([catalog.ts#L50-L60](../../lib/render/catalog.ts#L50-L60)).

## Ticket 10: Test `getSearchSchemaForModel` picks strict schema for o-models
Difficulty: Easy — 1h. Unit.
Acceptance: `getSearchSchemaForModel('openai:o3')` → strict; `'openai:gpt-4.1'` → standard. Anchors: [search.tsx#L81-L94](../../lib/schema/search.tsx#L81-L94).
Interview story: "Covered a model-specific branch that had no test."

## Ticket 11: Guard `truncateMessages` empty result at the caller
Difficulty: Medium — 2h. Correctness.
Acceptance: if truncation yields `[]` (huge last message), the stream path handles it gracefully instead of sending zero messages. Anchors: [context-window.ts#L280-L285](../../lib/utils/context-window.ts#L280-L285), caller [create-chat-stream-response.ts#L149-L159](../../lib/streaming/create-chat-stream-response.ts#L149-L159). Reject if: no test for the empty case.
Interview story: "Found and closed an edge case in context management."

## Ticket 12: Don't leak `err.message` from the upload 500
Difficulty: Easy — 45m. Security hygiene.
Acceptance: [upload/route.ts#L63-L69](../../app/api/upload/route.ts#L63-L69) returns a generic message; detail logged server-side only. Reject if: it also hides the *400* validation messages users need.
Interview story: "Closed an information-disclosure smell in an error path."

## Ticket 13: Add a `/api/health` route
Difficulty: Medium — 2h. Ops.
Acceptance: `GET /api/health` returns 200 + `{db:'ok'}` after a cheap `SELECT 1`; 503 on failure. Anchors: reuse the check idea from [db/index.ts#L72-L91](../../lib/db/index.ts#L72-L91). Reject if: it runs an expensive query or exposes internals.
Interview story: "Added the health check the deploy pipeline needed."

## Ticket 14: Constant-ize the `100`/`75`/`255` magic numbers in title/chat
Difficulty: Easy — 1h. Readability.
Acceptance: fallback-title length ([title-generator.ts#L26](../../lib/agents/title-generator.ts#L26)), title truncation `.substring(0,255)` ([lib/actions/chat.ts#L88](../../lib/actions/chat.ts#L88)) become named constants near the schema length limits ([schema.ts#L17-L20](../../lib/db/schema.ts#L17-L20)). Reject if: values change.

## Ticket 15: Test the fail-open path of a rate limiter
Difficulty: Easy — 1h. Unit.
Acceptance: with `MORPHIC_CLOUD_DEPLOYMENT` unset, `checkAndEnforceOverallChatLimit` returns null (no enforcement). Anchors: [chat-limits.ts#L33-L34](../../lib/rate-limit/chat-limits.ts#L33-L34).
Interview story: "Documented fail-open behavior with a test so nobody 'fixes' it by accident."

## Ticket 16: Add loading/skeleton for an empty model list
Difficulty: Medium — 2–3h. UX. Anchors: [model-selector-client.tsx](../../components/model-selector-client.tsx), data from [get-model-selector-data.ts](../../lib/model-selector/get-model-selector-data.ts). Reject if: it hides a real "no providers configured" state behind a spinner forever.

## Ticket 17: Sanitize/validate `chatId` in the upload key
Difficulty: Medium — 2h. Hardening.
Acceptance: `chatId` in the object key ([upload/route.ts#L76-L78](../../app/api/upload/route.ts#L76-L78)) is sanitized like the filename (or verified to belong to the user). Reject if: it breaks existing key format for valid ids.
Interview story: "Tightened an unverified-input-into-storage-path issue."

## Ticket 18: Test `model-selection` cookie precedence
Difficulty: Easy–Medium — 2h. Unit. Anchors: [model-selection.ts#L107-L164](../../lib/utils/model-selection.ts#L107-L164), existing [model-selection.test.ts](../../lib/utils/__tests__/model-selection.test.ts). Add a case: cookie provider disabled → falls back to default.

## Ticket 19: Add ARIA label to the copy-message shortcut action
Difficulty: Easy — 1h. Accessibility. Anchors: [message-actions.tsx](../../components/message-actions.tsx).

## Ticket 20: Document the four caching layers
Difficulty: Easy — 1h. Docs. Acceptance: a short `docs/` note listing unstable_cache / useChat state / model cache / history pushState with anchors (source: [02-framework-mental-models.md](../02-stack-and-language-mastery/02-framework-mental-models.md)). Reject if: it duplicates rather than links.

---

Picking order: 1, 2, 10, 15 first (pure test/validation, teach the ritual), then 3/11/12/17 (small hardening with a design thought), then the rest. Every one is a legitimate PR — write the description using [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
