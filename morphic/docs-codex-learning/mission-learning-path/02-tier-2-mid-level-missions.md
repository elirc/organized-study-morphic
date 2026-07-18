# Tier 2: Mid-Level Missions

### Mission 9: State Has a Home and a Reason

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Identify why chat state lives in `Chat` and mode state lives in a cookie-backed selector.

**The Concept:** State should live where decisions are made, like search mode living where both UI and API can observe it.

**Design Intent Before You Read the Code:** `Chat` owns volatile interaction state; `SearchModeSelector` owns persisted user preference; API reads cookie to enforce server behavior.

**Find It In The Code:** `components/chat.tsx:55-87`, `components/search-mode-selector.tsx:25-56`, `app/api/chat/route.ts:87-96`.

```ts
// app/api/chat/route.ts:90-94
const searchMode: SearchMode =
  searchModeCookie && ['quick', 'adaptive'].includes(searchModeCookie)
    ? (searchModeCookie as SearchMode)
    : 'quick'
```

**The Aha Moment:** State ownership is about who must trust and act on the value.

**Socratic Checkpoint:** 5 questions.

1. Why is `input` local to `Chat` instead of a global store?
2. Why is search mode in a cookie?
3. What invalid cookie value behavior exists?
4. Which server branch depends on search mode?
5. What test would protect adding a new mode?

**How to Self-Grade:** Strong answers include UI, server, and agent consequences.

**Connects To:** Mission 13 boundary chain; Mission 21 performance.

### Mission 10: The Custom Hook or Reusable Abstraction Ecosystem

**Tier:** Mid-Level

**Time Estimate:** 40 minutes

**Goal:** Study a custom hook that wraps drag/drop side effects.

**The Concept:** A hook is a reusable mini-pipeline: events in, state and callbacks out.

**Design Intent Before You Read the Code:** `useFileDropzone` should own drag state, client validation, upload request, and uploaded-file state updates, while `Chat` decides whether guests can use it.

**Find It In The Code:** `hooks/use-file-dropzone.ts:15-114`, `components/chat.tsx:454-474`, `app/api/upload/route.ts:15-115`.

```ts
// hooks/use-file-dropzone.ts:65-80
await Promise.all(initialFiles.map(async uf => {
  const formData = new FormData()
  formData.append('file', uf.file)
  formData.append('chatId', chatId)
  const res = await fetch('/api/upload', { method: 'POST', body: formData })
}))
```

**The Aha Moment:** Hooks can hide side effects, so review them like small services.

**Socratic Checkpoint:** 5 questions.

1. What validation happens client-side?
2. What validation happens server-side?
3. How are upload errors represented?
4. Why does `Chat` replace handlers for guests?
5. What filename bug might exist between hook and route response?

**How to Self-Grade:** Strong answers catch `uploaded.name` in the hook versus `filename` in route response (`hooks/use-file-dropzone.ts:87-90`, `app/api/upload/route.ts:96-101`).

**Connects To:** Mission 19 bug prediction; Mission 22 security audit.

### Mission 11: Side Effects Are Promises to the System

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Inspect effects in chat and understand their cleanup and intent.

**The Concept:** A side effect is a promise to the browser: listen, scroll, copy, or dispatch, and clean up after yourself.

**Design Intent Before You Read the Code:** Effects in `Chat` coordinate keyboard shortcuts, message events, scroll position, and section navigation. Bad effects cause duplicate handlers or stale state.

**Find It In The Code:** `components/chat.tsx:254-364`.

```tsx
// components/chat.tsx:263-297
useEffect(() => {
  const handleCopyMessage = (e: Event) => { /* uses ref to avoid stale messages */ }
  window.addEventListener(SHORTCUT_EVENTS.copyMessage, handleCopyMessage)
  return () => window.removeEventListener(SHORTCUT_EVENTS.copyMessage, handleCopyMessage)
}, [])
```

**The Aha Moment:** Cleanup and stale-closure avoidance are as important as the effect body.

**Socratic Checkpoint:** 5 questions.

1. Why does this use `messagesRef`?
2. Why check `defaultPrevented`?
3. What effect dispatches `messages-changed`?
4. Which effects use `requestAnimationFrame`?
5. What duplicate listener bug is being avoided?

