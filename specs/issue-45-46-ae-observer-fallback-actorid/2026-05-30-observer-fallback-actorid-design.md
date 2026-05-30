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

**Out of scope — casehubio/clinical#48:** `AeEscalationListener` and `IrbDecisionListener`
have the same class of problem but more constrained fallback data availability.
`AeEscalationListener` derives critical IDs (aeId, enrollmentId, grade) from a reactive case
context lookup — not from the event record — making the fallback conditional on what was
resolved before the throw. `IrbDecisionListener` needs the loaded `IrbApproval` entity for a
meaningful ledger subject ID. Both require separate design. See #48.

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

`DeviationLedgerWriter` Javadoc update: the comment referencing `SYSTEM_ACTOR` and
`ProtocolDeviationService.CLINICAL_SENDER` must be updated to reference
`ClinicalActors.CLINICAL_SERVICE` as the canonical identity constant.

Test updates (all assertions change from `"system"` to `"clinical-service"`):

| Test file | Note |
|-----------|------|
| `AdverseEventLedgerWriterTest` | actorId assertion line |
| `AeEscalationLedgerWriterTest` | add missing actorId assertion (currently absent) |
| `SafetyOfficerNotificationLedgerWriterTest` | actorId assertion line |
| `DeviationLedgerWriterTest` | expiration-job entry assertion; rename `writeResolutionEntry_expired_setsSystemActor` → `expired_uses_clinical_service_actorId` to avoid implying `"system"` |
| `DeviationExpirationJobTest` | actorId assertion line |

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

**Note on `writeEntry()` transaction contract:** `writeEntry()` carries no `@Transactional`
annotation — it inherits the caller's TX context. When called from `DefaultSafetyOfficerNotifier.notify()`
(REQUIRES_NEW), it runs in that TX. When called from `writeObserverFailureEntry()` (also
REQUIRES_NEW), it inherits that fresh TX. Callers must always provide a transaction.

### 4. `SafetyOfficerNotificationLedgerWriter` — new method

```java
/** Called from SafetyOfficerNotificationListener observer fallback path. */
@Transactional(Transactional.TxType.REQUIRES_NEW)
public void writeObserverFailureEntry(UUID aeId, UUID enrollmentId, UUID siteId, CtcaeGrade grade) {
    writeEntry(aeId, enrollmentId, siteId, grade, null, null, false);
    // connectorId/destination null: error occurred before connector config was reachable
}
```

`delivered=false + connectorId=null` is sufficient to identify observer-level failures
vs connector delivery failures (where connectorId is known) — no schema change needed.

### 5. `SafetyOfficerNotificationListener` — fallback

Add `@Inject SafetyOfficerNotificationLedgerWriter ledgerWriter`. Wrap body in try/catch
with a nested inner catch to handle the case where the DB is also unavailable
(the most likely root cause would cause the fallback write to fail too):

```java
try {
    // existing logic unchanged
} catch (Exception e) {
    Log.errorf(e, "Unexpected error in safety officer notification for AE %s — writing failed ledger entry", event.aeId());
    try {
        ledgerWriter.writeObserverFailureEntry(event.aeId(), event.enrollmentId(), event.siteId(), event.grade());
    } catch (Exception writeEx) {
        Log.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for AE %s", event.aeId());
    }
}
```

The inner catch has nowhere to escalate — the `@ObservesAsync` dispatcher has no retry
mechanism. The `AUDIT GAP:` prefix makes these log lines operator-searchable.

### 6. `DeviationLedgerWriter` — new method

