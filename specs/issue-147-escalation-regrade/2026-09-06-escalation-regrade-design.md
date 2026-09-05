# Escalation Re-evaluation on Grade Upgrade — Design Spec

**Issue:** casehubio/clinical#147
**Date:** 2026-09-06
**Status:** Draft
**Depends on:** 2026-07-21-ae-grade-regrading-design.md (D1 selective re-evaluation)

## Problem

When an AE is regraded upward (e.g., Grade 3→4) and an engine case already
exists from the original escalation, `AeEscalationCaseService.prepareAndMarkForRegrade()`
returns null — no new engine case starts. The higher-grade AE has no safety
review at the escalated grade.

Specifically: a Grade 3 AE creates an engine case with
`requiresDsmbEscalation: false` (per `DefaultAdverseEventEscalationPolicy`).
The YAML `dsmb-escalation` binding never fires. When the grade upgrades to 4,
the policy would require DSMB (`requiresDsmbEscalation: true`), but the guard
at `prepareAndMarkForRegrade:92` short-circuits — only calling
`signalGrade4Active()` and returning null.

## Design Decision

**D1: Always start a fresh engine case on upgrade regrade** (see `decisions.md`).

Remove the `engineCaseId != null` guard. Re-evaluate the policy at the new
grade. Start a fresh case with updated context. The old case continues to
completion if active — its results are valid historical documentation but the
AE entity tracks the new case as authoritative.

Not configurable — the `AdverseEventEscalationPolicy` SPI is the sole
configuration point for grade-based escalation requirements.

## Changes

### 1. `AeEscalationCaseService.prepareAndMarkForRegrade()`

**Remove the early-return guard** (lines 92-96):

```java
// REMOVE:
if (ae.engineCaseId != null) {
    if (SEVERE_GRADES.contains(grade)) {
        trialSafetySignalService.signalGrade4Active(siteId);
    }
    return null;
}
```

The `signalGrade4Active` call is redundant — `startEscalationForRegrade()`
already calls it at line 73-74 after case creation.

The method now always proceeds to:
1. Cancel any WorkItem (`ae.workItemId = null`)
2. Set `ae.escalationStatus = AeEscalationStatus.REQUESTED`
3. Re-evaluate policy at the new grade
4. Build initial context with new grade, memory, and CBR plan
5. Return context → `startCase()` → `persistCaseId()` overwrites `engineCaseId`

### 2. `AeStatusUpdater.markCompleted()` — case-ID guarded completion

**Add `expectedCaseId` parameter** for atomic TOCTOU-safe discrimination
inside the `REQUIRES_NEW` transaction:

```java
@Transactional(TxType.REQUIRES_NEW)
public boolean markCompleted(UUID aeId, UUID expectedCaseId) {
    AdverseEvent ae = AdverseEvent.findById(aeId);
    if (ae == null) { return false; }
    if (ae.escalationStatus == AeEscalationStatus.COMPLETED) { return false; }
    if (expectedCaseId != null && !expectedCaseId.equals(ae.engineCaseId)) {
        LOG.infof("Superseded escalation case %s completed for aeId=%s — current case is %s, skipping status update",
            expectedCaseId, aeId, ae.engineCaseId);
        return false;
    }
    ae.escalationStatus = AeEscalationStatus.COMPLETED;
    return true;
}
```

The `expectedCaseId != null` guard preserves backward compatibility — callers
that don't have a case ID can pass null to get the original behavior.

Returning `false` for superseded cases triggers the existing
`if (!firstCompletion) return;` guard in `AeEscalationListener`, which skips
all downstream processing (ledger write, memory store, `AeEscalationCompletedEvent`).

### 3. `AeEscalationListener.onCaseLifecycle()` — pass case ID

Pass `event.caseId()` to the updated `markCompleted`:

```java
boolean firstCompletion = statusUpdater.markCompleted(aeId, event.caseId());
```

No other changes to the listener. The existing `if (!firstCompletion) return;`
guard handles superseded cases automatically.

## Edge Cases

### Rapid successive upgrades (Grade 3→4→5)

Each regrade starts a new case. `engineCaseId` is overwritten each time.
Only the final case's completion triggers downstream processing — intermediate
cases are superseded. Case IDs: A (Grade 3), B (Grade 4), C (Grade 5).
Cases A and B complete with `markCompleted` returning false (ID mismatch).
Case C completion triggers ledger + event.

### Same-grade no-op

`regradeAdverseEvent()` in `AdverseEventService` guards against same-grade
regrades — `AeGradeChangedEvent` is never fired. No change needed.

### Downgrade after upgrade (Grade 3→4→2)

`AeGradeChangeEscalationListener.onGradeChanged()` gates on
`event.isUpgrade()`. The 4→2 downgrade is not an upgrade — no escalation
re-evaluation. The Grade 4 case (if still active) continues to completion.

### Concurrent case completions

`markCompleted` uses `REQUIRES_NEW` transaction — only one concurrent call
succeeds per AE. The case-ID check inside the transaction prevents TOCTOU
races.

### Grade 1→3 upgrade (no prior engine case)

`engineCaseId` is null. The guard removal doesn't affect this path — the
method proceeds as before (WorkItem cancellation, new case creation).

## Scope Exclusions

- **WorkItem cancellation for active superseded cases:** The old case's
  safety review WorkItem stays open. The human reviewer may complete it
  after the new case starts. This is harmless documentation — the new case
  creates its own WorkItem at the correct grade. Adding explicit WorkItem
  cancellation via engine API is deferred.
- **`previousEscalationCaseId` column:** The old case is traceable via
  ledger entries (which carry `aeId`). No entity schema change needed.
- **Engine context mutation API:** Not needed — fresh case approach avoids
  the need to update running case contexts.

## Testing Strategy

### Unit tests

- `AeEscalationCaseServiceTest`: add test for `prepareAndMarkForRegrade`
  when `engineCaseId` is not null — verify it returns a valid context
  (not null), sets `escalationStatus = REQUESTED`, and builds context
  with the new grade
- `AeStatusUpdaterTest`: new tests for case-ID matching — superseded
  case returns false; matching case returns true; null expectedCaseId
  falls back to original behavior

### Integration tests

- Grade 3→4 regrade with completed Grade 3 case: verify new case starts,
  new `engineCaseId` is persisted, `escalationStatus` transitions
  COMPLETED→REQUESTED→COMPLETED
- Grade 3→4 regrade with active Grade 3 case: verify new case starts,
  old case completion is silently ignored (returns false from markCompleted)
- Grade 3→4→5 rapid upgrade: verify only Grade 5 case's completion
  triggers downstream processing

### Edge case tests

- Same-grade regrade: verify no-op (existing test, no change)
- Downgrade after upgrade: verify no new escalation case
- `signalGrade4Active` called for Grade 4+ regardless of prior case state

## Flyway Migrations

None required. All changes are code-only — no new tables or columns.

## References

- `AeEscalationCaseService.java:92-96` — guard being removed
- `AeStatusUpdater.java:28-42` — completion handler gaining case-ID parameter
- `AeEscalationListener.java:35-92` — lifecycle listener passing case ID
- `DefaultAdverseEventEscalationPolicy.java:23-28` — Grade→requirements mapping
- `ae-escalation.yaml` — case definition with `requiresDsmbEscalation` filter
- `2026-07-21-ae-grade-regrading-design.md:189-211` — parent spec escalation section
- ICH E6(R3) §5.17 — proportional severity response obligation
- CTCAE v5.0 — grading as point-in-time clinical assessment
