# Sponsor Notification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement GCP §4.5 sponsor notification for MAJOR protocol deviations via the `SponsorNotifier` SPI backed by `DefaultSponsorNotifier` using `casehub-connectors`.

**Architecture:** A `SponsorNotifier` SPI in `api/` separates "decided to notify" (thin `SponsorNotificationListener`) from "how notification works" (`DefaultSponsorNotifier` — connector delivery + ledger write). `ProtocolDeviationResolvedEvent` is enriched with `deviationType` and `piId`. Per-trial sponsor config lives on `ClinicalTrial`. `DeviationLedgerWriter` gains `writeSponsorNotifiedEntry()` following the harness-ledger-writer.md protocol.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JPA (Panache), casehub-connectors-core 0.2-SNAPSHOT, Mockito, AssertJ, REST-assured

**Spec:** `docs/specs/2026-05-21-sponsor-notification-design.md`

---

## File Map

| Action | File | Purpose |
|--------|------|---------|
| Modify | `api/src/main/java/io/casehub/clinical/api/ProtocolDeviationResolvedEvent.java` | Add `deviationType`, `piId` fields |
| Create | `api/src/main/java/io/casehub/clinical/api/SponsorNotifier.java` | SPI interface |
| Create | `api/src/main/java/io/casehub/clinical/api/SponsorNotificationRequest.java` | Value object |
| Modify | `runtime/src/main/java/io/casehub/clinical/service/PiResponseListener.java` | Pass new event fields |
| Modify | `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java` | Pass new event fields |
| Modify | `runtime/src/main/java/io/casehub/clinical/entity/ClinicalTrial.java` | Add sponsor config fields |
| Create | `runtime/src/main/resources/db/migration/default/V108__sponsor_notification_config.sql` | clinical_trial columns |
| Modify | `runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java` | Accept sponsor config in POST |
| Modify | `runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java` | Add `sponsorNotifiedAt` |
| Create | `runtime/src/main/resources/db/migration/qhorus/V1008__sponsor_notified_at.sql` | ledger entry column |
| Modify | `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java` | Add `writeSponsorNotifiedEntry()` |
| Modify | `runtime/pom.xml` | Add casehub-connectors-core dependency |
| Create | `runtime/src/main/java/io/casehub/clinical/service/DefaultSponsorNotifier.java` | `@DefaultBean` SPI impl |
| Create | `runtime/src/main/java/io/casehub/clinical/service/SponsorNotificationListener.java` | CDI observer |
| Modify | `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java` | Add sponsor entry tests |
| Create | `runtime/src/test/java/io/casehub/clinical/service/DefaultSponsorNotifierTest.java` | Unit tests |
| Create | `runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationListenerTest.java` | Unit tests |
| Create | `runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationIntegrationTest.java` | @QuarkusTest full chain |
| Modify | `../parent/docs/PLATFORM.md` | Add cross-repo dependency row |

---

## Task 1: Enrich ProtocolDeviationResolvedEvent + SponsorNotifier SPI

**Files:**
- Modify: `api/src/main/java/io/casehub/clinical/api/ProtocolDeviationResolvedEvent.java`
- Create: `api/src/main/java/io/casehub/clinical/api/SponsorNotifier.java`
- Create: `api/src/main/java/io/casehub/clinical/api/SponsorNotificationRequest.java`

- [ ] **Step 1: Enrich ProtocolDeviationResolvedEvent**

Replace the record with:

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import java.util.UUID;

/**
 * Fired when a protocol deviation reaches a terminal PI authorisation state.
 * Consumers: casehubio/clinical#6 (IRB_REVIEW) and casehubio/clinical#13 (SPONSOR_NOTIFICATION).
 */
public record ProtocolDeviationResolvedEvent(
    UUID deviationId,
    UUID siteId,
    DeviationSeverity severity,
    EscalationRequirement escalationRequirement,
    PiApprovalStatus terminalStatus,
    String deviationType,   // always non-null — from ProtocolDeviation.deviationType
    String piId             // null for EXPIRED — senderId for PI responses
) {}
```

- [ ] **Step 2: Create SponsorNotifier SPI**

```java
package io.casehub.clinical.api;

public interface SponsorNotifier {
    void notify(SponsorNotificationRequest request);
}
```

- [ ] **Step 3: Create SponsorNotificationRequest**

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.PiApprovalStatus;
import java.util.UUID;

public record SponsorNotificationRequest(
    UUID deviationId,
    UUID siteId,
    UUID trialId,
    String deviationType,
    DeviationSeverity severity,
    PiApprovalStatus terminalStatus,
    String piId,                           // null for EXPIRED
    String sponsorNotificationConnectorId,
    String sponsorNotificationDestination
) {}
```

