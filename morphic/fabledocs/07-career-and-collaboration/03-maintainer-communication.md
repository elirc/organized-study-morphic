# Maintainer Communication

The skill: get unblocked without outsourcing your thinking, and disagree without friction. Morphic is a real OSS project ([CONTRIBUTING.md](../../CONTRIBUTING.md), [CODE_OF_CONDUCT.md](../../CODE_OF_CONDUCT.md), issue templates in [.github/ISSUE_TEMPLATE/](../../.github/ISSUE_TEMPLATE/)) — these are the norms for contributing to it and any OSS repo.

## Asking for help (the 15-minute rule + show your work)

Struggle for ~15 minutes, then ask — but ask with your work shown, so the maintainer corrects your thinking rather than doing it for you.

Bad: "How does auth work here?"
Good:
> I'm adding a health route and want to confirm the auth model. My read: middleware ([proxy.ts](../../proxy.ts)) treats `/api` as public ([supabase/middleware.ts#L44-L50](../../lib/supabase/middleware.ts#L44-L50)), so API routes authenticate themselves via `getCurrentUserId`. For a health check I'd skip auth entirely and just `SELECT 1`. Is skipping auth on `/api/health` acceptable, or do you want it gated?

The good version proves you read the code, states a hypothesis, and asks a decision you genuinely can't make alone. Maintainers answer these fast because they're cheap to answer.

## Bug reports (repro + minimal example)

Use the [bug template](../../.github/ISSUE_TEMPLATE/bug_report.yml). A good report:
1. **Repro steps** — exact, numbered.
2. **Expected vs actual.**
3. **Minimal example** — strip it to the smallest failing case.
4. **Environment** — deployment mode (Docker/anonymous/cloud), provider, env flags (these change behavior a lot here — [04-runtime-and-tooling-map.md](../01-codebase-cartography/04-runtime-and-tooling-map.md)).
5. **What you already checked** — "row exists in DB, so it's a cache issue not a write issue" (Scenario 1). This alone moves a report from "triage" to "actionable".

Example:
> **Bug:** chat not visible after refresh. **Repro:** 1) `ENABLE_AUTH=false`, 2) send a message, 3) refresh `/search/{id}`. **Expected:** message shown. **Actual:** empty. **Checked:** the `messages` row exists in Postgres, so persistence worked — looks like the title/cache path didn't `revalidateTag` (compare [shareChat](../../lib/actions/chat.ts#L265-L282) which does). **Env:** local, OpenAI, `ENABLE_SAVE_CHAT_HISTORY=true`.

## Proposing a feature

Open an issue ([feature template](../../.github/ISSUE_TEMPLATE/feature_request.yml)) before a big PR. State the problem, the user value, and a sketch — then ask if it aligns before investing. For anything touching contracts/migrations, offer an RFC ([02-writing-prs-and-rfcs.md](02-writing-prs-and-rfcs.md)). Unsolicited large PRs get rejected not because they're bad but because they weren't agreed — respect the maintainer's roadmap.

## Responding to review

- **Agree fast:** "Good catch, fixed in {commit}." Don't defend a real bug.
- **Disagree respectfully with evidence:** "I chose X because {reason + anchor}; Y would {tradeoff}. Happy to switch if you'd rather — your call." Then *actually* defer; the maintainer owns the codebase.
- **Ask when unsure:** "Do you want this behind a flag, or is a direct change fine here?"
- **Don't relitigate** a decision the maintainer already made — note your concern once, move on.

## Respectful disagreement (a template)

> I see your point on {their concern}. My worry with that approach is {specific risk + anchor}. Two options: (a) {yours}, (b) {theirs}. I lean (a) because {reason}, but I don't feel strongly — which do you prefer? I'll go with your call.

This states your position *and* your willingness to defer in one message. It's how you disagree without being difficult — a directly-interviewed skill.

## Interview angle

- "How do you get unblocked?" → 15-min rule + show-your-work; the point is *not* outsourcing the thinking.
- "Tell me about a disagreement with a senior/maintainer." → the template above; the STAR is "I made my case with evidence, then committed to their decision." ([Q49](../08-interview-prep/06-behavioral-star-stories.md))
- "How would you contribute to an OSS project you didn't write?" → read first (this whole curriculum), issue-before-big-PR, small blast radius, follow existing patterns.

Drill: write the "asking for help" message for Ticket M10 (SSRF guard) — you're unsure whether to block on hostname or resolved IP. Self-grade — Strong: you state your current understanding (DNS-rebinding means you must check the *resolved* IP), propose an approach, and ask one specific decision.
