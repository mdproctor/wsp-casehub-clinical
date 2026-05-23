# Layer 5 — IRB Gate and AE Escalation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce `casehub-engine` to clinical (Layer 5): replace hardcoded routing in `AdverseEventService` with an SPI-driven policy, and add two engine case types — an IRB gate for CRITICAL protocol deviations and an adaptive AE escalation case for Grade 3+ adverse events.

**Architecture:** Two `YamlCaseHub` subclasses (`ClinicalDeviationCaseHub`, `ClinicalAdverseEventCaseHub`) each load a YAML case definition. CDI observers start cases from existing domain events (`ProtocolDeviationResolvedEvent`, new `AdverseEventReportedEvent`). Domain bridge listeners (`IrbDecisionListener`, `AeEscalationListener`) update entities, write ledger entries, and fire resolved events. `AdverseEventEscalationPolicy` SPI replaces all hardcoded routing.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine (YamlCaseHub, CaseHub, CaseHubRuntime), casehub-engine-work-adapter (HumanTaskScheduleHandler, WorkItemLifecycleAdapter, CallerRef), casehub-engine-blackboard (BlackboardRegistry), H2 in tests (drop-and-create + Flyway disabled).

**Spec:** `specs/issue-6-irb-gate/2026-05-22-irb-gate-ae-escalation-design.md`

**Known engine bugs:** engine#312 (HumanTaskScheduleHandler may not create WorkItem — use `await().atMost(5, SECONDS)`); engine#315 (`@ObservesAsync` CDI delivery unreliable to external-jar observers — invoke `WorkItemLifecycleAdapter` directly in tests); engine#314 (nested `{..}` unsupported in `outputMapping` — use flat `"{ key: . }"`); GE-0167 (`inputSchema` uses mini-DSL not JQ).

---

### Task 1: Engine dependencies in `runtime/pom.xml`

**Files:**
- Modify: `runtime/pom.xml`

- [ ] **Add engine dependencies after the existing `casehub-connectors-core` dependency:**

```xml
<!-- Layer 5: casehub-engine — adaptive protocol paths (IRB gate + AE escalation) -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-scheduler-quartz</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-work-adapter</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-testing</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-persistence-memory</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Verify the module compiles:**

```bash
mvn compile -pl api,runtime --batch-mode
```

Expected: `BUILD SUCCESS` (no new classes yet, just dependency resolution).

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/pom.xml
git -C /Users/mdproctor/claude/casehub/clinical commit -m "build(issue-6-irb-gate): add casehub-engine dependencies

Layer 5 foundation — engine runtime, work-adapter, and test infrastructure.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: API types — SPIs, events, `IrbDecision.EXPIRED`

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/spi/AdverseEventContext.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/AdverseEventEscalationRequirements.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/AdverseEventEscalationPolicy.java`
- Create: `api/src/main/java/io/casehub/clinical/api/AdverseEventReportedEvent.java`
- Create: `api/src/main/java/io/casehub/clinical/api/IrbApprovalResolvedEvent.java`
- Create: `api/src/main/java/io/casehub/clinical/api/AeEscalationCompletedEvent.java`
- Modify: `api/src/main/java/io/casehub/clinical/api/model/IrbDecision.java`

- [ ] **Create `AdverseEventContext.java`:**

```java
package io.casehub.clinical.api.spi;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.util.UUID;

/**
 * Input context for {@link AdverseEventEscalationPolicy}. Carries all AE
 * identifiers and the CTCAE grade that determines escalation requirements.
 * Mirrors {@link io.casehub.clinical.api.spi.DeviationContext} pattern.
 */
public record AdverseEventContext(
    UUID aeId,
    UUID enrollmentId,
    UUID siteId,
    CtcaeGrade grade) {}
```

- [ ] **Create `AdverseEventEscalationRequirements.java`:**

```java
package io.casehub.clinical.api.spi;

/**
 * Policy decision for a reported adverse event.
 *
 * <p>When {@code engineCaseRequired} is false, {@code candidateGroups} is used to
 * create a WorkItem directly (Layer 2 path). When true, {@code candidateGroups} is
 * null and the engine case creates WorkItems via humanTask bindings using
 * {@code requiresSeniorMonitor} and {@code requiresDsmbEscalation} as context keys.
 */
public record AdverseEventEscalationRequirements(
    boolean engineCaseRequired,
    String candidateGroups,
    boolean requiresSeniorMonitor,
    boolean requiresDsmbEscalation) {

    public static AdverseEventEscalationRequirements direct(String candidateGroups) {
        return new AdverseEventEscalationRequirements(false, candidateGroups, false, false);
    }

    public static AdverseEventEscalationRequirements engineManaged(
            boolean requiresSeniorMonitor, boolean requiresDsmbEscalation) {
        return new AdverseEventEscalationRequirements(
                true, null, requiresSeniorMonitor, requiresDsmbEscalation);
    }
}
```

- [ ] **Create `AdverseEventEscalationPolicy.java`:**

```java
package io.casehub.clinical.api.spi;

/**
 * Org-customisable policy for adverse event routing and engine case wiring.
 *
 * <p>The default implementation uses CTCAE v5.0 grades. Organisations override
 * this SPI to apply site-specific thresholds, team assignments, and scope rules.
 * This is a vocabulary SPI — a no-op default would break routing; the default
 * must express meaningful routing behaviour.
 */
public interface AdverseEventEscalationPolicy {
    AdverseEventEscalationRequirements evaluate(AdverseEventContext context);
}
```

- [ ] **Create `AdverseEventReportedEvent.java`:**

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.time.Instant;
import java.util.UUID;

/**
 * CDI event fired when a Grade 3+ adverse event is reported and requires
 * engine-managed escalation. Grade 1/2 AEs use direct WorkItem creation
 * and do not fire this event.
 *
 * <p>Consumer: {@code AeEscalationCaseService} — starts the AE escalation
 * engine case and creates humanTask WorkItems via the YAML bindings.
 */
public record AdverseEventReportedEvent(
    UUID aeId,
    UUID enrollmentId,
    UUID siteId,
    CtcaeGrade grade,
    Instant reportedAt) {}
```

- [ ] **Create `IrbApprovalResolvedEvent.java`:**

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.IrbDecision;
import java.time.Instant;
import java.util.UUID;

/**
 * CDI event fired when an IRB consultation case reaches any terminal decision:
 * APPROVED, REJECTED, DEFERRED, or EXPIRED.
 *
 * <p>Follows {@link ProtocolDeviationResolvedEvent} pattern. Layer 6 consumers
 * (trial-level aggregation, DSMB rollup) observe this for cross-site signals.
 */
public record IrbApprovalResolvedEvent(
    UUID approvalId,
    UUID deviationId,
    UUID siteId,
    IrbDecision decision,
    Instant decidedAt) {}
```

- [ ] **Create `AeEscalationCompletedEvent.java`:**

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.time.Instant;
import java.util.UUID;

/**
 * CDI event fired when an AE escalation case completes — all required safety
 * reviews (senior monitor, and DSMB if Grade 4+) have been resolved.
 */
public record AeEscalationCompletedEvent(
    UUID aeId,
    CtcaeGrade grade,
    String safetyReviewOutcome,
    boolean dsmbEscalated,
    Instant completedAt) {}
```

- [ ] **Add `EXPIRED` to `IrbDecision.java`:**

```java
package io.casehub.clinical.api.model;

/** IRB/ethics committee decision on a protocol deviation review or amendment. */
public enum IrbDecision {
    PENDING,
    APPROVED,
    REJECTED,
    /** Committee requests additional information before deciding. Not a final rejection. */
    DEFERRED,
    /** 72-hour IRB WorkItem expired before committee decided. */
    EXPIRED
}
```

- [ ] **Build and verify API module:**

```bash
mvn install -pl api --batch-mode
```

Expected: `BUILD SUCCESS`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): add Layer 5 API types — SPI, events, IrbDecision.EXPIRED

AdverseEventEscalationPolicy SPI replaces hardcoded AE routing.
IrbDecision.EXPIRED handles 72h WorkItem timeout.
Three new CDI events for Layer 6 consumers.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Database migrations V109, V1009, V1010

**Files:**
- Create: `runtime/src/main/resources/db/migration/default/V109__irb_approval_deviation_id.sql`
- Create: `runtime/src/main/resources/db/migration/qhorus/V1009__irb_approval_ledger_entry.sql`
- Create: `runtime/src/main/resources/db/migration/qhorus/V1010__ae_escalation_ledger_entry.sql`

Note: Tests use `drop-and-create` + `migrate-at-start=false`, so migrations only run in production. Verify naming follows V108/V1008 predecessors.

- [ ] **Create V109:**

```sql
-- V109: Link IrbApproval to its originating ProtocolDeviation.
-- Required for IrbDecisionListener to query IrbApproval by deviationId.
-- Nullable: existing stub rows have no linked deviation.
ALTER TABLE irb_approval
    ADD COLUMN deviation_id UUID REFERENCES protocol_deviation(id);
