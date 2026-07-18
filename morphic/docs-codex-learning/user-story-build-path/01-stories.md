# User Stories

## Story 1: Make Empty Submit Feedback Clearer

**Difficulty:** Easy

**Estimated Time:** 1 hour

**Skills You'll Practice:** React events, controlled inputs, UX polish

**The Story:** As a chat user, I want a clear visual cue when I try to submit an empty prompt so that I understand why nothing happened.

**Why This Story Matters:** `ChatPanel` currently prevents empty Enter submits (`components/chat-panel.tsx:277-280`) and disables the submit button when input is empty (`components/chat-panel.tsx:369-371`). A small cue can make the app feel more intentional.

**Acceptance Criteria:**
- [ ] Pressing Enter on an empty prompt does not submit.
- [ ] The user receives a small non-blocking cue.
- [ ] The cue does not appear when Shift+Enter creates a newline.
- [ ] The submit button behavior remains unchanged.

**Files You'll Likely Touch:**
- `components/chat-panel.tsx` - textarea key handling and submit UI.

**Relevant Existing Patterns:**
- `toast.error` usage in `components/chat-panel.tsx:199-202`.
- Empty Enter guard in `components/chat-panel.tsx:270-288`.

**High-Level Implementation Plan:**
1. In `components/chat-panel.tsx`, find the Enter key handler.
2. Inside the `input.trim().length === 0` branch, add a non-blocking cue using the existing `toast` import.
3. Keep `e.preventDefault()` and `return`.
4. Manually verify normal submit, empty Enter, and Shift+Enter.

**Testing and Verification Plan:**
Manual verification is enough for a first UI polish story. If adding a test, use React Testing Library to simulate keydown and assert the form submit callback is not called.

**Tips:**
- Do not change `Chat.onSubmit`; the UI already blocks earlier.
- Keep the message short.
- Do not enable the disabled submit button.

**What Could Go Wrong:**
- Cue fires during IME composition; check `isComposing`.
- Cue fires on Shift+Enter; inspect the existing condition.
- Toast becomes annoying if held key repeats; consider whether this matters.

**Stretch Goal:** Add an accessible `aria-live` helper text for empty submit attempts.

**Connects To:** Story 2, because both modify small UI behavior safely.

## Story 2: Show Result Count In Search Results

**Difficulty:** Easy

**Estimated Time:** 1 hour

**Skills You'll Practice:** Props, conditional rendering, UI state

**The Story:** As a user reading search sources, I want to see how many sources are available so that I understand whether I am seeing all results or a preview.

**Why This Story Matters:** `SearchResults` hides results after the first three in grid mode (`components/search-results.tsx:29-31`). A count helps users trust the source preview.

**Acceptance Criteria:**
- [ ] Grid mode shows total result count.
- [ ] List mode remains readable.
- [ ] Empty result arrays do not crash.
- [ ] Existing "View more" behavior still works.

**Files You'll Likely Touch:**
- `components/search-results.tsx` - displays search result cards.

**Relevant Existing Patterns:**
- Local `showAllResults` state in `components/search-results.tsx:22-31`.
- Button pattern in `components/search-results.tsx:111-124`.

**High-Level Implementation Plan:**
1. Add a small count label near the results container.
2. Keep display-mode branches simple.
3. Make sure count handles `0`, `1`, and plural values.
4. Verify grid and list modes.

**Testing and Verification Plan:**
Add a small component test if the repo has nearby tests for this component; otherwise manually inspect with mocked results in a dev session.

**Tips:**
- Do not fetch result count; use `results.length`.
- Avoid changing card layout.
- Guard code that calls `new URL(result.url)`.

**What Could Go Wrong:**
- Count creates layout shift in tight spaces.
- Count duplicates "View more" meaning.
- Invalid URLs still throw before count renders.

**Stretch Goal:** Show "Showing 3 of N" before expanding.

**Connects To:** Story 4, where source display becomes richer.

## Story 3: Clarify Search Mode Labels

**Difficulty:** Easy

**Estimated Time:** 1.5 hours

**Skills You'll Practice:** Configuration-driven UI, copy changes, review discipline

**The Story:** As a new user, I want search mode descriptions to be clearer so that I can choose Quick or Adaptive confidently.

**Why This Story Matters:** Search mode has product and cost implications. The labels live in config (`lib/config/search-modes.ts:15-31`) and are displayed in `components/search-mode-selector.tsx:68-203`.

