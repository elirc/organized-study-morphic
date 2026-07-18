# Command Cheatsheet

All marked __inferred__ unless stated — Bun was not installed on the authoring machine, so none were executed here. Sources: [package.json#L6-L18](../../package.json#L6-L18), [AGENTS.md](../../AGENTS.md), [.github/workflows/ci.yml](../../.github/workflows/ci.yml).

## Setup

| Command | Purpose | Verified? |
| --- | --- | --- |
| `bun install` | install deps (pin: bun 1.2.12) | inferred |
| `cp .env.local.example .env.local` | create env file | inferred |
| `bun migrate` | apply DB migrations ([lib/db/migrate.ts](../../lib/db/migrate.ts)) | inferred |

## Develop

| Command | Purpose |
| --- | --- |
| `bun dev` | dev server (Turbopack) → :3000 |
| `bun run build` | production build (needs `DATABASE_URL`) |
| `bun start` | run production build |
| `bun chat` | CLI chat ([scripts/chat-cli.ts](../../scripts/chat-cli.ts)) |

## Quality gates (the pre-PR ritual)

| Command | Purpose |
| --- | --- |
| `bun lint` | ESLint + import sort |
| `bun lint --fix` | auto-fix + sort imports |
| `bun typecheck` | `tsc --noEmit` |
| `bun format` | Prettier write |
| `bun format:check` | Prettier check (CI-enforced) |
| `bun run build` | build must succeed |
| `bun run test` | Vitest (**not** `bun test`) |

Full ritual before push (inferred, from [AGENTS.md#L140-L152](../../AGENTS.md#L140-L152)):
```bash
bun lint && bun typecheck && bun format:check && bun run build && bun run test
```

## Tests

| Command | Purpose |
| --- | --- |
| `bun run test` | all tests |
| `bun run test:watch` | watch mode |
| `bun run test lib/utils/__tests__/citation.test.ts` | one file (inferred) |
| `bun run test lib/rate-limit` | one directory (inferred) |

Note: CI stubs `DATABASE_URL` ([ci.yml#L88-L90](../../.github/workflows/ci.yml#L88-L90)), so the RLS integration test ([lib/db/__tests__/rls-policies.integration.test.ts](../../lib/db/__tests__/rls-policies.integration.test.ts)) only exercises real policies when run locally against Postgres.

## Database (Drizzle Kit)

| Command | Purpose |
| --- | --- |
| `drizzle-kit generate` | generate SQL migration from [schema.ts](../../lib/db/schema.ts) (inferred; config [drizzle.config.ts](../../drizzle.config.ts)) |
| `bun migrate` | apply migrations |

Never hand-edit files in [drizzle/](../../drizzle/) — they're generated + append-only.

## Docker

| Command | Purpose |
| --- | --- |
| `docker compose up -d` | Postgres 17 + Redis + SearXNG + app |
| `docker compose down` | stop |
| `docker compose down -v` | stop + delete volumes (wipes DB) |
| `docker pull ghcr.io/miurla/morphic:latest` | prebuilt image |

## Useful env toggles (debugging)

| Var | Effect |
| --- | --- |
| `ENABLE_PERF_LOGGING=true` | `perfLog`/`perfTime` output ([route.ts#L25-L27](../../app/api/chat/route.ts#L25-L27)) |
| `ENABLE_LANGFUSE_TRACING=true` | LLM traces |
| `ENABLE_AUTH=false` | anonymous mode (blocked on cloud) |
| `MORPHIC_CLOUD_DEPLOYMENT=true` | enables rate limits, pinned models |
| `DATABASE_SSL_DISABLED=true` | local PG without SSL |

## Setup notes for this machine

- `node` = v22.16.0 (verified). `bun` not installed — install from https://bun.sh before running any command above.
- Use the **Bash** or **PowerShell** tool; the repo lives at `mastermorphic/morphic/`.
- Git note: this snapshot has **no commits yet** (`git log` fails), so there's no history to mine.
