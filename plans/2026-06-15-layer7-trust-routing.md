# Layer 7 — Trust-Weighted Safety Agent Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire trust-weighted safety agent routing (TrustWeightedAgentStrategy), write LedgerAttestation quality signals on SUSAR gate outcomes, add Grade 5 regulatory-submission case, and complete AeEscalationCompletedEvent.unexpected API extension.

**Architecture:** Three independent capabilities: (1) trust routing infrastructure — ClinicalTrustRoutingPolicyProvider displaces DefaultTrustRoutingPolicyProvider; SusarAgentAttestationWriter writes LedgerAttestation on gate outcomes so TrustScoreJob can improve routing over time; (2) regulatory-submission case started concurrently on Grade 5 unexpected AEs; (3) AeEscalationCompletedEvent gains `unexpected` field. The `casehub-engine-ledger` dependency activates TrustWeightedAgentStrategy, WorkerDecisionEventCapture, TrustScoreCache, and CaseLedgerEntryRepository by classpath presence.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-ledger 0.2-SNAPSHOT, casehub-ledger 0.2-SNAPSHOT, Panache Active Record, Vert.x event bus, JUnit 5 + Mockito, Awaitility.

**Spec:** `specs/2026-06-14-layer7-trust-routing-design.md`

---

## Task 1: Add casehub-engine-ledger dependency + Flyway config

**Files:**
- Modify: `runtime/pom.xml` — add dependency
- Modify: `runtime/src/main/resources/application.properties` — add engine-ledger Flyway location
- Modify: `runtime/src/test/resources/application.properties` — same + index-dependency

- [ ] **Step 1: Add casehub-engine-ledger to runtime/pom.xml**

Open `runtime/pom.xml`. After the `casehub-ledger` dependency (around line 32), add:

```xml
    <!-- Layer 7: casehub-engine-ledger — activates TrustWeightedAgentStrategy,
         WorkerDecisionEventCapture, TrustScoreCache, CaseLedgerEntryRepository by classpath presence -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-ledger</artifactId>
    </dependency>
```

- [ ] **Step 2: Add engine-ledger Flyway migration location to production application.properties**

In `runtime/src/main/resources/application.properties`, change line 29:
```properties
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus
```
to:
```properties
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus,classpath:db/engine-ledger/migration
```

casehub-engine-ledger ships V2000__case_ledger_entry.sql and V2001__worker_decision_entry.sql at `classpath:db/engine-ledger/migration`. Without this, WorkerDecisionEventCapture fails at startup because the tables don't exist.

- [ ] **Step 3: Add Flyway + index-dependency to test application.properties**

In `runtime/src/test/resources/application.properties`, find the `quarkus.flyway.qhorus.locations` line and add `classpath:db/engine-ledger/migration` to the end. Then add an index-dependency entry so Quarkus discovers the CDI beans from the jar:

```properties
# [find and update this line]
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus,classpath:db/engine-ledger/migration

# [add these at the end of the Engine index-dependency section]
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

- [ ] **Step 4: Compile and verify no startup errors**

```bash
mvn install -pl api --batch-mode && mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS. The engine-ledger Flyway migrations (V2000, V2001) will run at startup in tests. No CDI deployment errors.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/pom.xml \
  runtime/src/main/resources/application.properties \
  runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: add casehub-engine-ledger dep + Flyway config for trust routing

Activates TrustWeightedAgentStrategy, WorkerDecisionEventCapture, TrustScoreCache,
CaseLedgerEntryRepository by classpath presence. V2000/V2001 Flyway migrations added
to qhorus locations. Engine-ledger added to quarkus.index-dependency for test CDI.

Refs #8"
```

---

## Task 2: AeEscalationCompletedEvent.unexpected + call-site fixes

**Files:**
- Modify: `api/src/main/java/io/casehub/clinical/api/AeEscalationCompletedEvent.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerMemoryTest.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/TrialSafetySignalServiceTest.java`

- [ ] **Step 1: Write a failing test for unexpected propagation**

In `AeEscalationListenerTest.java`, add this test after the existing `completed_event_carries_siteId_from_case_context()` test:

```java
@Test
void completed_event_carries_unexpected_from_case_context() {
    UUID caseId = UUID.randomUUID();
    UUID aeId = UUID.randomUUID();
    UUID enrollmentId = UUID.randomUUID();
    UUID siteId = UUID.randomUUID();

    CaseContext ctx = mock(CaseContext.class);
    when(ctx.getPath("aeId")).thenReturn(aeId.toString());
    when(ctx.getPath("enrollmentId")).thenReturn(enrollmentId.toString());
    when(ctx.getPath("grade")).thenReturn("GRADE_5");
    when(ctx.getPath("siteId")).thenReturn(siteId.toString());
    when(ctx.getPath("safetyReview")).thenReturn(Map.of(AeEscalationListener.OUTCOME_KEY, "REVIEWED"));
    when(ctx.getPath("dsmbEscalation")).thenReturn("completed");
    when(ctx.getPath("tenantId")).thenReturn("test-tenant");
    when(ctx.getPath("unexpected")).thenReturn(true);  // propagated in Layer 8

    CaseInstance instance = mock(CaseInstance.class);
    when(instance.getCaseContext()).thenReturn(ctx);
    when(caseInstanceRepository.findByUuid(eq(caseId), any())).thenReturn(Uni.createFrom().item(instance));
    when(statusUpdater.markCompleted(aeId)).thenReturn(true);
    when(completedEvents.fireAsync(any())).thenReturn(CompletableFuture.completedFuture(null));

    listener.onCaseLifecycle(new CaseLifecycleEvent(
            caseId, null, "CompleteCase", "CaseCompleted", "COMPLETED", "system", "system", null));

    ArgumentCaptor<AeEscalationCompletedEvent> captor =
            ArgumentCaptor.forClass(AeEscalationCompletedEvent.class);
    verify(completedEvents).fireAsync(captor.capture());

    assertThat(captor.getValue().unexpected()).isTrue();
}
```

- [ ] **Step 2: Run failing test**

```bash
mvn install -pl api --batch-mode && \
mvn test -pl runtime -Dtest=AeEscalationListenerTest#completed_event_carries_unexpected_from_case_context --batch-mode
```

Expected: FAIL — compilation error `unexpected()` method not found on `AeEscalationCompletedEvent`.

- [ ] **Step 3: Add `unexpected` to AeEscalationCompletedEvent**

Replace the contents of `api/src/main/java/io/casehub/clinical/api/AeEscalationCompletedEvent.java` with:

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.time.Instant;
import java.util.UUID;

/**
 * CDI event fired when an AE escalation case completes — all required safety
 * reviews (senior monitor, and DSMB if Grade 4+) have been resolved.
 *
 * <p>{@code unexpected} is derived from the case context at fire time (set by
 * AeEscalationCaseService from AdverseEvent.unexpected, added in Layer 8).
 * It is a material fact about the AE that belongs in the completion event for
 * downstream consumers.
 */
public record AeEscalationCompletedEvent(
    UUID aeId,
    CtcaeGrade grade,
    UUID siteId,
    String safetyReviewOutcome,
    boolean dsmbEscalated,
    Instant completedAt,
    boolean unexpected) {}
```

