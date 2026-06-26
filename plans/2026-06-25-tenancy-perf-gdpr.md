# #89 + #87 + #79 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire MissingTenancyExceptionMapper (#89), fix listener transaction boundaries (#87), add GDPR Art.17 patient-scoped erasure (#79).

**Architecture:** Three independent concerns on one branch. #89 is a JAX-RS ExceptionMapper. #87 removes `@Transactional` from three `@ObservesAsync` listeners and ensures writes use REQUIRES_NEW. #79 makes `withdraw()` idempotent, enables erasure receipts, adds a patient-scoped `DELETE` endpoint.

**Tech Stack:** Java 21 / Quarkus 3.32.2 / Panache Active Record / JTA XA / Flyway

**Spec:** `specs/2026-06-25-tenancy-perf-gdpr-design.md` (Rev 2)

## Global Constraints

- Java 21 language level on Java 26 JVM
- Use `mvn` not `./mvnw`
- `api/` must be installed before `runtime/` tests: `mvn install -pl api --batch-mode`
- Tests use `drop-and-create` + Flyway disabled; production uses Flyway
- Two datasources: default (domain entities) + qhorus (ledger/qhorus entities)
- LedgerEntry subclasses in `io.casehub.clinical.ledger` package, never `io.casehub.clinical.entity`
- `@RolesAllowed` with `ClinicalGroups` constants from `api/`
- `quarkus.security.deny-unannotated-members=true` — new REST endpoints MUST have `@RolesAllowed`
- All commits reference issues: `Refs #89`, `Refs #87`, `Refs #79`
- IntelliJ MCP for all rename/move/find-references operations

---

### Task 1: MissingTenancyExceptionMapper (#89)

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/MissingTenancyExceptionMapper.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/MissingTenancyExceptionMapperTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.identity.MissingTenancyException` (platform-api JAR — `actorId()` accessor)
- Produces: JAX-RS `ExceptionMapper<MissingTenancyException>` → HTTP 400 + JSON body

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.clinical.resource;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.platform.api.identity.MissingTenancyException;
import jakarta.ws.rs.core.Response;
import org.junit.jupiter.api.Test;

class MissingTenancyExceptionMapperTest {

    private final MissingTenancyExceptionMapper mapper = new MissingTenancyExceptionMapper();

    @Test
    void maps_to_400_with_json_body() {
        var exception = new MissingTenancyException("user-abc");
        Response response = mapper.toResponse(exception);

        assertThat(response.getStatus()).isEqualTo(400);
        @SuppressWarnings("unchecked")
        var body = (java.util.Map<String, String>) response.getEntity();
        assertThat(body.get("error")).isEqualTo("missing_tenancy_claim");
        assertThat(body.get("actorId")).isEqualTo("user-abc");
        assertThat(body.get("message")).contains("tenancyId");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=MissingTenancyExceptionMapperTest --batch-mode`
Expected: compilation failure — `MissingTenancyExceptionMapper` does not exist.

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.clinical.resource;

import io.casehub.platform.api.identity.MissingTenancyException;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;
import java.util.Map;

@Provider
public class MissingTenancyExceptionMapper implements ExceptionMapper<MissingTenancyException> {

