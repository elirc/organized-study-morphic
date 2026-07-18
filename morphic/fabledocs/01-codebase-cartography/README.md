# 01 — Codebase Cartography

Before you can change a codebase safely, you need a map: what the modules are, who owns what, and how the 7 flows that matter actually move data. This module builds that map and makes you verify it against real files.

| File | What it gives you |
| --- | --- |
| [01-system-map.md](01-system-map.md) | Repo shape, ownership map, public vs private surfaces |
| [02-file-reading-order.md](02-file-reading-order.md) | 30 files in reading order, split junior/mid/senior |
| [03-domain-glossary.md](03-domain-glossary.md) | Product nouns → code locations; near-synonym traps |
| [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) | Bun, Next.js runtimes, env vars, build/test tooling |
| [05-key-flows.md](05-key-flows.md) | **The core artifact:** 7 end-to-end flow traces with drills |

Transferable skill: *cartography is the first thing a senior does in any new repo* — find the entry points, find the persistence, find the security boundary, then read one full flow before touching anything. Interviewers ask "how do you approach an unfamiliar codebase?"; this module is your worked answer.
