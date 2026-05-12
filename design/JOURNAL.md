# Design Journal — epic-adverse-event-escalation

### 2026-05-12 · §Epic4Implementation

**casehub-work package scope.** `quarkus.hibernate-orm.packages` must be `io.casehub.work.runtime` (not `.runtime.model`). casehub-work has Panache entities in `io.casehub.work.runtime.filter` (`FilterRule`) and other sub-packages beyond `.model`. The narrower scan silently omits these and causes a build-time entity-not-found failure.

**LedgerEntry subclasses must be in a dedicated package.** `AdverseEventLedgerEntry` lives in `io.casehub.clinical.ledger`, not `io.casehub.clinical.entity`. Quarkus Panache entities cannot be scanned by two persistence units simultaneously — if the same package appears in both `hibernate-orm.packages` and `hibernate-orm.qhorus.packages`, Quarkus throws `IllegalStateException` at build time. The separate `io.casehub.clinical.ledger` package is listed only under the qhorus PU.

**Quarkus ArC ignores beans.xml alternatives.** `JpaLedgerEntryRepository` is `@Alternative @ApplicationScoped`. Standard CDI `beans.xml` `<alternatives>` has no effect in Quarkus ArC — the config property `quarkus.arc.selected-alternatives` is required instead. Added to both `application.properties` and test properties.

**H2 multi-datasource JTA requires transactions=xa.** Any `@Transactional` method that writes to both the default and qhorus datasources needs `quarkus.datasource.*.jdbc.transactions=xa` in test `application.properties`. Agroal's default local-transaction mode raises "Failed to enlist" when a second datasource joins the same JTA transaction. H2 supports XA; PostgreSQL (production) handles XA natively and never shows this.

**Flyway V1004 conflict.** casehub-ledger itself owns V1004 (`actor_identity.sql`) — not just V1000-V1003. Consumer-owned ledger join tables (`ae_ledger_entry`) moved to V1005. Platform convention updated: casehub-ledger owns V1000-V1004; consumer joins start at V1005.

**Service test data setup.** `AdverseEvent.enrollment_id` has a FK to `patient_enrollment`. Service-level `@QuarkusTest` tests must create the full `ClinicalTrial → TrialSite → PatientEnrollment` chain before persisting an `AdverseEvent` — the service tests cannot use random UUIDs as `enrollmentId`.

### 2026-05-11 · §AdverseEventService

**SLA escalation is deployment config, not clinical code.** `AdverseEventService` sets `claimDeadline` and `candidateGroups` on the WorkItem — that is the full extent of clinical's escalation responsibility. What fires when the SLA is missed (DSMB alert, Slack, second WorkItem) is `EscalationPolicy` SPI configuration at deployment. This keeps clinical code portable across deployment environments without change.

**Two-datasource architecture locked in.** Default datasource: clinical domain entities + casehub-work entities. `qhorus` named datasource: casehub-ledger entities (`casehub.ledger.datasource=qhorus`). This follows the AML tutorial pattern and is now a documented convention in CLAUDE.md.

**Flyway numbering.** casehub-work's V1–V21+ migrations are at `classpath:db/migration` and are found by Quarkus when the JAR is on the classpath. Clinical domain migrations renamed V1-V6 → V100-V105 to avoid conflict. V1004 for `ae_ledger_entry` join table follows the platform V1004+ convention for consumer-owned ledger subclass joins.

**`AdverseEventLedgerEntry` pattern.** JOINED inheritance subclass of `LedgerEntry`. Fields `id`, `subjectId`, `sequenceNumber`, `entryType`, `actorId`, `actorType`, `actorRole`, and `occurredAt` must all be set manually by the caller — `LedgerEntryRepository.save()` has no builder. `subjectId = ae.id` (the AE is the audited aggregate, not the enrollment). First entry for an AE is always `sequenceNumber = 1`.

**CtcaeGrade SLA fix.** Grade 1 and 2 now return `Duration.ofDays(7)` instead of null. All grades now have non-empty `Optional<Duration>` from `sla()` — eliminates special-casing in service code.