- [ ] **Step 4: Propagate unexpected in AeEscalationListener**

In `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java`, add `unexpected` resolution alongside the other context lookups (after the `dsmbEscalated` line, around line 78):

```java
boolean dsmbEscalated = instance.getCaseContext().getPath("dsmbEscalation") != null;
boolean unexpected = Boolean.TRUE.equals(instance.getCaseContext().getPath("unexpected"));
Instant completedAt = Instant.now();
```

Then update the `fireAsync` call (around line 91):

```java
completedEvents.fireAsync(new AeEscalationCompletedEvent(
        aeId, grade, siteId, safetyReviewOutcome, dsmbEscalated, completedAt, unexpected));
```

- [ ] **Step 5: Fix AeEscalationListenerTest — add unexpected arg to existing constructor**

In `AeEscalationListenerTest`, the `buildContext` helper doesn't exist — it mocks ctx.getPath() inline. Add `unexpected` stub to the existing `completed_event_carries_siteId_from_case_context` test:

```java
when(ctx.getPath("unexpected")).thenReturn(null);  // absent from context — defaults to false
```

Add it after `when(ctx.getPath("tenantId")).thenReturn("test-tenant");`.

The fired event assertion in that test doesn't check `unexpected`, so no assertion change is needed.

- [ ] **Step 6: Fix AeEscalationListenerMemoryTest — add unexpected stub**

In `AeEscalationListenerMemoryTest`, the `buildContext()` helper method stubs context values. Add `unexpected` to the helper. The current signature is:

```java
private static CaseContext buildContext(UUID aeId, UUID enrollmentId, UUID siteId,
                                        String grade, Object safetyReview,
                                        Object dsmbEscalation, String tenantId)
```

Add inside the helper after `when(ctx.getPath("tenantId")).thenReturn(tenantId);`:

```java
when(ctx.getPath("unexpected")).thenReturn(null);
```

No call-site changes needed — no test asserts on `unexpected`.

- [ ] **Step 7: Fix TrialSafetySignalServiceTest — add false to constructor**

In `runtime/src/test/java/io/casehub/clinical/service/TrialSafetySignalServiceTest.java`, find the `completedEvent()` helper (around line 89):

```java
private AeEscalationCompletedEvent completedEvent(UUID siteId, CtcaeGrade grade) {
    return new AeEscalationCompletedEvent(UUID.randomUUID(), grade, siteId, "REVIEWED", true, Instant.now());
}
```

Change to:

```java
private AeEscalationCompletedEvent completedEvent(UUID siteId, CtcaeGrade grade) {
    return new AeEscalationCompletedEvent(UUID.randomUUID(), grade, siteId, "REVIEWED", true, Instant.now(), false);
}
```

- [ ] **Step 8: Run the new test + related tests to confirm pass**

```bash
mvn install -pl api --batch-mode && \
mvn test -pl runtime -Dtest="AeEscalationListenerTest,AeEscalationListenerMemoryTest,TrialSafetySignalServiceTest" --batch-mode
```

Expected: All tests PASS.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  api/src/main/java/io/casehub/clinical/api/AeEscalationCompletedEvent.java \
  runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerMemoryTest.java \
  runtime/src/test/java/io/casehub/clinical/service/TrialSafetySignalServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: add unexpected field to AeEscalationCompletedEvent

Material fact about the AE (unexpected qualifier, 21 CFR 312.32) belongs in the
completion event for downstream consumers. AeEscalationListener reads it from case
context (propagated in Layer 8). Breaking change — all constructors updated.

Refs #8"
```

---

## Task 3: ClinicalTrustRoutingPolicyProvider

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/routing/ClinicalTrustRoutingPolicyProvider.java`
- Create: `runtime/src/test/java/io/casehub/clinical/routing/ClinicalTrustRoutingPolicyProviderTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/routing/TrustRoutingPolicyProviderIntegrationTest.java`

- [ ] **Step 1: Write unit test first**

Create `runtime/src/test/java/io/casehub/clinical/routing/ClinicalTrustRoutingPolicyProviderTest.java`:

```java
package io.casehub.clinical.routing;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.clinical.api.ClinicalCapabilities;
import org.junit.jupiter.api.Test;

class ClinicalTrustRoutingPolicyProviderTest {

    private final ClinicalTrustRoutingPolicyProvider provider = new ClinicalTrustRoutingPolicyProvider();

    @Test
    void safety_monitoring_has_strict_threshold_and_tight_margin() {
        TrustRoutingPolicy policy = provider.forCapability(ClinicalCapabilities.SAFETY_MONITORING);
        assertThat(policy.threshold()).isEqualTo(0.75);
        assertThat(policy.minimumObservations()).isEqualTo(20);
        assertThat(policy.borderlineMargin()).isEqualTo(0.05);
        assertThat(policy.blendFactor()).isEqualTo(0.7);
    }

    @Test
    void safety_monitoring_requires_20_observations_before_trust_kicks_in() {
        TrustRoutingPolicy policy = provider.forCapability(ClinicalCapabilities.SAFETY_MONITORING);
        assertThat(policy.isBootstrap(19)).isTrue();
        assertThat(policy.isBootstrap(20)).isFalse();
    }

    @Test
    void eligibility_screening_has_moderate_threshold() {
        TrustRoutingPolicy policy = provider.forCapability(ClinicalCapabilities.ELIGIBILITY_SCREENING);
        assertThat(policy.threshold()).isEqualTo(0.70);
        assertThat(policy.minimumObservations()).isEqualTo(15);
        assertThat(policy.isBootstrap(14)).isTrue();
        assertThat(policy.isBootstrap(15)).isFalse();
    }

    @Test
    void protocol_review_has_conservative_observation_count() {
        TrustRoutingPolicy policy = provider.forCapability(ClinicalCapabilities.PROTOCOL_REVIEW);
        assertThat(policy.minimumObservations()).isEqualTo(25);
        assertThat(policy.isBootstrap(24)).isTrue();
        assertThat(policy.isBootstrap(25)).isFalse();
    }

    @Test
    void unconfigured_capabilities_return_default_non_null() {
        TrustRoutingPolicy policy = provider.forCapability(ClinicalCapabilities.IRB_CONSULTATION);
        assertThat(policy).isNotNull();
        assertThat(policy).isEqualTo(TrustRoutingPolicy.DEFAULT);
    }

    @Test
    void all_capabilities_return_non_null() {
        for (String cap : new String[]{
                ClinicalCapabilities.SAFETY_MONITORING,
                ClinicalCapabilities.ELIGIBILITY_SCREENING,
                ClinicalCapabilities.PROTOCOL_REVIEW,
                ClinicalCapabilities.IRB_CONSULTATION,
                ClinicalCapabilities.PI_AUTHORISATION,
                ClinicalCapabilities.DATA_SAFETY_MONITORING,
                ClinicalCapabilities.REGULATORY_SUBMISSION,
                ClinicalCapabilities.TRIAL_SUPERVISOR}) {
            assertThat(provider.forCapability(cap)).isNotNull();
        }
    }
}
```