**Acceptance Criteria:**
- [ ] Quick description explains fast concise search.
- [ ] Adaptive description explains deeper agentic search.
- [ ] UI layout does not overflow on mobile.
- [ ] No behavior changes.

**Files You'll Likely Touch:**
- `lib/config/search-modes.ts` - label/description config.

**Relevant Existing Patterns:**
- `SEARCH_MODE_CONFIGS` in `lib/config/search-modes.ts:15-31`.
- Mobile display in `components/search-mode-selector.tsx:70-125`.
- Desktop hover card in `components/search-mode-selector.tsx:127-203`.

**High-Level Implementation Plan:**
1. Edit descriptions only in `SEARCH_MODE_CONFIGS`.
2. Keep copy short enough for dropdown and hover card.
3. Run formatter if needed.
4. Verify both mobile and desktop widths.

**Testing and Verification Plan:**
Manual visual verification. No unit test needed unless copy is tied to behavior.

**Tips:**
- Avoid adding new modes.
- Do not touch cookie validation.
- Keep descriptions parallel in length.

**What Could Go Wrong:**
- Long text wraps awkwardly in mobile dropdown.
- Copy promises behavior the agent does not provide.
- You accidentally alter `value`, breaking cookies.

**Stretch Goal:** Add a tooltip to the submit area explaining current mode.

**Connects To:** Story 6, which changes validation around mode behavior.

## Story 4: Add A Compact Source List Toggle

**Difficulty:** Medium

**Estimated Time:** 2.5 hours

**Skills You'll Practice:** UI state, component reuse, source rendering

**The Story:** As a user reviewing an answer, I want to toggle sources between card and compact list views so that I can scan evidence faster.

**Why This Story Matters:** `SearchResults` already supports `displayMode?: 'grid' | 'list'` (`components/search-results.tsx:13-21`), but callers may not expose a user control.

**Acceptance Criteria:**
- [ ] User can switch between grid and list display where search results render.
- [ ] Preference only affects current display unless intentionally persisted.
- [ ] Existing result links still open in a new tab.
- [ ] No search API behavior changes.

**Files You'll Likely Touch:**
- `components/search-results.tsx` - existing rendering modes.
- `components/research-process-section.tsx` or related source display component - likely caller for search results.
- `components/ui/button.tsx` - existing button primitive to reuse, not modify.

**Relevant Existing Patterns:**
- `displayMode` prop in `components/search-results.tsx:13-21`.
- Grid/list branches in `components/search-results.tsx:33-128`.
- Button usage in `components/chat-panel.tsx:351-384`.

**High-Level Implementation Plan:**
1. Find where `SearchResults` is rendered with `rg "SearchResults"`.
2. Add local state in that caller for display mode.
3. Add two small icon/text controls using existing `Button`.
4. Pass the selected `displayMode` prop into `SearchResults`.
5. Verify both branches render.

**Testing and Verification Plan:**
Manual test with a prompt that triggers search. Optional component test for toggling prop-driven mode.

**Tips:**
- Do not add global state for this first version.
- Keep controls near the sources, not in the main chat input.
- Avoid changing `SearchResultItem`.

**What Could Go Wrong:**
- The caller renders during streaming and state resets unexpectedly.
- Control appears for empty results.
- Layout gets crowded on mobile.

**Stretch Goal:** Persist the display preference in a cookie like search mode.

**Connects To:** Story 5, where existing data becomes a richer UI feature.

## Story 5: Add A Current Chat Metadata Panel

**Difficulty:** Medium

**Estimated Time:** 3 hours

**Skills You'll Practice:** Props, metadata display, component composition

**The Story:** As a returning user, I want to see basic metadata for the current chat so that I understand which model and mode produced the answer.

**Why This Story Matters:** Stream metadata includes `traceId`, `searchMode`, and `modelId` (`lib/streaming/create-chat-stream-response.ts:186-195`; `lib/streaming/helpers/persist-stream-results.ts:24-30`). Surfacing it improves debugging and trust.

**Acceptance Criteria:**
- [ ] Latest assistant message metadata can be displayed in a compact UI.
- [ ] Missing metadata does not crash.
- [ ] The panel is hidden or minimal for empty chats.
- [ ] No persistence format changes.

**Files You'll Likely Touch:**
- `components/chat.tsx` - has access to messages.
- `components/chat-panel.tsx` or a new small component under `components/` - display location.
- `lib/types/ai.ts` - metadata type reference only, likely no change.

