# CaseMemoryStore Integration — casehub-clinical

**Date:** 2026-06-08 (revised 2026-06-09, 2026-06-09b)
**Issue:** casehubio/clinical#33 (closes), casehubio/clinical#69 (closes)
**Branch:** issue-33-casememorystore-integration
**Platform dependency:** casehubio/platform#79 (assertTenant async fix)
**Degradation tracking:** casehubio/clinical#70
**Deferred:** casehubio/clinical#71 (query isolation), casehubio/clinical#72 (DRUG domain), casehubio/clinical#73 (IRB domain)

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
1. **Multi-tenancy foundation** — `tenantId` on all six domain entities, CDI events, and entity creation paths (closes #69)
2. **Two memory domains** — patient, site (DRUG and IRB deferred: #72, #73)
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

Single Flyway migration V116 in `db/migration/default/`:

```sql
-- V116: Multi-tenancy foundation — tenant_id column on all six domain entities.
-- DEFAULT 'default' ensures safe migration on existing rows.
-- Query isolation (filtering reads by tenant_id) is tracked in casehubio/clinical#71.
-- Note: sponsor_notification.tenant_id already exists from V115 (nullable VARCHAR(64)).
ALTER TABLE clinical_trial     ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE trial_site         ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE patient_enrollment ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE protocol_deviation ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE adverse_event      ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE irb_approval       ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
```

Production deployments backfill from `CurrentPrincipal` before the default is removed.
Query isolation (every Panache read scoped to tenant) is tracked separately in #71 and
not part of this spec.

### Entity creation sites — tenantId population

All seven entity creation sites must set `tenantId` at persist time:

| Site | Entity | tenantId source |
|---|---|---|
| `TrialResource.register()` | `ClinicalTrial` | inject `CurrentPrincipal`; `trial.tenantId = principal.tenancyId()` |
| `SiteResource.add()` | `TrialSite` | inject `CurrentPrincipal`; `site.tenantId = principal.tenancyId()` |
| `PatientResource.enroll()` | `PatientEnrollment` | inject `CurrentPrincipal`; `enrollment.tenantId = principal.tenancyId()` |
| `DeviationResource.reportDeviation()` | `ProtocolDeviation` | inject `CurrentPrincipal`; `deviation.tenantId = principal.tenancyId()` |
| `AdverseEventService.reportAdverseEvent()` | `AdverseEvent` | inject `CurrentPrincipal`; `ae.tenantId = principal.tenancyId()` before `ae.persist()` |
| `IrbDeviationCaseService.prepareAndCreateApproval()` | `IrbApproval` | `approval.tenantId = event.tenantId()` (after `ProtocolDeviationResolvedEvent` gains `tenantId`) |
| `SponsorNotificationStore.createPending()` | `SponsorNotification` | `n.tenantId = req.tenantId()` (after `SponsorNotificationRequest` gains `tenantId`) |

`SponsorNotification.tenantId` already exists from V115 (nullable). `SponsorNotificationStore.createPending()`
currently leaves it null. Population path: `SponsorNotificationRequest` gains `String tenantId`;
`SponsorNotificationListener` passes `event.tenantId()`; `SponsorNotificationStore` sets `n.tenantId = req.tenantId()`.

### CDI event changes

Three of the five clinical CDI events in `api/` gain `String tenantId()`:

| Event | Emitting service | tenantId source |
|---|---|---|
| `AdverseEventReportedEvent` | `AdverseEventService` | `principal.tenancyId()` — injected `CurrentPrincipal`, sync ✅ |
| `IrbApprovalResolvedEvent` | `IrbDecisionListener` | `approval.tenantId` — entity field |
| `ProtocolDeviationResolvedEvent` | `PiResponseListener`, `DeviationExpirer` | `deviation.tenantId` — entity field |

`AeEscalationCompletedEvent` does **not** gain `tenantId`. The memory write for AE outcomes
(`storeAeOutcome`) happens in `AeEscalationListener` before the event fires. The event's
only production consumer (`TrialSafetySignalService`) does not need tenantId.

`IrbDecisionListener` and `PiResponseListener` load the entity before constructing the event —
the `tenantId` field adds a single field read, no additional DB query.

**Blast radius — `AdverseEventReportedEvent` constructor change:**

The record gains `String tenantId`. CDI observer method signatures are unaffected — only
construction sites break.

*Emitter (must update constructor call):*
- `AdverseEventService.reportAdverseEvent()` — stamps `principal.tenancyId()`

*Observers (no change — not construction sites):*
- `AeEscalationCaseService.onAdverseEventReported()` — reads `ae.tenantId` from entity in `prepareAndMarkRequested()`; does not use `event.tenantId()`
- `SafetyOfficerNotificationListener.onAeReported()` — does not use `tenantId`

*Test files with construction sites (all must add tenantId argument):*
- `AeEscalationLifecycleTest` — 1 factory method (`aeEvent()`)
- `SafetyOfficerNotificationIntegrationTest` — 3 construction sites
- `SafetyOfficerNotificationListenerTest` — 10+ construction sites (factory + inline)
- `DsmbRollupTest` — 1 factory method

**Blast radius — `IrbApprovalResolvedEvent` constructor change:**

The record gains `String tenantId`. CDI observer method signatures are unaffected — only
construction sites break.

*Emitter (must update constructor call):*
- `IrbDecisionListener.onWorkItemLifecycle()` — stamps `approval.tenantId`

*Observers (no change — not construction sites):*
- No production consumers outside the emitter.

*Test files with construction sites:*
- `IrbDecisionListenerTest` — 1 construction site

**Blast radius — `ProtocolDeviationResolvedEvent` constructor change:**

The record gains `String tenantId`. CDI observer method signatures are unaffected — only
construction sites break.

*Emitters (must add `deviation.tenantId` or `d.tenantId`):*
- `PiResponseListener.process()` — DONE / DECLINE paths
- `DeviationExpirer.expireOne()` — EXPIRED path

*Observers (no compile break — not construction sites; implementation changes to read new field):*
- `IrbDeviationCaseService.onDeviationResolved()` — reads `event.tenantId()` to set `approval.tenantId`
- `SponsorNotificationListener.onDeviationResolved()` — reads `event.tenantId()` to populate `SponsorNotificationRequest`

*Test files with construction sites (all must add tenantId argument):*
- `PiResponseListenerMemoryTest` — 2 construction sites (new test, see §8)
- `IrbCommitteePolicySpiTest` — 2 construction sites (`criticalDeviationApproved()` factory)
- `IrbGateLifecycleTest` — 2 construction sites (`criticalDeviationApproved()` factory)
- `IrbDeviationCaseServiceTest` — 3 construction sites (non-irb, pi-rejected, helper)
- `SponsorNotificationListenerTest` — 8+ construction sites across all test methods

---

## 2. Memory Domains

New class `ClinicalMemoryDomains` in package `io.casehub.clinical.memory`.

| Constant | Domain name | entityId convention | What is stored |
|---|---|---|---|
| `PATIENT` | `clinical-patient` | `patient:{enrollmentId}` | AE report facts (grade, site); AE outcome facts (safety review, DSMB escalation) |
| `SITE` | `clinical-site` | `site:{siteId}` | AE report events, deviation reports, PI decision outcomes (APPROVED / REJECTED / TIMELINE_BREACH) |

DRUG domain (`clinical-drug`) deferred to #72 — entityId convention, recall path design, and
cross-tenant pharmacovigilance tradeoff documented as spec-ready questions in that issue.

IRB domain (`clinical-irb`) deferred to #73 — `IrbApproval.deviationType` gap
(extra DB query vs. schema addition vs. entityId change) documented as the design question.

---

## 3. `ClinicalMemoryService`

**Package:** `io.casehub.clinical.memory`
**Scope:** `@ApplicationScoped`

No caller in clinical touches `CaseMemoryStore` directly. All domain semantics
live in this class.

- All writes wrapped in `try/catch` — failures log WARN, never propagate
- All queries return the context object (populated or `empty()`) — never throw
- `tenantId` is always an explicit parameter; the service does not inject `CurrentPrincipal`
- Constructor injection for `CaseMemoryStore` only

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
```

`storePiDecision` maps `PiApprovalStatus.EXPIRED` to outcome `"TIMELINE_BREACH"`;
`APPROVED` / `REJECTED` / `ESCALATED` to their name strings.

### Query methods

```java
ClinicalPatientContext queryPatientContext(UUID enrollmentId, String tenantId)
ClinicalSiteContext    querySiteContext(UUID siteId, String tenantId)
```

### Attribute conventions

All facts use `MemoryAttributeKeys` (`io.casehub.platform.api.memory.MemoryAttributeKeys`):
- `ACTOR_ID` → `"clinical-service"`
- `OUTCOME` → domain-specific string (e.g. `"GRADE_3"`, `"APPROVED"`, `"TIMELINE_BREACH"`)

CONFIDENCE is not set — it applies to probabilistic agent outputs, not deterministic domain facts.

---

## 4. Context Value Objects

### `ClinicalPatientContext`

```java
public record ClinicalPatientContext(List<Memory> aeHistory) {
    public static ClinicalPatientContext empty()
    public boolean hasHistory()
    public boolean hasPriorGrade3OrAbove()  // any memory with outcome attr >= GRADE_3
    public boolean hasPriorEscalation()     // any memory with outcome attr == "ESCALATED" | "DSMB_ESCALATED"
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
    public int recentTimelineBreachCount()   // count entries with outcome == "TIMELINE_BREACH"
    public Map<String, Object> toContextMap()
}
```

`complianceEvents` is already time-filtered by the store query (`withSince` 180 days) —
`recentTimelineBreachCount()` simply counts entries with `OUTCOME == "TIMELINE_BREACH"`;
no additional Java-side date arithmetic.

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

### Write — request-context paths (work on merge)

These run on an HTTP request thread with an active CDI request scope.

| Service | Method called | Domains written |
|---|---|---|
| `AdverseEventService.reportAdverseEvent()` | `storeAeReport(...)` | `PATIENT`, `SITE` |
| `ProtocolDeviationService.reportDeviation()` | `storeDeviationReport(...)` | `SITE` |

### Write — non-request-context paths (degrade to WARN until platform#79)

These run without an active CDI request scope — Quartz scheduler threads and CDI async
executor threads both lack request scope. `CaseMemoryStore` adapters call
`MemoryPermissions.assertTenant(input.tenantId(), principal)` where `principal` is a
`@RequestScoped CurrentPrincipal` proxy; without request scope, this throws
`SecurityException`. `ClinicalMemoryService.try/catch` absorbs it and logs WARN.
tenantId is sourced from entities and case context, not from `CurrentPrincipal`.
When platform#79 ships, all these writes activate automatically — no clinical code change.

| Service | Method called | Domains | tenantId source |
|---|---|---|---|
| `DeviationExpirer.expireOne()` | `storePiDecision(..., EXPIRED)` | `SITE` (outcome: `"TIMELINE_BREACH"`) | `d.tenantId` — entity field |
| `AeEscalationListener.onCaseLifecycle()` | `storeAeOutcome(...)` | `PATIENT` | case context `"tenantId"` key |
| `PiResponseListener.process()` | `storePiDecision(...)` | `SITE` | `deviation.tenantId` — entity field |

### Recall — `AeEscalationCaseService.prepareAndMarkRequested()`

Two additions inside the existing `@Transactional` method:

1. **`tenantId` in case context** — after loading `ae` (which now has `tenantId`):
   `ctx.put("tenantId", ae.tenantId)` — for `AeEscalationListener` to read at completion time.

2. **Prior context injection** — after loading `ae`:
```java
ctx.put("tenantId", ae.tenantId);
ClinicalPatientContext patientCtx =
    memoryService.queryPatientContext(ae.enrollmentId, ae.tenantId);
ClinicalSiteContext siteCtx =
    memoryService.querySiteContext(siteId, ae.tenantId);
ctx.put("patientContext", patientCtx.toContextMap());
ctx.put("siteContext",    siteCtx.toContextMap());
```

Until platform#79 ships, both context maps are always `empty()` — non-request-context
writes that populate the store are absorbed as WARN. The case runs exactly as today.

### `AeEscalationListener` addition

After `ledgerWriter.writeCompletionEntry(...)`, before `completedEvents.fireAsync(...)`.
`tenantId` is read from the case context (set by `AeEscalationCaseService`):

```java
String tenantId = resolveString(instance.getCaseContext().getPath("tenantId"));
if (tenantId != null) {
    memoryService.storeAeOutcome(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, tenantId);
}
```

`resolveString` is a private null-safe helper (same pattern as existing `resolveUuid`).

### `querySiteContext` query construction

```java
MemoryQuery query = MemoryQuery.forEntity("site:" + siteId, ClinicalMemoryDomains.SITE, tenantId)
    .withSince(Instant.now().minus(180, ChronoUnit.DAYS))
    .withLimit(50);
List<Memory> events = store.query(query);
return new ClinicalSiteContext(events);
```

The 180-day window and limit-50 are applied in the store query, not in Java post-processing.
`ClinicalSiteContext.recentTimelineBreachCount()` counts directly from the returned list.

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

**CDI wiring:**

`io.casehub.platform.memory.NoOpCaseMemoryStore` is `@DefaultBean @ApplicationScoped`.
`io.casehub.platform.memory.jpa.JpaMemoryStore` is `@ApplicationScoped` (no `@DefaultBean`) —
it displaces `NoOpCaseMemoryStore` by classpath presence alone. **No `selected-alternatives`
entry is needed in production `application.properties`.**

`io.casehub.platform.memory.inmem.InMemoryMemoryStore` is `@Alternative @Priority(10)` —
it requires a `selected-alternatives` entry in test `application.properties`:
```properties
quarkus.arc.selected-alternatives=...,io.casehub.platform.memory.inmem.InMemoryMemoryStore
```

Add index-dependency entries for `casehub-platform-memory-jpa` in production
`application.properties` (same pattern as `casehub-engine` indexing).

---

## 6. New Files

```
runtime/src/main/java/io/casehub/clinical/memory/
    ClinicalMemoryDomains.java
    ClinicalMemoryService.java
    ClinicalPatientContext.java
    ClinicalSiteContext.java

runtime/src/main/resources/db/migration/default/
    V116__add_tenant_id_domain_entities.sql
```

---

## 7. Modified Files

### api/ module

```
api/src/main/java/io/casehub/clinical/api/
    AdverseEventReportedEvent.java         — + tenantId()
    IrbApprovalResolvedEvent.java          — + tenantId()
    ProtocolDeviationResolvedEvent.java    — + tenantId()
    SponsorNotificationRequest.java        — + tenantId()
```

### runtime/ domain entities

```
runtime/src/main/java/io/casehub/clinical/entity/
    ClinicalTrial.java          — + String tenantId
    TrialSite.java              — + String tenantId
    PatientEnrollment.java      — + String tenantId
    ProtocolDeviation.java      — + String tenantId
    AdverseEvent.java           — + String tenantId
    IrbApproval.java            — + String tenantId
```

### runtime/ REST resources (CurrentPrincipal injection + entity.tenantId population)

```
runtime/src/main/java/io/casehub/clinical/resource/
    TrialResource.java          — inject CurrentPrincipal; trial.tenantId = principal.tenancyId()
    SiteResource.java           — inject CurrentPrincipal; site.tenantId = principal.tenancyId()
    PatientResource.java        — inject CurrentPrincipal; enrollment.tenantId = principal.tenancyId()
    DeviationResource.java      — inject CurrentPrincipal; deviation.tenantId = principal.tenancyId()
```

### runtime/ services

```
runtime/src/main/java/io/casehub/clinical/service/
    AdverseEventService.java
        — inject CurrentPrincipal; ae.tenantId = principal.tenancyId() before ae.persist()
        — inject ClinicalMemoryService; storeAeReport(ae.id, ae.enrollmentId, siteId, ae.grade, ae.tenantId)
        — include ae.tenantId in AdverseEventReportedEvent constructor

    ProtocolDeviationService.java
        — inject ClinicalMemoryService; storeDeviationReport(deviation.id, deviation.siteId,
          deviation.deviationType, deviation.severity, deviation.tenantId)

    DeviationExpirer.java
        — inject ClinicalMemoryService
        — in expireOne(): storePiDecision(d.id, d.siteId, d.deviationType, PiApprovalStatus.EXPIRED, d.tenantId)
        — include d.tenantId in ProtocolDeviationResolvedEvent constructor

    AeEscalationCaseService.java
        — inject ClinicalMemoryService
        — in prepareAndMarkRequested(): ctx.put("tenantId", ae.tenantId);
          queryPatientContext + querySiteContext; inject results into ctx

    AeEscalationListener.java
        — inject ClinicalMemoryService
        — storeAeOutcome after ledgerWriter.writeCompletionEntry (tenantId from case context)
        — include ae.tenantId (read from case context) in AeEscalationCompletedEvent: NO CHANGE
          (AeEscalationCompletedEvent does not gain tenantId)

    PiResponseListener.java
        — inject ClinicalMemoryService; storePiDecision after ledgerWriter.writeResolutionEntry
        — include deviation.tenantId in ProtocolDeviationResolvedEvent constructor

    IrbDecisionListener.java
        — include approval.tenantId in IrbApprovalResolvedEvent constructor
        (no memory write — IRB domain deferred to #73)

    IrbDeviationCaseService.java
        — approval.tenantId = event.tenantId() in prepareAndCreateApproval()
        (ProtocolDeviationResolvedEvent now carries tenantId)

    SponsorNotificationListener.java
        — pass event.tenantId() into SponsorNotificationRequest constructor

    SponsorNotificationStore.java
        — n.tenantId = req.tenantId() in createPending()
```

### runtime/ configuration

```
runtime/src/main/resources/application.properties
    — add casehub-platform-memory-jpa index-dependency entries

runtime/src/test/resources/application.properties
    — add InMemoryMemoryStore to selected-alternatives

runtime/pom.xml
    — add casehub-platform-memory-jpa and casehub-platform-memory-inmem dependencies
```

### Test files requiring constructor update

*`AdverseEventReportedEvent` gains tenantId:*
```
runtime/src/test/java/io/casehub/clinical/service/
    AeEscalationLifecycleTest.java               — aeEvent() factory method
    SafetyOfficerNotificationIntegrationTest.java — 3 construction sites
    SafetyOfficerNotificationListenerTest.java    — 10+ construction sites (factory + inline)
    DsmbRollupTest.java                          — aeEvent() factory method
```

*`IrbApprovalResolvedEvent` gains tenantId:*
```
runtime/src/test/java/io/casehub/clinical/service/
    IrbDecisionListenerTest.java                 — 1 construction site
```

*`ProtocolDeviationResolvedEvent` gains tenantId:*
```
runtime/src/test/java/io/casehub/clinical/service/
    IrbCommitteePolicySpiTest.java               — criticalDeviationApproved() factory (2 sites)
    IrbGateLifecycleTest.java                    — criticalDeviationApproved() factory (2 sites)
    IrbDeviationCaseServiceTest.java             — 3 construction sites
    SponsorNotificationListenerTest.java         — 8+ construction sites (all test methods)
    PiResponseListenerMemoryTest.java            — new test (see §8)
```

---

## 8. Testing

| Test class | Type | Approach |
|---|---|---|
| `ClinicalMemoryServiceTest` | JUnit 5 unit | `InMemoryMemoryStore`; verify text, entityId, domain, attributes; verify swallowing on store failure |
| `ClinicalPatientContextTest` | JUnit 5 unit | Grade boundary (`GRADE_2` → false, `GRADE_3` → true); `empty()` shape; `hasPriorEscalation()` |
| `ClinicalSiteContextTest` | JUnit 5 unit | `recentTimelineBreachCount()` counts TIMELINE_BREACH entries; `empty()` returns zero |
| `AeEscalationListenerMemoryTest` | `@QuarkusTest` | `@InjectMock ClinicalMemoryService`; verify `storeAeOutcome` called with correct args; verify skipped when tenantId absent from context |
| `PiResponseListenerMemoryTest` | `@QuarkusTest` | `@InjectMock ClinicalMemoryService`; DONE → `storePiDecision` with APPROVED; DECLINE → REJECTED |
| `AeEscalationContextInjectionTest` | `@QuarkusTest` | Seed `InMemoryMemoryStore`; call `prepareAndMarkRequested()`; assert `initialContext` shape for both patientContext and siteContext; assert JQ expression `.patientContext.hasPriorGrade3OrAbove` evaluates correctly against the injected map |

`AeEscalationContextInjectionTest` must include a JQ evaluation assertion — map key
presence is not sufficient, since the engine evaluates JQ over `initialContext`. Use
`JQEvaluator` (available from `casehub-platform-expression`) or inline JQ evaluation
to confirm `.patientContext.hasPriorGrade3OrAbove` resolves to the expected boolean.

All `@InjectMock` beans stub methods in `@BeforeEach` per GE-20260604-4298f9
(stub or face null-return side effects across test methods).

---

## Open Issues Filed

- **casehubio/platform#79** — `assertTenant` async fix (blocking for full async write path)
- **casehubio/clinical#70** — degradation tracking until platform#79 ships
- **casehubio/clinical#71** — query isolation: tenant_id filter on all Panache reads
- **casehubio/clinical#72** — DRUG domain: entityId convention, recall path design, cross-tenant pharmacovigilance tradeoff
- **casehubio/clinical#73** — IRB domain: `IrbApproval.deviationType` gap (extra DB query vs. schema addition vs. entityId change)