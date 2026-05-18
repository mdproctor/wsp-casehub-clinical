# casehub-clinical — Design

Living design document. Updated at each epic close via `design/JOURNAL.md` merge.

**Repository:** `casehubio/clinical`
**Platform doc:** `../parent/docs/PLATFORM.md`
**Baseline:** [ClinicalAgent arXiv 2404.14777](https://arxiv.org/abs/2404.14777)

---

## Module Structure

```
clinical/
├── api/       ← casehub-clinical-api   — enums, constants, SPIs, CDI events (no JPA, no Quarkus)
├── runtime/   ← casehub-clinical       — Quarkus app: JPA entities, REST, services, Flyway
└── pom.xml
```

**`api/`** contains enums, constants, SPI interfaces, and CDI event records. No POJO classes — following the platform persistence module split rule exception for self-contained application repos. Nothing outside this repo depends on `api/`.

**`runtime/`** depends on `api/`. Panache Active Record entities serve as domain objects — no separate POJO mapping layer.

---

## Domain Model

All entities are Panache Active Record (`extends PanacheEntityBase`). Relationships via FK columns only — no `@OneToMany` collections. UUID primary keys. Enums stored as VARCHAR.

### Core entities

```
ClinicalTrial(UUID id, String protocolId, TrialPhase phase, String sponsor,
              int targetEnrollment, TrialStatus status)

TrialSite(UUID id, UUID trialId, String investigatorId, SiteStatus status)
          // investigatorId: opaque string — wired to a qhorus actor ID (Epic 5)

PatientEnrollment(UUID id, UUID siteId, String patientId,
                  ConsentStatus consentStatus, EnrollmentStatus enrollmentStatus,
                  Instant enrolledAt)

AdverseEvent(UUID id, UUID enrollmentId, CtcaeGrade grade,
             EventActuality actuality, AeOutcome outcome,
             Instant occurredAt, Instant reportedAt, Instant slaDeadline,
             UUID workItemId)

ProtocolDeviation(UUID id, UUID siteId, String deviationType,
                  DeviationSeverity severity, PiApprovalStatus piApprovalStatus,
                  UUID commitmentId, String piCommandChannelName,
                  Instant commandedAt, Instant responseDeadline,
                  EscalationRequirement escalationRequirement)

IrbApproval(UUID id, UUID siteId, String reviewType, String committeeId,
            Instant decisionDeadline, IrbDecision decision)
```

`patientId` is an opaque string throughout — no PII fields. Pseudonymisation maps in cleanly in a later epic without touching this model.

### Enums and grading

`CtcaeGrade` follows CTCAE v5.0 directly. `sla()` returns non-empty `Optional<Duration>` for all grades — no special-casing needed at the service layer:

| Grade | Label | SLA |
|-------|-------|-----|
| GRADE_1 | Mild | 7 days |
| GRADE_2 | Moderate | 7 days |
| GRADE_3 | Severe | 24 hours |
| GRADE_4 | Life-threatening | 24 hours |
| GRADE_5 | Death | 1 hour (stricter than GCP minimum) |

`PiApprovalStatus` — 6-state machine:

| State | Meaning |
|-------|---------|
| PENDING | Deviation reported; COMMAND not yet issued (transient) |
| COMMANDED | COMMAND issued; Commitment OPEN; awaiting PI response |
| APPROVED | PI responded positively; MINOR deviations close here |
| REJECTED | PI declined; Commitment DECLINED |
| ESCALATED | PI approved; forwarded to IRB (CRITICAL) or sponsor (MAJOR) |
| EXPIRED | Deadline passed; Commitment FAILED; GCP SLA breach |

### Flyway version ranges

| Range | Purpose |
|-------|---------|
| V1–V99 | Core domain tables (trials, sites, patients, AEs, deviations, IRB) |
| V100–V999 | Clinical domain extensions (default datasource) |
| V1005+ | Ledger subclass join tables (qhorus datasource) |

Clinical uses datasource-scoped migration subdirectories (`db/migration/default/`, `db/migration/qhorus/`) to avoid version collision when casehub-work and casehub-qhorus are both on the classpath (each ships V1+).

---

## Platform Integration

### Adverse event SLA — casehub-work

`AdverseEventService.reportAdverseEvent()` creates a WorkItem for every adverse event with `claimDeadline = reportedAt + grade.sla()`. ClinicalAgent has no deadline tracking — this single integration delivers more regulatory value than ClinicalAgent's entire feature set.

`candidateGroups` routing: `"safety-officers"` for Grade 1-2; `"dsmb,safety-officers"` for Grade ≥ 3. Escalation on SLA miss is runtime configuration via casehub-work `EscalationPolicy` SPI — not clinical code.

### Tamper-evident audit — casehub-ledger

Domain-specific `LedgerEntry` subclasses using JPA JOINED inheritance. All must live in `io.casehub.clinical.ledger` — not `io.casehub.clinical.entity` — because Panache entities cannot span two persistence units.

| Subclass | Written when |
|----------|-------------|
| `AdverseEventLedgerEntry` | AE reported (V1005) |
| `ProtocolDeviationLedgerEntry` | COMMAND issued to PI (V1006); resolution events APPROVED/REJECTED/ESCALATED/EXPIRED (V1007 adds `terminal_status`, `resolved_at`) |

All protocol deviation ledger writes go through `DeviationLedgerWriter @ApplicationScoped`, which owns `sequenceNumber` computation via `findLatestBySubjectId` and ensures the full COMMAND→resolution chain is recorded. Written directly by: `ProtocolDeviationService` (COMMAND), `PiResponseListener` (PI response), `DeviationExpirationJob` (expiration).

### PI authorisation — casehub-qhorus

Protocol deviations require formal PI commitment via the qhorus COMMAND/Commitment lifecycle. No REST bypass — the PI's response arrives through a registered `HumanParticipatingChannelBackend` (Slack, email, PI dashboard). Bypassing would leave the Commitment with no RESPONSE message, making the audit trail structurally incomplete.

**Channel naming:** `clinical/deviation/{deviationId}/pi-oversight` — per-deviation, not per-site. Per-site channels were considered and rejected: `ChannelGateway.receiveHumanMessage()` passes `correlationId=null` (qhorus#154), so deviation identification from a per-site channel requires the channel name to encode the deviation ID anyway.

**Commitment lifecycle:** `MessageService.send()` auto-opens a Commitment for COMMAND type + non-null correlationId. The `correlationId` = `deviation.id.toString()` throughout — the stable correlator for all fulfillment operations.

**`DeviationResponsePolicy` SPI:** deadline and escalation path are co-located in one policy call. A deployer who needs "MAJOR oncology deviations at German sites require regulatory notification within 48h" implements one SPI. The default implementation reads from MicroProfile Config.

Default escalation mapping:

| Severity | Deadline | Escalation |
|----------|----------|------------|
| MINOR | 7 days | NONE |
| MAJOR | 72 hours | SPONSOR_NOTIFICATION |
| CRITICAL | 24 hours | IRB_REVIEW |

---

## Key Architecture Decisions

**`ProtocolDeviationResolvedEvent` for downstream decoupling.**
Fired by `PiResponseListener` and `DeviationExpirationJob` on every terminal state (APPROVED, REJECTED, ESCALATED, EXPIRED). Epic 6 (IRB gate) and Epic 13 (sponsor notification) observe this event — no modifications to the deviation service when they ship.

**`ClinicalInboundNormaliser` scoped to `/pi-oversight` channels.**
The `InboundNormaliser` SPI is application-wide. Scoping to channel names containing `/pi-oversight` prevents misclassifying messages on unrelated channels.

**`LedgerEntry` subclasses in `io.casehub.clinical.ledger`, not `io.casehub.clinical.entity`.**
Quarkus throws `IllegalStateException` at build time if the same package appears in two persistence unit configs. Ledger subclasses use the qhorus datasource; domain entities use the default datasource. Keeping them in separate packages is the only safe approach.

**Two-datasource architecture enforced by JpaLedgerEntryRepository `@Alternative`.**
Quarkus ArC ignores `beans.xml` alternatives — `quarkus.arc.selected-alternatives` config property is required in both `application.properties` files.

**Tests use `drop-and-create` + Flyway disabled.**
The classpath migration collision (casehub-work V1+ and casehub-qhorus V1+) cannot be resolved in tests without excluding JARs from scanning. Drop-and-create from Hibernate schema generation avoids the problem entirely.

**`quarkus.arc.exclude-types` for ledger SNAPSHOT services requiring reactive datasource.**
casehub-ledger 0.2-SNAPSHOT ships `LedgerVerificationService`, `LedgerComplianceReportService`, and `LedgerRetentionJob` which inject `ReactiveLedgerEntryRepository` — vetoed in the JDBC-only test environment. These must be excluded in test `application.properties`:
```properties
quarkus.arc.exclude-types=io.casehub.ledger.runtime.service.LedgerVerificationService,\
  io.casehub.ledger.runtime.service.LedgerComplianceReportService,\
  io.casehub.ledger.runtime.service.LedgerRetentionJob
```
When the ledger SNAPSHOT ships new services with reactive dependencies, add them here. Tracked upstream as clinical#17.

**`DeviationLedgerWriter` centralises sequenceNumber and ledger construction.**
Rather than each service constructing `ProtocolDeviationLedgerEntry` instances directly, all writes go through `DeviationLedgerWriter`. This ensures sequenceNumber is computed consistently from `findLatestBySubjectId` across all write sites. The pattern should be followed for any future ledger subclass that is written from multiple services.

---

## REST API (shipped)

```
POST   /trials                                                      → register ClinicalTrial
GET    /trials/{trialId}

POST   /trials/{trialId}/sites                                      → add TrialSite
GET    /trials/{trialId}/sites/{siteId}

POST   /trials/{trialId}/sites/{siteId}/patients                    → enroll patient
GET    /trials/{trialId}/sites/{siteId}/patients/{id}

POST   /trials/{trialId}/sites/{siteId}/patients/{id}/adverse-events → report AE (Epic 4)
GET    /trials/{trialId}/sites/{siteId}/patients/{id}/adverse-events/{aeId}

POST   /trials/{trialId}/sites/{siteId}/deviations                  → report deviation (Epic 5)
GET    /trials/{trialId}/sites/{siteId}/deviations/{id}
```

No PI response endpoint — the PI's formal response arrives via `HumanParticipatingChannelBackend`.

---

## Open / Deferred

| Issue | What | Blocker |
|-------|------|---------|
| casehubio/qhorus#153 | `MessageReceivedEvent` CDI hook — unblocks `PiResponseListenerIntegrationTest` | Required for full integration test |
| casehubio/clinical#6 | IRB gate — `ProtocolDeviationResolvedEvent` with `IRB_REVIEW` | casehubio/work#136 |
| casehubio/clinical#13 | Sponsor notification — `ProtocolDeviationResolvedEvent` with `SPONSOR_NOTIFICATION` | Connectors pattern |
| casehubio/clinical#16 | Remove redundant `commitmentService` calls (qhorus#154 auto-fulfills) | After qhorus#153 |
| casehubio/clinical#17 | Upstream ledger fix: reactive services need conditional activation (CDI startup blocker) | casehub-ledger session |
| casehubio/clinical#18 | `DeviationExpirationJob` REQUIRES_NEW — per-deviation transaction isolation | Refactor + test restructure |
| casehubio/clinical#19 | Inject `Clock` into `DeviationLedgerWriter` for deterministic timestamps | Low priority |
| casehubio/aml#20 | AML Flyway classpath collision — same fix as clinical | AML session |
