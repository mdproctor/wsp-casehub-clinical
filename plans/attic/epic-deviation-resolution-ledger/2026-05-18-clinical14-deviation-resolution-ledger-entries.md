# Protocol Deviation Resolution Ledger Entries — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Write tamper-evident `ProtocolDeviationLedgerEntry` records for PI response (APPROVED/REJECTED/ESCALATED) and deadline expiration (EXPIRED), and fix the hardcoded `sequenceNumber=1` bug.

**Architecture:** A new `DeviationLedgerWriter @ApplicationScoped` centralises all ledger writing for protocol deviations. It computes `sequenceNumber` dynamically via `findLatestBySubjectId` and provides two named methods: `writeCommandEntry` (called from `ProtocolDeviationService`) and `writeResolutionEntry` (called from `PiResponseListener` and `DeviationExpirationJob`). Two nullable columns (`terminal_status`, `resolved_at`) are added to `protocol_deviation_ledger_entry` via Flyway V1007.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache Active Record, casehub-ledger (`LedgerEntryRepository`, `LedgerEntryType`, `ActorType`), Flyway (qhorus datasource), Mockito (`mockito-junit-jupiter`), AssertJ.

**Issues:** Refs casehubio/clinical#14, Refs casehubio/clinical#15

---

## File Map

| Action | File |
|--------|------|
| CREATE | `runtime/src/main/resources/db/migration/qhorus/V1007__deviation_resolution_fields.sql` |
| MODIFY | `runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java` |
| MODIFY | `runtime/pom.xml` |
| CREATE | `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java` |
| CREATE | `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java` |
| MODIFY | `runtime/src/main/java/io/casehub/clinical/service/ProtocolDeviationService.java` |
| MODIFY | `runtime/src/test/java/io/casehub/clinical/service/ProtocolDeviationServiceTest.java` |
| MODIFY | `runtime/src/main/java/io/casehub/clinical/service/PiResponseListener.java` |
| MODIFY | `runtime/src/test/java/io/casehub/clinical/service/PiResponseListenerTest.java` |
| MODIFY | `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java` |
| MODIFY | `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java` |

---

## Task 1: Schema Foundation — Migration and Entity Fields

**Files:**
- Create: `runtime/src/main/resources/db/migration/qhorus/V1007__deviation_resolution_fields.sql`
- Modify: `runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java`

- [ ] **Step 1.1: Write the Flyway migration**

Create `runtime/src/main/resources/db/migration/qhorus/V1007__deviation_resolution_fields.sql`:

```sql
ALTER TABLE protocol_deviation_ledger_entry
    ADD COLUMN terminal_status VARCHAR(50),
    ADD COLUMN resolved_at     TIMESTAMP WITH TIME ZONE;
```

Nullable because existing COMMAND entries predate these columns. New COMMAND entries also leave them null — only resolution entries populate them.

- [ ] **Step 1.2: Add entity fields**

In `runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java`, add two fields after `escalationRequirement`:

```java
@Column(name = "terminal_status")
public String terminalStatus;

@Column(name = "resolved_at")
public Instant resolvedAt;
```

Full file after change:

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "protocol_deviation_ledger_entry")
@DiscriminatorValue("PROTOCOL_DEVIATION")
public class ProtocolDeviationLedgerEntry extends LedgerEntry {

    @Column(name = "deviation_id")
    public UUID deviationId;

    @Column(name = "site_id")
    public UUID siteId;

    public String severity;

    @Column(name = "pi_id")
    public String piId;

    @Column(name = "commanded_at")
    public Instant commandedAt;

    @Column(name = "response_deadline")
    public Instant responseDeadline;

    @Column(name = "escalation_requirement")
    public String escalationRequirement;

    @Column(name = "terminal_status")
    public String terminalStatus;

    @Column(name = "resolved_at")
    public Instant resolvedAt;
}
```

- [ ] **Step 1.3: Verify compile**

```bash
mvn compile -pl api,runtime --batch-mode
```

Expected: `BUILD SUCCESS`

- [ ] **Step 1.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/resources/db/migration/qhorus/V1007__deviation_resolution_fields.sql \
  runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "feat(schema): add terminal_status and resolved_at to protocol_deviation_ledger_entry — Refs #14 #15"
```

