# CaseMemoryStore Integration — casehub-clinical

**Date:** 2026-06-08 (revised 2026-06-09)
**Issue:** casehubio/clinical#33 (closes), casehubio/clinical#69 (closes)
**Branch:** issue-33-casememorystore-integration
**Platform dependency:** casehubio/platform#79 (assertTenant async fix)
**Degradation tracking:** casehubio/clinical#70
**Deferred:** DRUG and IRB domains — tracked in issues filed at end of this spec

---

## Problem

Every clinical trial case starts cold. A new AE escalation case for a patient
with prior Grade 3 hepatotoxicity starts without that history. A new deviation
review at a site with three recent reporting breaches starts without site
compliance context. CaseMemoryStore closes this gap by surfacing patient and
site history before the first agent runs, and accumulating outcome facts across
cases for future enrichment.

---

## Scope

1. **Multi-tenancy foundation** — `tenantId` on all domain entities (closes #69); `tenantId` on two CDI events; `CurrentPrincipal` injection into all entity-creating services
2. **Two memory domains** — PATIENT and SITE (DRUG and IRB deferred — see Deferred Issues)
3. **`ClinicalMemoryService`** — central write/read facade
4. **Context value objects** — `ClinicalPatientContext`, `ClinicalSiteContext`
5. **Integration points** — where facts are written and recalled
6. **Testing strategy**
7. **ARC42STORIES.MD update** — Layer 8

---

## 1. Multi-tenancy Foundation (closes #69)

### Entity changes — 6 tables

Add `String tenantId` field to:
```
ClinicalTrial, TrialSite, PatientEnrollment,
ProtocolDeviation, AdverseEvent, IrbApproval
```

Single Flyway migration **V111** in `db/migration/default/` with 6 ALTER TABLE
statements:

```sql
ALTER TABLE clinical_trial       ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE trial_site           ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE patient_enrollment   ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE protocol_deviation   ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE adverse_event        ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
ALTER TABLE irb_approval         ADD COLUMN tenant_id VARCHAR(255) NOT NULL DEFAULT 'default';
```

`DEFAULT 'default'` keeps the migration safe on existing rows. Production
deployments replace these defaults via a one-time backfill migration before
enabling multi-tenant routing.

`SponsorNotification` already has a nullable `tenant_id` column at V115 (marked
"Future multi-tenancy" in the entity comment). No new migration needed; the
population path is handled via `SponsorNotificationRequest` (see §Modified Files).

### Entity creation — tenantId population

Every entity-creating code path must set `tenantId` before `persist()`. The
`CurrentPrincipal` is the source of truth on request-scoped paths:

| Entity | Creation site | tenantId source |
|---|---|---|
| `ClinicalTrial` | `TrialResource.register()` | `principal.tenancyId()` — inject `CurrentPrincipal` |
| `TrialSite` | `SiteResource.addSite()` | `principal.tenancyId()` — inject `CurrentPrincipal` |
| `PatientEnrollment` | `PatientResource.enroll()` | `principal.tenancyId()` — inject `CurrentPrincipal` |
| `ProtocolDeviation` | `DeviationResource` | `principal.tenancyId()` — inject `CurrentPrincipal` |
| `AdverseEvent` | `PatientResource` → `AdverseEventService` | `principal.tenancyId()` — inject `CurrentPrincipal` in `AdverseEventService`; set `ae.tenantId` before `ae.persist()` |
| `IrbApproval` | `IrbDeviationCaseService.prepareAndCreateApproval()` | `event.tenantId()` from `ProtocolDeviationResolvedEvent` |
| `SponsorNotification` | `DurableSponsorNotifier` | `request.tenantId()` from `SponsorNotificationRequest` |

`PatientResource` constructs the `AdverseEvent` and passes it to
`AdverseEventService.reportAdverseEvent(ae)`. The service must inject
`CurrentPrincipal` and set `ae.tenantId = principal.tenancyId()` before
`ae.persist()`. This is the only case where a resource constructs an entity
that a service persists.

### Event changes — 2 events

Only two CDI events need `tenantId()` added. The criterion is: does the
event propagate to an async observer that needs tenantId for a memory write or
entity creation, and cannot read it from a loaded entity?

| Event | Gains tenantId | Emitters | Why |
|---|---|---|---|
| `ProtocolDeviationResolvedEvent` | ✅ | `PiResponseListener`, `DeviationExpirer` | `IrbDeviationCaseService` creates `IrbApproval` from the event; `SponsorNotificationListener` creates `SponsorNotification` via notifier; no entity to load tenantId from at observer call time |
| `AdverseEventReportedEvent` | ✅ | `AdverseEventService` | `AeEscalationCaseService` is `@ObservesAsync`; gets tenantId from the event to put in case context |
| `AeEscalationCompletedEvent` | ❌ | `AeEscalationListener` | Only consumer is `TrialSafetySignalService`, which does not need tenantId. `AeEscalationListener` reads tenantId from case context for memory writes — the event is not the carrier |
| `IrbApprovalResolvedEvent` | ❌ | `IrbDecisionListener` | No memory write on IRB domain in this release (IRB domain deferred) |

Emitter stamp sources for the two events that gain `tenantId`:

- `AdverseEventReportedEvent` — stamped in `AdverseEventService.reportAdverseEvent()` from `principal.tenancyId()` (request-context path, `CurrentPrincipal` available)
- `ProtocolDeviationResolvedEvent` — stamped by both emitters from `d.tenantId` or `deviation.tenantId` (entity field, loaded before fire in both `PiResponseListener` and `DeviationExpirer`)

### Query isolation

This spec adds `tenant_id` columns and populates them correctly. It does NOT
add tenant-scoped Panache queries to REST endpoints — `AdverseEvent.findById()`
and friends remain unscoped. Tenant isolation at query level is a GDPR compliance
gap tracked in casehubio/clinical#71 (filed at end of this spec). This is an
explicit deferral, not an oversight.

---

## 2. Memory Domains

New class `ClinicalMemoryDomains` in `io.casehub.clinical.memory`.

Two domains in this release:

| Constant | Domain name | entityId convention | What is stored |
|---|---|---|---|
| `PATIENT` | `clinical-patient` | `patient:{enrollmentId}` | AE report facts (grade, site); AE outcome facts (safety review, DSMB escalation) |
| `SITE` | `clinical-site` | `site:{siteId}` | AE report events by grade; deviation reports by type and severity; PI decision outcomes |

**Deferred to separate issues:**
- `DRUG` (`clinical-drug`, `protocol:{protocolId}`) — cross-site AE signals per protocol. Requires `protocolId` in case context (additional entity chain traversal) and a defined recall path. Filed as casehubio/clinical#72.
- `IRB` (`clinical-irb`, `deviation-type:{deviationType}`) — IRB decision precedent per deviation type. Requires either an extra entity load in `IrbDecisionListener` or adding `deviationType` to `IrbApproval`. Filed as casehubio/clinical#73.

---

## 3. `ClinicalMemoryService`

**Package:** `io.casehub.clinical.memory`
**Scope:** `@ApplicationScoped`

No caller touches `CaseMemoryStore` directly. All domain semantics live here.

Contract (follows aml `AmlMemoryService`):
- All writes wrapped in `try/catch` — failures log WARN, never propagate
- All queries return context objects or `empty()` — never throw
- `tenantId` is always an explicit parameter — the service does NOT inject `CurrentPrincipal`
- Constructor injection: `CaseMemoryStore` only (no `PreferenceProvider`)

### Write methods

```java
void storeAeReport(UUID enrollmentId, UUID siteId, CtcaeGrade grade, String tenantId)

void storeAeOutcome(UUID aeId, UUID enrollmentId, CtcaeGrade grade,
                    String safetyReview, boolean dsmbEscalated, String tenantId)

void storeDeviationReport(UUID deviationId, UUID siteId,
                          String deviationType, DeviationSeverity severity, String tenantId)

void storePiDecision(UUID deviationId, UUID siteId,
                     String deviationType, PiApprovalStatus status, String tenantId)
```

### Query methods

```java
ClinicalPatientContext queryPatientContext(UUID enrollmentId, String tenantId)
ClinicalSiteContext    querySiteContext(UUID siteId, String tenantId)
```

### Attribute conventions

Standard keys from `io.casehub.platform.api.memory.MemoryAttributeKeys`
(`MemoryAttributeKeys` exists in `casehub-platform-api` — do not redefine):

- `ACTOR_ID` → `"clinical-service"` on all facts
- `OUTCOME` → domain-specific string: `"GRADE_3"`, `"APPROVED"`, `"TIMELINE_BREACH"`, etc.

CONFIDENCE is omitted. Clinical facts are deterministic — AE grade is a reported
fact, PI approval is a recorded decision. Confidence belongs to probabilistic
agent assessments, not factual records. Copying AML's confidence pattern without
a clinical use case would be noise.

---

## 4. Context Value Objects

### `ClinicalPatientContext`

Wraps `List<Memory> aeHistory` (all PATIENT domain recalls for the enrollment).

```java
public record ClinicalPatientContext(List<Memory> aeHistory) {
    public static ClinicalPatientContext empty()
    public boolean hasHistory()
    public boolean hasPriorGrade3OrAbove()   // any memory where OUTCOME attr is GRADE_3/GRADE_4/GRADE_5
    public boolean hasPriorEscalation()      // any memory where OUTCOME attr contains "ESCALATED"
    public Map<String, Object> toContextMap()
}
```

`toContextMap()` shape (injected under key `patientContext` in engine initialContext):

```json
{
  "hasHistory": true,
  "hasPriorGrade3OrAbove": true,
  "hasPriorEscalation": false,
  "aeCount": 2,
  "facts": [
    { "grade": "GRADE_3", "outcome": "RESOLVED", "createdAt": "2026-04-12T10:00:00Z" }
  ]
}
```

YAML bindings in `AeEscalationCaseDefinition` key off
`.patientContext.hasPriorGrade3OrAbove` to tighten monitoring intensity.

### `ClinicalSiteContext`

Wraps `List<Memory> complianceEvents` (SITE domain, bounded by `withSince()`
in the query — see §5 Recall Path).

```java
public record ClinicalSiteContext(List<Memory> complianceEvents) {
    public static ClinicalSiteContext empty()
    public boolean hasComplianceIssues()
    public int recentTimelineBreachCount()   // count where OUTCOME == "TIMELINE_BREACH"
    public Map<String, Object> toContextMap()
}
```

`recentTimelineBreachCount()` counts matching entries in the `complianceEvents`
list. The time window is enforced by the query (`withSince` 180 days), not by
filtering in the context object — the store does the bounding; the context counts
what the store returns.

`toContextMap()` shape (injected under key `siteContext`):

```json
{
  "hasComplianceIssues": true,
  "recentTimelineBreachCount": 3,
  "facts": [...]
}
```

Both return `empty()` on any adapter failure. No case is ever blocked.

---

## 5. Integration Points

### Write — request-context paths (work on merge)

"Request-context" means the caller runs on the HTTP request thread with an
active CDI request scope and `CurrentPrincipal` available.

| Service | Method | Domains written |
|---|---|---|
| `AdverseEventService.reportAdverseEvent()` | `storeAeReport(enrollmentId, siteId, grade, tenantId)` | PATIENT, SITE |
| `ProtocolDeviationService.reportDeviation()` | `storeDeviationReport(deviationId, siteId, deviationType, severity, tenantId)` | SITE |

Both calls happen inside the `@Transactional` service method before
`fireAsync()`. tenantId comes from `ae.tenantId` / `deviation.tenantId` (just set
above in the same method via `CurrentPrincipal`).

### Write — async observer paths (degrade to WARN until platform#79)

"Async observer" means the caller runs on the CDI managed executor thread
(`@ObservesAsync`), outside request scope. `assertTenant()` in the adapters
will throw `SecurityException` (no `CurrentPrincipal` in scope); `ClinicalMemoryService`'s
try/catch absorbs it and logs WARN. When platform#79 ships, async writes activate
automatically — no clinical code change required.

| Observer | Method | Domains | tenantId source |
|---|---|---|---|
| `AeEscalationListener.onCaseLifecycle()` | `storeAeOutcome(...)` | PATIENT | case context `"tenantId"` key |
| `PiResponseListener.process()` | `storePiDecision(...)` | SITE | `deviation.tenantId` (loaded entity) |

`AeEscalationListener` reads `tenantId` from case context — NOT from an entity
load. `AeEscalationCaseService.prepareAndMarkRequested()` adds
`ctx.put("tenantId", ae.tenantId)` alongside the existing `aeId`, `enrollmentId`,
etc. fields.

### Recall path — `AeEscalationCaseService.prepareAndMarkRequested()`

This `@Transactional` method already loads `AdverseEvent ae`. After migration,
`ae.tenantId` is available. Two additions:

1. Stamp tenantId in case context for downstream use:
```java
ctx.put("tenantId", ae.tenantId);
```

2. Recall prior context (runs in `@ObservesAsync` thread — no request scope,
   same async degradation as writes until platform#79):
```java
ClinicalPatientContext patientCtx =
    memoryService.queryPatientContext(ae.enrollmentId, ae.tenantId);
// Resolve siteId from enrollment for site context
PatientEnrollment enrollment = PatientEnrollment.findById(ae.enrollmentId);
UUID siteId = enrollment != null ? enrollment.siteId : null;
ClinicalSiteContext siteCtx = siteId != null
    ? memoryService.querySiteContext(siteId, ae.tenantId)
    : ClinicalSiteContext.empty();
ctx.put("patientContext", patientCtx.toContextMap());
ctx.put("siteContext",    siteCtx.toContextMap());
```

`querySiteContext` uses `withSince(Instant.now().minus(180, DAYS)).withLimit(50)`
(enforced in the query, not in the context object). This bounds the compliance
event window correctly even on high-volume sites.

Until platform#79 ships, both context maps are always `empty()` — cases run
exactly as today, without enrichment. When the fix ships, historical context
activates automatically.

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

### CDI activation rules

**Production (`JpaMemoryStore`, `io.casehub.platform.memory.jpa`):**
`@ApplicationScoped` — displaces `NoOpCaseMemoryStore` (`@DefaultBean`) by
classpath presence alone. No `selected-alternatives` entry needed.

**Test (`InMemoryMemoryStore`, `io.casehub.platform.memory.inmem`):**
`@Alternative @Priority(10)` — must be activated explicitly:
```properties
# test application.properties — append to existing selected-alternatives value
quarkus.arc.selected-alternatives=...,\
  io.casehub.platform.memory.inmem.InMemoryMemoryStore
```

### Flyway for `memory-jpa`

`casehub-platform-memory-jpa` ships Flyway migrations at `classpath:db/memory/migration`
(V1000 in platform's own numbering). Clinical must include this path:

```properties
# application.properties (production)
quarkus.flyway.locations=classpath:db/migration/default,classpath:db/memory/migration

# application.properties (test) — Flyway disabled for H2 tests; no change needed
```

No `index-dependency` entry needed for `memory-jpa` — the module has no `quarkus:build`
goal; Jandex is not required for JPA entity discovery (EntityManager is injected).

---

## 6. New Files

```
runtime/src/main/java/io/casehub/clinical/memory/
    ClinicalMemoryDomains.java      — PATIENT + SITE domain constants
    ClinicalMemoryService.java      — central write/read facade
    ClinicalPatientContext.java     — patient prior context value object
    ClinicalSiteContext.java        — site compliance context value object

runtime/src/main/resources/db/migration/default/
    V111__add_tenant_id_to_entities.sql   — 6 ALTER TABLE statements
```

---

## 7. Modified Files (complete)

### `api/` module

```
api/src/main/java/io/casehub/clinical/api/
    AdverseEventReportedEvent.java          — + String tenantId()
    ProtocolDeviationResolvedEvent.java     — + String tenantId()
    SponsorNotificationRequest.java         — + String tenantId() (passes through to DurableSponsorNotifier)
```

### `runtime/` — entities

```
runtime/src/main/java/io/casehub/clinical/entity/
    ClinicalTrial.java                      — + String tenantId
    TrialSite.java                          — + String tenantId
    PatientEnrollment.java                  — + String tenantId
    ProtocolDeviation.java                  — + String tenantId
    AdverseEvent.java                       — + String tenantId
    IrbApproval.java                        — + String tenantId
```

### `runtime/` — REST resources (CurrentPrincipal injection)

```
runtime/src/main/java/io/casehub/clinical/resource/
    TrialResource.java                      — inject CurrentPrincipal; stamp trial.tenantId
    SiteResource.java                       — inject CurrentPrincipal; stamp site.tenantId
    PatientResource.java                    — inject CurrentPrincipal; stamp enrollment.tenantId
    DeviationResource.java                  — inject CurrentPrincipal; stamp deviation.tenantId
```

### `runtime/` — services

```
runtime/src/main/java/io/casehub/clinical/service/
    AdverseEventService.java            — inject CurrentPrincipal; set ae.tenantId; storeAeReport;
                                          stamp tenantId on AdverseEventReportedEvent
    ProtocolDeviationService.java       — storeDeviationReport (tenantId from principal, already injected? check)
    AeEscalationCaseService.java        — ctx.put("tenantId", ae.tenantId); queryPatientContext;
                                          querySiteContext; ctx.put patientContext + siteContext
    AeEscalationListener.java          — read tenantId from case context; storeAeOutcome (async, degrades)
    PiResponseListener.java             — stamp deviation.tenantId on ProtocolDeviationResolvedEvent;
                                          storePiDecision (async, degrades)
    DeviationExpirer.java               — stamp d.tenantId on ProtocolDeviationResolvedEvent (in expireOne)
    IrbDeviationCaseService.java        — stamp approval.tenantId = event.tenantId()
                                          (mechanically affected by event change; no memory write in this release)
    SponsorNotificationListener.java    — pass event.tenantId() through SponsorNotificationRequest
                                          (mechanically affected by event change)
    DurableSponsorNotifier.java         — set sn.tenantId = request.tenantId() on SponsorNotification
```

### `runtime/` — configuration

```
runtime/src/main/resources/application.properties
    — quarkus.flyway.locations: add classpath:db/memory/migration
    — no selected-alternatives change (JpaMemoryStore is @ApplicationScoped)
    — add quarkus.index-dependency entries for memory-jpa if Jandex probe needed

runtime/src/test/resources/application.properties
    — quarkus.arc.selected-alternatives: append InMemoryMemoryStore
```

### `runtime/pom.xml`

```
— add casehub-platform-memory-jpa (compile scope)
— add casehub-platform-memory-inmem (test scope)
```

### Test files (compilation breakage from event record changes)

Every construction site of `ProtocolDeviationResolvedEvent` and
`AdverseEventReportedEvent` must add the new `tenantId` argument.

```
api/src/test/java/io/casehub/clinical/api/
    ClinicalConstantsTest.java                   — 1 construction site

runtime/src/test/java/io/casehub/clinical/service/
    SponsorNotificationListenerTest.java         — 10+ construction sites (factory method + inline)
    IrbDeviationCaseServiceTest.java             — 2 construction sites
    IrbCommitteePolicySpiTest.java               — 1 factory method + 1 inline
    IrbGateLifecycleTest.java                    — 1 factory method + 1 inline
    AeEscalationListenerTest.java                — 0 (AeEscalationCompletedEvent unchanged)
    TrialSafetySignalServiceTest.java            — 0 (AeEscalationCompletedEvent unchanged)
```

`SponsorNotificationListenerTest` also constructs `SponsorNotificationRequest`;
if that record changes, test construction sites must be updated.

---

## 8. Testing

### JQ verification note

The patientContext and siteContext maps are injected into the engine's
`initialContext` as `Map<String, Object>`. YAML bindings reference them as
`.patientContext.hasPriorGrade3OrAbove`. CLAUDE.md documents JQ silent-failure
patterns (e.g. missing `[]` on array iteration). The injection test must verify:
1. The map key is present with the correct type (`boolean`, not `Boolean` boxed
   in a nested map that JQ would navigate differently)
2. A JQ expression test confirms `.patientContext.hasPriorGrade3OrAbove` evaluates
   to `true` when a prior Grade 3 fact exists — not just that the map key is set

| Test class | Type | What it verifies |
|---|---|---|
| `ClinicalMemoryServiceTest` | JUnit 5 unit | `InMemoryMemoryStore`; correct entityId, domain, text, attributes; exceptions swallowed |
| `ClinicalPatientContextTest` | JUnit 5 unit | `hasPriorGrade3OrAbove()` grade boundary (GRADE_2=false, GRADE_3=true); `empty()` shape |
| `ClinicalSiteContextTest` | JUnit 5 unit | `recentTimelineBreachCount()` counts correctly; no time-window filtering in context (store does it) |
| `AeEscalationListenerMemoryTest` | `@QuarkusTest` | `@InjectMock ClinicalMemoryService`; `storeAeOutcome()` called with correct args after case completes |
| `PiResponseListenerMemoryTest` | `@QuarkusTest` | `@InjectMock ClinicalMemoryService`; `storePiDecision()` called on DONE / DECLINE |
| `AeEscalationContextInjectionTest` | `@QuarkusTest` | Seed store; call `prepareAndMarkRequested()`; assert `initialContext` shape AND JQ expression evaluation |

All `@InjectMock` beans stubbed in `@BeforeEach` per GE-20260604-4298f9.

`ProtocolDeviationService` does not have `CurrentPrincipal` currently — check
whether it needs it for `storePiDecision`'s tenantId. In the write path:
`storePiDecision` is called from `PiResponseListener` (async), which reads
`deviation.tenantId` from the loaded entity. `ProtocolDeviationService` writes
the deviation at creation time and does not directly call memory service on the
SITE domain for the PI decision outcome. Confirm during implementation.

---

## 9. ARC42STORIES.MD — Layer 8

CaseMemoryStore integration is Layer 8 in the clinical layer taxonomy.
ARC42STORIES.MD §4 (Layer Taxonomy table) and §9.2 (chapter index) must be
updated with a Layer 8 entry upon completion of this issue. LAYER-LOG.md must
also receive a Layer 8 entry before the branch is closed. Neither update is
tracked separately — they are part of the closing checklist for #33.

---

## 10. Open Issues Filed (before leaving brainstorming)

| Issue | Repo | What |
|---|---|---|
| #70 | casehubio/clinical | Async memory writes degrade silently until platform#79 ships |
| #71 | casehubio/clinical | Query-level tenant isolation for all domain entities (Panache queries unscoped) |
| #72 | casehubio/clinical | DRUG domain: cross-site AE signals per protocol (recall path design required) |
| #73 | casehubio/clinical | IRB domain: IRB decision precedent per deviation type (requires deviationType in IrbDecisionListener) |
| platform#79 | casehubio/platform | Skip assertTenant when CDI request context is not active — trust MemoryInput.tenantId() directly |

---

## Review Point Disposition

This section records the outcome of each review finding for traceability.

| Finding | Status | Resolution |
|---|---|---|
| C1 MemoryAttributeKeys fabricated | **Reviewer wrong** | Class exists at `io.casehub.platform.api.memory.MemoryAttributeKeys`; verified from JAR. No spec change. |
| C2 JPA class name wrong | Partial — class name corrected | `JpaMemoryStore` is `@ApplicationScoped` (no `selected-alternatives`); `InMemoryMemoryStore` is `@Alternative @Priority(10)` (needs `selected-alternatives` in test). Spec updated. |
| C3 IrbDecisionListener extra query | Resolved by D2 deferral | IRB domain deferred to #73. C3 is moot in this release. |
| C4 Blast radius understated | Accepted | Modified files list expanded to include `DeviationExpirer`, `IrbDeviationCaseService`, `SponsorNotificationListener`, all test construction sites. |
| C5 AdverseEventService no CurrentPrincipal | Accepted | Added to modified files; spec now traces tenantId population explicitly. |
| C6 tenantId population unspecified | Accepted | All 7 entity creation paths traced in §1. |
| D1 6-month window on wrong data | Accepted | `querySiteContext` uses `withSince(Instant.now().minus(180, DAYS)).withLimit(50)`. Context object does not filter — counts what store returns. |
| D2 Write-only domains deferred cost | Accepted | DRUG and IRB domains deferred to #72 and #73. Only PATIENT and SITE in this release. |
| D3 CONFIDENCE on deterministic facts | Accepted | CONFIDENCE removed from all clinical memory writes. |
| D4 AeEscalationCompletedEvent.tenantId | Accepted | `AeEscalationCompletedEvent` does NOT gain tenantId. Listener reads tenantId from case context. |
| D5 PreferenceProvider unexplained | Accepted | Removed from `ClinicalMemoryService`. |
| D6 Query isolation unaddressed | Accepted | Explicitly scoped as column-only. Query isolation tracked in #71. |
| D7 SponsorNotification omission | Addressed | `SponsorNotification.tenantId` already exists (nullable at V115). Population path: `SponsorNotificationRequest` gains `tenantId()`; `DurableSponsorNotifier` stamps it. `SponsorNotificationListener` in modified files; no memory write from sponsor path. |
| S1 Six migrations for one concern | Accepted | Single V111 migration with 6 ALTER TABLE statements. |
| S2 DRUG signals cross-tenant | Resolved by D2 deferral | Addressed in #72 spec when DRUG domain is designed. |
| S3 JQ navigation unverified | Accepted | Testing section adds JQ expression evaluation requirement to `AeEscalationContextInjectionTest`. |
| Minor sync/async terminology | Accepted | "request-context paths" vs "async observer paths" throughout. |
