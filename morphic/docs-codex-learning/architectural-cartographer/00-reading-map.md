# Reading Map

## One-Paragraph Mental Model

Morphic is a chat-centered research app. Server routes determine user/session context, client chat components gather input and render streamed message parts, `/api/chat` chooses auth/rate-limit/model/search behavior, the researcher agent calls search/fetch/todo tools, and authenticated streams are saved into normalized PostgreSQL tables so chat history can be loaded, shared, edited, and reviewed later.

## Top 10 Files To Read In Order

| Order | File | Why Read It Now | Before Reading | Explain After Reading |
| --- | --- | --- | --- | --- |
| 1 | `package.json:6-18` | Defines the dev workflow. | Know Bun runs scripts. | Which commands run dev, tests, typecheck, lint, build, migration. |
| 2 | `app/layout.tsx:54-105` | Shows global providers and page shell. | Server components can await data. | Why `UserProvider`, `SidebarProvider`, `ArtifactRoot`, and `Header` wrap every page. |
| 3 | `app/page.tsx:6-18` | Minimal homepage entry. | `getCurrentUserId` may return undefined. | How server context becomes chat props. |
| 4 | `app/search/[id]/page.tsx:30-59` | Existing chat route. | Dynamic route params are async in this app. | How saved messages reach the chat UI. |
| 5 | `components/chat.tsx:89-217` | Main client state and AI SDK transport. | Client components use hooks. | What body `/api/chat` receives. |
| 6 | `components/chat-panel.tsx:197-420` | Input, keyboard, upload, mode/model controls. | Controlled input pattern. | How submit, stop, new chat, and upload UI interact. |
| 7 | `app/api/chat/route.ts:20-229` | Main API boundary. | API routes return `Response`. | Auth, guest gating, rate limits, search mode, model selection, streaming branch. |
| 8 | `lib/streaming/create-chat-stream-response.ts:33-235` | Authenticated streaming and persistence. | AI SDK streams can call `onFinish`. | How messages are prepared, pruned, streamed, and saved. |
| 9 | `lib/agents/researcher.ts:67-163` | Agent configuration. | Tools are model-call capabilities. | Why quick/adaptive modes expose different tools. |
| 10 | `lib/db/schema.ts:25-300` | Durable data model. | Drizzle defines tables in TypeScript. | How chats, messages, parts, feedback, indexes, constraints, and RLS connect. |

## Three Most Important Data Flows

1. New authenticated chat:
   `app/page.tsx:6-18` -> `components/chat.tsx:421-452` -> `components/chat.tsx:100-129` -> `app/api/chat/route.ts:151-169` -> `lib/streaming/create-chat-stream-response.ts:196-223` -> `lib/streaming/helpers/persist-stream-results.ts:81-108` -> `lib/actions/chat.ts:179-200`.

2. Existing chat load:
   `app/search/[id]/page.tsx:30-59` -> `lib/actions/chat.ts:63-69` -> `lib/db/actions.ts:155-193` -> `lib/utils/message-mapping.ts:127-260` and later rebuild helpers in the same file.

3. Search tool execution:
   `components/search-mode-selector.tsx:25-56` -> cookie read in `app/api/chat/route.ts:87-96` -> tool selection in `lib/agents/researcher.ts:91-113` -> provider selection in `lib/tools/search.ts:53-137` -> optional advanced route `app/api/advanced-search/route.ts:137-181`.

## Pre-Reading Checklist

- [ ] Can I identify which files are Server Components and which are Client Components?
- [ ] Can I explain what `useChat` owns versus what Morphic customizes?
- [ ] Can I name the public API routes?
- [ ] Can I name the database tables?
- [ ] Can I explain what a message part is?
- [ ] Can I find where search mode is stored?
- [ ] Can I find where user identity is derived?
- [ ] Can I explain the guest versus authenticated branch?
- [ ] Can I find one test for server logic and one test for UI rendering?
- [ ] Can I explain which environment variables are required for local dev?

## Red Flags Checklist For Reviews

- [ ] Request body parsed without validation at an API boundary.
- [ ] New DB action that skips `withRLS` or ownership checks.
- [ ] Client state duplicated in multiple components without a clear owner.
- [ ] `any` added to message/tool/data contracts without a narrowing strategy.
- [ ] Search or upload feature that accepts URLs/files without size, type, domain, or ownership checks.
- [ ] New streaming behavior that throws after stream start and breaks persistence.
- [ ] Cache invalidation omitted after chat mutation.
- [ ] Search mode change that increases cost or latency without rate-limit updates.
- [ ] Component reads cookies/local browser state without a subscription strategy.
- [ ] Tests only cover happy paths while the change touches auth, persistence, or streaming.