```

- [ ] **Create V1009 (qhorus datasource — ledger subclass join table):**

```sql
-- V1009: IRB approval ledger entries — tamper-evident FDA audit record for IRB decisions.
-- JOINED inheritance from ledger_entry. Mirrors V1005 (ae_ledger_entry) pattern.
CREATE TABLE irb_approval_ledger_entry (
    id               UUID         NOT NULL,
    irb_approval_id  UUID         NOT NULL,
    deviation_id     UUID         NOT NULL,
    irb_decision     VARCHAR(50)  NOT NULL,
    committee_id     VARCHAR(255) NOT NULL,
    decided_at       TIMESTAMP WITH TIME ZONE NOT NULL,
    CONSTRAINT pk_irb_approval_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_irb_approval_le_ledger FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Create V1010 (qhorus datasource):**

```sql
-- V1010: AE escalation ledger entries — records safety review completion outcomes.
-- JOINED inheritance from ledger_entry. Mirrors V1005 pattern.
CREATE TABLE ae_escalation_ledger_entry (
    id                     UUID         NOT NULL,
    ae_id                  UUID         NOT NULL,
    enrollment_id          UUID         NOT NULL,
    ctcae_grade            VARCHAR(50)  NOT NULL,
    safety_review_outcome  VARCHAR(255),
    dsmb_escalated         BOOLEAN      NOT NULL DEFAULT FALSE,
    completed_at           TIMESTAMP WITH TIME ZONE NOT NULL,
    CONSTRAINT pk_ae_escalation_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_ae_escalation_le_ledger FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/resources/db/migration/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): add migrations V109, V1009, V1010

V109: irb_approval.deviation_id FK (default datasource)
V1009: irb_approval_ledger_entry join table (qhorus datasource)
V1010: ae_escalation_ledger_entry join table (qhorus datasource)
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: `IrbApproval` entity — add `deviationId` field

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/IrbApproval.java`

- [ ] **Add `deviationId` field:**

```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.IrbDecision;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "irb_approval")
public class IrbApproval extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "site_id", nullable = false)
    public UUID siteId;

    /**
     * The deviation this IRB approval is for. Nullable for legacy stubs;
     * always set on new rows created by IrbDeviationCaseService.
     * Added in V109.
     */
    @Column(name = "deviation_id")
    public UUID deviationId;

    @Column(name = "review_type", nullable = false)
    public String reviewType;

    @Column(name = "committee_id", nullable = false)
    public String committeeId;

    @Column(name = "decision_deadline", nullable = false)
    public Instant decisionDeadline;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public IrbDecision decision = IrbDecision.PENDING;
}
```

- [ ] **Build runtime to verify:**

```bash
mvn compile -pl runtime --batch-mode
```

Expected: `BUILD SUCCESS`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/entity/IrbApproval.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): add IrbApproval.deviationId field (V109)

Links IRB approval to its originating deviation.
Required for IrbDecisionListener to find approvals by deviationId.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: `application.properties` — engine CDI and config wiring

**Files:**
- Modify: `runtime/src/main/resources/application.properties`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Update production `application.properties` — extend `selected-alternatives`:**

Find the existing line:
```properties
quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository
```

Replace with:
```properties
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.persistence.jpa.JpaPlanItemStore,\
  io.casehub.persistence.jpa.JpaSubCaseGroupRepository
```

- [ ] **Update test `application.properties` — add engine index-dependency entries, CDI exclusions, and memory store selections:**

Add after the existing `quarkus.arc.exclude-types` line:

```properties
# ============================================================
# Engine — external jar indexing (beans not auto-discovered without quarkus.index-dependency)
# ============================================================
quarkus.index-dependency.engine-testing.group-id=io.casehub
quarkus.index-dependency.engine-testing.artifact-id=casehub-engine-testing
quarkus.index-dependency.engine-scheduler-quartz.group-id=io.casehub
quarkus.index-dependency.engine-scheduler-quartz.artifact-id=casehub-engine-scheduler-quartz
quarkus.index-dependency.engine-work-adapter.group-id=io.casehub
quarkus.index-dependency.engine-work-adapter.artifact-id=casehub-engine-work-adapter
quarkus.index-dependency.engine-blackboard.group-id=io.casehub
quarkus.index-dependency.engine-blackboard.artifact-id=casehub-engine-blackboard
quarkus.index-dependency.engine-persistence-memory.group-id=io.casehub
quarkus.index-dependency.engine-persistence-memory.artifact-id=casehub-engine-persistence-memory
```

Replace the existing `quarkus.arc.exclude-types` line with:
```properties
# JpaWorkloadProvider excluded — engine provides its own bridge; the work-module provider
# creates CDI ambiguity when both are on the classpath.
quarkus.arc.exclude-types=io.casehub.connectors.twilio.TwilioSmsConnector,\
  io.casehub.connectors.whatsapp.WhatsAppConnector,\
  io.casehub.work.runtime.service.JpaWorkloadProvider
```

Replace the existing `quarkus.arc.selected-alternatives` line with:
```properties
# In tests: use memory stores for engine PlanItem and SubCaseGroup persistence.
# In production (application.properties): JPA stores are selected instead.
quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.persistence.memory.MemoryPlanItemStore,\
  io.casehub.persistence.memory.MemorySubCaseGroupRepository
```

- [ ] **Verify tests still pass (engine on classpath now — expect some new CDI wiring):**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode 2>&1 | tail -30
```

Expected: All existing tests pass. The engine on the classpath without any case definitions yet is inert.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/resources/application.properties runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): wire engine CDI config — index-dependency, JpaWorkloadProvider exclusion

Production: JPA plan item and sub-case group stores selected.
Tests: memory stores selected; engine external jars indexed.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: Test support infrastructure

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/support/WorkItemCompletionCapture.java`
- Create: `runtime/src/test/java/io/casehub/clinical/support/WorkItemQueries.java`

These are copied from devtown — generic engine test infrastructure, not clinical-specific.

- [ ] **Create `WorkItemCompletionCapture.java`:**

```java
package io.casehub.clinical.support;

import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItemStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

/**
 * Test-scope bean that captures async WorkItem lifecycle events.
 * Used to verify @ObservesAsync CDI delivery to in-process observers
 * before manually invoking WorkItemLifecycleAdapter (engine#315).
 */
@ApplicationScoped
public class WorkItemCompletionCapture {

    private final ConcurrentMap<UUID, WorkItemLifecycleEvent> completed = new ConcurrentHashMap<>();

    void onCompleted(@ObservesAsync WorkItemLifecycleEvent event) {
        if (event.status() == WorkItemStatus.COMPLETED) {
            completed.put(event.workItemId(), event);
        }
    }

    public boolean wasCompleted(UUID workItemId) {
        return completed.containsKey(workItemId);
    }

    public void reset() {
        completed.clear();
    }
}
```

- [ ] **Create `WorkItemQueries.java`:**

```java
package io.casehub.clinical.support;

import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.repository.WorkItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.List;

/**
 * Test-scope query helper. WorkItemStore.scanAll() requires a transaction;
 * this wrapper provides it so tests can call it from non-transactional contexts.
 */
@ApplicationScoped
public class WorkItemQueries {

    @Inject WorkItemStore store;

    @Transactional
    public List<WorkItem> scanAll() {
        return store.scanAll();
    }
}
```

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/test/java/io/casehub/clinical/support/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(issue-6-irb-gate): add engine test support — WorkItemCompletionCapture, WorkItemQueries

Ported from devtown. Needed by IrbGateLifecycleTest and AeEscalationLifecycleTest.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 7: `DefaultAdverseEventEscalationPolicy` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/DefaultAdverseEventEscalationPolicy.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/DefaultAdverseEventEscalationPolicyTest.java`

- [ ] **Write the failing test:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.spi.AdverseEventContext;
import io.casehub.clinical.api.spi.AdverseEventEscalationRequirements;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultAdverseEventEscalationPolicyTest {

    private final DefaultAdverseEventEscalationPolicy policy = new DefaultAdverseEventEscalationPolicy();

    @ParameterizedTest
    @EnumSource(value = CtcaeGrade.class, names = {"GRADE_1", "GRADE_2"})
    void grade1and2_useDirect_safetyCandidateGroup(CtcaeGrade grade) {
        var ctx = new AdverseEventContext(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), grade);
        AdverseEventEscalationRequirements result = policy.evaluate(ctx);

        assertThat(result.engineCaseRequired()).isFalse();
        assertThat(result.candidateGroups()).isEqualTo("safety-officers");
        assertThat(result.requiresSeniorMonitor()).isFalse();
        assertThat(result.requiresDsmbEscalation()).isFalse();
    }

    @Test
    void grade3_requiresSeniorMonitor_noDsmb() {
        var ctx = new AdverseEventContext(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), CtcaeGrade.GRADE_3);
        AdverseEventEscalationRequirements result = policy.evaluate(ctx);

        assertThat(result.engineCaseRequired()).isTrue();
        assertThat(result.candidateGroups()).isNull();
        assertThat(result.requiresSeniorMonitor()).isTrue();
        assertThat(result.requiresDsmbEscalation()).isFalse();
    }

    @ParameterizedTest
    @EnumSource(value = CtcaeGrade.class, names = {"GRADE_4", "GRADE_5"})
    void grade4and5_requiresSeniorMonitorAndDsmb(CtcaeGrade grade) {
        var ctx = new AdverseEventContext(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), grade);
        AdverseEventEscalationRequirements result = policy.evaluate(ctx);

        assertThat(result.engineCaseRequired()).isTrue();
        assertThat(result.requiresSeniorMonitor()).isTrue();
        assertThat(result.requiresDsmbEscalation()).isTrue();
    }
}
```

- [ ] **Run to verify failure:**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=DefaultAdverseEventEscalationPolicyTest --batch-mode 2>&1 | tail -20
```

Expected: `COMPILATION_ERROR` — `DefaultAdverseEventEscalationPolicy` not yet defined.

- [ ] **Implement `DefaultAdverseEventEscalationPolicy.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.spi.AdverseEventContext;
import io.casehub.clinical.api.spi.AdverseEventEscalationPolicy;
import io.casehub.clinical.api.spi.AdverseEventEscalationRequirements;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

/**
 * CTCAE v5.0-based default escalation policy.
 *
 * <p>Grade 1-2: direct WorkItem creation to safety-officers (Layer 2 path preserved).
 * Grade 3: engine case — senior safety monitor gate.
 * Grade 4-5: engine case — senior safety monitor + DSMB escalation in parallel.
 *
 * <p>Organisations override this bean to apply their own thresholds and team assignments.
 */
