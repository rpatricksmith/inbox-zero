# Verify Report: Harden Drive API routes with middleware patterns and error handling

**Result:** FAIL
**Created by:** AnaVerify
**Date:** 2026-05-28
**Spec:** .ana/plans/active/harden-drive-routes/spec.md
**Branch:** feature/harden-drive-routes

## Pre-Check Results
```
=== CONTRACT COMPLIANCE ===
  Contract: .ana/plans/active/harden-drive-routes/contract.yaml
  Seal: INTACT (hash sha256:4667ed67f918e2dabbb2db5b1ba62e31684904ef56a0b143581ada69adb952d8)
```

Tests: 3785 passed, 659 skipped. Build: not run (env vars missing — pre-existing). Lint: clean (1 pre-existing warning unrelated to this build).

## Contract Compliance
| ID | Says | Status | Evidence |
|----|------|--------|----------|
| A001 | The connections route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/connections/route.ts:8` — scope `"user/drive/connections"` present |
| A002 | The filings route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/filings/route.ts:21` — scope `"user/drive/filings"` present |
| A003 | The folders route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/folders/route.ts:16` — scope `"user/drive/folders"` present |
| A004 | The subfolders route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/folders/[folderId]/route.ts:15` — scope `"user/drive/folders/[folderId]"` present |
| A005 | The source items route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/source-items/route.ts:16` — scope `"user/drive/source-items"` present |
| A006 | The source items children route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/source-items/[folderId]/route.ts:20` — scope `"user/drive/source-items/[folderId]"` present |
| A007 | The filing preview route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/preview/route.ts:31` — scope `"user/drive/preview"` present |
| A008 | The attachments preview route identifies itself for logging and timing | ✅ SATISFIED | `apps/web/app/api/user/drive/preview/attachments/route.ts:31` — scope `"user/drive/preview/attachments"` present |
| A009 | All 8 routes have request timing enabled | ✅ SATISFIED | Grep confirms `requestTiming` in all 8 drive route files — 8 occurrences across 8 files |
| A010 | Subfolder listing failures return a structured error instead of crashing | ✅ SATISFIED | `apps/web/app/api/user/drive/folders/[folderId]/route.ts:76-83` — try/catch wraps `provider.listFolders()`, throws `SafeError` which middleware converts to `{ error: "..." }` JSON |
| A011 | Subfolder listing failures are logged with context for debugging | ✅ SATISFIED | `apps/web/app/api/user/drive/folders/[folderId]/route.ts:77-81` — `logger.error("Error listing subfolders", { folderId, driveConnectionId, error })` |
| A012 | Subfolder listing failures return HTTP 500 | ❌ UNSATISFIED | `SafeError("Failed to list subfolders from drive")` thrown without statusCode. Middleware `getSafeErrorStatusCode()` defaults to **400** when statusCode is undefined (`middleware.ts:755-761`). Fix: `new SafeError("Failed to list subfolders from drive", 500)` |
| A013 | Source item listing failures return a structured error instead of crashing | ✅ SATISFIED | `apps/web/app/api/user/drive/source-items/[folderId]/route.ts:82-89` — try/catch wraps `Promise.all()`, throws `SafeError` |
| A014 | Source item listing failures are logged with context for debugging | ✅ SATISFIED | `apps/web/app/api/user/drive/source-items/[folderId]/route.ts:83-87` — `logger.error("Error listing source items", { folderId, driveConnectionId, error })` |
| A015 | Source item listing failures return HTTP 500 | ❌ UNSATISFIED | Same issue as A012 — `SafeError("Failed to list source items from drive")` has no statusCode, defaults to 400. Fix: `new SafeError("Failed to list source items from drive", 500)` |
| A016 | Attachment preview failures return a structured error instead of crashing | ✅ SATISFIED | `apps/web/app/api/user/drive/preview/attachments/route.ts:91-101` — try/catch wraps `emailProvider.getMessagesWithAttachments()`, throws `SafeError` |
| A017 | Attachment preview failures are logged with context for debugging | ✅ SATISFIED | `apps/web/app/api/user/drive/preview/attachments/route.ts:97-99` — `logger.error("Error fetching messages with attachments", { error })` |
| A018 | Attachment preview failures return HTTP 500 | ❌ UNSATISFIED | Same issue — `SafeError("Failed to fetch messages with attachments")` has no statusCode, defaults to 400. Fix: `new SafeError("Failed to fetch messages with attachments", 500)` |
| A019 | Existing API response types are unchanged so client code keeps working | ✅ SATISFIED | `git diff main...HEAD` shows no changes to exported type signatures. Response shapes unchanged — only middleware args and internal error handling modified |
| A020 | The project builds without type errors | -- UNVERIFIABLE | Type-check build (`pnpm --filter inbox-zero-ai exec next build`) requires env vars (INTERNAL_API_KEY, NEXT_PUBLIC_BASE_URL, etc.) not available in worktree. Pre-existing environmental issue. Lint passes, tests pass, TypeScript overload resolution is correct (middleware accepted the scope+handler+options signature) |
| A021 | All existing tests continue to pass | ✅ SATISFIED | `pnpm run test -- --run`: 3785 passed, 659 skipped, 0 failures — matches baseline exactly |

## Independent Findings

**Prediction resolution:**

1. **Confirmed: SafeError without statusCode returns 400, not 500.** The builder used `throw new SafeError("message")` in all 3 error-handling routes. `SafeError` accepts an optional second arg `statusCode`. Without it, `getSafeErrorStatusCode(undefined)` returns 400. The fix is trivial: pass `500` as the second argument.

2. **Confirmed: `preview/route.ts` has an unprotected `getMessagesWithAttachments` call.** Line 109 calls `emailProvider.getMessagesWithAttachments()` without try/catch. The spec didn't require error handling here (it's covered by the per-attachment try/catch lower down), but it's the same pattern the spec identified as dangerous in `preview/attachments`. Worth noting for the next cycle.

3. **Not found: Response shape changes.** I checked all exported types in the diff — no changes. The `isKnownError: true` field added by SafeError middleware is pre-existing middleware behavior, not a new shape change.

**Surprised findings:**

4. The `preview/attachments` error logging at line 97-99 logs only `{ error }` without contextual fields like `emailAccountId`. The other two error handlers (`folders/[folderId]` and `source-items/[folderId]`) include `folderId` and `driveConnectionId` in their error logs. The asymmetry makes `preview/attachments` errors harder to debug in production.

5. The spec's Gotchas section was self-contradictory: it recommended "re-throw as SafeError" while the File Changes section said "return `NextResponse.json({ error: '...' }, { status: 500 })`". The builder followed Gotchas (correct prioritization for architectural guidance), but the Gotchas didn't mention that SafeError needs a statusCode to produce 500. This is an upstream planning issue.

## AC Walkthrough
- AC1: All 8 Drive routes pass a descriptive scope string → ✅ PASS — verified by reading all 8 files; each has `"user/drive/{resource}"` as first arg
- AC2: All 8 Drive routes pass `{ requestTiming: {} }` → ✅ PASS — grep confirms 8 occurrences across 8 files
- AC3: `folders/[folderId]` wraps provider call in try/catch → ✅ PASS — `provider.listFolders()` wrapped at line 64-83 with `logger.error()` and `SafeError` re-throw
- AC4: `source-items/[folderId]` wraps provider calls in try/catch → ✅ PASS — `Promise.all([listFolders, listFiles])` wrapped at line 68-89 with `logger.error()` and `SafeError` re-throw
- AC5: `preview/attachments` wraps provider call in try/catch → ✅ PASS — `emailProvider.getMessagesWithAttachments()` wrapped at line 91-101 with `logger.error()` and `SafeError` re-throw
- AC6: No changes to request/response shapes → ✅ PASS — no exported type changes in diff; middleware adds `isKnownError` but that's pre-existing behavior
- AC7: Project builds successfully → -- UNVERIFIABLE — env vars missing; lint passes, tests pass, TS overload resolution correct
- AC8: All existing tests pass → ✅ PASS — 3785 passed, 659 skipped, 0 regressions, matching baseline exactly

## Blockers

3 contract assertions UNSATISFIED (A012, A015, A018): All three error-handling routes throw `SafeError` without a `statusCode`, causing the middleware to return HTTP 400 instead of the contract-specified 500. The fix is a one-line change per route — pass `500` as the second argument to `new SafeError()`.

## Findings

- **Code — SafeError missing statusCode returns 400 instead of 500:** `apps/web/app/api/user/drive/folders/[folderId]/route.ts:82`, `apps/web/app/api/user/drive/source-items/[folderId]/route.ts:89`, `apps/web/app/api/user/drive/preview/attachments/route.ts:100` — all three `new SafeError("message")` calls omit the statusCode parameter. `getSafeErrorStatusCode(undefined)` at `apps/web/utils/middleware.ts:755-761` defaults to 400. These provider failures are server-side errors, not client errors — 500 is the correct status.
- **Code — Sparse error logging in preview/attachments:** `apps/web/app/api/user/drive/preview/attachments/route.ts:97-99` — logs only `{ error }` without `emailAccountId` or other context. The other two error handlers include `folderId` and `driveConnectionId`. In production, this makes attachment fetch errors harder to correlate with specific users or accounts.
- **Code — preview/route.ts has bare getMessagesWithAttachments call:** `apps/web/app/api/user/drive/preview/route.ts:109` — same unprotected provider call pattern that the spec identified as dangerous in `preview/attachments`. Not in scope for this build, but a latent risk — a Drive API failure here crashes the request.
- **Upstream — Spec Gotchas conflicted with File Changes on error pattern:** The Gotchas section recommended "catch, log, re-throw as SafeError" while File Changes said "return `NextResponse.json({ error }, { status: 500 })`". The Gotchas didn't mention SafeError's optional statusCode parameter. The builder made a reasonable choice following Gotchas, but the omission of statusCode created the contract violations.

## Deployer Handoff

This is a safe, additive change — scope strings and requestTiming to 8 routes, plus error handling on 3 routes. No response shape changes. Tests pass with zero regressions. The only issue blocking merge is the SafeError statusCode: change `new SafeError("message")` to `new SafeError("message", 500)` in 3 files. After that fix, this is ready to ship. The lint warning on `StepInboxProcessed.tsx` is pre-existing and unrelated.

## Verdict
**Shippable:** NO
3 contract assertions (A012, A015, A018) are UNSATISFIED. The error responses return HTTP 400 instead of the specified 500. The fix is trivial — add `500` as the second argument to `SafeError()` in 3 files — but the contract is the contract.