- [ ] **Step 2: Run failing test**

```bash
mvn install -pl api --batch-mode && \
mvn test -pl runtime -Dtest=ClinicalTrustRoutingPolicyProviderTest --batch-mode
```

Expected: FAIL — compilation error, class not found.

- [ ] **Step 3: Implement ClinicalTrustRoutingPolicyProvider**

Create `runtime/src/main/java/io/casehub/clinical/routing/ClinicalTrustRoutingPolicyProvider.java`:

```java
package io.casehub.clinical.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.clinical.api.ClinicalCapabilities;
import io.casehub.clinical.api.ClinicalTrustDimensions;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;

/**
 * Clinical-domain trust routing policies. Displaces {@code DefaultTrustRoutingPolicyProvider
 * @DefaultBean} from casehub-engine-ledger automatically by CDI priority.
 *
 * <p>Policy parameters:
 * <ul>
 *   <li>{@code threshold} — minimum trust score to route to this agent</li>
 *   <li>{@code minimumObservations} — Phase 0 → Phase 2 transition (bootstrap → trust-weighted)</li>
 *   <li>{@code borderlineMargin} — score within this distance of threshold → Phase 3 (EscalateToOversight)</li>
 *   <li>{@code blendFactor} — weight of capability score vs global score</li>
 *   <li>{@code qualityFloors} — per-dimension minimums enforced independently of global score</li>
 *   <li>{@code bootstrapEscalationRequired=false} — Phase 0 falls back to availability, not human escalation</li>
 * </ul>
 */
@ApplicationScoped
public class ClinicalTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {

    @Override
    public TrustRoutingPolicy forCapability(String capability) {
        return switch (capability) {
            case ClinicalCapabilities.SAFETY_MONITORING ->
                // Strict: safety-critical, tight borderline margin → Phase 3 near threshold
                new TrustRoutingPolicy(0.75, 20, 0.05, 0.7,
                        Map.of(ClinicalTrustDimensions.SAFETY_ACCURACY, 0.70), false);

            case ClinicalCapabilities.ELIGIBILITY_SCREENING ->
                // Moderate: reversible decision, wider margin acceptable
                new TrustRoutingPolicy(0.70, 15, 0.10, 0.6,
                        Map.of(ClinicalTrustDimensions.ELIGIBILITY_PRECISION, 0.65), false);

            case ClinicalCapabilities.PROTOCOL_REVIEW ->
                // Conservative: high minimum observations before trust kicks in
                new TrustRoutingPolicy(0.65, 25, 0.08, 0.6,
                        Map.of(ClinicalTrustDimensions.PROTOCOL_ADHERENCE, 0.60), false);

            default ->
                TrustRoutingPolicy.DEFAULT;  // availability routing for all other capabilities
        };
    }
}
```

- [ ] **Step 4: Run unit test to confirm pass**

```bash
mvn test -pl runtime -Dtest=ClinicalTrustRoutingPolicyProviderTest --batch-mode
```

Expected: All 6 tests PASS.

- [ ] **Step 5: Write CDI integration test**

Create `runtime/src/test/java/io/casehub/clinical/routing/TrustRoutingPolicyProviderIntegrationTest.java`:

```java
package io.casehub.clinical.routing;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.clinical.api.ClinicalCapabilities;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class TrustRoutingPolicyProviderIntegrationTest {

    @Inject TrustRoutingPolicyProvider provider;

    @Test
    void cdi_injects_clinical_provider_not_default() {
        assertThat(provider).isInstanceOf(ClinicalTrustRoutingPolicyProvider.class);
    }

    @Test
    void safety_monitoring_policy_not_null() {
        assertThat(provider.forCapability(ClinicalCapabilities.SAFETY_MONITORING)).isNotNull();
    }

    @Test
    void unconfigured_capability_returns_default_not_null() {
        TrustRoutingPolicy policy = provider.forCapability(ClinicalCapabilities.IRB_CONSULTATION);
        assertThat(policy).isNotNull();
        assertThat(policy).isEqualTo(TrustRoutingPolicy.DEFAULT);
    }
}
```

- [ ] **Step 6: Run integration test**

```bash
mvn test -pl runtime -Dtest=TrustRoutingPolicyProviderIntegrationTest --batch-mode
```

Expected: All 3 tests PASS (Quarkus startup includes engine-ledger CDI beans).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/routing/ClinicalTrustRoutingPolicyProvider.java \
  runtime/src/test/java/io/casehub/clinical/routing/ClinicalTrustRoutingPolicyProviderTest.java \
  runtime/src/test/java/io/casehub/clinical/routing/TrustRoutingPolicyProviderIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: ClinicalTrustRoutingPolicyProvider — per-capability trust policies

Displaces DefaultTrustRoutingPolicyProvider @DefaultBean from casehub-engine-ledger.
SAFETY_MONITORING: threshold=0.75, 20 min observations, safety-accuracy quality floor 0.70.
ELIGIBILITY_SCREENING: threshold=0.70, 15 min observations.
PROTOCOL_REVIEW: threshold=0.65, 25 min observations.
All other capabilities fall back to TrustRoutingPolicy.DEFAULT (availability routing).

Refs #8"
```

---

## Task 4: SusarAgentAttestationWriter

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarAgentAttestationWriter.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarAgentAttestationWriterTest.java`

- [ ] **Step 1: Write failing unit test**

Create `runtime/src/test/java/io/casehub/clinical/service/SusarAgentAttestationWriterTest.java`:

