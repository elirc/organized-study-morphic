# User Story Journal

## Why These Stories Were Chosen

The stories use Morphic's real product surface: asking questions, choosing search depth, uploading files, viewing sources, giving feedback, loading chat history, and keeping AI responses reliable. They are not toy tasks. Each one maps to existing files and patterns.

## Safest First Stories

Stories 1-3 are safest because they mostly touch presentational UI and small validation behavior: `components/chat-panel.tsx`, `components/search-results.tsx`, and `components/search-mode-selector.tsx`.

## Deeper Architecture Stories

Stories 6-10 require API boundaries, persistence, or security reasoning. They touch `app/api/**/route.ts`, `lib/actions/chat.ts`, `lib/db/actions.ts`, `lib/db/schema.ts`, `lib/streaming/**`, and tests.

## Repeated Files And Patterns

- Controlled UI props: `components/chat-panel.tsx:30-78`.
- Chat state and transport: `components/chat.tsx:89-217`.
- API validation style: `app/api/feedback/route.ts:7-24`.
- Tests for route behavior: `app/api/feedback/__tests__/route.test.ts:53-201`.
- DB actions and cache invalidation: `lib/actions/chat.ts:39-69`, `lib/actions/chat.ts:179-200`.
- RLS/authorization patterns: `lib/db/actions.ts:23-193`, `lib/db/__tests__/with-rls.test.ts:29-179`.

## What A Senior Would Watch

- Whether a story changes guest/authenticated behavior.
- Whether persisted data needs a migration.
- Whether API input is validated at the boundary.
- Whether cache tags are revalidated.
- Whether AI SDK message parts remain reconstructable.
- Whether new UI state belongs locally, in context, in a cookie, or on the server.