    @Override
    public Response toResponse(MissingTenancyException exception) {
        return Response.status(Response.Status.BAD_REQUEST)
                .type(MediaType.APPLICATION_JSON_TYPE)
                .entity(Map.of(
                        "error", "missing_tenancy_claim",
                        "message", "JWT does not contain required tenancyId claim",
                        "actorId", exception.actorId()))
                .build();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=MissingTenancyExceptionMapperTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat: add MissingTenancyExceptionMapper — maps to HTTP 400

OidcCurrentPrincipal.tenancyId() throws MissingTenancyException when
the JWT lacks the tenancyId claim. Without this mapper RESTEasy returns
500. Platform#115 tracks consolidation to casehub-platform-oidc.

Refs casehubio/clinical#89
```

---

### Task 2: SusarOversightListener + AeEscalation TX Fix (#87, parts 1–2)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightListener.java` — remove `@Transactional`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java` — remove `@Transactional`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java` — add `@Transactional(REQUIRES_NEW)` to both public methods

**Interfaces:**
- Consumes: `CaseInstanceRepository.findByUuid(UUID, String)` returning `Uni<CaseInstance>` (unchanged)
- Produces: Same external behavior. Internal: reactive call no longer holds JDBC connection.

- [ ] **Step 1: Run existing tests to establish baseline**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest="SusarOversightListener*,AeEscalationListener*,AeEscalationLedgerWriter*" --batch-mode`
Expected: all tests PASS (baseline before changes).

- [ ] **Step 2: Remove `@Transactional` from SusarOversightListener**

In `SusarOversightListener.java`, remove the `@Transactional` annotation from `onCaseLifecycle()` and the `jakarta.transaction.Transactional` import.

- [ ] **Step 3: Add `@Transactional(REQUIRES_NEW)` to AeEscalationLedgerWriter**

In `AeEscalationLedgerWriter.java`, add to both methods:

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public void writeCompletionEntry(...) { ... }

@Transactional(Transactional.TxType.REQUIRES_NEW)
public void writeObserverFailureEntry(...) { ... }
```

Add `import jakarta.transaction.Transactional;` to the imports.

- [ ] **Step 4: Remove `@Transactional` from AeEscalationListener**

In `AeEscalationListener.java`, remove `@Transactional` from `onCaseLifecycle()` and the `jakarta.transaction.Transactional` import.

- [ ] **Step 5: Run all affected tests**

Run: `mvn test -pl runtime -Dtest="SusarOversightListener*,AeEscalationListener*,AeEscalationLedgerWriter*" --batch-mode`
Expected: all PASS. The status updaters use REQUIRES_NEW (unchanged), the ledger writer now uses REQUIRES_NEW, and tests use in-memory stores where the change is transparent.

- [ ] **Step 6: Commit**

```
perf: remove outer @Transactional from SusarOversight + AeEscalation listeners

SusarOversightListener: outer TX was pointless — only write is
REQUIRES_NEW on SusarOversightStatusUpdater.

AeEscalationListener: outer TX removed; AeEscalationLedgerWriter
methods promoted to REQUIRES_NEW. Preserves FDA gap design: status
commits first, ledger commits independently, failure entry on error.

Reactive .await() no longer holds a JDBC connection from the Agroal
pool during the 5s timeout.

Refs casehubio/clinical#87
```

---

### Task 3: ProtocolAmendmentStatusUpdater + Listener Refactor (#87, part 3)

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/ProtocolAmendmentStatusUpdater.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/ProtocolAmendmentStatusUpdaterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ProtocolAmendmentListener.java` — remove `@Transactional`, remove switch, delegate to updater
- Modify: `runtime/src/test/java/io/casehub/clinical/service/ProtocolAmendmentListenerTest.java` — adjust for delegation

**Interfaces:**
- Consumes: `ProtocolAmendmentLedgerWriter.writeResolutionEntry(ProtocolAmendment)` (unchanged, no `@Transactional` — participates in caller's TX)
- Produces: `ProtocolAmendmentStatusUpdater.applyRecommendation(UUID amendmentId, String recommendation)` — `@Transactional(REQUIRES_NEW)`

- [ ] **Step 1: Write the updater test**

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

import io.casehub.clinical.api.model.AmendmentCaseStatus;
import io.casehub.clinical.api.model.ProtocolAmendmentStatus;
import io.casehub.clinical.api.spi.AmendmentRecommendation;
import io.casehub.clinical.entity.ProtocolAmendment;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class ProtocolAmendmentStatusUpdaterTest {

    @Inject ProtocolAmendmentStatusUpdater updater;
    @InjectMock ProtocolAmendmentLedgerWriter ledgerWriter;

    UUID amendmentId;

    @BeforeEach
    @Transactional
    void setup() {
        amendmentId = UUID.randomUUID();
        ProtocolAmendment a = new ProtocolAmendment();
        a.id = amendmentId;
        a.trialId = UUID.randomUUID();
        a.proposedChange = "Dose escalation";
        a.status = ProtocolAmendmentStatus.PROPOSED;
        a.amendmentCaseStatus = AmendmentCaseStatus.REQUESTED;
        a.tenantId = "default";
        a.proposedAt = Instant.now();
        a.persist();
    }

    @Test
    void proceed_sets_APPROVED_COMPLETED_and_writes_ledger() {
        updater.applyRecommendation(amendmentId, "PROCEED");

        ProtocolAmendment a = findAmendment(amendmentId);
        assertThat(a.supervisorRecommendation).isEqualTo(AmendmentRecommendation.PROCEED);
        assertThat(a.status).isEqualTo(ProtocolAmendmentStatus.APPROVED);
        assertThat(a.amendmentCaseStatus).isEqualTo(AmendmentCaseStatus.COMPLETED);
        verify(ledgerWriter).writeResolutionEntry(any());
    }

    @Test
    void halt_sets_HALTED_COMPLETED() {
        updater.applyRecommendation(amendmentId, "HALT");

        ProtocolAmendment a = findAmendment(amendmentId);
        assertThat(a.supervisorRecommendation).isEqualTo(AmendmentRecommendation.HALT);
        assertThat(a.status).isEqualTo(ProtocolAmendmentStatus.HALTED);
        assertThat(a.amendmentCaseStatus).isEqualTo(AmendmentCaseStatus.COMPLETED);
    }

    @Test
    void refer_to_dsmb_sets_SUPERVISED_COMPLETED() {
        updater.applyRecommendation(amendmentId, "REFER_TO_DSMB");

        ProtocolAmendment a = findAmendment(amendmentId);
        assertThat(a.supervisorRecommendation).isEqualTo(AmendmentRecommendation.REFER_TO_DSMB);
        assertThat(a.status).isEqualTo(ProtocolAmendmentStatus.SUPERVISED);
        assertThat(a.amendmentCaseStatus).isEqualTo(AmendmentCaseStatus.COMPLETED);
    }

    @Test
    void unknown_recommendation_sets_FAILED_and_leaves_supervisorRecommendation_null() {
        updater.applyRecommendation(amendmentId, "UNKNOWN_VALUE");

        ProtocolAmendment a = findAmendment(amendmentId);
        assertThat(a.supervisorRecommendation).isNull();
        assertThat(a.amendmentCaseStatus).isEqualTo(AmendmentCaseStatus.FAILED);
        verifyNoInteractions(ledgerWriter);
    }

    @Test
    void idempotent_when_supervisorRecommendation_already_set() {
        updater.applyRecommendation(amendmentId, "PROCEED");
        reset(ledgerWriter);

        updater.applyRecommendation(amendmentId, "HALT");

        ProtocolAmendment a = findAmendment(amendmentId);
        assertThat(a.supervisorRecommendation).isEqualTo(AmendmentRecommendation.PROCEED);
        verifyNoInteractions(ledgerWriter);
    }

    @Test
    void unknown_amendmentId_returns_silently() {
        updater.applyRecommendation(UUID.randomUUID(), "PROCEED");
        verifyNoInteractions(ledgerWriter);
    }

    @Transactional
    ProtocolAmendment findAmendment(UUID id) {
        return ProtocolAmendment.findById(id);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=ProtocolAmendmentStatusUpdaterTest --batch-mode`
Expected: compilation failure — `ProtocolAmendmentStatusUpdater` does not exist.

- [ ] **Step 3: Write the updater**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.AmendmentCaseStatus;
import io.casehub.clinical.api.model.ProtocolAmendmentStatus;
import io.casehub.clinical.api.spi.AmendmentRecommendation;
import io.casehub.clinical.entity.ProtocolAmendment;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ProtocolAmendmentStatusUpdater {

    private static final Logger LOG = Logger.getLogger(ProtocolAmendmentStatusUpdater.class);

    @Inject ProtocolAmendmentLedgerWriter ledgerWriter;

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void applyRecommendation(UUID amendmentId, String recommendation) {
        ProtocolAmendment amendment = ProtocolAmendment.findById(amendmentId);
        if (amendment == null) return;
        if (amendment.supervisorRecommendation != null) return;

        switch (recommendation) {
            case "PROCEED" -> {
                amendment.supervisorRecommendation = AmendmentRecommendation.PROCEED;
                amendment.status = ProtocolAmendmentStatus.APPROVED;
                amendment.amendmentCaseStatus = AmendmentCaseStatus.COMPLETED;
                ledgerWriter.writeResolutionEntry(amendment);
            }
            case "HALT" -> {
                amendment.supervisorRecommendation = AmendmentRecommendation.HALT;
                amendment.status = ProtocolAmendmentStatus.HALTED;
                amendment.amendmentCaseStatus = AmendmentCaseStatus.COMPLETED;
                ledgerWriter.writeResolutionEntry(amendment);
            }
            case "REFER_TO_DSMB" -> {
                amendment.supervisorRecommendation = AmendmentRecommendation.REFER_TO_DSMB;
                amendment.status = ProtocolAmendmentStatus.SUPERVISED;
                amendment.amendmentCaseStatus = AmendmentCaseStatus.COMPLETED;
                ledgerWriter.writeResolutionEntry(amendment);
            }
            default -> {
                LOG.errorf("unknown recommendation '%s' for amendmentId=%s — marking FAILED",
                        recommendation, amendmentId);
                amendment.amendmentCaseStatus = AmendmentCaseStatus.FAILED;
            }
        }
    }
}
```

- [ ] **Step 4: Run updater tests**

Run: `mvn test -pl runtime -Dtest=ProtocolAmendmentStatusUpdaterTest --batch-mode`
Expected: all PASS.

- [ ] **Step 5: Refactor ProtocolAmendmentListener to delegate**

Replace `ProtocolAmendmentListener.java`:

```java
package io.casehub.clinical.service;

import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.time.Duration;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ProtocolAmendmentListener {

    private static final Logger LOG = Logger.getLogger(ProtocolAmendmentListener.class);
    private static final Duration LOOKUP_TIMEOUT = Duration.ofSeconds(5);

    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject ProtocolAmendmentStatusUpdater statusUpdater;

    public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (!"GoalReached".equals(event.eventType()) && !"CaseCompleted".equals(event.eventType())) return;

        var instance = caseInstanceRepository
            .findByUuid(event.caseId(), event.tenancyId())
            .await().atMost(LOOKUP_TIMEOUT);
        if (instance == null) return;

        Object amendmentIdObj = instance.getCaseContext().getPath("amendmentId");
        if (amendmentIdObj == null) return;

        UUID amendmentId;
        try {
            amendmentId = UUID.fromString(amendmentIdObj.toString());
        } catch (IllegalArgumentException e) {
            LOG.warnf("ProtocolAmendmentListener: invalid amendmentId: %s", amendmentIdObj);
            return;
        }

        Object recObj = instance.getCaseContext().getPath("advisorRecommendation");
        if (recObj == null) {
            LOG.errorf("ProtocolAmendmentListener: advisorRecommendation absent from case context for amendmentId=%s " +
                "— amendment stays at current status; audit gap", amendmentId);
            return;
        }

        statusUpdater.applyRecommendation(amendmentId, recObj.toString());
    }
}
```

- [ ] **Step 6: Update ProtocolAmendmentListenerTest — inject updater instead of ledgerWriter**

In `ProtocolAmendmentListenerTest.java`:
- Remove `@InjectMock ProtocolAmendmentLedgerWriter ledgerWriter`
- Add `@InjectMock ProtocolAmendmentStatusUpdater statusUpdater` (stub `applyRecommendation` in `@BeforeEach` to be a no-op — the updater tests cover the write logic)
- Update `verify(ledgerWriter).writeResolutionEntry(any())` → `verify(statusUpdater).applyRecommendation(eq(amendmentId), eq("PROCEED"))` (and similar for each test)
- The listener tests now verify delegation, not write behavior. Write behavior is tested in `ProtocolAmendmentStatusUpdaterTest`.

Alternatively: the existing `ProtocolAmendmentListenerTest` already mocks `CaseInstanceRepository` and calls `listener.onCaseLifecycle()` directly. Since the updater has its own REQUIRES_NEW transaction, and the mock `statusUpdater` won't actually do Panache writes, the tests can simply verify the correct `applyRecommendation` call was made.

- [ ] **Step 7: Run all listener + updater tests**

Run: `mvn test -pl runtime -Dtest="ProtocolAmendmentListener*,ProtocolAmendmentStatusUpdater*" --batch-mode`
Expected: all PASS.

- [ ] **Step 8: Commit**

```
refactor: extract ProtocolAmendmentStatusUpdater — single switch owner

Move all recommendation handling (PROCEED, HALT, REFER_TO_DSMB, default)
into ProtocolAmendmentStatusUpdater with REQUIRES_NEW. Listener is now
filter → extract → delegate. Reactive .await() no longer holds a JDBC
connection.

Refs casehubio/clinical#87
```

---

### Task 4: WithdrawalResult + Idempotent withdraw() (#79, prerequisite)

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/WithdrawalResult.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ConsentWithdrawalService.java` — change return type from `void` to `WithdrawalResult`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java` — check result instead of catching exception
- Delete: `runtime/src/main/java/io/casehub/clinical/service/ConsentAlreadyWithdrawnException.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/ConsentWithdrawalServiceTest.java` — update double-withdraw test

**Interfaces:**
- Consumes: nothing new
- Produces: `WithdrawalResult withdraw(UUID enrollmentId, String tenantId)` — returns `WITHDRAWN` or `ALREADY_WITHDRAWN` instead of throwing

- [ ] **Step 1: Create the WithdrawalResult enum**

```java
package io.casehub.clinical.service;

public enum WithdrawalResult {
    WITHDRAWN,
    ALREADY_WITHDRAWN
}
```

- [ ] **Step 2: Update ConsentWithdrawalServiceTest — change double-withdraw assertion**

In `ConsentWithdrawalServiceTest.java`, replace `withdraw_throws_on_already_withdrawn`:

```java
@Test
void withdraw_returns_ALREADY_WITHDRAWN_on_double_call() {
    UUID enrollmentId = persistEnrollment("patient-xyz");
    setWithdrawn(enrollmentId);

    WithdrawalResult result = service.withdraw(enrollmentId, "default");
    assertThat(result).isEqualTo(WithdrawalResult.ALREADY_WITHDRAWN);
}
```

Update the happy-path test to assert the return value:

```java
@Test
void withdraw_sets_both_statuses_pseudonymizes_patientId_sets_withdrawnAt() {
    String originalPatientId = "patient-mrn-12345";
    UUID enrollmentId = persistEnrollment(originalPatientId);

    WithdrawalResult result = service.withdraw(enrollmentId, "default");
    assertThat(result).isEqualTo(WithdrawalResult.WITHDRAWN);
    // ... rest of assertions unchanged
}
```

Remove the `ConsentAlreadyWithdrawnException` import.

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=ConsentWithdrawalServiceTest --batch-mode`
Expected: compilation failure — `withdraw()` still returns `void`.

- [ ] **Step 4: Change withdraw() return type**

In `ConsentWithdrawalService.java`:

Change the method signature from `public void withdraw(...)` to `public WithdrawalResult withdraw(...)`.

Replace the exception throw:
```java
// Before:
if (enrollment.consentStatus == ConsentStatus.WITHDRAWN) {
    throw new ConsentAlreadyWithdrawnException(enrollmentId);
}

// After:
if (enrollment.consentStatus == ConsentStatus.WITHDRAWN) {
    return WithdrawalResult.ALREADY_WITHDRAWN;
}
```

Add `return WithdrawalResult.WITHDRAWN;` at the end of the method.

Remove the `ConsentAlreadyWithdrawnException` import.

- [ ] **Step 5: Update PatientResource.withdrawConsent()**

In `PatientResource.java`, replace the try/catch block in `withdrawConsent()`:

```java
// Before:
try {
    consentWithdrawalService.withdraw(enrollmentId, principal.tenancyId());
    return Response.noContent().build();
} catch (ConsentAlreadyWithdrawnException e) {
    return Response.status(Response.Status.CONFLICT).entity(e.getMessage()).build();
} catch (PatientEnrollmentNotFoundException e) {
    return Response.status(Response.Status.NOT_FOUND).build();
}

// After:
try {
    WithdrawalResult result = consentWithdrawalService.withdraw(enrollmentId, principal.tenancyId());
    if (result == WithdrawalResult.ALREADY_WITHDRAWN) {
        return Response.status(Response.Status.CONFLICT)
                .entity("Consent already withdrawn for enrollment " + enrollmentId).build();
    }
    return Response.noContent().build();
} catch (PatientEnrollmentNotFoundException e) {
    return Response.status(Response.Status.NOT_FOUND).build();
}
```

Remove the `ConsentAlreadyWithdrawnException` import. Add the `WithdrawalResult` import.

- [ ] **Step 6: Delete ConsentAlreadyWithdrawnException.java**

Delete `runtime/src/main/java/io/casehub/clinical/service/ConsentAlreadyWithdrawnException.java`.

- [ ] **Step 7: Run all affected tests**

Run: `mvn test -pl runtime -Dtest="ConsentWithdrawalService*,PatientResource*" --batch-mode`
Expected: all PASS.

- [ ] **Step 8: Commit**

```
refactor: make withdraw() idempotent — return WithdrawalResult

Replace ConsentAlreadyWithdrawnException with a WithdrawalResult enum.
"Already withdrawn" is a completed operation, not an error. The return
type fixes a JTA composition problem: RuntimeException from a REQUIRED
method marks the outer TX rollback-only before the caller can catch it.

Refs casehubio/clinical#79
```

---

### Task 5: Erasure Receipt Config + Flyway + receiptEntryId (#79)

**Files:**
- Modify: `runtime/src/main/resources/application.properties` — enable erasure receipt, add Flyway path
- Modify: `runtime/src/test/resources/application.properties` — enable erasure receipt
- Create: `runtime/src/main/resources/db/migration/qhorus/V2028__consent_withdrawal_receipt_entry_id.sql`
- Modify: `runtime/src/main/java/io/casehub/clinical/ledger/ConsentWithdrawalLedgerEntry.java` — add `receiptEntryId` column
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ConsentWithdrawalService.java` — set `receiptEntryId` from `ErasureResult`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/ConsentWithdrawalServiceTest.java` — verify receipt linkage

**Interfaces:**
- Consumes: `LedgerErasureService.ErasureResult.receiptEntryId()` returning `Optional<UUID>`
- Produces: `ConsentWithdrawalLedgerEntry.receiptEntryId` — nullable UUID linking to foundation `ErasureReceiptLedgerEntry`

- [ ] **Step 1: Add config — production application.properties**

Add to `runtime/src/main/resources/application.properties`:

```properties
casehub.ledger.erasure-receipt.enabled=true
```

Update the qhorus Flyway locations line to add `classpath:db/ledger/migration`:

```properties
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus,classpath:db/engine-ledger/migration,classpath:db/ledger/migration
```

- [ ] **Step 2: Add config — test application.properties**

Add to `runtime/src/test/resources/application.properties`:

```properties
casehub.ledger.erasure-receipt.enabled=true
```

- [ ] **Step 3: Create Flyway migration**

Create `runtime/src/main/resources/db/migration/qhorus/V2028__consent_withdrawal_receipt_entry_id.sql`:

```sql
ALTER TABLE consent_withdrawal_ledger_entry ADD COLUMN receipt_entry_id UUID;
```

- [ ] **Step 4: Add receiptEntryId column to ConsentWithdrawalLedgerEntry**

In `ConsentWithdrawalLedgerEntry.java`, add after the `memoriesErased` field:

```java
@Column(name = "receipt_entry_id")
public UUID receiptEntryId;
```

Do NOT add `receiptEntryId` to `domainContentBytes()` — it is post-erasure metadata, not identity-determining content (per ARC42STORIES.MD design decision).

- [ ] **Step 5: Set receiptEntryId in ConsentWithdrawalService**

In `ConsentWithdrawalService.withdraw()`, after the `entry.ledgerEntriesAffected = erasureResult.affectedEntryCount();` line, add:

```java
entry.receiptEntryId = erasureResult.receiptEntryId().orElse(null);
```

- [ ] **Step 6: Add test for receipt linkage**

Add to `ConsentWithdrawalServiceTest.java`:

```java
@Test
void withdraw_links_receipt_entry_id_when_erasure_receipt_enabled() {
    UUID enrollmentId = persistEnrollment("patient-receipt-test");

    service.withdraw(enrollmentId, "default");

    var entry = ledgerEntryRepository.findLatestBySubjectId(enrollmentId, "default");
    assertThat(entry).isPresent();
    var withdrawal = (ConsentWithdrawalLedgerEntry) entry.get();
    // receiptEntryId is set when erasure-receipt.enabled=true (test config)
    // With InMemoryLedgerEntryRepository, the receipt may not be persisted the same
    // way — assert non-null if the in-memory implementation supports it, or assert
    // the field exists and is settable.
    assertThat(withdrawal.receiptEntryId).as("receiptEntryId should be set when erasure receipt is enabled").isNotNull();
}
```

Note: This test depends on whether `InMemoryLedgerEntryRepository` supports the full `LedgerErasureService` flow. If it doesn't write receipts in test mode, adjust the assertion to verify the field is populated in the `ConsentWithdrawalLedgerEntry` entity (the service sets it from `ErasureResult.receiptEntryId()`). Verify at implementation time.

- [ ] **Step 7: Run tests**

Run: `mvn test -pl runtime -Dtest="ConsentWithdrawalService*" --batch-mode`
Expected: PASS.

- [ ] **Step 8: Commit**

```
feat: enable erasure receipts + receiptEntryId traceability

Enable casehub.ledger.erasure-receipt.enabled=true so foundation
ErasureReceiptLedgerEntry is written on each erase() call. Add
classpath:db/ledger/migration to qhorus Flyway (V1010). Link
ConsentWithdrawalLedgerEntry to the foundation receipt via receiptEntryId
(V2028 migration). receiptEntryId is post-erasure metadata — excluded
from domainContentBytes() per ARC42STORIES Merkle design decision.

Refs casehubio/clinical#79
```

---

### Task 6: GdprErasureService + GdprErasureResource (#79)

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/PatientNotFoundException.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/GdprErasureService.java`
- Create: `runtime/src/main/java/io/casehub/clinical/resource/GdprErasureResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/GdprErasureServiceTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/GdprErasureResourceTest.java`

**Interfaces:**
- Consumes: `ConsentWithdrawalService.withdraw(UUID, String)` returning `WithdrawalResult` (from Task 4)
- Produces: `DELETE /api/gdpr/erasure/patients/{patientId}` → 204 or 404

- [ ] **Step 1: Create PatientNotFoundException**

```java
package io.casehub.clinical.service;

/**
 * No active (non-withdrawn) enrollments found for this patientId.
 *
 * <p>After a successful GDPR erasure, the original patientId is pseudonymized.
 * Retries with the original ID will receive this exception. This is by design:
 * erased patients are unidentifiable.
 */
public class PatientNotFoundException extends RuntimeException {
    private final String patientId;

