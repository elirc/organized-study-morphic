# Tier 1: Junior Missions

### Mission 1: The App's Heartbeat

**Tier:** Junior

**Time Estimate:** 25 minutes

**Goal:** Identify how the app starts, what command runs it, and what renders first.

**The Concept:** The heartbeat is the pulse that proves the app is alive: scripts start the server, layout wraps the body, and the page renders the chat.

**Design Intent Before You Read the Code:** `package.json` owns commands, `app/layout.tsx` owns global shell, and `app/page.tsx` owns the homepage entry. If these are misunderstood, every feature trace starts in the wrong place.

**Find It In The Code:** Open `package.json:6-18`, `app/layout.tsx:54-105`, and `app/page.tsx:6-18`.

```tsx
// app/page.tsx:6-18
const userId = await getCurrentUserId() // server-side identity
const modelSelectorData = await getModelSelectorData() // server-side model availability
return <Chat isGuest={!userId} modelSelectorData={modelSelectorData} /> // client experience begins
```

**The Aha Moment:** The homepage is not the chat logic; it is a server-side adapter that passes context into the chat.

**Socratic Checkpoint:** 5 questions.

1. Which script starts dev mode?
2. Which file wraps every page?
3. Why can `app/page.tsx` use `await`?
4. What props does the homepage pass into `Chat`?
5. What would break if `getCurrentUserId` started throwing?

**How to Self-Grade:** Strong answers cite `package.json:7`, `app/layout.tsx:81-101`, and `app/page.tsx:6-18`, and explain server versus client responsibility.

**Connects To:** Mission 2 maps folders; Mission 8 maps routing.

### Mission 2: The Folder Mental Map

**Tier:** Junior

**Time Estimate:** 30 minutes

**Goal:** Build a folder-level map of frontend, backend, shared logic, and persistence.

**The Concept:** A codebase is like Morphic's search process: you need a ranked map before opening every result.

**Design Intent Before You Read the Code:** Top-level folders separate routes (`app`), UI (`components`), reusable hooks (`hooks`), domain logic (`lib`), and migrations (`drizzle`). Poor folder reading leads to random-walk debugging.

**Find It In The Code:** Use `rg --files`, then inspect `app/`, `components/`, `hooks/`, `lib/`, and `drizzle/`. Anchor files: `app/api/chat/route.ts:20-229`, `components/chat.tsx:38-586`, `lib/db/schema.ts:25-300`.

```text
app/        -> pages and HTTP routes
components/ -> chat UI and reusable UI
lib/        -> agents, actions, auth, DB, streaming, tools, types
drizzle/    -> database migrations
```

**The Aha Moment:** Morphic is organized by runtime boundary first, then feature responsibility.

**Socratic Checkpoint:** 5 questions.

1. Where do HTTP routes live?
2. Where does Drizzle schema live?
3. Where do AI tools live?
4. Where are reusable UI primitives?
5. Which folder would you inspect for a chat history bug?

**How to Self-Grade:** Strong answers name folders and one exact file from each. They explain why `lib/actions/chat.ts` and `lib/db/actions.ts` are different layers.

**Connects To:** Mission 3 contracts; Mission 5 backend route.

### Mission 3: TypeScript Is a Contract

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** See how TypeScript limits valid values and documents boundaries.

**The Concept:** Types are the form fields of the codebase: they tell you what data is allowed before the app submits work downstream.

**Design Intent Before You Read the Code:** Small unions like `SearchMode`, prop interfaces like `ChatPanelProps`, and inferred DB types prevent accidental misuse. Missing types allow subtle bugs.

**Find It In The Code:** Open `lib/types/search.ts:1-2`, `components/chat-panel.tsx:30-56`, `lib/db/schema.ts:70-300`, and `lib/schema/search.tsx:6-94`.

```ts
// lib/types/search.ts:1-2
export type SearchMode = 'quick' | 'adaptive' // only these cookie/API values are intended
```

**The Aha Moment:** A tiny union can protect UI, API, and agent behavior at once.

**Socratic Checkpoint:** 5 questions.

1. What values can `SearchMode` hold?
2. Which prop tells `ChatPanel` whether the user is a guest?
3. Where does Drizzle infer the `Chat` type?
4. Which schema validates search tool inputs?
5. Where does the project still use `any`?

**How to Self-Grade:** Strong answers include both good and weak type examples, citing `lib/types/ai.ts:35-42` for dynamic escape hatches.

**Connects To:** Mission 6 props as contracts; Mission 17 hidden TypeScript work.

### Mission 4: Your First UI Component

**Tier:** Junior

**Time Estimate:** 30 minutes

**Goal:** Understand a small component with local state and props.

**The Concept:** A simple UI component should be like one search result card: it receives data, manages tiny display state, and does not own the whole system.

**Design Intent Before You Read the Code:** `SearchResults` should render given results and expose local "view more" state. It should not fetch data or mutate global app state.

**Find It In The Code:** Open `components/search-results.tsx:13-128`.

```tsx
// components/search-results.tsx:18-31
export function SearchResults({ results, displayMode = 'grid' }: SearchResultsProps) {
  const [showAllResults, setShowAllResults] = useState(false) // local display state only
  const displayedGridResults = showAllResults ? results : results.slice(0, 3)
}
```

**The Aha Moment:** Good small components make ownership obvious.

**Socratic Checkpoint:** 5 questions.

1. What data comes in through props?
2. What state is local?
3. What changes when display mode is `list`?
4. Does this component fetch?
5. What could break if `result.url` is invalid?

