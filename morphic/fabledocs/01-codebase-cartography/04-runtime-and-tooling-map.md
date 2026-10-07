# Runtime and Tooling Map

## Toolchain

| Tool | Command | Verified? | Notes |
| --- | --- | --- | --- |
| Bun 1.2.12 | `bun install` | inferred | pinned in [package.json#L107-L109](../../package.json#L107-L109); CI uses the same pin ([ci.yml#L20-L22](../../.github/workflows/ci.yml#L20-L22)) |
| Dev server | `bun dev` | inferred | Next.js dev (Turbopack per [AGENTS.md#L9](../../AGENTS.md#L9)) |
| Build | `bun run build` | inferred | needs `DATABASE_URL` even in CI ([ci.yml#L110-L112](../../.github/workflows/ci.yml#L110-L112)) |
| Lint | `bun lint` | inferred | ESLint 9 flat config + `simple-import-sort` ([eslint.config.mjs](../../eslint.config.mjs)) |
| Types | `bun typecheck` | inferred | `tsc --noEmit` |
| Format | `bun format:check` | inferred | Prettier is CI-enforced — a PR fails on formatting alone |
| Tests | `bun run test` | inferred | **not** `bun test` (Bun's own runner lacks Vitest features — [AGENTS.md#L17](../../AGENTS.md#L17)) |
| Migrations | `bun migrate` | inferred | runs [lib/db/migrate.ts](../../lib/db/migrate.ts); SQL lives in [drizzle/](../../drizzle/) |
| CLI chat | `bun chat` | inferred | [scripts/chat-cli.ts](../../scripts/chat-cli.ts) |

CI ([ci.yml](../../.github/workflows/ci.yml)) runs lint, typecheck, format, test in parallel; build gates on all four. Tests run with a **dummy** `DATABASE_URL` ([ci.yml#L88-L90](../../.github/workflows/ci.yml#L88-L90)) — so no test in CI touches a real database; the "integration" RLS test ([lib/db/__tests__/rls-policies.integration.test.ts](../../lib/db/__tests__/rls-policies.integration.test.ts)) only means something when run locally against Postgres.

## Runtime boundaries

Three places code runs; confusing them is the #1 junior mistake in App Router codebases:

1. **Server (Node)** — API routes ([app/api/](../../app/api/chat/route.ts)), server components ([app/page.tsx](../../app/page.tsx), [app/search/[id]/page.tsx](../../app/search/[id]/page.tsx)), server actions ([lib/actions/chat.ts#L1](../../lib/actions/chat.ts#L1) `'use server'`), everything in `lib/db`, `lib/agents`, `lib/tools`. Can read secrets, hit Postgres.
2. **Edge-ish middleware** — [proxy.ts](../../proxy.ts) runs on every non-static request (matcher at [L42-L53](../../proxy.ts#L42-L53)): refreshes the Supabase session, stamps `x-base-url` headers so server code can reconstruct its own origin ([lib/utils/url.ts](../../lib/utils/url.ts) consumes them; the search tool uses this to call its own advanced-search route at [search.ts#L88-L101](../../lib/tools/search.ts#L88-L101)).
3. **Browser** — anything with `'use client'` ([components/chat.tsx#L1](../../components/chat.tsx#L1)). Only `NEXT_PUBLIC_*` env vars exist here.

`maxDuration = 300` on the chat route ([route.ts#L18](../../app/api/chat/route.ts#L18)) and `60` on the search page ([app/search/[id]/page.tsx#L11](../../app/search/[id]/page.tsx#L11)) are Vercel function-timeout hints — a deploy-platform contract living in code.

## Environment variables (high level, no secrets)

Grouped from the `process.env.*` reads in `lib/`, `app/` and [docs/CONFIGURATION.md](../../docs/CONFIGURATION.md). Upstream ships a `.env.local.example`, but dotfiles were not imported into this snapshot, so every name below was re-verified by grepping `process.env.<NAME>` in the code:

- **Model providers:** `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY`, `AI_GATEWAY_API_KEY`, `OLLAMA_BASE_URL`, `OPENAI_COMPATIBLE_*`. Presence of a key *is* the feature flag — [isProviderEnabled](../../lib/utils/registry.ts#L56-L76).
- **Search:** `SEARCH_API` picks the provider; `TAVILY_API_KEY`, `EXA_API_KEY`, `BRAVE_SEARCH_API_KEY`, `FIRECRAWL_API_KEY`, `SEARXNG_*` family.
- **Persistence:** `DATABASE_URL` (or `DATABASE_RESTRICTED_URL`, preferred at [lib/db/index.ts#L25-L28](../../lib/db/index.ts#L25-L28) so RLS actually binds), `DATABASE_SSL_DISABLED`.
- **Auth:** `ENABLE_AUTH`, `ANONYMOUS_USER_ID`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `ENABLE_GUEST_CHAT`.
- **Cloud/limits:** `MORPHIC_CLOUD_DEPLOYMENT`, `UPSTASH_REDIS_REST_URL/TOKEN`, `GUEST_CHAT_DAILY_LIMIT`, `ADAPTIVE_CHAT_DAILY_LIMIT`.
- **Uploads:** `R2_*` / `S3_ENDPOINT` family — upload is disabled unless fully configured ([upload/route.ts#L107-L115](../../app/api/upload/route.ts#L107-L115)).
- **Observability:** `ENABLE_LANGFUSE_TRACING`, `LANGFUSE_*`, `ENABLE_PERF_LOGGING`.

**Transferable concept — env-driven feature gating:** features here activate on *configuration presence* rather than a flag service. Cheap and honest, but it means behavior differs between environments in ways tests don't cover; note how every rate limiter silently no-ops when Upstash is absent ([guest-limit.ts#L38-L42](../../lib/rate-limit/guest-limit.ts#L38-L42)). Failure mode: you believe a limit is enforced in an env where it isn't.

## Vitest setup

[vitest.config.mts](../../vitest.config.mts): jsdom environment, globals on, `@` alias, and a **dummy `DATABASE_URL` injected at config time** ([L5-L7](../../vitest.config.mts#L5-L7)) so importing `lib/db` doesn't throw during collection — pair with the guard in [lib/db/index.ts#L13-L21](../../lib/db/index.ts#L13-L21). Tests colocate under `__tests__/` next to sources.

Drill: predict which of these break at *build* vs *runtime* if unset: `DATABASE_URL`, `TAVILY_API_KEY`, `NEXT_PUBLIC_SUPABASE_URL`. Verify: DB URL throws at module import ([db/index.ts#L13-L21](../../lib/db/index.ts#L13-L21)); Tavily fails inside the tool call ([tavily.ts#L19-L20](../../lib/tools/search/providers/tavily.ts#L19-L20)); missing Supabase vars just mean `getCurrentUser` returns null ([get-current-user.ts#L6-L11](../../lib/auth/get-current-user.ts#L6-L11)).
