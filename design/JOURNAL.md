# Design Journal — epic-adverse-event-escalation

### 2026-05-11 · §AdverseEventService

**SLA escalation is deployment config, not clinical code.** `AdverseEventService` sets `claimDeadline` and `candidateGroups` on the WorkItem — that is the full extent of clinical's escalation responsibility. What fires when the SLA is missed (DSMB alert, Slack, second WorkItem) is `EscalationPolicy` SPI configuration at deployment. This keeps clinical code portable across deployment environments without change.

**Two-datasource architecture locked in.** Default datasource: clinical domain entities + casehub-work entities. `qhorus` named datasource: casehub-ledger entities (`casehub.ledger.datasource=qhorus`). This follows the AML tutorial pattern and is now a documented convention in CLAUDE.md.

**Flyway numbering.** casehub-work's V1–V21+ migrations are at `classpath:db/migration` and are found by Quarkus when the JAR is on the classpath. Clinical domain migrations renamed V1-V6 → V100-V105 to avoid conflict. V1004 for `ae_ledger_entry` join table follows the platform V1004+ convention for consumer-owned ledger subclass joins.

**`AdverseEventLedgerEntry` pattern.** JOINED inheritance subclass of `LedgerEntry`. Fields `id`, `subjectId`, `sequenceNumber`, `entryType`, `actorId`, `actorType`, `actorRole`, and `occurredAt` must all be set manually by the caller — `LedgerEntryRepository.save()` has no builder. `subjectId = ae.id` (the AE is the audited aggregate, not the enrollment). First entry for an AE is always `sequenceNumber = 1`.

**CtcaeGrade SLA fix.** Grade 1 and 2 now return `Duration.ofDays(7)` instead of null. All grades now have non-empty `Optional<Duration>` from `sla()` — eliminates special-casing in service code.