```java
package io.casehub.clinical.service;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.argThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import io.casehub.clinical.api.ClinicalCapabilities;
import io.casehub.clinical.api.ClinicalTrustDimensions;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.event.ActionGateApprovedEvent;
import io.casehub.engine.common.internal.event.ActionGateExpiredEvent;
import io.casehub.engine.common.internal.event.ActionGateRejectedEvent;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class SusarAgentAttestationWriterTest {

    @Mock CaseLedgerEntryRepository caseLedgerEntryRepository;
    @Mock LedgerEntryRepository ledgerEntryRepository;
    @InjectMocks SusarAgentAttestationWriter writer;

    private UUID susarCaseId;
    private UUID workerEntryId;
    private AdverseEvent ae;
    private WorkerDecisionEntry workerEntry;

    @BeforeEach
    void setUp() {
        susarCaseId = UUID.randomUUID();
        workerEntryId = UUID.randomUUID();
        ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.susarOversightCaseId = susarCaseId;
        ae.tenantId = "test-tenant";

        workerEntry = new WorkerDecisionEntry();
        workerEntry.id = workerEntryId;
        workerEntry.capabilityTag = ClinicalCapabilities.SAFETY_MONITORING;

        // Default stub: case returns a matching worker entry
        when(caseLedgerEntryRepository.findWorkerDecisionsByCaseId(susarCaseId))
                .thenReturn(List.of(workerEntry));
    }

    @Test
    void approved_gate_writes_endorsed_attestation_with_human_attestor() {
        writer.onApproved(new ActionGateApprovedEvent(susarCaseId, 1L, null, "dr-smith"));

        verify(ledgerEntryRepository).saveAttestation(
                argThat(a ->
                        a.ledgerEntryId.equals(workerEntryId)
                        && a.subjectId.equals(susarCaseId)
                        && a.verdict == AttestationVerdict.ENDORSED
                        && a.attestorType == ActorType.HUMAN
                        && a.attestorId.equals("dr-smith")
                        && ClinicalCapabilities.SAFETY_MONITORING.equals(a.capabilityTag)
                        && ClinicalTrustDimensions.SAFETY_ACCURACY.equals(a.trustDimension)
                        && a.confidence == 1.0),
                eq("test-tenant"));
    }

    @Test
    void rejected_gate_writes_challenged_attestation() {
        writer.onRejected(new ActionGateRejectedEvent(susarCaseId, 1L, null, "dr-jones"));

        verify(ledgerEntryRepository).saveAttestation(
                argThat(a ->
                        a.verdict == AttestationVerdict.CHALLENGED
                        && a.attestorType == ActorType.HUMAN
                        && a.attestorId.equals("dr-jones")),
                eq("test-tenant"));
    }

    @Test
    void expired_gate_writes_challenged_with_system_attestor() {
        writer.onExpired(new ActionGateExpiredEvent(susarCaseId, 1L));

        verify(ledgerEntryRepository).saveAttestation(
                argThat(a ->
                        a.verdict == AttestationVerdict.CHALLENGED
                        && a.attestorType == ActorType.SYSTEM),
                eq("test-tenant"));
    }

    @Test
    void non_susar_gate_caseId_is_silently_ignored() {
        // findBySusarOversightCaseId returns null for non-SUSAR case IDs
        UUID nonSusarCaseId = UUID.randomUUID();
        writer.onApproved(new ActionGateApprovedEvent(nonSusarCaseId, 1L, null, "dr-smith"));

        verify(ledgerEntryRepository, never()).saveAttestation(any(), any());
    }

    @Test
    void missing_worker_decision_entry_logs_warn_and_skips_attestation() {
        when(caseLedgerEntryRepository.findWorkerDecisionsByCaseId(susarCaseId))
                .thenReturn(List.of()); // no entries

        writer.onApproved(new ActionGateApprovedEvent(susarCaseId, 1L, null, "dr-smith"));

        verify(ledgerEntryRepository, never()).saveAttestation(any(), any());
    }
}
```

Note: `AdverseEvent.findBySusarOversightCaseId()` is a static Panache method — it can't be mocked with Mockito. The test uses Mockito's `@InjectMocks` pattern, which means `SusarAgentAttestationWriter` must not call Panache directly; instead it must be testable. However, looking at the existing pattern (e.g., `SusarGateDecisionListenerTest`), these tests are `@QuarkusTest` with `@Transactional` and real DB setup. The writer must use `@Transactional` on each handler, which means the unit test approach above won't work for the Panache call.

**Revised approach:** The test for `SusarAgentAttestationWriter` should follow the `SusarGateDecisionListenerTest` pattern — `@QuarkusTest` with a real AE persisted in H2 and mocked `CaseLedgerEntryRepository` + `LedgerEntryRepository`.

Replace the above with a `@QuarkusTest` version:

```java
package io.casehub.clinical.service;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.argThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.reset;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.when;

import io.casehub.clinical.api.ClinicalCapabilities;
import io.casehub.clinical.api.ClinicalTrustDimensions;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.event.ActionGateApprovedEvent;
import io.casehub.engine.common.internal.event.ActionGateExpiredEvent;
import io.casehub.engine.common.internal.event.ActionGateRejectedEvent;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class SusarAgentAttestationWriterTest {

    @Inject SusarAgentAttestationWriter writer;
    @InjectMock CaseLedgerEntryRepository caseLedgerEntryRepository;
    @InjectMock LedgerEntryRepository ledgerEntryRepository;

    private UUID susarCaseId;
    private UUID workerEntryId;

    @BeforeEach
    void setUp() {
        reset(caseLedgerEntryRepository, ledgerEntryRepository);
        susarCaseId = UUID.randomUUID();
        workerEntryId = UUID.randomUUID();
        // Default stub for CaseLedgerEntryRepository
        WorkerDecisionEntry entry = new WorkerDecisionEntry();
        entry.id = workerEntryId;
        entry.capabilityTag = ClinicalCapabilities.SAFETY_MONITORING;
        when(caseLedgerEntryRepository.findWorkerDecisionsByCaseId(susarCaseId))
                .thenReturn(List.of(entry));
    }

    @Test
    @Transactional
    void approved_gate_writes_endorsed_with_human_attestor() {
        UUID aeId = persistAe(susarCaseId);

        writer.onApproved(new ActionGateApprovedEvent(susarCaseId, 1L, null, "dr-smith"));

        verify(ledgerEntryRepository).saveAttestation(
                argThat(a ->
                        a.ledgerEntryId.equals(workerEntryId)
                        && a.subjectId.equals(susarCaseId)
                        && a.verdict == AttestationVerdict.ENDORSED
                        && a.attestorType == ActorType.HUMAN
                        && "dr-smith".equals(a.attestorId)
                        && ClinicalCapabilities.SAFETY_MONITORING.equals(a.capabilityTag)
                        && ClinicalTrustDimensions.SAFETY_ACCURACY.equals(a.trustDimension)
                        && a.confidence == 1.0),
                eq("test-tenant"));
    }

    @Test
    @Transactional
    void rejected_gate_writes_challenged() {
        persistAe(susarCaseId);

        writer.onRejected(new ActionGateRejectedEvent(susarCaseId, 1L, null, "dr-jones"));

        verify(ledgerEntryRepository).saveAttestation(
                argThat(a ->
                        a.verdict == AttestationVerdict.CHALLENGED
                        && a.attestorType == ActorType.HUMAN),
                any());
    }

    @Test
    @Transactional
    void expired_gate_writes_challenged_with_system_attestor() {
        persistAe(susarCaseId);

        writer.onExpired(new ActionGateExpiredEvent(susarCaseId, 1L));

        verify(ledgerEntryRepository).saveAttestation(
                argThat(a ->
                        a.verdict == AttestationVerdict.CHALLENGED
                        && a.attestorType == ActorType.SYSTEM),
                any());
    }

    @Test
    @Transactional
    void non_susar_case_id_silently_skips_attestation() {
        // No AE has susarOversightCaseId = random UUID
        writer.onApproved(new ActionGateApprovedEvent(UUID.randomUUID(), 1L, null, "dr-smith"));

        verify(ledgerEntryRepository, never()).saveAttestation(any(), any());
    }

    @Test
    @Transactional
    void missing_worker_decision_entry_skips_attestation() {
        UUID caseId = UUID.randomUUID();
        persistAe(caseId);
        when(caseLedgerEntryRepository.findWorkerDecisionsByCaseId(caseId))
                .thenReturn(List.of());

        writer.onApproved(new ActionGateApprovedEvent(caseId, 1L, null, "dr-smith"));

        verify(ledgerEntryRepository, never()).saveAttestation(any(), any());
    }

    @Transactional
    UUID persistAe(UUID susarOversightCaseId) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_4;
        ae.unexpected = true;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "test-tenant";
        ae.susarOversightStatus = SusarOversightStatus.REQUESTED;
        ae.susarOversightCaseId = susarOversightCaseId;
        ae.persist();
        return ae.id;
    }
}
```

