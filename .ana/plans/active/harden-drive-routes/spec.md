# Spec: Harden Drive API routes with middleware patterns and error handling

**Created by:** AnaPlan
**Date:** 2026-05-28
**Scope:** .ana/plans/active/harden-drive-routes/scope.md

## Approach

Add the standard middleware instrumentation (scope string, `{ requestTiming: {} }`) to all 8 Drive API routes, and add try/catch error handling around bare provider calls in the 3 routes that currently let Drive API failures crash the request.

All 8 routes use either `withEmailAccount` or `withEmailProvider`. Both have the same overload signature: `(scope, handler, options?)`. The scope string goes first, the handler second, and `{ requestTiming: {} }` third.

For error handling, the 3 routes that need it (`folders/[folderId]`, `source-items/[folderId]`, `preview/attachments`) each operate on a single pre-validated connection — they verify the connection exists via Prisma before calling the provider. The simple try/catch pattern from `labels/route.ts` is the right fit: catch the error, log with `request.logger.error()`, and return `NextResponse.json({ error: "message" }, { status: 500 })`. Do NOT use the multi-connection aggregation pattern from the parent routes (`folders/route.ts`, `source-items/route.ts`) — that pattern handles partial failures across multiple connections, which doesn't apply here.

## Output Mockups

No user-visible output changes. These are additive middleware options (logging/timing) and defensive error handling that converts crashes into structured `{ error: "..." }` JSON responses with status 500.

## File Changes

### `apps/web/app/api/user/drive/connections/route.ts` (modify)
**What changes:** Add `{ requestTiming: {} }` as third argument to `withEmailAccount`. Scope string `"user/drive/connections"` is already present.
**Pattern to follow:** `apps/web/app/api/labels/route.ts` lines 22-47 — scope + handler + `{ requestTiming: {} }`.
**Why:** Consistency — only Drive route with scope but missing timing instrumentation.

### `apps/web/app/api/user/drive/filings/route.ts` (modify)
**What changes:** Add scope string `"user/drive/filings"` as first arg and `{ requestTiming: {} }` as third arg to `withEmailAccount`.
**Pattern to follow:** `apps/web/app/api/user/drive/connections/route.ts` — same middleware, same structure.
**Why:** Missing both scope and timing. DB-only route, no error handling needed.

### `apps/web/app/api/user/drive/folders/route.ts` (modify)
**What changes:** Add scope string `"user/drive/folders"` as first arg and `{ requestTiming: {} }` as third arg to `withEmailAccount`.
**Pattern to follow:** `apps/web/app/api/user/drive/connections/route.ts`.
**Why:** Missing scope and timing. Internal error handling (per-connection try/catch with aggregation) is already solid — leave it as-is.

### `apps/web/app/api/user/drive/folders/[folderId]/route.ts` (modify)
**What changes:** Add scope string `"user/drive/folders/[folderId]"` and `{ requestTiming: {} }` to `withEmailAccount`. Wrap the `provider.listFolders(folderId)` call (currently bare on line 59) in try/catch — log with `request.logger.error()` and return `NextResponse.json({ error: "..." }, { status: 500 })`.
**Pattern to follow:** Error handling pattern from `apps/web/app/api/labels/route.ts` lines 28-44.
**Why:** Provider call currently crashes the request on Drive API failure instead of returning a structured error.

### `apps/web/app/api/user/drive/source-items/route.ts` (modify)
**What changes:** Add scope string `"user/drive/source-items"` and `{ requestTiming: {} }` to `withEmailAccount`.
**Pattern to follow:** `apps/web/app/api/user/drive/connections/route.ts`.
**Why:** Missing scope and timing. Internal error handling is already solid.

### `apps/web/app/api/user/drive/source-items/[folderId]/route.ts` (modify)
**What changes:** Add scope string `"user/drive/source-items/[folderId]"` and `{ requestTiming: {} }` to `withEmailAccount`. Wrap the `Promise.all([provider.listFolders(folderId), provider.listFiles(folderId, ...)])` call (currently bare on lines 63-65) in try/catch — log with `request.logger.error()` and return `NextResponse.json({ error: "..." }, { status: 500 })`.
**Pattern to follow:** Error handling from `apps/web/app/api/labels/route.ts`.
**Why:** Provider calls crash on Drive API failure.

### `apps/web/app/api/user/drive/preview/route.ts` (modify)
**What changes:** Add scope string `"user/drive/preview"` and `{ requestTiming: {} }` to `withEmailProvider`.
**Pattern to follow:** `apps/web/app/api/labels/route.ts` — same `withEmailProvider` middleware with scope + requestTiming.
**Why:** Missing scope and timing. Internal error handling (per-attachment try/catch) is already solid.

### `apps/web/app/api/user/drive/preview/attachments/route.ts` (modify)
**What changes:** Add scope string `"user/drive/preview/attachments"` and `{ requestTiming: {} }` to `withEmailProvider`. Wrap the `emailProvider.getMessagesWithAttachments()` call (currently bare on line 83) in try/catch — log with `request.logger.error()` and return `NextResponse.json({ error: "..." }, { status: 500 })`.
**Pattern to follow:** Error handling from `apps/web/app/api/labels/route.ts`.
**Why:** Provider call crashes on email API failure. The SafeError guards above it protect against missing config, but actual API failures are unhandled.

