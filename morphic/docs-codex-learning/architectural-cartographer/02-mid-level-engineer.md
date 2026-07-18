# Mid-Level Engineer Guide

## Architecture Diagram

```text
Browser
  |
  | React client state, AI SDK useChat
  v
components/chat.tsx + components/chat-panel.tsx
  |
  | POST /api/chat with trigger, chatId, message/messages, messageId, isNewChat
  v
app/api/chat/route.ts
  |
  | auth, guest checks, rate limits, cookie searchMode, model selection
  v
lib/streaming/create-chat-stream-response.ts OR create-ephemeral-chat-stream-response.ts
  |
  | prepare/prune/truncate messages; researcher agent streams
  v
lib/agents/researcher.ts -> lib/tools/search.ts -> provider or /api/advanced-search
  |
  | onFinish for authenticated users
  v
lib/streaming/helpers/persist-stream-results.ts
  |
  v
lib/actions/chat.ts -> lib/db/actions.ts -> PostgreSQL tables in lib/db/schema.ts
```

## Type System Deep Dive

Morphic's type system is strongest around named domain boundaries and weakest around dynamic AI parts.

Strong examples:

- `SearchMode` is a two-value union (`lib/types/search.ts:1-2`).
- Search tool input is a Zod schema with defaults and transforms (`lib/schema/search.tsx:6-46`).
- Drizzle exports inferred row types (`lib/db/schema.ts:70`, `lib/db/schema.ts:120`, `lib/db/schema.ts:258`, `lib/db/schema.ts:300`).
- `ChatPanelProps` names every interaction the panel can perform (`components/chat-panel.tsx:30-56`).

Weak examples:

- Message metadata allows arbitrary keys (`lib/types/ai.ts:14-20`).
- Tool registry allows dynamic keys with `any` (`lib/types/ai.ts:35-42`).
- Chat submit builds `parts: any[]` (`components/chat.tsx:426-442`).

The senior lesson: dynamic AI systems often need escape hatches, but each escape hatch should be close to a boundary and paired with validation or type guards. This repo uses type guards in message mapping (`lib/utils/message-mapping.ts:67-102`) but could tighten request bodies.

## State Management Deep Dive

There is no Redux, Zustand, React Query, or SWR. State lives where it is needed:

- Global-ish boolean user context: `lib/contexts/user-context.tsx:5-18`.
- Chat send function context: `lib/contexts/chat-context.tsx:5-31`.
- Search mode cookie external store: `components/search-mode-selector.tsx:25-56`.
- Main chat state: `components/chat.tsx:55-87`.
- AI SDK state: `useChat` in `components/chat.tsx:89-217`.

Important piece of state: `searchMode`.

1. Browser reads cookie with `useSyncExternalStore` (`components/search-mode-selector.tsx:25-33`).
2. Browser writes cookie on selection (`components/search-mode-selector.tsx:46-56`).
3. API route reads cookie (`app/api/chat/route.ts:87-96`).
4. API route gates adaptive mode for cloud guests (`app/api/chat/route.ts:115-134`).
5. Researcher switches tools and max steps (`lib/agents/researcher.ts:91-113`).

## API Contract Map

| Route | Contract | Boundary Notes |
| --- | --- | --- |
| `POST /api/chat` | Body from AI SDK transport: `trigger`, `chatId`, `messageId`, `message`, `messages`, `isNewChat` (`components/chat.tsx:100-129`) | Manual validation only (`app/api/chat/route.ts:29-52`). |
| `POST /api/upload` | `multipart/form-data` with `file` and `chatId` (`app/api/upload/route.ts:33-44`) | Auth, type, size, storage config checks (`app/api/upload/route.ts:15-62`). |
| `POST /api/feedback` | JSON with `traceId`, `score`, optional `comment`, `messageId` (`app/api/feedback/route.ts:7-24`) | Tested for validation (`app/api/feedback/__tests__/route.test.ts:141-201`). |
| `POST /api/advanced-search` | JSON search request (`app/api/advanced-search/route.ts:137-160`) | Dense route; lacks explicit Zod boundary. |

## Component Interaction Map

`Chat` owns state and passes command props downward:

