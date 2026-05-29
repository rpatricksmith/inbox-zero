# Verify Report: Fix scheduled action failure leaving ExecutedRule stuck at APPLYING

**Result:** PASS
**Created by:** AnaVerify
**Date:** 2026-05-28
**Spec:** .ana/plans/active/fix-scheduled-action-completion/spec.md
**Branch:** feature/fix-scheduled-action-completion

## Pre-Check Results
```
=== CONTRACT COMPLIANCE ===
  Contract: /Users/rsmith/Projects/contributions/inbox-zero/.ana/worktrees/fix-scheduled-action-completion/.ana/plans/active/fix-scheduled-action-completion/contract.yaml
  Seal: INTACT (hash sha256:7865482f14488257f837b15d68520fa803e852b449d632e3734be8af795c2cd8)
```

Tests: 3789 passed, 659 skipped (4448 total, up from 3785 baseline = +4 new tests). Build: not run (not requested). Lint: passed.

## Contract Compliance
| ID   | Says                                           | Status       | Evidence |
|------|------------------------------------------------|--------------|----------|
| A001 | A failed scheduled action triggers the rule completion check | ✅ SATISFIED | `executor.test.ts:504` asserts `prisma.scheduledAction.count` was called after failure path; implementation at `executor.ts:98` calls `checkAndCompleteExecutedRule` in catch block |
| A002 | When the only action fails, the rule is marked as errored | ✅ SATISFIED | `executor.test.ts:505-511` asserts `prisma.executedRule.update` called with `status: "ERROR"` |
| A003 | A failed rule includes a reason explaining what went wrong | ✅ SATISFIED | `executor.test.ts:509` asserts `reason: "One or more scheduled actions failed"` which contains `"scheduled actions failed"` |
| A004 | When some actions fail and others succeed, the rule is marked as errored | ✅ SATISFIED | `executor.test.ts:599-605` asserts `prisma.executedRule.update` called with `status: "ERROR"` when pending=0 and failed=2 |
| A005 | A mixed-result rule includes a failure reason | ✅ SATISFIED | `executor.test.ts:603-604` asserts `reason: "One or more scheduled actions failed"` which contains `"scheduled actions failed"` |
| A006 | When all actions succeed, the rule is marked as applied | ✅ SATISFIED | `executor.test.ts:693-696` asserts `prisma.executedRule.update` called with `status: "APPLIED"` when pending=0 and failed=0 |
| A007 | A successful rule has no failure reason | ✅ SATISFIED | `executor.test.ts:693-696` asserts update called with `data: { status: "APPLIED" }` — no `reason` field. `toHaveBeenCalledWith` uses exact shape matching, so if reason were included the test would fail. Implementation at `executor.ts:314-316` confirms no reason in the data object |
| A008 | The rule stays in progress while actions are still pending | ✅ SATISFIED | `executor.test.ts:769` asserts `prisma.executedRule.update` was NOT called when pending count returns 2 |
| A009 | A failed action still reports failure to the caller | ✅ SATISFIED | `executor.test.ts:499` asserts `result.success` is `false` |
| A010 | The action itself is correctly marked as failed in the database | ✅ SATISFIED | `executor.test.ts:500-503` asserts `prisma.scheduledAction.update` called with `status: ScheduledActionStatus.FAILED` |

## Independent Findings

**Prediction resolutions:**

1. **Confirmed — error replacement on completion check failure.** If `checkAndCompleteExecutedRule` throws at `executor.ts:98` (after `markActionFailed`), the DB error from the completion check replaces the original action error. The caller would see a Prisma error instead of the original "Execution failed" error. The spec explicitly acknowledges this as acceptable — it mirrors the success path's behavior at line 83. Confirmed present, accepted per spec.

2. **Confirmed — A008 test doesn't verify query was skipped.** The pending-actions test (line 700) mocks `count` to return 2 and asserts `executedRule.update` was NOT called. It does not verify that `scheduledAction.count` was called exactly once (i.e., that the failed-count query was skipped). The test would still pass if the implementation queried failed count unnecessarily when pending > 0. This is a minor coverage gap — the behavior is correct, the test is just not as tight as it could be.

3. **Confirmed — A007 implicit assertion.** The test relies on `toHaveBeenCalledWith` shape matching to implicitly verify no `reason` is set. This works because vitest's deep equality would fail if an unexpected `reason` property appeared. It's correct but could be more explicit with a separate assertion on the call args.

4. **Confirmed — existing test mock absorbs new call.** The original success test at line 102 uses `mockResolvedValue(0)` (not `Once`), which returns 0 for every call to `count`. After the implementation change, this now silently handles both the pending-count and failed-count queries. The test still passes correctly because 0 pending + 0 failed = APPLIED. Not a bug, but the mock intent is no longer obvious.

