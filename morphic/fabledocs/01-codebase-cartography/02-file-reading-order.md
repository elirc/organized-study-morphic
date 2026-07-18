# File Reading Order

30 files. Read in order; each entry says why it matters, what to look for, what to skip. Junior path = 1–14. Mid adds 15–24. Senior adds 25–30.

## Junior path (locate & explain)

1. [package.json](../../package.json) — scripts + deps. Look for: `ai`, `drizzle-orm`, `@supabase/ssr`. Ignore: the radix-ui wall.
2. [README.md](../../README.md) — features list = product surface. Ignore badges.
3. [AGENTS.md](../../AGENTS.md) — commands + architecture notes. *Trap:* it references `public/config/models.json`, which doesn't exist in this tree — the real config is [config/models/cloud.json](../../config/models/cloud.json) plus dynamic fetching. Docs drift is normal; code wins.
4. [app/page.tsx](../../app/page.tsx) — home page is 19 lines. Server component fetching userId + model data, rendering `<Chat/>`.
5. [app/api/chat/route.ts](../../app/api/chat/route.ts) — THE endpoint. Look for ordering of gates. What to skip: perf-logging noise.
6. [components/chat.tsx](../../components/chat.tsx) — `useChat`, request shaping ([L102-L129](../../components/chat.tsx#L102-L129)), error-modal parsing ([L135-L214](../../components/chat.tsx#L135-L214)).
7. [lib/streaming/create-chat-stream-response.ts](../../lib/streaming/create-chat-stream-response.ts) — orchestration. Look for what happens in `onFinish` vs before streaming.
8. [lib/agents/researcher.ts](../../lib/agents/researcher.ts) — agent config. Look for `activeTools` and `stopWhen`.
9. [lib/tools/search.ts](../../lib/tools/search.ts) — `async *execute` generator; provider selection.
10. [lib/tools/search/providers/index.ts](../../lib/tools/search/providers/index.ts) + [tavily.ts](../../lib/tools/search/providers/tavily.ts) — factory + one concrete provider.
11. [lib/db/schema.ts](../../lib/db/schema.ts) — three core tables + `feedback`. Look for: cascade deletes ([L79-L81](../../lib/db/schema.ts#L79-L81)), check constraints ([L207-L224](../../lib/db/schema.ts#L207-L224)), RLS policies.
12. [lib/db/actions.ts](../../lib/db/actions.ts) — low-level CRUD. Look for `upsertMessage`'s delete-then-insert of parts ([L110-L118](../../lib/db/actions.ts#L110-L118)).
13. [lib/actions/chat.ts](../../lib/actions/chat.ts) — server actions + `unstable_cache` ([L17-L34](../../lib/actions/chat.ts#L17-L34)).
14. [lib/auth/get-current-user.ts](../../lib/auth/get-current-user.ts) — the whole identity story in 44 lines.

## Mid path (modify safely)

15. [lib/db/with-rls.ts](../../lib/db/with-rls.ts) — transaction-scoped `set_config`. This is the tenant-isolation mechanism.
16. [lib/streaming/helpers/prepare-messages.ts](../../lib/streaming/helpers/prepare-messages.ts) — regenerate/edit/new-chat branching. Hardest logic in the repo per line.
17. [lib/streaming/helpers/persist-stream-results.ts](../../lib/streaming/helpers/persist-stream-results.ts) — reliability: awaiting the optimistic insert, duplicate-key-as-success, retry.
18. [lib/streaming/create-ephemeral-chat-stream-response.ts](../../lib/streaming/create-ephemeral-chat-stream-response.ts) — guest path; diff it mentally against #7.
19. [lib/rate-limit/adaptive-limit.ts](../../lib/rate-limit/adaptive-limit.ts) — INCR+EXPIRE, timeout race, fail-open. Compare [guest-limit.ts](../../lib/rate-limit/guest-limit.ts) and [chat-limits.ts](../../lib/rate-limit/chat-limits.ts) — near-triplicates (a refactor kata).
20. [lib/utils/registry.ts](../../lib/utils/registry.ts) + [lib/utils/model-selection.ts](../../lib/utils/model-selection.ts) + [lib/models/fetch-models.ts](../../lib/models/fetch-models.ts) — the model stack: registry → selection policy → dynamic discovery with TTL cache.
21. [lib/utils/message-mapping.ts](../../lib/utils/message-mapping.ts) — 775 lines of UI↔DB part mapping. Skim the repetition; read [L127-L339](../../lib/utils/message-mapping.ts#L127-L339) closely. Ask: what would make this smaller? (A discriminated-union table — see refactor katas.)
22. [lib/utils/context-window.ts](../../lib/utils/context-window.ts) — token budgeting + truncation strategy.
23. [lib/agents/prompts/search-mode-prompts.ts](../../lib/agents/prompts/search-mode-prompts.ts) — prompts as code; the citation contract.
24. [lib/utils/citation.ts](../../lib/utils/citation.ts) + [components/render-message.tsx](../../components/render-message.tsx) — model output → rendered UI.

## Senior path (critique & design)

25. [lib/render/parse-spec-block.ts](../../lib/render/parse-spec-block.ts) + [spec-fence.ts](../../lib/render/spec-fence.ts) + [catalog.ts](../../lib/render/catalog.ts) — streaming-tolerant parser with prefix-diff incremental compilation ([parse-spec-block.ts#L72-L79](../../lib/render/parse-spec-block.ts#L72-L79)). The most senior code in the repo.
26. [lib/db/index.ts](../../lib/db/index.ts) — connection policy: restricted-user preference ([L25-L28](../../lib/db/index.ts#L25-L28)), SSL toggles, startup verification.
27. [proxy.ts](../../proxy.ts) + [lib/supabase/middleware.ts](../../lib/supabase/middleware.ts) — middleware, session refresh, public-path list ([middleware.ts#L44-L50](../../lib/supabase/middleware.ts#L44-L50)).
28. [app/api/upload/route.ts](../../app/api/upload/route.ts) — a compact security checklist: auth, size, MIME whitelist, filename sanitization, storage-config gating.
29. [app/api/feedback/route.ts](../../app/api/feedback/route.ts) + [lib/actions/feedback.ts](../../lib/actions/feedback.ts) — observability write-path; also an authz teaching seam (see risk register).
30. [.github/workflows/ci.yml](../../.github/workflows/ci.yml) + [vitest.config.mts](../../vitest.config.mts) — what CI actually enforces: lint, typecheck, format, test, build; dummy `DATABASE_URL` for tests ([vitest.config.mts#L5-L7](../../vitest.config.mts#L5-L7)).

## How to read (senior noticing habits)

For every file ask: What are the inputs and who validates them? What side effects fire and when? What invariant does this file protect? What would break if I called it twice? Where are its tests? Write the answers down for files 5, 7, 12, 16 — that's [04-code-reading-gym/01-annotation-drills.md](../04-code-reading-gym/01-annotation-drills.md).
