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

This overrides the previous design decision in
`2026-07-21-ae-grade-regrading-design.md:209-211` which treated the existing
case as sufficient and explicitly tested for "no duplicate." The override is
justified because `DefaultAdverseEventEscalationPolicy` returns different
requirements per grade (Grade 3 → senior monitor only; Grade 4 → senior
monitor + DSMB). A case created with Grade 3 requirements cannot satisfy
Grade 4 obligations without context mutation, which the engine doesn't support.

Not configurable — the `AdverseEventEscalationPolicy` SPI is the sole
configuration point for grade-based escalation requirements. The mechanism
for satisfying those requirements is plumbing, not policy.

## Changes

### 1. `AeGradeChangeEscalationListener` — add grade threshold gate

The listener currently fires for ANY upgrade. Grade 1→2 would incorrectly
enter the engine case path (policy returns `direct("safety-officers")` with
`engineCaseRequired=false`), starting a case that hangs forever — no bindings
fire when both `requiresSeniorMonitor` and `requiresDsmbEscalation` are false.

**Add grade threshold gate:**

```java
public void onGradeChanged(@ObservesAsync AeGradeChangedEvent event) {
    if (!event.isUpgrade()) return;
    if (event.newGrade().ordinal() < CtcaeGrade.GRADE_3.ordinal()) return;
    escalationService.startEscalationForRegrade(
        event.aeId(), event.enrollmentId(), event.siteId(),
        event.newGrade(), event.tenantId());
}
```

Grade 1→2 upgrades: no action (both grades are WorkItem-managed).
Grade 2→3 upgrades: enters the regrade path (threshold crossing).
Grade 3→4/5 upgrades: enters the regrade path (D1 handles existing case).

### 2. `AeEscalationCaseService.prepareAndMarkForRegrade()`

**Remove the early-return guard** (lines 92-96) and **null out `engineCaseId`**
to close the TOCTOU race window:

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

**Add `engineCaseId` nullification** before setting REQUESTED:

```java
ae.engineCaseId = null;  // supersede old case — closes TOCTOU window
ae.escalationStatus = AeEscalationStatus.REQUESTED;
```

**Why null:** The three-phase pattern creates a window between
`prepareAndMarkForRegrade()` (phase 1) and `persistCaseId()` (phase 3).
If the old case completes during this window, case-ID discrimination would
match the old ID (still on the entity) and set status to COMPLETED — losing
the new case's completion. Nulling `engineCaseId` in phase 1 ensures any
old-case completion during the window sees a null mismatch and is correctly
treated as superseded.

The method now proceeds to:
1. Null `engineCaseId` and cancel any WorkItem
2. Set `ae.escalationStatus = AeEscalationStatus.REQUESTED`
3. Re-evaluate policy at the new grade
4. Build initial context with new grade, memory, and CBR plan
5. Return context → `startCase()` → `persistCaseId()` writes new `engineCaseId`

### 3. `AeStatusUpdater.markCompleted()` — case-ID guarded completion

**Return `CompletionResult` enum** instead of boolean, with `expectedCaseId`
parameter:

```java
public enum CompletionResult {
    COMPLETED,          // status newly set to COMPLETED
    ALREADY_COMPLETED,  // idempotent — was already COMPLETED
    SUPERSEDED,         // case ID doesn't match current engineCaseId
    NOT_FOUND           // AE doesn't exist
}

@Transactional(TxType.REQUIRES_NEW)
public CompletionResult markCompleted(UUID aeId, UUID expectedCaseId) {
    AdverseEvent ae = AdverseEvent.findById(aeId);
    if (ae == null) { return NOT_FOUND; }
    if (ae.escalationStatus == AeEscalationStatus.COMPLETED) {
        return ALREADY_COMPLETED;
    }
    if (expectedCaseId != null && !expectedCaseId.equals(ae.engineCaseId)) {
        LOG.infof("Superseded case %s completed for aeId=%s — current case is %s",
            expectedCaseId, aeId, ae.engineCaseId);
        return SUPERSEDED;
    }
    ae.escalationStatus = AeEscalationStatus.COMPLETED;
    return COMPLETED;
}
```

### 4. `AeEscalationListener.onCaseLifecycle()` — differentiated completion handling

Replace the boolean `firstCompletion` check with `CompletionResult` branching:

```java
CompletionResult result = statusUpdater.markCompleted(aeId, event.caseId());
if (result == NOT_FOUND || result == ALREADY_COMPLETED) return;

// Resolve fields from context snapshot (existing code)
UUID enrollmentId = resolveUuid(snapshot.path("enrollmentId").asText(null));
// ... (grade, siteId, safetyReviewOutcome, dsmbEscalated, etc.)

// Both COMPLETED and SUPERSEDED write ledger entries (ALCOA+ compliance)
ledgerWriter.writeCompletionEntry(aeId, enrollmentId, grade,
    safetyReviewOutcome, dsmbEscalated, completedAt);

if (result == SUPERSEDED) return;  // ledger recorded, skip authoritative actions

// Authoritative completion only:
memoryService.storeAeOutcome(...);
completedEvents.fireAsync(new AeEscalationCompletedEvent(...));
```

