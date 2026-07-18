# Tier 3: Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions

**Tier:** Senior

**Time Estimate:** 75 minutes

**Goal:** Infer why Morphic chose its main boundaries.

**The Concept:** Architecture is a trail of trade-offs, like search results ranked by constraints rather than preferences.

**Design Intent Before You Read the Code:** Explain why Next.js API routes, AI SDK streaming, Drizzle parts, server actions, and provider abstractions fit this app.

**Find It In The Code:** `app/api/chat/route.ts:20-229`, `lib/streaming/create-chat-stream-response.ts:33-235`, `lib/db/schema.ts:122-258`, `lib/tools/search.ts:20-173`.

```ts
// lib/db/schema.ts:122-200
export const parts = pgTable('parts', { /* many typed columns for message part variants */ })
```

**The Aha Moment:** The schema is designed around reconstructing rich AI messages, not just storing chat text.

**Socratic Checkpoint:** 5 questions.

1. Why use API routes instead of only server actions for chat?
2. Why normalize message parts?
3. Why provider factory for search?
4. Why two stream paths for guest/auth?
5. What trade-off does advanced search route density reveal?

**How to Self-Grade:** Strong answers name benefits and costs for each decision.

**Connects To:** Mission 21 performance; Mission 24 git history.

### Mission 19: Find the Bugs Before They Happen

**Tier:** Senior

**Time Estimate:** 60 minutes

**Goal:** Predict bugs from ownership, boundary, and type clues.

**The Concept:** Senior debugging starts before the bug report by noticing pressure points.

**Design Intent Before You Read the Code:** Look for mismatched response shapes, duplicated allowed-value lists, unvalidated input, and stale effects.

**Find It In The Code:** `hooks/use-file-dropzone.ts:81-90`, `app/api/upload/route.ts:96-101`, `components/search-mode-selector.tsx:38-43`, `app/api/chat/route.ts:90-94`.

```ts
// hook expects uploaded.name, route returns filename
name: uploaded.name // hooks/use-file-dropzone.ts:88
filename: file.name // app/api/upload/route.ts:97
```

**The Aha Moment:** Many bugs are contract mismatches hiding between files.

**Socratic Checkpoint:** 5 questions.

1. Which client/server response mismatch did you find?
2. Which allowed-value lists can drift?
3. Which route lacks schema validation?
4. Which effect guards against duplicate listeners?
5. Which bug would a unit test catch fastest?

**How to Self-Grade:** Strong answers describe symptoms, not just code smells.

**Connects To:** Mission 20 bug injection; Mission 23 missing test.

### Mission 20: The Bug Injection Challenge

**Tier:** Senior

**Time Estimate:** 70 minutes

**Goal:** Design tests for realistic bugs without modifying production code.

**The Concept:** Bug injection is a flight simulator for debugging.

**Design Intent Before You Read the Code:** Use user-visible symptoms and test scenarios. Do not change production files.

**Find It In The Code:** `components/chat-panel.tsx:90-98`, `components/chat-panel.tsx:270-288`, `lib/streaming/helpers/persist-stream-results.ts:35-79`, `app/api/chat/route.ts:115-134`.

```tsx
// components/chat-panel.tsx:270-288
if (e.key === 'Enter' && !e.shiftKey && !isComposing && !enterDisabled) {
  textarea.form?.requestSubmit()
}
```

**The Aha Moment:** Good bug exercises name the symptom first, then the likely code.

**Socratic Checkpoint:** 5 questions.

1. What test proves IME composition does not submit?
2. What test proves adaptive guest gate returns JSON?
3. What test proves persistence retry does not throw to stream?
4. What test proves upload response shape is consumed correctly?
5. What test proves invalid search cookie resets?

**How to Self-Grade:** Strong answers include setup, action, assertion, and mocked boundary.

**Connects To:** Mission 23 missing test; Mission 19 bug prediction.

### Mission 21: Performance X-Ray

**Tier:** Senior

**Time Estimate:** 70 minutes

**Goal:** Identify performance hot paths and trade-offs.

**The Concept:** Performance is the latency budget of the answer engine: search, model, DB, and rendering all spend from it.

**Design Intent Before You Read the Code:** Look for fan-out, repeated auth/DB calls, cache design, context pruning, and streaming transforms.

**Find It In The Code:** `lib/streaming/create-chat-stream-response.ts:138-159`, `app/api/advanced-search/route.ts:148-164`, `app/api/advanced-search/route.ts:258-263`, `lib/db/actions.ts:162-175`.

