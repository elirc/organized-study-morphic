# Technology Journal

## Technologies Identified

- Next.js App Router: `app/layout.tsx`, `app/page.tsx`, `app/api/**/route.ts`.
- React: client components, hooks, context.
- TypeScript: props, unions, Drizzle inferred types, tool types.
- Vercel AI SDK: `useChat`, `DefaultChatTransport`, `ToolLoopAgent`, `UIMessage` streaming.
- Drizzle ORM/PostgreSQL: schema, actions, migrations, RLS policies.
- Supabase auth: server client and middleware.
- Zod: AI tool schemas.
- Tailwind CSS/shadcn/Radix: UI primitives and styling.
- Vitest/React Testing Library: unit and route/component tests.
- Redis/SearXNG/search providers: advanced search caching and provider abstraction.
- Docker/config: local stack and env management.

## Prioritization

Highest priority: Next.js, React, TypeScript, AI SDK, Drizzle, API routes, testing. These are on the main chat path.

Medium priority: Supabase, Redis/search providers, Docker, Tailwind. They matter for deployment, auth, search, and UI work.

Absent technologies: Express, NestJS, Prisma, TypeORM, GraphQL, React Query, Zustand, Redux, SWR. Their absence is reasonable because the repo uses App Router/API routes, server actions, AI SDK hooks, context, cookies, and Drizzle.

## Strong Patterns

- Server-to-client prop boundary in `app/page.tsx:6-18`.
- AI SDK transport customization in `components/chat.tsx:89-129`.
- Search tool schema and provider abstraction in `lib/schema/search.tsx:6-94` and `lib/tools/search.ts:20-173`.
- RLS helper tests in `lib/db/__tests__/with-rls.test.ts:29-179`.
- Route validation tests in `app/api/feedback/__tests__/route.test.ts:53-201`.

## Weak Or Inconsistent Patterns

- Manual API validation in `/api/chat` (`app/api/chat/route.ts:29-52`).
- Dynamic `any` in message/tool contracts (`lib/types/ai.ts:19-42`).
- Dense advanced search route (`app/api/advanced-search/route.ts:137-659`).
- Upload route should verify chat ownership (`app/api/upload/route.ts:41-62`).

## What Juniors Might Misunderstand

- `useChat` does not mean there is no backend; it prepares a request to `/api/chat`.
- `UIMessage` is made of parts, not just text.
- Server actions and DB actions are different layers.
- Anonymous mode is a deliberate local deployment mode, not "no user."

## What Mid-Level And Senior Engineers Should Notice

- The architecture is optimized for streaming UX and rich AI message persistence.
- The highest-risk boundaries are public route bodies, crawler URLs, upload ownership, and dynamic tool typing.
- Tests are present and useful, but the main chat route needs more direct coverage.