    public PatientNotFoundException(String patientId) {
        super("No active enrollments found for patient: " + patientId);
        this.patientId = patientId;
    }

    public String patientId() { return patientId; }
}
```

- [ ] **Step 2: Write GdprErasureServiceTest**

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.casehub.clinical.entity.PatientEnrollment;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class GdprErasureServiceTest {

    @Inject GdprErasureService erasureService;

    @Test
    void erasePatient_withdraws_single_enrollment() {
        String patientId = "GDPR-PAT-" + UUID.randomUUID();
        UUID enrollmentId = persistEnrollment(patientId);

        int count = erasureService.erasePatient(patientId, "default");
        assertThat(count).isEqualTo(1);

        PatientEnrollment updated = findEnrollment(enrollmentId);
        assertThat(updated.consentStatus).isEqualTo(ConsentStatus.WITHDRAWN);
        assertThat(updated.patientId).startsWith("erased-");
    }

    @Test
    void erasePatient_withdraws_multiple_enrollments_across_sites() {
        String patientId = "GDPR-MULTI-" + UUID.randomUUID();
        UUID e1 = persistEnrollment(patientId);
        UUID e2 = persistEnrollment(patientId);
        UUID e3 = persistEnrollment(patientId);

        int count = erasureService.erasePatient(patientId, "default");
        assertThat(count).isEqualTo(3);

        assertThat(findEnrollment(e1).consentStatus).isEqualTo(ConsentStatus.WITHDRAWN);
        assertThat(findEnrollment(e2).consentStatus).isEqualTo(ConsentStatus.WITHDRAWN);
        assertThat(findEnrollment(e3).consentStatus).isEqualTo(ConsentStatus.WITHDRAWN);
    }

    @Test
    void erasePatient_throws_when_no_enrollments_found() {
        assertThatThrownBy(() -> erasureService.erasePatient("NONEXISTENT", "default"))
                .isInstanceOf(PatientNotFoundException.class);
    }

    @Test
    void erasePatient_skips_already_withdrawn_enrollments() {
        String patientId = "GDPR-PARTIAL-" + UUID.randomUUID();
        UUID active = persistEnrollment(patientId);
        UUID withdrawn = persistEnrollment(patientId);
        setWithdrawn(withdrawn);

        int count = erasureService.erasePatient(patientId, "default");
        assertThat(count).isEqualTo(1);
        assertThat(findEnrollment(active).consentStatus).isEqualTo(ConsentStatus.WITHDRAWN);
    }

    @Transactional
    UUID persistEnrollment(String patientId) {
        PatientEnrollment e = new PatientEnrollment();
        e.id = UUID.randomUUID();
        e.siteId = UUID.randomUUID();
        e.patientId = patientId;
        e.tenantId = "default";
        e.consentStatus = ConsentStatus.PENDING;
        e.enrollmentStatus = EnrollmentStatus.CANDIDATE;
        e.persist();
        return e.id;
    }

    @Transactional
    void setWithdrawn(UUID id) {
        PatientEnrollment e = PatientEnrollment.findById(id);
        e.consentStatus = ConsentStatus.WITHDRAWN;
        e.patientId = "erased-" + UUID.randomUUID();
    }

    @Transactional
    PatientEnrollment findEnrollment(UUID id) {
        return PatientEnrollment.findById(id);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=GdprErasureServiceTest --batch-mode`
Expected: compilation failure — `GdprErasureService` does not exist.

- [ ] **Step 4: Write GdprErasureService**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.entity.PatientEnrollment;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.List;

@ApplicationScoped
public class GdprErasureService {