---

## Task 2: `DeviationLedgerWriterTest` — Failing Unit Tests

**Files:**
- Modify: `runtime/pom.xml`
- Create: `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java`

- [ ] **Step 2.1: Add mockito-junit-jupiter to runtime/pom.xml**

In `runtime/pom.xml`, add inside `<dependencies>` after the assertj dependency:

```xml
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
```

The version is managed by the Quarkus platform BOM — no version element needed.

- [ ] **Step 2.2: Write all unit tests**

Create `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class DeviationLedgerWriterTest {

    @Mock
    LedgerEntryRepository ledgerEntryRepository;

    @InjectMocks
    DeviationLedgerWriter writer;

    private ProtocolDeviation dev;

    @BeforeEach
    void setUp() {
        dev = new ProtocolDeviation();
        dev.id = UUID.randomUUID();
        dev.siteId = UUID.randomUUID();
        dev.severity = DeviationSeverity.MINOR;
        dev.escalationRequirement = EscalationRequirement.NONE;
        dev.commandedAt = Instant.now();
        dev.responseDeadline = Instant.now().plusSeconds(3600);
        when(ledgerEntryRepository.save(any())).thenAnswer(i -> i.getArgument(0));
    }

    @Test
    void writeCommandEntry_sequenceNumber1WhenNoPriorEntries() {
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.empty());

        writer.writeCommandEntry(dev, "pi-001");

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.sequenceNumber).isEqualTo(1);
    }

    @Test
    void writeCommandEntry_sequenceNumberIncrements() {
        LedgerEntry prior = new ProtocolDeviationLedgerEntry();
        prior.sequenceNumber = 2;
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.of(prior));

        writer.writeCommandEntry(dev, "pi-001");

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.sequenceNumber).isEqualTo(3);
    }

    @Test
    void writeCommandEntry_setsCorrectFields() {
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.empty());

        writer.writeCommandEntry(dev, "pi-001");

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.entryType).isEqualTo(LedgerEntryType.COMMAND);
        assertThat(entry.actorId).isEqualTo("clinical-service");
        assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
        assertThat(entry.actorRole).isEqualTo("deviation-reporter");
        assertThat(entry.subjectId).isEqualTo(dev.id);
        assertThat(entry.deviationId).isEqualTo(dev.id);
        assertThat(entry.siteId).isEqualTo(dev.siteId);
        assertThat(entry.severity).isEqualTo("MINOR");
        assertThat(entry.piId).isEqualTo("pi-001");
        assertThat(entry.terminalStatus).isNull();
        assertThat(entry.resolvedAt).isNull();
        assertThat(entry.id).isNotNull();
    }

    @Test
    void writeResolutionEntry_approved_setsCorrectFields() {
        LedgerEntry prior = new ProtocolDeviationLedgerEntry();
        prior.sequenceNumber = 1;
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.of(prior));

        writer.writeResolutionEntry(dev, PiApprovalStatus.APPROVED, "pi-actor-001", ActorType.HUMAN, "pi-authoriser");

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.entryType).isEqualTo(LedgerEntryType.EVENT);
        assertThat(entry.sequenceNumber).isEqualTo(2);
        assertThat(entry.actorId).isEqualTo("pi-actor-001");
        assertThat(entry.actorType).isEqualTo(ActorType.HUMAN);
        assertThat(entry.actorRole).isEqualTo("pi-authoriser");
        assertThat(entry.terminalStatus).isEqualTo("APPROVED");
        assertThat(entry.resolvedAt).isNotNull();
        assertThat(entry.subjectId).isEqualTo(dev.id);
    }

    @Test
    void writeResolutionEntry_escalated_setsTerminalStatus() {
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.empty());

        writer.writeResolutionEntry(dev, PiApprovalStatus.ESCALATED, "pi-actor-001", ActorType.HUMAN, "pi-authoriser");

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.terminalStatus).isEqualTo("ESCALATED");
    }

    @Test
    void writeResolutionEntry_rejected_setsTerminalStatus() {
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.empty());

        writer.writeResolutionEntry(dev, PiApprovalStatus.REJECTED, "pi-actor-001", ActorType.HUMAN, "pi-authoriser");

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.terminalStatus).isEqualTo("REJECTED");
        assertThat(entry.actorType).isEqualTo(ActorType.HUMAN);
    }

    @Test
    void writeResolutionEntry_expired_setsSystemActor() {
        when(ledgerEntryRepository.findLatestBySubjectId(dev.id)).thenReturn(Optional.empty());

        writer.writeResolutionEntry(dev, PiApprovalStatus.EXPIRED, "system", ActorType.SYSTEM, "deviation-expiration-job");

        ProtocolDeviationLedgerEntry entry = captureEntry();
        assertThat(entry.terminalStatus).isEqualTo("EXPIRED");
        assertThat(entry.actorId).isEqualTo("system");
        assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
        assertThat(entry.actorRole).isEqualTo("deviation-expiration-job");
    }

    private ProtocolDeviationLedgerEntry captureEntry() {
        ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerEntryRepository).save(captor.capture());
        return (ProtocolDeviationLedgerEntry) captor.getValue();
    }
}
```

