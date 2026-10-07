# Junior Engineer Guide

## Setup And Run

The repo uses Bun as the expected package manager (`package.json:107-109`). The best reconstructed local setup is:

```bash
bun install
cp .env.local.example .env.local
# Fill OPENAI_API_KEY or another provider key, DATABASE_URL, and TAVILY_API_KEY or another search setup.
bun migrate
bun dev
```

Important scripts:

- `bun dev`: starts Next dev server (`package.json:7`).
- `bun run build`: production build (`package.json:8`).
- `bun lint`: ESLint (`package.json:10`).
- `bun typecheck`: TypeScript (`package.json:11`).
- `bun format:check`: Prettier check (`package.json:13`).
- `bun migrate`: Drizzle migrations (`package.json:14`, `drizzle.config.ts:11-18`).
- `bun run test`: Vitest (`package.json:15`, `vitest.config.mts:17-21`).

Required local environment is reconstructed from upstream `.env.local.example` (not imported into this snapshot; cross-check `docs/CONFIGURATION.md` and the `process.env.*` reads in `lib/`): at least one AI provider key, `DATABASE_URL`, and one search provider or Docker SearXNG. Auth is optional and defaults to anonymous mode in the example (`.env.local.example:73-85`). Docker Compose provides PostgreSQL, Redis, SearXNG, and app wiring (`docker-compose.yaml:4-75`).

No seed script was found. The app appears to create chat data through normal use.

## Folder Orientation

| Folder | Role |
| --- | --- |
| `app/` | Next.js App Router pages and API routes. |
| `components/` | Client and shared UI components, including chat, message rendering, sidebar, and shadcn primitives. |
| `hooks/` | Browser-facing custom hooks such as file dropzone and keyboard helpers. |
| `lib/` | Domain logic: agents, actions, auth, config, DB, schemas, streaming, tools, types, utilities. |
| `drizzle/` | Generated database migrations and metadata. |
| `public/` | Images, provider logos, model config assets. |
| `scripts/` | CLI/performance helper scripts. |

Three important frontend folders:

- `components/`: read `components/chat.tsx`, `components/chat-panel.tsx`, and `components/render-message.tsx` first.
- `components/ui/`: reusable primitives wrapped around Radix/shadcn patterns.
- `hooks/`: small reusable browser behaviors such as `hooks/use-file-dropzone.ts:15-114`.

Three important backend folders:

- `app/api/`: HTTP API routes, especially `app/api/chat/route.ts`.
- `lib/streaming/`: AI SDK response preparation and persistence.
- `lib/db/`: Drizzle schema, query actions, RLS helpers, and tests.

## Frontend Entry Walkthrough

```tsx
// app/page.tsx:6-18
export default async function Page() {
  const userId = await getCurrentUserId() // Server-only identity lookup.
  const isCloudDeployment = process.env.MORPHIC_CLOUD_DEPLOYMENT === 'true'
  const modelSelectorData = await getModelSelectorData() // Server computes model availability.

  return (
    <Chat
      isGuest={!userId} // Client receives a simple boolean, not the full auth object.
      isCloudDeployment={isCloudDeployment}
      modelSelectorData={modelSelectorData}
    />
  )
}
```

The page does not render input fields itself. Its job is to convert server facts into props for the chat component.

## Backend Entry Walkthrough

```ts
// app/api/chat/route.ts:29-52
const body = await req.json()
const { message, messages, chatId, trigger, messageId, isNewChat } = body

if (trigger === 'regenerate-message') {
  if (!messageId) return new Response('messageId is required for regeneration', { status: 400 })
} else if (trigger === 'submit-message') {
  if (!message) return new Response('message is required for submission', { status: 400 })
}
```

This is the public boundary for chat. A junior should notice that validation is manual and minimal here. A stronger version would parse `body` with a Zod schema before trusting the fields.

## TypeScript Orientation

Beginner-relevant patterns:

- Props define component contracts: `ChatPanelProps` in `components/chat-panel.tsx:30-56`.
- Domain unions restrict valid values: `SearchMode = 'quick' | 'adaptive'` in `lib/types/search.ts:1-2`.
- Drizzle infers selected row types: `export type Chat = InferSelectModel<typeof chats>` in `lib/db/schema.ts:70`.
- Zod defines AI tool input shape: `lib/schema/search.tsx:6-46`.

## Component Anatomy 1: Chat State Owner

```tsx
// components/chat.tsx:55-77
const [chatId, setChatId] = useState(() => providedId || generateId())
const [uploadedFiles, setUploadedFiles] = useState<UploadedFile[]>([])
const [input, setInput] = useState('')
```

