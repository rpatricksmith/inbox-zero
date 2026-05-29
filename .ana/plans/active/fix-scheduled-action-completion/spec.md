# Spec: Fix scheduled action failure leaving ExecutedRule stuck at APPLYING

**Created by:** AnaPlan
**Date:** 2026-05-28
**Scope:** .ana/plans/active/fix-scheduled-action-completion/scope.md

## Approach

Two bugs, one root cause: `checkAndCompleteExecutedRule` is only called on the success path and only checks for the absence of pending work — it doesn't account for failed work.

**Bug 1 — completion check never runs on failure:** In `executeScheduledAction`, the catch block at line 97 calls `markActionFailed` but never calls `checkAndCompleteExecutedRule`. If this was the last action for the rule, the ExecutedRule stays APPLYING forever.

**Bug 2 — completion check ignores failures:** `checkAndCompleteExecutedRule` counts only PENDING and EXECUTING actions. When that count is 0, it sets APPLIED. It never checks whether any actions are FAILED — so even if a sibling action failed earlier, the rule gets marked APPLIED.

**Fix:**
1. Add a `checkAndCompleteExecutedRule` call after `markActionFailed` in the catch block — mirroring the success path.
2. Modify `checkAndCompleteExecutedRule` to count FAILED actions after confirming no pending/executing remain. If any failed, set `ExecutedRuleStatus.ERROR` with reason `"One or more scheduled actions failed"`. If none failed, set `APPLIED` as before.

This mirrors how `execute.ts` (the immediate-action path) already handles ERROR transitions via `ExecutedRuleStatus.ERROR`.

**Open question from scope resolved:** "Whether to set an error reason" → Yes. The ExecutedRule model has a nullable `reason` field. The immediate path sets a reason on ERROR. Use a generic string since scheduled actions don't have structured error codes like the immediate path's `ActionFailure` type.

## Output Mockups

No user-visible output changes. This fix affects internal status transitions in the database. The observable effect is that ExecutedRules will correctly transition to ERROR (instead of staying stuck at APPLYING) when scheduled actions fail, which downstream code already handles — the E2E helpers already treat ERROR as a terminal status.

## File Changes

### `apps/web/utils/scheduled-actions/executor.ts` (modify)
**What changes:** Two modifications:
1. In `executeScheduledAction`'s catch block: add `checkAndCompleteExecutedRule` call after `markActionFailed`, same as the success path on line 83.
2. In `checkAndCompleteExecutedRule`: after confirming no pending/executing actions remain, count FAILED actions. If any exist, set `ExecutedRuleStatus.ERROR` with reason. Otherwise set `APPLIED` as before.
**Pattern to follow:** The success path in the same file (lines 82–83) and the ERROR transition in `apps/web/utils/ai/choose-rule/execute.ts` (lines 98–101, 106–114).
**Why:** Without this, failed scheduled actions leave ExecutedRules permanently stuck at APPLYING, and mixed success/failure scenarios incorrectly mark rules as APPLIED.

### `apps/web/utils/scheduled-actions/executor.test.ts` (modify)
**What changes:** Add three new test cases covering the three scenarios: single action fails (rule → ERROR), mixed success/failure (rule → ERROR), all succeed (rule → APPLIED, already covered but verify completion check assertions).
**Pattern to follow:** The existing success test at line 66 — it mocks `prisma.scheduledAction.count` returning 0 and asserts `prisma.executedRule.update` was called with APPLIED.
**Why:** The existing failure test at line 272 only asserts the ScheduledAction status — it never checks the ExecutedRule status or whether the completion check ran.

## Acceptance Criteria

- [x] AC1: When a scheduled action fails and it is the only (or last) action for an ExecutedRule, the ExecutedRule transitions to ERROR — not stuck at APPLYING.
- [x] AC2: When a rule has multiple scheduled actions and some fail while others succeed, the ExecutedRule transitions to ERROR — not APPLIED.
- [x] AC3: When all scheduled actions succeed, existing behavior is unchanged — ExecutedRule transitions to APPLIED.
- [x] AC4: Tests cover all three scenarios: single action fails, mixed success/failure, all succeed.
- [x] Tests pass with `pnpm run test -- --run`
- [x] No build errors

## Testing Strategy

- **Unit tests:** Extend `executor.test.ts` with three new test cases. Mock `prisma.scheduledAction.count` to return appropriate values for each scenario. Assert `prisma.executedRule.update` is called with the correct status and reason.
- **Test matrix:**
  | Scenario | count(pending/executing) | count(failed) | Expected status | Expected reason |
  |---|---|---|---|---|
  | Single action fails, last for rule | 0 | 1 | ERROR | "One or more scheduled actions failed" |
  | Mixed: some fail, some succeed, last completes | 0 | ≥1 | ERROR | "One or more scheduled actions failed" |
  | All succeed | 0 | 0 | APPLIED | none |
  | Actions still pending | ≥1 | (not checked) | no update | — |
