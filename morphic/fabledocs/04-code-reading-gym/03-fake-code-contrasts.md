# Fake-Code Contrasts

Bad-vs-better pairs, each tied to a real Morphic pattern. **All code here is illustrative and fake** — labeled per the ground rules. Read the bad version, articulate the instinct behind it, then read better.

---

## 1. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
// BAD: component reads raw DB columns
function Msg({ row }) {
  return <p>{row.text_text ?? row.tool_search_output?.results?.[0]?.title}</p>
}
```
```ts
// Illustrative fake code: not from this repo
// BETTER: DB rows map to a UI type at one boundary, then components read that type
const parts = dbParts.map(mapDBPartToUIMessagePart) // one place
function Msg({ part }: { part: UIMessagePart }) { /* switch on part.type */ }
```
Instinct: "the data's right there, just read it." Real anchor: the mapping boundary exists precisely for this — [message-mapping.ts#L344-L666](../../lib/utils/message-mapping.ts#L344-L666). Coupling UI to columns means a schema change breaks components.

## 2. Missing permission filter (trusting the app, not the DB)

```ts
// Illustrative fake code: not from this repo
// BAD: forgot the userId filter — returns everyone's chats
const rows = await db.select().from(chats).where(eq(chats.id, chatId))
```
```ts
// Illustrative fake code: not from this repo
// BETTER: run inside RLS; the DB filters even if you forget
return withRLS(userId, tx => tx.select().from(chats).where(eq(chats.id, chatId)))
```
Instinct: authorization is the app's job. Real anchor: [with-rls.ts#L39-L70](../../lib/db/with-rls.ts) + policies. The better version *cannot* leak cross-tenant even with a forgotten filter — that's the whole point of RLS.

## 3. N+1 delete loop

```ts
// Illustrative fake code: not from this repo
// BAD-ish: one round-trip per chat (this is roughly what clearChats does today)
for (const chat of chats) await deleteChat(chat.id, userId)
```
```ts
// Illustrative fake code: not from this repo
// BETTER: one statement, RLS-scoped
await withRLS(userId, tx => tx.delete(chats).where(eq(chats.userId, userId)))
```
Instinct: reuse the existing single-delete function. Real anchor: [lib/actions/chat.ts#L229-L233](../../lib/actions/chat.ts#L229-L233). At 10 chats nobody notices; the better version is O(1) round-trips and cascades handle messages/parts.

## 4. Stale cache key (cross-user poisoning)

```ts
// Illustrative fake code: not from this repo
// BAD: cache key omits the requesting user — a public/private view can bleed
unstable_cache(fn, ['chat', chatId], { tags: [`chat-${chatId}`] })
```
```ts
// Illustrative fake code: not from this repo
// BETTER: key includes requesting user (this is what the repo does)
unstable_cache(fn, ['chat-with-messages', chatId, userId ?? 'anonymous'], { tags: [...] })
```
Instinct: cache by the obvious id. Real anchor: [lib/actions/chat.ts#L22-L31](../../lib/actions/chat.ts#L22-L31). Because the loader applies visibility rules, the *result* depends on the viewer — so the viewer must be in the key.

## 5. Side effect on the critical path

```ts
// Illustrative fake code: not from this repo
// BAD: user waits for the DB insert before the model even starts
await createChatWithFirstMessage(chatId, message, userId)
const stream = await agent.stream(...)
```
```ts
// Illustrative fake code: not from this repo
// BETTER: fire the insert, stream immediately, settle at the end (idempotent)
context.pendingInitialSave = createChatWithFirstMessage(chatId, message, userId).catch(...)
return [message] // model starts now; await pendingInitialSave in onFinish
```
Instinct: do writes before reads, always in order. Real anchor: [prepare-messages.ts#L91-L116](../../lib/streaming/helpers/prepare-messages.ts#L91-L116). Safe *only* because ids are client-generated and the write is idempotent — the better version without those preconditions is a bug.

## 6. Swallowed error with no trail

```ts
// Illustrative fake code: not from this repo
// BAD: silent catch — outage looks like success
try { await trackChatEvent(...) } catch {}
```
```ts
// Illustrative fake code: not from this repo
// BETTER: contain but log; never let analytics break the request
try { await trackChatEvent(...) } catch (e) { console.error('Analytics tracking failed:', e) }
```
Instinct: "analytics shouldn't crash the app." Correct — but *silent* is the bug. Real anchor: [route.ts#L202-L205](../../app/api/chat/route.ts#L202-L205) does the better version. Next rung (not in repo): a metric so you *know* it's failing.

## 7. `any` erasing a real check

```ts
// Illustrative fake code: not from this repo
// BAD: typo compiles, breaks at runtime
const col = `tool_${name}_input` as any
row[col] = part.input
```
```ts
// Illustrative fake code: not from this repo
// BETTER: constrain name to a union so bad keys don't compile
const col = `tool_${name}_input` as const // name: ToolName
```
Instinct: templated keys are convenient. Real anchor: the live version uses `as keyof DBMessagePart` ([message-mapping.ts#L110-L121](../../lib/utils/message-mapping.ts#L110-L121)) — which accepts any string the caller produces. A `ToolName` union would catch typos.

## 8. Trusting client input for a server decision

```ts
// Illustrative fake code: not from this repo
// BAD: believe the client that this file is an image
if (file.type.startsWith('image/')) await store(file)
```
```ts
// Illustrative fake code: not from this repo
// BETTER: whitelist + verify content, not just the claimed type
if (!ALLOWED.includes(file.type)) reject()
if (!(await sniffMagicBytes(file)) ) reject() // don't trust file.type alone
```
Instinct: `file.type` is right there. Real anchor: [upload/route.ts#L55-L60](../../app/api/upload/route.ts#L55-L60) does the whitelist (good) but not the sniff — the "better" adds the check the server can't get from the claimed type.

## 9. Casual public-contract change

```ts
// Illustrative fake code: not from this repo
// BAD: rename a citation format the prompt AND the UI depend on
content.replace(/\[(\d+)\]\{(#\w+)\}/, ...) // changed () to {}
```
Instinct: "just a regex tweak." Real anchor: the citation format is a three-party contract — prompt ([search-mode-prompts.ts#L72-L101](../../lib/agents/prompts/search-mode-prompts.ts#L72-L101)), tool output, and parser ([citation.ts#L82-L83](../../lib/utils/citation.ts#L82-L83)). Changing one side silently breaks the others; there's no compiler between them. Better: change all three in one PR + add a test that renders a real citation end to end.

## 10. Fetching in an effect for derivable data

```tsx
// Illustrative fake code: not from this repo
// BAD: effect + state to derive sections from messages
useEffect(() => setSections(buildSections(messages)), [messages])
```
```tsx
// Illustrative fake code: not from this repo
// BETTER: derive during render
const sections = useMemo(() => buildSections(messages), [messages])
```
Instinct: state feels safer. Real anchor: [chat.tsx#L224-L252](../../components/chat.tsx#L224-L252). The effect version adds a render and a window where `sections` lags `messages`.

## 11. Losing backpressure without meaning to

```ts
// Illustrative fake code: not from this repo
// BAD: no consumeStream — abandoned tabs abort persistence
return result.toUIMessageStreamResponse({ onFinish: persist })
```
Real anchor: the repo *deliberately* calls `result.consumeStream()` ([create-chat-stream-response.ts#L184](../../lib/streaming/create-chat-stream-response.ts#L184)) so `onFinish` runs even if the client leaves — trading wasted tokens for guaranteed persistence. The "bad" version isn't wrong universally; it's wrong *for this requirement*. Lesson: context decides.

## 12. Non-atomic counter

```ts
// Illustrative fake code: not from this repo
// BAD-ish: two ops; crash between leaves an immortal key (repo does this)
const count = await redis.incr(key)
if (count === 1) await redis.expire(key, ttl)
```
```lua
-- Illustrative fake code: not from this repo
-- BETTER: atomic via Lua
local c = redis.call('INCR', KEYS[1])
if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end
return c
```
Real anchor: [adaptive-limit.ts#L81-L89](../../lib/rate-limit/adaptive-limit.ts#L81-L89). The two-op version is *usually* fine; the atomic version removes the immortal-key edge.

---

Drill: pick three of these, find the real anchor, and write one sentence on *when the "bad" version is actually acceptable* (contrasts #5 and #11 both have legitimate uses). That nuance — "it depends, here's on what" — is the mid-level review voice.
