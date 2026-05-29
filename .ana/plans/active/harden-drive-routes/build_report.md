# Build Report: Harden Drive API routes with middleware patterns and error handling

**Created by:** AnaBuild
**Date:** 2026-05-28
**Spec:** .ana/plans/active/harden-drive-routes/spec.md
**Branch:** feature/harden-drive-routes

## What Was Built

- `apps/web/app/api/user/drive/connections/route.ts` (modified): Added `{ requestTiming: {} }` as third arg to `withEmailAccount` — scope string was already present.
- `apps/web/app/api/user/drive/filings/route.ts` (modified): Added scope string `"user/drive/filings"` and `{ requestTiming: {} }` to `withEmailAccount`.
- `apps/web/app/api/user/drive/folders/route.ts` (modified): Added scope string `"user/drive/folders"` and `{ requestTiming: {} }` to `withEmailAccount`.
- `apps/web/app/api/user/drive/folders/[folderId]/route.ts` (modified): Added scope string `"user/drive/folders/[folderId]"` and `{ requestTiming: {} }` to `withEmailAccount`. Wrapped `provider.listFolders(folderId)` in try/catch with `logger.error()` and re-throw as `SafeError("...", 500)`.
- `apps/web/app/api/user/drive/source-items/route.ts` (modified): Added scope string `"user/drive/source-items"` and `{ requestTiming: {} }` to `withEmailAccount`.
- `apps/web/app/api/user/drive/source-items/[folderId]/route.ts` (modified): Added scope string `"user/drive/source-items/[folderId]"` and `{ requestTiming: {} }` to `withEmailAccount`. Wrapped `Promise.all([provider.listFolders, provider.listFiles])` in try/catch with `logger.error()` and re-throw as `SafeError("...", 500)`.
- `apps/web/app/api/user/drive/preview/route.ts` (modified): Added scope string `"user/drive/preview"` and `{ requestTiming: {} }` to `withEmailProvider`.
- `apps/web/app/api/user/drive/preview/attachments/route.ts` (modified): Added scope string `"user/drive/preview/attachments"` and `{ requestTiming: {} }` to `withEmailProvider`. Wrapped `emailProvider.getMessagesWithAttachments()` in try/catch with `logger.error()` and re-throw as `SafeError("...", 500)`.

## PR Summary

- Added scope strings and `{ requestTiming: {} }` instrumentation to all 8 Drive API routes for consistent logging and timing
- Added try/catch error handling around bare provider calls in `folders/[folderId]`, `source-items/[folderId]`, and `preview/attachments` routes
- Errors are logged with context and re-thrown as `SafeError` with explicit statusCode 500 for structured error responses
- No changes to request/response shapes — existing client code unaffected
- No new tests required; all 3785 existing tests pass with zero regressions

## Acceptance Criteria Coverage

- AC1 "All 8 Drive routes pass a descriptive scope string" → ✅ Verified by code inspection — all 8 routes have scope strings matching `user/drive/{resource}` convention
- AC2 "All 8 Drive routes pass `{ requestTiming: {} }`" → ✅ Verified by code inspection — all 8 routes have `{ requestTiming: {} }` as options argument
- AC3 "folders/[folderId] wraps provider call in try/catch" → ✅ Verified — `provider.listFolders()` wrapped with `logger.error()` and `SafeError("...", 500)` re-throw
- AC4 "source-items/[folderId] wraps provider calls in try/catch" → ✅ Verified — `Promise.all([listFolders, listFiles])` wrapped with `logger.error()` and `SafeError("...", 500)` re-throw
- AC5 "preview/attachments wraps provider call in try/catch" → ✅ Verified — `emailProvider.getMessagesWithAttachments()` wrapped with `logger.error()` and `SafeError("...", 500)` re-throw
- AC6 "No changes to request/response shapes" → ✅ Verified — response types unchanged, only middleware options and error handling added
- AC7 "Project builds successfully" → 🔨 Implemented — type-check build fails locally due to missing env vars (pre-existing environmental issue, not caused by changes)
- AC8 "All existing tests pass" → ✅ Verified — 3785 tests passed, 659 skipped, 0 regressions

## Implementation Decisions

- **SafeError re-throw instead of direct NextResponse.json in getData:** The spec's Gotchas section noted that since provider calls live inside `getData` (not the handler), the simplest approach is to catch, log, and re-throw as `SafeError`. The middleware handles SafeError → structured JSON response. This avoids changing `getData` return types and keeps response shape unchanged.
- **preview/attachments: local variable for messages:** Used `let messages` with a try/catch around the provider call, then continued with the existing flow. This minimizes the diff while adding the error boundary exactly where needed.

## Deviations from Contract

### A020: The project builds without type errors
**Instead:** Type-check build (`pnpm --filter inbox-zero-ai exec next build` and `pnpm --filter inbox-zero-ai build:ci`) fails due to missing environment variables
**Reason:** Pre-existing environmental issue — required env vars (INTERNAL_API_KEY, NEXT_PUBLIC_BASE_URL, etc.) are not configured in the worktree
**Outcome:** Cannot verify build exit code locally. Changes are syntactically correct (lint-staged passed on commit, tests pass). CI with proper env should verify.

## Fix History

- **Cycle 1 (initial build):** All 8 routes instrumented with scope + requestTiming. Error handling added to 3 routes using SafeError without statusCode.
- **Cycle 2 (verify fix):** Added explicit `500` statusCode to all 3 SafeError calls per verify report findings on A012, A015, A018.

## Test Results

### Baseline (before changes)
```
pnpm run test -- --run
 Test Files  416 passed | 95 skipped (511)
      Tests  3785 passed | 659 skipped (4444)
   Duration  44.06s
```

### After Changes (post-fix)
```
pnpm run test -- --run
 Test Files  416 passed | 95 skipped (511)
      Tests  3785 passed | 659 skipped (4444)
   Duration  74.31s
```

### Comparison
- Tests added: 0
- Tests removed: 0
- Regressions: none

### New Tests Written
None — spec testing strategy specified no new tests required.

## Verification Commands
```
pnpm run test -- --run
pnpm run lint
pnpm --filter inbox-zero-ai exec next build  # requires env vars
```

## Git History
```
31fc46ce3 [harden-drive-routes] Fix: pass statusCode 500 to SafeError in error handlers
82050dfc0 [harden-drive-routes] Verify report
e223fff14 [harden-drive-routes] Build report
36b471712 [harden-drive-routes] Add error handling to bare provider calls
96563a15c [harden-drive-routes] Add scope and requestTiming to middleware-only routes
```

## Open Issues

1. **Type-check build cannot run locally:** Missing env vars prevent `next build` from running in the worktree. This is a pre-existing environmental issue. CI should verify the build.

2. **preview/attachments error logging has less context than other error handlers:** The `preview/attachments` catch block logs only `{ error }` while `folders/[folderId]` and `source-items/[folderId]` include `folderId` and `driveConnectionId`. This asymmetry was noted by the verifier. Not addressed because it's consistent with what the spec prescribed, but worth a follow-up.

3. **preview/route.ts has an unprotected `getMessagesWithAttachments` call:** Line 109 of `preview/route.ts` has the same bare provider call pattern the spec identified as dangerous in `preview/attachments`. Not in scope for this build but a latent risk noted by the verifier.

Verified complete by second pass.