**Relevant Existing Patterns:**
- `UIMessageMetadata` in `lib/types/ai.ts:14-20`.
- Metadata generation in `lib/streaming/create-chat-stream-response.ts:186-195`.
- Existing composition in `components/chat-panel.tsx:344-349`.

**High-Level Implementation Plan:**
1. Identify latest assistant message in `Chat` or a child component.
2. Extract `metadata.searchMode` and `metadata.modelId`.
3. Create a small presentational component for metadata.
4. Render it only when metadata exists.
5. Verify old saved chats without metadata.

**Testing and Verification Plan:**
Add a component test if creating a pure presentational component. Manually test new and old chats.

**Tips:**
- Do not mutate message metadata.
- Keep UI small; this is operational context, not primary content.
- Treat `metadata` as optional.

**What Could Go Wrong:**
- Metadata shape differs between AI SDK versions.
- UI appears during streaming before metadata arrives.
- The panel leaks trace IDs too prominently for normal users.

**Stretch Goal:** Add a copy trace ID button for debugging.

**Connects To:** Story 7, which formalizes API validation around metadata-affecting flows.

## Story 6: Validate Chat API Request With Zod

**Difficulty:** Medium

**Estimated Time:** 4 hours

**Skills You'll Practice:** API validation, Zod, tests, boundary design

**The Story:** As a maintainer, I want `/api/chat` to validate request bodies with a schema so that malformed clients fail predictably.

**Why This Story Matters:** `/api/chat` currently parses raw JSON and checks only a few fields (`app/api/chat/route.ts:29-52`). This is the central product route.

**Acceptance Criteria:**
- [ ] Malformed JSON/body shape returns 400.
- [ ] `submit-message` requires `message`.
- [ ] `regenerate-message` requires `messageId`.
- [ ] Existing valid chat requests still work.
- [ ] Tests cover invalid trigger/body cases.

**Files You'll Likely Touch:**
- `app/api/chat/route.ts` - parse and validation boundary.
- `lib/schema/` - likely new chat request schema.
- `app/api/chat/__tests__/route.test.ts` - likely new test file.

**Relevant Existing Patterns:**
- Zod search schema in `lib/schema/search.tsx:6-94`.
- Feedback route validation tests in `app/api/feedback/__tests__/route.test.ts:141-201`.
- AI SDK request body construction in `components/chat.tsx:100-129`.

**High-Level Implementation Plan:**
1. Define a Zod schema for the chat route body.
2. Replace direct destructuring with `safeParse`.
3. Preserve existing error semantics where practical.
4. Add focused route tests with mocked dependencies.
5. Run `bun run test` for the new tests.

**Testing and Verification Plan:**
Add route tests for missing `messageId`, missing `message`, invalid trigger, and valid minimal request reaching the streaming branch with mocks.

**Tips:**
- Keep schema permissive around AI SDK `message` shape at first.
- Do not validate every message part deeply in this story.
- Preserve guest/auth behavior.

**What Could Go Wrong:**
- Schema too strict breaks AI SDK transport.
- Test mocks accidentally import real provider code.
- Error status changes break client error handling.

**Stretch Goal:** Add a typed `ChatRouteBody` export inferred from the schema.

**Connects To:** Story 8, which adds persistence-facing behavior.

## Story 7: Add Upload Chat Ownership Check

**Difficulty:** Medium

**Estimated Time:** 4 hours

**Skills You'll Practice:** authorization, API routes, server actions, tests

**The Story:** As an authenticated user, I want uploads to be associated only with chats I own so that files cannot be placed under another chat's path.

**Why This Story Matters:** Upload route authenticates the user and validates file type/size, but it uses `chatId` from form data without loading the chat (`app/api/upload/route.ts:41-62`).

**Acceptance Criteria:**
- [ ] Upload succeeds for a chat owned by current user.
- [ ] Upload returns 403 for a chat not owned by current user.
- [ ] Missing `chatId` returns 400.
- [ ] Guest upload remains unauthorized.

**Files You'll Likely Touch:**
- `app/api/upload/route.ts` - route validation and authorization.
- `lib/actions/chat.ts` or `lib/db/actions.ts` - reuse existing `loadChat`/`getChat` pattern.
- `app/api/upload/__tests__/route.test.ts` - likely new route test.

