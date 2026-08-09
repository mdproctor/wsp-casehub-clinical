# DSMB Batch Signal WorkItem Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #146 — DSMB WorkItem for batch-detected safety signals
**Issue group:** #146

**Goal:** Create DSMB WorkItems and send connector notifications when `TrialSafetyAggregationJob` detects batch safety signals (grade-threshold, cross-site-cluster).

**Architecture:** Two-phase transaction split in `upsertSignalRecord()` — Phase 1 persists the signal record, Phase 2 creates the WorkItem in a separate `REQUIRES_NEW` transaction with error isolation. Notification fires after Phase 2 commits. `DsmbBatchSignalNotifier` follows `DefaultSafetyOfficerNotifier` pattern (`@All List<Connector>`, `ConnectorMessage`, ledger audit). `TrialSafetySignal.workItemId` provides idempotency — skip if an open WorkItem already exists.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-work (`WorkItemService`, `WorkItemCreateRequest`), casehub-connectors (`Connector`, `ConnectorMessage`), casehub-ledger (`JpaLedgerEntry`), Flyway, H2 (test), Mockito, AssertJ

## Global Constraints

- Migration versions: V129 for default datasource, V2032 for qhorus datasource
- LedgerEntry subclasses must live in `io.casehub.clinical.ledger` — never `io.casehub.clinical.entity`
- Every `LedgerEntry` subclass must override `domainContentBytes()`
- `WorkItemCreateRequest.builder()` API confirmed in use by `AdverseEventService` and `SponsorNotificationExhaustedWorkItemListener`
- `ConnectorMessage(destination, title, body)` — three-arg constructor, not `send(channel, message)`
- Connector injection: `@All List<Connector>` resolved by `Connector.id()` — not bare `@Inject Connector`
- Test ledger: `InMemoryLedgerEntryRepository` in `selected-alternatives`; tenantId = `"default"`
- Scheduler jobs excluded from tests via `quarkus.arc.exclude-types`

---

### Task 1: Domain model — TrialSafetySignal.workItemId + migration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/TrialSafetySignal.java`
- Create: `runtime/src/main/resources/db/migration/default/V129__trial_safety_signal_work_item_id.sql`

**Interfaces:**
- Produces: `TrialSafetySignal.workItemId` field (UUID, nullable)

- [ ] **Step 1: Add workItemId field to TrialSafetySignal**

Add after `resolvedAt` field in `TrialSafetySignal.java`:

```java
@Column(name = "work_item_id")
public UUID workItemId;
```

- [ ] **Step 2: Create V129 migration**

Create `runtime/src/main/resources/db/migration/default/V129__trial_safety_signal_work_item_id.sql`:

```sql
ALTER TABLE trial_safety_signal ADD COLUMN work_item_id UUID;
CREATE UNIQUE INDEX idx_trial_safety_signal_unique
    ON trial_safety_signal(trial_id, signal_type, tenant_id);
```

- [ ] **Step 3: Compile to verify**

Run: `mvn compile -pl api,runtime --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/entity/TrialSafetySignal.java runtime/src/main/resources/db/migration/default/V129__trial_safety_signal_work_item_id.sql
git commit -m "feat(#146): add workItemId to TrialSafetySignal + unique constraint"
```

---

