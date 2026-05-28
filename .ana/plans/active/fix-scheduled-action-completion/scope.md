# Scope: Fix scheduled action failure leaving ExecutedRule stuck at APPLYING

**Created by:** Ana
**Date:** 2026-05-28

## Intent

When a scheduled action fails, the parent ExecutedRule is never transitioned out of APPLYING status. It stays there forever. And when a sibling action succeeds and triggers the completion check, it doesn't notice FAILED siblings — so it marks the rule APPLIED even though some actions failed. The user wants to fix this as an open-source contribution that demonstrates real understanding of the codebase.

## Complexity Assessment
- **Kind:** fix
- **Size:** small — 1 source file, 1 test file
- **Surface:** web
- **Files affected:** `apps/web/utils/scheduled-actions/executor.ts`, `apps/web/utils/scheduled-actions/executor.test.ts`
- **Blast radius:** Low. `checkAndCompleteExecutedRule` is a private function called only within `executor.ts`. The only behavioral change is that ExecutedRules will now correctly transition to ERROR or APPLIED instead of staying stuck at APPLYING. The ERROR status already exists in the enum and is used by the immediate-action path in `execute.ts`.
- **Estimated effort:** ~1 hour
- **Multi-phase:** no

## Approach

Two bugs, one root cause: `checkAndCompleteExecutedRule` is only called on the success path and only checks for the absence of pending work — it doesn't account for failed work.

Fix both by calling the completion check after failure too, and teaching it to distinguish between "all done successfully" (APPLIED) and "all done but some failed" (ERROR). This aligns with how the immediate-action path already handles errors in `execute.ts`.

## Acceptance Criteria
- AC1: When a scheduled action fails and it is the only (or last) action for an ExecutedRule, the ExecutedRule transitions to ERROR — not stuck at APPLYING.
- AC2: When a rule has multiple scheduled actions and some fail while others succeed, the ExecutedRule transitions to ERROR — not APPLIED.
- AC3: When all scheduled actions succeed, existing behavior is unchanged — ExecutedRule transitions to APPLIED.
- AC4: Tests cover all three scenarios: single action fails, mixed success/failure, all succeed.

## Edge Cases & Risks
- If `markActionFailed` itself throws (DB error), the subsequent `checkAndCompleteExecutedRule` call won't run. This is acceptable — the existing success path has the same characteristic with `markActionCompleted`. The rule stays APPLYING, which is at least honest about being incomplete.
- A race between two actions completing simultaneously could result in both calling `checkAndCompleteExecutedRule`. This is safe — Prisma `count` + `update` is not atomic, but the worst case is two updates to the same terminal status, which is idempotent.

## Rejected Approaches
- **Adding retry logic for failed actions** — Out of scope. This fixes the status tracking bug. Retry is a separate feature.
- **Surfacing failures to users via notification** — Separate concern. The rule engine already handles ERROR status downstream; this fix just ensures it gets set.
- **Making the completion check atomic (transaction)** — Dynamic transactions are banned in this codebase. The non-atomic check is fine here because the outcome is idempotent.

## Exploration Findings

### Patterns Discovered
- `execute.ts` (immediate actions) already sets ExecutedRuleStatus.ERROR on failure — lines 87-103 in that file. The scheduled action path should mirror this.
- The `ExecutedRuleStatus` enum already has ERROR — no schema migration needed.
- E2E test helpers at `__tests__/e2e/flows/helpers/polling.ts:67` already treat ERROR as a terminal status: `const TERMINAL_STATUSES = ["APPLIED", "SKIPPED", "ERROR"]`. So downstream code already handles this correctly.

### Constraints Discovered
- [TYPE-VERIFIED] ExecutedRuleStatus enum (schema.prisma:1693-1700) — has APPLIED, APPLYING, REJECTED (deprecated), PENDING (deprecated), SKIPPED, ERROR
- [TYPE-VERIFIED] ScheduledActionStatus enum — has PENDING, EXECUTING, COMPLETED, FAILED, CANCELLED
- [OBSERVED] No dynamic Prisma transactions — codebase convention, enforced in AGENTS.md

### Test Infrastructure
- `executor.test.ts` uses vitest with mocked prisma. Existing test "should handle execution errors and mark as failed" (line 272) only asserts the ScheduledAction status — never checks ExecutedRule status or whether `checkAndCompleteExecutedRule` was called.

## For AnaPlan

### Structural Analog
`apps/web/utils/ai/choose-rule/execute.ts` — the immediate-action execution path. It handles the success→APPLIED and failure→ERROR transitions for non-delayed actions. The scheduled action executor should mirror this pattern.

### Relevant Code Paths
- `apps/web/utils/scheduled-actions/executor.ts` — the entire file. Success path: lines 82-83 (markActionCompleted → checkAndCompleteExecutedRule). Failure path: line 97 (markActionFailed only). Completion check: lines 278-301.
- `apps/web/utils/scheduled-actions/executor.test.ts` — existing tests. The success test (line 66) mocks `prisma.scheduledAction.count` and `prisma.executedRule.update`. The failure test (line 272) does not.

### Patterns to Follow
- Mirror the success path structure: `markActionFailed` then `checkAndCompleteExecutedRule`, same as `markActionCompleted` then `checkAndCompleteExecutedRule` on line 82-83.
- Use `ExecutedRuleStatus.ERROR` consistent with `execute.ts`.

### Known Gotchas
- `checkAndCompleteExecutedRule` currently only checks `[PENDING, EXECUTING]` in the count query. The fix needs to add a second query or modify the existing one to also count FAILED actions and choose the right terminal status.
- The test file uses `as any` casts on mock return values — follow the existing test patterns, don't try to improve the test infrastructure in this PR.

### Things to Investigate
- Whether `checkAndCompleteExecutedRule` should set an error reason on the ExecutedRule (the immediate path in `execute.ts` sets a reason string). Design judgment call — could include a generic "One or more scheduled actions failed" or omit it.
