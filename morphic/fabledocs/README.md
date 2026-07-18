# Morphic Upskill Curriculum

A training lab built from the real Morphic codebase for one learner profile: a **junior fullstack JS engineer** (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how the real flows work, with exact file/line anchors.
2. **The transferable skill** — why the pattern exists, what the alternatives are, how it fails, and how to talk about it in an interview.

## What Morphic is

Morphic is an open-source AI answer engine — think Perplexity you can self-host. A user types a question; a **research agent** ([lib/agents/researcher.ts](../lib/agents/researcher.ts)) built on the Vercel AI SDK's `ToolLoopAgent` runs web searches ([lib/tools/search.ts](../lib/tools/search.ts)) against pluggable providers (Tavily, SearXNG, Exa, Brave, Firecrawl), fetches pages, and streams a cited, Markdown answer back over Server-Sent Events. Chats persist to **PostgreSQL through Drizzle ORM with row-level security** ([lib/db/schema.ts](../lib/db/schema.ts)); auth is **Supabase** with an anonymous mode for personal Docker use ([lib/auth/get-current-user.ts](../lib/auth/get-current-user.ts)); **Redis** backs rate limiting and search caching. The frontend is **Next.js 16 App Router + React 19**, with a generative-UI system where the LLM emits validated JSON "spec blocks" that render as real components ([lib/render/](../lib/render/catalog.ts)). It runs on Bun, ships via Docker or Vercel, and tests with Vitest.

It is a near-ideal training repo: it has a real security boundary (RLS + auth + rate limits), a real streaming/async core, a real plugin architecture, real caching, and real rough edges you can learn to spot.

## Learning tracks

| Track | Directory | What you get |
| --- | --- | --- |
| Cartography | [01-codebase-cartography/](01-codebase-cartography/README.md) | System map, reading order, glossary, the 7 key flows |
| Stack mastery | [02-stack-and-language-mastery/](02-stack-and-language-mastery/README.md) | JS/TS/React/Next mental models anchored to real files |
| Architecture | [03-architecture-and-patterns/](03-architecture-and-patterns/README.md) | Layers, data model, authz, async reliability, pattern catalog, critique |
| Reading gym | [04-code-reading-gym/](04-code-reading-gym/README.md) | Annotation drills, trace tables, fake-code contrasts, review katas |
| Quality | [05-quality-engineering/](05-quality-engineering/README.md) | Testing, debugging, performance, security, observability |
| Contribution | [06-contribution-practice/](06-contribution-practice/README.md) | Junior tickets → mid features → senior projects → katas |
| Career | [07-career-and-collaboration/](07-career-and-collaboration/README.md) | Review mindset, PRs/RFCs, maintainer communication |
| **Interview prep** | [08-interview-prep/](08-interview-prep/README.md) | 50+ question cards (mostly repo-anchored), system design walkthrough, STAR stories, 2-week cram plan |
| Reference | [09-reference/](09-reference/verification-log.md) | Commands, risk register, rubrics, verification log |

## How to use it

- **One weekend:** [00-fast-track.md](00-fast-track.md). Run it, trace two flows, make one safe change, teach one flow back aloud.
- **Two weeks:** Fast track → [05-key-flows.md](01-codebase-cartography/05-key-flows.md) → [pattern catalog](03-architecture-and-patterns/05-pattern-catalog.md) → 2–3 [junior tickets](06-contribution-practice/01-good-first-tickets.md) → daily drills from [04-code-reading-gym/](04-code-reading-gym/README.md).
- **Eight weeks:** All of the above, plus one [mid-level ticket](06-contribution-practice/02-mid-level-feature-tickets.md) per week, one [senior project](06-contribution-practice/03-senior-build-projects.md), and the full [interview module](08-interview-prep/README.md).
- **Ongoing contributor:** Use [06-contribution-practice/](06-contribution-practice/README.md) and [07-career-and-collaboration/](07-career-and-collaboration/README.md) as your operating manual; keep [09-reference/risk-register.md](09-reference/risk-register.md) open when reviewing.

### Recommended paths by profile

- **Brand-new junior:** 00 → 01 (all) → 02 (all) → 04 drills → first ticket. Skip 03-06/critique until week 3.
- **Junior who knows React/Node:** 00 → 01-05 (key flows) → 03 (all) → 04 → tickets. Skim 02, do its drills only.
- **Mid-level, new to this repo:** 01-01, 01-05, 03-05 (patterns), 03-06 (critique), then straight to mid tickets.
- **Senior doing architecture review:** [03-06 architecture critique](03-architecture-and-patterns/06-architecture-critique.md) + [09-reference/risk-register.md](09-reference/risk-register.md).
- **Interview in two weeks:** go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md). It schedules everything else.

## Conventions in these docs

- **Anchors:** claims about code link to exact files and lines, e.g. [app/api/chat/route.ts#L58](../app/api/chat/route.ts#L58). Line numbers were verified against the snapshot on 2026-07-09; if the repo moves, trust the file, then the function name, then the line.
- **Fake code is labeled.** Any snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Everything else is real.
- **Verified vs inferred:** commands actually executed are marked __verified__; commands read from configs but not run are marked __inferred__. See [09-reference/verification-log.md](09-reference/verification-log.md). (Bun was unavailable on the authoring machine, so all runtime commands are inferred.)
- **Suspicions are labeled** as *investigate* or *possible risk*, never asserted as bugs. Confirmed-by-reading defects are labeled as such.
- **Drills are mandatory.** Every module ends in work: annotate, trace, predict, fix, or say it aloud. Each drill has a self-grading rubric — grade yourself honestly; "Basic" answers are how you find your gaps.
- **Interview angle sections** appear wherever a concept is common interview material, with junior/mid/senior answer contrasts and cross-links into [08-interview-prep/](08-interview-prep/README.md).

## Senior vocabulary you will meet (and be interviewed on)

*Invariant* (a condition that must always hold — e.g., "a private chat's messages are only readable by its owner"), *boundary* (where one layer's responsibility ends), *contract* (the shape both sides agree on), *ownership*, *idempotency* (safe to run twice), *isolation* (tenant/user data separation), *authorization* vs *authentication*, *consistency*, *latency*, *observability*, *migration*, *rollback*, *blast radius* (how much breaks if this goes wrong). Each is defined in context where it first matters.

## The mindset ladder

- **Junior asks:** "How do I make it work?"
- **Mid-level asks:** "Is this the right pattern? What does it cost?"
- **Senior asks:** "What does this commit us to, who pays that cost, and how do we reduce the risk?"

Interviews for mid-level roles are testing exactly the second and third questions. A definition proves you studied; a tradeoff anchored to real code you've read proves you can do the job. This curriculum exists to give you those anchors.
