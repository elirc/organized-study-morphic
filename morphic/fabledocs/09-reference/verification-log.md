# Verification Log

Running log of what was inspected while authoring this curriculum. Anything not listed here as **verified** should be treated as **inferred**.

**Authoring date:** 2026-07-09
**Environment:** Windows 11, Node v22.16.0 available. **Bun is NOT installed on the authoring machine**, so no `bun` commands were executed. All install/dev/test commands are marked **inferred** from [package.json](../../package.json#L6-L18), [AGENTS.md](../../AGENTS.md), and [.github/workflows/ci.yml](../../.github/workflows/ci.yml).

## Commands run

| Command | Result |
| --- | --- |
| `git log --oneline` (repo) | Failed: branch `main` has no commits yet — this is a fresh snapshot without git history. Claims about "recent changes" are impossible; none are made. |
| `bun --version` | Not found (bun not installed on authoring machine) |
| `node --version` | v22.16.0 |
| `bun install` / `bun dev` / `bun run test` / `bun run build` | **NOT run** — marked inferred everywhere they appear |

## Files read in full (line-anchored claims come from these)

- [app/api/chat/route.ts](../../app/api/chat/route.ts) — chat POST endpoint, auth, rate limits, mode/model selection
- [lib/streaming/create-chat-stream-response.ts](../../lib/streaming/create-chat-stream-response.ts) — authenticated stream path
- [lib/streaming/create-ephemeral-chat-stream-response.ts](../../lib/streaming/create-ephemeral-chat-stream-response.ts) — guest stream path
- [lib/streaming/helpers/prepare-messages.ts](../../lib/streaming/helpers/prepare-messages.ts) — regeneration/edit/new-chat message prep
- [lib/streaming/helpers/persist-stream-results.ts](../../lib/streaming/helpers/persist-stream-results.ts) — post-stream persistence
- [lib/agents/researcher.ts](../../lib/agents/researcher.ts) — ToolLoopAgent construction
- [lib/agents/title-generator.ts](../../lib/agents/title-generator.ts)
- [lib/agents/prompts/search-mode-prompts.ts](../../lib/agents/prompts/search-mode-prompts.ts) — quick/adaptive prompts, citation contract
- [lib/tools/search.ts](../../lib/tools/search.ts), [lib/tools/fetch.ts](../../lib/tools/fetch.ts), [lib/tools/question.ts](../../lib/tools/question.ts), [lib/tools/todo.ts](../../lib/tools/todo.ts)
- [lib/tools/search/providers/index.ts](../../lib/tools/search/providers/index.ts), [tavily.ts](../../lib/tools/search/providers/tavily.ts), [searxng.ts](../../lib/tools/search/providers/searxng.ts)
- [lib/db/schema.ts](../../lib/db/schema.ts), [lib/db/with-rls.ts](../../lib/db/with-rls.ts), [lib/db/actions.ts](../../lib/db/actions.ts), [lib/db/index.ts](../../lib/db/index.ts)
- [lib/actions/chat.ts](../../lib/actions/chat.ts), [lib/actions/feedback.ts](../../lib/actions/feedback.ts)
- [lib/auth/get-current-user.ts](../../lib/auth/get-current-user.ts), [lib/supabase/middleware.ts](../../lib/supabase/middleware.ts), [proxy.ts](../../proxy.ts)
- [lib/rate-limit/guest-limit.ts](../../lib/rate-limit/guest-limit.ts), [chat-limits.ts](../../lib/rate-limit/chat-limits.ts), [adaptive-limit.ts](../../lib/rate-limit/adaptive-limit.ts)
- [lib/utils/registry.ts](../../lib/utils/registry.ts), [lib/models/fetch-models.ts](../../lib/models/fetch-models.ts), [lib/utils/model-selection.ts](../../lib/utils/model-selection.ts), [lib/config/load-models-config.ts](../../lib/config/load-models-config.ts), [lib/config/default-model.ts](../../lib/config/default-model.ts), [config/models/cloud.json](../../config/models/cloud.json)
- [lib/utils/context-window.ts](../../lib/utils/context-window.ts), [lib/utils/message-mapping.ts](../../lib/utils/message-mapping.ts), [lib/utils/citation.ts](../../lib/utils/citation.ts)
- [lib/render/parse-spec-block.ts](../../lib/render/parse-spec-block.ts), [spec-fence.ts](../../lib/render/spec-fence.ts), [catalog.ts](../../lib/render/catalog.ts), [schema.ts](../../lib/render/schema.ts)
- [lib/schema/search.tsx](../../lib/schema/search.tsx)
- [components/chat.tsx](../../components/chat.tsx), [components/render-message.tsx](../../components/render-message.tsx)
- [app/page.tsx](../../app/page.tsx), [app/search/[id]/page.tsx](../../app/search/[id]/page.tsx)
- [app/api/upload/route.ts](../../app/api/upload/route.ts), [app/api/chats/route.ts](../../app/api/chats/route.ts), [app/api/feedback/route.ts](../../app/api/feedback/route.ts)
- [hooks/use-file-dropzone.ts](../../hooks/use-file-dropzone.ts)
- [vitest.config.mts](../../vitest.config.mts), [.github/workflows/ci.yml](../../.github/workflows/ci.yml), `.env.local.example` (referenced upstream; absent from this snapshot), [package.json](../../package.json), [AGENTS.md](../../AGENTS.md), [README.md](../../README.md)
- [lib/db/__tests__/with-rls.test.ts](../../lib/db/__tests__/with-rls.test.ts) (first 80 lines)
- [app/api/advanced-search/route.ts](../../app/api/advanced-search/route.ts) (first 120 lines)

## Files skimmed or read partially

- `lib/tools/search/providers/exa.ts`, `brave.ts`, `firecrawl.ts` — existence and role confirmed from `providers/index.ts`; internals not read
- `components/*` beyond chat.tsx / render-message.tsx — roles inferred from names and imports
- `drizzle/*.sql` migrations — not read; schema claims come from `lib/db/schema.ts`
- `docker-compose.yaml`, `Dockerfile` — not read line-by-line; claims come from README/AGENTS.md
- `lib/analytics/*`, `lib/utils/perf-*.ts` — read only where imported by files above

## Known uncertainties (carried into risk-register.md)

1. **SSRF surface in fetch tool** — [lib/tools/fetch.ts](../../lib/tools/fetch.ts#L9-L21) fetches arbitrary URLs server-side with no private-IP/localhost blocklist visible in the file. Labeled *possible risk / investigate*; there may be infrastructure-level mitigation not visible in the repo.
2. **`/api/chats` unclamped pagination** — [app/api/chats/route.ts](../../app/api/chats/route.ts#L13-L14) parses `limit`/`offset` without bounds. Behavior with `limit=100000` not tested.
3. **Feedback update auth binding** — [app/api/feedback/route.ts](../../app/api/feedback/route.ts#L61-L68) updates message metadata by client-supplied `messageId`; when Supabase is unconfigured `userId` is null and [withOptionalRLS](../../lib/db/with-rls.ts#L80-L90) runs without RLS. Not exploited/verified; labeled investigate.
4. **`upsertMessage` conflict path drops metadata** — [lib/db/actions.ts](../../lib/db/actions.ts#L102-L107) `onConflictDoUpdate` only sets `role`. Whether metadata staleness is observable in the UI was not tested.
5. **SearXNG `site` query param** — [searxng.ts](../../lib/tools/search/providers/searxng.ts#L40-L42) sends `include_domains` as a `site` parameter; whether stock SearXNG honors this parameter was not verified.
6. **AGENTS.md drift** — [AGENTS.md](../../AGENTS.md) references `public/config/models.json`; the tree contains [config/models/cloud.json](../../config/models/cloud.json) plus dynamic fetching in [fetch-models.ts](../../lib/models/fetch-models.ts). Treated as documentation drift, verified by file inventory.
7. **Stale context-window table** — [context-window.ts](../../lib/utils/context-window.ts#L12-L38) lists older model IDs; unknown models fall back to a 16,384-token window. Truncation behavior for current default models (`gpt-5.4-mini`, Gemini 3.x) not empirically tested.
8. Runtime behavior (streaming, RLS enforcement against a live Postgres, Docker stack) was **not exercised** — no database or API keys on the authoring machine.

## Not covered

- `docs-codex-learning/` (pre-existing learning docs from another tool — intentionally untouched)
- `scripts/chat-cli.ts`, `scripts/test-cache-performance.ts` (read: no)
- `components/inspector/*`, `components/artifact/*` internals
- Supabase auth UI pages under `app/auth/*` beyond route existence
- `lib/firecrawl/*`, `lib/ollama/*` internals

## Refinement pass (2026-10-07)

Static only — nothing installed or run.

- All relative links and `#L` anchors in `fabledocs/` and `docs-codex-learning/` were resolved by script against this snapshot: no out-of-range anchors; the only dead targets were `.env.local.example` links. That file is referenced by upstream docs but is **not in this snapshot**, so the env-var section of the runtime map now cites the `process.env.<NAME>` reads instead (each name grepped and found in `lib/` or `app/`).
- R15 upgraded to *confirmed (code)*: `updateChatTitle` in `lib/db/actions.ts` L374–L388 has no `revalidateTag`, and neither does `persist-stream-results.ts`.
- New R16: `app/api/chat/route.ts` L208–L212 revalidates the chat tag before `onFinish` persists the answer, while the analytics IIFE at L175–L186 can re-read (and re-cache) the chat. Ordering confirmed by reading; the user-visible staleness was not reproduced.