**Relevant Existing Patterns:**
- Current upload checks in `app/api/upload/route.ts:15-62`.
- Chat loading action in `lib/actions/chat.ts:63-69`.
- DB permission check in `lib/db/actions.ts:52-80`.

**High-Level Implementation Plan:**
1. Validate `chatId` exists after form parsing.
2. Load chat with current `userId`.
3. Reject if missing or `chat.userId !== userId`.
4. Keep file validation and storage upload unchanged.
5. Add tests with mocked auth/chat/storage.

**Testing and Verification Plan:**
Mock `getCurrentUserId`, chat load, and R2 client. Test 401, 400, 403, and successful path.

**Tips:**
- Do not call storage before authorization.
- Keep object key format unchanged.
- Consider new-chat timing: file upload happens before first persisted chat may exist.

**What Could Go Wrong:**
- New chats may not be in DB yet when upload happens.
- Strict ownership check can block uploads for unsaved chats.
- Tests may need to model pending chat creation behavior.

**Stretch Goal:** Create an explicit temporary upload session concept for new chats.

**Connects To:** Story 9, where persistence and ownership become broader.

## Story 8: Add Chat Pinning

**Difficulty:** Hard

**Estimated Time:** 8 hours

**Skills You'll Practice:** schema changes, migrations, server actions, sidebar UI, tests

**The Story:** As a frequent user, I want to pin important chats so that they stay easy to find.

**Why This Story Matters:** Chat history is central to authenticated use. The schema has chats with title, user, visibility, and createdAt (`lib/db/schema.ts:25-70`) but no priority marker.

**Acceptance Criteria:**
- [ ] Authenticated user can pin/unpin own chats.
- [ ] Pinned chats appear before unpinned chats in history.
- [ ] Guests do not see pin controls.
- [ ] Ownership checks prevent pinning another user's chat.
- [ ] Tests cover action behavior.

**Files You'll Likely Touch:**
- `lib/db/schema.ts` - add `pinned` or `pinnedAt`.
- `drizzle/` - generated migration.
- `lib/db/actions.ts` - update query ordering and mutation.
- `lib/actions/chat.ts` - server action wrapper and cache invalidation.
- `components/sidebar/*` - render pin control in history item.
- Tests under `lib/actions/__tests__/` or `lib/db/__tests__/`.

**Relevant Existing Patterns:**
- Chat table definition in `lib/db/schema.ts:25-70`.
- Chat pagination in `lib/actions/chat.ts:47-56`.
- Delete/share actions with auth and cache invalidation in `lib/actions/chat.ts:202-282`.
- Sidebar files under `components/sidebar/`.

**High-Level Implementation Plan:**
1. Decide `pinnedAt` over boolean if ordering by pin time matters.
2. Update schema and generate migration.
3. Add DB action to set/clear pin for a user's chat.
4. Update chat list ordering.
5. Add server action with `getCurrentUserId`.
6. Add sidebar button/menu item.
7. Add tests for auth/ownership and ordering.

**Testing and Verification Plan:**
Run migration locally, test pin/unpin manually, add action tests with mocked DB actions, and run `bun run test`.

**Tips:**
- Cache invalidation should include chat list tags.
- Keep UI control small in `chat-menu-item`.
- Do not expose pinning on shared public pages unless intended.

**What Could Go Wrong:**
- Migration drift between `lib/db/schema.ts` and `drizzle/`.
- Chat ordering becomes unstable for same timestamps.
- Cache revalidation does not update sidebar.

**Stretch Goal:** Add a "Pinned" section header in chat history.

**Connects To:** Story 9, which also changes persisted chat metadata.

## Story 9: Add Per-Message Bookmarking

**Difficulty:** Hard

**Estimated Time:** 10 hours

**Skills You'll Practice:** DB modeling, message actions, UI controls, cache invalidation, tests

**The Story:** As a researcher, I want to bookmark important assistant messages so that I can return to useful answers inside long chats.

**Why This Story Matters:** Messages already have metadata and parts (`lib/db/schema.ts:72-120`, `lib/db/schema.ts:122-258`). Bookmarking teaches you how to extend persisted message-level behavior.

**Acceptance Criteria:**
- [ ] User can bookmark/unbookmark assistant messages in own chats.
- [ ] Bookmarked state persists after reload.
- [ ] Bookmarks are not allowed for messages outside owned chats.
- [ ] UI indicates bookmarked messages clearly.
- [ ] Tests cover authorization and persistence.