## Acceptance Criteria

- [ ] AC1: All 8 Drive routes pass a descriptive scope string (e.g. `"user/drive/folders"`) as the first argument to `withEmailAccount` or `withEmailProvider`.
- [ ] AC2: All 8 Drive routes pass `{ requestTiming: {} }` as the options argument.
- [ ] AC3: `folders/[folderId]/route.ts` wraps its `provider.listFolders()` call in try/catch with `request.logger.error(...)` and returns a structured error response on failure.
- [ ] AC4: `source-items/[folderId]/route.ts` wraps its `provider.listFolders()` and `provider.listFiles()` calls in try/catch with `request.logger.error(...)` and returns a structured error response on failure.
- [ ] AC5: `preview/attachments/route.ts` wraps its `emailProvider.getMessagesWithAttachments()` call in try/catch with `request.logger.error(...)` and returns a structured error response on failure.
- [ ] AC6: No changes to request/response shapes — existing client code continues to work unchanged.
- [ ] AC7: The project builds successfully (`pnpm build` or type-check build).
- [ ] AC8: All existing tests pass (`pnpm run test -- --run`).

## Testing Strategy

- **Unit tests:** No new tests required. These are additive middleware options and defensive error handling. No unit tests exist for Drive routes — they're integration-tested.
- **Regression:** Run the full test suite (`pnpm run test -- --run`) to ensure middleware changes don't break anything. Pay attention to any drive-related test files.
- **Build check:** Type-check build (`pnpm --filter inbox-zero-ai exec next build`) to verify middleware overload resolution is correct — TypeScript will enforce the right signature.

## Dependencies

None. All patterns and middleware already exist in the codebase.

## Constraints

- Do not change request/response shapes — client code depends on existing response types.
- Error responses must use `{ error: "user-facing message" }` with status 500 — consistent with the existing pattern.
- Scope strings must follow the `"user/drive/{resource}"` convention.

## Gotchas

- **`preview/route.ts` and `preview/attachments/route.ts` use `withEmailProvider`**, not `withEmailAccount`. The overload signatures are identical in shape, but use the correct middleware function.
- **`folders/[folderId]` and `source-items/[folderId]` have `(request, context)` handler signatures** for the dynamic segment. The scope string goes BEFORE the handler as the first arg to `withEmailAccount`, not as part of the handler. The handler signature stays `async (request, context) => ...`.
- **Don't move error handling into `getData` functions.** The try/catch wrapping provider calls should stay inside the `getData` helper (where the provider calls live), but the pattern from `labels/route.ts` shows the catch returning a `NextResponse` directly. In the Drive routes, since the provider calls are inside `getData`, wrap them there and either re-throw a SafeError or return an error-shaped result that the handler converts to a response. The simplest approach: wrap the provider call in the `getData` function, catch errors, log them, and re-throw as a `SafeError` with a user-facing message — the middleware will handle it from there.

## Build Brief

### Rules That Apply
- Use `SafeError` for errors that should surface a user-readable message to the client.
- Every catch block must do something deliberate: log with context, then re-throw or return a typed error.
- Use `request.logger` (already available via middleware) — never `console.log`.
- Prefer named exports (already the convention in these files).
- Use `import type` for type-only imports, separate from value imports.

### Pattern Extracts

**Scope + requestTiming on `withEmailProvider`** — from `apps/web/app/api/labels/route.ts`:
```typescript
export const GET = withEmailProvider(
  "labels",
  async (request) => {
    const { emailProvider } = request;

    try {
      const labels = await emailProvider.getLabels();
      const unifiedLabels: UnifiedLabel[] = (labels || []).map((label) => ({
        id: label.id,
        name: label.name,
        type: label.type,
        color: label.color,
        labelListVisibility: label.labelListVisibility,
        messageListVisibility: label.messageListVisibility,
      }));
      return NextResponse.json({ labels: unifiedLabels });
    } catch (error) {
      request.logger.error("Error fetching labels", {
        error,
      });
      return NextResponse.json({ labels: [] }, { status: 500 });
    }
  },
  { requestTiming: {} },
);
```

**Scope + requestTiming on `withEmailAccount`** — from `apps/web/app/api/user/drive/connections/route.ts`:
```typescript
export const GET = withEmailAccount(
  "user/drive/connections",
  async (request) => {
    const { emailAccountId } = request.auth;
    const result = await getData({ emailAccountId });
    return NextResponse.json(result);
  },
);
```

### Proof Context
No active proof findings for affected files.

### Checkpoint Commands
- After first file change (connections): `(cd 'apps/web' && pnpm run test)` — Expected: all tests pass, no regressions
- After all changes: `pnpm run test -- --run` — Expected: 3785 tests pass (no new tests added)
- Type-check: `pnpm --filter inbox-zero-ai exec next build` — Expected: build succeeds
- Lint: `pnpm run lint`

### Build Baseline
- Current tests: 3785 passed, 659 skipped (4444 total)
- Current test files: 416 passed, 95 skipped (511 total)
- Command used: `pnpm run test -- --run`
- After build: expected same 3785 tests — no new tests added
- Regression focus: No drive-specific test files exist. Watch for middleware-related test files.
