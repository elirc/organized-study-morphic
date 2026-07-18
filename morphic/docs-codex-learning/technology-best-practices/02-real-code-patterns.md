# Real Code Patterns

## Technology: Next.js App Router

### Where It Is Used

Pages live under `app/**/page.tsx`; API routes under `app/api/**/route.ts`; global shell in `app/layout.tsx`; proxy middleware in `proxy.ts`.

### Important Files

- `app/layout.tsx` - global providers and shell.
- `app/page.tsx` - homepage server entry.
- `app/search/[id]/page.tsx` - dynamic route that loads saved chat.
- `app/api/chat/route.ts` - main backend route.

### Best-Practice Example 1: Server Entry Adapts Context

File path: `app/page.tsx:6-18`

```tsx
const userId = await getCurrentUserId() // server-side only
const modelSelectorData = await getModelSelectorData() // avoids client fetching at first render
return <Chat isGuest={!userId} modelSelectorData={modelSelectorData} />
```

What it does: resolves server context and passes simple props to the client. Junior engineers may miss that this keeps auth/model discovery off the client. Senior engineers notice that this preserves a clean server/client boundary.

### Best-Practice Example 2: Dynamic Route Loads Data Before UI

File path: `app/search/[id]/page.tsx:30-59`

This route loads a chat by ID, handles `notFound`, redirects private anonymous access, and hydrates `Chat` with saved messages. It keeps persistence lookup out of the client component.

### Weak or Risky Pattern, If Present

`app/api/chat/route.ts:29-52` manually destructures body fields. Improve with Zod validation to make malformed client errors consistent.

### Anti-Patterns to Avoid

- Do not move server-only auth calls into client components.
- Do not add route logic to UI components.
- Do not trust raw API JSON without validation.

### Interview Talking Points

- "The app uses Server Components for auth/model context and Client Components for chat interaction."
- "API routes act as policy boundaries before streaming."

### Mini Practice Tasks

- Trace `/search?q=hello` through `app/search/page.tsx`.
- Add a test plan for `/api/chat` invalid body behavior.

## Technology: React

### Where It Is Used

`components/chat.tsx`, `components/chat-panel.tsx`, `components/search-mode-selector.tsx`, `hooks/use-file-dropzone.ts`, `lib/contexts/*.tsx`.

### Important Files

- `components/chat.tsx` - state owner.
- `components/chat-panel.tsx` - controlled input and controls.
- `hooks/use-file-dropzone.ts` - custom hook side effects.

### Best-Practice Example 1: Controlled Input And Submission

File path: `components/chat-panel.tsx:254-289`

```tsx
<Textarea
  value={input} // parent-owned state
  disabled={isLoading || isToolInvocationInProgress()} // derived UI state
  onChange={handleInputChange}
  onKeyDown={e => { /* submit on Enter when safe */ }}
/>
```

This separates state ownership from input rendering. A junior may focus on the JSX; a mid-level engineer notices composition, keyboard behavior, IME handling, and disabled states.

### Best-Practice Example 2: Effect Cleanup

File path: `components/chat.tsx:263-297`

The copy-message effect registers a window listener and removes it on cleanup. It uses a ref to avoid stale messages.

### Weak or Risky Pattern, If Present

`components/chat.tsx:426-442` builds `parts: any[]`. This is practical because AI SDK parts are dynamic, but a typed helper would reduce mistakes.

### Anti-Patterns to Avoid

- Do not duplicate `chatId` state in children.
- Do not add global state when local state is enough.
- Do not ignore cleanup for window listeners.

### Interview Talking Points

- "The app keeps chat state high in `Chat` and passes controlled props to `ChatPanel`."
- "Search mode is a cookie-backed external store because server and client both need it."

### Mini Practice Tasks

- Add a small presentational metadata component.
- Write a component test for empty-submit behavior.

## Technology: TypeScript

### Where It Is Used

Throughout the repo: props, unions, schemas, DB types, AI SDK message contracts.

### Important Files

- `lib/types/search.ts`
- `lib/types/ai.ts`
- `components/chat-panel.tsx`
- `lib/db/schema.ts`
- `lib/utils/message-mapping.ts`

### Best-Practice Example 1: Union Type For Search Mode

File path: `lib/types/search.ts:1-2`

```ts
export type SearchMode = 'quick' | 'adaptive'
```

This protects UI, API, and agent configuration from arbitrary mode strings. Senior note: allowed-value lists still appear elsewhere and must be updated together.

### Best-Practice Example 2: Type Guards For Dynamic Parts

File path: `lib/utils/message-mapping.ts:67-102`

Type guards narrow unknown AI SDK parts before mapping them to DB rows. This is essential around dynamic model/tool output.

### Weak or Risky Pattern, If Present

`lib/types/ai.ts:35-42` allows dynamic tool keys with `any`. Acceptable for runtime tools, but risky if business logic starts depending on unvalidated output.

### Anti-Patterns to Avoid

- Do not cast raw request bodies directly into domain types.
- Do not add `any` far from a boundary.
- Do not update a union without updating validation lists.