- `ChatMessages` receives message sections, stream status, reload/edit callbacks, and tool results (`components/chat.tsx:488-533`).
- `ChatPanel` receives input, submit handler, upload state, model/search controls, stop/new chat commands, and section navigation (`components/chat.tsx:534-556`).
- `DragOverlay` receives drag visibility (`components/chat.tsx:557`).
- `ErrorModal` receives typed error state and retry/auth-close callbacks (`components/chat.tsx:558-582`).

## Full-Stack Feature Trace: Submit A New Chat

1. User types in `ChatPanel` controlled textarea (`components/chat-panel.tsx:254-289`).
2. Form submit calls parent `handleSubmit` after model availability check (`components/chat-panel.tsx:197-208`).
3. `Chat.onSubmit` builds AI SDK parts and calls `sendMessage` (`components/chat.tsx:421-452`).
4. `useChat` transport prepares the API body (`components/chat.tsx:100-129`).
5. `POST /api/chat` validates trigger requirements (`app/api/chat/route.ts:37-52`).
6. Route derives user, guest status, rate limits, search mode, and model (`app/api/chat/route.ts:57-144`).
7. Authenticated route calls `createChatStreamResponse` (`app/api/chat/route.ts:159-169`).
8. Stream helper verifies chat access for existing chats (`lib/streaming/create-chat-stream-response.ts:56-73`).
9. Helper prepares messages, configures researcher, prunes/truncates context, and streams (`lib/streaming/create-chat-stream-response.ts:112-186`).
10. On finish, it persists response message (`lib/streaming/create-chat-stream-response.ts:196-223`).
11. Persistence attaches metadata, awaits initial save, retries DB write, and updates title (`lib/streaming/helpers/persist-stream-results.ts:24-108`).
12. Chat cache is invalidated (`app/api/chat/route.ts:208-212`).

## Diff Reading Exercise

Hypothetical change: "Add a `deep` search mode."

Review order:

1. Type contract: `lib/types/search.ts:1-2` must include the new value.
2. UI config: `lib/config/search-modes.ts:15-31` must add label/icon/description.
3. Cookie handling: `components/search-mode-selector.tsx:29-43` currently only accepts `quick` and `adaptive`.
4. API route validation: `app/api/chat/route.ts:90-94` currently only accepts `quick` and `adaptive`.
5. Rate limit behavior: `app/api/chat/route.ts:136-143` only applies adaptive-specific limit.
6. Agent behavior: `lib/agents/researcher.ts:91-113` must decide prompt, tools, max steps.
7. Tests: add model-selection/search-mode tests around the new branch.

Good review question: Does the new mode change cost, latency, or provider requirements?

## Non-Obvious Patterns

- Streaming persistence happens in `onFinish`, not before response. This keeps UI responsive but means persistence failures must not break the stream (`lib/streaming/helpers/persist-stream-results.ts:81-97`).
- New chat optimization skips loading a chat on first turn (`lib/streaming/create-chat-stream-response.ts:56-73`), reducing DB work.
- Message parts are normalized to support UI reconstruction and tool outputs (`lib/db/schema.ts:122-258`).
- Cache tags are chat-specific (`lib/actions/chat.ts:16-34`, `app/api/chat/route.ts:208-212`).

## Mid-Level Socratic Checkpoint

1. Why does `Chat` own `chatId` instead of `ChatPanel`?
2. What changes when the user is a guest?
3. Where should request validation ideally live for `/api/chat`?
4. Why does search mode need both client and server validation?
5. What could go wrong if `persistStreamResults` threw after streaming started?
6. Why normalize message parts instead of storing only raw Markdown?
7. Which existing test style would you copy for a new API route?
8. What cache tag would you revalidate after deleting a chat?

## How To Self-Grade

Strong answers mention ownership, trust boundaries, cost/performance, cache invalidation, and tests. They cite `components/chat.tsx:55-87`, `app/api/chat/route.ts:68-85`, `app/api/chat/route.ts:87-96`, `lib/streaming/helpers/persist-stream-results.ts:81-97`, `lib/db/schema.ts:122-258`, and `app/api/feedback/__tests__/route.test.ts:53-201`.