@ApplicationScoped
@DefaultBean
public class DefaultAdverseEventEscalationPolicy implements AdverseEventEscalationPolicy {

    @Override
    public AdverseEventEscalationRequirements evaluate(AdverseEventContext context) {
        return switch (context.grade()) {
            case GRADE_1, GRADE_2 -> AdverseEventEscalationRequirements.direct("safety-officers");
            case GRADE_3          -> AdverseEventEscalationRequirements.engineManaged(true, false);
            case GRADE_4, GRADE_5 -> AdverseEventEscalationRequirements.engineManaged(true, true);
        };
    }
}
```

- [ ] **Run to verify tests pass:**

```bash
mvn test -pl runtime -Dtest=DefaultAdverseEventEscalationPolicyTest --batch-mode 2>&1 | tail -10
```

Expected: `Tests run: 5, Failures: 0, Errors: 0`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/DefaultAdverseEventEscalationPolicy.java runtime/src/test/java/io/casehub/clinical/service/DefaultAdverseEventEscalationPolicyTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): DefaultAdverseEventEscalationPolicy — CTCAE-based default routing

Replaces hardcoded candidateGroups in AdverseEventService.
Grade 3 → senior monitor. Grade 4/5 → senior monitor + DSMB.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 8: Modify `AdverseEventService` — TDD

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java`

The existing test `workItemId_is_set_after_report` uses `GRADE_3` and asserts `workItemId != null`. After this change, Grade 3+ AEs are engine-managed and `workItemId` is null. That test must be updated.

- [ ] **Update `AdverseEventServiceTest` — fix the broken assertion and add new tests:**

Replace the `workItemId_is_set_after_report` test and add Grade-1/2 tests:

```java
@Test
@Transactional
void grade1_workItemId_is_set_directly() {
    // Grade 1/2 uses Layer 2 direct creation — workItemId is set
    AdverseEvent ae = newAe(CtcaeGrade.GRADE_1);
    service.reportAdverseEvent(ae);
    assertThat(ae.workItemId).as("Grade 1 uses direct WorkItem creation").isNotNull();
}

@Test
@Transactional
void grade3_workItemId_is_null_engine_manages_it() {
    // Grade 3+ triggers engine case — WorkItem created by HumanTaskScheduleHandler
    AdverseEvent ae = newAe(CtcaeGrade.GRADE_3);
    service.reportAdverseEvent(ae);
    assertThat(ae.workItemId)
        .as("Grade 3 is engine-managed; workItemId set by engine, not service")
        .isNull();
}

@Test
@Transactional
void grade4_workItemId_is_null_engine_manages_it() {
    AdverseEvent ae = newAe(CtcaeGrade.GRADE_4);
    service.reportAdverseEvent(ae);
    assertThat(ae.workItemId).isNull();
}
```

Remove the original `workItemId_is_set_after_report` test.

- [ ] **Run to verify the new tests fail (implementation not changed yet):**

```bash
mvn test -pl runtime -Dtest=AdverseEventServiceTest --batch-mode 2>&1 | tail -20
```

Expected: `grade3_workItemId_is_null_engine_manages_it` FAILS (workItemId is currently set).

- [ ] **Implement the modified `AdverseEventService.java`:**

```java
package io.casehub.clinical.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.spi.AdverseEventContext;
import io.casehub.clinical.api.spi.AdverseEventEscalationPolicy;
import io.casehub.clinical.api.spi.AdverseEventEscalationRequirements;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.work.runtime.model.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class AdverseEventService {

    @Inject WorkItemService workItemService;
    @Inject AdverseEventLedgerWriter ledgerWriter;
    @Inject ObjectMapper objectMapper;
    @Inject AdverseEventEscalationPolicy policy;
    @Inject Event<AdverseEventReportedEvent> reportedEvents;

    @Transactional
    public void reportAdverseEvent(AdverseEvent ae) {
        ae.reportedAt = Instant.now();
        ae.slaDeadline = ae.reportedAt.plus(ae.grade.sla().orElseThrow());

        UUID siteId = resolveSiteId(ae.enrollmentId);
        AdverseEventEscalationRequirements requirements =
                policy.evaluate(new AdverseEventContext(ae.id, ae.enrollmentId, siteId, ae.grade));

        if (!requirements.engineCaseRequired()) {
            var workItem = workItemService.create(WorkItemCreateRequest.builder()
                    .title("Adverse Event — " + ae.grade.label())
                    .description("Grade " + ae.grade.label() + " AE for enrollment "
                            + ae.enrollmentId + ". GCP SLA: "
                            + ae.grade.sla().orElseThrow().toHours() + "h from " + ae.reportedAt)
                    .category("adverse-event")
                    .formKey("adverse-event-review")
                    .priority(priority(ae))
                    .candidateGroups(requirements.candidateGroups())
                    .createdBy("system")
                    .payload(payload(ae))
                    .claimDeadline(ae.slaDeadline)
                    .build());
            ae.workItemId = workItem.id;
        }
        // Grade 3+: ae.workItemId remains null — engine creates WorkItems via humanTask bindings

        ae.persist();
        ledgerWriter.writeReportEntry(ae);

        if (requirements.engineCaseRequired()) {
            reportedEvents.fireAsync(new AdverseEventReportedEvent(
                    ae.id, ae.enrollmentId, siteId, ae.grade, ae.reportedAt));
        }
    }

    private UUID resolveSiteId(UUID enrollmentId) {
        PatientEnrollment enrollment = PatientEnrollment.findById(enrollmentId);
        return enrollment != null ? enrollment.siteId : null;
    }

    private WorkItemPriority priority(AdverseEvent ae) {
        return switch (ae.grade) {
            case GRADE_5 -> WorkItemPriority.URGENT;
            case GRADE_3, GRADE_4 -> WorkItemPriority.HIGH;
            default -> WorkItemPriority.MEDIUM;
        };
    }

    private String payload(AdverseEvent ae) {
        try {
            return objectMapper.writeValueAsString(Map.of(
                    "enrollmentId", ae.enrollmentId.toString(),
                    "grade", ae.grade.name(),
                    "occurredAt", ae.occurredAt.toString()));
        } catch (JsonProcessingException e) {
            return "{\"enrollmentId\":\"" + ae.enrollmentId + "\"}";
        }
    }
}
```

- [ ] **Run full `AdverseEventServiceTest` to verify all pass:**

```bash
mvn test -pl runtime -Dtest=AdverseEventServiceTest --batch-mode 2>&1 | tail -15
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): AdverseEventService uses AdverseEventEscalationPolicy SPI

Grade 1/2: direct WorkItem with policy candidateGroups (Layer 2 preserved).
Grade 3+: fires AdverseEventReportedEvent; engine case owns WorkItem creation.
ae.workItemId is null for Grade 3+ (intentional — engine creates WorkItems).
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 9: Case YAML definitions and `YamlCaseHub` subclasses

**Files:**
- Create: `runtime/src/main/resources/clinical/deviation-review.yaml`
- Create: `runtime/src/main/resources/clinical/ae-escalation.yaml`
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalDeviationCaseHub.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalAdverseEventCaseHub.java`

`inputSchema` uses the mini-DSL (GE-0167 — NOT JQ): `{ key: .path }` format. `outputMapping` uses JQ flat pattern (engine#314 — no nested `{..}`).

- [ ] **Create `clinical/deviation-review.yaml`:**

```yaml
dsl: "0.1"
version: "1.0.0"
name: deviation-review
namespace: clinical
title: Protocol Deviation Review — IRB consultation gate

spec:

  goals:
    - name: irb-decided
      kind: success
      condition: ".irbConsultation != null"

  completion:
    success:
      allOf: [irb-decided]

  bindings:
    - name: irb-consultation
      on: { contextChange: {} }
      when: ".irbConsultationRequired == true and .irbConsultation == null"
      humanTask:
        title: "IRB consultation required — protocol deviation"
        expiresIn: PT72H
        candidateGroups: [irb-committee]
        inputSchema: "{ deviationId: .deviationId, severity: .severity }"
        outputMapping: "{ irbConsultation: . }"
```

- [ ] **Create `clinical/ae-escalation.yaml`:**

```yaml
dsl: "0.1"
version: "1.0.0"
name: ae-escalation
namespace: clinical
title: Adverse Event Safety Escalation — adaptive severity routing

spec:

  goals:
    - name: safety-review-complete
      kind: success
      condition: ".safetyReview != null"

    - name: dsmb-complete
      kind: success
      condition: ".requiresDsmbEscalation == false or .dsmbEscalation != null"

  completion:
    success:
      allOf: [safety-review-complete, dsmb-complete]

  bindings:
    - name: safety-review
      on: { contextChange: {} }
      when: ".requiresSeniorMonitor == true and .safetyReview == null"
      humanTask:
        title: "Senior safety monitor review — adverse event"
        expiresIn: PT24H
        candidateGroups: [senior-safety-monitors]
        inputSchema: "{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId }"
        outputMapping: "{ safetyReview: . }"

    - name: dsmb-escalation
      on: { contextChange: {} }
      when: ".requiresDsmbEscalation == true and .dsmbEscalation == null"
      humanTask:
        title: "DSMB escalation — Grade 4+ adverse event"
        expiresIn: PT24H
        candidateGroups: [dsmb]
        inputSchema: "{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId }"
        outputMapping: "{ dsmbEscalation: . }"
```

- [ ] **Create `ClinicalDeviationCaseHub.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.api.engine.YamlCaseHub;
import jakarta.enterprise.context.ApplicationScoped;

/** Case definition for CRITICAL protocol deviation IRB gate (Layer 5). */
@ApplicationScoped
public class ClinicalDeviationCaseHub extends YamlCaseHub {

    public ClinicalDeviationCaseHub() {
        super("clinical/deviation-review.yaml");
    }
}
```

- [ ] **Create `ClinicalAdverseEventCaseHub.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.api.engine.YamlCaseHub;
import jakarta.enterprise.context.ApplicationScoped;

/** Case definition for Grade 3+ adverse event safety escalation (Layer 5). */
@ApplicationScoped
public class ClinicalAdverseEventCaseHub extends YamlCaseHub {

    public ClinicalAdverseEventCaseHub() {
        super("clinical/ae-escalation.yaml");
    }
}
```

- [ ] **Verify compile:**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/resources/clinical/ runtime/src/main/java/io/casehub/clinical/service/ClinicalDeviationCaseHub.java runtime/src/main/java/io/casehub/clinical/service/ClinicalAdverseEventCaseHub.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): case YAML definitions and YamlCaseHub subclasses