### Interview Talking Points

- "The repo combines TypeScript static contracts with Zod runtime schemas for tool input."
- "Dynamic AI output forces escape hatches, so type guards and tests matter."

### Mini Practice Tasks

- Infer a type from a new Zod schema.
- Replace one local `any[]` with a small typed helper.

## Technology: Vercel AI SDK

### Where It Is Used

`components/chat.tsx`, `lib/streaming/create-chat-stream-response.ts`, `lib/streaming/create-ephemeral-chat-stream-response.ts`, `lib/agents/researcher.ts`, `lib/tools/search.ts`.

### Important Files

- `components/chat.tsx:89-217`
- `lib/streaming/create-chat-stream-response.ts:33-235`
- `lib/agents/researcher.ts:67-163`

### Best-Practice Example 1: Custom Transport Body

File path: `components/chat.tsx:100-129`

The app customizes AI SDK request body with `trigger`, `chatId`, `messageId`, `message`, `messages`, and `isNewChat`. This connects the SDK to Morphic's persistence model.

### Best-Practice Example 2: Stream Completion Persistence

File path: `lib/streaming/create-chat-stream-response.ts:186-223`

```ts
return result.toUIMessageStreamResponse({
  onFinish: async ({ responseMessage, isAborted }) => {
    if (isAborted || !responseMessage) return
    await persistStreamResults(responseMessage, chatId, userId, ...)
  }
})
```

The stream can render immediately while persistence happens at completion.

### Weak or Risky Pattern, If Present

Persistence failures are logged and swallowed in places (`lib/streaming/helpers/persist-stream-results.ts:81-97`). That protects UX but can hide data-loss incidents unless monitored.

### Anti-Patterns to Avoid

- Do not block streaming on nonessential side effects.
- Do not forget abort handling.
- Do not pass provider-specific message parts without compatibility handling.

### Interview Talking Points

- "AI SDK handles streaming and tool loops; Morphic adds policy, tools, and persistence."
- "Authenticated and guest streams share agent behavior but differ in persistence."

### Mini Practice Tasks

- Trace `regenerate-message`.
- Add a route test for the adaptive guest gate.

## Technology: Drizzle ORM And PostgreSQL

### Where It Is Used

`lib/db/schema.ts`, `lib/db/actions.ts`, `lib/db/with-rls.ts`, `drizzle/`.

### Important Files

- `lib/db/schema.ts:25-300`
- `lib/db/actions.ts:23-193`
- `lib/db/__tests__/with-rls.test.ts:29-179`

### Best-Practice Example 1: Schema With Indexes And RLS

File path: `lib/db/schema.ts:25-68`

Chats include indexes for user/time queries and RLS policies for own/private and public-readable access.

### Best-Practice Example 2: Atomic Message Upsert

File path: `lib/db/actions.ts:86-125`

The action inserts/updates a message, deletes existing parts, inserts new mapped parts, and returns the DB message inside a transaction/RLS context.

### Weak or Risky Pattern, If Present

The parts table has many nullable columns and JSON `any` fields (`lib/db/schema.ts:135-200`). This is flexible but makes invariants depend on mapping code and check constraints.

### Anti-Patterns to Avoid

- Do not write DB actions without RLS context.
- Do not mutate chats/messages without cache invalidation at action layer.
- Do not change schema without migrations.

### Interview Talking Points

- "Morphic normalizes AI message parts so rich streamed output can be persisted and reconstructed."
- "RLS is enforced at schema policy and transaction helper levels."

### Mini Practice Tasks

- Add a hypothetical `pinnedAt` field plan.
- Write a test for unauthorized chat mutation.

## Technology: Vitest

### Where It Is Used

Unit and integration-style tests under `__tests__`.

### Important Files

- `vitest.config.mts:1-22`
- `lib/actions/__tests__/chat.test.ts:32-240`
- `app/api/feedback/__tests__/route.test.ts:53-201`
- `components/__tests__/render-message.test.tsx:37-82`

### Best-Practice Example 1: Route Test By Calling Handler

File path: `app/api/feedback/__tests__/route.test.ts:70-87`

The test builds a `Request`, calls `POST`, and asserts response status/body. This is a good model for API route tests.

### Best-Practice Example 2: RLS Helper Contract

File path: `lib/db/__tests__/with-rls.test.ts:29-179`

Tests assert `set_config` is parameterized and optional RLS skips transactions without user ID.

### Weak or Risky Pattern, If Present

Main `/api/chat` route lacks direct tests despite centrality.

### Anti-Patterns to Avoid

- Do not hit real external providers in unit tests.
- Do not test implementation details when a route contract is enough.
- Do not use `bun test`; use `bun run test` from `package.json:15`.

### Interview Talking Points

- "Tests mock external modules and focus on behavior at boundaries."
- "The route test style is lightweight because handlers are plain exported functions."

### Mini Practice Tasks

- Add a failing test for invalid chat route body.
- Add a component test for a small UI behavior.

