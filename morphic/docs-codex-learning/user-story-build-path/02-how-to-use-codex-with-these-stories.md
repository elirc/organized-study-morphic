# How To Use Codex With These Stories

## Start With Story 1

Read the story, open the cited file, and write a three-step implementation plan before asking Codex anything. Then make the smallest change yourself.

## Ask For Hints Instead Of Full Solutions

Use:

```text
I am working on Story 1. I found the Enter key handler in components/chat-panel.tsx. Ask me what condition should trigger feedback before giving me any code.
```

## Ask Codex To Explain Existing Patterns First

```text
Explain how components/chat-panel.tsx handles submit behavior. Do not suggest changes yet. Focus on state owner, callbacks, and failure cases.
```

## Ask Codex To Review Your Attempt

```text
Review my diff for Story 2. Focus on behavior regressions, accessibility, and whether I changed unrelated code.
```

## Ask Codex To Create Tests After You Implement

```text
I implemented Story 6. Based on my diff, propose focused Vitest tests. Use existing route test style from app/api/feedback/__tests__/route.test.ts.
```

## Ask Codex To Explain A Diff

```text
Explain this diff as if I need to present it in PR review. Separate intent, touched boundaries, risk, and tests.
```

## Prevent Unrelated Changes

Tell Codex:

```text
Only touch the files required for Story X. Do not refactor nearby code. Do not change formatting outside edited lines except what the project formatter requires.
```

## Checklist Before Accepting AI-Generated Code

- [ ] I can explain every changed line.
- [ ] The code follows existing local patterns.
- [ ] No unrelated files changed.
- [ ] Public inputs are validated.
- [ ] Auth/ownership behavior is preserved.
- [ ] Cache invalidation is considered for persisted data.
- [ ] Tests or manual verification cover the risky path.

## Checklist Before Opening A PR

- [ ] `bun lint`
- [ ] `bun typecheck`
- [ ] `bun format:check`
- [ ] `bun run build`
- [ ] `bun run test`
- [ ] PR description names files, behavior, tests, and risks.

## Example Prompts

Understanding a file:

```text
Walk me through app/api/chat/route.ts chronologically. Ask me to summarize each phase before moving to the next.
```

Tracing a bug:

```text
The upload UI says a file uploaded, but the filename is missing later. Help me trace client response shape versus server response shape. Give hints first.
```

Planning a change:

```text
I want to add chat pinning. Help me identify schema, action, UI, cache, and test files. Do not write code yet.
```

Reviewing a diff:

```text
Review this diff like a senior engineer. Prioritize bugs, auth issues, data loss, missing tests, and unintended behavior changes.
```

Writing tests:

```text
Suggest Vitest tests for my new upload ownership check. Use mocks and keep the tests focused on route behavior.
```

Hints only:

```text
I am stuck on where bookmark state should be stored. Ask me trade-off questions before suggesting an approach.
```