```ts
// lib/streaming/create-chat-stream-response.ts:141-153
modelMessages = pruneMessages(...)
if (shouldTruncateMessages(modelMessages, model)) {
  modelMessages = truncateMessages(modelMessages, maxTokens, model.id)
}
```

**The Aha Moment:** The app protects model latency with pruning but still risks network fan-out in advanced crawling.

**Socratic Checkpoint:** 5 questions.

1. Where is context size controlled?
2. Where is search cache checked?
3. Where can crawl concurrency spike?
4. Which DB load uses `Promise.all`?
5. What metric would you add first?

**How to Self-Grade:** Strong answers separate model, DB, network, and rendering costs.

**Connects To:** Mission 18 architecture; Mission 22 security.

### Mission 22: The Security Audit

**Tier:** Senior

**Time Estimate:** 75 minutes

**Goal:** Audit auth, authorization, validation, upload, and crawling boundaries.

**The Concept:** Security is about what the app refuses to do.

**Design Intent Before You Read the Code:** Inspect every place external input crosses into server behavior.

**Find It In The Code:** `lib/auth/get-current-user.ts:22-40`, `app/api/chat/route.ts:29-52`, `app/api/upload/route.ts:15-62`, `app/api/advanced-search/route.ts:313-407`, `lib/db/schema.ts:53-68`.

```ts
// app/api/upload/route.ts:41-62
const file = formData.get('file') as File
const chatId = formData.get('chatId') as string
// validates file, but should also verify chat ownership for chatId
```

**The Aha Moment:** Authenticated user plus unverified resource ID is not full authorization.

**Socratic Checkpoint:** 5 questions.

1. What protects anonymous mode from cloud use?
2. Which API route should use Zod first?
3. What upload ownership check is missing?
4. What crawler URL classes should be rejected?
5. How does RLS complement app checks?

**How to Self-Grade:** Strong answers distinguish authentication, authorization, validation, and network egress.

**Connects To:** Mission 23 tests; Mission 19 bug prediction.

### Mission 23: Write the Test That Doesn't Exist

**Tier:** Senior

**Time Estimate:** 80 minutes

**Goal:** Design a focused test for one risky behavior.

**The Concept:** A good test is a small contract that catches a large class of mistakes.

**Design Intent Before You Read the Code:** Copy test style from existing route/action tests. Mock external services. Assert behavior at the route or helper boundary.

**Find It In The Code:** `app/api/feedback/__tests__/route.test.ts:53-201`, `lib/actions/__tests__/chat.test.ts:32-240`, `lib/db/__tests__/with-rls.test.ts:29-179`, `app/api/chat/route.ts:115-134`.

```ts
// app/api/feedback/__tests__/route.test.ts:70-87
const request = new Request('http://localhost:3000/api/feedback', { method: 'POST', body: JSON.stringify(...) })
const response = await POST(request)
expect(response.status).toBe(200)
```

**The Aha Moment:** Existing tests are templates for new tests if you match the boundary.

**Socratic Checkpoint:** 5 questions.

1. Which behavior is most expensive to break?
2. What modules need mocking?
3. What should the test assert?
4. What should the test not care about?
5. Where should the test file live?

**How to Self-Grade:** Strong answers choose one behavior, one boundary, and minimal mocks.

**Connects To:** Mission 20 bug injection; Mission 24 git history.

### Mission 24: The Git History Tells a Story

**Tier:** Senior

**Time Estimate:** 45 minutes

**Goal:** Use recent commits to infer project direction.

**The Concept:** Git history is architectural archaeology.

**Design Intent Before You Read the Code:** Recent commits show where maintainers are investing: dependencies, model defaults, rate limits, and UX polish.

**Find It In The Code:** Recent commits inspected with `git log --oneline -5`: `be62d2a`, `ecd581b`, `07963f8`, `1a3eeef`, `9b4d1b9`. Connect to `app/api/chat/route.ts:140-143` and `components/chat-panel.tsx:264`.

```text
07963f8 feat: add per-user daily limit for adaptive search mode (#836)
```

**The Aha Moment:** Commits explain why code paths exist, not only when they changed.

**Socratic Checkpoint:** 5 questions.

1. What does dependency bumping imply?
2. Why does changing the quick model matter?
3. Which commit explains adaptive rate limits?
4. Which commit maps to placeholder UX?
5. What commit theme would you expect next?

**How to Self-Grade:** Strong answers connect commit messages to code and product trade-offs.

**Connects To:** Mission 18 architecture decisions; Mission 25 writing docs.