- [ ] **Step 2: Run failing test**

```bash
mvn install -pl api --batch-mode && \
mvn test -pl runtime -Dtest=SusarAgentAttestationWriterTest --batch-mode
```

Expected: FAIL — compilation error, `SusarAgentAttestationWriter` not found.

- [ ] **Step 3: Implement SusarAgentAttestationWriter**

Create `runtime/src/main/java/io/casehub/clinical/service/SusarAgentAttestationWriter.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ClinicalActors;
import io.casehub.clinical.api.ClinicalCapabilities;
import io.casehub.clinical.api.ClinicalTrustDimensions;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.event.ActionGateApprovedEvent;
import io.casehub.engine.common.internal.event.ActionGateExpiredEvent;
import io.casehub.engine.common.internal.event.ActionGateRejectedEvent;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.vertx.ConsumeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Writes LedgerAttestation quality signals when SUSAR oversight gates are decided.
 *
 * <p>Observes the same three gate event addresses as SusarGateDecisionListener;
 * uses the same DB-discriminator pattern (findBySusarOversightCaseId) to identify
 * SUSAR oversight gates. The attestation anchors to the WorkerDecisionEntry written
 * by WorkerDecisionEventCapture when the safety-monitoring agent ran in the SUSAR
 * oversight case.
 *
 * <p>TrustScoreJob (casehub-ledger, 24h schedule) reads these attestations and
 * ingests them into Bayesian Beta trust scores per actor/capability pair. Routing
 * quality improves on each cycle.
 *
 * <p>attestorType follows SusarDecisionLedgerWriter pattern: HUMAN when a named
 * actor approved/rejected, SYSTEM when the service acts (expiry).
 */
@ApplicationScoped
public class SusarAgentAttestationWriter {

    private static final Logger LOG = Logger.getLogger(SusarAgentAttestationWriter.class);

    @Inject CaseLedgerEntryRepository caseLedgerEntryRepository;
    @Inject LedgerEntryRepository ledgerEntryRepository;

    @ConsumeEvent(value = "casehub.action.gate.approved", blocking = true)
    @Transactional
    public void onApproved(ActionGateApprovedEvent event) {
        writeAttestation(event.caseId(), AttestationVerdict.ENDORSED, event.approvedBy(), Instant.now());
    }

    @ConsumeEvent(value = "casehub.action.gate.rejected", blocking = true)
    @Transactional
    public void onRejected(ActionGateRejectedEvent event) {
        writeAttestation(event.caseId(), AttestationVerdict.CHALLENGED, event.rejectedBy(), Instant.now());
    }

    @ConsumeEvent(value = "casehub.action.gate.expired", blocking = true)
    @Transactional
    public void onExpired(ActionGateExpiredEvent event) {
        writeAttestation(event.caseId(), AttestationVerdict.CHALLENGED, ClinicalActors.CLINICAL_SERVICE, Instant.now());
    }

    private void writeAttestation(UUID caseId, AttestationVerdict verdict, String attestorId, Instant now) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(caseId);
        if (ae == null) return; // not a SUSAR oversight gate

        caseLedgerEntryRepository.findWorkerDecisionsByCaseId(ae.susarOversightCaseId)
                .stream()
                .filter(e -> ClinicalCapabilities.SAFETY_MONITORING.equals(e.capabilityTag))
                .findFirst()
                .ifPresentOrElse(
                        entry -> {
                            LedgerAttestation attestation = new LedgerAttestation();
                            attestation.id = UUID.randomUUID();
                            attestation.ledgerEntryId = entry.id;
                            attestation.subjectId = ae.susarOversightCaseId;
                            attestation.attestorId = attestorId != null ? attestorId : ClinicalActors.CLINICAL_SERVICE;
                            // Mirror SusarDecisionLedgerWriter line 44: HUMAN when named actor, SYSTEM otherwise
                            attestation.attestorType = ClinicalActors.CLINICAL_SERVICE.equals(attestorId) || attestorId == null
                                    ? ActorType.SYSTEM : ActorType.HUMAN;
                            attestation.attestorRole = "safety-gate-outcome";
                            attestation.verdict = verdict;
                            attestation.capabilityTag = ClinicalCapabilities.SAFETY_MONITORING;
                            attestation.trustDimension = ClinicalTrustDimensions.SAFETY_ACCURACY;
                            attestation.confidence = 1.0;
                            attestation.occurredAt = now;
                            ledgerEntryRepository.saveAttestation(attestation, ae.tenantId);
                        },
                        () -> LOG.warnf("SusarAgentAttestationWriter: no WorkerDecisionEntry for " +
                                "susarOversightCaseId=%s — attestation skipped", ae.susarOversightCaseId)
                );
    }
}
```

- [ ] **Step 4: Run tests**

```bash
mvn test -pl runtime -Dtest=SusarAgentAttestationWriterTest --batch-mode
```

Expected: All 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/SusarAgentAttestationWriter.java \
  runtime/src/test/java/io/casehub/clinical/service/SusarAgentAttestationWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: SusarAgentAttestationWriter — LedgerAttestation on SUSAR gate outcomes

