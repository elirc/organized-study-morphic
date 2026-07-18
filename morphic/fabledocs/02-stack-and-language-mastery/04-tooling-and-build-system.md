# Tooling and Build System

The transferable skill: read a repo's toolchain in ten minutes and know exactly what gates a PR. Here that's five CI jobs ([ci.yml](../../.github/workflows/ci.yml)): lint, typecheck, format, test → build.

## Bun as package manager + script runner

- Pinned: `"engines": { "bun": "1.2.12" }` ([package.json#L107-L109](../../package.json#L107-L109)); CI pins the same ([ci.yml#L20-L22](../../.github/workflows/ci.yml#L20-L22)). Version pinning across dev/CI is the cheap way to kill "works on my machine".
- `bun.lock` is the lockfile — commit it, never hand-edit.
- Trap (documented in [AGENTS.md#L17](../../AGENTS.md#L17)): `bun test` invokes Bun's built-in runner, which would half-work and confuse you; the script `bun run test` runs Vitest with `NODE_ENV=test` ([package.json#L15](../../package.json#L15)). Lesson: *always read the scripts block before typing a familiar command*.

## ESLint 9 flat config + import sorting

[eslint.config.mjs](../../eslint.config.mjs) uses the flat-config format with `eslint-plugin-simple-import-sort`. Import order is CI-enforced — `bun lint --fix` (inferred) auto-sorts. Why teams do this: diffs stop churning on import order, and merge conflicts in import blocks disappear. Notice the sorted groups in any file, e.g. [chat.tsx#L3-L29](../../components/chat.tsx#L3-L29): react/next → vendors → `@/lib` → `@/hooks` → relative.

## Prettier as a CI gate

`bun format:check` fails the pipeline on formatting ([ci.yml#L48-L65](../../.github/workflows/ci.yml#L48-L65)). Transferable rule: **format on save or lose CI minutes** — set up your editor before your first commit ([.vscode/settings.json](../../.vscode/settings.json) exists for this).

## TypeScript

`bun typecheck` = `tsc --noEmit` — types are checked separately from the build (Turbopack/Next transpiles without typechecking for speed). Consequence: a PR can *run* locally with type errors; only `typecheck` catches them. Path alias `@/*` defined in [tsconfig.json](../../tsconfig.json) and mirrored for Vitest at [vitest.config.mts#L11-L15](../../vitest.config.mts#L11-L15) — aliases must be declared in *every* tool that resolves modules (tsc, bundler, test runner); mismatches are a classic "works in dev, fails in test" cause. Note the second alias there: `'zod/v4': 'zod'` — a version-bridge shim.

## Vitest

[vitest.config.mts](../../vitest.config.mts): jsdom environment (so component tests run), `globals: true` (no imports for `describe/it`), setup file [vitest.setup.ts](../../vitest.setup.ts), and the dummy `DATABASE_URL` at config time ([L5-L7](../../vitest.config.mts#L5-L7)) so importing the db module during collection doesn't throw ([lib/db/index.ts#L13-L21](../../lib/db/index.ts#L13-L21) explicitly allows the test env). Tests colocate in `__tests__/` directories next to the code — find any with `Glob **/__tests__/*.test.ts*`.

Targeted runs (inferred): `bun run test lib/rate-limit/__tests__/guest-limit.test.ts`, watch mode via `bun run test:watch`.

## Drizzle Kit (migrations)

Flow: edit [lib/db/schema.ts](../../lib/db/schema.ts) → `drizzle-kit generate` (config: [drizzle.config.ts](../../drizzle.config.ts)) emits SQL into [drizzle/](../../drizzle/) (11 migrations, `0000_black_lifeguard.sql` … `0010_lonely_kang.sql`, plus `meta/` snapshots) → `bun migrate` applies via [lib/db/migrate.ts](../../lib/db/migrate.ts). Rules: never edit generated SQL or snapshots by hand; migrations are append-only history (that's why there are 11 files, not 1 evolving file). The whimsical names are Drizzle's auto-naming.

## Docker & deploy surfaces

`docker-compose.yaml` runs Postgres 17 + Redis + SearXNG + the app ([AGENTS.md#L22](../../AGENTS.md#L22), inferred — compose file not read line-by-line); GitHub Actions also builds/publishes images ([docker-build.yml](../../.github/workflows/docker-build.yml), [release.yml](../../.github/workflows/release.yml)). Vercel deploy uses `maxDuration` hints in routes ([route.ts#L18](../../app/api/chat/route.ts#L18)).

## Pre-PR ritual (from [AGENTS.md#L140-L152](../../AGENTS.md#L140-L152), inferred)

```bash
bun lint && bun typecheck && bun format:check && bun run build && bun run test
```

Run it before pushing; CI runs exactly this and PRs fail on any step.

## Drills

1. Without opening configs: which job catches (a) an unused variable, (b) a tab where spaces belong, (c) a broken import path, (d) a failing RLS unit test, (e) a missing env var at build? Then verify against [ci.yml](../../.github/workflows/ci.yml). (a=lint, b=format, c=typecheck *and* build, d=test, e=build — note DATABASE_URL is stubbed at [L112](../../.github/workflows/ci.yml#L110-L112).)
2. Trace the `@/lib/db` import in a test file through both resolvers (tsconfig paths, vitest alias). What happens if only one is set?

## Interview angle

- "What does your CI run and why in that order?" → parallel cheap checks gate the expensive build; cite the `needs:` at [ci.yml#L95](../../.github/workflows/ci.yml#L92-L95). ([Q10](../08-interview-prep/01-js-ts-node-deep-dive.md))
- "How do schema migrations work in your team?" → generate-from-schema, append-only, review the SQL; cite the drizzle/ layout. (Q27)
- "A test passes locally, fails in CI" → env/alias/version drift checklist, in that order; this repo pins all three (env stub, alias mirror, bun pin).
