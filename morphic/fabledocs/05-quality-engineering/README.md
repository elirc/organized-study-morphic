# 05 — Quality Engineering

The disciplines that separate "it works on my machine" from "it works in production and I'd know if it didn't."

| File | Covers |
| --- | --- |
| [01-testing-strategy.md](01-testing-strategy.md) | What's tested here, the test pyramid, flake prevention |
| [02-writing-tests-here.md](02-writing-tests-here.md) | Concrete recipes using the repo's real helpers |
| [03-systematic-debugging.md](03-systematic-debugging.md) | Reproduce→narrow→fix→regress, on 5 repo-specific scenarios |
| [04-performance-thinking.md](04-performance-thinking.md) | Perf domains, hotspots, measure-first |
| [05-security-checklist.md](05-security-checklist.md) | Authz/IDOR/SSRF/XSS/uploads mapped to real code + pre-merge checklist |
| [06-observability-and-operations.md](06-observability-and-operations.md) | Logging/tracing/health; "how would I know this broke?" per flow |

The honest framing for this repo: it has *good* tests in a few areas (render, rate-limit, RLS, utils) and *no* tests in others (routes, tools, message mapping). That gap is a feature for learning — it's where your contributions have the highest value and where you practice arguing coverage in reviews.
