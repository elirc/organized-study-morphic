# Senior Engineer Guide

## Architecture Critique

| Area | Score | Rationale |
| --- | ---: | --- |
| Scalability | 3.5/5 | Streaming, pruning, caching, and provider abstraction are solid. Advanced search route can fan out many crawls with `Promise.all` (`app/api/advanced-search/route.ts:258-263`). |
| TypeScript Discipline | 3/5 | Good unions, props, Drizzle types, and Zod schemas. `any` remains common around AI tool/message boundaries (`lib/types/ai.ts:19-42`, `components/chat.tsx:426-442`). |
| Separation of Concerns | 3.5/5 | Chat flow is layered. Some routes are too dense, especially advanced search (`app/api/advanced-search/route.ts:137-659`). |
| Testability | 3.5/5 | Vitest coverage exists across actions, routes, render, RLS, streaming helpers. Main `/api/chat` integration behavior is still hard to test. |
| Maintainability | 3.5/5 | File organization is understandable. Dynamic message part persistence is complex and needs careful docs/tests. |
| Security Posture | 3/5 | Auth/RLS is thoughtful. API validation, upload ownership, and crawler SSRF protection deserve strengthening. |
| Performance | 3.5/5 | Context pruning, cache tags, and Redis search cache help. Advanced crawling and repeated auth calls can be hot spots. |
| Developer Experience | 4/5 | Clear scripts, Docker Compose, env example, tests, and path aliases. Setup is spread across files. |

## Performance Audit

Finding: advanced search performs unbounded concurrent crawls relative to `maxResults * multiplier` (`app/api/advanced-search/route.ts:258-263`).

Suggested shape:

```ts
// Documentation-only sketch: limit concurrent crawls instead of Promise.all over every result.
const crawledResults = []
for (const batch of chunk(generalResults.slice(0, limit), 5)) {
  crawledResults.push(...(await Promise.all(batch.map(result => crawlPage(result, query)))))
}
```

Finding: `getCurrentUserId` increments perf counters and can call Supabase repeatedly (`lib/auth/get-current-user.ts:18-44`). Some routes call it multiple times indirectly, such as page metadata and page rendering in `app/search/[id]/page.tsx:13-37`.

Senior question: can request-level user context be passed deeper instead of rediscovered?

## Security Audit

Finding: `/api/chat` trusts raw JSON shape after minimal manual checks (`app/api/chat/route.ts:29-52`).

Suggested corrected snippet:

```ts
// Documentation-only sketch.
const chatRequestSchema = z.object({
  chatId: z.string().min(1),
  trigger: z.enum(['submit-message', 'regenerate-message']),
  messageId: z.string().optional(),
  isNewChat: z.boolean().optional(),
  message: z.any().optional(),
  messages: z.array(z.any()).optional()
})
const parsed = chatRequestSchema.safeParse(await req.json())
if (!parsed.success) return Response.json({ error: parsed.error.flatten() }, { status: 400 })
```

Finding: upload route accepts a `chatId` and uses it in object storage path without checking that the authenticated user owns that chat (`app/api/upload/route.ts:41-62`, `app/api/upload/route.ts:76-94`).

Suggested check:

```ts
// Documentation-only sketch.
const chat = await loadChat(chatId, userId)
if (!chat || chat.userId !== userId) {
  return NextResponse.json({ error: 'Unauthorized chat' }, { status: 403 })
}
```

Finding: advanced crawler accepts URLs from search results and fetches HTML (`app/api/advanced-search/route.ts:313-407`). Search-result origin lowers risk, but SSRF protections should still reject private IPs, localhost, and unsupported protocols before fetching.

## TypeScript Discipline Review

Good:

- `SearchMode` union (`lib/types/search.ts:1-2`).
- Zod tool schemas (`lib/schema/search.tsx:6-94`).
- Drizzle table inference (`lib/db/schema.ts:70-300`).
- Type guards for message mapping (`lib/utils/message-mapping.ts:67-102`).

Needs care:

- `metadata: jsonb(...).$type<Record<string, any>>()` (`lib/db/schema.ts:85`).
- Tool input/output JSON columns are `any` (`lib/db/schema.ts:173-194`).
- Chat submit uses `parts: any[]` (`components/chat.tsx:426-442`).

## Custom Abstractions Inventory