**Audit trail rationale:** A completed safety review has independent audit
value under ALCOA+ data integrity principles. The superseded case's review
was performed by a human at the historical grade — that completion must be
recorded in the ledger. The `grade` field in the ledger entry (from the
case context snapshot) distinguishes Grade 3 from Grade 4 completions.
Memory store and downstream events fire only for the authoritative
(current-case) completion.

## Edge Cases

### Rapid successive upgrades (Grade 3→4→5)

Each regrade starts a new case. `engineCaseId` is nulled in phase 1, then
set to the new case ID in phase 3. Only the final case's completion triggers
memory store and `AeEscalationCompletedEvent`. Intermediate case completions
write ledger entries but skip authoritative actions.

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

`engineCaseId` is null. The method proceeds as before — WorkItem cancellation,
new case creation. The nullification of `engineCaseId` is a no-op.

### Grade 1→2 upgrade (below threshold)

Blocked by the new grade threshold gate in `AeGradeChangeEscalationListener`.
No escalation re-evaluation — both grades are WorkItem-managed.

### TOCTOU window (old case completes during phase 2-3)

`engineCaseId` is null (cleared in phase 1). Old case completes →
`markCompleted(aeId, oldCaseId)` → `ae.engineCaseId == null != oldCaseId` →
returns `SUPERSEDED`. Ledger entry written. No status corruption. New case
completes → `markCompleted(aeId, newCaseId)` → match → `COMPLETED`.

## Scope Exclusions

- **WorkItem cancellation for active superseded cases:** The old case's
  safety review WorkItem stays open. The human reviewer may complete it
  after the new case starts — their review is recorded in the ledger but
  does not update authoritative state. Adding explicit WorkItem cancellation
  via engine API is deferred. A future issue could also inject regrade context
  into the superseded case's WorkItem payload to inform the reviewer.
- **`previousEscalationCaseId` column:** The old case is traceable via
  ledger entries (which carry `aeId` and `grade`). No entity schema change
  needed.
- **`AdverseEventContext` regrade awareness:** The policy SPI context has no
  `previousGrade` field and cannot distinguish initial report from regrade.
  Custom policies that need different escalation behavior for grade transitions
  would need this. Deferred — the default CTCAE-based policy treats both
  scenarios identically.
- **Engine context mutation API:** Not needed — fresh case approach avoids
  the need to update running case contexts.
- **Reviewer notification on supersession:** The human reviewer on the old
  case is not notified that the AE grade changed. This requires engine
  context mutation (not supported). Deferred.

## Testing Strategy

### Unit tests

- `AeEscalationCaseServiceTest`: verify `prepareAndMarkForRegrade` with
  existing `engineCaseId` — returns valid context (not null), nulls
  `engineCaseId`, sets `escalationStatus = REQUESTED`, builds context
  with new grade
- `AeStatusUpdaterTest`: new tests for `CompletionResult` — SUPERSEDED
  for case-ID mismatch; COMPLETED for matching case; ALREADY_COMPLETED
  for repeat; NOT_FOUND for missing AE
- `AeGradeChangeEscalationListenerTest`: verify Grade 1→2 upgrade does
  NOT call `startEscalationForRegrade`; Grade 2→3 and Grade 3→4 DO

### Integration tests

- Grade 3→4 regrade with completed Grade 3 case: verify new case starts,
  new `engineCaseId` is persisted, `escalationStatus` transitions
  COMPLETED→REQUESTED→COMPLETED
- Grade 3→4 regrade with active Grade 3 case: verify new case starts,
  old case completion writes ledger entry but returns SUPERSEDED
- Grade 3→4→5 rapid upgrade: verify only Grade 5 case's completion
  triggers memory store and downstream event; all three write ledger entries

### Edge case tests

- Grade 1→2 upgrade: no escalation service call (threshold gate)
- Same-grade regrade: no-op (existing test, no change)
- Downgrade after upgrade: no new escalation case
- `signalGrade4Active` called for Grade 4+ regardless of prior case state
- TOCTOU window: old case completes with null `engineCaseId` → SUPERSEDED

## Flyway Migrations

None required. All changes are code-only — no new tables or columns.

## References

- `AeEscalationCaseService.java:92-96` — guard being removed
- `AeGradeChangeEscalationListener.java:13-17` — listener gaining threshold gate
- `AeStatusUpdater.java:28-42` — completion handler gaining CompletionResult
- `AeEscalationListener.java:35-92` — lifecycle listener with differentiated handling
- `DefaultAdverseEventEscalationPolicy.java:23-28` — Grade→requirements mapping
  (primary source for why Grade 3 and Grade 4 need different responses)
- `ae-escalation.yaml` — case definition with `requiresDsmbEscalation` filter
- `2026-07-21-ae-grade-regrading-design.md:189-211,480` — previous design decision
  (explicitly "no duplicate" for Grade 3→4; this spec overrides that decision)
- ICH E6(R3) §5.17 — context: reporting obligations proportional to severity
  (informed the policy design, not a direct mandate on engine case management)
- CTCAE v5.0 — context: severity grading classification system
- Decision review R1-02 through R1-09 — adversarial findings that shaped this spec