- [ ] **Step 2.3: Run tests and verify they fail**

```bash
mvn test -pl runtime -Dtest=DeviationLedgerWriterTest --batch-mode
```

Expected: FAIL — `DeviationLedgerWriter` does not exist yet.

- [ ] **Step 2.4: Commit failing tests**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/pom.xml \
  runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "test: DeviationLedgerWriterTest — failing unit tests for ledger writer — Refs #14 #15"
```

---

## Task 3: `DeviationLedgerWriter` Implementation

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java`

- [ ] **Step 3.1: Write the implementation**

Create `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.UUID;

@ApplicationScoped
public class DeviationLedgerWriter {

    static final String SYSTEM_ACTOR = "clinical-service";

    @Inject
    LedgerEntryRepository ledgerEntryRepository;

    public void writeCommandEntry(ProtocolDeviation dev, String piId) {
        ProtocolDeviationLedgerEntry entry = baseEntry(dev);
        entry.entryType = LedgerEntryType.COMMAND;
        entry.actorId = SYSTEM_ACTOR;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "deviation-reporter";
        entry.occurredAt = dev.commandedAt;
        entry.piId = piId;
        entry.commandedAt = dev.commandedAt;
        entry.responseDeadline = dev.responseDeadline;
        entry.escalationRequirement = dev.escalationRequirement != null
            ? dev.escalationRequirement.name() : null;
        ledgerEntryRepository.save(entry);
    }

    public void writeResolutionEntry(ProtocolDeviation dev, PiApprovalStatus terminalStatus,
                                     String actorId, ActorType actorType, String actorRole) {
        ProtocolDeviationLedgerEntry entry = baseEntry(dev);
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = actorType;
        entry.actorRole = actorRole;
        entry.occurredAt = Instant.now();
        entry.terminalStatus = terminalStatus.name();
        entry.resolvedAt = entry.occurredAt;
        ledgerEntryRepository.save(entry);
    }

    private ProtocolDeviationLedgerEntry baseEntry(ProtocolDeviation dev) {
        ProtocolDeviationLedgerEntry entry = new ProtocolDeviationLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = dev.id;
        entry.sequenceNumber = nextSequenceNumber(dev.id);
        entry.deviationId = dev.id;
        entry.siteId = dev.siteId;
        entry.severity = dev.severity.name();
        return entry;
    }

    private int nextSequenceNumber(UUID deviationId) {
        return ledgerEntryRepository.findLatestBySubjectId(deviationId)
            .map(e -> e.sequenceNumber + 1)
            .orElse(1);
    }
}
```

- [ ] **Step 3.2: Run unit tests and verify they pass**