- `ChatProvider` passes `sendMessage` without prop drilling (`lib/contexts/chat-context.tsx:5-31`).
- `UserProvider` exposes a simple user-present boolean (`lib/contexts/user-context.tsx:5-18`).
- `withRLS`/`withOptionalRLS` wraps DB calls with user context (`lib/db/actions.ts:16-18`, tested in `lib/db/__tests__/with-rls.test.ts:29-179`).
- `researcher` configures AI tools by search mode (`lib/agents/researcher.ts:67-163`).
- `createSearchTool` wraps search providers into AI tool streaming (`lib/tools/search.ts:20-173`).
- Message mapping converts AI SDK parts to relational DB rows (`lib/utils/message-mapping.ts:127-260`).

## Testing Assessment

Strengths:

- Server actions are mocked and tested (`lib/actions/__tests__/chat.test.ts:32-240`).
- API route validation has focused route tests (`app/api/feedback/__tests__/route.test.ts:53-201`).
- RLS helpers test injection safety and error mapping (`lib/db/__tests__/with-rls.test.ts:29-179`).
- Rendering behavior has component tests (`components/__tests__/render-message.test.tsx:37-82`).

Gap: `/api/chat` deserves route-level tests for guest disabled, adaptive guest cloud gate, no model, provider disabled, and regenerate without `messageId`.

## Example Test For Risky Untested Behavior

Do not add this test directly without adapting mocks.

```ts
import { describe, expect, it, vi } from 'vitest'
import { POST } from '@/app/api/chat/route'

describe('POST /api/chat adaptive guest cloud gate', () => {
  it('returns authRequired JSON for cloud guest using adaptive mode', async () => {
    vi.stubEnv('MORPHIC_CLOUD_DEPLOYMENT', 'true')
    vi.mocked(getCurrentUserId).mockResolvedValue(undefined)
    vi.mocked(selectModel).mockResolvedValue({ id: 'x', providerId: 'openai' } as any)
    vi.mocked(isProviderEnabled).mockReturnValue(true)

    const req = new Request('http://localhost/api/chat', {
      method: 'POST',
      headers: { cookie: 'searchMode=adaptive' },
      body: JSON.stringify({ trigger: 'submit-message', chatId: 'c1', message: { id: 'm1', role: 'user', parts: [] } })
    })

    const res = await POST(req)
    expect(res.status).toBe(401)
    await expect(res.json()).resolves.toMatchObject({ authRequired: true, mode: 'adaptive' })
  })
})
```

## Bug Injection Exercise

1. Symptom: pressing Enter during Japanese IME composition submits early. Test: simulate composition start, Enter, composition end in `ChatPanel`.
2. Symptom: uploaded dragged files show "uploaded" but have no usable filename. Test: mock `/api/upload` response and inspect `hooks/use-file-dropzone.ts:81-90` versus upload route response `app/api/upload/route.ts:96-101`.
3. Symptom: adaptive mode silently becomes quick after a new mode is added. Test: cookie value survives selector and API route validation.
4. Symptom: private chat appears as 404 instead of login redirect for anonymous user. Test: `app/search/[id]/page.tsx:36-44` with `loadChat` returning null under no RLS context.
5. Symptom: search route crawls internal network URL. Test: provider returns `http://127.0.0.1:...` and crawler rejects before fetch.

## Git History Learning Exercise

Recent commits inspected:

- `be62d2a chore: bump next and ai-sdk packages to latest patches (#841)` implies active dependency maintenance.
- `ecd581b chore: switch quick model to Gemini 3.1 Flash Lite GA (#840)` implies model defaults are product behavior, not just config.
- `07963f8 feat: add per-user daily limit for adaptive search mode (#836)` explains why adaptive mode has special rate-limit checks (`app/api/chat/route.ts:140-143`).
- `1a3eeef feat: show "Reply..." placeholder for follow-up messages` maps to `components/chat-panel.tsx:264`.
- `9b4d1b9 feat: add footer message with typewriter tips next to assistant logo (#830)` points to UI polish and onboarding cues.

## If I Owned This Codebase

| Item | Effort | Impact | Risk | Why It Matters |
| --- | --- | --- | --- | --- |
| Add Zod schemas for public API bodies | M | High | Low | Improves security and debugging at boundaries. |
| Add `/api/chat` route tests | M | High | Medium | Main product path currently relies on lower-level tests. |
| Verify upload chat ownership | S | High | Low | Prevents object path misuse across chats. |
| Limit advanced crawl concurrency and add URL safety checks | M | High | Medium | Protects performance and network boundary. |
| Reduce `any` around message parts with discriminated unions | L | Medium | Medium | Makes persistence safer but must match AI SDK evolution. |
| Centralize setup guide in existing docs later, if allowed | S | Medium | Low | Current setup must be reconstructed from several files. |

