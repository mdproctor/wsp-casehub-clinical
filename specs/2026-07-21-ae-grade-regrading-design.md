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
        return previousGrade == null || newGrade.ordinal() > previousGrade.ordinal();
    }

    public boolean isDowngrade() {
        return previousGrade != null && newGrade.ordinal() < previousGrade.ordinal();
    }
}
```

**Lifecycle note:** `AeGradeChangedEvent` is fired only from `regradeAdverseEvent()`
(after-commit), never from `reportAdverseEvent()`. Initial reports fire
`AdverseEventReportedEvent` via the existing path. The `previousGrade == null`
handling in `isUpgrade()`/`isDowngrade()` is defensive — the `AeGradeChange`
history entity records initial reports with `previousGrade=null` (D3), but
no event is fired for those. The `siteId` field is derived from
`PatientEnrollment.findById(ae.enrollmentId).siteId` — same lookup as
`reportAdverseEvent()` (see Service Layer below).

## Service Layer

### AdverseEventService changes

**New method: `regradeAdverseEvent`**

```
@Transactional
regradeAdverseEvent(UUID aeId, CtcaeGrade newGrade, String changedBy, String reason):
    1. Load AE (tenant-scoped)
    2. Guard: newGrade == ae.grade → no-op, return
    3. CtcaeGrade previousGrade = ae.grade
    4. Record: insert AeGradeChange(previousGrade, newGrade, now, changedBy, reason)
    5. Update: ae.grade = newGrade
    6. SLA (upgrade only, tighten-only):
       if newGrade.ordinal() > previousGrade.ordinal():
           Instant newDeadline = Instant.now().plus(newGrade.sla().orElseThrow())
           if newDeadline.isBefore(ae.slaDeadline):
               ae.slaDeadline = newDeadline
    7. Ledger: write AeGradeChangeLedgerEntry
    8. Derive siteId: PatientEnrollment enrollment = PatientEnrollment.findById(ae.enrollmentId)
       UUID siteId = enrollment != null ? enrollment.siteId : null
    9. After-commit: fire AeGradeChangedEvent(aeId, ae.enrollmentId, siteId,
       previousGrade, newGrade, now, changedBy, ae.tenantId)
       — same TransactionSynchronizationRegistry pattern as reportAdverseEvent()
```

**Design decision D4: SLA tighten-only.** On upgrade, the SLA deadline is
recalculated only if the new grade's SLA produces an earlier deadline than the
current one. On downgrade, the SLA is never relaxed — a Grade 3→1 downgrade
keeps the 24h SLA. Rationale: once a serious reporting obligation has been
identified (ICH E6(R3) §5.17), it cannot be withdrawn by a later downgrade.
The regulatory clock started at the point the serious grade was assigned.

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
`event.isUpgrade()`. Each listener delegates to the appropriate service via
a new public method — **no `AdverseEventReportedEvent` is fired** for
regrades. `AdverseEventReportedEvent` is observed by four production services
(`AeEscalationCaseService`, `RegulatorySubmissionCaseService`,
`SusarOversightCaseService`, `SafetyOfficerNotificationListener`) and firing
it from a regrade would cause double invocations and spurious notifications.

### Escalation re-evaluation

**Listener:** `AeGradeChangeEscalationListener` observes `@ObservesAsync
AeGradeChangedEvent`, gates on `event.isUpgrade()`.

If the new grade crosses the Grade 3 threshold (`previousGrade < GRADE_3`,
`newGrade >= GRADE_3`) and `ae.engineCaseId == null`:

1. **Cancel existing WorkItem** if `ae.workItemId != null` — the WorkItem was
   created at report time for Grade 1/2 AEs where `engineCaseRequired()` was
   false. An engine case now takes over SLA governance.
2. **Start escalation case** via a new public method
   `AeEscalationCaseService.startEscalationForRegrade(UUID aeId, UUID
   enrollmentId, UUID siteId, CtcaeGrade grade, String tenantId)` — follows
   the same three-phase pattern (prepare-and-mark, startCase().join(),
   persistCaseId) but accepts regrade context directly instead of
   `AdverseEventReportedEvent`.

If an engine case already exists (Grade 3→4 upgrade): no action needed — the
case is already active.

### SUSAR criteria re-evaluation

**Listener:** `AeGradeChangeSusarListener` observes `@ObservesAsync
AeGradeChangedEvent`, gates on `event.isUpgrade()`.

Delegates to a new public method
`SusarOversightCaseService.reevaluateForRegrade(UUID aeId, UUID siteId,
String tenantId)`. This method:
- Loads the AE internally
- Checks `ae.unexpected && ae.suspected` and grade threshold (Grade 4/5) via
  `SusarEvaluatorFunction` — same criteria as the initial report path
- Gates on `susarOversightStatus == NONE` (idempotency)
- Starts a SUSAR oversight case if criteria met

### Regulatory submission re-evaluation

**Listener:** `AeGradeChangeRegulatoryListener` observes `@ObservesAsync
AeGradeChangedEvent`, gates on `event.isUpgrade()`.

Delegates to a new public method
`RegulatorySubmissionCaseService.reevaluateForRegrade(UUID aeId, UUID siteId,
String tenantId)`. This method:
- Loads the AE internally
- Checks `isIndReportable(ae.grade)` (Grade 3+) AND `ae.unexpected` — per
  21 CFR 312.32, IND reporting requires unexpected AEs
- Gates on `regulatorySubmissionStatus == NONE` (idempotency)
- Starts a regulatory submission case if criteria met

### Trajectory alert re-evaluation

**Listener:** `AeGradeChangeTrajectoryListener` observes `@ObservesAsync
AeGradeChangedEvent` (both upgrades and downgrades — trajectory records all
grade changes).

Calls `AeTrajectoryAlertService.evaluate(event.aeId(), event.tenantId())`.
The method takes `(UUID aeId, String tenantId)` — it loads the AE from the
database internally, so the updated grade is picked up automatically.

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

The `ae-trajectory` schema already has `grade` as a top-level scalar feature.
The trajectory time series inner fields need `grade` added:

```java
FeatureField.timeSeries("aeTrajectory", "ts",
    new SimilaritySpec.DtwSpec(new WarpingConstraint.SakoeChibaBand(3)),
    new TrendSpec(Set.of(TrendType.SLOPE, TrendType.ACCELERATION, TrendType.CHANGE_POINTS), ChronoUnit.HOURS),
    FeatureField.numeric("ts", 0, 7776000),
    FeatureField.numeric("escalation", 0, 3),
    FeatureField.numeric("susar", 0, 3),
    FeatureField.numeric("regulatory", 0, 3),
    FeatureField.numeric("grade", 1, 5))     // NEW — grade ordinal (1-5)