- **Edge cases:** The existing failure test (line 272) should be updated to also assert that `checkAndCompleteExecutedRule` runs by adding `prisma.scheduledAction.count` mock and `prisma.executedRule.update` assertion.

## Dependencies

None. The `ExecutedRuleStatus.ERROR` enum value already exists. The `reason` field on ExecutedRule already exists and is nullable.

## Constraints

- No dynamic Prisma transactions (codebase convention from AGENTS.md).
- The non-atomic completion check (count + update) is acceptable — the worst case is two updates to the same terminal status, which is idempotent. This matches the existing pattern.

## Gotchas

- `checkAndCompleteExecutedRule` will now be called from both the success and failure paths. The function needs to handle being called with FAILED actions present. The second count query (for FAILED) should only run when `pendingActions === 0` — no point counting failures if work is still in progress.
- The test file uses `as any` casts on mock return values — follow this existing pattern, don't try to improve the test infrastructure.
- `prisma.scheduledAction.count` is called with different `where` clauses for pending/executing vs failed. In tests, you'll need to handle the mock returning different values for different calls. Use `mockResolvedValueOnce` in sequence: first call returns pending count, second call returns failed count.
- If `markActionFailed` throws (DB error), the subsequent `checkAndCompleteExecutedRule` won't run. This is acceptable — matches the success path's behavior with `markActionCompleted`.

## Build Brief

### Rules That Apply
- Use `import type` for type-only imports, separate from value imports.
- Use path aliases (`@/`) for all imports.
- Use `ExecutedRuleStatus` and `ScheduledActionStatus` from `@/generated/prisma/enums` — already imported in executor.ts.
- Mock Prisma with `vi.mock("@/utils/prisma")` — already set up in the test file.
- Use `vi.clearAllMocks()` in `beforeEach` — already present.
- Assert on specific expected values, not just `.toBeDefined()`.
- Test behavior (what status gets written to DB), not implementation (which internal functions were called).

### Pattern Extracts

Success path in executor.ts (lines 82–83) — the pattern to mirror in the catch block:
```typescript
    await markActionCompleted(scheduledAction.id, executedAction?.id, log);
    await checkAndCompleteExecutedRule(scheduledAction.executedRuleId, log);
```

Current catch block (lines 91–99) — where to add the completion check call:
```typescript
  } catch (error: unknown) {
    log.error("Failed to execute scheduled action", {
      scheduledActionId: scheduledAction.id,
      error,
    });

    await markActionFailed(scheduledAction.id, error, log);
    return { success: false, error };
  }
```

Current completion check (lines 278–301) — the function to modify:
```typescript
async function checkAndCompleteExecutedRule(
  executedRuleId: string,
  log: Logger,
) {
  const pendingActions = await prisma.scheduledAction.count({
    where: {
      executedRuleId,
      status: {
        in: [ScheduledActionStatus.PENDING, ScheduledActionStatus.EXECUTING],
      },
    },
  });

  if (pendingActions === 0) {
    await prisma.executedRule.update({
      where: { id: executedRuleId },
      data: { status: ExecutedRuleStatus.APPLIED },
    });

    log.info("Completed ExecutedRule - all scheduled actions finished", {
      executedRuleId,
    });
  }
}
```

ERROR transition in execute.ts (lines 98–101) — the structural analog:
```typescript
      await prisma.executedRule.update({
        where: { id: executedRule.id },
        data: { status: ExecutedRuleStatus.ERROR },
      });
```

Existing test success assertion pattern (lines 102–115):
```typescript
      prisma.scheduledAction.count.mockResolvedValue(0);
      prisma.executedRule.update.mockResolvedValue({
        id: "rule-123",
        createdAt: new Date(),
        updatedAt: new Date(),
        messageId: "msg-123",
        threadId: "thread-123",
        emailAccountId: "account-123",
        status: "APPLIED",
        automated: true,
        reason: null,
        ruleId: null,
        matchMetadata: null,
      });
```

### Proof Context

No active proof findings for affected files.

### Checkpoint Commands

- After modifying `executor.ts`: `(cd 'apps/web' && pnpm run test -- --run utils/scheduled-actions/executor.test.ts)` — Expected: existing 4 tests pass
- After updating `executor.test.ts`: `(cd 'apps/web' && pnpm run test -- --run utils/scheduled-actions/executor.test.ts)` — Expected: 7 tests pass (4 existing + 3 new)
- After all changes: `pnpm run test -- --run` — Expected: 3785+ tests pass
- Lint: `pnpm run lint`

### Build Baseline
- Current tests: 3785 passed, 659 skipped (4444 total)
- Current test files: 416 passed, 95 skipped (511 total)
- Command used: `pnpm run test -- --run`
- After build: expected 3788+ tests in 416 test files (3 new tests in existing file)
- Regression focus: `apps/web/utils/scheduled-actions/executor.test.ts`
