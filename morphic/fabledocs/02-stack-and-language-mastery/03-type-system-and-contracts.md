# Type System and Contracts

TypeScript's job here is to make **contracts** (shapes both sides agree on) machine-checked. The skill ladder: junior annotates; mid designs types that make invalid states unrepresentable; senior knows where the type system *stops* and runtime validation must take over.

## Inference and single-source-of-truth types

- DB row types are *derived*, not written: `InferSelectModel<typeof chats>` ([schema.ts#L70](../../lib/db/schema.ts#L70)), `$inferInsert` ([L259](../../lib/db/schema.ts#L259)). Change the table, every consumer re-checks. This is the pattern to name when asked "how do you keep types and DB in sync?"
- Zod schemas produce their own types: `z.infer<typeof todoItemSchema>` ([todo.ts#L18](../../lib/tools/todo.ts#L18)). One definition validates at runtime *and* types at compile time — the boundary ideal.
- Gnarly-but-honest inference: `TxInstance` extracted from the transaction callback's parameter type ([with-rls.ts#L7](../../lib/db/with-rls.ts#L7)) — deriving a type the library doesn't export. Know the trick; don't overuse it.

## Discriminated unions and narrowing

The message-part system is one giant discriminated union on `type` (+ `state` for tools). Narrowing done right: the switch in [message-mapping.ts#L347-L666](../../lib/utils/message-mapping.ts#L347-L666) — each `case` narrows to a shape; tool parts further switch on `tool_state` so `output` only exists when `state === 'output-available'`. Invalid states (an output-error part with an output) are unrepresentable in the *UI* types.

Type guards where unions cross untyped boundaries: [message-mapping.ts#L68-L102](../../lib/utils/message-mapping.ts#L68-L102) (`isToolCallPart`, `isExtendedToolPart`) and [lib/types/dynamic-tools.ts](../../lib/types/dynamic-tools.ts) guards consumed at [chat.tsx#L504-L524](../../components/chat.tsx#L504-L524). A guard is a *runtime check that teaches the compiler* — the pattern for any `unknown` data.

## `any` vs `unknown` — where this repo pays the tax

The DB columns are typed `.$type<any>()` ([schema.ts#L173-L198](../../lib/db/schema.ts#L173-L198)) and parts flow as `part: any` in several UI spots ([render-message.tsx#L56](../../components/render-message.tsx#L56), [chat.tsx#L427](../../components/chat.tsx#L427)). Consequence: the compiler can't catch a misspelled `tool_serach_input` column key built by string template ([message-mapping.ts#L110-L121](../../lib/utils/message-mapping.ts#L110-L121) — `as keyof DBMessagePart` erases the check). This is the repo's largest type hole and a ready-made refactor kata.

Contrast with disciplined `unknown`: error handling narrows before use — `error instanceof Error ? error.message : String(error)` ([create-chat-stream-response.ts#L219-L221](../../lib/streaming/create-chat-stream-response.ts#L219-L221)); `catch` + shape-check in [load-models-config.ts#L19-L40](../../lib/config/load-models-config.ts#L19-L40) (an `asserts` function — runtime validation that *narrows*: `asserts json is ModelsConfig`).

Rule to carry: **`any` disables the compiler for everyone downstream; `unknown` forces the next reader to prove the shape.** Accept `any` only at generated/vendor edges, and wrap it fast.

## Generics with constraints

- `withRLS<T>(userId, callback: (tx) => Promise<T>): Promise<T>` ([with-rls.ts#L39-L43](../../lib/db/with-rls.ts#L39-L43)) — the wrapper is transparent to the payload type. This is the 80% use of generics in app code: *pass a type through a wrapper*.
- Constrained generic: `wrapSearchToolForQuickMode<T extends ReturnType<typeof createSearchTool>>` ([researcher.ts#L20-L22](../../lib/agents/researcher.ts#L20-L22)) — but note it ends in `as T` ([L62](../../lib/agents/researcher.ts#L62)): the constraint documents intent, the cast bypasses proof. Casting away is sometimes pragmatic; say so out loud in review.
- `satisfies` for literal checking without widening: `satisfies Model` in the fetchers ([fetch-models.ts#L222-L227](../../lib/models/fetch-models.ts#L222-L227)) — object must match `Model` but keeps its precise inferred type.

## Where types end and Zod begins

Compile-time types vanish at runtime. Every *external* input needs runtime validation:

| Boundary | Validation | Anchor |
| --- | --- | --- |
| LLM → tool args | Zod schemas (per-model variants) | [lib/schema/search.tsx](../../lib/schema/search.tsx), [todo.ts#L5-L27](../../lib/tools/todo.ts#L5-L27) |
| LLM → spec blocks | Zod catalog validation | [catalog.ts#L7-L70](../../lib/render/catalog.ts#L7-L70), throw at [parse-spec-block.ts#L99-L106](../../lib/render/parse-spec-block.ts#L99-L106) |
| Config file | hand-rolled `asserts` fn | [load-models-config.ts#L19-L40](../../lib/config/load-models-config.ts#L19-L40) |
| HTTP body → `/api/chat` | **hand-rolled presence checks only** | [route.ts#L30-L52](../../app/api/chat/route.ts#L30-L52) — no Zod; `message.parts` shape is trusted (*gap; ticket material*) |
| DB rows → UI | switch + thrown errors | [message-mapping.ts#L344-L666](../../lib/utils/message-mapping.ts#L344-L666) |

The asymmetry is the lesson: LLM inputs (untrusted, weird) get Zod; the HTTP body (equally untrusted!) doesn't. When you ask "where's the Zod for the route body?", you're doing senior review.

## Pitfall checklist

- `as` casts near generics — each is a proof you skipped.
- String-keyed dynamic property access defeats `keyof` checking ([message-mapping.ts#L110-L121](../../lib/utils/message-mapping.ts#L110-L121)).
- `.$type<any>()` on jsonb columns pushes validation to every reader.
- Deriving types from libraries' internals (`Parameters<…>` tricks) breaks on upgrades — isolate them in one file like with-rls.ts does.

## Drills

1. Design (on paper, fake code labeled) a `DBPartColumns` mapped type that makes `tool_${name}_input` keys type-safe:
   ```ts
   // Illustrative fake code: not from this repo
   type ToolName = 'search' | 'fetch' | 'question' | 'todoWrite' | 'todoRead'
   type ToolColumns = { [K in ToolName as `tool_${K}_input`]?: unknown }
   ```
   What breaks in [createToolPartMapping](../../lib/utils/message-mapping.ts#L105-L122) when you apply it? Self-grade — Strong: you catch that `getToolNameFromType` returns plain `string` and must be narrowed first.
2. Find every `as any`/`: any` in [chat.tsx](../../components/chat.tsx) and classify: vendor-edge (acceptable) vs laziness (fixable). ([L427](../../components/chat.tsx#L427), [L498](../../components/chat.tsx#L498), [L544](../../components/chat.tsx#L544)…)
3. Write the Zod schema for the `/api/chat` body from the fields at [route.ts#L31](../../app/api/chat/route.ts#L31). What do you do with `message.parts`? (Reuse/derive from the AI SDK's UIMessage schema if exported; otherwise minimal structural check.)

## Interview angle

- "`unknown` vs `any`?" → downstream-blast-radius framing + the jsonb column example. ([Q5](../08-interview-prep/01-js-ts-node-deep-dive.md))
- "How do you type data crossing a boundary?" → Zod once, infer types from it; cite todo tool. (Q6)
- "Discriminated unions — real use?" → part types + tool_state narrowing. (Q8)
- "When is a cast acceptable?" → wrapper `as T` example: documented, contained, tested-around. (Q9)
