# Security Checklist

Each item mapped to real code, with the repo's status. Use the pre-merge checklist at the bottom on every PR that touches a boundary.

## Authorization / IDOR

- **Status: strong.** Two layers: app ownership checks ([create-chat-stream-response.ts#L64-L70](../../lib/streaming/create-chat-stream-response.ts#L64-L70), [db/actions.ts#L328-L334](../../lib/db/actions.ts#L328-L334)) + Postgres RLS ([schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67)). Guessing another user's `chatId` returns nothing under their RLS context.
- **Watch:** every `withOptionalRLS(null, …)` call ([with-rls.ts#L80-L90](../../lib/db/with-rls.ts#L80-L90)) runs *without* isolation — audit each as intended-public.

## Input validation

- **Status: mixed.** LLM inputs get Zod ([search.tsx](../../lib/schema/search.tsx), [todo.ts](../../lib/tools/todo.ts)); spec blocks get catalog validation ([catalog.ts](../../lib/render/catalog.ts)). But the `/api/chat` and `/api/chats` bodies get only presence checks ([route.ts#L38-L52](../../app/api/chat/route.ts#L38-L52)) — no schema on `message.parts`. **Fix:** add Zod (Ticket + Kata 3's *safe* version).

## SSRF (Server-Side Request Forgery)

- **Status: possible risk / investigate.** The fetch tool retrieves *arbitrary* URLs server-side ([fetch.ts#L14-L21](../../lib/tools/fetch.ts#L14-L21)) with no visible private-IP/localhost/metadata-endpoint denylist. An LLM (influenced by fetched content or a user URL) could target `http://169.254.169.254/…` or internal services. The search self-call also builds a URL from forwarded headers ([search.ts#L88-L101](../../lib/tools/search.ts#L88-L101), Pattern 14). **Investigate** real exposure (network egress rules may mitigate); if unmitigated, add a denylist + block private ranges. This is the repo's most important open security question.

## XSS

- **Status: strong by design.** No `dangerouslySetInnerHTML` on model output; the generative UI is a Zod-validated component catalog ([catalog.ts](../../lib/render/catalog.ts)) — the model can't inject HTML. Markdown goes through streamdown (sanitizing). Citation hrefs are `encodeURI`'d ([citation.ts#L107](../../lib/utils/citation.ts#L107)). **Watch:** `Image src` in the catalog accepts any string ([catalog.ts#L51-L57](../../lib/render/catalog.ts#L51-L57)) — a model could point it at a tracking URL; low severity, worth noting.

## CSRF

- **Status: relies on Supabase/SameSite cookies + JSON APIs.** State-changing routes are POST/JSON (not form-encoded), and auth is cookie-based via `@supabase/ssr`. **Watch:** server actions are POST endpoints too; Next.js provides some protection, but confirm any new action mutating data checks auth ([lib/actions/chat.ts](../../lib/actions/chat.ts) pattern).

## Injection (SQL)

- **Status: strong.** Drizzle parameterizes; the one raw SQL call parameterizes the user id ([with-rls.ts#L48-L50](../../lib/db/with-rls.ts#L48-L50)) — explicitly tested against `"user'; DROP TABLE…"` ([with-rls.test.ts#L56-L70](../../lib/db/__tests__/with-rls.test.ts#L56-L70)).

## Secrets

- **Status: good hygiene.** Server-only env for keys; only `NEXT_PUBLIC_*` reach the client; `.env.local` git-ignored; `.env.local.example` uses placeholders. **Watch:** 500 handlers that echo `err.message` ([upload/route.ts#L63-L69](../../app/api/upload/route.ts#L63-L69)) can leak internal detail — return generic messages.

## File uploads

- **Status: mostly good.** Auth-gated, 5MB cap, MIME whitelist, filename sanitized, userId-namespaced keys ([upload/route.ts#L44-L94](../../app/api/upload/route.ts#L44-L94)). **Gaps:** MIME from client `file.type` (no magic-byte sniff); public-by-URL storage = "unlisted", not private; `chatId` in key is unverified.

## Rate limiting / abuse

- **Status: advisory.** Fail-open, cloud-only, spoofable guest IP ([guest-limit.ts](../../lib/rate-limit/guest-limit.ts), [route.ts#L77-L82](../../app/api/chat/route.ts#L77-L82)). Fine for the product; know it's not an abuse control.

## Open redirects / auth flows

- **Status: not deeply audited here.** Auth pages/routes under `app/auth/*` + `middleware` redirects to `/auth/login` ([supabase/middleware.ts#L55-L60](../../lib/supabase/middleware.ts#L55-L60)). If you touch OAuth callback ([app/auth/oauth/route.ts](../../app/auth/oauth/route.ts)), verify redirect targets are validated.

## Dependency risk

- Large tree (radix, AI SDK, supabase). Keep `bun.lock` committed; watch advisories on `jsdom`, `node-html-parser` (both parse untrusted content in the fetch/advanced-search paths).

## Cookies / sessions

- Supabase SSR handles refresh in middleware ([supabase/middleware.ts#L33-L41](../../lib/supabase/middleware.ts#L33-L41)); the "don't touch code between createServerClient and getUser" comment ([L33-L37](../../lib/supabase/middleware.ts#L33-L37)) is a real footgun warning — heed it.

## Webhooks

- Only outbound (Slack feedback via `SLACK_WEBHOOK_URL`); no inbound webhook receivers to authenticate.

---

## Pre-merge security checklist

Before approving any PR touching a route, action, tool, or query:

- [ ] Does every new data-reading path run under RLS (or is null-userId provably public)?
- [ ] Does every new mutation authenticate the caller *and* verify ownership?
- [ ] Are new external inputs (HTTP body, tool args, uploaded content) schema-validated on the server?
- [ ] Does any new code fetch a URL the user/model controls? (SSRF — denylist private ranges.)
- [ ] Is model/user text ever rendered as HTML? (Must be sanitized or catalog-bounded.)
- [ ] Do error responses avoid leaking internals (stack, `err.message`, SQL)?
- [ ] Do new files land in the correct namespace/prefix (uploads under userId)?
- [ ] Any new secret read only server-side, never `NEXT_PUBLIC_`?
- [ ] New raw SQL? Parameterized and tested against injection?

## Interview angle

- "Walk me through file-upload security." → the upload checklist above; gaps + fixes. ([Q33](../08-interview-prep/03-api-and-data-modeling-questions.md))
- "What's SSRF and is this app exposed?" → the fetch-tool analysis; how you'd *investigate* before claiming a vuln. (Q34)
- "How does this app stop XSS?" → capability-bounded generative UI vs raw HTML — a memorable, specific answer. (Q19)
