# Framework Mental Models — React 19 + Next.js 16 App Router

## The server/client split (the App Router's one big idea)

Components are **server by default**: they run once per request, can await data, and ship HTML (not JS) to the browser. `'use client'` marks the interactive islands. Morphic's split is textbook:

- Server: [app/page.tsx](../../app/page.tsx) and [app/search/[id]/page.tsx](../../app/search/[id]/page.tsx) — fetch userId, chat, model data; pass plain props down.
- Client: [components/chat.tsx#L1](../../components/chat.tsx#L1) — owns `useChat`, local state, DOM events.

The rule of thumb the repo follows: *fetch high (server), interact low (client), pass serializable props across the boundary* (`savedMessages`, `isGuest`, `modelSelectorData` at [search/[id]/page.tsx#L50-L58](../../app/search/[id]/page.tsx#L50-L58)).

**Server actions** (`'use server'`, [lib/actions/chat.ts#L1](../../lib/actions/chat.ts#L1)) are RPC endpoints disguised as imports — every exported function is *publicly callable*. That's why each one re-authenticates ([getChats → getCurrentUserId](../../lib/actions/chat.ts#L39-L45)) instead of trusting the caller, and why the one that doesn't ([upsertMessage](../../lib/actions/chat.ts#L167-L200)) carries a shouting comment about its caller contract. Treat "exported server action without an auth check" as a review red flag.

## Render vs commit, and why effects are escape hatches

Render = compute JSX from props/state (pure, may run repeatedly). Commit = apply to DOM. Effects run *after* commit — they're for syncing with things React doesn't own.

Real usage worth copying:
- **Derived data goes in `useMemo`, not effects+state:** sections derived from messages ([chat.tsx#L224-L252](../../components/chat.tsx#L224-L252)). A junior would `useEffect(() => setSections(...), [messages])` — extra render, race-prone.
- **Refs to escape stale closures:** the copy-shortcut listener registers once but must see current messages — `messagesRef` updated per render ([chat.tsx#L258-L297](../../components/chat.tsx#L258-L297)). This is the *stale closure* interview question in production form: without the ref, the listener registered with `[]` deps would forever see the initial (empty) `messages`.
- **Effects that own cleanup:** scroll listeners + `requestAnimationFrame` with cancel on unmount ([chat.tsx#L309-L344](../../components/chat.tsx#L309-L344)).
- **Window events as a poor-man's bus:** `chat-history-updated` dispatch ([chat.tsx#L132-L134](../../components/chat.tsx#L132-L134)) decouples the sidebar from the chat without lifting state. Tradeoff: invisible contract, no types — fine at this scale, notice when it stops being fine.

Note the defensive comments about **Next.js component caching** — multiple `Chat` instances can be mounted, so the shortcut handler checks `defaultPrevented` + visibility ([chat.tsx#L254-L267](../../components/chat.tsx#L254-L267)) and `handleNewChat` manually resets state ([L61-L72](../../components/chat.tsx#L61-L72)). Framework caching keeps component instances alive longer than a junior expects; state you assumed died on navigation didn't.

## Data fetching & caching layers (know which cache bit you)

Four distinct caches touch a chat page:
1. `unstable_cache` around DB reads, tagged per chat ([lib/actions/chat.ts#L17-L34](../../lib/actions/chat.ts#L17-L34)), invalidated by `revalidateTag` on writes.
2. `useChat`'s in-memory message state (client).
3. The module-level model cache ([fetch-models.ts#L39-L44](../../lib/models/fetch-models.ts#L39-L44)).
4. Browser/history state — `window.history.pushState` swaps the URL without a navigation so the stream isn't interrupted ([chat.tsx#L446-L451](../../components/chat.tsx#L446-L451)).

The repo's own comment at [prepare-messages.ts#L61-L63](../../lib/streaming/helpers/prepare-messages.ts#L61-L63) is the key lesson: **revalidation doesn't help the request that's already holding stale data** — hence in-memory slices during regeneration.

## Streaming UI with `useChat`

`useChat` ([chat.tsx#L89-L217](../../components/chat.tsx#L89-L217)) is a state machine over an SSE stream: `status` ∈ submitted/streaming/ready/error; messages grow parts incrementally. Morphic customizes the transport (request body shaping, [L102-L129](../../components/chat.tsx#L102-L129)), throttles re-renders (`experimental_throttle: 100`, [L215](../../components/chat.tsx#L215) — 10 renders/s instead of per-token), and supplies the same `generateId` as the DB ([L216](../../components/chat.tsx#L216)) so client ids are DB-valid — a tiny decision that makes optimistic persistence possible.

Error UX is *parsed out of error strings* ([L135-L214](../../components/chat.tsx#L135-L214)) — regex-matching JSON out of `error.message` to distinguish 401/403/429. That's a smell forced by the transport flattening errors; a structured error channel is the fix (mid-level ticket in [06-contribution-practice](../06-contribution-practice/02-mid-level-feature-tickets.md)).

Rendering: [render-message.tsx#L86-L180](../../components/render-message.tsx#L86-L180) interleaves text with buffered tool/reasoning segments — a fold over parts, with keys derived from messageId+index. Keys matter here: index-keys are safe only because parts append-only during a stream.

## Middleware

[proxy.ts](../../proxy.ts) (Next 16 middleware, named `proxy`) runs on almost every request: refresh Supabase session cookies ([lib/supabase/middleware.ts#L39-L41](../../lib/supabase/middleware.ts#L39-L41)), redirect unauthenticated users off private pages ([L55-L60](../../lib/supabase/middleware.ts#L55-L60) — note the public-path list includes `/api`, so API routes do their *own* auth), stamp base-URL headers. Mental model: middleware = per-request film between CDN and app; keep it cheap.

## Pitfall checklist

- Secrets in client components (`NEXT_PUBLIC_` only in browser) — check imports when moving code across the boundary.
- Server action without auth = public unauthenticated endpoint.
- `useEffect` for derivable data → `useMemo`.
- Event listeners with `[]` deps reading state → refs ([chat.tsx#L258-L261](../../components/chat.tsx#L258-L261) is the template).
- Assuming component state dies on navigation (Next caching says otherwise).
- Forgetting that `revalidateTag` doesn't refresh already-fetched values in the same request.

## Drills

1. In [chat.tsx](../../components/chat.tsx), list every `useEffect` and label it: subscription / DOM measurement / event-bus / sync-to-ref. Which could be deleted with a different design? Self-grade — Strong: you defend `messagesRef` and question the double scroll-check effects ([L309-L344](../../components/chat.tsx#L309-L344)).
2. Explain why `handleNewChat` must reset `input` and `uploadedFiles` ([L61-L72](../../components/chat.tsx#L61-L72)) instead of relying on remount. (Component caching keeps the instance.)
3. Trace one token from SSE frame to pixel: transport → `useChat` state → `sections` memo → `RenderMessage` → `AnswerSection`.

## Interview angle

- "Server components vs client components — when and why?" → fetch high, interact low; cite the page/Chat split. ([Q13-Q15](../08-interview-prep/02-frontend-framework-questions.md))
- "What's a stale closure? Show a real fix." → `messagesRef` pattern with anchor. (Q16)
- "How do you render a streaming LLM response without janking?" → throttle + append-only parts + memoized derivation. (Q17)
- "Where can caching bite you in Next.js?" → four-layer list above; the regeneration staleness comment is a senior-grade detail. (Q18)
