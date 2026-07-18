# Testing Strategy

## What actually has tests (from the file tree)

| Area | Test files | Style |
| --- | --- | --- |
| Rate limits | [guest-limit.test.ts](../../lib/rate-limit/__tests__/guest-limit.test.ts), [adaptive-limit.test.ts](../../lib/rate-limit/__tests__/adaptive-limit.test.ts) | unit, Redis mocked |
| RLS wrapper | [with-rls.test.ts](../../lib/db/__tests__/with-rls.test.ts) (mocked), [rls-policies.integration.test.ts](../../lib/db/__tests__/rls-policies.integration.test.ts) (real PG) | unit + integration |
| Render/spec system | [parse-spec-block](../../lib/render/__tests__/parse-spec-block.test.ts), [spec-fence](../../lib/render/__tests__/spec-fence.test.ts), [migrations](../../lib/render/__tests__/migrations.test.ts), [strip-spec-blocks](../../lib/render/__tests__/strip-spec-blocks.test.ts), [llm-image-output.e2e](../../lib/render/__tests__/llm-image-output.e2e.test.ts) | unit + e2e |
| Utils | [citation](../../lib/utils/__tests__/citation.test.ts), [context-window](../../lib/utils/__tests__/context-window.test.ts), [domain](../../lib/utils/__tests__/domain.test.ts), [model-selection](../../lib/utils/__tests__/model-selection.test.ts) | pure-function unit |
| Streaming helpers | [prepare-messages](../../lib/streaming/helpers/__tests__/prepare-messages.test.ts), [prune-messages-integration](../../lib/streaming/__tests__/prune-messages-integration.test.ts), [create-ephemeral](../../lib/streaming/__tests__/create-ephemeral-chat-stream-response.test.ts) | unit |
| Actions/API | [feedback route](../../app/api/feedback/__tests__/route.test.ts), [actions/chat](../../lib/actions/__tests__/chat.test.ts), [actions/feedback](../../lib/actions/__tests__/feedback.test.ts), [models/fetch-models](../../lib/models/__tests__/fetch-models.test.ts) | unit, mocked |
| Components | [render-message](../../components/__tests__/render-message.test.tsx), [research-process-section](../../components/__tests__/research-process-section.test.tsx) | RTL |

**Untested (high-value contribution targets):** `app/api/chat/route.ts`, the search/fetch tools + providers, `message-mapping.ts` (775 lines, zero tests), the upload route, most components.

## The pyramid, this repo's version

- **Wide base — pure-function units.** Anything deterministic with no IO: `processCitations`, `truncateMessages`, model filters. Cheapest, fastest, most tests. Model: [citation.test.ts](../../lib/utils/__tests__/citation.test.ts) — a table of input→expected with edge cases (spaces, same-domain, out-of-range).
- **Middle — units with mocked IO.** Rate limits mock `@upstash/redis` ([guest-limit.test.ts#L8-L13](../../lib/rate-limit/__tests__/guest-limit.test.ts#L8-L13)); RLS mocks `db.transaction` ([with-rls.test.ts#L10-L14](../../lib/db/__tests__/with-rls.test.ts#L10-L14)). You test *your* logic, not the vendor's.
- **Narrow top — integration/e2e.** The RLS integration test needs a real Postgres (so it's meaningless in CI, which stubs `DATABASE_URL` — [ci.yml#L88-L90](../../.github/workflows/ci.yml#L88-L90)); the spec e2e exercises real parsing.

## What belongs at each layer

- **Unit:** pure transforms, one branch per test, edge cases. If it has an `if`, it wants a test per side.
- **Mocked-IO unit:** your control flow around a dependency — the *contract you have with Redis/DB*, not Redis itself. Assert what you'd break if you refactored (status codes, keys, TTLs).
- **Integration:** things that only fail across a real boundary — RLS policies (a mock can't prove Postgres enforces them). This is why the repo keeps a separate real-PG test.
- **Component (RTL):** rendering branches — [render-message.test.tsx](../../components/__tests__/render-message.test.tsx). Test what the user sees, not internal state.

## What NOT to test

- The AI SDK, Drizzle, Zod — vendor code. Test your *use* of them.
- Exact LLM output (nondeterministic) — test the deterministic wrapper (fallback, parsing, stripping), as [title-generator](../../lib/agents/title-generator.ts) *should* be tested.
- Getters/trivial pass-throughs.

## Flake prevention (what this repo does right)

- **Mock time-free, mock IO:** limits mock Redis so no network, no clock dependence on the *result* (though the midnight math is real — a place to inject a clock).
- **Reset between tests:** `beforeEach(() => { mockRedisIncr.mockReset(); … })` ([guest-limit.test.ts#L16-L23](../../lib/rate-limit/__tests__/guest-limit.test.ts#L16-L23)) and env re-set per test — no cross-test leakage.
- **Deterministic fixtures:** citation maps are hand-built objects, not fetched.

Flake sources to avoid when you add tests: real network, `Date.now()` in assertions (the limiters compute `getSecondsUntilMidnight` — if you test the TTL value, freeze the clock with `vi.setSystemTime`), shared module-level caches (`modelsCache`, `encoderCache`) surviving across tests — reset or isolate.

## Interview angle

- "What's your testing philosophy?" → pyramid + "test the contract, not the vendor"; cite the Redis-mocked limiter tests. ([Q40](../08-interview-prep/01-js-ts-node-deep-dive.md) area / behavioral)
- "How do you decide what to test?" → per-branch on pure logic; contract-level on IO; integration only where a mock can't prove the property (RLS).
- "How do you fight flaky tests?" → determinism checklist above; the title-test kata ([04-review-katas.md](../04-code-reading-gym/04-review-katas.md), Kata 9).

Drill: pick `truncateMessages` ([context-window.ts#L189-L285](../../lib/utils/context-window.ts#L189-L285)) and list the test cases you'd write (under limit; over limit; huge first message; huge last message; empty input; all-assistant input). Then diff against the actual [context-window.test.ts](../../lib/utils/__tests__/context-window.test.ts) — what's missing?
