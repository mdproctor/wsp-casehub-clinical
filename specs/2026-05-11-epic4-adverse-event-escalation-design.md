# Design Spec — Epic 4: Adverse Event Escalation

**Date:** 2026-05-11  
**Issue:** casehubio/clinical#4  
**Approach:** B — WorkItem + SLA + `AdverseEventLedgerEntry`  
**Deferred:** casehub-connectors safety officer notification → casehubio/clinical#11

---

## Problem

GCP ICH E6(R3) §5.17 requires serious adverse events (Grade ≥ 3) to be reported to the
sponsor within 24 hours. Grade 5 (death) uses a stricter 1-hour internal SLA. Non-serious
AEs (Grade 1-2) carry a 7-day reporting window. ClinicalAgent has no deadline tracking
whatsoever — this single feature delivers more regulatory value than ClinicalAgent's entire
codebase.

The platform primitive that enforces deadlines is the `casehub-work` WorkItem
`claimDeadline`. Escalation policy (what happens when the deadline is missed) is runtime
configuration via `EscalationPolicy` SPI — not clinical code. Clinical's job is to set the
right deadline and write a tamper-evident ledger record.

---

## Domain Model Changes

### `CtcaeGrade` — fix 7-day SLA for Grade 1-2 (`api` module)

Grade 1 and 2 currently carry `null` SLA — incorrect. GCP requires non-serious AEs to be
reported within 7 days. Updated:

| Grade | Label | SLA |
|-------|-------|-----|
| GRADE_1 | Mild | 7 days |
| GRADE_2 | Moderate | 7 days |
| GRADE_3 | Severe | 24 hours |
| GRADE_4 | Life-threatening | 24 hours |
| GRADE_5 | Death | 1 hour (internal policy, stricter than GCP minimum) |

`grade.sla()` now returns a non-empty `Optional<Duration>` for all grades — no
special-casing needed in service code.

### `AdverseEvent` entity — add `workItemId` (`runtime` module)

New nullable column `UUID workItemId`. Set by `AdverseEventService` when the WorkItem is
created; null until reporting occurs. Provides a stable domain → platform link without
querying casehub-work.

**Flyway V7:** adds `work_item_id UUID` to `adverse_event` table.

### `AdverseEventLedgerEntry` — new subclass (`runtime` module)

Domain-specific ledger entry following the platform JOINED inheritance pattern. Flyway V1004
creates the join table (namespace: V1004+ reserved for ledger subclass joins per
PLATFORM.md).

```java
@Entity
@Table(name = "ae_ledger_entry")
@DiscriminatorValue("ADVERSE_EVENT")
public class AdverseEventLedgerEntry extends LedgerEntry {
    public UUID adverseEventId;
    public UUID enrollmentId;
    public String ctcaeGrade;    // e.g. "GRADE_3"
    public Instant reportedAt;
    public Instant slaDeadline;
}
```

---

## New Dependencies (`runtime/pom.xml`)

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ledger</artifactId>
</dependency>
```

Both version-managed by parent BOM. Same pattern as casehub-aml.

---

## Service Layer

### `AdverseEventService` — new `@ApplicationScoped` service

Single public method: `reportAdverseEvent(AdverseEvent ae)`. Executes in one transaction:

1. Set `ae.reportedAt = Instant.now()` (server-side; client value ignored — reportedAt anchors the regulatory SLA clock)
2. Set `ae.slaDeadline = ae.reportedAt + ae.grade.sla()`
3. Create WorkItem via embedded casehub-work `WorkItemService`:
   - `claimDeadline = ae.slaDeadline`
   - `category = "adverse-event"`
   - `candidateGroups` = grade-appropriate routing group (`"safety-officers"` for Grade 1-2; `"dsmb,safety-officers"` for Grade ≥ 3)
   - `payload` = JSON with `enrollmentId`, `grade`, `occurredAt`
4. Write `AdverseEventLedgerEntry` with all audit fields
5. Set `ae.workItemId` from created WorkItem id and persist

The REST resource calls this service — it does not persist `AdverseEvent` directly.

---

## REST Layer

**Endpoint:** `POST /trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events`

The route already exists. Change: delegate to `AdverseEventService` instead of direct entity persist.

**Request body:**
```json
{
  "grade": "GRADE_3",
  "actuality": "ACTUAL",
  "occurredAt": "2026-05-11T08:00:00Z",
  "description": "Grade 3 neutropenia following cycle 2"
}
```

`reportedAt` is set server-side. `slaDeadline` is computed server-side. Client must not supply either.

**Response:** `201 Created` with `Location` header and body including `workItemId`.

No new endpoints required.

---

## Testing

### Unit tests (`api` module)

- `CtcaeGradeTest` — all 5 grades return correct SLA; `sla()` is non-empty for all grades
- SLA arithmetic: `reportedAt + grade.sla()` is correct for each grade

### Integration tests (`runtime`, `@QuarkusTest` with H2)

- `AdverseEventServiceTest` — report AE for each grade; verify `workItemId` set, `slaDeadline` correct, `AdverseEventLedgerEntry` persisted with correct fields
- `AdverseEventResourceTest` — POST via REST; verify 201 + Location + `workItemId` in response; verify DB state

### Correctness tests

- Grade 3 `slaDeadline` = `reportedAt + 24h` exactly
- Grade 5 `slaDeadline` = `reportedAt + 1h` (not 24h — stricter policy applies)
- `reportedAt` is server-set — client-supplied value is ignored
- All 5 grades produce non-null `slaDeadline`

### Robustness tests

- Non-existent enrollment → 404
- Missing required fields (`grade`, `occurredAt`) → 400
- `AdverseEventService` is transactional — ledger write failure rolls back WorkItem creation; no partial state

### End-to-end (showcase extension)

Extend `ShowcaseScenarioTest`:
- Enroll patient at Site A → report Grade 3 AE → verify `workItemId` set and `slaDeadline` = `reportedAt + 24h`
- Report Grade 5 AE → verify `slaDeadline` = `reportedAt + 1h`

---

## Escalation Policy

DSMB escalation on SLA miss is **runtime configuration** via casehub-work `EscalationPolicy`
SPI — not clinical code. Clinical sets `candidateGroups` so the platform knows the routing
pool; the deployer configures what happens when `claimDeadline` passes. Different trial types
or phases can carry different escalation configs.

Safety officer notification via casehub-connectors is deferred to casehubio/clinical#11,
which lists its prerequisites.

---

## Platform Protocol Checks

- **Capability ownership:** WorkItem SLA → casehub-work ✅; ledger audit → casehub-ledger ✅; escalation policy → casehub-work SPI ✅
- **Boundary rule:** domain logic (CTCAE grade policy, `candidateGroups` mapping) stays in clinical; no platform repos touched
- **Flyway numbering:** domain migrations V7 (adverse_event update); ledger subclass V1004 ✅
- **Persistence module split:** `AdverseEventLedgerEntry` in `runtime` (JPA module) not in `api` ✅
- **No-POJO rule (clinical exception):** Active Record entities remain the domain objects per CLAUDE.md