```bash
mvn test -pl runtime -Dtest=DeviationLedgerWriterTest --batch-mode
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`

- [ ] **Step 3.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "feat: DeviationLedgerWriter — centralised ledger writing with dynamic sequenceNumber — Refs #14 #15"
```

---

## Task 4: Refactor `ProtocolDeviationService` + Extend Integration Test

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ProtocolDeviationService.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/ProtocolDeviationServiceTest.java`

- [ ] **Step 4.1: Add ledger assertions to the existing integration test**

In `ProtocolDeviationServiceTest`, the test at `@Order(5)` currently checks:
```java
assertThat(entries.get(0)).isInstanceOf(ProtocolDeviationLedgerEntry.class);
ProtocolDeviationLedgerEntry entry = (ProtocolDeviationLedgerEntry) entries.get(0);
assertThat(entry.deviationId).isEqualTo(deviationId);
```

Extend it to also assert the new fields:

```java
@Test
@Order(5)
@Transactional
void ledgerEntryIsWritten() {
    assertThat(deviationId).as("deviationId set by Order(1)").isNotNull();
    var entries = ledgerRepo.findBySubjectId(deviationId);
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0)).isInstanceOf(ProtocolDeviationLedgerEntry.class);
    ProtocolDeviationLedgerEntry entry = (ProtocolDeviationLedgerEntry) entries.get(0);
    assertThat(entry.deviationId).isEqualTo(deviationId);
    assertThat(entry.sequenceNumber).isEqualTo(1);
    assertThat(entry.terminalStatus).isNull();
    assertThat(entry.resolvedAt).isNull();
    assertThat(entry.actorId).isEqualTo("clinical-service");
}
```

- [ ] **Step 4.2: Run the integration test and confirm it passes**

```bash
mvn test -pl runtime -Dtest=ProtocolDeviationServiceTest --batch-mode
```

Expected: `Tests run: 6, Failures: 0` — these assertions describe existing correct behaviour.

- [ ] **Step 4.3: Refactor `ProtocolDeviationService` to use `DeviationLedgerWriter`**

Replace the entire content of `ProtocolDeviationService.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.api.spi.DeviationContext;
import io.casehub.clinical.api.spi.DeviationResponsePolicy;
import io.casehub.clinical.api.spi.DeviationResponseRequirements;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.time.Instant;

@ApplicationScoped
public class ProtocolDeviationService {

    static final String CLINICAL_SENDER = "clinical-service";
    static final String CHANNEL_ALLOWED_TYPES = "QUERY,COMMAND";

    @Inject DeviationResponsePolicy policy;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject DeviationLedgerWriter ledgerWriter;

    @Transactional
    public void reportDeviation(ProtocolDeviation deviation) {
        TrialSite site = TrialSite.findById(deviation.siteId);
        ClinicalTrial trial = ClinicalTrial.findById(site.trialId);

        var context = new DeviationContext(
            deviation.id, deviation.siteId, site.trialId,
            trial.protocolId, trial.phase, deviation.severity, deviation.deviationType
        );
        DeviationResponseRequirements requirements = policy.evaluate(context);

        String channelName = "clinical/deviation/" + deviation.id + "/pi-oversight";
        ensureChannel(channelName);

        var channel = channelService.findByName(channelName).orElseThrow();
        Instant now = Instant.now();
        Instant responseDeadline = now.plus(requirements.piResponseDeadline());
        String content = buildCommandContent(deviation, responseDeadline);

        messageService.send(
            channel.id,
            CLINICAL_SENDER,
            MessageType.COMMAND,
            content,
            deviation.id.toString(),
            null,
            null,
            site.investigatorId,
            ActorType.SYSTEM
        );

        deviation.piCommandChannelName = channelName;
        deviation.commandedAt = now;
        deviation.responseDeadline = responseDeadline;
        deviation.escalationRequirement = requirements.escalationRequirement();
        deviation.piApprovalStatus = PiApprovalStatus.COMMANDED;
        deviation.persist();

        ledgerWriter.writeCommandEntry(deviation, site.investigatorId);
    }

    private void ensureChannel(String name) {
        if (channelService.findByName(name).isPresent()) {
            return;
        }
        channelService.create(
            name,
            "PI governance channel for protocol deviation",
            ChannelSemantic.APPEND,
            null, null, null, null, null,
            CHANNEL_ALLOWED_TYPES
        );
    }

    private String buildCommandContent(ProtocolDeviation dev, Instant responseDeadline) {
        return "{\"deviationId\":\"" + dev.id
            + "\",\"deviationType\":\"" + dev.deviationType
            + "\",\"severity\":\"" + dev.severity
            + "\",\"responseDeadline\":\"" + responseDeadline
            + "\"}";
    }
}
```