    @Inject ConsentWithdrawalService consentWithdrawalService;

    @Transactional
    public int erasePatient(String patientId, String tenantId) {
        List<PatientEnrollment> enrollments = PatientEnrollment.find(
                "patientId = ?1 AND tenantId = ?2 AND consentStatus != ?3",
                patientId, tenantId, ConsentStatus.WITHDRAWN).list();

        if (enrollments.isEmpty()) {
            throw new PatientNotFoundException(patientId);
        }

        int count = 0;
        for (PatientEnrollment enrollment : enrollments) {
            WithdrawalResult result = consentWithdrawalService.withdraw(enrollment.id, tenantId);
            if (result == WithdrawalResult.WITHDRAWN) {
                count++;
            }
        }
        return count;
    }
}
```

- [ ] **Step 5: Run service tests**

Run: `mvn test -pl runtime -Dtest=GdprErasureServiceTest --batch-mode`
Expected: all PASS.

- [ ] **Step 6: Write GdprErasureResourceTest**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;

@QuarkusTest
class GdprErasureResourceTest {

    @Inject FixedCurrentPrincipal principal;

    @Test
    @TestSecurity(user = "sponsor-user", roles = {ClinicalGroups.SPONSOR})
    void delete_returns_204_and_erases_enrollment() {
        String patientId = "ERASURE-" + UUID.randomUUID();
        persistEnrollment(patientId);

        given()
            .when().delete("/api/gdpr/erasure/patients/{patientId}", patientId)
            .then()
            .statusCode(204)
            .header("X-Enrollments-Erased", "1");
    }

    @Test
    @TestSecurity(user = "coordinator-user", roles = {ClinicalGroups.COORDINATOR})
    void delete_returns_204_for_coordinator() {
        String patientId = "ERASURE-COORD-" + UUID.randomUUID();
        persistEnrollment(patientId);

        given()
            .when().delete("/api/gdpr/erasure/patients/{patientId}", patientId)
            .then()
            .statusCode(204);
    }

    @Test
    @TestSecurity(user = "sponsor-user", roles = {ClinicalGroups.SPONSOR})
    void delete_returns_404_for_unknown_patient() {
        given()
            .when().delete("/api/gdpr/erasure/patients/{patientId}", "NONEXISTENT-" + UUID.randomUUID())
            .then()
            .statusCode(404);
    }

    @Test
    @TestSecurity(user = "pi-user", roles = {ClinicalGroups.INVESTIGATOR})
    void delete_returns_403_for_investigator() {
        given()
            .when().delete("/api/gdpr/erasure/patients/{patientId}", "ANY")
            .then()
            .statusCode(403);
    }

    @Transactional
    void persistEnrollment(String patientId) {
        PatientEnrollment e = new PatientEnrollment();
        e.id = UUID.randomUUID();
        e.siteId = UUID.randomUUID();
        e.patientId = patientId;
        e.tenantId = "default";
        e.consentStatus = ConsentStatus.PENDING;
        e.enrollmentStatus = EnrollmentStatus.CANDIDATE;
        e.persist();
    }
}
```

- [ ] **Step 7: Write GdprErasureResource**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.service.GdprErasureService;
import io.casehub.clinical.service.PatientNotFoundException;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.Response;

@Path("/api/gdpr/erasure")
public class GdprErasureResource {