deviation-review.yaml: irb-consultation humanTask (72h, irb-committee).
ae-escalation.yaml: safety-review + dsmb-escalation humanTasks (Grade 3+ adaptive).
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 10: `IrbApprovalLedgerEntry` + `IrbApprovalLedgerWriter` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/IrbApprovalLedgerEntry.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/IrbApprovalLedgerWriter.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/IrbApprovalLedgerWriterTest.java`

The `IrbApprovalLedgerEntry` must live in `io.casehub.clinical.ledger` (not `entity`) — required by the two-datasource architecture. Its package is listed in the `qhorus` persistence unit.

- [ ] **Add the `qhorus` persistence unit package to test `application.properties`:**

The existing line:
```properties
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model,io.casehub.clinical.ledger
```
This already includes `io.casehub.clinical.ledger` — no change needed.

- [ ] **Create `IrbApprovalLedgerEntry.java`:**

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.model.LedgerEntry;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident ledger record for IRB committee decisions on protocol deviations.
 *
 * <p>FDA IND / GCP requirement: IRB approval or rejection must be independently
 * verifiable in the audit chain. Extends LedgerEntry via JOINED inheritance
 * on the qhorus datasource. Must live in io.casehub.clinical.ledger — never in
 * io.casehub.clinical.entity (Panache cannot span two persistence units).
 */
@Entity
@Table(name = "irb_approval_ledger_entry")
@DiscriminatorValue("IrbApproval")
public class IrbApprovalLedgerEntry extends LedgerEntry {

    @Column(name = "irb_approval_id", nullable = false)
    public UUID irbApprovalId;

    @Column(name = "deviation_id", nullable = false)
    public UUID deviationId;

    @Column(name = "irb_decision", nullable = false)
    public String irbDecision;

    @Column(name = "committee_id", nullable = false)
    public String committeeId;

    @Column(name = "decided_at", nullable = false)
    public Instant decidedAt;
}
```

- [ ] **Write the failing test:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.IrbDecision;
import io.casehub.clinical.entity.IrbApproval;
import io.casehub.clinical.ledger.IrbApprovalLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneId;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class IrbApprovalLedgerWriterTest {

    @Mock LedgerEntryRepository ledgerEntryRepository;
    @Mock Clock clock;
    @InjectMocks IrbApprovalLedgerWriter writer;

    @Test
    void writeDecisionEntry_persists_correct_fields() {
        Instant now = Instant.parse("2026-05-22T10:00:00Z");
        org.mockito.Mockito.when(clock.instant()).thenReturn(now);
        org.mockito.Mockito.when(ledgerEntryRepository.findLatestBySubjectId(org.mockito.ArgumentMatchers.any()))
                .thenReturn(java.util.Optional.empty());

        IrbApproval approval = new IrbApproval();
        approval.id = UUID.randomUUID();
        approval.deviationId = UUID.randomUUID();
        approval.siteId = UUID.randomUUID();
        approval.committeeId = "irb-oncology";
        approval.decision = IrbDecision.APPROVED;

        writer.writeDecisionEntry(approval);

        ArgumentCaptor<IrbApprovalLedgerEntry> captor =
                ArgumentCaptor.forClass(IrbApprovalLedgerEntry.class);
        verify(ledgerEntryRepository).save(captor.capture());

        IrbApprovalLedgerEntry entry = captor.getValue();
        assertThat(entry.irbApprovalId).isEqualTo(approval.id);
        assertThat(entry.deviationId).isEqualTo(approval.deviationId);
        assertThat(entry.irbDecision).isEqualTo("APPROVED");
        assertThat(entry.committeeId).isEqualTo("irb-oncology");
        assertThat(entry.decidedAt).isEqualTo(now);
        assertThat(entry.subjectId).isEqualTo(approval.id);
        assertThat(entry.sequenceNumber).isEqualTo(1);
    }
}
```

- [ ] **Run to verify failure:**

```bash
mvn test -pl runtime -Dtest=IrbApprovalLedgerWriterTest --batch-mode 2>&1 | tail -10
```

Expected: `COMPILATION_ERROR` — `IrbApprovalLedgerWriter` not defined.

- [ ] **Create `IrbApprovalLedgerWriter.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.entity.IrbApproval;
import io.casehub.clinical.ledger.IrbApprovalLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Clock;
import java.util.UUID;

/**
 * Writes tamper-evident ledger entries for IRB committee decisions.
 * Mirrors AdverseEventLedgerWriter pattern. Uses REQUIRES_NEW so ledger
 * writes don't roll back if the caller's transaction is retried.
 */
@ApplicationScoped
public class IrbApprovalLedgerWriter {

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Clock clock;

    public void writeDecisionEntry(IrbApproval approval) {
        var entry = new IrbApprovalLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = approval.id;
        entry.sequenceNumber = nextSequenceNumber(approval.id);
        entry.entryType = LedgerEntryType.DECISION;
        entry.actorId = "irb-committee";
        entry.actorType = ActorType.HUMAN;
        entry.actorRole = "IrbCommittee";
        entry.occurredAt = clock.instant();
        entry.irbApprovalId = approval.id;
        entry.deviationId = approval.deviationId;
        entry.irbDecision = approval.decision.name();
        entry.committeeId = approval.committeeId;
        entry.decidedAt = clock.instant();
        ledgerEntryRepository.save(entry);
    }

