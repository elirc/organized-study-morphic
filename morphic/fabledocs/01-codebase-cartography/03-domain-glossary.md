# Domain Glossary

Product and code nouns, with where they live. The traps are the near-synonyms — interviewers (and code reviewers) notice when you use them precisely.

| Term | Meaning here | Code home |
| --- | --- | --- |
| **Chat** | A conversation thread owned by a user; has `title`, `visibility` | [lib/db/schema.ts#L26-L68](../../lib/db/schema.ts#L26-L68) |
| **Message** | One turn (user or assistant) in a chat | [schema.ts#L73-L118](../../lib/db/schema.ts#L73-L118) |
| **Part** | A typed fragment of a message: text, reasoning, file, source, tool call/result, data | [schema.ts#L123-L256](../../lib/db/schema.ts#L123-L256) |
| **UIMessage** | AI SDK's client-side message shape (`{id, role, parts, metadata}`) | [lib/types/ai.ts](../../lib/types/ai.ts); mapped to DB in [message-mapping.ts](../../lib/utils/message-mapping.ts) |
| **Search mode** | `quick` (≤20 steps, search+fetch, forced `optimized`) vs `adaptive` (≤50 steps, +todoWrite) | [researcher.ts#L92-L113](../../lib/agents/researcher.ts#L92-L113); cookie read at [route.ts#L89-L94](../../app/api/chat/route.ts#L89-L94) |
| **Search type** | `optimized` (snippets included) vs `general` (raw results, video/image capable) — a *tool input*, not the mode | [lib/schema/search.tsx#L8-L12](../../lib/schema/search.tsx#L8-L12) |
| **Search provider** | Vendor adapter: tavily, searxng, exa, brave, firecrawl | [providers/index.ts#L8-L14](../../lib/tools/search/providers/index.ts#L8-L14) |
| **Provider (model)** | LLM vendor: openai, anthropic, google, ollama, gateway, openai-compatible | [lib/utils/registry.ts#L9-L29](../../lib/utils/registry.ts#L9-L29) |
| **Researcher** | The tool-looping agent that answers queries | [lib/agents/researcher.ts#L67-L153](../../lib/agents/researcher.ts#L67-L153) |
| **Trigger** | Why the client hit `/api/chat`: `submit-message` or `regenerate-message` (AI SDK standard values) | [route.ts#L38-L52](../../app/api/chat/route.ts#L38-L52) |
| **Ephemeral chat** | Guest conversation: full history sent from client, nothing persisted | [create-ephemeral-chat-stream-response.ts](../../lib/streaming/create-ephemeral-chat-stream-response.ts) |
| **Guest** | Request with no userId (`ENABLE_GUEST_CHAT` gates it) | [route.ts#L68-L85](../../app/api/chat/route.ts#L68-L85) |
| **Anonymous mode** | `ENABLE_AUTH=false`: everyone shares one user ID — *not* the same as guest | [get-current-user.ts#L23-L40](../../lib/auth/get-current-user.ts#L23-L40) |
| **Cloud deployment** | `MORPHIC_CLOUD_DEPLOYMENT=true`: enables rate limits, pinned models, blocks anonymous mode | grep `MORPHIC_CLOUD_DEPLOYMENT`; e.g. [adaptive-limit.ts#L47](../../lib/rate-limit/adaptive-limit.ts#L47) |
| **Spec block** | A fenced ```spec JSON block the LLM emits; validated against a component catalog and rendered as UI | [lib/render/catalog.ts](../../lib/render/catalog.ts), [spec-fence.ts](../../lib/render/spec-fence.ts) |
| **Citation map** | `{1: SearchResultItem}` attached to each search result, keyed by toolCallId in the UI | [search.ts#L145-L151](../../lib/tools/search.ts#L145-L151), [citation.ts#L21-L45](../../lib/utils/citation.ts#L21-L45) |
| **Visibility** | `private` (owner only) vs `public` (shareable read-only) | [schema.ts#L35-L41](../../lib/db/schema.ts#L35-L41), share action at [lib/actions/chat.ts#L265-L282](../../lib/actions/chat.ts#L265-L282) |
| **RLS** | Postgres row-level security; policies compare `user_id` to a transaction-local setting | [schema.ts#L53-L67](../../lib/db/schema.ts#L53-L67), [with-rls.ts#L44-L54](../../lib/db/with-rls.ts#L44-L54) |
| **Adaptive limit** | Daily Redis cap (default 30) on adaptive-mode chats per user | [adaptive-limit.ts#L6](../../lib/rate-limit/adaptive-limit.ts#L6) |
| **Trace / traceId** | Langfuse observability trace grouping one research run | [create-chat-stream-response.ts#L75-L94](../../lib/streaming/create-chat-stream-response.ts#L75-L94) |

## Near-synonym traps

- **Search mode vs search type vs search depth.** Mode is a user setting (cookie) that configures the *agent*; type is a per-call *tool argument*; depth (`basic`/`advanced`) is a provider hint. Quick mode force-overrides type to `optimized` ([researcher.ts#L33-L36](../../lib/agents/researcher.ts#L33-L36)).
- **`lib/actions/` vs `lib/db/actions.ts`.** The former are `'use server'` application actions (auth-aware, cache-aware); the latter is raw persistence (RLS-aware, no auth). Calling the wrong layer skips authorization — see the warning comment at [lib/actions/chat.ts#L167-L178](../../lib/actions/chat.ts#L167-L178).
- **Guest vs anonymous.** Guest = unauthenticated user of a normal deployment (ephemeral, rate-limited by IP). Anonymous = auth disabled globally, everything persists under one shared ID.
- **Provider.** Overloaded: search vendor *and* LLM vendor. Say "search provider" / "model provider".
- **`loadChat` (two of them).** [lib/actions/chat.ts#L63-L69](../../lib/actions/chat.ts#L63-L69) (cached, chat+messages) wraps [lib/db/actions.ts#L155-L193](../../lib/db/actions.ts#L155-L193) (`loadChatWithMessages`). There is also a bare `loadChat` in db/actions ([L131-L150](../../lib/db/actions.ts#L131-L150)) returning messages only.

Drill: without looking, explain to a rubber duck the difference between quick/adaptive, optimized/general, and basic/advanced. Then verify each against its anchor. Self-grade: Strong = you also said *where* each is decided (cookie / model's tool call / env-provider hint).