Observes gate approved/rejected/expired events. Discriminates via
AdverseEvent.findBySusarOversightCaseId() — same pattern as SusarGateDecisionListener.
Anchors LedgerAttestation to WorkerDecisionEntry in the SUSAR oversight case.
AttestorType: HUMAN when named actor present, SYSTEM on expiry (mirrors
SusarDecisionLedgerWriter line 44). TrustScoreJob ingests these attestations on its
24h cycle to improve Bayesian Beta trust scores for safety-monitoring agents.

Refs #8"
```

---

## Task 5: RegulatorySubmissionStatus enum + AdverseEvent fields + V112 migration

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Create: `runtime/src/main/resources/db/migration/default/V112__ae_regulatory_submission.sql`

- [ ] **Step 1: Create RegulatorySubmissionStatus enum in api/**

Create `api/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java`:

```java
package io.casehub.clinical.api.model;

public enum RegulatorySubmissionStatus {
    /** Default — AE is not Grade 5 + unexpected; no IND submission triggered. */
    NONE,
    /** Grade 5 + unexpected confirmed; regulatory submission case started. */
    PENDING,
    /** IND expedited safety report filed. */
    FILED
}
```

- [ ] **Step 2: Add fields to AdverseEvent entity**

In `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`, add after the `susarOversightCaseId` field:

```java
@Column(name = "regulatory_submission_status", nullable = false, length = 20)
@Enumerated(EnumType.STRING)
public RegulatorySubmissionStatus regulatorySubmissionStatus = RegulatorySubmissionStatus.NONE;

@Column(name = "regulatory_submission_case_id")
public UUID regulatorySubmissionCaseId;
```

Also add the import:
```java
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
```

- [ ] **Step 3: Write V112 migration SQL**

Create `runtime/src/main/resources/db/migration/default/V112__ae_regulatory_submission.sql`:

```sql
ALTER TABLE adverse_event
    ADD COLUMN regulatory_submission_status VARCHAR(20) NOT NULL DEFAULT 'NONE',
    ADD COLUMN regulatory_submission_case_id UUID;
```

- [ ] **Step 4: Compile to verify no errors**

```bash
mvn install -pl api --batch-mode && mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  api/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java \
  runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java \
  runtime/src/main/resources/db/migration/default/V112__ae_regulatory_submission.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: RegulatorySubmissionStatus + AdverseEvent fields + V112 migration

Tracks IND expedited safety reporting obligation lifecycle: NONE → PENDING → FILED.
V112 adds regulatory_submission_status (NOT NULL DEFAULT 'NONE') and
regulatory_submission_case_id (nullable) to adverse_event table.

Refs #8"
```

---

## Task 6: ClinicalComplianceSupplement.regulatorySubmission()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`

- [ ] **Step 1: Add factory method**

In `ClinicalComplianceSupplement.java`, add after `sponsorNotification()`:

```java
public static ComplianceSupplement regulatorySubmission() {
    ComplianceSupplement s = new ComplianceSupplement();
    s.planRef = "21 CFR 312.32(c)(1)(i) — IND expedited safety reporting, unexpected fatal/life-threatening AE";
    s.algorithmRef = "RegulatorySubmissionCaseService (rule-based Grade 5 + unexpected criteria)";
    s.humanOverrideAvailable = true;
    return s;
}
```

- [ ] **Step 2: Compile**

```bash
mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: ClinicalComplianceSupplement.regulatorySubmission() factory

EU AI Act Art.12 + 21 CFR 312.32(c)(1)(i) compliance supplement for the
regulatory submission ledger entry. LedgerProcessor build-time validator requires
attach() on every LedgerEntry subclass with @Column fields.

Refs #8"
```

---