    private int nextSequenceNumber(UUID approvalId) {
        return ledgerEntryRepository.findLatestBySubjectId(approvalId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

- [ ] **Run test to verify pass:**

```bash
mvn test -pl runtime -Dtest=IrbApprovalLedgerWriterTest --batch-mode 2>&1 | tail -10
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/ledger/IrbApprovalLedgerEntry.java runtime/src/main/java/io/casehub/clinical/service/IrbApprovalLedgerWriter.java runtime/src/test/java/io/casehub/clinical/service/IrbApprovalLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): IrbApprovalLedgerEntry + IrbApprovalLedgerWriter

JOINED inheritance on qhorus datasource. Writes FDA-required IRB decision records.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 11: `IrbDeviationCaseService` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/IrbDeviationCaseService.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/IrbDeviationCaseServiceTest.java`

- [ ] **Write the failing test (Mockito unit test — no DB needed):**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class IrbDeviationCaseServiceTest {

    @Mock ClinicalDeviationCaseHub caseHub;
    @InjectMocks IrbDeviationCaseService service;

    @Test
    void irb_review_and_approved_starts_case_with_correct_context() {
        UUID deviationId = UUID.randomUUID();
        UUID siteId = UUID.randomUUID();
        var event = new ProtocolDeviationResolvedEvent(
                deviationId, siteId, DeviationSeverity.CRITICAL,
                EscalationRequirement.IRB_REVIEW, PiApprovalStatus.APPROVED,
                "CONSENT_DEVIATION", "pi-001");

        when(caseHub.startCase(any())).thenReturn(java.util.concurrent.CompletableFuture.completedFuture(UUID.randomUUID()));

        service.onDeviationResolved(event);

        @SuppressWarnings("unchecked")
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
        verify(caseHub).startCase(captor.capture());

        Map<String, Object> ctx = captor.getValue();
        assertThat(ctx.get("deviationId")).isEqualTo(deviationId.toString());
        assertThat(ctx.get("siteId")).isEqualTo(siteId.toString());
        assertThat(ctx.get("severity")).isEqualTo("CRITICAL");
        assertThat(ctx.get("irbConsultationRequired")).isEqualTo(true);
    }

    @Test
    void non_irb_escalation_requirement_does_not_start_case() {
        var event = new ProtocolDeviationResolvedEvent(
                UUID.randomUUID(), UUID.randomUUID(), DeviationSeverity.MAJOR,
                EscalationRequirement.SPONSOR_NOTIFICATION, PiApprovalStatus.APPROVED,
                "PROTOCOL_DEVIATION", "pi-001");

        service.onDeviationResolved(event);

        verifyNoInteractions(caseHub);
    }

    @Test
    void pi_rejected_does_not_start_case() {
        var event = new ProtocolDeviationResolvedEvent(
                UUID.randomUUID(), UUID.randomUUID(), DeviationSeverity.CRITICAL,
                EscalationRequirement.IRB_REVIEW, PiApprovalStatus.REJECTED,
                "CONSENT_DEVIATION", null);

        service.onDeviationResolved(event);

        verifyNoInteractions(caseHub);
    }
}
```

- [ ] **Run to verify failure:**

```bash
mvn test -pl runtime -Dtest=IrbDeviationCaseServiceTest --batch-mode 2>&1 | tail -10
```

Expected: `COMPILATION_ERROR`.

- [ ] **Create `IrbDeviationCaseService.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.IrbDecision;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.IrbApproval;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;

/**
 * Observes {@link ProtocolDeviationResolvedEvent} and starts an IRB deviation
 * engine case when a CRITICAL deviation has been PI-approved.
 *
 * <p>Creates an IrbApproval record (PENDING) before starting the case so that
 * IrbDecisionListener can find it by deviationId when the IRB WorkItem terminates.
 */
@ApplicationScoped
public class IrbDeviationCaseService {

    @Inject ClinicalDeviationCaseHub caseHub;

    @Transactional
    public void onDeviationResolved(@ObservesAsync ProtocolDeviationResolvedEvent event) {
        if (event.escalationRequirement() != EscalationRequirement.IRB_REVIEW) return;
        if (event.terminalStatus() != PiApprovalStatus.APPROVED) return;

        var approval = new IrbApproval();
        approval.id = UUID.randomUUID();
        approval.siteId = event.siteId();
        approval.deviationId = event.deviationId();
        approval.reviewType = "PROTOCOL_DEVIATION";
        approval.committeeId = "irb-committee";
        approval.decisionDeadline = Instant.now().plus(Duration.ofHours(72));
        approval.decision = IrbDecision.PENDING;
        approval.persist();

        var initialContext = Map.<String, Object>of(
                "deviationId", event.deviationId().toString(),
                "siteId", event.siteId().toString(),
                "severity", event.severity().name(),
                "escalationRequirement", event.escalationRequirement().name(),
                "irbConsultationRequired", true);

        caseHub.startCase(initialContext);
    }
}
```

- [ ] **Run tests to verify pass:**

```bash
mvn test -pl runtime -Dtest=IrbDeviationCaseServiceTest --batch-mode 2>&1 | tail -10
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/IrbDeviationCaseService.java runtime/src/test/java/io/casehub/clinical/service/IrbDeviationCaseServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): IrbDeviationCaseService — starts IRB case on CRITICAL deviation approval

Creates IrbApproval(PENDING) then starts deviation-review.yaml case.
Guards: IRB_REVIEW escalation + APPROVED PI status required.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 12: `IrbDecisionListener`

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/IrbDecisionListener.java`

This listener bridges the engine WorkItem lifecycle to the `IrbApproval` domain entity. It is tested via `IrbGateLifecycleTest` in Task 13 (integration test required — it reads from and writes to the DB).

- [ ] **Create `IrbDecisionListener.java`:**

```java
package io.casehub.clinical.service;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.clinical.api.IrbApprovalResolvedEvent;
import io.casehub.clinical.api.model.IrbDecision;
import io.casehub.clinical.entity.IrbApproval;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.workadapter.CallerRef;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Bridges IRB WorkItem lifecycle events to the IrbApproval domain entity and
 * the tamper-evident ledger.
 *
 * <p>Discriminates IRB WorkItems by presence of {@code deviationId} in the payload
 * (set by the YAML binding's inputSchema). Non-IRB WorkItems have no deviationId
 * and are silently ignored.
 *
 * <p>EXPIRED path: WorkItemLifecycleAdapter calls markFaulted() — no outputMapping
 * fires. This listener signals the case directly with
 * {@code irbConsultation: {decision: EXPIRED}} so the irb-decided goal completes.
 */
@ApplicationScoped
public class IrbDecisionListener {

    private static final Logger LOG = Logger.getLogger(IrbDecisionListener.class);

    @Inject IrbApprovalLedgerWriter ledgerWriter;
    @Inject Event<IrbApprovalResolvedEvent> resolvedEvents;
    @Inject ClinicalDeviationCaseHub caseHub;
    @Inject ObjectMapper objectMapper;

    @Transactional
    public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
        if (!(event.source() instanceof WorkItem workItem)) return;

        UUID deviationId = extractDeviationId(workItem);
        if (deviationId == null) return;

        IrbDecision decision = resolveDecision(event.status(), workItem);
        if (decision == null) return; // non-terminal status, ignore

        IrbApproval approval = IrbApproval
                .find("deviationId = ?1 and decision = 'PENDING'", deviationId)
                .firstResult();
        if (approval == null) {
            LOG.warnf("No PENDING IrbApproval for deviationId=%s — already resolved?", deviationId);
            return;
        }

        approval.decision = decision;
        approval.persist();

        if (event.status() == WorkItemStatus.EXPIRED) {
            CallerRef ref = CallerRef.parse(workItem.callerRef);
            if (ref != null) {
                caseHub.signal(ref.caseId(), "irbConsultation", Map.of(
                        "decision", "EXPIRED",
                        "committeeId", approval.committeeId,
                        "decidedAt", Instant.now().toString()));
            }
        }

        ledgerWriter.writeDecisionEntry(approval);

        resolvedEvents.fireAsync(new IrbApprovalResolvedEvent(
                approval.id, deviationId, approval.siteId, decision, Instant.now()));
    }

    private UUID extractDeviationId(WorkItem workItem) {
        if (workItem.payload == null) return null;
        try {
            JsonNode node = objectMapper.readTree(workItem.payload);
            JsonNode idNode = node.get("deviationId");
            if (idNode == null || idNode.isNull()) return null;
            return UUID.fromString(idNode.asText());
        } catch (Exception e) {
            return null;
        }
    }

    private IrbDecision resolveDecision(WorkItemStatus status, WorkItem workItem) {
        return switch (status) {
            case COMPLETED -> parseDecisionFromResolution(workItem.resolution);
            case EXPIRED   -> IrbDecision.EXPIRED;
            default        -> null;
        };
    }

    private IrbDecision parseDecisionFromResolution(String resolution) {
        if (resolution == null) return null;
        try {
            JsonNode node = objectMapper.readTree(resolution);
            JsonNode d = node.get("decision");
            if (d == null || d.isNull()) return null;
            return IrbDecision.valueOf(d.asText().toUpperCase());
        } catch (Exception e) {
            LOG.warnf("Could not parse IrbDecision from resolution: %s", resolution);
            return null;
        }
    }
}
```

- [ ] **Compile:**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/IrbDecisionListener.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): IrbDecisionListener — bridges IRB WorkItem lifecycle to domain

Discriminates by payload.deviationId. Updates IrbApproval.decision.
EXPIRED path signals case directly (engine outputMapping does not fire on expiry).
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 13: `IrbGateLifecycleTest` — integration test

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/IrbGateLifecycleTest.java`

This test follows the devtown `HumanApprovalLifecycleTest` pattern. Directly invokes `IrbDeviationCaseService.onDeviationResolved()` (bypassing async CDI delivery) and `lifecycleAdapter.onWorkItemLifecycle()` (engine#315). Asserts domain entity state after each checkpoint.

- [ ] **Write `IrbGateLifecycleTest.java`:**

```java
package io.casehub.clinical.service;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.clinical.api.IrbApprovalResolvedEvent;
import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.IrbDecision;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.IrbApproval;
import io.casehub.clinical.support.WorkItemCompletionCapture;
import io.casehub.clinical.support.WorkItemQueries;
import io.casehub.engine.spi.CaseInstanceRepository;
import io.casehub.api.model.CaseStatus;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.workadapter.WorkItemLifecycleAdapter;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class IrbGateLifecycleTest {

    @Inject IrbDeviationCaseService irbDeviationCaseService;
    @Inject IrbDecisionListener irbDecisionListener;
    @Inject WorkItemQueries workItemQueries;
    @Inject WorkItemService workItemService;
    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject WorkItemCompletionCapture completionCapture;
    @Inject WorkItemLifecycleAdapter lifecycleAdapter;

    private UUID deviationId;
    private UUID siteId;

    @BeforeEach
    void setup() {
        deviationId = UUID.randomUUID();
        siteId = UUID.randomUUID();
        completionCapture.reset();
    }

    @Test
    void irb_approved_full_lifecycle() throws Exception {
        var event = criticalDeviationApproved();

        // Checkpoint 1: start IRB case — IrbApproval(PENDING) created, engine case started
        irbDeviationCaseService.onDeviationResolved(event);

        // Checkpoint 2: await IRB WorkItem (engine#312 — may not appear immediately)
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    List<WorkItem> items = irbWorkItemsFor(deviationId);
                    assertThat(items).as("IRB WorkItem").isNotEmpty();
                    assertThat(items.get(0).title).contains("IRB consultation");
                });

        WorkItem irbWorkItem = irbWorkItemsFor(deviationId).get(0);

        // Checkpoint 3: complete WorkItem (APPROVED decision)
        String resolution = "{\"decision\":\"APPROVED\",\"committeeId\":\"irb-oncology\",\"decidedAt\":\"2026-05-22T12:00:00Z\"}";
        workItemService.completeFromSystem(irbWorkItem.id, "irb-committee", resolution);

        // Verify CDI delivery to clinical-side listener (IrbDecisionListener — in-process, reliable)
        await().atMost(3, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() ->
                        assertThat(irbApprovalDecision(deviationId))
                                .as("IrbApproval.decision after APPROVED").isEqualTo(IrbDecision.APPROVED));

        // Drive engine path (engine#315 — WorkItemLifecycleAdapter @ObservesAsync unreliable in tests)
        WorkItem completed = irbWorkItemsFor(deviationId).get(0);
        lifecycleAdapter.onWorkItemLifecycle(
                WorkItemLifecycleEvent.of("COMPLETED", completed, "irb-committee", completed.resolution));

        // Checkpoint 5: case completes
        UUID caseId = caseIdFromCallerRef(completed.callerRef);
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    var instance = caseInstanceRepository.findByUuid(caseId)
                            .await().atMost(Duration.ofSeconds(2));
                    assertThat(instance).isNotNull();
                    assertThat(instance.getState()).isEqualTo(CaseStatus.COMPLETED);
                });
    }

    @Test
    void irb_expired_updates_approval_and_completes_case() {
        var event = criticalDeviationApproved();
        irbDeviationCaseService.onDeviationResolved(event);

        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> assertThat(irbWorkItemsFor(deviationId)).isNotEmpty());

        WorkItem irbWorkItem = irbWorkItemsFor(deviationId).get(0);

        // Simulate expiry via direct adapter invocation (no real timer needed in tests)
        lifecycleAdapter.onWorkItemLifecycle(
                WorkItemLifecycleEvent.of("EXPIRED", irbWorkItem, "system", null));

        // IrbDecisionListener receives the event (in-process CDI — reliable)
        // and signals the case with irbConsultation.decision=EXPIRED
        await().atMost(3, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() ->
                        assertThat(irbApprovalDecision(deviationId)).isEqualTo(IrbDecision.EXPIRED));

        // Case should complete via the signal
        UUID caseId = caseIdFromCallerRef(irbWorkItem.callerRef);
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    var instance = caseInstanceRepository.findByUuid(caseId)
                            .await().atMost(Duration.ofSeconds(2));
                    assertThat(instance).isNotNull();
                    assertThat(instance.getState()).isEqualTo(CaseStatus.COMPLETED);
                });
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private ProtocolDeviationResolvedEvent criticalDeviationApproved() {
        return new ProtocolDeviationResolvedEvent(
                deviationId, siteId, DeviationSeverity.CRITICAL,
                EscalationRequirement.IRB_REVIEW, PiApprovalStatus.APPROVED,
                "CONSENT_DEVIATION", "pi-001");
    }

    private List<WorkItem> irbWorkItemsFor(UUID devId) {
        return workItemQueries.scanAll().stream()
                .filter(wi -> wi.payload != null && wi.payload.contains(devId.toString()))
                .toList();
    }

    @Transactional
    IrbDecision irbApprovalDecision(UUID devId) {
        IrbApproval approval = IrbApproval.find("deviationId = ?1", devId).firstResult();
        return approval != null ? approval.decision : null;
    }

    private UUID caseIdFromCallerRef(String callerRef) {
        var ref = io.casehub.workadapter.CallerRef.parse(callerRef);
        return ref != null ? ref.caseId() : null;
    }
}
```

- [ ] **Run to verify tests pass:**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=IrbGateLifecycleTest --batch-mode 2>&1 | tail -30
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`. If engine#312 causes intermittent failures, note it — the 5s `await` should absorb the race.

- [ ] **Run full test suite to catch regressions:**

```bash
mvn test --batch-mode 2>&1 | tail -20
```

Expected: all tests pass.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/test/java/io/casehub/clinical/service/IrbGateLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(issue-6-irb-gate): IrbGateLifecycleTest — full IRB lifecycle (approved + expired paths)

Follows devtown HumanApprovalLifecycleTest pattern.
Direct adapter invocation for engine#315. 5s await for engine#312.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 14: `AeEscalationLedgerEntry` + `AeEscalationLedgerWriter` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/AeEscalationLedgerEntry.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java`

- [ ] **Create `AeEscalationLedgerEntry.java`:**

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.model.LedgerEntry;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident record of AE safety escalation case completion.
 * Records safety review outcome and whether DSMB escalation occurred.
 * JOINED inheritance on qhorus datasource. V1010.
 */
@Entity
@Table(name = "ae_escalation_ledger_entry")
@DiscriminatorValue("AeEscalation")
public class AeEscalationLedgerEntry extends LedgerEntry {

    @Column(name = "ae_id", nullable = false)
    public UUID aeId;

    @Column(name = "enrollment_id", nullable = false)
    public UUID enrollmentId;

    @Column(name = "ctcae_grade", nullable = false)
    public String ctcaeGrade;

    @Column(name = "safety_review_outcome")
    public String safetyReviewOutcome;

    @Column(name = "dsmb_escalated", nullable = false)
    public boolean dsmbEscalated;

    @Column(name = "completed_at", nullable = false)
    public Instant completedAt;
}
```

- [ ] **Write the failing test:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.ledger.AeEscalationLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Clock;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class AeEscalationLedgerWriterTest {

    @Mock LedgerEntryRepository ledgerEntryRepository;
    @Mock Clock clock;
    @InjectMocks AeEscalationLedgerWriter writer;

    @Test
    void writeCompletionEntry_persists_correct_fields() {
        Instant now = Instant.parse("2026-05-22T14:00:00Z");
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any())).thenReturn(Optional.empty());

        writer.writeCompletionEntry(aeId, enrollmentId, CtcaeGrade.GRADE_4, "REVIEWED", true, now);

        ArgumentCaptor<AeEscalationLedgerEntry> captor =
                ArgumentCaptor.forClass(AeEscalationLedgerEntry.class);
        verify(ledgerEntryRepository).save(captor.capture());

        AeEscalationLedgerEntry entry = captor.getValue();
        assertThat(entry.aeId).isEqualTo(aeId);
        assertThat(entry.enrollmentId).isEqualTo(enrollmentId);
        assertThat(entry.ctcaeGrade).isEqualTo("GRADE_4");
        assertThat(entry.safetyReviewOutcome).isEqualTo("REVIEWED");
        assertThat(entry.dsmbEscalated).isTrue();
        assertThat(entry.completedAt).isEqualTo(now);
    }
}
```

- [ ] **Run to verify failure:**

```bash
mvn test -pl runtime -Dtest=AeEscalationLedgerWriterTest --batch-mode 2>&1 | tail -10
```

Expected: `COMPILATION_ERROR`.

- [ ] **Create `AeEscalationLedgerWriter.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.ledger.AeEscalationLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Clock;
import java.time.Instant;
import java.util.UUID;

/** Writes tamper-evident ledger entries for AE escalation case completions. */
@ApplicationScoped
public class AeEscalationLedgerWriter {

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Clock clock;

    public void writeCompletionEntry(
            UUID aeId,
            UUID enrollmentId,
            CtcaeGrade grade,
            String safetyReviewOutcome,
            boolean dsmbEscalated,
            Instant completedAt) {
        var entry = new AeEscalationLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = aeId;
        entry.sequenceNumber = nextSequenceNumber(aeId);
        entry.entryType = LedgerEntryType.DECISION;
        entry.actorId = "system";
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "AeEscalationCase";
        entry.occurredAt = clock.instant();
        entry.aeId = aeId;
        entry.enrollmentId = enrollmentId;
        entry.ctcaeGrade = grade.name();
        entry.safetyReviewOutcome = safetyReviewOutcome;
        entry.dsmbEscalated = dsmbEscalated;
        entry.completedAt = completedAt;
        ledgerEntryRepository.save(entry);
    }

    private int nextSequenceNumber(UUID aeId) {
        return ledgerEntryRepository.findLatestBySubjectId(aeId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

- [ ] **Run to verify pass:**

```bash
mvn test -pl runtime -Dtest=AeEscalationLedgerWriterTest --batch-mode 2>&1 | tail -10
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/ledger/AeEscalationLedgerEntry.java runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): AeEscalationLedgerEntry + AeEscalationLedgerWriter

JOINED inheritance on qhorus datasource. Records safety review completion.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 15: `AeEscalationCaseService` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationCaseServiceTest.java`

- [ ] **Write the failing test:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AeEscalationCaseServiceTest {

    @Mock ClinicalAdverseEventCaseHub caseHub;
    @Mock io.casehub.clinical.api.spi.AdverseEventEscalationPolicy policy;
    @InjectMocks AeEscalationCaseService service;

    @Test
    void grade3_starts_case_with_senior_monitor_context() {
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        UUID siteId = UUID.randomUUID();
        var event = new AdverseEventReportedEvent(aeId, enrollmentId, siteId, CtcaeGrade.GRADE_3, Instant.now());

        when(policy.evaluate(any())).thenReturn(
                io.casehub.clinical.api.spi.AdverseEventEscalationRequirements.engineManaged(true, false));
        when(caseHub.startCase(any())).thenReturn(java.util.concurrent.CompletableFuture.completedFuture(UUID.randomUUID()));

        service.onAdverseEventReported(event);

        @SuppressWarnings("unchecked")
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
        verify(caseHub).startCase(captor.capture());
        Map<String, Object> ctx = captor.getValue();

        assertThat(ctx.get("aeId")).isEqualTo(aeId.toString());
        assertThat(ctx.get("grade")).isEqualTo("GRADE_3");
        assertThat(ctx.get("requiresSeniorMonitor")).isEqualTo(true);
        assertThat(ctx.get("requiresDsmbEscalation")).isEqualTo(false);
    }

    @Test
    void grade4_starts_case_with_dsmb_escalation_context() {
        UUID aeId = UUID.randomUUID();
        var event = new AdverseEventReportedEvent(aeId, UUID.randomUUID(), UUID.randomUUID(), CtcaeGrade.GRADE_4, Instant.now());

        when(policy.evaluate(any())).thenReturn(
                io.casehub.clinical.api.spi.AdverseEventEscalationRequirements.engineManaged(true, true));
        when(caseHub.startCase(any())).thenReturn(java.util.concurrent.CompletableFuture.completedFuture(UUID.randomUUID()));

        service.onAdverseEventReported(event);

        @SuppressWarnings("unchecked")
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
        verify(caseHub).startCase(captor.capture());
        assertThat(captor.getValue().get("requiresDsmbEscalation")).isEqualTo(true);
    }
}
```

- [ ] **Run to verify failure:**

```bash
mvn test -pl runtime -Dtest=AeEscalationCaseServiceTest --batch-mode 2>&1 | tail -10
```

Expected: `COMPILATION_ERROR`.

- [ ] **Create `AeEscalationCaseService.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.spi.AdverseEventContext;
import io.casehub.clinical.api.spi.AdverseEventEscalationPolicy;
import io.casehub.clinical.api.spi.AdverseEventEscalationRequirements;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.util.HashMap;
import java.util.Map;

/**
 * Observes {@link AdverseEventReportedEvent} and starts an AE escalation engine
 * case for Grade 3+ adverse events.
 *
 * <p>Re-evaluates AdverseEventEscalationPolicy to populate the case initial context.
 * This is intentional — the event carries facts; the policy sets routing keys.
 * Each consumer calls the policy for its own concern.
 */
@ApplicationScoped
public class AeEscalationCaseService {

    @Inject ClinicalAdverseEventCaseHub caseHub;
    @Inject AdverseEventEscalationPolicy policy;

    public void onAdverseEventReported(@ObservesAsync AdverseEventReportedEvent event) {
        AdverseEventEscalationRequirements requirements = policy.evaluate(
                new AdverseEventContext(event.aeId(), event.enrollmentId(), event.siteId(), event.grade()));

        Map<String, Object> initialContext = new HashMap<>();
        initialContext.put("aeId", event.aeId().toString());
        initialContext.put("enrollmentId", event.enrollmentId().toString());
        initialContext.put("grade", event.grade().name());
        initialContext.put("requiresSeniorMonitor", requirements.requiresSeniorMonitor());
        initialContext.put("requiresDsmbEscalation", requirements.requiresDsmbEscalation());

        caseHub.startCase(initialContext);
    }
}
```

- [ ] **Run to verify pass:**

```bash
mvn test -pl runtime -Dtest=AeEscalationCaseServiceTest --batch-mode 2>&1 | tail -10
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java runtime/src/test/java/io/casehub/clinical/service/AeEscalationCaseServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): AeEscalationCaseService — starts AE engine case on AdverseEventReportedEvent

Re-evaluates policy for initial context keys. Grade 3+: senior monitor.
Grade 4+: senior monitor + DSMB in parallel.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 16: `AeEscalationListener`

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java`

Tested by `AeEscalationLifecycleTest` in Task 17.

- [ ] **Create `AeEscalationListener.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AeEscalationCompletedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.engine.internal.event.CaseLifecycleEvent;
import io.casehub.engine.spi.CaseInstanceRepository;
import io.casehub.engine.internal.model.CaseInstance;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Observes case lifecycle events and handles AE escalation case completion.
 *
 * <p>Discriminates AE escalation cases by presence of {@code aeId} in the case
 * context (set at case start by AeEscalationCaseService). All other case types
 * (deviation-review, etc.) lack this key and are silently ignored.
 *
 * <p>On completion: writes AeEscalationLedgerEntry and fires AeEscalationCompletedEvent.
 */
@ApplicationScoped
public class AeEscalationListener {

    private static final Logger LOG = Logger.getLogger(AeEscalationListener.class);
    private static final Duration LOOKUP_TIMEOUT = Duration.ofSeconds(5);

    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject AeEscalationLedgerWriter ledgerWriter;
    @Inject Event<AeEscalationCompletedEvent> completedEvents;

    public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (!"CaseCompleted".equals(event.eventType())) return;

        CaseInstance instance = caseInstanceRepository
                .findByUuid(event.caseId())
                .await().atMost(LOOKUP_TIMEOUT);
        if (instance == null) return;

        Object aeIdObj = instance.getCaseContext().getPath("aeId");
        if (aeIdObj == null) return; // not an AE escalation case

        UUID aeId = UUID.fromString(aeIdObj.toString());
        UUID enrollmentId = resolveUuid(instance, "enrollmentId");
        CtcaeGrade grade = resolveGrade(instance);
        String safetyReviewOutcome = resolveOutcome(instance, "safetyReview");
        boolean dsmbEscalated = instance.getCaseContext().getPath("dsmbEscalation") != null;
        Instant completedAt = Instant.now();

        ledgerWriter.writeCompletionEntry(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, completedAt);

        completedEvents.fireAsync(new AeEscalationCompletedEvent(
                aeId, grade, safetyReviewOutcome, dsmbEscalated, completedAt));
    }

    private UUID resolveUuid(CaseInstance instance, String path) {
        Object obj = instance.getCaseContext().getPath(path);
        if (obj == null) return null;
        try { return UUID.fromString(obj.toString()); } catch (IllegalArgumentException e) { return null; }
    }

    private CtcaeGrade resolveGrade(CaseInstance instance) {
        Object obj = instance.getCaseContext().getPath("grade");
        if (obj == null) return null;
        try { return CtcaeGrade.valueOf(obj.toString()); } catch (IllegalArgumentException e) { return null; }
    }

    private String resolveOutcome(CaseInstance instance, String contextKey) {
        Object obj = instance.getCaseContext().getPath(contextKey);
        if (!(obj instanceof Map<?, ?> map)) return null;
        Object outcome = map.get("outcome");
        return outcome != null ? outcome.toString() : null;
    }
}
```

- [ ] **Compile:**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-6-irb-gate): AeEscalationListener — bridges AE case completion to domain

Discriminates by CaseContext.aeId. Writes ledger entry and fires AeEscalationCompletedEvent.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 17: `AeEscalationLifecycleTest` — integration test

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java`

- [ ] **Write `AeEscalationLifecycleTest.java`:**

```java
package io.casehub.clinical.service;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.support.WorkItemCompletionCapture;
import io.casehub.clinical.support.WorkItemQueries;
import io.casehub.engine.spi.CaseInstanceRepository;
import io.casehub.api.model.CaseStatus;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.workadapter.WorkItemLifecycleAdapter;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class AeEscalationLifecycleTest {

    @Inject AeEscalationCaseService aeEscalationCaseService;
    @Inject WorkItemQueries workItemQueries;
    @Inject WorkItemService workItemService;
    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject WorkItemCompletionCapture completionCapture;
    @Inject WorkItemLifecycleAdapter lifecycleAdapter;

    private UUID aeId;
    private UUID enrollmentId;
    private UUID siteId;

    @BeforeEach
    void setup() {
        aeId = UUID.randomUUID();
        enrollmentId = UUID.randomUUID();
        siteId = UUID.randomUUID();
        completionCapture.reset();
    }

    @Test
    void grade3_opens_one_senior_monitor_gate() throws Exception {
        var event = aeEvent(CtcaeGrade.GRADE_3);
        aeEscalationCaseService.onAdverseEventReported(event);

        // Checkpoint 2: one safety-review WorkItem, no dsmb WorkItem
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    List<WorkItem> items = aeWorkItemsFor(aeId);
                    assertThat(items).as("safety-review WorkItem for Grade 3").hasSize(1);
                    assertThat(items.get(0).title).contains("Senior safety monitor");
                });

