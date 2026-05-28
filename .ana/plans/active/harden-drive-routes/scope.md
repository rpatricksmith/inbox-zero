# Scope: Harden Drive API routes with middleware patterns and error handling

**Created by:** Ana
**Date:** 2026-05-28

## Intent

The Drive filing feature has 7 API routes that were built without the middleware instrumentation (scope strings, request timing) and consistent error handling that well-structured routes elsewhere in the codebase use. One route (`connections`) was written to standard; the other six weren't. The dynamic-segment routes (`folders/[folderId]`, `source-items/[folderId]`) make the same Drive provider API calls as their parent routes but lack the try/catch error handling those parents have — meaning Drive API failures crash the request instead of being handled gracefully. This PR brings the entire Drive route surface up to the project's own standard.

## Complexity Assessment
- **Kind:** chore
- **Size:** small — mechanical changes following established patterns, no new logic
- **Surface:** web
- **Files affected:** 7 route files in `apps/web/app/api/user/drive/`
  - `connections/route.ts` — add requestTiming only (scope already present)
  - `filings/route.ts` — add scope, requestTiming
  - `folders/route.ts` — add scope, requestTiming
  - `folders/[folderId]/route.ts` — add scope, requestTiming, add error handling around Drive API calls
  - `source-items/route.ts` — add scope, requestTiming
  - `source-items/[folderId]/route.ts` — add scope, requestTiming, add error handling around Drive API calls
  - `preview/route.ts` — add scope, requestTiming
  - `preview/attachments/route.ts` — add scope, requestTiming
- **Blast radius:** None. Scope strings and requestTiming are additive middleware options — they change logging output, not behavior. Error handling additions wrap existing provider calls in try/catch, converting unhandled crashes into structured error responses. No changes to request/response shapes or business logic.
- **Estimated effort:** ~30 minutes
- **Multi-phase:** no

## Approach

Add the standard middleware options (scope string, requestTiming) to all Drive routes, and add try/catch with contextual logging to the dynamic-segment routes that currently let Drive API failures crash the request. Follow the patterns already established by `connections/route.ts` (scope string), `labels/route.ts` (requestTiming), and `folders/route.ts` (graceful Drive error handling with per-connection try/catch).

## Acceptance Criteria
- AC1: All 7 Drive routes pass a descriptive scope string (e.g. `"user/drive/folders"`) as the first argument to `withEmailAccount` or `withEmailProvider`.
- AC2: All 7 Drive routes pass `{ requestTiming: {} }` as the options argument.
- AC3: `folders/[folderId]/route.ts` wraps its `provider.listFolders()` call in try/catch with `request.logger.error(...)` and returns a structured error response on failure.
- AC4: `source-items/[folderId]/route.ts` wraps its `provider.listFolders()` and `provider.listFiles()` calls in try/catch with `request.logger.error(...)` and returns a structured error response on failure.
- AC5: `preview/attachments/route.ts` wraps its `emailProvider.getMessagesWithAttachments()` call in try/catch with `request.logger.error(...)` and returns a structured error response on failure.
- AC6: No changes to request/response shapes — existing client code continues to work unchanged.
- AC7: The project builds successfully (`pnpm build` or type-check build).

## Edge Cases & Risks
- **Middleware overload signature resolution:** `withEmailAccount` and `withEmailProvider` use overloaded signatures — the first arg can be a scope string or the handler. Adding a scope string changes which overload is invoked. This is well-tested across the codebase (dozens of routes use scope strings) and TypeScript will enforce correctness.
- **Error response shape in dynamic-segment routes:** The new try/catch blocks return `{ error: "..." }` with status 500, matching the pattern in `threads/[id]/route.ts`. Client-side SWR hooks already handle error responses.

## Rejected Approaches
- **Add requestTiming to all 79 routes missing it:** Scope creep. The Drive routes are a coherent unit; a codebase-wide sweep is a separate effort.
- **Refactor Drive routes to share error handling via a wrapper:** Over-engineering for 7 simple routes. The project avoids premature abstraction (per AGENTS.md: "Default to inlining and co-locating logic at the call site").
- **Delete the dead `labels/create` route in this PR:** Unrelated change that muddies the PR's purpose.

