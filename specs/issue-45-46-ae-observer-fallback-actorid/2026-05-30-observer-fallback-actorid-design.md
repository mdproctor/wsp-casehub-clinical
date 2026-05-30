# Design: AE Observer Fallback + actorId Alignment

**Branch:** issue-45-46-ae-observer-fallback-actorid  
**Issues:** casehubio/clinical#45, casehubio/clinical#46  
**Date:** 2026-05-30

---

## Problem

### #46 — actorId inconsistency

`DeviationLedgerWriter.SYSTEM_ACTOR = "clinical-service"` is the intended system identity
in the tamper-evident audit trail. Every other automated writer uses the bare string `"system"`:
`AdverseEventLedgerWriter`, `AeEscalationLedgerWriter`, `SafetyOfficerNotificationLedgerWriter`,
and `DeviationExpirer` (which passes `"system"` directly to `writeResolutionEntry()`).

An FDA auditor querying by `actorId = "clinical-service"` misses AE, escalation, and safety
officer entries. `IrbApprovalLedgerWriter` correctly uses `"irb-committee"` with `ActorType.HUMAN`
and is not changed.

### #45 — observer silent failure

Both `SafetyOfficerNotificationListener` and `SponsorNotificationListener` are
`@Transactional @ObservesAsync`. If Panache lookups or the notifier call throws past the
notifier's own catch, the `@ObservesAsync` dispatcher silently swallows it. No fallback entry
is written — indistinguishable in the FDA audit trail from "notification never triggered"
(ICH E6(R3) §5.17 / 21 CFR 312.32 gap).

An end-to-end integration test covering the full safety officer notification chain
(listener → notifier → connector → ledger) is also missing.

---

## Design

### 1. `ClinicalActors` constant (api module)

New class `io.casehub.clinical.api.ClinicalActors`:

```java
public final class ClinicalActors {
    public static final String CLINICAL_SERVICE = "clinical-service";
    private ClinicalActors() {}
}
```

Placed in `api/` so both production code and tests can import it without a `runtime` dependency.

### 2. actorId alignment (#46)

Apply `ClinicalActors.CLINICAL_SERVICE` everywhere the system acts as the recording actor:

| File | Change |
|------|--------|
| `AdverseEventLedgerWriter` | `"system"` → `ClinicalActors.CLINICAL_SERVICE` |
| `AeEscalationLedgerWriter` | `"system"` → `ClinicalActors.CLINICAL_SERVICE` |
| `SafetyOfficerNotificationLedgerWriter` | `"system"` → `ClinicalActors.CLINICAL_SERVICE` |
| `DeviationExpirer` | `"system"` literal in `writeResolutionEntry()` call → constant |
| `DeviationLedgerWriter` | Remove `SYSTEM_ACTOR` field; inline `ClinicalActors.CLINICAL_SERVICE` |

`IrbApprovalLedgerWriter` stays `"irb-committee"` / `ActorType.HUMAN` — unchanged.

Test updates (string assertions change from `"system"` to `"clinical-service"`):
- `AdverseEventLedgerWriterTest`
- `SafetyOfficerNotificationLedgerWriterTest`
- `DeviationLedgerWriterTest` (expiration-job entry)
- `DeviationExpirationJobTest`

### 3. Transaction model for observer fallback

**Why REQUIRES_NEW:** The observers run `@Transactional` (REQUIRED). Catching an exception
inside the method body does not auto-rollback the JTA transaction — but a Hibernate error
may have already marked the session for rollback. The fallback ledger write must use
`REQUIRES_NEW` to suspend whatever outer TX state exists and commit in a fresh transaction.

**Why a separate method, not annotating `writeEntry()`:** The existing
`DefaultSafetyOfficerNotifier.notify()` is `@Transactional(REQUIRES_NEW)` and calls
`writeEntry()` within that context. Adding `REQUIRES_NEW` to `writeEntry()` would create
nested `REQUIRES_NEW` on the success path (one extra TX per notification). A dedicated
`writeObserverFailureEntry()` keeps the success path unchanged.

