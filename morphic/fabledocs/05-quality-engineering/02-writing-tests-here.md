# Writing Tests Here

Concrete recipes using this repo's real helpers and conventions. Commands are __inferred__ (Bun not on the authoring machine). All illustrative test code is labeled fake — it shows the *shape*; adapt to the actual signatures.

Run: `bun run test` (all), `bun run test path/to/file.test.ts` (one), `bun run test:watch` (dev). Colocate as `__tests__/name.test.ts` next to the source; `globals: true` means no `describe/it` imports needed ([vitest.config.mts#L17-L21](../../vitest.config.mts#L17-L21)).

---

## Recipe 1: Pure-function happy path (model = citation.test.ts)

```ts
// Illustrative fake code: not from this repo
import { truncateMessages } from '@/lib/utils/context-window'
it('returns all messages when under budget', () => {
  const msgs = [{ role: 'user', content: 'hi' }]
  expect(truncateMessages(msgs, 10_000, 'gpt-4.1')).toEqual(msgs)
})
```
Pattern: input → expected, one assertion per behavior. See real: [citation.test.ts#L28-L35](../../lib/utils/__tests__/citation.test.ts#L28-L35).

## Recipe 2: Validation-failure path

```ts
// Illustrative fake code: not from this repo
it('drops citations whose number is out of range', () => {
  const out = processCitations('x [999](#t1)', { t1: { 1: item } })
  expect(out).toBe('x ') // invalid → empty string, see citation.ts#L88-L90
})
```
Assert the *soft-fail* behavior, not a throw — matches how the code actually degrades.

## Recipe 3: Permission failure (mock the DB transaction)

```ts
// Illustrative fake code: not from this repo
import { RLSViolationError, withRLS } from '@/lib/db/with-rls'
vi.mock('@/lib/db', () => ({ db: { transaction: vi.fn() } }))
it('translates RLS violations', async () => {
  vi.mocked(db.transaction).mockRejectedValue(
    new Error('new row violates row-level security policy'))
  await expect(withRLS('u1', async () => {})).rejects.toThrow(RLSViolationError)
})
```
Real: [with-rls.test.ts#L72-L80](../../lib/db/__tests__/with-rls.test.ts#L72-L80). You test the error *translation*, not Postgres.

## Recipe 4: Cross-tenant rejection (integration, real Postgres)

This is the one test type a mock can't fake — it must run against a database with the policies applied. Pattern (see [rls-policies.integration.test.ts](../../lib/db/__tests__/rls-policies.integration.test.ts)): insert a chat as user A under `withRLS('A', …)`, then attempt to read it under `withRLS('B', …)` and assert empty. Requires a local `DATABASE_URL` + `bun migrate`; excluded from CI by the stubbed URL.

## Recipe 5: Rate-limit boundary (mock Redis, control the count)

```ts
// Illustrative fake code: not from this repo
vi.mock('@upstash/redis', () => ({
  Redis: vi.fn(() => ({ incr: mockIncr, expire: mockExpire }))
}))
beforeEach(() => { process.env.MORPHIC_CLOUD_DEPLOYMENT = 'true'; /* + upstash vars */ })
it('blocks the request that exceeds the limit', async () => {
  mockIncr.mockResolvedValue(11)
  const res = await checkAndEnforceGuestLimit('1.2.3.4')
  expect(res?.status).toBe(401)
})
```
Real: [guest-limit.test.ts#L30-L40](../../lib/rate-limit/__tests__/guest-limit.test.ts#L30-L40). Note the env setup in `beforeEach` — the limiter no-ops outside cloud mode, so you must enable it to test enforcement.

## Recipe 6: Component render branch (RTL)

```tsx
// Illustrative fake code: not from this repo
import { render, screen } from '@testing-library/react'
it('renders a user file part as a file section', () => {
  render(<RenderMessage message={{ role:'user', parts:[{type:'file',url:'u',filename:'f.pdf',mediaType:'application/pdf'}] }} .../>)
  expect(screen.getByText('f.pdf')).toBeInTheDocument()
})
```
Real: [render-message.test.tsx](../../components/__tests__/render-message.test.tsx). Test the visible output of each `part.type` branch ([render-message.tsx#L56-L81](../../components/render-message.tsx#L56-L81)).

## Recipe 7: The missing message-mapping test (your contribution)

`message-mapping.ts` has no test and is pure (UI parts ↔ DB columns). A round-trip property is the highest-value test you could add:

```ts
// Illustrative fake code: not from this repo
it('round-trips a search tool part through DB columns', () => {
  const uiPart = { type:'tool-search', state:'output-available', toolCallId:'c1',
                   input:{query:'x'}, output:{results:[]} }
  const [db] = mapUIMessagePartsToDBParts([uiPart], 'm1')
  expect(mapDBPartToUIMessagePart(db)).toEqual(uiPart)
})
```
This would have caught the whole class of column-key bugs the `as keyof` casts hide. Write it for real as ticket #? in [06-contribution-practice](../06-contribution-practice/01-good-first-tickets.md).

## Recipe 8: Async side-effect ordering (prepare-messages style)

Mock the action modules and assert *what was called with what*, e.g. that a regeneration deletes from the right index:

```ts
// Illustrative fake code: not from this repo
vi.mock('@/lib/actions/chat', () => ({ deleteMessagesFromIndex: vi.fn(), loadChat: vi.fn(), upsertMessage: vi.fn(), /* … */ }))
it('deletes the assistant message and everything after on regenerate', async () => {
  vi.mocked(loadChat).mockResolvedValue(chatWith([u1, a1, u2, a2]))
  await prepareMessages({ trigger:'regenerate-message', messageId:a1.id, /*…*/ } as any, null)
  expect(deleteMessagesFromIndex).toHaveBeenCalledWith(expect.anything(), a1.id, expect.anything())
})
```
Real: [prepare-messages.test.ts](../../lib/streaming/helpers/__tests__/prepare-messages.test.ts).

## Conventions checklist

- Reset mocks + env in `beforeEach` (prevents cross-test bleed).
- Use `@/` imports (aliased in vitest config).
- Freeze the clock with `vi.setSystemTime` if you assert on TTL/reset timestamps.
- One behavior per `it`; name it as the behavior, not the function.
- Prefer asserting the *contract* (status code, key format, part shape) over internals.

Drill: add Recipe 7 for real, run `bun run test lib/utils` (inferred), and make it pass. If it *fails*, you found a bug — write it up per [06-contribution-practice/01](../06-contribution-practice/01-good-first-tickets.md).