**How to Self-Grade:** Strong answers mention Next.js component caching comments in `components/chat.tsx:254-257`.

**Connects To:** Mission 19 bug prediction; Mission 15 diff review.

### Mission 12: The Full API or Module Contract

**Tier:** Mid-Level

**Time Estimate:** 50 minutes

**Goal:** Map `/api/chat` as a contract, not just a function.

**The Concept:** An API route is the border crossing where UI intent becomes server authority.

**Design Intent Before You Read the Code:** The route should validate request intent, authenticate, enforce limits, select model/search mode, choose persistence mode, and handle errors.

**Find It In The Code:** `app/api/chat/route.ts:20-229`.

```ts
// app/api/chat/route.ts:96-113
const selectedModel = await selectModel({ searchMode, cookieStore })
if (!selectedModel) return new Response('No enabled model is available', { status: 503 })
if (!isProviderEnabled(selectedModel.providerId)) return new Response(`Selected provider is not enabled ${selectedModel.providerId}`, { status: 404 })
```

**The Aha Moment:** The route is a policy coordinator, not the model-streaming implementation.

**Socratic Checkpoint:** 5 questions.

1. Which checks happen before model selection?
2. Which checks happen only for guests?
3. Which checks happen only for authenticated users?
4. What does the route do after creating the stream response?
5. What validation is missing?

**How to Self-Grade:** Strong answers list chronological phases and identify manual JSON validation as a gap.

**Connects To:** Mission 13 boundary chain; Mission 23 missing test.

### Mission 13: The Middleware, Pipeline, or Boundary Chain

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Understand auth/session middleware and RLS boundaries.

**The Concept:** Boundaries are layered locks: middleware refreshes session, auth resolves user, RLS constrains DB access.

**Design Intent Before You Read the Code:** Do not rely on one check. The app uses Supabase session updates, environment-controlled auth mode, route checks, and DB RLS helpers.

**Find It In The Code:** `proxy.ts:5-53`, `lib/auth/get-current-user.ts:18-44`, `lib/db/schema.ts:53-68`, `lib/db/__tests__/with-rls.test.ts:29-179`.

```ts
// lib/auth/get-current-user.ts:22-40
if (process.env.ENABLE_AUTH === 'false') {
  if (process.env.MORPHIC_CLOUD_DEPLOYMENT === 'true') throw new Error(...)
  return process.env.ANONYMOUS_USER_ID || 'anonymous-user'
}
```

**The Aha Moment:** Anonymous mode is still a user identity, but only for safe personal deployments.

**Socratic Checkpoint:** 5 questions.

1. What does `proxy.ts` do when Supabase is not configured?
2. Why is cloud auth disabling blocked?
3. How does RLS know the current user?
4. Which tests show SQL injection safety?
5. What should a new DB action use?

**How to Self-Grade:** Strong answers cite `set_config` expectations in `lib/db/__tests__/with-rls.test.ts:44-69`.

**Connects To:** Mission 22 security; Mission 18 architecture decisions.

### Mission 14: End-to-End Feature Trace

**Tier:** Mid-Level

**Time Estimate:** 60 minutes

**Goal:** Trace one prompt from submit to persisted assistant response.

**The Concept:** A feature trace is an answer engine for code: each layer answers "what happens next?"

**Design Intent Before You Read the Code:** You should see UI state, API body, route policy, stream preparation, agent execution, persistence, and UI update.

**Find It In The Code:** `components/chat.tsx:421-452`, `components/chat.tsx:100-129`, `app/api/chat/route.ts:20-229`, `lib/streaming/create-chat-stream-response.ts:112-223`, `lib/streaming/helpers/persist-stream-results.ts:24-108`.

```ts
// lib/streaming/create-chat-stream-response.ts:196-212
onFinish: async ({ responseMessage, isAborted }) => {
  if (isAborted || !responseMessage) return
  await persistStreamResults(responseMessage, chatId, userId, titlePromise, parentTraceId, searchMode, context.modelId, ...)
}
```

**The Aha Moment:** Persistence is a stream completion side effect, not the first thing the route does.

**Socratic Checkpoint:** 5 questions.

