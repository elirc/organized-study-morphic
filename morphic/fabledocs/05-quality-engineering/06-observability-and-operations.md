# Observability and Operations

"How would I know this broke?" — asked of every flow. Observability is the answer being "an alert/log/trace" instead of "a user emails us."

## What's instrumented

| Signal | Mechanism | Anchor |
| --- | --- | --- |
| LLM traces | Langfuse (opt-in) | [create-chat-stream-response.ts#L79-L94](../../lib/streaming/create-chat-stream-response.ts#L79-L94), flush in `finally` [L213-L227](../../lib/streaming/create-chat-stream-response.ts#L213-L227) |
| Request timing | `perfLog`/`perfTime` behind `ENABLE_PERF_LOGGING` | [route.ts#L214-L219](../../app/api/chat/route.ts#L214-L219), [perf-logging.ts](../../lib/utils/perf-logging.ts) |
| Op counters | `incrementAuthCallCount` / `incrementDbOperationCount` | [perf-tracking.ts](../../lib/utils/perf-tracking.ts), reset per request [route.ts#L25-L27](../../app/api/chat/route.ts#L25-L27) |
| Product analytics | `trackChatEvent`, `trackAdaptiveLimitEvent` | [route.ts#L173-L206](../../app/api/chat/route.ts#L173-L206), [analytics/](../../lib/analytics/index.ts) |
| User feedback | thumbs → Langfuse score + message metadata | [feedback/route.ts](../../app/api/feedback/route.ts), [actions/feedback.ts](../../lib/actions/feedback.ts) |
| Errors | `console.error`/`console.warn` everywhere | e.g. [persist-stream-results.ts#L94-L97](../../lib/streaming/helpers/persist-stream-results.ts#L94-L97) |
| DB startup health | current-user verification | [db/index.ts#L68-L100](../../lib/db/index.ts#L68-L100) |
| App telemetry hooks | OpenTelemetry via `@vercel/otel` | [instrumentation.ts](../../instrumentation.ts) |

## The gap: swallow-and-log

The dominant error pattern is `catch (e) { console.error(...) }` with no metric — analytics ([route.ts#L202-L205](../../app/api/chat/route.ts#L202-L205)), persistence retries ([persist-stream-results.ts#L94-L97](../../lib/streaming/helpers/persist-stream-results.ts#L94-L97)), rate-limit failures ([guest-limit.ts#L75-L78](../../lib/rate-limit/guest-limit.ts#L75-L78)), title generation ([title-generator.ts#L61-L74](../../lib/agents/title-generator.ts#L61-L74)), citation drops ([citation.ts#L88-L100](../../lib/utils/citation.ts#L88-L100)). Correct for resilience, blind for operations: at scale you'd never notice these degrading. The upgrade is a counter/metric at each swallow — "log-and-alert" not "log-and-forget."

## "How would I know it broke?" per flow

| Flow | Symptom | Current visibility | Better |
| --- | --- | --- | --- |
| Chat persistence (Flow 1) | answers not saved | server error log only | metric on persist failures + retries |
| Rate limits (Flow 3) | limiter down = no limits | one `console.error` | alert on Redis-timeout rate; fail-open counter |
| Search (Flow 4) | vendor down / slow | thrown → tool error in UI | per-vendor latency + error-rate; timeout |
| Citations (Flow 4) | links silently dropped | none | metric on dropped-citation count |
| Context truncation (Pattern 10) | quality drop on long chats | dev-only console log | metric on truncation events + unknown-model warns |
| Regeneration (Flow 5) | lost content on error | none | alert if delete succeeds but regen fails |
| Model discovery | all providers 500 | per-vendor `console.warn` | metric on empty model list |

## Operational surfaces

- **Deploy/rollback:** Vercel (per-route `maxDuration`) or Docker images ([docker-build.yml](../../.github/workflows/docker-build.yml), [release.yml](../../.github/workflows/release.yml)). Rollback = redeploy previous image/build; **DB migrations are forward-only** ([drizzle/](../../drizzle/) has no down files), so a rollback that needs a schema revert requires a new inverse migration — plan migrations to be backward-compatible with the previous app version (expand/contract).
- **Health:** the DB startup check ([db/index.ts#L68-L100](../../lib/db/index.ts#L68-L100)) verifies the connection *and* that it's the restricted (RLS-binding) user — a security-relevant health signal, not just liveness. No dedicated `/health` route exists (a small contribution).
- **Config drift risk:** because features gate on env presence, "is X enabled in prod?" is answered by env, not code — keep a runbook of which vars are set per environment.

## The blast-radius habit

For any change, ask "if this fails, what's the blast radius, and would we see it?" The repo answers the first well (contained failures) and the second poorly (silent). That asymmetry *is* the observability story to tell in interviews: "resilient but unobservable" is a real, nameable maturity stage.

## Interview angle

- "How do you know your service is healthy?" → traces + timing + the swallow-and-log gap you'd close first. (behavioral / system design)
- "Something's silently wrong in prod — how do you find it with no alert?" → Scenario-3-style: instrument the decision point, because the failure is a swallowed branch.
- "How do you do zero-downtime migrations?" → forward-only + expand/contract + backward-compatible schema; cite the no-down-files reality. (Q27)

Drill: pick the citation-drop flow. Design the metric (name, labels, where emitted at [citation.ts#L88-L100](../../lib/utils/citation.ts#L88-L100)) and the alert threshold. Self-grade — Strong: you emit *before* the silent `return ''`, label by reason (out-of-range / no-map / invalid-url), and set an alert on a *rate*, not a count.
