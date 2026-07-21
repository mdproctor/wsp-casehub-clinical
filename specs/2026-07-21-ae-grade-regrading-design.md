# AE Grade Regrading — Design Spec

**Issue:** casehubio/clinical#135
**Date:** 2026-07-21
**Status:** Draft

## Problem

`AdverseEvent.grade` is set once at report time and never changes. In clinical
reality, adverse event severity changes over time — a Grade 1 nausea can
escalate to Grade 3 if the patient develops severe dehydration. The current
model cannot represent this, which means:

- No audit trail of grade transitions
- Trajectory time series has no grade dimension (tracks escalation/susar/regulatory
  status but not the clinical severity that drives them)
- A Grade 1→3 upgrade doesn't trigger the 24h SLA, engine case, or SUSAR
  evaluation that a Grade 3 AE requires at report time

## Design Decisions

### D1: Selective re-evaluation by direction

Grade **upgrades** trigger full re-evaluation (SLA, escalation, SUSAR, IND).
Grade **downgrades** are informational — recorded in history and audit trail
but don't trigger re-evaluation. Rationale: upgrading to serious is a new
regulatory reporting obligation (ICH E6(R3) §5.17); downgrading doesn't
un-report a serious event.

### D2: Mutable current grade + separate history entity

`AdverseEvent.grade` remains the mutable current grade field. A new
`AeGradeChange` entity records the full transition history. All 79 existing
call sites that read `ae.grade` continue unchanged. History is available for
trajectory analysis, audit, and dashboard display without forcing a join on
every grade access.

### D3: Initial report recorded as first history entry

`reportAdverseEvent()` inserts an `AeGradeChange(previousGrade=null,
newGrade=reportedGrade)` at report time. This gives a uniform timeline in one
table — no special-casing "was this the original grade or a change?"

## Domain Model

### AeGradeChange entity

```java
@Entity
@Table(name = "ae_grade_change")
public class AeGradeChange extends PanacheEntityBase {
    @Id
    public UUID id;

    @Column(name = "adverse_event_id", nullable = false)
    public UUID adverseEventId;

    @Enumerated(EnumType.STRING)
    @Column(name = "previous_grade")
    public CtcaeGrade previousGrade;   // null for initial report

    @Enumerated(EnumType.STRING)
    @Column(name = "new_grade", nullable = false)
    public CtcaeGrade newGrade;

    @Column(name = "changed_at", nullable = false)
    public Instant changedAt;

    @Column(name = "changed_by", nullable = false)
    public String changedBy;

    @Column(length = 500)
    public String reason;              // clinical justification
}
```

Lives in `io.casehub.clinical.entity` (default datasource, same as
`AdverseEvent`).

**Queries:**
- `findByAdverseEventId(UUID aeId)` — full history, ordered by `changedAt`
- `findLatestByAdverseEventId(UUID aeId)` — most recent change

### AdverseEvent — no structural change

`AdverseEvent.grade` stays as-is. Updated in-place during regrade. No new
fields needed on the AE entity.

## Domain Event

```java
// api module
public record AeGradeChangedEvent(
    UUID aeId,
    UUID enrollmentId,
    UUID siteId,
    CtcaeGrade previousGrade,
    CtcaeGrade newGrade,
    Instant changedAt,
    String changedBy,
    String tenantId
) {
    public boolean isUpgrade() {
        return newGrade.ordinal() > previousGrade.ordinal();
    }

    public boolean isDowngrade() {
        return newGrade.ordinal() < previousGrade.ordinal();
    }
}
```

## Service Layer

### AdverseEventService changes

**New method: `regradeAdverseEvent`**

```
@Transactional
regradeAdverseEvent(UUID aeId, CtcaeGrade newGrade, String changedBy, String reason):
    1. Load AE (tenant-scoped)
    2. Guard: newGrade == ae.grade → no-op, return
    3. Record: insert AeGradeChange(ae.grade, newGrade, now, changedBy, reason)
    4. Update: ae.grade = newGrade
    5. SLA: if upgrade AND newGrade.sla() < remaining time → ae.slaDeadline = now + newGrade.sla()
    6. Ledger: write AeGradeChangeLedgerEntry
    7. After-commit: fire AeGradeChangedEvent (same TransactionSynchronizationRegistry pattern)
```

**Existing method: `reportAdverseEvent` — one addition**

After `ae.persist()`, insert the initial grade history entry:
```
AeGradeChange initial = new AeGradeChange();
initial.id = UUID.randomUUID();
initial.adverseEventId = ae.id;
initial.previousGrade = null;
initial.newGrade = ae.grade;
initial.changedAt = ae.reportedAt;
initial.changedBy = "system";
initial.reason = "Initial report";
initial.persist();
```

## Upgrade Re-evaluation Listeners

All listeners observe `@ObservesAsync AeGradeChangedEvent` and gate on
`event.isUpgrade()`.

### Escalation re-evaluation

If the new grade crosses the Grade 3 threshold (`previousGrade < GRADE_3`,
`newGrade >= GRADE_3`) and `ae.engineCaseId == null`:
- Fire `AdverseEventReportedEvent` to trigger the existing escalation flow
- This reuses the entire escalation pipeline (engine case creation, safety
  review binding, DSMB rollup) without duplication

If an engine case already exists (Grade 3→4 upgrade): no action needed — the
case is already active.

### SUSAR criteria re-evaluation