**Files You'll Likely Touch:**
- `lib/db/schema.ts` - decide metadata update vs new column/table.
- `lib/db/actions.ts` - message metadata update or bookmark table query.
- `lib/actions/chat.ts` or new action file - server action.
- `components/message-actions.tsx` - likely UI control.
- `components/render-message.tsx` - message display state may flow here.
- Tests in `lib/actions/__tests__/`.

**Relevant Existing Patterns:**
- Feedback updates message metadata via feedback route/action (`app/api/feedback/route.ts:60-68`).
- Message metadata type in `lib/types/ai.ts:14-20`.
- Message persistence in `lib/db/actions.ts:86-125`.

**High-Level Implementation Plan:**
1. Choose storage: metadata is faster; a table is more queryable.
2. Add DB mutation with ownership check through chat relationship.
3. Add server action that gets current user.
4. Add UI button near existing message actions.
5. Ensure saved messages hydrate bookmarked state.
6. Add tests around unauthorized message IDs.

**Testing and Verification Plan:**
Manual: bookmark, reload, unbookmark, attempt from another user in tests. Add DB/action tests with mocked user context.

**Tips:**
- Avoid updating full message parts for bookmark-only changes.
- Preserve existing feedback metadata.
- Use cache revalidation for `chat-${chatId}`.

**What Could Go Wrong:**
- Metadata overwrite removes trace/search/model info.
- Bookmark UI appears for user messages unintentionally.
- Ownership check only checks message ID, not chat owner.

**Stretch Goal:** Add a bookmarks-only navigation list inside a chat.

**Connects To:** Story 10, because both require senior-level boundary and persistence judgment.

## Story 10: Add Advanced Search Crawl Safety Controls

**Difficulty:** Expert

**Estimated Time:** 16 hours

**Skills You'll Practice:** security, performance, API design, tests, refactoring plan

**The Story:** As an operator, I want advanced search crawling to reject unsafe URLs and limit concurrency so that Morphic remains reliable and safe during deep search.

**Why This Story Matters:** Advanced search crawls result URLs and parses HTML (`app/api/advanced-search/route.ts:313-407`). It currently uses `Promise.all` for many crawls (`app/api/advanced-search/route.ts:258-263`).

**Acceptance Criteria:**
- [ ] Crawler rejects localhost, private IPs, non-HTTP(S), and malformed URLs.
- [ ] Crawl concurrency is limited.
- [ ] Failed/rejected URLs do not fail the whole search.
- [ ] Existing cache behavior remains.
- [ ] Tests cover URL safety and concurrency helper.

**Files You'll Likely Touch:**
- `app/api/advanced-search/route.ts` - current implementation.
- New helper under `lib/tools/search/` or `lib/utils/` - URL safety/concurrency logic.
- Tests under `lib/utils/__tests__/` or route tests.

**Relevant Existing Patterns:**
- Cache helpers in `app/api/advanced-search/route.ts:66-113`.
- Advanced search flow in `app/api/advanced-search/route.ts:137-311`.
- HTTP fetch helpers in `app/api/advanced-search/route.ts:539-644`.
- Search provider abstraction in `lib/tools/search/providers/`.

**High-Level Implementation Plan:**
1. Extract URL safety helper with pure tests.
2. Extract a small concurrency-limited mapper helper or use a tiny internal queue.
3. Apply safety check before `fetchHtmlWithTimeout`.
4. Replace broad `Promise.all` with limited concurrency.
5. Preserve result shape and cache key behavior.
6. Add tests for unsafe URLs and partial crawl failure.
7. Manually verify advanced SearXNG mode.

**Testing and Verification Plan:**
Unit test pure helpers. Integration-test route behavior with mocked SearXNG/fetch if feasible. Manual test with advanced search enabled.

**Tips:**
- Keep behavior-compatible result shape.
- Do not turn crawler rejection into a 500.
- Reject IP ranges after DNS resolution if you implement hostname resolution; string checks alone are incomplete.
- Watch serverless runtime constraints around Node networking.

**What Could Go Wrong:**
- Concurrency helper changes result order.
- Safety check blocks valid public domains.
- Tests become flaky if they hit the network.

**Stretch Goal:** Add per-request crawl budget metrics using existing perf logging patterns.

**Connects To:** This is the capstone story; it uses architecture, security, performance, API, and testing judgment.

