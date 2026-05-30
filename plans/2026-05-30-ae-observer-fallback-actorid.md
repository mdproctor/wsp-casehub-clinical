# AE Observer Fallback + actorId Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix actorId inconsistency across all ledger writers (#46) and add exception fallback with double-catch to SafetyOfficerNotificationListener and SponsorNotificationListener (#45), plus an end-to-end integration test for the safety officer notification path.

**Architecture:** Single-constant `ClinicalActors.CLINICAL_SERVICE` replaces scattered `"system"` literals across all system-actor ledger writers. Observer fallback uses `@Transactional(REQUIRES_NEW)` dedicated `writeObserverFailureEntry()` methods on each ledger writer — separate from the success path — to guarantee the fallback write commits regardless of the outer transaction state. A double-catch pattern prevents the fallback write failure from also being swallowed silently.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache Active Record, JTA/CDI, JUnit 5 / AssertJ / Mockito, `@QuarkusTest` with H2 + XA

---

## File Map

**New files:**
- `api/src/main/java/io/casehub/clinical/api/ClinicalActors.java` — system actor constant
- `runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationIntegrationTest.java` — end-to-end integration test

**Modified files (production):**
- `runtime/src/main/java/io/casehub/clinical/service/AdverseEventLedgerWriter.java` — `"system"` → constant
- `runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java` — `"system"` → constant
- `runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriter.java` — `"system"` → constant; add `writeObserverFailureEntry()`
- `runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationListener.java` — inject ledger writer; add try/catch with double-catch
- `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java` — remove `SYSTEM_ACTOR`; inline constant; update Javadoc; add `writeObserverFailureEntry()`
- `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java` — `"system"` → constant
- `runtime/src/main/java/io/casehub/clinical/service/SponsorNotificationListener.java` — inject ledger writer and clock; add try/catch with double-catch

**Modified files (tests):**
- `runtime/src/test/java/io/casehub/clinical/service/AdverseEventLedgerWriterTest.java` — assertion update
- `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java` — add missing actorId assertion
- `runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriterTest.java` — assertion update; new test for `writeObserverFailureEntry`
- `runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationListenerTest.java` — add `LedgerEntryRepository` injection; new fallback test
- `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java` — rename one test method; new test for `writeObserverFailureEntry`
- `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java` — assertion update
- `runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationListenerTest.java` — add `LedgerEntryRepository` injection; new fallback test

---

## Phase 1: actorId alignment (#46)

---

### Task 1: Create `ClinicalActors` constant

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/ClinicalActors.java`

- [ ] **Step 1: Create the constant class**

```java
package io.casehub.clinical.api;

public final class ClinicalActors {
    public static final String CLINICAL_SERVICE = "clinical-service";
    private ClinicalActors() {}
}
```

- [ ] **Step 2: Build api module to verify it compiles**

```bash
mvn install -pl api --batch-mode
```
Expected: `BUILD SUCCESS`

---

### Task 2: Align `AdverseEventLedgerWriter`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AdverseEventLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventLedgerWriter.java`

- [ ] **Step 1: Update test assertion to expect the correct value (test will now fail)**

In `AdverseEventLedgerWriterTest`, find the line:
```java
assertThat(entry.actorId).isEqualTo("system");
```
Change to:
```java
assertThat(entry.actorId).isEqualTo("clinical-service");
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl runtime -Dtest=AdverseEventLedgerWriterTest --batch-mode
```
Expected: FAIL — `expected: "clinical-service" but was: "system"`

- [ ] **Step 3: Update the writer**

In `AdverseEventLedgerWriter.java`, add the import and change the hardcoded string:

Replace:
```java
        entry.actorId = "system";
```
With:
```java
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
```

Add import at top of file:
```java
import io.casehub.clinical.api.ClinicalActors;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl runtime -Dtest=AdverseEventLedgerWriterTest --batch-mode
```
Expected: PASS

---

### Task 3: Align `AeEscalationLedgerWriter`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java`

- [ ] **Step 1: Add the missing actorId assertion to the existing test (will now fail)**

In `AeEscalationLedgerWriterTest.writeCompletionEntry_persists_correct_fields`, add after the existing assertions:
```java
        assertThat(entry.actorId).isEqualTo("clinical-service");
        assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
```

