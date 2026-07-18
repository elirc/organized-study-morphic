# Cross-Cutting Engineering Lessons

## Separation Of Concerns

Concept: each layer owns one kind of decision.

Where it appears: `app/page.tsx:6-18` owns server context; `components/chat.tsx:89-217` owns chat state/transport; `app/api/chat/route.ts:20-229` owns policy; `lib/streaming/create-chat-stream-response.ts:33-235` owns streaming; `lib/db/actions.ts:23-193` owns persistence queries.

Junior miss: reading all files as one big call stack. Senior notice: boundaries reduce blast radius when changing behavior.

Practice: draw the chat flow and mark each ownership boundary.

## Type Safety

Concept: types make contracts visible before runtime.

Where it appears: `lib/types/search.ts:1-2`, `components/chat-panel.tsx:30-56`, `lib/schema/search.tsx:6-94`, `lib/db/schema.ts:70-300`.

Junior miss: seeing types as autocomplete. Senior notice: dynamic AI output requires disciplined narrowing.

Practice: find three `any` usages and explain whether each is acceptable.

## Validation Boundaries

Concept: external input should be parsed and rejected early.

Where it appears: good pattern in `app/api/feedback/route.ts:12-24`; tool schema in `lib/schema/search.tsx:6-94`; weak pattern in `app/api/chat/route.ts:29-52`.

Junior miss: TypeScript does not validate HTTP JSON. Senior notice: route schemas improve security and error behavior.

Practice: design a Zod schema for `/api/chat` without over-validating AI SDK message internals.

## API Design

Concept: API routes coordinate trust, policy, and response shape.

Where it appears: chat route policy phases in `app/api/chat/route.ts:57-144`; upload validation in `app/api/upload/route.ts:15-62`; feedback route in `app/api/feedback/route.ts:7-80`.

Junior miss: thinking API routes should do all business logic. Senior notice: routes should delegate to focused helpers after enforcing boundaries.

Practice: list every branch in `/api/chat` and the status code it returns.

## State Management

Concept: state should live where it is changed and used.

Where it appears: local chat state in `components/chat.tsx:55-87`; cookie-backed mode in `components/search-mode-selector.tsx:25-56`; contexts in `lib/contexts/chat-context.tsx:5-31` and `lib/contexts/user-context.tsx:5-18`.

Junior miss: reaching for Redux. Senior notice: local/context/cookie state is enough here.

Practice: decide where a "source display mode" preference should live for temporary versus persistent behavior.

## Data Fetching

Concept: server-side loading should hydrate UI when data is needed before render.

Where it appears: `app/search/[id]/page.tsx:30-59` loads a saved chat; `app/page.tsx:6-18` loads model selector data.

Junior miss: fetching everything in `useEffect`. Senior notice: Server Components reduce client waterfalls.

Practice: trace how saved messages reach `Chat`.

## Database Access

Concept: persistence code should enforce ownership and data shape.

Where it appears: schema/RLS in `lib/db/schema.ts:25-300`; DB actions in `lib/db/actions.ts:23-193`; server actions in `lib/actions/chat.ts:39-338`.

Junior miss: skipping authorization because "the route already checked." Senior notice: defense-in-depth with RLS and action checks.

Practice: review `upsertMessage` and explain why the comment in `lib/actions/chat.ts:167-178` matters.

## Error Handling

Concept: errors should fail the right layer without hiding critical data loss.

Where it appears: UI modal error classification in `components/chat.tsx:135-214`; persistence retry in `lib/streaming/helpers/persist-stream-results.ts:81-97`; route catch in `app/api/chat/route.ts:222-228`.

Junior miss: catching everything with generic toast. Senior notice: some failures are user-facing; others need monitoring.

Practice: classify five errors as user-facing, log-only, or retryable.

## Testing Strategy

Concept: test behavior at the smallest meaningful boundary.

Where it appears: feedback route tests (`app/api/feedback/__tests__/route.test.ts:53-201`), RLS helper tests (`lib/db/__tests__/with-rls.test.ts:29-179`), render test (`components/__tests__/render-message.test.tsx:37-82`).

Junior miss: testing only happy paths. Senior notice: auth, persistence, validation, and streaming need failure tests.

Practice: write a test plan for `/api/chat` adaptive guest gate.

## Security Posture

Concept: the system must prove identity, validate input, authorize resources, and constrain network access.

Where it appears: anonymous mode guard (`lib/auth/get-current-user.ts:22-40`), RLS (`lib/db/schema.ts:53-68`), upload checks (`app/api/upload/route.ts:15-62`), crawler (`app/api/advanced-search/route.ts:313-407`).

Junior miss: authentication is not authorization. Senior notice: upload `chatId` and crawler URLs are sensitive boundaries.

Practice: design checks for upload chat ownership.

## Performance

Concept: latency comes from model context, search providers, crawling, DB, and rendering.

Where it appears: context pruning (`lib/streaming/create-chat-stream-response.ts:138-159`), advanced search cache (`app/api/advanced-search/route.ts:66-113`), crawl fan-out (`app/api/advanced-search/route.ts:258-263`), DB parallel load (`lib/db/actions.ts:162-175`).

Junior miss: only optimizing React re-renders. Senior notice: network and model costs dominate.

Practice: propose a concurrency limit for advanced crawling.

## Configuration Management

Concept: config determines which features can run.

Where it appears: `.env.local.example:18-161`, `docker-compose.yaml:15-28`, `drizzle.config.ts:6-18`, `vitest.config.mts:5-21`.

Junior miss: local success may depend on Docker-provided services. Senior notice: docs should distinguish Docker and non-Docker setup.

Practice: write a minimal `.env.local` checklist for local non-Docker dev.