- [ ] **Step 4.4: Run full test suite and confirm no regressions**

```bash
mvn test -pl runtime --batch-mode
```

Expected: all previously passing tests still pass.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/ProtocolDeviationService.java \
  runtime/src/test/java/io/casehub/clinical/service/ProtocolDeviationServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "refactor: ProtocolDeviationService delegates ledger writing to DeviationLedgerWriter — Refs #14 #15"
```

---

## Task 5: Wire `PiResponseListener` + Integration Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/PiResponseListener.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/PiResponseListenerTest.java`

- [ ] **Step 5.1: Add failing ledger assertions to `PiResponseListenerTest`**

Add `@Inject LedgerEntryRepository ledgerRepo;` to the class fields.

Add three new test methods at `@Order(6)`, `@Order(7)`, `@Order(8)`:

```java
@Inject LedgerEntryRepository ledgerRepo;

@Test @Order(6)
@Transactional
void approvedMinorLedgerEntryWritten() {
    var entries = ledgerRepo.findBySubjectId(minorDeviationId);
    assertThat(entries).hasSize(1);
    ProtocolDeviationLedgerEntry entry = (ProtocolDeviationLedgerEntry) entries.get(0);
    assertThat(entry.terminalStatus).isEqualTo("APPROVED");
    assertThat(entry.actorId).isEqualTo("human:pi-L");
    assertThat(entry.actorType).isEqualTo(ActorType.HUMAN);
    assertThat(entry.actorRole).isEqualTo("pi-authoriser");
    assertThat(entry.resolvedAt).isNotNull();
    assertThat(entry.sequenceNumber).isEqualTo(1);
}

@Test @Order(7)
@Transactional
void escalatedCriticalLedgerEntryWritten() {
    var entries = ledgerRepo.findBySubjectId(criticalDeviationId);
    assertThat(entries).hasSize(1);
    ProtocolDeviationLedgerEntry entry = (ProtocolDeviationLedgerEntry) entries.get(0);
    assertThat(entry.terminalStatus).isEqualTo("ESCALATED");
    assertThat(entry.actorId).isEqualTo("human:pi-L");
}

@Test @Order(8)
@Transactional
void rejectedLedgerEntryWritten() {
    var entries = ledgerRepo.findBySubjectId(rejectedDeviationId);
    assertThat(entries).hasSize(1);
    ProtocolDeviationLedgerEntry entry = (ProtocolDeviationLedgerEntry) entries.get(0);
    assertThat(entry.terminalStatus).isEqualTo("REJECTED");
    assertThat(entry.actorId).isEqualTo("human:pi-L");
}
```

Add the necessary imports to the file:
```java
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
```

- [ ] **Step 5.2: Run the new tests and confirm they fail**

```bash
mvn test -pl runtime -Dtest=PiResponseListenerTest --batch-mode
```

Expected: Orders 6-8 FAIL — `entries` is empty because `PiResponseListener` doesn't write ledger entries yet.

- [ ] **Step 5.3: Wire `DeviationLedgerWriter` into `PiResponseListener`**

Replace `PiResponseListener.java` with:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.CommitmentService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.util.UUID;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@ApplicationScoped
public class PiResponseListener {

    private static final Pattern CHANNEL_PATTERN =
        Pattern.compile("clinical/deviation/([0-9a-f-]+)/pi-oversight");

