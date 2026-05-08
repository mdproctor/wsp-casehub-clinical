# Design: Epics 1 + 2 — Project Scaffold and Domain Model
2026-05-07

## Scope

Epic 1 (project scaffold) and Epic 2 (domain model) are designed together because the module
structure determines where domain types live. These two epics are prerequisites for Epic 3
(multi-site sub-case structure).

GitHub issues: casehubio/clinical#1 (scaffold), casehubio/clinical#2 (domain model)

---

## 1. Module Structure

```
clinical/                     ← git repo root, parent pom.xml
├── api/                      ← casehub-clinical-api (pure Java, zero JPA, zero Quarkus)
├── runtime/                  ← casehub-clinical (Quarkus app, JPA + Flyway + REST)
└── pom.xml
```

**Parent POM:** imports casehub-parent BOM. Declares `api/` and `runtime/` as modules.

**`api/`:** no Quarkus, no JPA deps. Contains enums and constants only — no POJO/record classes.
Nothing outside this repo depends on `api/` — following the platform's persistence module split
rule exception for self-contained application repos (PLATFORM.md Step 4).

**`runtime/`:** depends on `api/`. Full Quarkus stack: `quarkus-hibernate-orm-panache`,
`quarkus-rest`, Flyway. Panache Active Record entities serve as domain objects directly — no
separate POJO mapping layer needed.

Flyway version range: **V1–V999** (clinical domain tables). Ledger subclass join tables (Epic 7)
use V1004+ per platform convention.

---

## 2. Domain Model Field Definitions

`api/` contains **enums and constants only** — no POJO classes. The field definitions below
are implemented as Panache Active Record JPA entities in `runtime/`, following the
`WorkItem extends PanacheEntityBase` pattern from casehub-work. The entity IS the domain
object; no separate POJO mapping layer.

`patientId` is an opaque string throughout — no PII fields.
Pseudonymisation maps in cleanly in a later epic without touching this model.

### Entities

```java
ClinicalTrial(UUID id, String protocolId, TrialPhase phase, String sponsor,
              int targetEnrollment, TrialStatus status)

TrialSite(UUID id, UUID trialId, String investigatorId, SiteStatus status)
          // investigatorId: opaque string for now; wired to a Qhorus actor ID in Epic 5

PatientEnrollment(UUID id, UUID siteId, String patientId,
                  ConsentStatus consentStatus, EnrollmentStatus enrollmentStatus,
                  Instant enrolledAt)

AdverseEvent(UUID id, UUID enrollmentId, CtcaeGrade grade,
             EventActuality actuality, AeOutcome outcome,
             Instant occurredAt, Instant reportedAt, @Nullable Instant slaDeadline)

ProtocolDeviation(UUID id, UUID siteId, String deviationType,
                  DeviationSeverity severity, PiApprovalStatus piApprovalStatus)

IrbApproval(UUID id, UUID siteId, String reviewType, String committeeId,
            Instant decisionDeadline, IrbDecision decision)
```

### Enums

```
TrialPhase:         EARLY_PHASE_I, PHASE_I, PHASE_I_II, PHASE_II,
                    PHASE_II_III, PHASE_III, PHASE_IV
TrialStatus:        PLANNING, ENROLLING, ACTIVE, COMPLETED, TERMINATED
SiteStatus:         PENDING, ACTIVE, SUSPENDED, CLOSED
ConsentStatus:      PENDING, OBTAINED, WITHDRAWN
EnrollmentStatus:   CANDIDATE, SCREENING, ELIGIBLE, INELIGIBLE,
                    ENROLLED, ON_STUDY, OFF_STUDY, WITHDRAWN
EventActuality:     ACTUAL, POTENTIAL
AeOutcome:          ONGOING, RESOLVING, RESOLVED, FATAL, UNKNOWN
DeviationSeverity:  MINOR, MAJOR, CRITICAL
PiApprovalStatus:   PENDING, APPROVED, REJECTED
IrbDecision:        PENDING, APPROVED, REJECTED, DEFERRED
```

