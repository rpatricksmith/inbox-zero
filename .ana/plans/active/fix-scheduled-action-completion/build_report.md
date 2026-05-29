# Build Report: Fix scheduled action failure leaving ExecutedRule stuck at APPLYING

**Created by:** AnaBuild
**Date:** 2026-05-28
**Spec:** .ana/plans/active/fix-scheduled-action-completion/spec.md
**Branch:** feature/fix-scheduled-action-completion

## What Was Built

- `apps/web/utils/scheduled-actions/executor.ts` (modified): Added `checkAndCompleteExecutedRule` call in catch block after `markActionFailed` (mirroring success path). Updated `checkAndCompleteExecutedRule` to count FAILED actions when no pending/executing remain — sets `ExecutedRuleStatus.ERROR` with reason if any failed, `APPLIED` if none failed.
- `apps/web/utils/scheduled-actions/executor.test.ts` (modified): Added 4 new test cases covering single-action failure (ERROR), mixed success/failure (ERROR), all-succeed (APPLIED), and still-pending (no update). Updated existing failure test and account-not-found test to mock the completion check's count calls.

## PR Summary

- Fix ExecutedRule getting stuck at APPLYING when scheduled actions fail by calling the completion check on the failure path
- Update `checkAndCompleteExecutedRule` to detect FAILED actions and set `ExecutedRuleStatus.ERROR` with reason instead of blindly marking APPLIED
- Add 4 tests covering failure completion, mixed results, all-succeed, and still-pending scenarios
- Update 2 existing tests to properly mock the new completion check behavior on the failure path

## Acceptance Criteria Coverage

- AC1 "When a scheduled action fails and it is the only/last action, ExecutedRule transitions to ERROR" -> executor.test.ts "should transition ExecutedRule to ERROR when last action fails" (4 assertions: result.success, scheduledAction.update status, scheduledAction.count called, executedRule.update with ERROR+reason)
- AC2 "When some actions fail and others succeed, ExecutedRule transitions to ERROR" -> executor.test.ts "should transition ExecutedRule to ERROR when some actions fail and others succeed" (2 assertions: result.success, executedRule.update with ERROR+reason)
- AC3 "When all actions succeed, ExecutedRule transitions to APPLIED" -> executor.test.ts "should transition ExecutedRule to APPLIED when all actions succeed" (2 assertions: result.success, executedRule.update with APPLIED)
- AC4 "Tests cover all three scenarios" -> All three scenario tests present plus a fourth for still-pending

## Implementation Decisions

- Added a fourth test for "actions still pending" (contract A008) — the spec listed 3 new tests but the contract had an assertion for this scenario. The test verifies `executedRule.update` is NOT called when pending count > 0.
- Used `mockResolvedValueOnce` chaining for `prisma.scheduledAction.count` to return different values for the pending-count call vs the failed-count call, as the spec's Gotchas section recommended.

## Deviations from Contract

None — contract followed exactly.

## Test Results

### Baseline (before changes)
```
pnpm run test -- --run
Test Files  416 passed | 95 skipped (511)
Tests  3785 passed | 659 skipped (4444)
```

### After Changes
```
pnpm run test -- --run
Test Files  416 passed | 95 skipped (511)
Tests  3789 passed | 659 skipped (4448)
```

### Comparison
- Tests added: 4
- Tests removed: 0
- Regressions: none

### New Tests Written
- `apps/web/utils/scheduled-actions/executor.test.ts`:
  - "should transition ExecutedRule to ERROR when last action fails" (A001, A002, A003, A009, A010)
  - "should transition ExecutedRule to ERROR when some actions fail and others succeed" (A004, A005)
  - "should transition ExecutedRule to APPLIED when all actions succeed" (A006, A007)
  - "should not update ExecutedRule status when actions are still pending" (A008)

### Contract Coverage
10/10 assertions tagged: A001, A002, A003, A004, A005, A006, A007, A008, A009, A010

## Verification Commands
```
pnpm run test -- --run
(cd 'apps/web' && pnpm run test -- --run utils/scheduled-actions/executor.test.ts)
pnpm run lint
```

## Git History
```
761921454 [fix-scheduled-action-completion] Add tests for failure completion scenarios
eae04ce63 [fix-scheduled-action-completion] Fix completion check for failed scheduled actions
```

## Open Issues

During one full test suite run, 2 unrelated tests timed out (`utils/auth-login-providers.test.ts` and `utils/actions/assistant-chat.server-action-boundary.test.ts`). These are environmental/flaky — they passed on the subsequent run and are in completely unrelated modules. Not introduced by this build.

Verified complete by second pass.