    @Inject CommitmentService commitmentService;
    @Inject Event<ProtocolDeviationResolvedEvent> resolvedEvent;
    @Inject DeviationLedgerWriter ledgerWriter;

    // @ObservesAsync MessageReceivedEvent — awaiting casehubio/qhorus#153
    // When qhorus#153 ships and casehub-qhorus-api is updated, add:
    //
    // void onMessage(@ObservesAsync io.casehub.qhorus.api.gateway.MessageReceivedEvent event) {
    //     if (event.messageType() != MessageType.DONE && event.messageType() != MessageType.DECLINE) return;
    //     process(event.channelName(), event.messageType(), event.senderId());
    // }

    @Transactional
    public void process(String channelName, MessageType messageType, String senderId) {
        if (messageType != MessageType.DONE && messageType != MessageType.DECLINE) return;

        Matcher m = CHANNEL_PATTERN.matcher(channelName);
        if (!m.matches()) return;

        UUID deviationId = UUID.fromString(m.group(1));
        ProtocolDeviation deviation = ProtocolDeviation.findById(deviationId);
        if (deviation == null) return;
        if (deviation.piApprovalStatus != PiApprovalStatus.COMMANDED) return;

        boolean approved = messageType == MessageType.DONE;

        if (approved) {
            boolean needsEscalation = deviation.escalationRequirement != null
                && deviation.escalationRequirement != EscalationRequirement.NONE;
            deviation.piApprovalStatus = needsEscalation
                ? PiApprovalStatus.ESCALATED : PiApprovalStatus.APPROVED;
            commitmentService.fulfill(deviationId.toString());
        } else {
            deviation.piApprovalStatus = PiApprovalStatus.REJECTED;
            commitmentService.decline(deviationId.toString());
        }

        ledgerWriter.writeResolutionEntry(deviation, deviation.piApprovalStatus,
            senderId, ActorType.HUMAN, "pi-authoriser");

        resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
            deviation.id, deviation.siteId, deviation.severity,
            deviation.escalationRequirement != null
                ? deviation.escalationRequirement : EscalationRequirement.NONE,
            deviation.piApprovalStatus
        ));
    }
}
```

- [ ] **Step 5.4: Run and confirm tests pass**

```bash
mvn test -pl runtime -Dtest=PiResponseListenerTest --batch-mode
```

Expected: `Tests run: 8, Failures: 0, Errors: 0`

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/PiResponseListener.java \
  runtime/src/test/java/io/casehub/clinical/service/PiResponseListenerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "feat: PiResponseListener writes resolution ledger entry on PI response — Refs #14"
```

---

## Task 6: Wire `DeviationExpirationJob` + Integration Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java`

- [ ] **Step 6.1: Add failing ledger assertions to `DeviationExpirationJobTest`**

Add `@Inject LedgerEntryRepository ledgerRepo;` to class fields.

Extend the `overdueCommandedDeviationIsMarkedExpired` test with ledger assertions, and add a new two-deviation test:

