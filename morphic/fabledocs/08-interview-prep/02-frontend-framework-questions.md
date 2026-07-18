# Frontend / React / Next Questions (Q13–Q22)

---

## Q13: Server components vs client components — when and why?
Anchor: server pages [app/page.tsx](../../app/page.tsx), [app/search/[id]/page.tsx](../../app/search/[id]/page.tsx); client [chat.tsx](../../components/chat.tsx).
Junior: "Server renders on the server, client in the browser."
Mid: "Fetch high, interact low. Morphic's pages are server components that fetch userId + chat + model data and pass serializable props to the client `<Chat/>`, which owns all interactivity via `useChat`. Server components ship HTML, not JS — smaller bundles."
Senior: "The boundary is a *serialization* boundary — only serializable props cross, no functions/classes. And a server action is a public endpoint, so I check auth in each one, not at the page. The subtle Next.js trap here is component caching keeping client instances alive across navigation, which is why `handleNewChat` manually resets state ([chat.tsx#L61-L72](../../components/chat.tsx#L61-L72))."
Follow-ups: what can't cross the boundary; where does data fetching go.

## Q14: What is a server action and what are its risks?
Anchor: [lib/actions/chat.ts#L1](../../lib/actions/chat.ts#L1) (`'use server'`), auth-per-action, the no-auth exception with its warning [L167-L200](../../lib/actions/chat.ts#L167-L200).
Junior: "A function that runs on the server, called from the client."
Mid: "It's RPC disguised as an import — every exported `'use server'` function is publicly invocable. So Morphic re-authenticates inside each (`getChats` calls `getCurrentUserId`). The one that doesn't (`upsertMessage`) carries a loud comment about its caller contract."
Senior: "'Server action without an auth check' is a review red flag equal to an unauthenticated endpoint. The contract-by-comment on `upsertMessage` is a smell — it's safe only because callers are trusted, an invariant the compiler can't enforce. I'd rather the auth live in the action or the type."
Follow-ups: CSRF on server actions; input validation.

## Q15: How do you avoid unnecessary re-renders / derive state correctly?
Anchor: `useMemo` sections [chat.tsx#L224-L252](../../components/chat.tsx#L224-L252); throttle [chat.tsx#L215](../../components/chat.tsx#L215).
Junior: "Use `React.memo`."
Mid: "Derive during render with `useMemo`, don't mirror into state via effects. Morphic derives `sections` from `messages` with `useMemo` — an effect+state version adds a render and a lag window. For the token stream it uses `experimental_throttle: 100` to cap re-renders to ~10/s."
Senior: "Derived-state-in-effect is the most common junior perf bug — extra render, race between the effect and the source. `React.memo` helps only with referential stability, so it pairs with memoized props. Throttling a stream is a UX-vs-latency tradeoff: smoother, slightly less live."
Follow-ups: `useCallback` vs `useMemo`; when memo hurts.

## Q16: What's a stale closure? Show a real fix.
Anchor: `messagesRef` [chat.tsx#L258-L297](../../components/chat.tsx#L258-L297).
Junior: "A closure capturing an old value."
Mid: "An event listener registered with `[]` deps captures the *initial* state forever. Morphic's copy-message listener must see current messages, so it reads from a `messagesRef` that a separate effect keeps updated — the listener registers once but always sees fresh data."
Senior: "The alternatives — re-registering the listener on every `messages` change (churny) or putting `messages` in deps (defeats 'register once') — are worse. The ref is the idiomatic escape hatch for 'stable subscription, current value'. There's also a multi-instance guard (`defaultPrevented` + visibility) because Next caching can mount several Chats."
Follow-ups: why not just add deps; functional `setState` updater.

## Q17: How do you render a streaming LLM response without jank?
Anchor: `useChat` + throttle [chat.tsx#L89-L217](../../components/chat.tsx#L89-L217); part-fold [render-message.tsx#L86-L180](../../components/render-message.tsx#L86-L180).
Junior: "Append tokens as they arrive."
Mid: "`useChat` is a state machine over SSE; messages grow parts incrementally. Throttle re-renders (100ms), derive view structure with `useMemo`, and fold parts into interleaved text/tool segments. Keys by messageId+index are safe because parts are append-only during a stream."
Senior: "The interesting bit is progressive parsing of *structured* output mid-stream — spec blocks parse incrementally and fail-soft to 'pending' so users never see raw JSON ([spec-fence.ts#L25-L50](../../lib/render/spec-fence.ts#L25-L50)). Waiting for the fence to close would freeze the UI for seconds."
Follow-ups: virtualization for long chats; scroll anchoring.

## Q18: Where does caching bite you in Next.js?
Anchor: the four layers ([02-framework-mental-models.md](../02-stack-and-language-mastery/02-framework-mental-models.md)); regeneration staleness [prepare-messages.ts#L61-L63](../../lib/streaming/helpers/prepare-messages.ts#L61-L63).
Junior: "Stale data after an update."
Mid: "Four caches touch a chat page: `unstable_cache` (tagged, 60s TTL), `useChat` client state, the module model cache, and history `pushState`. Writes call `revalidateTag('chat-{id}')`."
Senior: "The load-bearing subtlety: `revalidateTag` doesn't refresh a value *already read in the same request* — which is exactly why regeneration uses in-memory message slices instead of re-reading. And the cache key includes the requesting userId ([lib/actions/chat.ts#L22-L31](../../lib/actions/chat.ts#L22-L31)) to prevent cross-user cache poisoning, since the loader applies visibility rules."
Follow-ups: tag vs path revalidation; cache stampede.

## Q19: How would you let an LLM render UI safely (no XSS)?
Anchor: render catalog [catalog.ts](../../lib/render/catalog.ts); parser [parse-spec-block.ts](../../lib/render/parse-spec-block.ts).
Junior: "Sanitize the HTML."
Mid: "Don't let it write HTML at all. Morphic has the model emit a validated JSON spec that maps to a *whitelist* of components (Heading/Stack/Button/Grid/Image) with Zod-typed props — XSS-safe by construction, no `dangerouslySetInnerHTML`."
Senior: "The catalog is a *capability boundary*: adding a component grants the model a new power, so you review catalog changes like permissions. The parser also handles streaming (incremental prefix-diff compile) and migrates legacy persisted specs so old chats keep rendering. The residual risk is `Image src` accepting any URL — low severity, worth noting."
Follow-ups: compare with sanitized markdown; SSR of untrusted structure.

## Q20: How do you handle errors from a streaming API on the client?
Anchor: error parsing [chat.tsx#L135-L214](../../components/chat.tsx#L135-L214).
Junior: "Show a toast."
Mid: "Morphic maps error types to different UX: 429→rate-limit modal, 401→auth modal, 403→forbidden, else toast. It inspects the error message for status codes and parses JSON bodies out of it."
Senior: "That's a *smell* — the AI SDK transport flattens errors to a string, so the client regex-extracts JSON from `error.message`. The structure was created on the server ([chat-limits.ts#L90-L106](../../lib/rate-limit/chat-limits.ts#L90-L106)) and destroyed in transit. The right fix is a structured error channel; I'd migrate with a fallback to the string-parse during rollout (ticket M2)."
Follow-ups: retry UX; partial-stream failures.

## Q21: How do you build accessible, keyboard-friendly UI in a component-heavy app?
Anchor: shadcn/radix primitives [components/ui/](../../components/ui/index.ts); keyboard shortcuts [lib/keyboard-shortcuts.ts](../../lib/keyboard-shortcuts.ts), [use-keyboard-shortcut.ts](../../hooks/use-keyboard-shortcut.ts).
Junior: "Add `aria-label`s."
Mid: "Lean on accessible primitives (radix handles focus traps, roles, keyboard nav in dialogs/menus), then add labels for icon-only buttons. Morphic ships a shortcut system with a discoverability dialog."
Senior: "Primitives get you focus management and ARIA roles for free — reinventing a dropdown is where a11y bugs breed. The custom layer to audit is the shortcut handler's multi-instance/visibility logic and the clipboard action (needs secure context, [chat.tsx#L287](../../components/chat.tsx#L287))."
Follow-ups: focus management on route change; live regions for streaming text.

## Q22: Describe the layered architecture of a frontend+backend app you know.
Anchor: [01-boundaries-and-layers.md](../03-architecture-and-patterns/01-boundaries-and-layers.md).
Junior: "Frontend calls backend calls database."
Mid: "Browser (render/optimistic UX) → API routes (parse/auth/status) → orchestration (sequence the work) → domain services (agents/tools/limits) → application actions (auth+cache) → persistence (SQL+RLS). Morphic's cleanest detail is splitting app actions (auth-aware) from db actions (RLS-aware) so authorization lives in one layer."
Senior: "I'd call out a couple of real leaks too — rate limiters returning `Response` objects (HTTP shape in a lib), and a client-computed `isNewChat` the server trusts. Naming both the discipline and its exceptions is honest and shows I actually read it, not just the README."
Follow-ups: where does business logic go; how do you keep boundaries from eroding.