## Open Questions

None — all patterns are established elsewhere in the codebase.

## Exploration Findings

### Patterns Discovered
- `connections/route.ts` is the only Drive route with a scope string — the pattern was started but not completed across the feature
- `folders/route.ts` and `source-items/route.ts` iterate over Drive connections with per-connection try/catch and aggregate errors, only throwing if ALL connections fail. Their `[folderId]` siblings skip this entirely.
- `preview/route.ts` has thorough error handling around individual attachment processing but the route-level `emailProvider.getMessagesWithAttachments()` call is protected. `preview/attachments/route.ts` makes the same provider call without protection.

### Constraints Discovered
- [TYPE-VERIFIED] Middleware overloads (utils/middleware.ts:637-664) — scope string must be first arg, handler second, options third
- [OBSERVED] `{ requestTiming: {} }` uses defaults — no custom thresholds needed (consistent with labels/route.ts, user/rules/route.ts, threads/route.ts)

### Test Infrastructure
- No unit tests exist for the Drive route files — these are API routes tested via integration/e2e. The changes are additive middleware options and defensive error handling, so no new tests are required.

## For AnaPlan

### Structural Analog
`apps/web/app/api/user/drive/connections/route.ts` — the one Drive route already written to standard. Shows the scope string pattern for this feature area.

For error handling: `apps/web/app/api/user/drive/folders/route.ts` lines 66-90 — try/catch per Drive connection with `logger.warn()`, connection error aggregation, and `SafeError` throw when all connections fail.

For requestTiming: `apps/web/app/api/labels/route.ts` line 46 — `{ requestTiming: {} }` as third arg to `withEmailProvider`.

### Relevant Code Paths
- `apps/web/app/api/user/drive/connections/route.ts` — scope string already present, only needs requestTiming
- `apps/web/app/api/user/drive/filings/route.ts` — needs scope + requestTiming, no error handling needed (DB only)
- `apps/web/app/api/user/drive/folders/route.ts` — needs scope + requestTiming, internal error handling already good
- `apps/web/app/api/user/drive/folders/[folderId]/route.ts` — needs scope + requestTiming + error handling (lines 55-59: bare provider calls)
- `apps/web/app/api/user/drive/source-items/route.ts` — needs scope + requestTiming, internal error handling already good
- `apps/web/app/api/user/drive/source-items/[folderId]/route.ts` — needs scope + requestTiming + error handling (lines 59-65: bare provider calls)
- `apps/web/app/api/user/drive/preview/route.ts` — needs scope + requestTiming, internal error handling already good
- `apps/web/app/api/user/drive/preview/attachments/route.ts` — needs scope + requestTiming + error handling (line 83: bare emailProvider call)

### Patterns to Follow
- Scope string naming: `"user/drive/{resource}"` — matches `"user/drive/connections"` and broader convention like `"user/stats/senders"`, `"user/rules"`
- Error handling in provider routes: try/catch with `request.logger.error("Description", { error, emailAccountId, ...context })`, return `NextResponse.json({ error: "User-facing message" }, { status: 500 })`
- RequestTiming: `{ requestTiming: {} }` — use defaults, no custom thresholds

### Known Gotchas
- `preview/route.ts` and `preview/attachments/route.ts` use `withEmailProvider` (not `withEmailAccount`) — the overload signatures are slightly different but follow the same pattern. Check types.
- `folders/[folderId]` and `source-items/[folderId]` use `context.params` — the handler signature is `async (request, context)`, not just `async (request)`. The scope string goes before the handler, not after.

### Things to Investigate
- For the error handling in `folders/[folderId]` and `source-items/[folderId]`: should these match the parent route's pattern (per-connection error aggregation with partial failure tolerance), or is a simpler try/catch sufficient since these routes operate on a single known connection? The simpler pattern is likely correct — these routes already validate the connection exists via `prisma.driveConnection.findFirst()` before calling the provider.