### CtcaeGrade (per CTCAE v5.0)

| Grade | Label | SLA |
|-------|-------|-----|
| GRADE_1 | Mild | none |
| GRADE_2 | Moderate | none |
| GRADE_3 | Severe | 24h |
| GRADE_4 | Life-threatening | 24h |
| GRADE_5 | Death | 1h |

SLA duration is a method on the enum (`Optional<Duration> sla()`). Returns empty for grades 1–2.

### Constants

```java
ClinicalCapabilities        // eligibility-screening, safety-monitoring, protocol-review,
                            // irb-consultation, pi-authorisation, data-safety-monitoring,
                            // regulatory-submission, trial-supervisor

ClinicalTrustDimensions     // safety-accuracy, eligibility-precision, protocol-adherence
```

### Validation against FHIR

Domain model validated against FHIR R5 ResearchStudy, ResearchSubject, and AdverseEvent
resources. Key alignments:
- `TrialPhase` values match FHIR phase codes including combined-phase arms (PHASE_I_II etc.)
- `targetEnrollment` maps to FHIR `recruitment.targetNumber`
- `EnrollmentStatus` maps to FHIR `ResearchSubject.subjectState`
- `occurredAt` / `reportedAt` distinction matches FHIR `occurrenceDateTime` / `recordedDate`
- `EventActuality` (ACTUAL | POTENTIAL) maps to FHIR `AdverseEvent.actuality`
- CTCAE grading: FHIR explicitly defers to implementation guides — `CtcaeGrade` defined
  directly per CTCAE v5.0

---

## 3. JPA Entities (`runtime/`)

One Panache Active Record entity per domain object. Relationships via FK columns — no
`@OneToMany` collections. Queries done explicitly via Panache `find()`.

```
ClinicalTrialEntity    → table: clinical_trial      (V1)
TrialSiteEntity        → table: trial_site           (V2)
PatientEnrollmentEntity → table: patient_enrollment  (V3)
AdverseEventEntity     → table: adverse_event        (V4)
ProtocolDeviationEntity → table: protocol_deviation  (V5)
IrbApprovalEntity      → table: irb_approval         (V6)
```

UUID primary keys. Enum columns stored as VARCHAR. `Instant` columns as `TIMESTAMP`.
`slaDeadline` on `adverse_event` is computed at insert time from `reportedAt + grade.sla()`.
Null for Grade 1 and 2 (no SLA). Column is nullable.

---

## 4. REST Endpoints (`runtime/`)

Pure domain CRUD — no case engine calls (that is Epic 3). Seeding the engine case context
on trial/site/patient creation is out of scope here.

```
POST   /trials                                          → register ClinicalTrial
GET    /trials/{trialId}                                → get trial

POST   /trials/{trialId}/sites                          → add TrialSite
GET    /trials/{trialId}/sites/{siteId}                 → get site

POST   /trials/{trialId}/sites/{siteId}/patients        → enroll patient
GET    /trials/{trialId}/sites/{siteId}/patients/{id}   → get enrollment
```

Adverse event and protocol deviation endpoints arrive in Epics 4 and 5 respectively, where
they trigger case engine bindings.

---

## Reference Standards

- ICH E6(R3) GCP: https://www.ich.org/page/efficacy-guidelines
- CTCAE v5.0: https://ctep.cancer.gov/protocoldevelopment/electronic_applications/ctc.htm
- FHIR ResearchStudy: https://hl7.org/fhir/researchstudy.html
- FHIR ResearchSubject: https://hl7.org/fhir/researchsubject.html
- FHIR AdverseEvent: https://hl7.org/fhir/adverseevent.html
- 21 CFR Part 312: https://www.ecfr.gov/current/title-21/chapter-I/subchapter-D/part-312