Add import if not present:
```java
import io.casehub.platform.api.identity.ActorType;
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl runtime -Dtest=AeEscalationLedgerWriterTest --batch-mode
```
Expected: FAIL — `expected: "clinical-service" but was: "system"`

- [ ] **Step 3: Update the writer**

In `AeEscalationLedgerWriter.java`, add import and change the hardcoded string:

Replace:
```java
        entry.actorId = "system";
```
With:
```java
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
```

Add import:
```java
import io.casehub.clinical.api.ClinicalActors;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl runtime -Dtest=AeEscalationLedgerWriterTest --batch-mode
```
Expected: PASS

---

### Task 4: Align `SafetyOfficerNotificationLedgerWriter`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriter.java`

- [ ] **Step 1: Update test assertion (will now fail)**

In `SafetyOfficerNotificationLedgerWriterTest.writeEntry_persists_correct_fields_on_successful_delivery`, find:
```java
        assertThat(entry.actorId).isEqualTo("system");
```
Change to:
```java
        assertThat(entry.actorId).isEqualTo("clinical-service");
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl runtime -Dtest=SafetyOfficerNotificationLedgerWriterTest --batch-mode
```
Expected: FAIL

- [ ] **Step 3: Update the writer**

In `SafetyOfficerNotificationLedgerWriter.java`, add import and change:

Replace:
```java
        entry.actorId = "system";
```
With:
```java
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
```

Add import:
```java
import io.casehub.clinical.api.ClinicalActors;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl runtime -Dtest=SafetyOfficerNotificationLedgerWriterTest --batch-mode
```
Expected: PASS

---

### Task 5: Align `DeviationExpirer` and clean up `DeviationLedgerWriter`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java`

- [ ] **Step 1: Update `DeviationExpirationJobTest` assertion (will fail)**

In `DeviationExpirationJobTest.overdueCommandedDeviationIsMarkedExpired`, find:
```java
        assertThat(entry.actorId).isEqualTo("system");
```
Change to:
```java
        assertThat(entry.actorId).isEqualTo("clinical-service");
```

- [ ] **Step 2: Rename the misleading unit test method in `DeviationLedgerWriterTest`**

The test `writeResolutionEntry_expired_setsSystemActor` tests pass-through behavior (not that actorId must be "system"). Rename to make this clear:

Replace method signature:
```java
    void writeResolutionEntry_expired_setsSystemActor() {
```
With:
```java
    void writeResolutionEntry_expired_stores_provided_actorId() {
```

The assertion `assertThat(entry.actorId).isEqualTo("system")` stays — the test passes "system" explicitly and the writer stores it as-is. This is correct behavior.

- [ ] **Step 3: Run `DeviationExpirationJobTest` to verify step 1 fails**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=DeviationExpirationJobTest --batch-mode
```
Expected: FAIL — `expected: "clinical-service" but was: "system"`

- [ ] **Step 4: Update `DeviationExpirer` to use the constant**

In `DeviationExpirer.java`, add import and change line 49:

Replace:
```java
        ledgerWriter.writeResolutionEntry(d, PiApprovalStatus.EXPIRED,
            "system", ActorType.SYSTEM, "deviation-expiration-job");
```
With:
```java
        ledgerWriter.writeResolutionEntry(d, PiApprovalStatus.EXPIRED,
            ClinicalActors.CLINICAL_SERVICE, ActorType.SYSTEM, "deviation-expiration-job");
```

Add import:
```java
import io.casehub.clinical.api.ClinicalActors;
```

- [ ] **Step 5: Clean up `DeviationLedgerWriter` — remove `SYSTEM_ACTOR`, inline constant, update Javadoc**

In `DeviationLedgerWriter.java`, make these changes:

Add import:
```java
import io.casehub.clinical.api.ClinicalActors;
```

Remove the class-level constant (delete this line):
```java
    static final String SYSTEM_ACTOR = "clinical-service";
```

In `writeCommandEntry`, replace:
```java
        entry.actorId = SYSTEM_ACTOR;
```
With:
```java
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
```

In `writeSponsorNotifiedEntry`, replace:
```java
        entry.actorId = SYSTEM_ACTOR;
