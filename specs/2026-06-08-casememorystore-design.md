# CaseMemoryStore Integration — casehub-clinical

**Date:** 2026-06-08
**Issue:** casehubio/clinical#33 (closes), casehubio/clinical#69 (closes)
**Branch:** issue-33-casememorystore-integration
**Platform dependency:** casehubio/platform#79 (assertTenant async fix)
**Degradation tracking:** casehubio/clinical#70

---

## Problem

Every clinical trial case starts cold. A new AE escalation case for a patient
with prior Grade 3 hepatotoxicity starts without that history. A new deviation
review at a site with three recent reporting breaches starts without site
compliance context. CaseMemoryStore closes this gap by surfacing patient and
site history before the first agent runs, and accumulating outcome facts across
cases.

---

## Scope

This spec covers:
1. **Multi-tenancy foundation** — `tenantId` on all domain entities and CDI events (closes #69)
2. **Four memory domains** — patient, site, drug, IRB
3. **`ClinicalMemoryService`** — central write/read facade
4. **Context value objects** — `ClinicalPatientContext`, `ClinicalSiteContext`
5. **Integration points** — where facts are written and recalled
6. **Testing strategy**

---

## 1. Multi-tenancy Foundation (closes #69)

### Entity changes

Add `String tenantId` to all six domain entities:

```
ClinicalTrial, TrialSite, PatientEnrollment,
ProtocolDeviation, AdverseEvent, IrbApproval
```

Flyway migrations in `db/migration/default/` (V111–V116), one per table:

```sql
ALTER TABLE clinical_trial ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
-- repeated for trial_site, patient_enrollment, protocol_deviation, adverse_event, irb_approval
```

`DEFAULT 'default'` ensures safe migration on existing rows. Production deployments
backfill from `CurrentPrincipal` before the default is removed.

### Event changes

All five clinical CDI events in `api/` gain `String tenantId()`:

| Event | Emitting service | tenantId source |
|---|---|---|
| `AdverseEventReportedEvent` | `AdverseEventService` | `principal.tenancyId()` — sync ✅ |
| `AeEscalationCompletedEvent` | `AeEscalationListener` | case context `"tenantId"` key (set by `AeEscalationCaseService`) |
| `IrbApprovalResolvedEvent` | `IrbDecisionListener` | `approval.tenantId` — entity field |
| `ProtocolDeviationResolvedEvent` | `PiResponseListener` | `deviation.tenantId` — entity field |

`IrbDecisionListener` and `PiResponseListener` load the entity before constructing
the event — the tenantId field adds a single field read, no additional DB query.

`AeEscalationListener` does not load `AdverseEvent` from DB. Instead,
`AeEscalationCaseService.prepareAndMarkRequested()` adds `ctx.put("tenantId", ae.tenantId)`
to the case context (alongside `aeId`, `enrollmentId`, etc.) so `AeEscalationListener`
can read it from `instance.getCaseContext().getPath("tenantId")`.

`SponsorNotification` is out of scope — no memory integration path.

---

## 2. Memory Domains

New class `ClinicalMemoryDomains` in package `io.casehub.clinical.memory`.

| Constant | Domain name | entityId convention | What is stored |
|---|---|---|---|
| `PATIENT` | `clinical-patient` | `patient:{enrollmentId}` | AE report facts (grade, site), AE outcome facts (safety review, DSMB escalation) |
| `SITE` | `clinical-site` | `site:{siteId}` | AE report events, deviation reports, PI decision outcomes, timeline breach signals |
| `DRUG` | `clinical-drug` | `protocol:{protocolId}` | Cross-site AE grade/outcome signals per protocol |
| `IRB` | `clinical-irb` | `deviation-type:{deviationType}` | IRB decisions as precedent (APPROVED/REJECTED/EXPIRED per deviation type) |

`DRUG` and `IRB` domains are write-only in this release — no recall path wired
into engine context yet. Facts accumulate for future agent queries or YAML
binding extensions.

---

## 3. `ClinicalMemoryService`

**Package:** `io.casehub.clinical.memory`
**Scope:** `@ApplicationScoped`

No caller in clinical touches `CaseMemoryStore` directly. All domain semantics
live in this class. Follows aml's `AmlMemoryService` contract exactly:

- All writes wrapped in `try/catch` — failures log WARN, never propagate
- All queries return the context object (populated or `empty()`) — never throw
- `tenantId` is always an explicit parameter; the service does not inject `CurrentPrincipal`
- Constructor injection for `CaseMemoryStore` and `PreferenceProvider`

### Write methods

```java
void storeAeReport(UUID aeId, UUID enrollmentId, UUID siteId,
                   CtcaeGrade grade, String tenantId)

void storeAeOutcome(UUID aeId, UUID enrollmentId, CtcaeGrade grade,
                    String safetyReview, boolean dsmbEscalated, String tenantId)

void storeDeviationReport(UUID deviationId, UUID siteId,
                          String deviationType, DeviationSeverity severity, String tenantId)

void storePiDecision(UUID deviationId, UUID siteId,
                     String deviationType, PiApprovalStatus status, String tenantId)

void storeIrbDecision(UUID deviationId, String deviationType,
                      IrbDecision decision, String tenantId)

void storeDrugAeSignal(String protocolId, CtcaeGrade grade,
                       String outcome, String tenantId)
```

### Query methods

```java
ClinicalPatientContext queryPatientContext(UUID enrollmentId, String tenantId)
ClinicalSiteContext    querySiteContext(UUID siteId, String tenantId)
```

### Attribute conventions

All facts use standard `MemoryAttributeKeys`:
- `ACTOR_ID` → `"clinical-service"`
- `OUTCOME` → domain-specific string (e.g. `"GRADE_3"`, `"APPROVED"`, `"TIMELINE_BREACH"`)
- `CONFIDENCE` → `"1.0000"` for deterministic facts (AE report); outcome-weighted for results

---

## 4. Context Value Objects

### `ClinicalPatientContext`

```java
public record ClinicalPatientContext(List<Memory> aeHistory) {
    public static ClinicalPatientContext empty()
    public boolean hasHistory()
    public boolean hasPriorGrade3OrAbove()  // outcome attr grade >= GRADE_3
    public boolean hasPriorEscalation()     // outcome attr == "ESCALATED" | "DSMB_ESCALATED"
    public Map<String, Object> toContextMap()
}
```

`toContextMap()` shape injected into engine `initialContext` under key `patientContext`:

```json
{
  "hasHistory": true,
  "hasPriorGrade3OrAbove": true,
  "hasPriorEscalation": false,
  "aeCount": 2,
  "facts": [
    { "grade": "GRADE_3", "outcome": "RESOLVED", "createdAt": "2026-04-12T..." }
  ]
}
```

YAML bindings in `AeEscalationCaseDefinition` can key off
`.patientContext.hasPriorGrade3OrAbove` to tighten monitoring intensity.

### `ClinicalSiteContext`

```java
public record ClinicalSiteContext(List<Memory> complianceEvents) {
    public static ClinicalSiteContext empty()
    public boolean hasComplianceIssues()
    public int recentTimelineBreachCount()   // outcome == "TIMELINE_BREACH", last 6 months
    public Map<String, Object> toContextMap()
}
```

`toContextMap()` shape injected under key `siteContext`:

```json
{
  "hasComplianceIssues": true,
  "recentTimelineBreachCount": 3,
  "facts": [...]
}
```

Deviation review case YAML can key off `.siteContext.recentTimelineBreachCount >= 3`
to lower the DSMB notification threshold for repeat-offending sites.

Both return `empty()` on any adapter failure. No case is ever blocked by a
memory read failure.

---

## 5. Integration Points

### Write — sync paths (work on merge)

| Service | Method called | Domains written |
|---|---|---|
| `AdverseEventService.reportAdverseEvent()` | `storeAeReport(...)` | `PATIENT`, `SITE` |
| `ProtocolDeviationService.reportDeviation()` | `storeDeviationReport(...)` | `SITE` |

### Write — async paths (degrade to WARN until platform#79)

| Listener | Method called | Domains written |
|---|---|---|
| `AeEscalationListener.onCaseLifecycle()` | `storeAeOutcome(...)` + `storeDrugAeSignal(...)` | `PATIENT`, `DRUG` |
| `PiResponseListener.process()` | `storePiDecision(...)` | `SITE` |
| `IrbDecisionListener.onWorkItemLifecycle()` | `storeIrbDecision(...)` | `IRB` |

All async writes are wrapped in `ClinicalMemoryService`'s try/catch. When
platform#79 ships, writes activate automatically — no clinical code change needed.

### Recall — `AeEscalationCaseService.prepareAndMarkRequested()`

Two additions inside the existing `@Transactional` method:

1. **`protocolId` in case context** — load `PatientEnrollment → TrialSite → ClinicalTrial`
   to derive `trial.protocolId`; add as `ctx.put("protocolId", trial.protocolId)`.
   Used by `AeEscalationListener` at completion time for `storeDrugAeSignal`.

2. **Prior context injection** — after loading `ae` (which now has `tenantId`):
```java
ctx.put("tenantId", ae.tenantId);   // for AeEscalationListener to read later
ClinicalPatientContext patientCtx =
    memoryService.queryPatientContext(ae.enrollmentId, ae.tenantId);
ClinicalSiteContext siteCtx =
    memoryService.querySiteContext(siteId, ae.tenantId);
ctx.put("patientContext", patientCtx.toContextMap());
ctx.put("siteContext",    siteCtx.toContextMap());
```

Until platform#79 ships, both context maps are always `empty()` — the case runs
exactly as it does today. When the fix ships, historical enrichment activates
automatically.

### `AeEscalationListener` addition

After `ledgerWriter.writeCompletionEntry(...)`, before `completedEvents.fireAsync(...)`.
Both `tenantId` and `protocolId` are read from the case context (set by `AeEscalationCaseService`):

```java
String tenantId  = resolveString(instance.getCaseContext().getPath("tenantId"));
String protocolId = resolveString(instance.getCaseContext().getPath("protocolId"));
if (tenantId != null) {
    memoryService.storeAeOutcome(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, tenantId);
    if (protocolId != null) {
        memoryService.storeDrugAeSignal(protocolId, grade, safetyReviewOutcome, tenantId);
    }
}
```

`resolveString` is a private null-safe helper (same pattern as existing `resolveUuid`).
`storeDrugAeSignal` receives the raw `protocolId` string; `ClinicalMemoryService` constructs
the entityId as `"protocol:" + protocolId` internally.

### Maven dependencies

```xml
<!-- runtime/pom.xml -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-memory-jpa</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-memory-inmem</artifactId>
    <scope>test</scope>
</dependency>
```

Production `application.properties` — add `memory-jpa` to `selected-alternatives`
and index-dependency entries (same pattern as `casehub-engine` indexing).

Test `application.properties` — add `memory-inmem` to `selected-alternatives`.

---

## 6. New Files

```
runtime/src/main/java/io/casehub/clinical/memory/
    ClinicalMemoryDomains.java
    ClinicalMemoryService.java
    ClinicalPatientContext.java
    ClinicalSiteContext.java

runtime/src/main/resources/db/migration/default/
    V111__add_tenant_id_clinical_trial.sql
    V112__add_tenant_id_trial_site.sql
    V113__add_tenant_id_patient_enrollment.sql
    V114__add_tenant_id_protocol_deviation.sql
    V115__add_tenant_id_adverse_event.sql
    V116__add_tenant_id_irb_approval.sql
```

---

## 7. Modified Files

```
api/src/main/java/io/casehub/clinical/api/
    AdverseEventReportedEvent.java         — + tenantId()
    AeEscalationCompletedEvent.java        — + tenantId()
    IrbApprovalResolvedEvent.java          — + tenantId()
    ProtocolDeviationResolvedEvent.java    — + tenantId()

runtime/src/main/java/io/casehub/clinical/entity/
    ClinicalTrial.java                     — + String tenantId
    TrialSite.java                         — + String tenantId
    PatientEnrollment.java                 — + String tenantId
    ProtocolDeviation.java                 — + String tenantId
    AdverseEvent.java                      — + String tenantId
    IrbApproval.java                       — + String tenantId

runtime/src/main/java/io/casehub/clinical/service/
    AdverseEventService.java               — inject ClinicalMemoryService; storeAeReport
    ProtocolDeviationService.java          — inject ClinicalMemoryService; storeDeviationReport
    AeEscalationCaseService.java           — protocolId in ctx; queryPatientContext + querySiteContext
    AeEscalationListener.java             — storeAeOutcome + storeDrugAeSignal; stamp ae.tenantId on event
    PiResponseListener.java               — storePiDecision; stamp deviation.tenantId on event
    IrbDecisionListener.java              — storeIrbDecision; stamp approval.tenantId on event

runtime/src/main/resources/application.properties
runtime/src/test/resources/application.properties
runtime/pom.xml
```

---

## 8. Testing

| Test class | Type | Approach |
|---|---|---|
| `ClinicalMemoryServiceTest` | JUnit 5 unit | `InMemoryMemoryStore`; verify text, entityId, domain, attributes; verify swallowing |
| `ClinicalPatientContextTest` | JUnit 5 unit | Grade boundary (`GRADE_2` → false, `GRADE_3` → true); `empty()` shape |
| `ClinicalSiteContextTest` | JUnit 5 unit | `recentTimelineBreachCount()` 6-month time window |
| `AeEscalationListenerMemoryTest` | `@QuarkusTest` | `@InjectMock ClinicalMemoryService`; verify `storeAeOutcome` args |
| `IrbDecisionListenerMemoryTest` | `@QuarkusTest` | `@InjectMock ClinicalMemoryService`; APPROVED / REJECTED / EXPIRED |
| `PiResponseListenerMemoryTest` | `@QuarkusTest` | `@InjectMock ClinicalMemoryService`; DONE / DECLINE |
| `AeEscalationContextInjectionTest` | `@QuarkusTest` | Seed store; call `prepareAndMarkRequested()`; assert `initialContext` shape |

All `@InjectMock` beans stub methods in `@BeforeEach` per GE-20260604-4298f9
(stub or face null-return side effects across test methods).

---

## Open Issues Filed

- **casehubio/platform#79** — `assertTenant` async fix (blocking for full async write path)
- **casehubio/clinical#70** — degradation tracking until platform#79 ships