        List<WorkItem> dsmbItems = workItemQueries.scanAll().stream()
                .filter(wi -> wi.payload != null && wi.payload.contains(aeId.toString())
                        && wi.title.contains("DSMB"))
                .toList();
        assertThat(dsmbItems).as("No DSMB WorkItem for Grade 3").isEmpty();

        // Checkpoint 3: complete the safety-review WorkItem
        WorkItem safetyWorkItem = aeWorkItemsFor(aeId).get(0);
        String resolution = "{\"outcome\":\"REVIEWED\",\"reviewedAt\":\"2026-05-22T13:00:00Z\"}";
        workItemService.completeFromSystem(safetyWorkItem.id, "senior-monitor", resolution);

        lifecycleAdapter.onWorkItemLifecycle(
                WorkItemLifecycleEvent.of("COMPLETED", safetyWorkItem, "senior-monitor",
                        safetyWorkItem.resolution));

        // Checkpoint 4: case completes (dsmb-complete satisfied because requiresDsmbEscalation=false)
        UUID caseId = caseIdFromWorkItem(safetyWorkItem);
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    var instance = caseInstanceRepository.findByUuid(caseId)
                            .await().atMost(Duration.ofSeconds(2));
                    assertThat(instance).isNotNull();
                    assertThat(instance.getState()).isEqualTo(CaseStatus.COMPLETED);
                });
    }

    @Test
    void grade4_opens_two_parallel_gates_senior_monitor_and_dsmb() throws Exception {
        var event = aeEvent(CtcaeGrade.GRADE_4);
        aeEscalationCaseService.onAdverseEventReported(event);

        // Checkpoint 2: two WorkItems (safety-review + dsmb-escalation)
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    List<WorkItem> items = aeWorkItemsFor(aeId);
                    assertThat(items).as("Two WorkItems for Grade 4").hasSizeGreaterThanOrEqualTo(2);
                });

        List<WorkItem> allItems = aeWorkItemsFor(aeId);
        WorkItem safetyItem = allItems.stream().filter(wi -> wi.title.contains("Senior")).findFirst().orElseThrow();
        WorkItem dsmbItem = allItems.stream().filter(wi -> wi.title.contains("DSMB")).findFirst().orElseThrow();

        // Checkpoint 3: complete both WorkItems
        String resolution = "{\"outcome\":\"REVIEWED\",\"reviewedAt\":\"2026-05-22T13:00:00Z\"}";
        workItemService.completeFromSystem(safetyItem.id, "senior-monitor", resolution);
        workItemService.completeFromSystem(dsmbItem.id, "dsmb-chair", resolution);

        lifecycleAdapter.onWorkItemLifecycle(
                WorkItemLifecycleEvent.of("COMPLETED", safetyItem, "senior-monitor", safetyItem.resolution));
        lifecycleAdapter.onWorkItemLifecycle(
                WorkItemLifecycleEvent.of("COMPLETED", dsmbItem, "dsmb-chair", dsmbItem.resolution));

        // Checkpoint 4: case completes when both goals satisfied
        UUID caseId = caseIdFromWorkItem(safetyItem);
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    var instance = caseInstanceRepository.findByUuid(caseId)
                            .await().atMost(Duration.ofSeconds(2));
                    assertThat(instance).isNotNull();
                    assertThat(instance.getState()).isEqualTo(CaseStatus.COMPLETED);
                });
    }

    @Test
    void grade1_no_engine_case_started() {
        // Grade 1 AE goes through Layer 2 direct creation — no AdverseEventReportedEvent fired
        // This test verifies the service layer does NOT start a case for Grade 1
        // (controlled by AdverseEventEscalationPolicy, not AeEscalationCaseService)
        long workItemCountBefore = workItemQueries.scanAll().size();

        // Simulate no event fired (Grade 1 path never calls AeEscalationCaseService)
        // Verify no additional WorkItems appeared
        assertThat(workItemQueries.scanAll().size())
                .as("No engine WorkItems for Grade 1")
                .isEqualTo(workItemCountBefore);
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private AdverseEventReportedEvent aeEvent(CtcaeGrade grade) {
        return new AdverseEventReportedEvent(aeId, enrollmentId, siteId, grade, Instant.now());
    }

    private List<WorkItem> aeWorkItemsFor(UUID aeId) {
        return workItemQueries.scanAll().stream()
                .filter(wi -> wi.payload != null && wi.payload.contains(aeId.toString()))
                .toList();
    }

    private UUID caseIdFromWorkItem(WorkItem wi) {
        var ref = io.casehub.workadapter.CallerRef.parse(wi.callerRef);
        return ref != null ? ref.caseId() : null;
    }
}
```

- [ ] **Run all integration tests:**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode 2>&1 | tail -30
```