```
With:
```java
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
```

Replace the existing class Javadoc comment block (the paragraph starting with `Note: SYSTEM_ACTOR and ProtocolDeviationService.CLINICAL_SENDER are the same string`):

Remove:
```
 * Note: SYSTEM_ACTOR and ProtocolDeviationService.CLINICAL_SENDER are the same string
 * "clinical-service" — both identify the clinical harness in their respective contexts
 * (ledger actor vs qhorus message sender). Keep them in sync if the harness identity changes.
```
Replace with:
```
 * The clinical harness uses {@link ClinicalActors#CLINICAL_SERVICE} as its system actorId
 * across all ledger writers; {@code ProtocolDeviationService.CLINICAL_SENDER} uses the same
 * identity string in the qhorus context — keep both in sync if the harness identity changes.
```

- [ ] **Step 6: Run tests to verify all pass**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest="DeviationExpirationJobTest,DeviationLedgerWriterTest" --batch-mode
```
Expected: PASS

---

### Task 6: Commit #46

- [ ] **Step 1: Run full test suite to confirm no regressions**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode
```
Expected: BUILD SUCCESS, all tests green

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/src/main/java/io/casehub/clinical/api/ClinicalActors.java runtime/src/main/java/io/casehub/clinical/service/AdverseEventLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java runtime/src/test/java/io/casehub/clinical/service/AdverseEventLedgerWriterTest.java runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriterTest.java runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java && git -C /Users/mdproctor/claude/casehub/clinical commit -m "fix: align ledger writer actorId to ClinicalActors.CLINICAL_SERVICE across all system writers

Closes #46 Refs #45"
```

---

## Phase 2: Observer exception fallback (#45)

---

### Task 7: `SafetyOfficerNotificationLedgerWriter.writeObserverFailureEntry`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriter.java`

- [ ] **Step 1: Write the failing test**

In `SafetyOfficerNotificationLedgerWriterTest`, add this test method:

```java
    @Test
    void writeObserverFailureEntry_writes_delivered_false_with_null_connector_fields() {
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any())).thenReturn(Optional.empty());

        writer.writeObserverFailureEntry(aeId, enrollmentId, siteId, CtcaeGrade.GRADE_3);

        SafetyOfficerNotificationLedgerEntry entry = captureEntry();
        assertThat(entry.delivered).isFalse();
        assertThat(entry.connectorId).isNull();
        assertThat(entry.destination).isNull();
        assertThat(entry.aeId).isEqualTo(aeId);
        assertThat(entry.siteId).isEqualTo(siteId);
        assertThat(entry.enrollmentId).isEqualTo(enrollmentId);
        assertThat(entry.ctcaeGrade).isEqualTo("GRADE_3");
        assertThat(entry.actorId).isEqualTo("clinical-service");
        assertThat(entry.sequenceNumber).isEqualTo(1);
    }
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl runtime -Dtest=SafetyOfficerNotificationLedgerWriterTest --batch-mode
```
Expected: FAIL — method does not exist yet

- [ ] **Step 3: Implement `writeObserverFailureEntry`**

In `SafetyOfficerNotificationLedgerWriter.java`, add the method and its import. Add before the `nextSequenceNumber` private method:

```java
    /** Called from SafetyOfficerNotificationListener observer fallback path. Commits in its own REQUIRES_NEW transaction. */
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void writeObserverFailureEntry(UUID aeId, UUID enrollmentId, UUID siteId, CtcaeGrade grade) {
        writeEntry(aeId, enrollmentId, siteId, grade, null, null, false);
    }
```

Add import:
```java
import jakarta.transaction.Transactional;
```

Note: `writeEntry()` has no `@Transactional` — it inherits the REQUIRES_NEW context provided by its caller. `connectorId=null` and `destination=null` distinguish this entry (error before connector config was reached) from connector delivery failures (where both fields are populated).

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl runtime -Dtest=SafetyOfficerNotificationLedgerWriterTest --batch-mode
```
Expected: PASS

---

### Task 8: `SafetyOfficerNotificationListener` fallback

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationListenerTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationListener.java`

- [ ] **Step 1: Add `LedgerEntryRepository` injection to the test class**

In `SafetyOfficerNotificationListenerTest`, add to the class fields (alongside `@Inject SafetyOfficerNotificationListener listener`):

```java
    @Inject io.casehub.ledger.runtime.repository.LedgerEntryRepository ledgerEntryRepository;
```

- [ ] **Step 2: Write the failing test**

Add this test method to `SafetyOfficerNotificationListenerTest`:

```java
    @Test
    @Transactional
    void unexpected_exception_from_notifier_writes_observer_failure_entry() {
        final UUID aeId = UUID.randomUUID();
        final UUID enrollmentId = UUID.randomUUID();
        final AdverseEventReportedEvent event = new AdverseEventReportedEvent(
            aeId, enrollmentId, siteId, CtcaeGrade.GRADE_3, Instant.now());

        org.mockito.Mockito.doThrow(new RuntimeException("injected test failure"))
            .when(safetyOfficerNotifier).notify(org.mockito.ArgumentMatchers.any());

        org.assertj.core.api.Assertions.assertThatCode(() -> listener.onAeReported(event))
            .doesNotThrowAnyException();

        io.casehub.clinical.ledger.SafetyOfficerNotificationLedgerEntry entry =
            (io.casehub.clinical.ledger.SafetyOfficerNotificationLedgerEntry)
            ledgerEntryRepository.findLatestBySubjectId(aeId).orElse(null);
        assertThat(entry).isNotNull();
        assertThat(entry.delivered).isFalse();
        assertThat(entry.connectorId).isNull();
    }
```

Add imports to the test class if not already present:
```java
import io.casehub.clinical.ledger.SafetyOfficerNotificationLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.doThrow;
```

- [ ] **Step 3: Run test to verify it fails**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=SafetyOfficerNotificationListenerTest --batch-mode
```
Expected: FAIL — the listener doesn't catch exceptions yet, so it propagates or no entry is written

- [ ] **Step 4: Implement the fallback in `SafetyOfficerNotificationListener`**

Replace the entire content of `SafetyOfficerNotificationListener.java` with:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.SafetyOfficerNotificationRequest;
import io.casehub.clinical.api.SafetyOfficerNotifier;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.TrialSite;
import io.quarkus.logging.Log;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class SafetyOfficerNotificationListener {

    @Inject SafetyOfficerNotifier notifier;
    @Inject SafetyOfficerNotificationLedgerWriter ledgerWriter;

    @Transactional
    public void onAeReported(@ObservesAsync final AdverseEventReportedEvent event) {
        try {
            if (event.siteId() == null) {
                Log.errorf("AE %s has no siteId — safety officer notification skipped", event.aeId());
                return;
            }
            final TrialSite site = TrialSite.findById(event.siteId());
            if (site == null) {
                Log.warnf("TrialSite %s not found — safety officer notification skipped", event.siteId());
                return;
            }
            final ClinicalTrial trial = ClinicalTrial.findById(site.trialId);
            if (trial == null) {
                Log.warnf("Trial %s not found — safety officer notification skipped", site.trialId);
                return;
            }
            if (trial.safetyOfficerConnectorId == null || trial.safetyOfficerDestination == null) {
                Log.warnf("Trial %s has incomplete safety officer notification config (connectorId=%s, destination=%s) — skipping",
                    site.trialId, trial.safetyOfficerConnectorId, trial.safetyOfficerDestination);
                return;
            }
            notifier.notify(new SafetyOfficerNotificationRequest(
                event.aeId(), event.enrollmentId(), event.siteId(), event.grade(),
                trial.safetyOfficerConnectorId, trial.safetyOfficerDestination));
        } catch (Exception e) {
            Log.errorf(e, "Unexpected error in safety officer notification for AE %s — writing failed ledger entry", event.aeId());
            try {
                ledgerWriter.writeObserverFailureEntry(event.aeId(), event.enrollmentId(), event.siteId(), event.grade());
            } catch (Exception writeEx) {
                Log.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for AE %s", event.aeId());
            }
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=SafetyOfficerNotificationListenerTest --batch-mode
```
Expected: PASS

---

### Task 9: `DeviationLedgerWriter.writeObserverFailureEntry`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java`

- [ ] **Step 1: Write the failing test**

In `DeviationLedgerWriterTest`, add these imports if not already present:
```java
import io.casehub.clinical.api.model.DeviationSeverity;
import java.time.Instant;
```

Add this test method:

```java
    @Test
    void writeObserverFailureEntry_persists_with_null_sponsorNotifiedAt_and_clinical_service_actorId() {
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.empty());

        writer.writeObserverFailureEntry(dev.id, dev.siteId, DeviationSeverity.MINOR, FIXED_INSTANT);

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.actorId).isEqualTo("clinical-service");
        assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
        assertThat(entry.actorRole).isEqualTo("sponsor-notifier-observer-failed");
        assertThat(entry.sponsorNotifiedAt).isNull();
        assertThat(entry.subjectId).isEqualTo(dev.id);
        assertThat(entry.deviationId).isEqualTo(dev.id);
        assertThat(entry.siteId).isEqualTo(dev.siteId);
        assertThat(entry.severity).isEqualTo("MINOR");
        assertThat(entry.entryType).isEqualTo(LedgerEntryType.EVENT);
        assertThat(entry.occurredAt).isEqualTo(FIXED_INSTANT);
        assertThat(entry.sequenceNumber).isEqualTo(1);
    }
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl runtime -Dtest=DeviationLedgerWriterTest --batch-mode
```
Expected: FAIL — method does not exist yet

- [ ] **Step 3: Implement `writeObserverFailureEntry`**

In `DeviationLedgerWriter.java`, add the import and the method. Add before the `baseEntry` private method:

```java
    /** Called from SponsorNotificationListener observer fallback path. Commits in its own REQUIRES_NEW transaction. */
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void writeObserverFailureEntry(UUID deviationId, UUID siteId,
            io.casehub.clinical.api.model.DeviationSeverity severity, java.time.Instant now) {
        var entry = new io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = deviationId;
        entry.sequenceNumber = nextSequenceNumber(deviationId);
        entry.deviationId = deviationId;
        entry.siteId = siteId;
        entry.severity = severity.name();
        entry.entryType = io.casehub.ledger.api.model.LedgerEntryType.EVENT;
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
        entry.actorType = io.casehub.platform.api.identity.ActorType.SYSTEM;
        entry.actorRole = "sponsor-notifier-observer-failed";
        entry.occurredAt = now;
        entry.sponsorNotifiedAt = null;
        ledgerEntryRepository.save(entry);
    }
```

Add import:
```java
import jakarta.transaction.Transactional;
```

Note: Use fully-qualified names in the method or add the imports. If the class already imports `ProtocolDeviationLedgerEntry`, `LedgerEntryType`, `ActorType`, and the model classes, use the short names. Check existing imports in the file and add any that are missing:
```java
import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.platform.api.identity.ActorType;
import jakarta.transaction.Transactional;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl runtime -Dtest=DeviationLedgerWriterTest --batch-mode
```
Expected: PASS

---

### Task 10: `SponsorNotificationListener` fallback

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationListenerTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SponsorNotificationListener.java`

- [ ] **Step 1: Add `LedgerEntryRepository` injection to the test class**

In `SponsorNotificationListenerTest`, add to the class fields:

```java
    @Inject io.casehub.ledger.runtime.repository.LedgerEntryRepository ledgerEntryRepository;
```

- [ ] **Step 2: Write the failing test**

Add this test method to `SponsorNotificationListenerTest`:

```java
    @Test
    @Transactional
    void unexpected_exception_from_notifier_writes_observer_failure_entry() {
        final UUID deviationId = UUID.randomUUID();
        final io.casehub.clinical.api.ProtocolDeviationResolvedEvent event =
            new io.casehub.clinical.api.ProtocolDeviationResolvedEvent(
                deviationId, siteId,
                io.casehub.clinical.api.model.DeviationSeverity.MAJOR,
                io.casehub.clinical.api.model.EscalationRequirement.SPONSOR_NOTIFICATION,
                io.casehub.clinical.api.model.PiApprovalStatus.ESCALATED,
                "INFORMED_CONSENT", "dr-smith@v1");

        org.mockito.Mockito.doThrow(new RuntimeException("injected test failure"))
            .when(sponsorNotifier).notify(org.mockito.ArgumentMatchers.any());

        org.assertj.core.api.Assertions.assertThatCode(() -> listener.onDeviationResolved(event))
            .doesNotThrowAnyException();

        var entries = ledgerEntryRepository.findBySubjectId(deviationId);
        assertThat(entries).hasSize(1);
        io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry entry =
            (io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry) entries.get(0);
        assertThat(entry.actorRole).isEqualTo("sponsor-notifier-observer-failed");
        assertThat(entry.sponsorNotifiedAt).isNull();
        assertThat(entry.actorId).isEqualTo("clinical-service");
    }
```

Add imports to test class:
```java
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.doThrow;
```

- [ ] **Step 3: Run test to verify it fails**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=SponsorNotificationListenerTest --batch-mode
```
Expected: FAIL

- [ ] **Step 4: Implement the fallback in `SponsorNotificationListener`**

Replace the entire content of `SponsorNotificationListener.java` with:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.SponsorNotificationRequest;
import io.casehub.clinical.api.SponsorNotifier;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.TrialSite;
import io.quarkus.logging.Log;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Clock;

@ApplicationScoped
public class SponsorNotificationListener {

    @Inject SponsorNotifier sponsorNotifier;
    @Inject DeviationLedgerWriter deviationLedgerWriter;
    @Inject Clock clock;

    @Transactional
    public void onDeviationResolved(@ObservesAsync ProtocolDeviationResolvedEvent event) {
        if (event.escalationRequirement() != EscalationRequirement.SPONSOR_NOTIFICATION) return;

        try {
            TrialSite site = TrialSite.findById(event.siteId());
            if (site == null) {
                Log.warnf("TrialSite %s not found — sponsor notification skipped", event.siteId());
                return;
            }

            ClinicalTrial trial = ClinicalTrial.findById(site.trialId);
            if (trial == null) {
                Log.warnf("Trial %s not found — sponsor notification skipped", site.trialId);
                return;
            }
            if (trial.sponsorNotificationConnectorId == null || trial.sponsorNotificationDestination == null) {
                Log.warnf("Trial %s has incomplete sponsor notification config (connectorId=%s, destination=%s) — skipping",
                    site.trialId, trial.sponsorNotificationConnectorId, trial.sponsorNotificationDestination);
                return;
            }

            sponsorNotifier.notify(new SponsorNotificationRequest(
                site.trialId,
                event.siteId(),
                event.deviationId(),
                event.deviationType(),
                event.severity(),
                event.terminalStatus(),
                event.piId(),
                trial.sponsorNotificationConnectorId,
                trial.sponsorNotificationDestination
            ));
        } catch (Exception e) {
            Log.errorf(e, "Unexpected error in sponsor notification for deviation %s — writing failed ledger entry", event.deviationId());
            try {
                deviationLedgerWriter.writeObserverFailureEntry(
                    event.deviationId(), event.siteId(), event.severity(), clock.instant());
            } catch (Exception writeEx) {
                Log.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for deviation %s", event.deviationId());
            }
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=SponsorNotificationListenerTest --batch-mode
```
Expected: PASS

---

### Task 11: `SafetyOfficerNotificationIntegrationTest`

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationIntegrationTest.java`

**Setup notes:**
- `TestSlackConnector` is `@Mock @Singleton` — beans are shared across `@QuarkusTest` instances. Call `slackConnector.reset()` in `@BeforeEach`.
- Call `listener.onAeReported(event)` directly (synchronous). `DefaultSafetyOfficerNotifier.notify()` runs in `REQUIRES_NEW` but is still synchronous. Ledger write completes before the listener returns. **No Awaitility needed.**
- Entity setup: only `ClinicalTrial` + `TrialSite` needed. The listener reads site+trial from DB and passes IDs directly from the event record. No `AdverseEvent` entity is needed.
- Query the ledger after the listener call in a `@Transactional` test method. The `REQUIRES_NEW` write is already committed so it's visible.

- [ ] **Step 1: Create the integration test**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.SiteStatus;
import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.clinical.api.model.TrialStatus;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.ledger.SafetyOfficerNotificationLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class SafetyOfficerNotificationIntegrationTest {

    @Inject SafetyOfficerNotificationListener listener;
    @Inject TestSlackConnector slackConnector;
    @Inject LedgerEntryRepository ledgerEntryRepository;

    private UUID siteId;

    @BeforeEach
    @Transactional
    void setUp() {
        slackConnector.reset();

        UUID trialId = UUID.randomUUID();
        siteId = UUID.randomUUID();

        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId;
        trial.protocolId = "ONCO-INT-001";
        trial.phase = TrialPhase.PHASE_III;
        trial.sponsor = "Integration Pharma";
        trial.targetEnrollment = 100;
        trial.status = TrialStatus.ACTIVE;
        trial.safetyOfficerConnectorId = "slack";
        trial.safetyOfficerDestination = "https://hooks.slack.com/safety-officer-integration-test";
        trial.persist();

        TrialSite site = new TrialSite();
        site.id = siteId;
        site.trialId = trialId;
        site.investigatorId = "dr-jones@v1";
        site.status = SiteStatus.ACTIVE;
        site.persist();
    }

    @Test
    @Transactional
    void grade3_ae_triggers_safety_officer_slack_notification() {
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        AdverseEventReportedEvent event = new AdverseEventReportedEvent(
            aeId, enrollmentId, siteId, CtcaeGrade.GRADE_3, Instant.now());

        listener.onAeReported(event);

        assertThat(slackConnector.sent()).hasSize(1);
        assertThat(slackConnector.sent().get(0).destination())
            .isEqualTo("https://hooks.slack.com/safety-officer-integration-test");
        assertThat(slackConnector.sent().get(0).body()).contains("Grade 3");

        SafetyOfficerNotificationLedgerEntry entry =
            (SafetyOfficerNotificationLedgerEntry)
            ledgerEntryRepository.findLatestBySubjectId(aeId).orElse(null);
        assertThat(entry).isNotNull();
        assertThat(entry.delivered).isTrue();
        assertThat(entry.aeId).isEqualTo(aeId);
        assertThat(entry.siteId).isEqualTo(siteId);
        assertThat(entry.connectorId).isEqualTo("slack");
        assertThat(entry.destination).isEqualTo("https://hooks.slack.com/safety-officer-integration-test");
    }

    @Test
    @Transactional
    void connector_delivery_failure_writes_failed_ledger_entry() {
        slackConnector.setShouldThrow(true);

        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        AdverseEventReportedEvent event = new AdverseEventReportedEvent(
            aeId, enrollmentId, siteId, CtcaeGrade.GRADE_4, Instant.now());

        listener.onAeReported(event);

        assertThat(slackConnector.sent()).isEmpty();

        SafetyOfficerNotificationLedgerEntry entry =
            (SafetyOfficerNotificationLedgerEntry)
            ledgerEntryRepository.findLatestBySubjectId(aeId).orElse(null);
        assertThat(entry).isNotNull();
        assertThat(entry.delivered).isFalse();
        assertThat(entry.connectorId).isEqualTo("slack");
    }
}
```

- [ ] **Step 2: Run the integration test**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=SafetyOfficerNotificationIntegrationTest --batch-mode
```
Expected: PASS

---

### Task 12: Full test run and commit #45

- [ ] **Step 1: Run the full test suite**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode
```
Expected: BUILD SUCCESS, all tests green

- [ ] **Step 2: Verify `SponsorNotificationIntegrationTest` still passes (Mockito doNothing interaction)**

The `SponsorNotificationIntegrationTest` mocks `DeviationLedgerWriter` via `@InjectMock`. After #45, `SponsorNotificationListener` calls `deviationLedgerWriter.writeObserverFailureEntry(...)` in the catch block. Mockito's default for unstubbed void methods is `doNothing()` — no stub needed. Confirm it passes as part of the full suite above.

- [ ] **Step 3: Commit #45**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/SafetyOfficerNotificationListener.java runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/SponsorNotificationListener.java runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationLedgerWriterTest.java runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationListenerTest.java runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationListenerTest.java runtime/src/test/java/io/casehub/clinical/service/SafetyOfficerNotificationIntegrationTest.java && git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: observer exception fallback for SafetyOfficerNotificationListener and SponsorNotificationListener

- Add writeObserverFailureEntry (REQUIRES_NEW) to SafetyOfficerNotificationLedgerWriter and DeviationLedgerWriter
- Wrap observer bodies in double try/catch — inner catch logs AUDIT GAP if fallback write also fails
- Fallback uses event-record data only (no entity loading) so it works regardless of which lookup failed
- Add SafetyOfficerNotificationIntegrationTest covering happy path and connector failure
- Add unit tests for both fallback paths

Closes #45"
```