`Chat` owns the state that affects the whole chat experience: the current chat ID, input text, uploaded files, scroll state, error modal, and AI SDK messages.

## Component Anatomy 2: Input Panel

```tsx
// components/chat-panel.tsx:254-289
<Textarea
  value={input} // Controlled input: parent owns the value.
  disabled={isLoading || isToolInvocationInProgress()} // UI blocks while model/tools run.
  onChange={handleInputChange}
  onKeyDown={e => {
    if (e.key === 'Enter' && !e.shiftKey && !isComposing && !enterDisabled) {
      if (input.trim().length === 0) {
        e.preventDefault()
        return
      }
      e.preventDefault()
      ;(e.target as HTMLTextAreaElement).form?.requestSubmit()
    }
  }}
/>
```

The component protects user experience details: composition input, empty submits, loading state, and keyboard submission.

## Component Anatomy 3: Search Results

```tsx
// components/search-results.tsx:18-31
export function SearchResults({ results, displayMode = 'grid' }: SearchResultsProps) {
  const [showAllResults, setShowAllResults] = useState(false)
  const displayedGridResults = showAllResults ? results : results.slice(0, 3)
  const additionalResultsCount = results.length > 3 ? results.length - 3 : 0
}
```

This is a small UI component with local state. It does not fetch data; it only controls presentation of data it receives.

## Backend Anatomy 1: Chat Route

```ts
// app/api/chat/route.ts:151-169
const response = isGuest
  ? await createEphemeralChatStreamResponse({ messages, model: selectedModel, abortSignal, searchMode, chatId })
  : await createChatStreamResponse({ message, model: selectedModel, chatId, userId, trigger, messageId, abortSignal, isNewChat, searchMode })
```

The route branches by persistence model. Guests can chat, but their stream is not saved to the database.

## Backend Anatomy 2: Upload Route

```ts
// app/api/upload/route.ts:48-60
if (file.size > MAX_FILE_SIZE) return NextResponse.json({ error: 'File too large (max 5MB)' }, { status: 400 })
if (!ALLOWED_TYPES.includes(file.type)) return NextResponse.json({ error: 'Unsupported file type' }, { status: 400 })
```

This route shows boundary validation: auth, storage configuration, content type, file existence, size, and MIME type.

## Backend Anatomy 3: Feedback Route

```ts
// app/api/feedback/route.ts:12-24
if (!traceId) return new Response('traceId is required', { status: 400 })
if (score === undefined || (score !== 1 && score !== -1)) {
  return new Response('score must be 1 (good) or -1 (bad)', { status: 400 })
}
```

The route validates a tiny contract manually and has tests for both valid and invalid scores (`app/api/feedback/__tests__/route.test.ts:53-201`).

## Domain Glossary

Technical terms:

- Server Component: a React component that can run on the server and await data, such as `app/page.tsx`.
- Client Component: a React component with `'use client'`, hooks, browser events, and local state, such as `components/chat.tsx`.
- API Route: a file under `app/api/**/route.ts` that handles HTTP requests.
- UIMessage: AI SDK message shape with role, parts, metadata, and tool output.
- RLS: PostgreSQL Row Level Security; this repo uses policies and `set_config`.
- Stream: an AI response delivered incrementally to the UI.

Domain terms:

- Chat: a conversation owned by a user or anonymous user.
- Message: one user or assistant turn.
- Part: a typed fragment inside a message: text, reasoning, file, source, tool result, or dynamic data.
- Search mode: `quick` or `adaptive`; controls agent prompt, tools, and max steps.
- Researcher: the AI agent configured with search/fetch/todo tools.
- Artifact: side-panel/rendered output experience around generated UI/spec content.

## Junior Socratic Checkpoint

1. Which file creates the first `Chat` component on the homepage?
2. Where does the chat API request body get assembled?
3. What is the difference between `message` and `messages` in the chat API request?
4. Which table stores message text?
5. Where is the search mode chosen in the UI?
6. Why can file upload be disabled even when the button renders?
7. Which command should run tests, and why not `bun test`?

## How To Self-Grade

A strong answer cites `app/page.tsx:6-18`, `components/chat.tsx:100-129`, `components/chat.tsx:421-452`, `app/api/chat/route.ts:151-169`, `lib/db/schema.ts:122-258`, `components/search-mode-selector.tsx:25-56`, `app/api/upload/route.ts:22-31`, and `package.json:15`. It explains behavior, not just filenames.

