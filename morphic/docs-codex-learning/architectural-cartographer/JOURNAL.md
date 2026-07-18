# Architectural Inspection Journal

## First-Pass Mental Model

Morphic is a full-stack Next.js answer engine. Server components decide who the user is and which chat to load. Client components manage the chat interaction. The AI SDK sends structured messages to `/api/chat`. The API route checks auth, rate limits, search mode, and model availability, then delegates to streaming helpers. Authenticated streams are persisted to PostgreSQL through Drizzle; guest streams are ephemeral. Search is exposed as AI tools and can use Tavily, SearXNG, Exa, Firecrawl, or Brave depending on configuration.

Evidence:

- `package.json:6-18` defines Bun/Next/Vitest scripts.
- `app/layout.tsx:54-105` wraps the app with theme, user, sidebar, header, artifact, toaster, and analytics providers.
- `app/page.tsx:6-18` is the homepage chat entry.
- `components/chat.tsx:89-217` owns `useChat` transport configuration.
- `app/api/chat/route.ts:20-229` is the main backend route.
- `lib/db/schema.ts:25-300` defines chats, messages, parts, and feedback.

## Important Discoveries

- The chat route has two distinct persistence models. Authenticated users use `createChatStreamResponse`, while guests use `createEphemeralChatStreamResponse` (`app/api/chat/route.ts:151-169`).
- Search mode is stored in a cookie in the browser (`components/search-mode-selector.tsx:25-56`) and read by the server route (`app/api/chat/route.ts:87-96`).
- Persistence stores AI SDK message parts in a normalized `parts` table rather than one JSON blob (`lib/db/schema.ts:122-258`), but many tool columns are typed as `any`.
- Row-level security is modeled both in Drizzle schema policies (`lib/db/schema.ts:53-68`, `lib/db/schema.ts:91-118`) and runtime transaction helpers (`lib/db/actions.ts:16-18`, `lib/db/__tests__/with-rls.test.ts:29-179`).
- File upload exists but is gated to authenticated users and R2/S3-compatible configuration (`app/api/upload/route.ts:15-70`).
- The project has meaningful unit tests for actions, rendering, feedback routes, RLS helpers, streaming helpers, rate limits, and render parsing.

## Teaching Anchors Chosen

The main teaching spine is the chat feature because it crosses almost every layer:

1. `app/page.tsx:6-18` for server component entry.
2. `components/chat.tsx:38-586` for client state, AI SDK transport, message grouping, submit handling, drag/drop, and error UX.
3. `components/chat-panel.tsx:30-420` for input UI, keyboard behavior, file upload trigger, search mode selector, and model selector.
4. `app/api/chat/route.ts:20-229` for API boundary behavior.
5. `lib/streaming/create-chat-stream-response.ts:33-235` for model preparation, tool agent streaming, metadata, and persistence callback.
6. `lib/agents/researcher.ts:67-163` for tool selection by search mode.
7. `lib/tools/search.ts:20-173` for provider-backed search as an AI tool.
8. `lib/db/schema.ts:25-300` and `lib/db/actions.ts:23-193` for persistence.

## Where A Junior Might Get Confused

- The app mixes Server Components and Client Components. `app/page.tsx:6-18` can `await` user/model data, while `components/chat.tsx:1-586` must use hooks and browser APIs.
- `useChat` hides a lot of behavior. The important custom piece is `prepareSendMessagesRequest` in `components/chat.tsx:100-129`.
- Database writes are not in the route directly. They happen after streaming finishes through `persistStreamResults` (`lib/streaming/create-chat-stream-response.ts:196-223`).
- `UIMessage` parts are more important than a single `content` string. Persistence maps parts into DB rows (`lib/utils/message-mapping.ts:127-260`).

## Where A Mid-Level Engineer Should Slow Down

- Auth and authorization are layered: `getCurrentUserId` can return an anonymous ID when `ENABLE_AUTH=false` (`lib/auth/get-current-user.ts:18-44`), API route gates guests (`app/api/chat/route.ts:68-85`), and chat loading checks ownership (`lib/db/actions.ts:182-185`).
- Search mode changes tools and step counts (`lib/agents/researcher.ts:91-113`), so UI changes to modes can affect cost, latency, and product behavior.
- Advanced search route does network crawling, Redis caching, relevance scoring, and HTML parsing in one route file (`app/api/advanced-search/route.ts:137-659`). That is powerful but dense.

## Where A Senior Engineer Should Be Skeptical

- Several request bodies are parsed manually instead of with Zod at the API boundary, such as `app/api/chat/route.ts:29-52` and `app/api/feedback/route.ts:7-24`.
- `any` appears in message metadata, tool inputs/outputs, and component callbacks (`lib/types/ai.ts:19-42`, `lib/db/schema.ts:173-194`, `components/chat.tsx:421-442`).
- Advanced search can crawl many pages with `Promise.all` (`app/api/advanced-search/route.ts:258-263`), which deserves concurrency and SSRF scrutiny.
- Upload keys include user and chat IDs, but the route does not verify chat ownership for the provided `chatId` (`app/api/upload/route.ts:41-62`).

## Missing Or Weak Patterns

- No tRPC, GraphQL, Express, NestJS, Prisma, TypeORM, Redux, Zustand, SWR, or React Query were found. Their absence is acceptable for a Next.js App Router app that uses server actions, API routes, and AI SDK hooks.
- Runtime validation exists for AI tool schemas (`lib/schema/search.tsx:6-94`) but is not consistently used at public API request boundaries.
- Setup information exists across README, `.env.local.example`, Docker, and package scripts, but the one-stop local setup path has to be reconstructed.

## How To Use Checkpoints

For every checkpoint, write a short answer with four parts:

1. Files involved.
2. Data shape crossing the boundary.
3. Failure modes.
4. What you would test.

Then compare your answer to the rubric. If you cannot cite a line range, reopen the file.