    @Inject GdprErasureService erasureService;
    @Inject CurrentPrincipal principal;

    @DELETE
    @Path("/patients/{patientId}")
    @RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.COORDINATOR})
    public Response erasePatient(@PathParam("patientId") String patientId) {
        try {
            int count = erasureService.erasePatient(patientId, principal.tenancyId());
            return Response.noContent()
                    .header("X-Enrollments-Erased", String.valueOf(count))
                    .build();
        } catch (PatientNotFoundException e) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
    }
}
```

- [ ] **Step 8: Run all tests**

Run: `mvn test -pl runtime -Dtest="GdprErasure*" --batch-mode`
Expected: all PASS.

- [ ] **Step 9: Commit**

```
feat: add GDPR Art.17 patient-scoped erasure endpoint

DELETE /api/gdpr/erasure/patients/{patientId} finds all non-withdrawn
enrollments for the patient and withdraws each in a single XA transaction.
SPONSOR/COORDINATOR RBAC (cross-site authority). Returns 204 with
X-Enrollments-Erased header. Post-erasure retries return 404 by design
(pseudonymized patientId is unidentifiable).

Refs casehubio/clinical#79
```

---

### Task 7: Documentation Updates

**Files:**
- Modify: `CLAUDE.md` (project repo — `/Users/mdproctor/claude/casehub/clinical/CLAUDE.md`)
- Modify: `ARC42STORIES.MD` (project repo)

**Interfaces:**
- Consumes: all prior task deliverables
- Produces: updated project documentation

- [ ] **Step 1: Update CLAUDE.md**

Four changes:
1. Replace `MissingTenancyClaimException` → `MissingTenancyException` (search and replace all occurrences)
2. Update REST endpoint count from "19" to "20" where referenced
3. In the Flyway migration structure section, add `classpath:db/ledger/migration` to the documented qhorus Flyway locations
4. Add to ecosystem conventions: `ConsentWithdrawalService.withdraw()` returns `WithdrawalResult` (not void, not exception). Document the `SPONSOR/COORDINATOR` RBAC on `GdprErasureResource`.

- [ ] **Step 2: Update ARC42STORIES.MD**

Update line 1382 (Layer 8 code audit table): change `returns void; throws ConsentAlreadyWithdrawnException / PatientEnrollmentNotFoundException` to `returns WithdrawalResult (WITHDRAWN / ALREADY_WITHDRAWN); throws PatientEnrollmentNotFoundException`.

- [ ] **Step 3: Run full test suite to confirm nothing is broken**

Run: `mvn test --batch-mode`
Expected: all PASS.

- [ ] **Step 4: Commit documentation**

```
docs: update CLAUDE.md + ARC42STORIES for #89, #87, #79

CLAUDE.md: MissingTenancyException name correction, 20 REST endpoints,
db/ledger/migration Flyway path, WithdrawalResult return type,
SPONSOR/COORDINATOR RBAC on GdprErasureResource.

ARC42STORIES: ConsentWithdrawalService returns WithdrawalResult (was
void + throws ConsentAlreadyWithdrawnException).

Refs casehubio/clinical#89, casehubio/clinical#87, casehubio/clinical#79
```