### 4. `SafetyOfficerNotificationLedgerWriter` — new method

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public void writeObserverFailureEntry(UUID aeId, UUID enrollmentId, UUID siteId, CtcaeGrade grade) {
    writeEntry(aeId, enrollmentId, siteId, grade, null, null, false);
    // connectorId/destination null: error occurred before connector config was reachable
}
```

`delivered=false + connectorId=null` is sufficient to identify observer-level failures
vs connector delivery failures — no schema change needed.

### 5. `SafetyOfficerNotificationListener` — fallback

Add `@Inject SafetyOfficerNotificationLedgerWriter ledgerWriter`. Wrap body in try/catch:

```java
try {
    // existing logic unchanged
} catch (Exception e) {
    Log.errorf(e, "Unexpected error in safety officer notification for AE %s — writing failed ledger entry", event.aeId());
    ledgerWriter.writeObserverFailureEntry(event.aeId(), event.enrollmentId(), event.siteId(), event.grade());
}
```

### 6. `DeviationLedgerWriter` — new method

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public void writeObserverFailureEntry(UUID deviationId, UUID siteId, DeviationSeverity severity, Instant now) {
    var entry = new ProtocolDeviationLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = deviationId;
    entry.sequenceNumber = nextSequenceNumber(deviationId);
    entry.deviationId = deviationId;
    entry.siteId = siteId;
    entry.severity = severity.name();
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = ClinicalActors.CLINICAL_SERVICE;
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = "sponsor-notifier-observer-failed";
    entry.occurredAt = now;
    entry.sponsorNotifiedAt = null;
    ledgerEntryRepository.save(entry);
}
```

All fields from the event record — no entity loading.

### 7. `SponsorNotificationListener` — fallback

Add `@Inject DeviationLedgerWriter deviationLedgerWriter` and `@Inject Clock clock`.
Early-return check stays outside try (filter, not failure). Wrap the rest:

```java
try {
    // existing lookup + notify logic
} catch (Exception e) {
    Log.errorf(e, "Unexpected error in sponsor notification for deviation %s — writing failed ledger entry", event.deviationId());
    deviationLedgerWriter.writeObserverFailureEntry(
        event.deviationId(), event.siteId(), event.severity(), clock.instant());
}
```

### 8. Tests

**`SafetyOfficerNotificationListenerTest` — new test:**
`unexpected_exception_from_notifier_writes_observer_failure_entry()`
- Configure existing `@InjectMock SafetyOfficerNotifier` to `doThrow(RuntimeException)`
- Call `listener.onAeReported(event)`
- Assert no exception propagates from listener
- Inject `LedgerEntryRepository`; query `findLatestBySubjectId(aeId)`, assert `delivered=false`
- `@Transactional` on test method so query works; `REQUIRES_NEW` on writer means entry is already committed

**`SafetyOfficerNotificationIntegrationTest` — new class:**
- `grade3_ae_triggers_slack_notification()`: call listener directly; assert connector sent 1 message; query ledger for `delivered=true`
- `connector_delivery_failure_writes_failed_ledger_entry()`: `slackConnector.setShouldThrow(true)`; assert `delivered=false`
- No `@InjectMock SafetyOfficerNotificationLedgerWriter` — verifies actual DB writes

**`DeviationLedgerWriterTest` — new test:**
`writeObserverFailureEntry_persists_with_null_sponsorNotifiedAt_and_correct_actor()`

---

## Scope boundaries

- No schema changes — no Flyway migrations needed
- `IrbApprovalLedgerWriter` unchanged
- `AeEscalationLedgerWriterTest` — add actorId assertion (currently missing)
- Sponsor listener integration test not added (sponsor path already covered by `SponsorNotificationIntegrationTest`)