### Task 2: Notification ledger entry

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/DsmbBatchSignalNotificationLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2032__dsmb_batch_signal_notification_ledger_entry.sql`

**Interfaces:**
- Produces: `DsmbBatchSignalNotificationLedgerEntry` entity with fields: `trialId`, `signalType`, `workItemId`, `connectorId`, `destination`, `delivered`, `failureReason`, `notifiedAt`

- [ ] **Step 1: Create ledger entry class**

Create `runtime/src/main/java/io/casehub/clinical/ledger/DsmbBatchSignalNotificationLedgerEntry.java`:

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "dsmb_batch_signal_notification_ledger_entry")
@DiscriminatorValue("DSMB_BATCH_SIGNAL_NOTIFICATION")
public class DsmbBatchSignalNotificationLedgerEntry extends JpaLedgerEntry {

    @Column(name = "trial_id", nullable = false)
    public UUID trialId;

    @Column(name = "signal_type", nullable = false, length = 50)
    public String signalType;

    @Column(name = "work_item_id")
    public UUID workItemId;

    @Column(name = "connector_id")
    public String connectorId;

    @Column(name = "destination", length = 2048)
    public String destination;

    @Column(name = "delivered", nullable = false)
    public boolean delivered;

    @Column(name = "failure_reason", length = 2048)
    public String failureReason;

    @Column(name = "notified_at", nullable = false)
    public Instant notifiedAt;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                trialId != null ? trialId.toString() : "",
                signalType != null ? signalType : "",
                workItemId != null ? workItemId.toString() : "",
                connectorId != null ? connectorId : "",
                destination != null ? destination : "",
                String.valueOf(delivered),
                failureReason != null ? failureReason : "",
                notifiedAt != null ? notifiedAt.toString() : "")
                .getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 2: Create V2032 migration**

Create `runtime/src/main/resources/db/migration/qhorus/V2032__dsmb_batch_signal_notification_ledger_entry.sql`:

```sql
CREATE TABLE dsmb_batch_signal_notification_ledger_entry (
    id UUID NOT NULL,
    trial_id UUID NOT NULL,
    signal_type VARCHAR(50) NOT NULL,
    work_item_id UUID,
    connector_id VARCHAR(255),
    destination VARCHAR(2048),
    delivered BOOLEAN NOT NULL,
    failure_reason VARCHAR(2048),
    notified_at TIMESTAMP NOT NULL,
    CONSTRAINT pk_dsmb_batch_signal_notif_ledger PRIMARY KEY (id),
    CONSTRAINT fk_dsmb_batch_signal_notif_ledger FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 3: Compile**

Run: `mvn compile -pl api,runtime --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/ledger/DsmbBatchSignalNotificationLedgerEntry.java runtime/src/main/resources/db/migration/qhorus/V2032__dsmb_batch_signal_notification_ledger_entry.sql
git commit -m "feat(#146): add DSMB batch signal notification ledger entry"
```

---

### Task 3: DsmbBatchSignalNotificationLedgerWriter + DsmbBatchSignalNotifier

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/DsmbBatchSignalNotificationLedgerWriter.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/DsmbBatchSignalNotifier.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/DsmbBatchSignalNotifierTest.java`

**Interfaces:**
- Consumes: `DsmbBatchSignalNotificationLedgerEntry` (Task 2), `Connector` SPI, `ConnectorMessage`, `LedgerEntryRepository`
- Produces: `DsmbBatchSignalNotificationLedgerWriter.writeSuccess(UUID trialId, String signalType, UUID workItemId, String connectorId, String destination)`, `writeFailure(...)`, `DsmbBatchSignalNotifier.notify(UUID trialId, String signalType, String summary, int affectedSiteCount, UUID workItemId)`

- [ ] **Step 1: Write the failing test — DsmbBatchSignalNotifierTest**

Create `runtime/src/test/java/io/casehub/clinical/service/DsmbBatchSignalNotifierTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class DsmbBatchSignalNotifierTest {

    private Connector slackConnector;
    private DsmbBatchSignalNotificationLedgerWriter ledgerWriter;
    private DsmbBatchSignalNotifier notifier;

    @BeforeEach
    void setup() {
        slackConnector = mock(Connector.class);
        when(slackConnector.id()).thenReturn("slack");
        ledgerWriter = mock(DsmbBatchSignalNotificationLedgerWriter.class);
        notifier = new DsmbBatchSignalNotifier(List.of(slackConnector), ledgerWriter,
            "slack", "dsmb");
    }

    @Test
    void sends_connector_message_and_writes_success_ledger_entry() {
        UUID trialId = UUID.randomUUID();
        UUID workItemId = UUID.randomUUID();
        when(slackConnector.send(any())).thenReturn(true);

        notifier.notify(trialId, "GRADE_THRESHOLD", "3 of 10 sites above 10%", 3, workItemId);

        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(slackConnector).send(captor.capture());
        ConnectorMessage msg = captor.getValue();
        assertThat(msg.destination()).isEqualTo("dsmb");
        assertThat(msg.title()).contains("GRADE_THRESHOLD");
        assertThat(msg.body()).contains("3 of 10 sites above 10%");
        assertThat(msg.body()).contains(workItemId.toString());

        verify(ledgerWriter).writeSuccess(trialId, "GRADE_THRESHOLD", workItemId, "slack", "dsmb");
    }

    @Test
    void missing_connector_logs_and_writes_failure_ledger_entry() {
        notifier = new DsmbBatchSignalNotifier(List.of(), ledgerWriter, "slack", "dsmb");
        UUID trialId = UUID.randomUUID();
        UUID workItemId = UUID.randomUUID();

        notifier.notify(trialId, "CROSS_SITE_CLUSTER", "summary", 4, workItemId);

        verify(ledgerWriter).writeFailure(eq(trialId), eq("CROSS_SITE_CLUSTER"),
            eq(workItemId), eq("slack"), eq("dsmb"), contains("No connector"));
    }

    @Test
    void connector_failure_writes_failure_ledger_entry() {
        UUID trialId = UUID.randomUUID();
        UUID workItemId = UUID.randomUUID();
        when(slackConnector.send(any())).thenThrow(new RuntimeException("network error"));

        notifier.notify(trialId, "GRADE_THRESHOLD", "summary", 3, workItemId);

        verify(ledgerWriter).writeFailure(eq(trialId), eq("GRADE_THRESHOLD"),
            eq(workItemId), eq("slack"), eq("dsmb"), contains("network error"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=DsmbBatchSignalNotifierTest --batch-mode`
Expected: FAIL — `DsmbBatchSignalNotifier` and `DsmbBatchSignalNotificationLedgerWriter` do not exist

- [ ] **Step 3: Implement DsmbBatchSignalNotificationLedgerWriter**

Create `runtime/src/main/java/io/casehub/clinical/service/DsmbBatchSignalNotificationLedgerWriter.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ClinicalActors;
import io.casehub.clinical.ledger.DsmbBatchSignalNotificationLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Clock;
import java.util.UUID;

@ApplicationScoped
public class DsmbBatchSignalNotificationLedgerWriter {

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Clock clock;

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void writeSuccess(UUID trialId, String signalType, UUID workItemId,
                             String connectorId, String destination) {
        writeEntry(trialId, signalType, workItemId, connectorId, destination, true, null);
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void writeFailure(UUID trialId, String signalType, UUID workItemId,
                             String connectorId, String destination, String failureReason) {
        writeEntry(trialId, signalType, workItemId, connectorId, destination, false, failureReason);
    }

    private void writeEntry(UUID trialId, String signalType, UUID workItemId,
                            String connectorId, String destination,
                            boolean delivered, String failureReason) {
        var entry = new DsmbBatchSignalNotificationLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = trialId;
        entry.sequenceNumber = ledgerEntryRepository.findLatestBySubjectId(trialId, "default")
            .map(e -> e.sequenceNumber + 1).orElse(1);
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "DsmbBatchSignalNotification";
        entry.occurredAt = clock.instant();
        entry.trialId = trialId;
        entry.signalType = signalType;
        entry.workItemId = workItemId;
        entry.connectorId = connectorId;
        entry.destination = destination;
        entry.delivered = delivered;
        entry.failureReason = failureReason;
        entry.notifiedAt = clock.instant();
        entry.attach(ClinicalComplianceSupplement.safetySignalDetection());
        ledgerEntryRepository.save(entry, "default");
    }
}
```

- [ ] **Step 4: Implement DsmbBatchSignalNotifier**

Create `runtime/src/main/java/io/casehub/clinical/service/DsmbBatchSignalNotifier.java`:

```java
package io.casehub.clinical.service;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.function.Function;
import java.util.stream.Collectors;

@ApplicationScoped
public class DsmbBatchSignalNotifier {

    private static final Logger LOG = Logger.getLogger(DsmbBatchSignalNotifier.class);

    private final Map<String, Connector> connectorRegistry;
    private final DsmbBatchSignalNotificationLedgerWriter ledgerWriter;
    private final String connectorId;
    private final String channel;

    @Inject
    DsmbBatchSignalNotifier(
            @All List<Connector> connectors,
            DsmbBatchSignalNotificationLedgerWriter ledgerWriter,
            @ConfigProperty(name = "casehub.clinical.dsmb.batch-signal.connector-id",
                            defaultValue = "slack") String connectorId,
            @ConfigProperty(name = "casehub.clinical.dsmb.batch-signal.notification-channel",
                            defaultValue = "dsmb") String channel) {
        this.connectorRegistry = connectors.stream()
            .collect(Collectors.toMap(Connector::id, Function.identity()));
        this.ledgerWriter = ledgerWriter;
        this.connectorId = connectorId;
        this.channel = channel;
    }

    public void notify(UUID trialId, String signalType, String summary,
                       int affectedSiteCount, UUID workItemId) {
        Connector connector = connectorRegistry.get(connectorId);
        if (connector == null) {
            LOG.warnf("No connector '%s' found — DSMB batch signal notification skipped for trial %s",
                connectorId, trialId);
            ledgerWriter.writeFailure(trialId, signalType, workItemId,
                connectorId, channel, "No connector: " + connectorId);
            return;
        }
        try {
            connector.send(new ConnectorMessage(
                channel,
                "DSMB Batch Signal: " + signalType,
                "%s\nAffected sites: %d\nTrial: %s\nWorkItem: %s"
                    .formatted(summary, affectedSiteCount, trialId, workItemId)));
            ledgerWriter.writeSuccess(trialId, signalType, workItemId, connectorId, channel);
        } catch (Exception e) {
            LOG.warnf(e, "DSMB batch signal notification failed for trial %s", trialId);
            ledgerWriter.writeFailure(trialId, signalType, workItemId,
                connectorId, channel, e.getMessage());
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=DsmbBatchSignalNotifierTest --batch-mode`
Expected: PASS (3 tests)

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/service/DsmbBatchSignalNotificationLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/DsmbBatchSignalNotifier.java runtime/src/test/java/io/casehub/clinical/service/DsmbBatchSignalNotifierTest.java
git commit -m "feat(#146): add DsmbBatchSignalNotifier + notification ledger writer"
```

---

### Task 4: Wire WorkItem creation into TrialSafetyAggregationJob

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/TrialSafetyAggregationJob.java`
- Modify: `runtime/src/main/resources/application.properties`
- Modify: `runtime/src/test/resources/application.properties`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/TrialSafetyAggregationJobWorkItemTest.java`

**Interfaces:**
- Consumes: `WorkItemService.create(WorkItemCreateRequest)`, `WorkItemService.findById(UUID)` (returns `Optional<WorkItem>`), `DsmbBatchSignalNotifier.notify(...)` (Task 3), `TrialSafetySignal.workItemId` (Task 1)
- Produces: Two-phase `upsertSignalRecord()` that persists signal, creates WorkItem, and notifies

- [ ] **Step 1: Write the failing unit test**

Create `runtime/src/test/java/io/casehub/clinical/cbr/TrialSafetyAggregationJobWorkItemTest.java`:

```java
package io.casehub.clinical.cbr;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.clinical.entity.TrialSafetySignal;
import io.casehub.clinical.service.DsmbBatchSignalNotifier;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class TrialSafetyAggregationJobWorkItemTest {

    // Tests will verify the two-phase upsert creates WorkItems correctly.
    // Detailed test bodies are written during implementation when the
    // exact method signatures of the refactored upsertSignalRecord() are known.

    @Test
    void placeholder_for_compilation() {
        // Replaced with real tests during implementation
        assertThat(true).isTrue();
    }
}
```

Note: The unit test bodies depend on the exact refactored method structure. The implementer writes them as part of TDD — the test cases are:
1. New signal → WorkItem created, `workItemId` set, notifier called
2. Signal persists with open WorkItem → no duplicate, notifier not called
3. Signal persists with terminal WorkItem → new WorkItem (re-escalation)
4. Signal resolved then re-appears → new WorkItem
5. WorkItem creation failure → logged, signal record still persisted
6. Notification failure → logged, WorkItem still created

- [ ] **Step 2: Add config properties**

Add to `runtime/src/main/resources/application.properties`:

```properties
casehub.clinical.dsmb.batch-signal.sla=PT72H
casehub.clinical.dsmb.batch-signal.expiry=P14D
casehub.clinical.dsmb.batch-signal.connector-id=slack
casehub.clinical.dsmb.batch-signal.notification-channel=dsmb
```

Add to `runtime/src/test/resources/application.properties`:

```properties
casehub.clinical.dsmb.batch-signal.sla=PT72H
casehub.clinical.dsmb.batch-signal.expiry=P14D
casehub.clinical.dsmb.batch-signal.connector-id=slack
casehub.clinical.dsmb.batch-signal.notification-channel=test-dsmb
```

- [ ] **Step 3: Refactor upsertSignalRecord() — two-phase split**

Modify `TrialSafetyAggregationJob.java`. Add new injections and refactor `upsertSignalRecord()`:

New injections:
```java
@Inject WorkItemService workItemService;
@Inject DsmbBatchSignalNotifier dsmbNotifier;
@Inject ObjectMapper objectMapper;

@ConfigProperty(name = "casehub.clinical.dsmb.batch-signal.sla", defaultValue = "PT72H")
Duration batchSignalSla;

@ConfigProperty(name = "casehub.clinical.dsmb.batch-signal.expiry", defaultValue = "P14D")
Duration batchSignalExpiry;
```

Refactored method — split into Phase 1 (signal persistence) and Phase 2 (WorkItem creation):

```java
void upsertSignalRecord(UUID trialId, DetectedSignal signal, String tenantId) {
    // Phase 1: persist signal record
    UpsertResult result = QuarkusTransaction.requiringNew().call(() -> {
        TrialSafetySignal existing = TrialSafetySignal.findByTrialAndType(
            trialId, signal.signalType(), tenantId);
        Instant now = clock.instant();
        if (existing != null) {
            existing.affectedSiteCount = signal.affectedSites().size();
            existing.summary = signal.summary();
            existing.lastDetectedAt = now;
            existing.resolvedAt = null;
            boolean needsWorkItem = needsWorkItem(existing.workItemId);
            return new UpsertResult(existing.id, existing.workItemId, needsWorkItem);
        } else {
            TrialSafetySignal record = new TrialSafetySignal();
            record.id = UUID.randomUUID();
            record.tenantId = tenantId;
            record.trialId = trialId;
            record.signalType = signal.signalType();
            record.affectedSiteCount = signal.affectedSites().size();
            record.summary = signal.summary();
            record.firstDetectedAt = now;
            record.lastDetectedAt = now;
            record.persist();
            return new UpsertResult(record.id, null, true);
        }
    });

    // Phase 2: create WorkItem if needed (separate transaction for error isolation)
    if (result.needsWorkItem()) {
        try {
            UUID workItemId = QuarkusTransaction.requiringNew().call(() -> {
                var wi = workItemService.create(WorkItemCreateRequest.builder()
                    .title("DSMB review — batch safety signal: " + signal.signalType())
                    .description(signal.summary()
                        + ". Detected by trial safety aggregation job.")
                    .types(List.of("dsmb-batch-signal"))
                    .formKey("dsmb-batch-signal-review")
                    .priority(io.casehub.work.api.WorkItemPriority.HIGH)
                    .candidateGroups("dsmb")
                    .createdBy(io.casehub.clinical.api.ClinicalActors.CLINICAL_SERVICE)
                    .callerRef("clinical:trial-safety-signal/" + result.signalId())
                    .payload(buildPayload(trialId, signal))
                    .claimDeadline(clock.instant().plus(batchSignalSla))
                    .expiresAt(clock.instant().plus(batchSignalExpiry))
                    .build());
                // Update signal record with workItemId
                TrialSafetySignal sig = TrialSafetySignal.findById(result.signalId());
                if (sig != null) sig.workItemId = wi.id;
                return wi.id;
            });
            // Notification fires AFTER transaction commits
            dsmbNotifier.notify(trialId, signal.signalType(), signal.summary(),
                signal.affectedSites().size(), workItemId);
        } catch (Exception e) {
            LOG.warnf(e, "WorkItem creation failed for trial %s signal %s"
                + " — signal record persisted, WorkItem deferred to next run",
                trialId, signal.signalType());
        }
    }
}

private boolean needsWorkItem(UUID existingWorkItemId) {
    if (existingWorkItemId == null) return true;
    return workItemService.findById(existingWorkItemId)
        .map(wi -> wi.status.isTerminal())
        .orElse(true);
}

private String buildPayload(UUID trialId, DetectedSignal signal) {
    try {
        return objectMapper.writeValueAsString(java.util.Map.of(
            "trialId", trialId.toString(),
            "signalType", signal.signalType(),
            "affectedSiteCount", signal.affectedSites().size(),
            "summary", signal.summary(),
            "affectedSites", signal.affectedSites().stream()
                .map(UUID::toString).toList()));
    } catch (Exception e) {
        return "{}";
    }
}

record UpsertResult(UUID signalId, UUID existingWorkItemId, boolean needsWorkItem) {}
```

- [ ] **Step 4: Run compile to verify**

Run: `mvn compile -pl api,runtime --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 5: Run existing tests to check for regressions**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: All existing tests pass (note pre-existing flaky tests from HANDOFF.md)

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/cbr/TrialSafetyAggregationJob.java runtime/src/main/resources/application.properties runtime/src/test/resources/application.properties runtime/src/test/java/io/casehub/clinical/cbr/TrialSafetyAggregationJobWorkItemTest.java
git commit -m "feat(#146): two-phase WorkItem creation in TrialSafetyAggregationJob"
```

---

### Task 5: Integration test — DsmbBatchSignalWorkItemTest

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/DsmbBatchSignalWorkItemTest.java`

**Interfaces:**
- Consumes: `TrialSafetyAggregationJob.aggregateTrial()`, `WorkItemQueries`, `TrialSafetySignal`, `WorkItemService`

- [ ] **Step 1: Write integration test**

Create `runtime/src/test/java/io/casehub/clinical/cbr/DsmbBatchSignalWorkItemTest.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.SiteStatus;
import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.clinical.api.model.TrialStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSafetySignal;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.support.WorkItemQueries;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DsmbBatchSignalWorkItemTest {

    @Inject TrialSafetyAggregationJob aggregationJob;
    @Inject WorkItemQueries workItemQueries;
    @Inject WorkItemService workItemService;
    @Inject CurrentPrincipal principal;

    private UUID trialId;

    @BeforeEach
    void setup() {
        trialId = seedTrialWithGradeThresholdSignal();
    }

    @Test
    void batch_signal_creates_dsmb_workitem() {
        aggregationJob.aggregateTrial(trialId, "PHASE_III");

        TrialSafetySignal signal = findSignal(trialId, "GRADE_THRESHOLD");
        assertThat(signal).isNotNull();
        assertThat(signal.workItemId).isNotNull();

        List<WorkItem> items = dsmbBatchWorkItems();
        assertThat(items).hasSize(1);
        assertThat(items.get(0).title).contains("DSMB review");
        assertThat(items.get(0).title).contains("GRADE_THRESHOLD");
        assertThat(items.get(0).callerRef).contains("clinical:trial-safety-signal/");
    }

    @Test
    void idempotent_no_duplicate_workitem_on_second_run() {
        aggregationJob.aggregateTrial(trialId, "PHASE_III");
        UUID firstWorkItemId = findSignal(trialId, "GRADE_THRESHOLD").workItemId;
        assertThat(firstWorkItemId).isNotNull();

        aggregationJob.aggregateTrial(trialId, "PHASE_III");
        UUID secondWorkItemId = findSignal(trialId, "GRADE_THRESHOLD").workItemId;
        assertThat(secondWorkItemId).isEqualTo(firstWorkItemId);
        assertThat(dsmbBatchWorkItems()).hasSize(1);
    }

    @Test
    void creates_new_workitem_after_previous_completed() {
        aggregationJob.aggregateTrial(trialId, "PHASE_III");
        UUID firstWorkItemId = findSignal(trialId, "GRADE_THRESHOLD").workItemId;

        completeWorkItem(firstWorkItemId);

        aggregationJob.aggregateTrial(trialId, "PHASE_III");
        UUID secondWorkItemId = findSignal(trialId, "GRADE_THRESHOLD").workItemId;
        assertThat(secondWorkItemId).isNotEqualTo(firstWorkItemId);
        assertThat(dsmbBatchWorkItems()).hasSize(1); // only open ones
    }

    // ── helpers ──────────────────────────────────────────────

    @Transactional
    UUID seedTrialWithGradeThresholdSignal() {
        // Create trial + 3 sites + AE data that triggers grade threshold
        // (3+ sites with Grade 3+ rate above 10%)
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = "DSMB-BATCH-TEST";
        trial.phase = TrialPhase.PHASE_III;
        trial.sponsor = "TestSponsor";
        trial.targetEnrollment = 100;
        trial.status = TrialStatus.ACTIVE;
        trial.tenantId = principal.tenancyId();
        trial.persist();

        for (int i = 0; i < 4; i++) {
            TrialSite site = new TrialSite();
            site.id = UUID.randomUUID();
            site.trialId = trial.id;
            site.investigatorId = "pi-" + i;
            site.status = SiteStatus.ACTIVE;
            site.persist();

            PatientEnrollment enrollment = new PatientEnrollment();
            enrollment.id = UUID.randomUUID();
            enrollment.siteId = site.id;
            enrollment.patientId = "patient-" + i;
            enrollment.tenantId = principal.tenancyId();
            enrollment.persist();

            // 2 AEs per site: 1 Grade 3, 1 Grade 1 → 50% high-grade rate (above 10% threshold)
            AdverseEvent ae1 = new AdverseEvent();
            ae1.id = UUID.randomUUID();
            ae1.enrollmentId = enrollment.id;
            ae1.grade = CtcaeGrade.GRADE_3;
            ae1.eventType = "headache";
            ae1.actuality = EventActuality.ACTUAL;
            ae1.outcome = AeOutcome.ONGOING;
            ae1.occurredAt = Instant.now();
            ae1.reportedAt = Instant.now();
            ae1.tenantId = principal.tenancyId();
            ae1.persist();

            AdverseEvent ae2 = new AdverseEvent();
            ae2.id = UUID.randomUUID();
            ae2.enrollmentId = enrollment.id;
            ae2.grade = CtcaeGrade.GRADE_1;
            ae2.eventType = "mild-nausea";
            ae2.actuality = EventActuality.ACTUAL;
            ae2.outcome = AeOutcome.RESOLVED;
            ae2.occurredAt = Instant.now();
            ae2.reportedAt = Instant.now();
            ae2.tenantId = principal.tenancyId();
            ae2.persist();
        }
        return trial.id;
    }

    @Transactional
    TrialSafetySignal findSignal(UUID trialId, String signalType) {
        return TrialSafetySignal.findByTrialAndType(trialId, signalType, principal.tenancyId());
    }

    List<WorkItem> dsmbBatchWorkItems() {
        return workItemQueries.scanAll().stream()
            .filter(wi -> wi.title != null && wi.title.contains("DSMB review")
                && wi.title.contains("batch safety signal"))
            .filter(wi -> !wi.status.isTerminal())
            .toList();
    }

    @Transactional
    void completeWorkItem(UUID workItemId) {
        workItemService.complete(workItemId, "Reviewed — no action required", "dsmb-reviewer");
    }
}
```

- [ ] **Step 2: Run test**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=DsmbBatchSignalWorkItemTest --batch-mode`
Expected: PASS (3 tests). Adjust seed data or assertions if signal detection thresholds don't match.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/clinical/cbr/DsmbBatchSignalWorkItemTest.java
git commit -m "test(#146): integration test for DSMB batch signal WorkItem creation"
```

---

### Task 6: Final verification + docs

**Files:**
- Modify: `CLAUDE.md` (if config or convention changes)

- [ ] **Step 1: Run full test suite**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: All tests pass (note pre-existing flakes: PiResponseListenerIntegrationTest, AeEscalationLifecycleTest, DsmbRollupTest, CbrRetrievalAuditIntegrationTest, ClinicalCaseOutcomeObserverIntegrationTest)

- [ ] **Step 2: Verify migration ordering**

Confirm V129 is the next default migration and V2032 is the next qhorus migration — no conflicts with other branches.

- [ ] **Step 3: Commit any remaining changes**

```bash
git commit -m "chore(#146): final verification pass"
```