```java
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;

// Add field:
@Inject LedgerEntryRepository ledgerRepo;

@Test
@Transactional
void overdueCommandedDeviationIsMarkedExpired() {
    ProtocolDeviation dev = new ProtocolDeviation();
    dev.id = UUID.randomUUID();
    dev.siteId = siteId;
    dev.deviationType = "overdue"; dev.severity = DeviationSeverity.MINOR;
    dev.piApprovalStatus = PiApprovalStatus.COMMANDED;
    dev.escalationRequirement = EscalationRequirement.NONE;
    dev.piCommandChannelName = "clinical/deviation/" + dev.id + "/pi-oversight";
    dev.commandedAt = Instant.now().minus(10, ChronoUnit.DAYS);
    dev.responseDeadline = Instant.now().minus(3, ChronoUnit.DAYS);
    dev.persist();

    job.checkExpiredCommitments();

    ProtocolDeviation loaded = ProtocolDeviation.findById(dev.id);
    assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.EXPIRED);

    var entries = ledgerRepo.findBySubjectId(dev.id);
    assertThat(entries).hasSize(1);
    ProtocolDeviationLedgerEntry entry = (ProtocolDeviationLedgerEntry) entries.get(0);
    assertThat(entry.terminalStatus).isEqualTo("EXPIRED");
    assertThat(entry.actorId).isEqualTo("system");
    assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
    assertThat(entry.actorRole).isEqualTo("deviation-expiration-job");
    assertThat(entry.resolvedAt).isNotNull();
    assertThat(entry.sequenceNumber).isEqualTo(1);
}

@Test
@Transactional
void twoOverdueDeviationsEachGetIndependentLedgerEntry() {
    UUID devId1 = UUID.randomUUID(), devId2 = UUID.randomUUID();
    for (UUID id : new UUID[]{devId1, devId2}) {
        ProtocolDeviation d = new ProtocolDeviation();
        d.id = id; d.siteId = siteId;
        d.deviationType = "overdue"; d.severity = DeviationSeverity.MINOR;
        d.piApprovalStatus = PiApprovalStatus.COMMANDED;
        d.escalationRequirement = EscalationRequirement.NONE;
        d.piCommandChannelName = "clinical/deviation/" + id + "/pi-oversight";
        d.commandedAt = Instant.now().minus(10, ChronoUnit.DAYS);
        d.responseDeadline = Instant.now().minus(1, ChronoUnit.DAYS);
        d.persist();
    }

    job.checkExpiredCommitments();

    assertThat(ledgerRepo.findBySubjectId(devId1)).hasSize(1);
    assertThat(ledgerRepo.findBySubjectId(devId2)).hasSize(1);
}
```

- [ ] **Step 6.2: Run and confirm new assertions fail**

```bash
mvn test -pl runtime -Dtest=DeviationExpirationJobTest --batch-mode
```

Expected: `overdueCommandedDeviationIsMarkedExpired` FAILS on ledger assertions; `twoOverdue...` FAILS — no entries written yet.

- [ ] **Step 6.3: Wire `DeviationLedgerWriter` into `DeviationExpirationJob`**

Replace `DeviationExpirationJob.java` with:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.runtime.message.CommitmentService;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.util.List;

@ApplicationScoped
public class DeviationExpirationJob {

    @Inject CommitmentService commitmentService;
    @Inject Event<ProtocolDeviationResolvedEvent> resolvedEvent;
    @Inject DeviationLedgerWriter ledgerWriter;

    @Scheduled(every = "${casehub.clinical.deviation.expiration-check-interval:1h}",
               identity = "deviation-expiration")
    @Transactional
    public void checkExpiredCommitments() {
        List<ProtocolDeviation> overdue = ProtocolDeviation
            .find("piApprovalStatus = ?1 and responseDeadline < ?2",
                  PiApprovalStatus.COMMANDED, Instant.now())
            .list();

        for (ProtocolDeviation d : overdue) {
            try {
                d.piApprovalStatus = PiApprovalStatus.EXPIRED;
                commitmentService.fail(d.id.toString());
                ledgerWriter.writeResolutionEntry(d, PiApprovalStatus.EXPIRED,
                    "system", ActorType.SYSTEM, "deviation-expiration-job");
                resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
                    d.id, d.siteId, d.severity,
                    d.escalationRequirement != null ? d.escalationRequirement : EscalationRequirement.NONE,
                    PiApprovalStatus.EXPIRED
                ));
            } catch (Exception e) {
                d.piApprovalStatus = PiApprovalStatus.COMMANDED;
                org.jboss.logging.Logger.getLogger(DeviationExpirationJob.class)
                    .errorf(e, "Failed to expire deviation %s — will retry next run", d.id);
            }
        }
    }
}
```

- [ ] **Step 6.4: Run full test suite**

```bash
mvn test --batch-mode
```

Expected: all tests pass, including the 56 previously passing + new assertions.

- [ ] **Step 6.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java \
  runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "feat: DeviationExpirationJob writes EXPIRED ledger entry per expired deviation — Closes #14 Closes #15"
```