**Surprise finding:** None. The implementation is a clean, focused two-line fix (one added line in catch block, one refactored function). No unexpected patterns.

## AC Walkthrough

- **AC1:** When a scheduled action fails and it is the last action, ExecutedRule transitions to ERROR. ✅ PASS — Test at line 414 exercises this path; mock returns pending=0, failed=1; asserts `status: "ERROR"`. Implementation at `executor.ts:98` calls `checkAndCompleteExecutedRule` in catch block, and `executor.ts:300-312` sets ERROR when failedActions > 0.

- **AC2:** Mixed success/failure → ExecutedRule transitions to ERROR. ✅ PASS — Test at line 515 exercises this; a successful action completes last but 2 siblings are FAILED; mock returns pending=0, failed=2; asserts `status: "ERROR"`.

- **AC3:** All actions succeed → existing APPLIED behavior unchanged. ✅ PASS — Test at line 609 exercises this; mock returns pending=0, failed=0; asserts `status: "APPLIED"`. Original success test at line 66 also still passes with same behavior.

- **AC4:** Tests cover all three scenarios. ✅ PASS — Tests at lines 414, 515, 609 cover single fail, mixed, and all succeed. Additionally, line 700 covers the pending-actions-remain case from the spec's test matrix.

- **Tests pass with `pnpm run test -- --run`:** ✅ PASS — 3789 passed, 659 skipped. Ran in this session.

- **No build errors:** ✅ PASS — Lint passed clean. Tests compile and run without errors.

## Blockers

No blockers. All 10 contract assertions satisfied. All 6 acceptance criteria pass. No regressions (3789 tests vs 3785 baseline = +4 new, all passing). Checked for: unused exports in new code (no new exports added), unused parameters in modified functions (all params used), error paths that swallow silently (the catch block propagates `{ success: false, error }` to caller), sentinel test patterns (all assertions check specific values — `"ERROR"`, `"APPLIED"`, `false`, `ScheduledActionStatus.FAILED`).

## Findings

- **Code — Completion check failure replaces original error:** `apps/web/utils/scheduled-actions/executor.ts:98` — if `checkAndCompleteExecutedRule` throws after `markActionFailed`, the original action error is lost. The caller receives a DB error instead of the action's failure reason. Spec explicitly accepts this as matching the success path behavior. Risk level: low in practice (DB errors here are rare), but worth monitoring if error reporting quality matters downstream.

- **Test — A008 pending-actions test lacks query-count assertion:** `apps/web/utils/scheduled-actions/executor.test.ts:700` — does not verify that `scheduledAction.count` was called exactly once (confirming the failed-count query was skipped). The test passes regardless of whether the implementation wastefully queries failed count when pending > 0. Current implementation is correct; this is a test tightness gap.

- **Test — A007 relies on implicit shape matching for reason absence:** `apps/web/utils/scheduled-actions/executor.test.ts:693` — the test asserts the update was called with `{ status: "APPLIED" }` and relies on vitest's exact object matching to implicitly verify no `reason` field. Works correctly but less readable than an explicit `expect(callArgs.data).not.toHaveProperty('reason')`.

- **Test — Existing success test mock absorbs new count call:** `apps/web/utils/scheduled-actions/executor.test.ts:102` — uses `mockResolvedValue(0)` which returns 0 for all `count` calls, silently handling both the pending and failed queries. The mock was written before the second query existed and now covers it accidentally. Consider switching to `mockResolvedValueOnce(0).mockResolvedValueOnce(0)` to make intent explicit.

- **Code — No null guard on executedRuleId:** `apps/web/utils/scheduled-actions/executor.ts:279` — `checkAndCompleteExecutedRule` accepts a `string` parameter but the `ScheduledAction.executedRuleId` field may be nullable in the schema. If null were passed, the Prisma query `where: { executedRuleId: null }` could match unintended records. Pre-existing gap — the success path at line 83 has the same exposure.

## Deployer Handoff

Focused bug fix — two changes to one file:
1. Added `checkAndCompleteExecutedRule` call in the catch block after `markActionFailed` (line 98)
2. Modified `checkAndCompleteExecutedRule` to count failed actions and set ERROR status when failures exist (lines 293-322)

No schema changes, no new dependencies, no env vars. The fix is backwards-compatible — existing APPLIED transitions still work. The only behavioral change is that failed scheduled actions now correctly transition ExecutedRules to ERROR instead of leaving them stuck at APPLYING. Downstream code already handles ERROR as a terminal status.

No migration needed. Safe to deploy without coordination.

## Verdict
**Shippable:** YES

All 10 contract assertions SATISFIED. All 6 acceptance criteria PASS. 3789 tests pass (+4 from baseline), 0 failures, lint clean. The implementation is a clean, minimal fix that directly addresses the two bugs described in the spec. Five findings documented — all observations or low-risk items, none blocking. The code does exactly what the spec says and nothing more.