```

The existing `TrendSpec(SLOPE, ACCELERATION, CHANGE_POINTS)` on the
`aeTrajectory` time series automatically computes trend enrichment for
all inner fields, including the new `grade` dimension. This yields
`aeTrajectory.grade.slope`, `aeTrajectory.grade.acceleration`, and
`aeTrajectory.grade.changePoints` — fulfilling issue #135's TrendAnalyzer
enrichment requirement with no additional code beyond adding the inner field.

Existing stored cases have no `grade` inner field — the CBR engine handles
missing fields gracefully (DTW skips dimensions not present in both cases).

## Ledger

### AeGradeChangeLedgerEntry

```java
// io.casehub.clinical.ledger
@Entity
@Table(name = "ae_grade_change_ledger_entry")
@DiscriminatorValue("AE_GRADE_CHANGE")
public class AeGradeChangeLedgerEntry extends JpaLedgerEntry {
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

Follows the existing `PatientResource` hierarchy at
`/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events`.

```
POST /trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/regrade

Request body:
{
    "grade": "GRADE_3",
    "reason": "Patient developed severe dehydration"
}

Response: 200 with updated AdverseEvent
```

```
GET /trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/grade-history

Response: 200 with List<AeGradeChange> ordered by changedAt
```

Both endpoints are added to `PatientResource` (same class as
`reportAdverseEvent` and `getAdverseEvent`). They require `INVESTIGATOR`
or `COORDINATOR` role (`@RolesAllowed`). `changedBy` is derived from the
authenticated `CurrentPrincipal` — not passed in the request body. Path
parameters `trialId`, `siteId`, and `enrollmentId` are validated against
the tenant-scoped entities, consistent with existing endpoint guards.

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

  -- Retroactive initial-grade entries for existing AEs (D3 uniformity)
  INSERT INTO ae_grade_change (id, adverse_event_id, previous_grade, new_grade, changed_at, changed_by, reason)
  SELECT gen_random_uuid(), id, NULL, grade, reported_at, 'migration', 'Retroactive initial grade entry'
  FROM adverse_event
  WHERE NOT EXISTS (
      SELECT 1 FROM ae_grade_change gc WHERE gc.adverse_event_id = adverse_event.id
  );
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

- **Grade trend dashboard/API exposure**: the TrendAnalyzer automatically
  computes `aeTrajectory.grade.slope`, `.acceleration`, and `.changePoints`
  once grade is added as a time series inner field. These values are
  available to the CBR engine for matching and alerting. Exposing grade
  velocity as standalone dashboard widgets or dedicated API endpoints is
  deferred to a future issue.
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
