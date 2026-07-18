# Systematic Debugging

The method, every time: **reproduce → narrow (which layer?) → hypothesize → test cheaply → fix root cause → add regression coverage.** The narrowing question is always "is the bug in layer A or layer B?" — you binary-search the stack, not guess.

Tools in this repo: browser devtools + network tab (SSE frames), `ENABLE_PERF_LOGGING=true` for `perfLog`/`perfTime` timing ([route.ts#L25-L27](../../app/api/chat/route.ts#L25-L27), [perf-logging.ts](../../lib/utils/perf-logging.ts)), Langfuse traces ([telemetry.ts](../../lib/utils/telemetry.ts)), server `console.*`, and the Next.js MCP `get_errors`/`get_logs` (AGENTS.md#L180-L187). Query logging: Drizzle logs SQL when you enable it in [db/index.ts](../../lib/db/index.ts).

---

## Scenario 1: "My chat isn't saved after refresh"

Reproduction: send a message, refresh `/search/{id}`, message gone.
First question: is it a *write* failure or a *read/cache* failure? (Layer: persistence vs cache.)
Narrowing path:
1. Check the DB directly (`select * from messages where chat_id=…`). Row present → read/cache bug; absent → write bug.
2. If absent: the answer streamed but `onFinish` didn't persist — check server logs for the errors that [persist-stream-results.ts#L86-L97](../../lib/streaming/helpers/persist-stream-results.ts#L86-L97) logs, and whether the process was killed before `maxDuration` ([route.ts#L18](../../app/api/chat/route.ts#L18)).
3. If present: cache staleness — `loadChat` has a 60s TTL ([lib/actions/chat.ts#L29-L30](../../lib/actions/chat.ts#L29-L30)); did the write path `revalidateTag`? (Compare Table 5 in [trace-tables](../04-code-reading-gym/02-trace-tables.md).)
Useful probes: server logs, the perf timings, a raw SQL query.
Likely root causes: instance killed mid-stream (write); missing `revalidateTag` on some path (read).
Regression test: mock the persist path, assert `upsertMessage` called with the response message; for cache, an action test asserting `revalidateTag('chat-{id}')` fired.
Senior lesson: "not saved" splits cleanly into write-vs-cache — never debug both at once.

---

## Scenario 2: "Regenerating lost my previous answer when the model errored"

Reproduction: click regenerate; force an LLM error (bad key); the old answer is gone.
First question: is deletion happening before or after the new stream? (It's before — [prepare-messages.ts#L57-L64](../../lib/streaming/helpers/prepare-messages.ts#L57-L64).)
Narrowing path:
1. Confirm deletion timing: add a log before `deleteMessagesFromIndex` and one in the stream `onError`. Delete fires first → confirmed compensation gap.
2. This is not a bug in the delete or the stream individually — it's the *ordering* (a design flaw, Flow 5).
Root cause: no compensation/rollback around a destructive op.
Fix: delete after successful regeneration, or soft-delete + sweep (architecture-critique #2).
Regression test: mock the model to reject, assert the pre-existing assistant message still exists afterward.
Senior lesson: some "bugs" are correct code in the wrong order — the fix is sequencing, not a patch.

---

## Scenario 3: "Long conversations give worse answers over time"

Reproduction: a 40-message chat; later answers ignore earlier context.
First question: is context being *dropped* (truncation) or is the model just bad? (Layer: context-window vs model.)
Narrowing path:
1. Enable perf logging / add a log of `modelMessages.length` before/after truncation ([create-chat-stream-response.ts#L149-L159](../../lib/streaming/create-chat-stream-response.ts#L149-L159)).
2. Check the model id against the context table ([context-window.ts#L12-L38](../../lib/utils/context-window.ts#L12-L38)). Current defaults (`gpt-5.4-mini`, Gemini 3.x) aren't listed → 16k fallback → aggressive truncation. **This is the confirmed stale-table risk.**
3. Confirm: does the truncated set actually fit the model's real window? If the real window is 128k but we truncate to ~13k, we're throwing away good context.
Root cause: hardcoded, stale model→window map.
Fix: source windows from the models config or use a generous default + log unknown models.
Regression test: `getMaxAllowedTokens` for the current default returns a sane budget, and unknown models log.
Senior lesson: silent-degradation bugs need a *log at the decision point* — you can't see truncation from the outside.

---

## Scenario 4: "Guest gets rate-limited immediately / never"

Reproduction: guest requests blocked at request 1, or never blocked.
First question: is the limiter even *running*? (It no-ops outside cloud/without Upstash — [guest-limit.ts#L34-L43](../../lib/rate-limit/guest-limit.ts#L34-L43).)
Narrowing path:
1. "Never blocked": check `MORPHIC_CLOUD_DEPLOYMENT` and Upstash vars — most likely the limiter is fail-open by config.
2. "Immediately blocked": inspect the IP key. Behind a proxy, `x-forwarded-for` might be the proxy's IP → *all guests share one bucket* ([route.ts#L77-L82](../../app/api/chat/route.ts#L77-L82)). Log the derived `ip`.
3. Or a stuck key: INCR succeeded, EXPIRE failed once → immortal key at yesterday's high count. Check `TTL rl:guest:chat:{ip}:{date}` in Redis.
Root causes: config no-op; shared-IP bucketing; immortal key (INCR/EXPIRE non-atomicity, Pattern 5).
Regression test: limiter test asserting no-op paths return `null`; enforcement path returns 401 at count>limit (exists — [guest-limit.test.ts](../../lib/rate-limit/__tests__/guest-limit.test.ts)).
Senior lesson: before debugging a limiter's *math*, prove it's *enabled* — half of "rate limit bugs" are "it wasn't running."

---

## Scenario 5: "A citation link points nowhere / is missing"

Reproduction: model wrote `[1](#toolCallId)` but the rendered link is gone.
First question: did the model emit a *valid* id, or did the parser drop it? (Layer: prompt/model vs parser.)
Narrowing path:
1. Inspect the raw assistant text (before processing) in the DB `text_text` — is the `[1](#id)` there with a real toolCallId?
2. Check the citation map for that message: does `extractCitationMaps` find a `tool-search` output with a `citationMap`? ([citation.ts#L28-L42](../../lib/utils/citation.ts#L28-L42)) If the search errored, there's no map → all its citations drop.
3. `processCitations` silently deletes unmatched/invalid citations ([citation.ts#L88-L100](../../lib/utils/citation.ts#L88-L100)) — so "missing link" often means "model cited a search that didn't return that result".
Root cause: usually model error (fabricated/duplicated id) hidden by soft-fail; occasionally a real map-extraction bug.
Fix/observability: the *right* fix is often a metric on dropped citations, not a code change — you can't currently see model citation errors.
Regression test: `processCitations` with an id absent from the map returns empty (exists in [citation.test.ts](../../lib/utils/__tests__/citation.test.ts)).
Senior lesson: soft-fail is user-friendly and debugging-hostile — add observability where you swallow.

---

## Interview version (how to narrate a debugging round)

Say the method out loud, then drive it: *"First I'd reproduce and decide which layer — write or cache. I'd query the DB directly: if the row's there, it's a cache/revalidation issue; if not, persistence. Given this repo persists in the stream's onFinish, I'd suspect the instance died before maxDuration, and I'd check the logs that persist-stream-results emits…"* Naming the binary split and the specific probe *before* touching code is what interviewers score. Timed rounds in [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

Drill: for each scenario, write the single cheapest probe that would confirm/deny your first hypothesis. Strong = your probe distinguishes the two layers in one observation.