- [ ] **Step 4: Fix compilation — update fire sites**

`PiResponseListener.process()` — update the `resolvedEvent.fireAsync(...)` call:

```java
resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
    deviation.id,
    deviation.siteId,
    deviation.severity,
    deviation.escalationRequirement != null
        ? deviation.escalationRequirement : EscalationRequirement.NONE,
    deviation.piApprovalStatus,
    deviation.deviationType,     // new
    senderId                     // new — piId; senderId is the method param
));
```

`DeviationExpirer.expire()` (the method that fires the event) — update similarly:

```java
resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
    deviation.id,
    deviation.siteId,
    deviation.severity,
    deviation.escalationRequirement != null
        ? deviation.escalationRequirement : EscalationRequirement.NONE,
    PiApprovalStatus.EXPIRED,
    deviation.deviationType,     // new
    null                         // new — piId is null for EXPIRED
));
```

- [ ] **Step 5: Compile to confirm no errors**

```bash
mvn compile -pl api,runtime --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Run existing tests to confirm nothing broken**

```bash
mvn test -pl runtime --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: 77 tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  api/src/main/java/io/casehub/clinical/api/ProtocolDeviationResolvedEvent.java \
  api/src/main/java/io/casehub/clinical/api/SponsorNotifier.java \
  api/src/main/java/io/casehub/clinical/api/SponsorNotificationRequest.java \
  runtime/src/main/java/io/casehub/clinical/service/PiResponseListener.java \
  runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(api): enrich ProtocolDeviationResolvedEvent + SponsorNotifier SPI

Add deviationType and piId (nullable) to ProtocolDeviationResolvedEvent.
Introduce SponsorNotifier SPI and SponsorNotificationRequest value object.
Update PiResponseListener and DeviationExpirer fire sites.