**How to Self-Grade:** Strong answers mention `new URL(result.url)` in `components/search-results.tsx:49-67` and the lack of error guarding around malformed URLs.

**Connects To:** Mission 6 contracts; Mission 16 composition.

### Mission 5: Your First Backend Route

**Tier:** Junior

**Time Estimate:** 40 minutes

**Goal:** Understand a public API route and its validation.

**The Concept:** A backend route is Morphic's checkpoint before a request becomes action.

**Design Intent Before You Read the Code:** Routes should parse input, reject invalid requests, call focused logic, and return clear responses.

**Find It In The Code:** Open `app/api/feedback/route.ts:7-80` and tests in `app/api/feedback/__tests__/route.test.ts:53-201`.

```ts
// app/api/feedback/route.ts:12-24
if (!traceId) return new Response('traceId is required', { status: 400 })
if (score === undefined || (score !== 1 && score !== -1)) {
  return new Response('score must be 1 (good) or -1 (bad)', { status: 400 })
}
```

**The Aha Moment:** Tests make route contracts more readable than comments.

**Socratic Checkpoint:** 5 questions.

1. What fields does the route require?
2. What does it do when tracing is disabled?
3. Why does DB feedback failure not fail the route?
4. Which tests cover invalid input?
5. What validation library could improve this route?

**How to Self-Grade:** Strong answers cite route lines and matching tests, especially `app/api/feedback/__tests__/route.test.ts:141-201`.

**Connects To:** Mission 12 API contract; Mission 23 missing tests.

### Mission 6: Props, Inputs, Or Parameters Are Contracts

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Trace how component props define responsibilities.

**The Concept:** Props are like API request bodies inside the frontend: the caller promises to provide what the callee needs.

**Design Intent Before You Read the Code:** `Chat` owns state; `ChatPanel` receives state and command callbacks. If `ChatPanel` starts owning chat persistence, responsibilities blur.

**Find It In The Code:** Open `components/chat-panel.tsx:30-78` and `components/chat.tsx:534-556`.

```tsx
// components/chat.tsx:534-556
<ChatPanel
  chatId={chatId} // identity
  input={input} // controlled value
  handleSubmit={onSubmit} // parent behavior
  uploadedFiles={uploadedFiles} // shared state
  setUploadedFiles={setUploadedFiles} // controlled mutation
/>
```

**The Aha Moment:** Prop lists reveal ownership boundaries before you read component internals.

**Socratic Checkpoint:** 5 questions.

1. Which component owns `input`?
2. Which component owns `chatId`?
3. Why is `stop` passed down?
4. What prop gates guest upload behavior?
5. Which prop supports message navigation dots?

**How to Self-Grade:** Strong answers cite both the interface and call site, not one or the other.

**Connects To:** Mission 9 state ownership; Mission 14 full trace.

### Mission 7: Following Data Into The App

**Tier:** Junior

**Time Estimate:** 45 minutes

**Goal:** Follow a submitted prompt from UI into the API request.

**The Concept:** A prompt is like a search query entering Morphic's pipeline: it starts as text, becomes message parts, then becomes an API body.

**Design Intent Before You Read the Code:** Input text should be transformed once, cleared after send, and encoded in the shape the server expects.

**Find It In The Code:** Open `components/chat-panel.tsx:197-208`, `components/chat.tsx:421-452`, and `components/chat.tsx:100-129`.

```tsx
// components/chat.tsx:426-442
const parts: any[] = []
if (input.trim()) parts.push({ type: 'text', text: input }) // UI text becomes AI SDK part
uploaded.forEach(f => parts.push({ type: 'file', url: f.url!, filename: f.name!, mediaType: f.file.type }))
sendMessage({ role: 'user', parts }) // AI SDK owns the next hop
```

**The Aha Moment:** The app does not send raw input text; it sends typed message parts.

**Socratic Checkpoint:** 5 questions.

1. Where is empty input blocked?
2. Where are file parts added?
3. When is input cleared?
4. Which function prepares the HTTP body?
5. Why do guests include `messages` in the body?

**How to Self-Grade:** Strong answers connect `onSubmit` to `prepareSendMessagesRequest` and explain guest ephemeral context.

**Connects To:** Mission 14 end-to-end trace; Mission 17 TypeScript hidden work.

### Mission 8: Navigation, Routing, Or Execution Flow Is The App's Skeleton

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Understand how new searches and existing chats route through the app.

**The Concept:** Routing is the skeleton: it decides which body of data the chat UI wakes up with.

**Design Intent Before You Read the Code:** `/search?q=` creates a new chat from a query; `/search/[id]` loads persisted chat history.

**Find It In The Code:** Open `app/search/page.tsx:11-33`, `app/search/[id]/page.tsx:30-59`, and `components/chat.tsx:446-450`.

```tsx
// app/search/[id]/page.tsx:36-53
const chat = await loadChat(id, userId) // persistence lookup
if (!chat) notFound()
return <Chat id={id} savedMessages={chat.messages} isGuest={!userId} /> // hydrate UI with history
```

**The Aha Moment:** The same `Chat` component supports homepage, query-started search, and persisted chat routes.

**Socratic Checkpoint:** 5 questions.

1. What happens when `/search` has no `q`?
2. Which route loads saved messages?
3. Where does a new chat push the browser URL?
4. Why does `/search/[id]` call `notFound`?
5. Why might anonymous users redirect to login?

**How to Self-Grade:** Strong answers mention `redirect('/')`, `notFound()`, `savedMessages`, and `window.history.pushState`.

**Connects To:** Mission 14 feature trace; Mission 18 architecture decisions.