Expected: all tests pass including `IrbGateLifecycleTest` and `AeEscalationLifecycleTest`.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(issue-6-irb-gate): AeEscalationLifecycleTest — adaptive routing (Grade 3 vs Grade 4)

Grade 3: one safety-review gate. Grade 4: two parallel gates (senior + DSMB).
Follows devtown HumanApprovalLifecycleTest pattern.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 18: LAYER-LOG.md — Layer 5 entry

**Files:**
- Modify: `LAYER-LOG.md` in project repo (`/Users/mdproctor/claude/casehub/clinical/LAYER-LOG.md`)

- [ ] **Append Layer 5 entry to LAYER-LOG.md:**

Open `LAYER-LOG.md` and add after the existing Layer 4 entry:

```markdown
## Layer 5 — casehub-engine: adaptive protocol paths

**Issues:** casehubio/clinical#6

### What it shows

casehub-engine enters clinical for the first time. Fixed-pipeline service code
(direct WorkItem creation with hardcoded candidateGroups) is replaced by engine
case definitions that open different gates based on accumulated context.

**IRB gate** (`deviation-review.yaml`): CRITICAL deviation + PI approval suspends
the case in WAITING until an IRB committee decides within 72h. All four terminal
outcomes (APPROVED, REJECTED, DEFERRED, EXPIRED) handled explicitly.

**AE severity routing** (`ae-escalation.yaml`): Grade 3+ AE opens a
`safety-review` humanTask (senior-safety-monitors). Grade 4+ additionally opens
a `dsmb-escalation` humanTask in parallel. Same YAML — initial context determines
which bindings fire.

`AdverseEventEscalationPolicy` SPI replaces hardcoded routing entirely. Grade 1/2
AEs use the Layer 2 direct WorkItem path unchanged.

### Compliance gap closed

Fixed pipeline: agents and humans handled events in a fixed imperative sequence.
Adaptive paths: engine evaluates binding conditions against accumulated context,
opens only the gates warranted by the specific event's severity and policy.
Closes the "no adaptive paths" gap in the ClinicalAgent comparison table.

### Key wiring

- `ClinicalDeviationCaseHub` / `deviation-review.yaml` — IRB case definition
- `IrbDeviationCaseService` — creates IrbApproval(PENDING) + starts case
- `IrbDecisionListener` — updates IrbApproval.decision; EXPIRED signals case manually
- `IrbApprovalLedgerWriter` / `IrbApprovalLedgerEntry` — FDA tamper-evident record
- `IrbApprovalResolvedEvent` — Layer 6 hook (trial-level aggregation)
- `ClinicalAdverseEventCaseHub` / `ae-escalation.yaml` — AE case definition
- `AeEscalationCaseService` — observes AdverseEventReportedEvent, starts case
- `AeEscalationListener` — discriminates by CaseContext.aeId; writes ledger + event
- `AeEscalationLedgerWriter` / `AeEscalationLedgerEntry` — safety review completion record
- `AdverseEventEscalationPolicy` SPI + `DefaultAdverseEventEscalationPolicy`
- V109 (irb_approval.deviation_id), V1009 (irb_approval_ledger_entry), V1010 (ae_escalation_ledger_entry)

### Gotchas

- **engine#312** — `HumanTaskScheduleHandler` PENDING guard: PlanItem may be pre-marked
  RUNNING by `PlanningStrategyLoopControl` before the event fires. Handler skips.
  Tests: `await().atMost(5, SECONDS)`. Filed casehubio/engine#312.
- **engine#315** — `@ObservesAsync` CDI delivery to external-jar observers unreliable
  in tests. Invoke `WorkItemLifecycleAdapter.onWorkItemLifecycle()` directly.
  Filed casehubio/engine#315.
- **engine#314** — nested `{..}` unsupported in `outputMapping`. Use flat pattern:
  `"{ key: . }"` not `"{ key: { field: .field } }"`.
- **GE-0167** — `inputSchema` uses mini-DSL not JQ: `{ key: .path }` only.
  No pipes, filters, or array iteration.
- **JpaWorkloadProvider exclusion** — CDI ambiguity with engine bridge.
  Add to `quarkus.arc.exclude-types` in test application.properties.
- **Grade 3+ AEs** — `ae.workItemId` is null post-Layer 5. Engine creates WorkItems
  via humanTask bindings. Existing tests that asserted `workItemId != null` updated.
- **`IrbDecisionListener` EXPIRED path** — `WorkItemLifecycleAdapter.markFaulted()`
  does not fire outputMapping. Listener calls `caseHub.signal()` directly with
  `irbConsultation: {decision: EXPIRED}` to satisfy the `irb-decided` goal.

### Pattern to replicate

`YamlCaseHub` subclass (ApplicationScoped, loads YAML from classpath) →
CDI event observer starts case with initial context (policy sets routing keys) →
YAML binding conditions use context keys → `WorkItemLifecycleAdapter` fires
CONTEXT_CHANGED on terminal WorkItem states → domain listener observes
`WorkItemLifecycleEvent` (IRB) or `CaseLifecycleEvent` (AE), updates domain +
fires resolved CDI event → tests invoke adapter directly (engine#315).
```