Refs #13"
```

---

## Task 2: ClinicalTrial sponsor config + migration + TrialResource

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/ClinicalTrial.java`
- Create: `runtime/src/main/resources/db/migration/default/V108__sponsor_notification_config.sql`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java`
- Test: `runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java`

- [ ] **Step 1: Write failing test**

Add to `TrialResourceTest`:

```java
@Test
void register_with_sponsor_config_persists_connector_fields() {
    String body = """
        {
          "protocolId": "ONCO-2026-001",
          "phase": "PHASE_II",
          "sponsor": "Pfizer",
          "targetEnrollment": 50,
          "sponsorNotificationConnectorId": "slack",
          "sponsorNotificationDestination": "https://hooks.slack.com/T000/B000/xxx"
        }
        """;

    String location = given()
        .contentType(ContentType.JSON)
        .body(body)
        .when().post("/trials")
        .then().statusCode(201)
        .extract().header("Location");

    String id = location.substring(location.lastIndexOf('/') + 1);
    given()
        .when().get("/trials/" + id)
        .then().statusCode(200)
        .body("sponsorNotificationConnectorId", equalTo("slack"))
        .body("sponsorNotificationDestination", equalTo("https://hooks.slack.com/T000/B000/xxx"));
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl runtime -Dtest=TrialResourceTest#register_with_sponsor_config_persists_connector_fields --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: FAIL — fields not present in response (null)

- [ ] **Step 3: Add fields to ClinicalTrial entity**

```java
@Column(name = "sponsor_notification_connector_id")
public String sponsorNotificationConnectorId;

@Column(name = "sponsor_notification_destination")
public String sponsorNotificationDestination;
```

- [ ] **Step 4: Create migration**

`runtime/src/main/resources/db/migration/default/V108__sponsor_notification_config.sql`:
```sql
ALTER TABLE clinical_trial
    ADD COLUMN sponsor_notification_connector_id VARCHAR(64),
    ADD COLUMN sponsor_notification_destination  VARCHAR(512);
```

- [ ] **Step 5: Update TrialResource — add optional fields to request and mapping**

```java
public record RegisterTrialRequest(
    @NotBlank String protocolId,
    @NotNull TrialPhase phase,
    @NotBlank String sponsor,
    @Positive int targetEnrollment,
    String sponsorNotificationConnectorId,    // optional
    String sponsorNotificationDestination     // optional
) {}
```

In the `register()` method, add after `trial.status = TrialStatus.PLANNING;`:
```java
trial.sponsorNotificationConnectorId = req.sponsorNotificationConnectorId();
trial.sponsorNotificationDestination = req.sponsorNotificationDestination();
```

- [ ] **Step 6: Run test to verify it passes**

```bash
mvn test -pl runtime -Dtest=TrialResourceTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all TrialResourceTest tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/entity/ClinicalTrial.java \
  runtime/src/main/resources/db/migration/default/V108__sponsor_notification_config.sql \
  runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java \
  runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: ClinicalTrial sponsor notification config fields + V108 migration

Add sponsorNotificationConnectorId and sponsorNotificationDestination to
ClinicalTrial. TrialResource.RegisterTrialRequest accepts both as optional fields.
Migration V108 adds columns to clinical_trial table.

Refs #13"
```

---

## Task 3: ProtocolDeviationLedgerEntry.sponsorNotifiedAt + DeviationLedgerWriter.writeSponsorNotifiedEntry

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V1008__sponsor_notified_at.sql`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java`

- [ ] **Step 1: Write failing tests for writeSponsorNotifiedEntry**

Add to `DeviationLedgerWriterTest`:

```java
@Test
void writeSponsorNotifiedEntry_sets_sponsor_notifier_role_and_notified_at_when_delivered() {
    when(ledgerEntryRepository.findLatestBySubjectId(dev.id))
        .thenReturn(Optional.of(existingEntry(2)));
    ArgumentCaptor<ProtocolDeviationLedgerEntry> cap =
        ArgumentCaptor.forClass(ProtocolDeviationLedgerEntry.class);

    writer.writeSponsorNotifiedEntry(dev, FIXED_INSTANT, true);

    verify(ledgerEntryRepository).save(cap.capture());
    ProtocolDeviationLedgerEntry entry = cap.getValue();
    assertThat(entry.actorRole).isEqualTo("sponsor-notifier");
    assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
    assertThat(entry.entryType).isEqualTo(LedgerEntryType.EVENT);
    assertThat(entry.occurredAt).isEqualTo(FIXED_INSTANT);
    assertThat(entry.sponsorNotifiedAt).isEqualTo(FIXED_INSTANT);
    assertThat(entry.sequenceNumber).isEqualTo(3);
}

@Test
void writeSponsorNotifiedEntry_sets_failed_role_and_null_notified_at_when_not_delivered() {
    when(ledgerEntryRepository.findLatestBySubjectId(dev.id))
        .thenReturn(Optional.empty());
    ArgumentCaptor<ProtocolDeviationLedgerEntry> cap =
        ArgumentCaptor.forClass(ProtocolDeviationLedgerEntry.class);

    writer.writeSponsorNotifiedEntry(dev, FIXED_INSTANT, false);

    verify(ledgerEntryRepository).save(cap.capture());
    ProtocolDeviationLedgerEntry entry = cap.getValue();
    assertThat(entry.actorRole).isEqualTo("sponsor-notifier-failed");
    assertThat(entry.sponsorNotifiedAt).isNull();
    assertThat(entry.sequenceNumber).isEqualTo(1);
}
```

Add helper to test class:
```java
private LedgerEntry existingEntry(int seq) {
    ProtocolDeviationLedgerEntry e = new ProtocolDeviationLedgerEntry();
    e.sequenceNumber = seq;
    return e;
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=DeviationLedgerWriterTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: FAIL — `writeSponsorNotifiedEntry` method does not exist

- [ ] **Step 3: Add sponsorNotifiedAt to ProtocolDeviationLedgerEntry**

```java
@Column(name = "sponsor_notified_at")
public Instant sponsorNotifiedAt;
```

- [ ] **Step 4: Create migration**

`runtime/src/main/resources/db/migration/qhorus/V1008__sponsor_notified_at.sql`:
```sql
ALTER TABLE protocol_deviation_ledger_entry
    ADD COLUMN sponsor_notified_at TIMESTAMP WITH TIME ZONE;
```

- [ ] **Step 5: Add writeSponsorNotifiedEntry to DeviationLedgerWriter**

```java
public void writeSponsorNotifiedEntry(ProtocolDeviation dev, Instant notifiedAt, boolean delivered) {
    ProtocolDeviationLedgerEntry entry = baseEntry(dev);
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = SYSTEM_ACTOR;
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = delivered ? "sponsor-notifier" : "sponsor-notifier-failed";
    entry.occurredAt = notifiedAt;
    entry.sponsorNotifiedAt = delivered ? notifiedAt : null;
    ledgerEntryRepository.save(entry);
}
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=DeviationLedgerWriterTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all DeviationLedgerWriterTest tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java \
  runtime/src/main/resources/db/migration/qhorus/V1008__sponsor_notified_at.sql \
  runtime/src/main/java/io/casehub/clinical/service/DeviationLedgerWriter.java \
  runtime/src/test/java/io/casehub/clinical/service/DeviationLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: ProtocolDeviationLedgerEntry.sponsorNotifiedAt + DeviationLedgerWriter.writeSponsorNotifiedEntry

V1008 migration adds sponsor_notified_at column to ledger entry table.
DeviationLedgerWriter.writeSponsorNotifiedEntry() writes EVENT entry with
actorRole 'sponsor-notifier' (delivered) or 'sponsor-notifier-failed' (not delivered).

Refs #13"
```

---

## Task 4: casehub-connectors-core dependency + DefaultSponsorNotifier

**Files:**
- Modify: `runtime/pom.xml`
- Create: `runtime/src/main/java/io/casehub/clinical/service/DefaultSponsorNotifier.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/DefaultSponsorNotifierTest.java`

- [ ] **Step 1: Add casehub-connectors-core dependency to runtime/pom.xml**

Add inside `<dependencies>`:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-connectors-core</artifactId>
</dependency>
```

- [ ] **Step 2: Create test-scope stub connector**

`TestSlackConnector` must exist before the tests that reference it (required for compilation).

Create `runtime/src/test/java/io/casehub/clinical/service/TestSlackConnector.java`:

```java
package io.casehub.clinical.service;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.quarkus.test.Mock;
import java.util.ArrayList;
import java.util.List;

@Mock
public class TestSlackConnector implements Connector {

    public static final List<ConnectorMessage> sent = new ArrayList<>();
    public static boolean shouldThrow = false;

    @Override
    public String id() { return "slack"; }

    @Override
    public void send(ConnectorMessage message) {
        if (shouldThrow) throw new RuntimeException("Connector failure");
        sent.add(message);
    }
}
```

- [ ] **Step 3: Write failing tests for DefaultSponsorNotifier**

Create `DefaultSponsorNotifierTest.java`. Note: `recordAttempt()` calls `ProtocolDeviation.findById()`, so setUp must persist a `ProtocolDeviation` for each test's deviationId. We use a fixed `deviationId` per test.

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.SponsorNotificationRequest;
import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;

@QuarkusTest
class DefaultSponsorNotifierTest {

    @Inject DefaultSponsorNotifier notifier;
    @InjectMock DeviationLedgerWriter ledgerWriter;

    private UUID deviationId;
    private UUID siteId;

    @BeforeEach
    @Transactional
    void setUp() {
        TestSlackConnector.sent.clear();
        TestSlackConnector.shouldThrow = false;

        deviationId = UUID.randomUUID();
        siteId = UUID.randomUUID();

        ProtocolDeviation dev = new ProtocolDeviation();
        dev.id = deviationId;
        dev.siteId = siteId;
        dev.deviationType = "CONSENT_DEVIATION";
        dev.severity = DeviationSeverity.MAJOR;
        dev.escalationRequirement = EscalationRequirement.SPONSOR_NOTIFICATION;
        dev.piApprovalStatus = PiApprovalStatus.COMMANDED;
        dev.commandedAt = Instant.now();
        dev.responseDeadline = Instant.now().plusSeconds(3600);
        dev.persist();
    }

    @Test
    void escalated_notification_sends_to_connector_and_writes_delivered_ledger_entry() {
        SponsorNotificationRequest req = request(PiApprovalStatus.ESCALATED, "dr-smith@v1", "slack");

        notifier.notify(req);

        assertThat(TestSlackConnector.sent).hasSize(1);
        assertThat(TestSlackConnector.sent.get(0).body())
            .contains("CONSENT_DEVIATION")
            .contains("dr-smith@v1")
            .contains("corrective action committed");
        verify(ledgerWriter).writeSponsorNotifiedEntry(
            any(ProtocolDeviation.class), any(Instant.class), eq(true));
    }

    @Test
    void unknown_connector_id_writes_failed_ledger_entry_without_sending() {
        SponsorNotificationRequest req = request(PiApprovalStatus.ESCALATED, "dr-smith@v1", "unknown-connector");

        notifier.notify(req);

        assertThat(TestSlackConnector.sent).isEmpty();
        verify(ledgerWriter).writeSponsorNotifiedEntry(
            any(ProtocolDeviation.class), any(Instant.class), eq(false));
    }

    @Test
    void expired_notification_body_omits_pi_reference() {
        SponsorNotificationRequest req = new SponsorNotificationRequest(
            deviationId, siteId, UUID.randomUUID(),
            "CONSENT_DEVIATION", DeviationSeverity.MAJOR,
            PiApprovalStatus.EXPIRED, null,   // piId null for EXPIRED
            "slack", "https://hooks.slack.com/test"
        );

        notifier.notify(req);

        assertThat(TestSlackConnector.sent).hasSize(1);
        assertThat(TestSlackConnector.sent.get(0).body())
            .contains("deadline expired")
            .doesNotContain("null");
        verify(ledgerWriter).writeSponsorNotifiedEntry(
            any(ProtocolDeviation.class), any(Instant.class), eq(true));
    }

    @Test
    void connector_send_exception_writes_failed_ledger_entry_without_rethrowing() {
        TestSlackConnector.shouldThrow = true;
        SponsorNotificationRequest req = request(PiApprovalStatus.ESCALATED, "dr-smith@v1", "slack");

        notifier.notify(req);  // must not throw

        verify(ledgerWriter).writeSponsorNotifiedEntry(
            any(ProtocolDeviation.class), any(Instant.class), eq(false));
    }

    private SponsorNotificationRequest request(PiApprovalStatus status, String piId, String connectorId) {
        return new SponsorNotificationRequest(
            deviationId, siteId, UUID.randomUUID(),
            "CONSENT_DEVIATION", DeviationSeverity.MAJOR,
            status, piId, connectorId, "https://hooks.slack.com/test"
        );
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=DefaultSponsorNotifierTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: FAIL — `DefaultSponsorNotifier` class does not exist

- [ ] **Step 6: Implement DefaultSponsorNotifier**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.SponsorNotificationRequest;
import io.casehub.clinical.api.SponsorNotifier;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.quarkus.arc.DefaultBean;
import io.quarkus.logging.Log;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.time.Clock;
import java.time.Instant;

@ApplicationScoped
@DefaultBean
public class DefaultSponsorNotifier implements SponsorNotifier {

    @Inject @Any Instance<Connector> connectors;
    @Inject DeviationLedgerWriter ledgerWriter;
    @Inject Clock clock;

    @Override
    public void notify(SponsorNotificationRequest req) {
        Connector connector = connectors.stream()
            .filter(c -> c.id().equals(req.sponsorNotificationConnectorId()))
            .findFirst()
            .orElse(null);

        if (connector == null) {
            Log.errorf("No connector '%s' found — sponsor notification not delivered for deviation %s",
                req.sponsorNotificationConnectorId(), req.deviationId());
            recordAttempt(req, false);
            return;
        }

        try {
            connector.send(new ConnectorMessage(
                req.sponsorNotificationDestination(),
                buildTitle(req),
                buildBody(req)
            ));
            recordAttempt(req, true);
        } catch (Exception e) {
            Log.errorf(e, "Sponsor notification delivery failed for deviation %s", req.deviationId());
            recordAttempt(req, false);
        }
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    void recordAttempt(SponsorNotificationRequest req, boolean delivered) {
        ProtocolDeviation dev = ProtocolDeviation.findById(req.deviationId());
        if (dev == null) {
            Log.errorf("ProtocolDeviation %s not found — cannot write ledger entry", req.deviationId());
            return;
        }
        ledgerWriter.writeSponsorNotifiedEntry(dev, clock.instant(), delivered);
    }

    private String buildTitle(SponsorNotificationRequest req) {
        return "[MAJOR Deviation] " + req.deviationType() + " — " + req.terminalStatus().name();
    }

    private String buildBody(SponsorNotificationRequest req) {
        return switch (req.terminalStatus()) {
            case ESCALATED -> "PI " + req.piId() + " approved — corrective action committed. " +
                "Site: " + req.siteId() + ". Type: " + req.deviationType() + ". " +
                "Ref: clinical/deviation/" + req.deviationId() + "/pi-oversight";
            case REJECTED -> "PI " + req.piId() + " refused to authorise — no corrective action. " +
                "Site: " + req.siteId() + ". Type: " + req.deviationType() + ".";
            case EXPIRED -> "PI response deadline expired — no response received. " +
                "Site: " + req.siteId() + ". Type: " + req.deviationType() + ".";
            default -> "Deviation " + req.deviationId() + " resolved as " + req.terminalStatus().name();
        };
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=DefaultSponsorNotifierTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all 4 tests pass

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/pom.xml \
  runtime/src/main/java/io/casehub/clinical/service/DefaultSponsorNotifier.java \
  runtime/src/test/java/io/casehub/clinical/service/DefaultSponsorNotifierTest.java \
  runtime/src/test/java/io/casehub/clinical/service/TestSlackConnector.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: DefaultSponsorNotifier — connector delivery + ledger write

@DefaultBean SponsorNotifier implementation. Resolves Connector by id from
CDI Instance<Connector>. Delivery exception caught and ledgered as failed.
recordAttempt() runs in REQUIRES_NEW transaction to isolate ledger write.

Refs #13"
```

---

## Task 5: SponsorNotificationListener

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/SponsorNotificationListener.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationListenerTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.SponsorNotificationRequest;
import io.casehub.clinical.api.SponsorNotifier;
import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.TrialSite;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

@QuarkusTest
class SponsorNotificationListenerTest {

    @Inject SponsorNotificationListener listener;
    @InjectMock SponsorNotifier sponsorNotifier;

    private UUID trialId;
    private UUID siteId;

    @BeforeEach
    @Transactional
    void setUp() {
        trialId = UUID.randomUUID();
        siteId = UUID.randomUUID();

        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId;
        trial.protocolId = "TEST-001";
        trial.phase = io.casehub.clinical.api.model.TrialPhase.PHASE_II;
        trial.sponsor = "Test Sponsor";
        trial.targetEnrollment = 10;
        trial.sponsorNotificationConnectorId = "slack";
        trial.sponsorNotificationDestination = "https://hooks.slack.com/test";
        trial.persist();

        TrialSite site = new TrialSite();
        site.id = siteId;
        site.trialId = trialId;
        site.investigatorId = "dr-smith@v1";
        site.persist();
    }

    @Test
    void sponsor_notification_event_calls_spi_with_correct_request() {
        ProtocolDeviationResolvedEvent event = new ProtocolDeviationResolvedEvent(
            UUID.randomUUID(), siteId, DeviationSeverity.MAJOR,
            EscalationRequirement.SPONSOR_NOTIFICATION, PiApprovalStatus.ESCALATED,
            "CONSENT_DEVIATION", "dr-smith@v1"
        );

        listener.onDeviationResolved(event);

        ArgumentCaptor<SponsorNotificationRequest> cap =
            ArgumentCaptor.forClass(SponsorNotificationRequest.class);
        verify(sponsorNotifier).notify(cap.capture());
        SponsorNotificationRequest req = cap.getValue();
        assertThat(req.trialId()).isEqualTo(trialId);
        assertThat(req.deviationType()).isEqualTo("CONSENT_DEVIATION");
        assertThat(req.piId()).isEqualTo("dr-smith@v1");
        assertThat(req.terminalStatus()).isEqualTo(PiApprovalStatus.ESCALATED);
        assertThat(req.sponsorNotificationConnectorId()).isEqualTo("slack");
        assertThat(req.sponsorNotificationDestination()).isEqualTo("https://hooks.slack.com/test");
    }

    @Test
    void irb_review_escalation_does_not_call_spi() {
        ProtocolDeviationResolvedEvent event = new ProtocolDeviationResolvedEvent(
            UUID.randomUUID(), siteId, DeviationSeverity.CRITICAL,
            EscalationRequirement.IRB_REVIEW, PiApprovalStatus.ESCALATED,
            "PROTOCOL_PROCEDURE", "dr-smith@v1"
        );

        listener.onDeviationResolved(event);

        verifyNoInteractions(sponsorNotifier);
    }

    @Test
    void none_escalation_does_not_call_spi() {
        ProtocolDeviationResolvedEvent event = new ProtocolDeviationResolvedEvent(
            UUID.randomUUID(), siteId, DeviationSeverity.MINOR,
            EscalationRequirement.NONE, PiApprovalStatus.APPROVED,
            "MINOR_DEVIATION", "dr-smith@v1"
        );

        listener.onDeviationResolved(event);

        verifyNoInteractions(sponsorNotifier);
    }

    @Test
    void missing_connector_config_does_not_call_spi() {
        // Trial with no connector config
        UUID noConfigTrialId = UUID.randomUUID();
        UUID noConfigSiteId = UUID.randomUUID();
        persistTrialAndSite(noConfigTrialId, noConfigSiteId, null, null);

        ProtocolDeviationResolvedEvent event = new ProtocolDeviationResolvedEvent(
            UUID.randomUUID(), noConfigSiteId, DeviationSeverity.MAJOR,
            EscalationRequirement.SPONSOR_NOTIFICATION, PiApprovalStatus.ESCALATED,
            "CONSENT_DEVIATION", "dr-smith@v1"
        );

        listener.onDeviationResolved(event);

        verifyNoInteractions(sponsorNotifier);
    }

    @Transactional
    void persistTrialAndSite(UUID tid, UUID sid, String connectorId, String destination) {
        ClinicalTrial t = new ClinicalTrial();
        t.id = tid; t.protocolId = "X"; t.phase = io.casehub.clinical.api.model.TrialPhase.PHASE_I;
        t.sponsor = "S"; t.targetEnrollment = 1;
        t.sponsorNotificationConnectorId = connectorId;
        t.sponsorNotificationDestination = destination;
        t.persist();
        TrialSite s = new TrialSite();
        s.id = sid; s.trialId = tid; s.investigatorId = "x";
        s.persist();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=SponsorNotificationListenerTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: FAIL — `SponsorNotificationListener` does not exist

- [ ] **Step 3: Implement SponsorNotificationListener**

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

@ApplicationScoped
public class SponsorNotificationListener {

    @Inject SponsorNotifier sponsorNotifier;

    @Transactional
    public void onDeviationResolved(@ObservesAsync ProtocolDeviationResolvedEvent event) {
        if (event.escalationRequirement() != EscalationRequirement.SPONSOR_NOTIFICATION) return;

        TrialSite site = TrialSite.findById(event.siteId());
        if (site == null) {
            Log.warnf("TrialSite %s not found — sponsor notification skipped", event.siteId());
            return;
        }

        ClinicalTrial trial = ClinicalTrial.findById(site.trialId);
        if (trial == null
                || trial.sponsorNotificationConnectorId == null
                || trial.sponsorNotificationDestination == null) {
            Log.warnf("Trial %s has no sponsor notification config — skipping", site.trialId);
            return;
        }

        sponsorNotifier.notify(new SponsorNotificationRequest(
            event.deviationId(),
            event.siteId(),
            site.trialId,
            event.deviationType(),
            event.severity(),
            event.terminalStatus(),
            event.piId(),
            trial.sponsorNotificationConnectorId,
            trial.sponsorNotificationDestination
        ));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=SponsorNotificationListenerTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all 4 tests pass

- [ ] **Step 5: Run full suite**

```bash
mvn test -pl runtime --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all tests pass (count increases by new tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/SponsorNotificationListener.java \
  runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationListenerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: SponsorNotificationListener — routes SPONSOR_NOTIFICATION events to SponsorNotifier SPI

Thin observer: filters escalation requirement, resolves TrialSite -> ClinicalTrial
for sponsor config, builds SponsorNotificationRequest, delegates to SponsorNotifier.
Missing config is a deployment gap — logs warning, skips without error.

Refs #13"
```

---

## Task 6: Integration test — full chain with stub SponsorNotifier

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationIntegrationTest.java`

This test wires `ProtocolDeviationService → ProtocolDeviation → PiResponseListener → SponsorNotificationListener → SponsorNotifier` through real CDI events. It uses `TestSlackConnector` (already registered) to capture deliveries and `@InjectMock DeviationLedgerWriter` to suppress ledger writes.

- [ ] **Step 1: Write integration test**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.time.Duration;
import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.doNothing;

@QuarkusTest
class SponsorNotificationIntegrationTest {

    @Inject PiResponseListener piResponseListener;
    @InjectMock DeviationLedgerWriter ledgerWriter;

    private UUID deviationId;
    private String channelName;

    @BeforeEach
    @Transactional
    void setUp() {
        TestSlackConnector.sent.clear();
        TestSlackConnector.shouldThrow = false;
        doNothing().when(ledgerWriter).writeResolutionEntry(any(), any(), any(), any(), any());
        doNothing().when(ledgerWriter).writeSponsorNotifiedEntry(any(), any(Instant.class), any(Boolean.class));

        UUID trialId = UUID.randomUUID();
        UUID siteId = UUID.randomUUID();
        deviationId = UUID.randomUUID();

        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId;
        trial.protocolId = "ONCO-001"; trial.phase = TrialPhase.PHASE_III;
        trial.sponsor = "Roche"; trial.targetEnrollment = 100;
        trial.sponsorNotificationConnectorId = "slack";
        trial.sponsorNotificationDestination = "https://hooks.slack.com/integration-test";
        trial.persist();

        TrialSite site = new TrialSite();
        site.id = siteId; site.trialId = trialId; site.investigatorId = "dr-jones@v1";
        site.persist();

        ProtocolDeviation dev = new ProtocolDeviation();
        dev.id = deviationId; dev.siteId = siteId;
        dev.deviationType = "INFORMED_CONSENT";
        dev.severity = DeviationSeverity.MAJOR;
        dev.escalationRequirement = EscalationRequirement.SPONSOR_NOTIFICATION;
        dev.piApprovalStatus = PiApprovalStatus.COMMANDED;
        dev.piCommandChannelName = "clinical/deviation/" + deviationId + "/pi-oversight";
        dev.commandedAt = Instant.now();
        dev.responseDeadline = Instant.now().plusSeconds(3600);
        dev.persist();

        channelName = "clinical/deviation/" + deviationId + "/pi-oversight";
    }

    @Test
    void major_deviation_pi_approval_triggers_slack_notification() {
        piResponseListener.process(channelName, MessageType.DONE, "dr-jones@v1");

        Awaitility.await().atMost(Duration.ofSeconds(5)).until(() ->
            !TestSlackConnector.sent.isEmpty()
        );

        assertThat(TestSlackConnector.sent).hasSize(1);
        assertThat(TestSlackConnector.sent.get(0).body()).contains("INFORMED_CONSENT");
        assertThat(TestSlackConnector.sent.get(0).body()).contains("dr-jones@v1");
        assertThat(TestSlackConnector.sent.get(0).destination())
            .isEqualTo("https://hooks.slack.com/integration-test");
    }

    @Test
    void major_deviation_pi_rejection_also_triggers_sponsor_notification() {
        piResponseListener.process(channelName, MessageType.DECLINE, "dr-jones@v1");

        Awaitility.await().atMost(Duration.ofSeconds(5)).until(() ->
            !TestSlackConnector.sent.isEmpty()
        );

        assertThat(TestSlackConnector.sent).hasSize(1);
        assertThat(TestSlackConnector.sent.get(0).body()).contains("refused to authorise");
    }

    @Test
    void connector_delivery_failure_does_not_propagate_exception() {
        TestSlackConnector.shouldThrow = true;

        // Should complete without exception even when connector throws
        piResponseListener.process(channelName, MessageType.DONE, "dr-jones@v1");

        // Give async events time to process
        Awaitility.await().atMost(Duration.ofSeconds(5)).untilAsserted(() ->
            Mockito.verify(ledgerWriter, Mockito.atLeastOnce())
                .writeSponsorNotifiedEntry(any(), any(Instant.class), any(Boolean.class))
        );
    }
}
```

- [ ] **Step 2: Run integration test**

```bash
mvn test -pl runtime -Dtest=SponsorNotificationIntegrationTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all 3 tests pass

- [ ] **Step 3: Run full suite**

```bash
mvn test -pl runtime --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all tests pass

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/SponsorNotificationIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test: SponsorNotificationIntegrationTest — full chain PI approval -> Slack delivery

Verifies end-to-end: PiResponseListener -> CDI event -> SponsorNotificationListener
-> DefaultSponsorNotifier -> TestSlackConnector. Covers DONE/DECLINE/connector-failure paths.

Refs #13"
```

---

## Task 7: PLATFORM.md cross-repo dependency update

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`

- [ ] **Step 1: Add casehub-connectors-core dependency row**

In the `## Cross-Repo Dependency Map` table, add after the existing `casehub-connectors-core` rows:

```markdown
| `casehub-connectors-core` | `casehub-clinical` | `runtime` | sponsor notification delivery |
```

- [ ] **Step 2: Commit to parent repo**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: add casehub-connectors-core -> casehub-clinical dependency row

clinical#13 (sponsor notification) adds casehub-connectors-core as a
runtime dependency for DefaultSponsorNotifier connector delivery.

Refs casehubio/clinical#13"
```

---

## Final Verification

- [ ] **Run complete test suite**

```bash
mvn test --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all tests pass, 0 failures

- [ ] **Verify test count increased from 77 baseline**

New tests: DeviationLedgerWriterTest (+2), DefaultSponsorNotifierTest (+3), SponsorNotificationListenerTest (+4), SponsorNotificationIntegrationTest (+3) = +12 minimum