Call `SusarCriteriaEvaluator.apply()` with the updated AE. If SUSAR criteria
now met and `susarOversightStatus == NONE`, the evaluator's existing logic
handles the rest.

### Regulatory submission re-evaluation

Call `RegulatorySubmissionCaseService.prepareAndMark()` if the new grade makes
the AE IND-reportable and `regulatorySubmissionStatus == NONE`.

### Trajectory alert re-evaluation

Call `AeTrajectoryAlertService.evaluate()` with the updated grade. The
trajectory now has a new data point.

## Trajectory Integration

### AeTrajectoryBuilder changes

The `Observation` inner class gains a `grade` field:

```java
private static class Observation {
    long secondsSinceReport;
    int escalation;
    int susar;
    int regulatory;
    int grade;        // CtcaeGrade ordinal + 1 (1-5)
}
```

The `doBuild()` method queries `AeGradeChange.findByAdverseEventId(ae.id)` and
injects grade change timestamps as observation points. Between grade changes,
the grade holds steady. This adds grade as a DTW-matchable dimension alongside
escalation/susar/regulatory.

The `toFeatureMap()` method includes `"grade"` in the output:
```java
Map.of(
    "ts", FeatureValue.number(secondsSinceReport),
    "escalation", FeatureValue.number(escalation),
    "susar", FeatureValue.number(susar),
    "regulatory", FeatureValue.number(regulatory),
    "grade", FeatureValue.number(grade)
);
```

### ClinicalCbrSchemaInitializer

The `ae-trajectory` schema already has `grade` as a top-level feature. The
trajectory time series inner fields need `grade` added. The schema initializer
updates the trajectory struct fields to include grade.

## Ledger

### AeGradeChangeLedgerEntry

```java
// io.casehub.clinical.ledger
@Entity
@Table(name = "ae_grade_change_ledger_entry")
public class AeGradeChangeLedgerEntry extends LedgerEntry {
    @Column(name = "previous_grade")
    public String previousGrade;

    @Column(name = "new_grade", nullable = false)
    public String newGrade;

    @Column(length = 500)
    public String reason;

    @Column(name = "changed_by")
    public String changedBy;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
            previousGrade != null ? previousGrade : "",
            newGrade,
            reason != null ? reason : "",
            changedBy != null ? changedBy : ""
        ).getBytes(StandardCharsets.UTF_8);
    }
}
```

Written by a new `AeGradeChangeLedgerWriter` service, called from
`regradeAdverseEvent()`. EU AI Act Art.12 `ComplianceSupplement` attached
via `ClinicalComplianceSupplement`.

## REST API

```
POST /api/patients/{patientId}/adverse-events/{aeId}/regrade

Request body:
{
    "grade": "GRADE_3",
    "reason": "Patient developed severe dehydration"
}

Response: 200 with updated AdverseEvent
```

```
GET /api/patients/{patientId}/adverse-events/{aeId}/grade-history

Response: 200 with List<AeGradeChange> ordered by changedAt
```

Both endpoints require `INVESTIGATOR` or `COORDINATOR` role
(`@RolesAllowed`). `changedBy` is derived from the authenticated
principal — not passed in the request body.

## Flyway Migrations

- `V127__ae_grade_change.sql` — default datasource
  ```sql
  CREATE TABLE ae_grade_change (
      id UUID PRIMARY KEY,
      adverse_event_id UUID NOT NULL REFERENCES adverse_event(id),
      previous_grade VARCHAR(20),
      new_grade VARCHAR(20) NOT NULL,
      changed_at TIMESTAMP WITH TIME ZONE NOT NULL,
      changed_by VARCHAR(255) NOT NULL,
      reason VARCHAR(500),
      CONSTRAINT fk_ae_grade_change_ae FOREIGN KEY (adverse_event_id)
          REFERENCES adverse_event(id)
  );
  CREATE INDEX idx_ae_grade_change_ae_id ON ae_grade_change(adverse_event_id);
  ```

- `V2030__ae_grade_change_ledger_entry.sql` — qhorus datasource
  ```sql
  CREATE TABLE ae_grade_change_ledger_entry (
      id UUID PRIMARY KEY REFERENCES ledger_entry(id),
      previous_grade VARCHAR(20),
      new_grade VARCHAR(20) NOT NULL,
      reason VARCHAR(500),
      changed_by VARCHAR(255)
  );
  ```

## Dashboard Integration

`TrialDashboardResource` adverse events list gains a `gradeHistory` field
showing the timeline. The existing `grade` field continues to show current
grade.

## Scope Exclusions

- **Grade trend analysis (slope/acceleration)**: deferred. The grade
  dimension in the trajectory time series enables this in a future issue,
  but computing and exposing grade velocity is not part of this issue.
- **Automatic grade inference from vitals/labs**: regrading is a human
  clinical judgment, not a system action.

## Testing Strategy

- **Unit:** `AeGradeChange` queries, `regradeAdverseEvent` logic (guard,
  SLA recalculation, direction detection), ledger writer
- **Integration:** full regrade flow with escalation re-evaluation (Grade
  1→3 upgrade triggers engine case), SUSAR re-evaluation, trajectory
  builder with grade dimension
- **Edge cases:** same-grade no-op, Grade 5→3 downgrade (no re-evaluation),
  Grade 2→3 upgrade (SLA tightens), Grade 3→4 upgrade (engine case already
  exists — no duplicate), regrade of AE with existing trajectory matches
- **Robustness:** regrade of nonexistent AE, regrade with invalid grade,
  concurrent regrade race