```java
/** Called from SponsorNotificationListener observer fallback path. */
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
Early-return check stays outside try (filter, not failure). Wrap the rest with the
same double-catch pattern:

```java
try {
    // existing lookup + notify logic
} catch (Exception e) {
    Log.errorf(e, "Unexpected error in sponsor notification for deviation %s — writing failed ledger entry", event.deviationId());
    try {
        deviationLedgerWriter.writeObserverFailureEntry(
            event.deviationId(), event.siteId(), event.severity(), clock.instant());
    } catch (Exception writeEx) {
        Log.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for deviation %s", event.deviationId());
    }
}
```

### 8. Tests

**`SafetyOfficerNotificationListenerTest` — new test:**
`unexpected_exception_from_notifier_writes_observer_failure_entry()`
- Configure existing `@InjectMock SafetyOfficerNotifier` to `doThrow(RuntimeException)`
- Call `listener.onAeReported(event)` — must not throw
- Add `@Inject LedgerEntryRepository ledgerEntryRepository` to the test class
- In the test method (annotated `@Transactional`), call `ledgerEntryRepository.findLatestBySubjectId(aeId)`, cast to `SafetyOfficerNotificationLedgerEntry`, assert `delivered=false`
- `REQUIRES_NEW` on `writeObserverFailureEntry` commits the entry before the test query runs, even within the test's `@Transactional` context (READ_COMMITTED sees it)

**`SafetyOfficerNotificationIntegrationTest` — new class:**

Setup:
- `@Inject SafetyOfficerNotificationListener listener` — real bean
- `@Inject TestSlackConnector slackConnector` — real mock connector
- `@Inject LedgerEntryRepository ledgerEntryRepository` — for DB assertions
- `@BeforeEach @Transactional setUp()`: create `ClinicalTrial` (with `safetyOfficerConnectorId="slack"` and `safetyOfficerDestination`) and `TrialSite`. No `AdverseEvent` needed — the listener uses only event-record data (`aeId`, `enrollmentId`, `siteId`, `grade`), not DB-loaded AE fields.
- `slackConnector.reset()` in `@BeforeEach`

Call `listener.onAeReported(event)` **directly** — this is a synchronous CDI call, not `fireAsync`. `DefaultSafetyOfficerNotifier.notify()` runs in `REQUIRES_NEW` but is still synchronous from the test thread. Connector delivery and ledger write both complete before the listener returns. **No Awaitility needed.** (Contrast: `SponsorNotificationIntegrationTest` uses Awaitility because `piResponseListener.process()` fires a CDI async event internally — the delivery happens on a different thread.)

Test methods:
- `grade3_ae_triggers_safety_officer_slack_notification()`: assert `slackConnector.sent()` has 1 message with correct destination; query `ledgerEntryRepository.findLatestBySubjectId(aeId)`, cast, assert `delivered=true`
- `connector_delivery_failure_writes_failed_ledger_entry()`: `slackConnector.setShouldThrow(true)`; assert `delivered=false` in ledger

No `@InjectMock SafetyOfficerNotificationLedgerWriter` — this test verifies actual DB writes.

**Interaction with `SponsorNotificationIntegrationTest`:** After #45, `SponsorNotificationListener` has a try/catch that calls `deviationLedgerWriter.writeObserverFailureEntry(...)`. That test mocks `DeviationLedgerWriter` via `@InjectMock`. Mockito's default for unstubbed void methods is `doNothing()`, so the new method requires no explicit stub and existing tests are unaffected.

**`DeviationLedgerWriterTest` — new test:**
`writeObserverFailureEntry_persists_with_null_sponsorNotifiedAt_and_clinical_service_actorId()`
Verify: `actorId = "clinical-service"`, `actorRole = "sponsor-notifier-observer-failed"`,
`sponsorNotifiedAt = null`, `subjectId = deviationId`.

---

## Scope boundaries

- No schema changes — no Flyway migrations needed
- `IrbApprovalLedgerWriter` unchanged — `"irb-committee"` is a named external actor, not system
- `AeEscalationListener` + `IrbDecisionListener` observer fallback deferred to casehubio/clinical#48
- Sponsor listener integration test not added — sponsor path already covered by `SponsorNotificationIntegrationTest`