## Task 7: RegulatorySubmissionLedgerEntry + V2023 migration

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2023__regulatory_submission_ledger_entry.sql`

- [ ] **Step 1: Create ledger entry class**

Create `runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java`:

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident record for IND expedited safety report filing obligation.
 *
 * <p>Written in Phase 1 of RegulatorySubmissionCaseService when Grade 5 + unexpected
 * criteria are confirmed. Records that the obligation was identified, when, and for which AE.
 * EU AI Act Art.12 compliance supplement attached at write time.
 *
 * <p>JOINED inheritance on qhorus datasource. V2023.
 *
 * <p>{@code domainContentBytes()} uses aeId + grade only — both are stable identifiers
 * that survive any subsequent erasure or pseudonymization of the AE record.
 */
@Entity
@Table(name = "regulatory_submission_ledger_entry")
@DiscriminatorValue("RegulatorySubmission")
public class RegulatorySubmissionLedgerEntry extends LedgerEntry {

    @Column(name = "ae_id", nullable = false)
    public UUID aeId;

    @Column(name = "ctcae_grade", nullable = false, length = 20)
    public String grade;

    @Column(name = "filed_at", nullable = false)
    public Instant filedAt;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                aeId  != null ? aeId.toString()  : "",
                grade != null ? grade             : "")
                .getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 2: Create V2023 migration**

Create `runtime/src/main/resources/db/migration/qhorus/V2023__regulatory_submission_ledger_entry.sql`:

```sql
CREATE TABLE regulatory_submission_ledger_entry (
    id          UUID        NOT NULL,
    ae_id       UUID        NOT NULL,
    ctcae_grade VARCHAR(20) NOT NULL,
    filed_at    TIMESTAMP   NOT NULL,
    CONSTRAINT pk_regulatory_submission_ledger_entry
        PRIMARY KEY (id),
    CONSTRAINT fk_regulatory_submission_ledger_entry_base
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 3: Compile**

```bash
mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java \
  runtime/src/main/resources/db/migration/qhorus/V2023__regulatory_submission_ledger_entry.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: RegulatorySubmissionLedgerEntry + V2023 migration

JOINED inheritance on qhorus datasource. Records aeId, ctcae_grade, filed_at.
domainContentBytes() uses aeId + grade only — stable identifiers that survive
patient erasure. EU AI Act Art.12 compliance supplement attached at write time.

Refs #8"
```

---

## Task 8: RegulatorySubmissionLedgerWriter

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java`

- [ ] **Step 1: Write failing test**

Create `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java`:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.argThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.ledger.RegulatorySubmissionLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class RegulatorySubmissionLedgerWriterTest {

    @Mock LedgerEntryRepository ledgerEntryRepository;
    @InjectMocks RegulatorySubmissionLedgerWriter writer;

    @Test
    void writes_entry_with_correct_fields() {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_5;
        ae.tenantId = "test-tenant";

        // Stub: no prior entries for this AE
        org.mockito.Mockito.when(ledgerEntryRepository.findLatestBySubjectId(any(), any()))
                .thenReturn(Optional.empty());

        writer.writeEntry(ae);

        verify(ledgerEntryRepository).save(
                argThat(entry -> {
                    RegulatorySubmissionLedgerEntry rsle = (RegulatorySubmissionLedgerEntry) entry;
                    return rsle.aeId.equals(ae.id)
                            && "GRADE_5".equals(rsle.grade)
                            && rsle.filedAt != null
                            && rsle.subjectId.equals(ae.enrollmentId)
                            && rsle.sequenceNumber == 1
                            && rsle.id != null;
                }),
                eq("test-tenant"));
    }
}
```

- [ ] **Step 2: Run failing test**

```bash
mvn install -pl api --batch-mode && \
mvn test -pl runtime -Dtest=RegulatorySubmissionLedgerWriterTest --batch-mode
```

Expected: FAIL — compilation error, class not found.

- [ ] **Step 3: Implement RegulatorySubmissionLedgerWriter**

Create `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ClinicalActors;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.ledger.RegulatorySubmissionLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Clock;
import java.util.UUID;

/**
 * Writes tamper-evident record when Grade 5 + unexpected AE triggers IND expedited
 * safety reporting obligation (21 CFR 312.32(c)(1)(i)).
 *
 * <p>Written in Phase 1 of RegulatorySubmissionCaseService in the same transaction
 * as the status update. EU AI Act Art.12 compliance supplement attached via
 * ClinicalComplianceSupplement.regulatorySubmission().
 */
@ApplicationScoped
public class RegulatorySubmissionLedgerWriter {

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Clock clock;

    @Transactional(Transactional.TxType.MANDATORY)
    public void writeEntry(AdverseEvent ae) {
        RegulatorySubmissionLedgerEntry entry = new RegulatorySubmissionLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = ae.enrollmentId;
        entry.sequenceNumber = nextSequenceNumber(ae.enrollmentId, ae.tenantId);
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "RegulatorySubmission";
        entry.occurredAt = clock.instant();
        entry.aeId = ae.id;
        entry.grade = ae.grade != null ? ae.grade.name() : null;
        entry.filedAt = clock.instant();
        entry.attach(ClinicalComplianceSupplement.regulatorySubmission());
        ledgerEntryRepository.save(entry, ae.tenantId);
    }

    private int nextSequenceNumber(UUID enrollmentId, String tenantId) {
        return ledgerEntryRepository.findLatestBySubjectId(enrollmentId, tenantId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

Note: `@Transactional(MANDATORY)` ensures this is always called within an existing transaction. If called outside a transaction, it will throw — which is the correct safety net.

- [ ] **Step 4: Run test**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionLedgerWriterTest --batch-mode
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: RegulatorySubmissionLedgerWriter

Writes RegulatorySubmissionLedgerEntry in Phase 1 of the regulatory submission service.
@Transactional(MANDATORY) — must be called within an existing TX. AttorType.SYSTEM
(algorithmic classification, no human actor). EU AI Act Art.12 supplement attached.

Refs #8"
```

---

## Task 9: ClinicalRegulatorySubmissionCaseHub + YAML

**Files:**
- Create: `runtime/src/main/resources/clinical/regulatory-submission.yaml`
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalRegulatorySubmissionCaseHub.java`

- [ ] **Step 1: Create regulatory-submission.yaml**

Create `runtime/src/main/resources/clinical/regulatory-submission.yaml`:

```yaml
dsl: "0.1"
version: "1.0.0"
name: regulatory-submission
namespace: clinical
title: IND Expedited Safety Report Filing

spec:

  capabilities:
    - name: regulatory-submission
      inputSchema: "{ grade: .grade, unexpected: .unexpected, aeId: .aeId }"

  goals:
    - name: submission-complete
      kind: success
      condition: ".submissionFiled != null"

  bindings:
    - name: file-ind-report
      on:
        contextChange:
          filter: ".grade != null and .submissionFiled == null"
      capability: regulatory-submission
```

- [ ] **Step 2: Create ClinicalRegulatorySubmissionCaseHub**

Create `runtime/src/main/java/io/casehub/clinical/service/ClinicalRegulatorySubmissionCaseHub.java`:

```java
package io.casehub.clinical.service;

import io.casehub.api.engine.YamlCaseHub;
import jakarta.enterprise.context.ApplicationScoped;

/**
 * Case definition for IND expedited safety report filing (21 CFR 312.32(c)(1)(i)).
 *
 * <p>Started when Grade 5 + unexpected AE is reported — concurrently with the AE
 * escalation case and SUSAR oversight case (all three observe AdverseEventReportedEvent).
 *
 * <p>No Java function worker — routes to the regulatory-submission capability via
 * trust-weighted routing. In Phase 0 (bootstrap, no trust data) falls back to
 * availability routing.
 */
@ApplicationScoped
public class ClinicalRegulatorySubmissionCaseHub extends YamlCaseHub {

    public ClinicalRegulatorySubmissionCaseHub() {
        super("clinical/regulatory-submission.yaml");
    }
}
```

- [ ] **Step 3: Compile**

```bash
mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS. Quarkus YAML mapper validates the YAML on startup.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/resources/clinical/regulatory-submission.yaml \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalRegulatorySubmissionCaseHub.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: ClinicalRegulatorySubmissionCaseHub + regulatory-submission.yaml

Case definition for IND expedited safety report filing. Goal: submissionFiled != null.
Single binding fires on contextChange when grade is set and submissionFiled is absent.
No Java function worker — routes to regulatory-submission capability via trust routing.

Refs #8"
```

---

## Task 10: RegulatorySubmissionCaseService

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCaseServiceTest.java`

- [ ] **Step 1: Write integration test**

Create `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCaseServiceTest.java`:

```java
package io.casehub.clinical.service;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class RegulatorySubmissionCaseServiceTest {

    @Inject RegulatorySubmissionCaseService service;

    @Test
    void grade5_unexpected_starts_regulatory_case() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_5, true);
        AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_5);

        service.onAdverseEventReported(event);

        await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
            AdverseEvent ae = findAe(aeId);
            assertThat(ae.regulatorySubmissionStatus).isEqualTo(RegulatorySubmissionStatus.PENDING);
            assertThat(ae.regulatorySubmissionCaseId).isNotNull();
        });
    }

    @Test
    void grade4_unexpected_does_not_start_regulatory_case() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_4, true);
        AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_4);

        service.onAdverseEventReported(event);

        AdverseEvent ae = findAe(aeId);
        assertThat(ae.regulatorySubmissionStatus).isEqualTo(RegulatorySubmissionStatus.NONE);
        assertThat(ae.regulatorySubmissionCaseId).isNull();
    }

    @Test
    void grade5_expected_does_not_start_regulatory_case() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_5, false); // unexpected=false
        AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_5);

        service.onAdverseEventReported(event);

        AdverseEvent ae = findAe(aeId);
        assertThat(ae.regulatorySubmissionStatus).isEqualTo(RegulatorySubmissionStatus.NONE);
    }

    @Test
    void idempotency_guard_prevents_double_start() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_5, true);
        setStatus(aeId, RegulatorySubmissionStatus.PENDING);
        AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_5);

        service.onAdverseEventReported(event);

        assertThat(findAe(aeId).regulatorySubmissionCaseId).isNull();
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    @Transactional
    UUID persistAe(CtcaeGrade grade, boolean unexpected) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = grade;
        ae.unexpected = unexpected;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "test-tenant";
        ae.persist();
        return ae.id;
    }

    @Transactional
    void setStatus(UUID aeId, RegulatorySubmissionStatus status) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        ae.regulatorySubmissionStatus = status;
    }

    AdverseEventReportedEvent buildEvent(UUID aeId, CtcaeGrade grade) {
        return new AdverseEventReportedEvent(
                aeId, UUID.randomUUID(), UUID.randomUUID(), grade, Instant.now(), "test-tenant");
    }

    @Transactional
    AdverseEvent findAe(UUID aeId) {
        return AdverseEvent.findById(aeId);
    }
}
```

- [ ] **Step 2: Run failing test**

```bash
mvn install -pl api --batch-mode && \
mvn test -pl runtime -Dtest=RegulatorySubmissionCaseServiceTest --batch-mode
```

Expected: FAIL — compilation error, `RegulatorySubmissionCaseService` not found.

- [ ] **Step 3: Implement RegulatorySubmissionCaseService**

Create `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Observes AdverseEventReportedEvent concurrently with AeEscalationCaseService and
 * SusarOversightCaseService. Starts an IND expedited safety report filing case
 * when Grade 5 + unexpected criteria are met (21 CFR 312.32(c)(1)(i)).
 *
 * <p>Three-phase pattern keeps startCase().join() outside any @Transactional boundary
 * to avoid deadlocking the Agroal pool (same as TrialActivationService, ADR-0004).
 *
 * <p>Phase 1 writes the RegulatorySubmissionLedgerEntry in the same transaction as
 * the status update — ledger evidence is established at the moment the obligation is
 * identified, before the engine case is started.
 */
@ApplicationScoped
public class RegulatorySubmissionCaseService {

    private static final Logger LOG = Logger.getLogger(RegulatorySubmissionCaseService.class);
    private static final Set<CtcaeGrade> REPORTABLE_GRADES = Set.of(CtcaeGrade.GRADE_5);

    @Inject ClinicalRegulatorySubmissionCaseHub regulatorySubmissionCaseHub;
    @Inject RegulatorySubmissionLedgerWriter ledgerWriter;

    public void onAdverseEventReported(@ObservesAsync AdverseEventReportedEvent event) {
        try {
            Map<String, Object> initialContext = prepareAndMark(event);
            if (initialContext == null) return;
            UUID caseId = regulatorySubmissionCaseHub.startCase(initialContext).toCompletableFuture().join();
            persistCaseId(event.aeId(), caseId);
        } catch (Exception e) {
            LOG.errorf(e, "RegulatorySubmissionCaseService: case start failed for aeId=%s", event.aeId());
            try {
                markFailed(event.aeId());
            } catch (Exception ex) {
                LOG.errorf(ex, "RegulatorySubmissionCaseService: markFailed failed for aeId=%s", event.aeId());
            }
        }
    }

    @Transactional
    Map<String, Object> prepareAndMark(AdverseEventReportedEvent event) {
        AdverseEvent ae = AdverseEvent.findById(event.aeId());
        if (ae == null) {
            LOG.warnf("RegulatorySubmissionCaseService: AE not found for aeId=%s — skipping", event.aeId());
            return null;
        }
        // Only Grade 5 + unexpected triggers IND expedited safety reporting
        if (!REPORTABLE_GRADES.contains(ae.grade) || !ae.unexpected) {
            return null;
        }
        // Idempotency guard — protects against CDI at-least-once re-delivery
        if (ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.NONE) {
            LOG.debugf("RegulatorySubmissionCaseService: aeId=%s already processed (status=%s) — skipping",
                    event.aeId(), ae.regulatorySubmissionStatus);
            return null;
        }
        ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.PENDING;
        // Ledger write in same TX — evidence established at obligation identification time
        ledgerWriter.writeEntry(ae);
        Map<String, Object> ctx = new HashMap<>();
        ctx.put("aeId", ae.id.toString());
        ctx.put("grade", ae.grade.name());
        ctx.put("unexpected", ae.unexpected);
        ctx.put("siteId", event.siteId().toString());
        ctx.put("tenantId", ae.tenantId);
        return ctx;
    }

    @Transactional
    void persistCaseId(UUID aeId, UUID caseId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) return;
        ae.regulatorySubmissionCaseId = caseId;
    }

    @Transactional
    void markFailed(UUID aeId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) return;
        ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.NONE; // reset to allow retry
    }
}
```

- [ ] **Step 4: Run tests**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionCaseServiceTest --batch-mode
```

Expected: All 4 tests PASS. The three-phase case start may take up to 10 seconds for the first test.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCaseServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: RegulatorySubmissionCaseService — IND expedited safety reporting

Three-phase @ObservesAsync AdverseEventReportedEvent observer (ADR-0004 pattern).
Grade 5 + unexpected → marks PENDING + writes ledger entry (Phase 1 TX) → starts
regulatory-submission engine case (Phase 2, outside TX) → persists caseId (Phase 3).
Concurrent with AeEscalationCaseService and SusarOversightCaseService.

Refs #8"
```

---

## Task 11: Full test suite + code review

- [ ] **Step 1: Run full test suite**

```bash
mvn install -pl api --batch-mode && mvn test --batch-mode 2>&1 | tail -30
```

Expected: All tests PASS. Note any flaky tests due to engine#393 (CaseLifecycleEvent timing) — these are pre-existing and not introduced by Layer 7.

- [ ] **Step 2: Request code review**

Invoke `superpowers:requesting-code-review` skill. Review scope: all new and modified files from Tasks 1–10.

- [ ] **Step 3: Fix any Minor or above findings**

Any finding Minor or above that can't be fixed immediately must be captured as a GitHub issue before sign-off.

```bash
gh issue create --repo casehubio/clinical \
  --title "Layer 7: <finding>" \
  --body "Found during code review of Layer 7 implementation. <description>"
```

- [ ] **Step 4: Run implementation-doc-sync**

Invoke `implementation-doc-sync` skill to update LAYER-LOG.md and any other docs affected by the implementation.

- [ ] **Step 5: Final commit and push**

```bash
git -C /Users/mdproctor/claude/casehub/clinical push
```