1. Where is the user message converted into parts?
2. Where is the model selected?
3. Where is context pruned?
4. Where are DB retries handled?
5. What happens differently for guests?

**How to Self-Grade:** Strong answers can draw the flow without looking and cite at least five files.

**Connects To:** Mission 21 performance; Mission 23 tests.

### Mission 15: Read the Diff Like an Engineer

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Practice reviewing a hypothetical search-mode change.

**The Concept:** A diff is not text; it is a behavior change crossing contracts.

**Design Intent Before You Read the Code:** A new mode touches type, UI, cookie validation, API validation, rate limits, agent tools, and tests.

**Find It In The Code:** `lib/types/search.ts:1-2`, `lib/config/search-modes.ts:15-31`, `components/search-mode-selector.tsx:38-56`, `app/api/chat/route.ts:87-96`, `lib/agents/researcher.ts:91-113`.

```ts
// components/search-mode-selector.tsx:39-43
if (savedMode && !['quick', 'adaptive'].includes(savedMode)) {
  setCookie('searchMode', 'quick') // new modes vanish unless this list changes
}
```

**The Aha Moment:** Review every list of allowed values, not just the type alias.

**Socratic Checkpoint:** 5 questions.

1. Which files need updating for a new mode?
2. Which file controls visible label/icon?
3. Which file controls agent tools?
4. Which tests would fail if only the UI changed?
5. What product risk does a new mode create?

**How to Self-Grade:** Strong answers mention cost, rate limits, and server validation.

**Connects To:** Mission 18 architecture decisions; Mission 24 git history.

### Mission 16: Composition Over Inheritance

**Tier:** Mid-Level

**Time Estimate:** 40 minutes

**Goal:** See how Morphic composes UI primitives and domain components.

**The Concept:** Composition lets the app assemble a search answer from reusable parts instead of building one giant object.

**Design Intent Before You Read the Code:** Components should compose primitives (`Button`, `DropdownMenu`, `HoverCard`) and domain components (`SearchModeSelector`, `ModelSelectorClient`) without inheritance chains.

**Find It In The Code:** `components/chat-panel.tsx:16-25`, `components/chat-panel.tsx:291-385`, `components/search-mode-selector.tsx:68-203`.

```tsx
// components/chat-panel.tsx:344-349
<SearchModeSelector />
{!isCloudDeployment && modelSelectorData && (
  <ModelSelectorClient data={modelSelectorData} />
)}
```

**The Aha Moment:** Component composition makes feature controls swappable and reviewable.

**Socratic Checkpoint:** 5 questions.

1. Which controls are imported into `ChatPanel`?
2. Which UI primitives does `SearchModeSelector` compose?
3. Why is model selector hidden on cloud deployment?
4. What props keep composition clean?
5. Where would inheritance make this harder?

**How to Self-Grade:** Strong answers describe dependencies flowing downward and behavior callbacks flowing upward.

**Connects To:** Mission 19 bug prediction; Mission 4 small UI component.

### Mission 17: TypeScript's Hidden Work

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Identify where TypeScript silently protects persistence and rendering.

**The Concept:** TypeScript does its best work before you notice: narrowing, inference, and discriminated shapes keep the answer engine coherent.

**Design Intent Before You Read the Code:** Message parts must be mapped safely. The code uses type guards, switch statements, Drizzle types, and tool-part shapes.

**Find It In The Code:** `lib/utils/message-mapping.ts:67-122`, `lib/utils/message-mapping.ts:127-260`, `lib/types/ai.ts:14-57`.

```ts
// lib/utils/message-mapping.ts:95-102
function isExtendedToolPart(part: any): part is ExtendedToolPart {
  return typeof part === 'object' && part !== null && typeof part.type === 'string' && part.type.startsWith('tool-')
}
```

**The Aha Moment:** Type guards turn unknown AI SDK parts into safer persistence decisions.

**Socratic Checkpoint:** 5 questions.

1. Which message parts have dedicated DB columns?
2. What happens to `step-result`?
3. How are dynamic tools identified?
4. Where does `any` enter?
5. What discriminated union would you design here?

**How to Self-Grade:** Strong answers explain both safety and remaining uncertainty around dynamic tools.

**Connects To:** Mission 23 tests; Mission 20 bug injection.

