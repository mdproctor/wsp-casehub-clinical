---
layout: post
title: "Silence is not an audit trail"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [gcp, cdi, ledger, jta, compliance]
---

Issues #45 and #46 are in main.

#46 was straightforward. Five ledger writers were using the string `"system"` as their actorId; `DeviationLedgerWriter` had already defined `SYSTEM_ACTOR = "clinical-service"` and used it correctly. An FDA auditor querying by actorId got inconsistent results — AE, escalation, and safety officer entries all invisible to a `WHERE actor_id = 'clinical-service'` filter. The fix was a `ClinicalActors.CLINICAL_SERVICE` constant in `api/`, four writers updated, `SYSTEM_ACTOR` removed. `IrbApprovalLedgerWriter` stays on `"irb-committee"` with `ActorType.HUMAN` — that's the IRB committee acting, not the system.

#45 was the interesting one.

The previous entry noted that `SafetyOfficerNotificationListener` had a compliance gap: its `@ObservesAsync @Transactional` method caught connector delivery failures inside the notifier, but if anything threw before or after the notifier call — a Panache lookup failing, a transient DB error during the trial entity load — the CDI event dispatcher swallowed it. No ledger entry. No signal. The absence is indistinguishable in the audit trail from "the notification was never triggered."

The fix is a double try/catch. The outer catch wraps the entire body: if it fires, we call `ledgerWriter.writeObserverFailureEntry(event.aeId(), event.enrollmentId(), event.siteId(), event.grade())`. The inner catch wraps that call: if even the fallback write fails — which it will if the DB is completely unavailable — we log `"AUDIT GAP: could not write observer failure entry for AE %s"`. That prefix is operator-searchable. We can't escalate further without a retry mechanism that doesn't exist, but at least the failure isn't silent.

`writeObserverFailureEntry` is `@Transactional(REQUIRES_NEW)`. This matters because the outer observer transaction may be in rollback-only state when the exception fires — a Hibernate error can mark the session before the exception even propagates. `REQUIRES_NEW` suspends whatever outer transaction state exists and opens a fresh one. If it commits, the fallback entry is durable regardless of what happens to the outer TX. Both `SafetyOfficerNotificationListener` and `SponsorNotificationListener` got the same treatment.

Code review caught a semantic issue with the fallback entry. `writeObserverFailureEntry` was delegating to `writeEntry()`, which sets `notifiedAt = clock.instant()` unconditionally. Claude flagged that for observer-level failures — where the error occurred before the notifier was ever called — `notifiedAt` should be null. No notification was attempted. Setting the timestamp implies one was. The logic was right.

But `SafetyOfficerNotificationLedgerEntry.notifiedAt` is `@Column(name = "notified_at", nullable = false)`. Setting it to null doesn't fail at `save()`. Hibernate queues the insert; the `ConstraintViolationException` fires at flush time, when the `REQUIRES_NEW` transaction commits. The inner catch swallowed the wrapped `RollbackException`. No exception propagated. But the fallback entry was never written, and the test's `assertThat(entry).isNotNull()` failed — appearing to suggest the listener had thrown, which it hadn't.

I reverted. The column stays NOT NULL. `notifiedAt = clock.instant()` stays. The semantic distinction between failure modes is `connectorId`: null means the error happened before the notifier was reached and connector config was never loaded; a set `connectorId` with `delivered=false` means the notifier ran and the connector threw. No schema migration, no new column.

The integration test covers the full chain: `SafetyOfficerNotificationIntegrationTest` fires `AdverseEventReportedEvent` against a real `TestSlackConnector` and a real H2 ledger, then queries `LedgerEntryRepository.findLatestBySubjectId(aeId)` and verifies `delivered=true`, the correct site and connector ID. Calling `listener.onAeReported()` directly — no Awaitility — because `DefaultSafetyOfficerNotifier.notify()` is `REQUIRES_NEW` but synchronous from the test thread. The commit is complete before the listener returns.

One gap filed as #49: the early-return paths in both listeners — site not found, trial not found, connector config missing — currently log a warning and return. No ledger entry is written. ICH E6(R3) §5.17 requires that non-notification be independently verifiable too, not just failed-notification. The `delivered=false` entries we added cover errors; they don't cover deliberate skips. That's the next scope.

158 tests passing.