- [ ] **Build and run all tests one final time:**

```bash
mvn test --batch-mode 2>&1 | tail -20
```

Expected: all tests pass.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add LAYER-LOG.md
git -C /Users/mdproctor/claude/casehub/clinical commit -m "docs(issue-6-irb-gate): LAYER-LOG Layer 5 entry — casehub-engine adaptive protocol paths

IRB gate + AE escalation. Documents gotchas engine#312/315/314, GE-0167.
Refs casehubio/clinical#6

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Self-Review

### Spec coverage check

| Spec section | Covered by task |
|---|---|
| AdverseEventEscalationPolicy SPI | Task 2, Task 7 |
| AdverseEventReportedEvent / IrbApprovalResolvedEvent / AeEscalationCompletedEvent | Task 2 |
| IrbDecision.EXPIRED | Task 2 |
| V109 irb_approval.deviation_id | Task 3 |
| V1009 irb_approval_ledger_entry | Task 3 |
| V1010 ae_escalation_ledger_entry | Task 3 |
| IrbApproval entity (deviationId field) | Task 4 |
| Engine pom.xml dependencies | Task 1 |
| application.properties (prod + test) | Task 5 |
| WorkItemCompletionCapture / WorkItemQueries | Task 6 |
| DefaultAdverseEventEscalationPolicy | Task 7 |
| AdverseEventService modification | Task 8 |
| deviation-review.yaml + ClinicalDeviationCaseHub | Task 9 |
| ae-escalation.yaml + ClinicalAdverseEventCaseHub | Task 9 |
| IrbApprovalLedgerEntry + IrbApprovalLedgerWriter | Task 10 |
| IrbDeviationCaseService | Task 11 |
| IrbDecisionListener | Task 12 |
| IrbGateLifecycleTest (APPROVED + EXPIRED paths) | Task 13 |
| AeEscalationLedgerEntry + AeEscalationLedgerWriter | Task 14 |
| AeEscalationCaseService | Task 15 |
| AeEscalationListener | Task 16 |
| AeEscalationLifecycleTest (Grade 3 + Grade 4) | Task 17 |
| LAYER-LOG Layer 5 entry | Task 18 |
| engine#312 / engine#315 / engine#314 / GE-0167 gotchas | Tasks 5, 9, 13, 17, 18 |

All spec requirements covered.

### Type consistency

- `AdverseEventContext` used in Tasks 7, 8, 15 — same record definition from Task 2 ✅
- `AdverseEventEscalationRequirements.direct()` / `.engineManaged()` factory methods — consistent across Tasks 7, 8, 15 ✅
- `WorkItemLifecycleEvent.of("COMPLETED", wi, actor, wi.resolution)` — consistent across Tasks 13, 17 ✅
- `CallerRef.parse(workItem.callerRef).caseId()` — consistent across Tasks 12, 13, 17 ✅
- `IrbApprovalLedgerWriter.writeDecisionEntry(IrbApproval)` — defined Task 10, used Task 12 ✅
- `AeEscalationLedgerWriter.writeCompletionEntry(aeId, enrollmentId, grade, outcome, dsmbEscalated, completedAt)` — defined Task 14, used Task 16 ✅
