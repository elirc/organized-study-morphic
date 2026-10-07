# 30-Day Technology Practice Plan

## Day 1

Focus area: scripts and setup. Files to read: `package.json:6-18`, `docs/CONFIGURATION.md` (upstream `.env.local.example` is absent from this snapshot). Concept: dev workflow. Exercise: write the commands from install to test. Self-check: why `bun run test`? Optional Codex prompt: "Quiz me on this repo's scripts."

## Day 2

Focus area: app shell. Files: `app/layout.tsx:54-105`. Concept: providers. Exercise: list every provider and its purpose. Self-check: what wraps `children`? Prompt: "Ask me to explain RootLayout."

## Day 3

Focus area: homepage entry. Files: `app/page.tsx:6-18`. Concept: server-to-client props. Exercise: explain `isGuest`. Self-check: what runs server-side?

## Day 4

Focus area: routes. Files: `app/search/page.tsx:11-33`, `app/search/[id]/page.tsx:30-59`. Concept: routing and data loading. Exercise: draw new query versus saved chat flow.

## Day 5

Focus area: chat state. Files: `components/chat.tsx:55-87`. Concept: state ownership. Exercise: identify each state variable's owner and consumer.

## Day 6

Focus area: AI SDK transport. Files: `components/chat.tsx:89-129`. Concept: custom request body. Exercise: write the JSON body for submit versus regenerate.

## Day 7

Focus area: controlled input. Files: `components/chat-panel.tsx:254-289`. Concept: input events. Exercise: explain IME handling.

## Day 8

Focus area: component contracts. Files: `components/chat-panel.tsx:30-78`, `components/chat.tsx:534-556`. Concept: props. Exercise: mark data props versus command props.

## Day 9

Focus area: small UI. Files: `components/search-results.tsx:13-128`. Concept: local display state. Exercise: plan Story 2 without coding.

## Day 10

Focus area: contexts. Files: `lib/contexts/chat-context.tsx:5-31`, `lib/contexts/user-context.tsx:5-18`. Concept: minimal global state. Exercise: decide when context is appropriate.

## Day 11

Focus area: search mode. Files: `components/search-mode-selector.tsx:25-203`, `lib/config/search-modes.ts:15-31`. Concept: cookie-backed state. Exercise: trace value from click to cookie.

## Day 12

Focus area: chat API. Files: `app/api/chat/route.ts:20-229`. Concept: API policy coordinator. Exercise: list every return status.

## Day 13

Focus area: auth. Files: `lib/auth/get-current-user.ts:5-44`, `proxy.ts:5-53`. Concept: auth modes. Exercise: explain anonymous mode safely.

## Day 14

Focus area: streaming. Files: `lib/streaming/create-chat-stream-response.ts:33-235`. Concept: stream lifecycle. Exercise: identify prepare, stream, finish, error phases.

## Day 15

Focus area: guest streaming. Files: `lib/streaming/create-ephemeral-chat-stream-response.ts:24-129`. Concept: no persistence branch. Exercise: compare with authenticated stream.

## Day 16

Focus area: agent tools. Files: `lib/agents/researcher.ts:67-163`. Concept: mode-based tools. Exercise: compare quick/adaptive tools and max steps.

## Day 17

Focus area: search tool. Files: `lib/tools/search.ts:20-173`. Concept: provider abstraction. Exercise: trace `general` versus `optimized`.

## Day 18

Focus area: Zod. Files: `lib/schema/search.tsx:6-94`. Concept: runtime schemas. Exercise: design chat request schema.

## Day 19

Focus area: DB schema. Files: `lib/db/schema.ts:25-300`. Concept: tables, indexes, constraints, RLS. Exercise: draw chats -> messages -> parts.

## Day 20

Focus area: DB actions. Files: `lib/db/actions.ts:23-193`. Concept: transactions and RLS. Exercise: explain `upsertMessage`.

## Day 21

Focus area: server actions. Files: `lib/actions/chat.ts:39-338`. Concept: authenticated wrappers and cache invalidation. Exercise: trace `deleteChat`.

## Day 22

Focus area: message mapping. Files: `lib/utils/message-mapping.ts:67-260`. Concept: dynamic data persistence. Exercise: map one text part and one tool part.

## Day 23

Focus area: upload. Files: `hooks/use-file-dropzone.ts:15-114`, `app/api/upload/route.ts:15-115`. Concept: client/server validation. Exercise: find response shape mismatch.

## Day 24

Focus area: advanced search. Files: `app/api/advanced-search/route.ts:137-311`. Concept: cache and crawl flow. Exercise: identify performance risks.

## Day 25

Focus area: tests. Files: `app/api/feedback/__tests__/route.test.ts:53-201`. Concept: route testing. Exercise: outline `/api/chat` tests.

## Day 26

Focus area: RLS tests. Files: `lib/db/__tests__/with-rls.test.ts:29-179`. Concept: security testing. Exercise: explain parameterized `set_config`.

## Day 27

Focus area: component tests. Files: `components/__tests__/render-message.test.tsx:37-82`. Concept: rendering contracts. Exercise: design a `ChatPanel` test.

## Day 28

Focus area: code review. Files: use Story 6 relevant files. Concept: diff reading. Exercise: review hypothetical Zod validation diff.

## Day 29

Focus area: feature planning. Files: Story 8 relevant files. Concept: schema-to-UI planning. Exercise: write a pinning implementation plan.

## Day 30

Focus area: system explanation. Files: all top 10 from `architectural-cartographer/00-reading-map.md`. Concept: interview narrative. Exercise: explain Morphic in 3 minutes, then in 10 minutes. Prompt: "Ask me senior follow-up questions about this architecture."

