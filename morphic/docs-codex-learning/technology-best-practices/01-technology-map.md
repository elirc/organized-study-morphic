# Technology Map

## Next.js App Router

**Role in This Project:** Owns pages, layouts, dynamic routes, API routes, and server/client boundaries.

**Where It Appears:** `app/layout.tsx`, `app/page.tsx`, `app/search/page.tsx`, `app/search/[id]/page.tsx`, `app/api/**/route.ts`, `proxy.ts`.

**Why It Matters:** Most feature work starts by knowing whether code runs on the server, client, or HTTP route.

**What to Study First:** `app/layout.tsx:54-105`, `app/page.tsx:6-18`, `app/api/chat/route.ts:20-229`, `app/search/[id]/page.tsx:30-59`.

**Related Technologies:** React, Supabase, AI SDK, Drizzle.

**Learning Priority:** High. It defines the app skeleton.

## React

**Role in This Project:** Builds chat UI, controlled input, message rendering, side effects, context, and reusable components.

**Where It Appears:** `components/`, `hooks/`, `lib/contexts/`.

**Why It Matters:** The product experience is a streamed chat UI with many interaction states.

**What to Study First:** `components/chat.tsx:38-586`, `components/chat-panel.tsx:30-420`, `components/search-mode-selector.tsx:25-203`, `hooks/use-file-dropzone.ts:15-114`.

**Related Technologies:** AI SDK React hook, Tailwind, shadcn/Radix.

**Learning Priority:** High.

## TypeScript

**Role in This Project:** Defines props, message metadata, search modes, database row types, and tool contracts.

**Where It Appears:** Nearly all `.ts` and `.tsx` files.

**Why It Matters:** AI apps pass complex dynamic data; types keep boundaries understandable.

**What to Study First:** `lib/types/search.ts:1-2`, `components/chat-panel.tsx:30-56`, `lib/types/ai.ts:14-57`, `lib/db/schema.ts:70-300`.

**Related Technologies:** Zod, Drizzle, AI SDK.

**Learning Priority:** High.

## Vercel AI SDK

**Role in This Project:** Handles chat state, transport, model messages, tool loop agent, streaming, and UI message responses.

**Where It Appears:** `components/chat.tsx`, `lib/streaming/`, `lib/agents/researcher.ts`, `lib/tools/`.

**Why It Matters:** It is the core product engine.

**What to Study First:** `components/chat.tsx:89-217`, `lib/streaming/create-chat-stream-response.ts:33-235`, `lib/agents/researcher.ts:67-163`.

**Related Technologies:** Search providers, TypeScript, React.

**Learning Priority:** High.

## Drizzle ORM And PostgreSQL

**Role in This Project:** Persists chats, messages, message parts, and feedback with indexes, constraints, and RLS policies.

**Where It Appears:** `lib/db/schema.ts`, `lib/db/actions.ts`, `drizzle/`, `drizzle.config.ts`.

**Why It Matters:** Chat history, sharing, feedback, and message reconstruction depend on it.

**What to Study First:** `lib/db/schema.ts:25-300`, `lib/db/actions.ts:23-193`, `lib/db/__tests__/with-rls.test.ts:29-179`.

**Related Technologies:** SQL, RLS, server actions.

**Learning Priority:** High.

## Supabase Auth

**Role in This Project:** Provides user sessions when configured; local anonymous mode can bypass it.

**Where It Appears:** `lib/auth/get-current-user.ts`, `lib/supabase/`, `proxy.ts`, auth pages under `app/auth/`.

**Why It Matters:** Guest/auth behavior changes persistence, sidebar, rate limits, upload, and routing.

**What to Study First:** `lib/auth/get-current-user.ts:5-44`, `proxy.ts:5-53`, `app/layout.tsx:59-72`.

**Related Technologies:** Next.js middleware, RLS, Drizzle.

**Learning Priority:** Medium-High.

## Zod

**Role in This Project:** Defines AI tool input schemas, especially search.

**Where It Appears:** `lib/schema/search.tsx`, `lib/schema/question.ts`, `lib/schema/fetch.tsx`.

**Why It Matters:** It validates structured inputs to model tools. Public API routes should use more of this pattern.

**What to Study First:** `lib/schema/search.tsx:6-94`.

**Related Technologies:** AI SDK tools, TypeScript.

**Learning Priority:** Medium-High.

## Tailwind, shadcn, Radix

**Role in This Project:** Styles components and provides UI primitives.

**Where It Appears:** `components/ui/`, `components/chat-panel.tsx`, `components/search-mode-selector.tsx`, `app/globals.css`.

**Why It Matters:** Most UI changes should reuse existing primitives and utility classes.

**What to Study First:** `components/chat-panel.tsx:179-420`, `components/search-mode-selector.tsx:68-203`, `components/ui/button.tsx`.

**Related Technologies:** React.

**Learning Priority:** Medium.

## Vitest And React Testing Library

**Role in This Project:** Tests server actions, DB helpers, API routes, render behavior, rate limits, and utilities.

**Where It Appears:** `**/__tests__/**/*.test.ts`, `**/__tests__/**/*.test.tsx`, `vitest.config.mts`.

**Why It Matters:** Tests are the safest way to learn and change risky behavior.

**What to Study First:** `app/api/feedback/__tests__/route.test.ts:53-201`, `lib/actions/__tests__/chat.test.ts:32-240`, `components/__tests__/render-message.test.tsx:37-82`.

**Related Technologies:** Next.js, React, Drizzle.

**Learning Priority:** High.

## Docker, Redis, SearXNG, Config

**Role in This Project:** Provides local services and optional advanced search cache/provider behavior.

**Where It Appears:** `docker-compose.yaml`, `.env.local.example`, `app/api/advanced-search/route.ts`.

**Why It Matters:** Local setup and advanced search depend on external services.

**What to Study First:** `.env.local.example:18-68`, `docker-compose.yaml:4-75`, `app/api/advanced-search/route.ts:30-113`.

**Related Technologies:** Next.js API routes, Redis, search providers.

**Learning Priority:** Medium.

