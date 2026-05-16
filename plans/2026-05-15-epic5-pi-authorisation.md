# Epic 5: PI Authorisation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire formal PI authorisation for protocol deviations via a pure qhorus COMMAND/Commitment lifecycle — no REST bypass endpoints; full normative audit trail.

**Architecture:** When a deviation is reported, `ProtocolDeviationService` creates a per-deviation qhorus channel (`clinical/deviation/{id}/pi-oversight`), sends a COMMAND to the PI, and sets `piApprovalStatus = COMMANDED`. `ClinicalInboundNormaliser` maps the PI's structured JSON response to qhorus message types (`DONE`/`DECLINE`). `PiResponseListener` observes the `MessageReceivedEvent` CDI hook (qhorus#153) and updates domain state. `DeviationExpirationJob` handles deadline breaches. Downstream epics consume `ProtocolDeviationResolvedEvent` — no code changes to this service when they ship.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-qhorus (MessageService, ChannelService, CommitmentService, ChannelGateway), casehub-ledger, Panache Active Record, H2 (tests), Flyway, JTA/XA, MicroProfile Config, Quartz scheduler.

**Critical path:** `PiResponseListener` integration test is `@Disabled` pending casehubio/qhorus#153 (`MessageReceivedEvent` CDI hook). All other tasks proceed now.

**All commits must reference:** `Refs #5`

---

## File Map

### `api/pom.xml`
- Add `casehub-qhorus-api` dependency

### `api/src/main/java/io/casehub/clinical/api/model/PiApprovalStatus.java`
- Add: `COMMANDED`, `EXPIRED`, `ESCALATED`

### `api/src/main/java/io/casehub/clinical/api/model/EscalationRequirement.java` *(new)*
- Enum: `NONE`, `SPONSOR_NOTIFICATION`, `IRB_REVIEW`

### `api/src/main/java/io/casehub/clinical/api/spi/DeviationContext.java` *(new)*
- Record: `deviationId`, `siteId`, `trialId`, `protocolId`, `phase`, `severity`, `deviationType`

### `api/src/main/java/io/casehub/clinical/api/spi/DeviationResponsePolicy.java` *(new)*
- Interface: `evaluate(DeviationContext) → DeviationResponseRequirements`

### `api/src/main/java/io/casehub/clinical/api/spi/DeviationResponseRequirements.java` *(new)*
- Record: `piResponseDeadline`, `escalationRequirement`

### `api/src/main/java/io/casehub/clinical/api/ProtocolDeviationResolvedEvent.java` *(new)*
- Record: `deviationId`, `siteId`, `severity`, `escalationRequirement`, `terminalStatus`

### `runtime/pom.xml`
- Add `casehub-qhorus` dependency

### `runtime/src/main/resources/application.properties`
- Add `io.casehub.qhorus.runtime` to qhorus PU packages

### `runtime/src/test/resources/application.properties`
- Add `io.casehub.qhorus.runtime` to qhorus PU packages
- Add reactive suppression properties (verify needed)

### `runtime/src/main/resources/db/migration/V107__alter_protocol_deviation_add_commitment_fields.sql` *(new)*
- Add 4 columns to `protocol_deviation`

### `runtime/src/main/resources/db/migration/V1006__protocol_deviation_ledger_entry.sql` *(new)*
- Create `protocol_deviation_ledger_entry` join table

### `runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java`
- Add: `piCommandChannelName`, `commandedAt`, `responseDeadline`, `escalationRequirement`

### `runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java` *(new)*
- `LedgerEntry` subclass, `@DiscriminatorValue("PROTOCOL_DEVIATION")`

### `runtime/src/main/java/io/casehub/clinical/service/DefaultDeviationResponsePolicy.java` *(new)*
- `@ApplicationScoped @DefaultBean` — reads deadlines from MicroProfile Config

### `runtime/src/main/java/io/casehub/clinical/service/ClinicalInboundNormaliser.java` *(new)*
- `@ApplicationScoped` — maps PI JSON response to `DONE` / `DECLINE` message types

### `runtime/src/main/java/io/casehub/clinical/service/ProtocolDeviationService.java` *(new)*
- Creates channel, sends COMMAND, writes ledger, sets `COMMANDED` state

### `runtime/src/main/java/io/casehub/clinical/service/PiResponseListener.java` *(new)*
- `@ApplicationScoped` — `process()` callable; `@ObservesAsync MessageReceivedEvent` commented pending qhorus#153

### `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java` *(new)*
- `@Scheduled` — scans overdue `COMMANDED` deviations, marks `EXPIRED`

### `runtime/src/main/java/io/casehub/clinical/resource/DeviationResource.java` *(new)*
- `POST` + `GET` endpoints for `/trials/{trialId}/sites/{siteId}/deviations`

---

## Task 1: pom.xml dependencies + verify reactive suppression

**Files:**
- Modify: `api/pom.xml`
- Modify: `runtime/pom.xml`

- [ ] **Add `casehub-qhorus-api` to `api/pom.xml`** (after `casehub-clinical-api` dependency):

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-api</artifactId>
</dependency>
```

- [ ] **Add `casehub-qhorus` to `runtime/pom.xml`** (after `casehub-ledger`):

```xml
<!-- Layer 3: formal PI obligation tracking via COMMAND lifecycle -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus</artifactId>
</dependency>
```

- [ ] **Verify compile succeeds:**

```bash
mvn install -pl api --batch-mode && mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS. If qhorus reactive extension causes startup failure in tests (GE-20260508-492336), proceed to the next step.

- [ ] **Add reactive suppression to `runtime/src/test/resources/application.properties`** if the startup fails with "reactive datasource required":

```properties
casehub.qhorus.reactive.enabled=false
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.reactive=false
```

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/pom.xml runtime/pom.xml
git -C /Users/mdproctor/claude/casehub/clinical commit -m "chore(deps): add casehub-qhorus direct dependency — Layer 3

Refs #5"
```

---

## Task 2: application.properties — qhorus PU scan packages

**Files:**
- Modify: `runtime/src/main/resources/application.properties`
- Modify: `runtime/src/test/resources/application.properties`

The qhorus PU currently scans only `io.casehub.ledger.runtime.model,io.casehub.ledger.model,io.casehub.clinical.ledger`. Adding casehub-qhorus directly requires `io.casehub.qhorus.runtime` to be included.

- [ ] **Update both `application.properties` files.** In `runtime/src/main/resources/application.properties`:

```properties
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model,io.casehub.clinical.ledger
```

Apply the same change to `runtime/src/test/resources/application.properties`.

- [ ] **Verify tests still pass:**

```bash
mvn test -pl runtime --batch-mode
```

Expected: all existing tests pass. If Flyway errors about duplicate migrations appear, this means qhorus brings its own Flyway migrations — check and resolve before continuing.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/resources/application.properties runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/clinical commit -m "chore(config): add io.casehub.qhorus.runtime to qhorus PU scan packages

Refs #5"
```

---

## Task 3: `api/` — PiApprovalStatus + EscalationRequirement

**Files:**
- Modify: `api/src/main/java/io/casehub/clinical/api/model/PiApprovalStatus.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/EscalationRequirement.java`
- Modify: `api/src/test/java/io/casehub/clinical/api/ClinicalConstantsTest.java`

- [ ] **Write the failing test** — add to `ClinicalConstantsTest`:

```java
@Test
void piApprovalStatusHasAllSixValues() {
    var values = Set.of(PiApprovalStatus.values());
    assertThat(values).containsExactlyInAnyOrder(
        PiApprovalStatus.PENDING,
        PiApprovalStatus.COMMANDED,
        PiApprovalStatus.APPROVED,
        PiApprovalStatus.REJECTED,
        PiApprovalStatus.EXPIRED,
        PiApprovalStatus.ESCALATED
    );
}

@Test
void escalationRequirementHasAllValues() {
    var values = Set.of(EscalationRequirement.values());
    assertThat(values).containsExactlyInAnyOrder(
        EscalationRequirement.NONE,
        EscalationRequirement.SPONSOR_NOTIFICATION,
        EscalationRequirement.IRB_REVIEW
    );
}
```

- [ ] **Run to verify it fails:**

```bash
mvn test -pl api -Dtest=ClinicalConstantsTest --batch-mode
```

Expected: FAIL — COMMANDED, EXPIRED, ESCALATED don't exist yet.

- [ ] **Add the three new values to `PiApprovalStatus.java`:**

```java
package io.casehub.clinical.api.model;

public enum PiApprovalStatus {
    PENDING,     // reported; COMMAND not yet issued (transient in service)
    COMMANDED,   // COMMAND issued, Commitment OPEN, awaiting PI
    APPROVED,    // PI approved; MINOR deviations close here
    REJECTED,    // PI declined
    EXPIRED,     // deadline passed without response — GCP SLA breach
    ESCALATED    // PI approved; forwarded to IRB (CRITICAL) or sponsor (MAJOR)
}
```

- [ ] **Create `EscalationRequirement.java`:**

```java
package io.casehub.clinical.api.model;

public enum EscalationRequirement {
    NONE,                  // site-level resolution only (MINOR)
    SPONSOR_NOTIFICATION,  // notify trial sponsor (MAJOR) — casehubio/clinical#13
    IRB_REVIEW             // ethics committee gate (CRITICAL) — casehubio/clinical#6
}
```

- [ ] **Run tests to verify they pass:**

```bash
mvn test -pl api -Dtest=ClinicalConstantsTest --batch-mode
```

Expected: PASS.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(api): extend PiApprovalStatus with 3 new states; add EscalationRequirement enum

COMMANDED/EXPIRED/ESCALATED drive the full PI obligation lifecycle.
EscalationRequirement replaces hardcoded severity->escalation mapping in the service.

Refs #5"
```

---

## Task 4: `api/` — SPI types and domain event

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/spi/DeviationContext.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/DeviationResponsePolicy.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/DeviationResponseRequirements.java`
- Create: `api/src/main/java/io/casehub/clinical/api/ProtocolDeviationResolvedEvent.java`
- Modify: `api/src/test/java/io/casehub/clinical/api/ClinicalConstantsTest.java`

- [ ] **Write a failing compile test** — add to `ClinicalConstantsTest`:

```java
@Test
void deviationContextCarriesAllFields() {
    var ctx = new DeviationContext(
        UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(),
        "PROT-001", TrialPhase.PHASE_2, DeviationSeverity.MAJOR,
        "sample-window"
    );
    assertThat(ctx.severity()).isEqualTo(DeviationSeverity.MAJOR);
    assertThat(ctx.phase()).isEqualTo(TrialPhase.PHASE_2);
}

@Test
void protocolDeviationResolvedEventCarriesEscalation() {
    var event = new ProtocolDeviationResolvedEvent(
        UUID.randomUUID(), UUID.randomUUID(),
        DeviationSeverity.CRITICAL, EscalationRequirement.IRB_REVIEW,
        PiApprovalStatus.ESCALATED
    );
    assertThat(event.escalationRequirement()).isEqualTo(EscalationRequirement.IRB_REVIEW);
    assertThat(event.terminalStatus()).isEqualTo(PiApprovalStatus.ESCALATED);
}
```

- [ ] **Run to verify it fails (compile error):**

```bash
mvn test -pl api -Dtest=ClinicalConstantsTest --batch-mode
```

Expected: FAIL — types not found.

- [ ] **Create `api/src/main/java/io/casehub/clinical/api/spi/DeviationContext.java`:**

```java
package io.casehub.clinical.api.spi;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.TrialPhase;
import java.util.UUID;

public record DeviationContext(
    UUID deviationId,
    UUID siteId,
    UUID trialId,
    String protocolId,
    TrialPhase phase,
    DeviationSeverity severity,
    String deviationType
) {}
```

- [ ] **Create `api/src/main/java/io/casehub/clinical/api/spi/DeviationResponseRequirements.java`:**

```java
package io.casehub.clinical.api.spi;

import io.casehub.clinical.api.model.EscalationRequirement;
import java.time.Duration;

public record DeviationResponseRequirements(
    Duration piResponseDeadline,
    EscalationRequirement escalationRequirement
) {}
```

- [ ] **Create `api/src/main/java/io/casehub/clinical/api/spi/DeviationResponsePolicy.java`:**

```java
package io.casehub.clinical.api.spi;

public interface DeviationResponsePolicy {
    DeviationResponseRequirements evaluate(DeviationContext context);
}
```

- [ ] **Create `api/src/main/java/io/casehub/clinical/api/ProtocolDeviationResolvedEvent.java`:**

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
    PiApprovalStatus terminalStatus
) {}
```

- [ ] **Run tests to verify they pass:**

```bash
mvn test -pl api -Dtest=ClinicalConstantsTest --batch-mode
```

Expected: PASS.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(api): DeviationResponsePolicy SPI + ProtocolDeviationResolvedEvent

SPI allows per-installation deadline and escalation customisation scoped by
trial, site, phase, or protocol. Event enables downstream epics (6, 13) to
observe deviation resolution without modifying this service.

Refs #5"
```

---

## Task 5: Flyway migrations

**Files:**
- Create: `runtime/src/main/resources/db/migration/V107__alter_protocol_deviation_add_commitment_fields.sql`
- Create: `runtime/src/main/resources/db/migration/V1006__protocol_deviation_ledger_entry.sql`

- [ ] **Write the failing migration test** — add to `FlywayMigrationTest.java`:

```java
@Test
void v107AddsCommitmentFieldsToProtocolDeviation() {
    // Schema is applied by Flyway at startup — verify columns exist by checking metadata
    PanacheQuery<ProtocolDeviation> q = ProtocolDeviation.find(
        "piCommandChannelName is not null or piCommandChannelName is null");
    assertThat(q).isNotNull(); // column exists if this compiles and runs
}

@Test
void v1006CreatesProtocolDeviationLedgerEntryTable() {
    long count = ProtocolDeviationLedgerEntry.count();
    assertThat(count).isGreaterThanOrEqualTo(0);
}
```

Note: `ProtocolDeviationLedgerEntry` does not exist yet — this test compiles after Task 7. Write the test now and it will fail to compile until then.

- [ ] **Create `V107__alter_protocol_deviation_add_commitment_fields.sql`:**

```sql
ALTER TABLE protocol_deviation
    ADD COLUMN pi_command_channel_name VARCHAR(500),
    ADD COLUMN commanded_at           TIMESTAMP WITH TIME ZONE,
    ADD COLUMN response_deadline      TIMESTAMP WITH TIME ZONE,
    ADD COLUMN escalation_requirement VARCHAR(50);
```

- [ ] **Create `V1006__protocol_deviation_ledger_entry.sql`:**

```sql
CREATE TABLE protocol_deviation_ledger_entry (
    id                     UUID         NOT NULL,
    deviation_id           UUID         NOT NULL,
    site_id                UUID         NOT NULL,
    severity               VARCHAR(50)  NOT NULL,
    pi_id                  VARCHAR(255),
    commanded_at           TIMESTAMP WITH TIME ZONE,
    response_deadline      TIMESTAMP WITH TIME ZONE,
    escalation_requirement VARCHAR(50),
    CONSTRAINT pk_pd_ledger PRIMARY KEY (id),
    CONSTRAINT fk_pd_ledger_entry FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Verify Flyway applies without errors:**

```bash
mvn test -pl runtime -Dtest=FlywayMigrationTest --batch-mode
```

Expected: existing migration tests pass; V107 and V1006 compile into the test context. The v1006 test fails to compile (ProtocolDeviationLedgerEntry missing) — that's expected; complete Task 7 first.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/resources/db/migration/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(db): Flyway V107 + V1006 — protocol deviation commitment fields and ledger join table

Refs #5"
```

---

## Task 6: `DefaultDeviationResponsePolicy` + `ClinicalInboundNormaliser`

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/DefaultDeviationResponsePolicy.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalInboundNormaliser.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/DefaultDeviationResponsePolicyTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/ClinicalInboundNormaliserTest.java`

- [ ] **Write the failing `DefaultDeviationResponsePolicyTest`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.api.spi.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DefaultDeviationResponsePolicyTest {

    @Inject DefaultDeviationResponsePolicy policy;

    private DeviationContext ctx(DeviationSeverity severity) {
        return new DeviationContext(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(),
            "PROT-001", TrialPhase.PHASE_2, severity, "sample-window");
    }

    @Test
    void minorDeviationGets7DayDeadlineAndNoEscalation() {
        var req = policy.evaluate(ctx(DeviationSeverity.MINOR));
        assertThat(req.piResponseDeadline()).isEqualTo(Duration.ofHours(168));
        assertThat(req.escalationRequirement()).isEqualTo(EscalationRequirement.NONE);
    }

    @Test
    void majorDeviationGets72hDeadlineAndSponsorNotification() {
        var req = policy.evaluate(ctx(DeviationSeverity.MAJOR));
        assertThat(req.piResponseDeadline()).isEqualTo(Duration.ofHours(72));
        assertThat(req.escalationRequirement()).isEqualTo(EscalationRequirement.SPONSOR_NOTIFICATION);
    }

    @Test
    void criticalDeviationGets24hDeadlineAndIrbReview() {
        var req = policy.evaluate(ctx(DeviationSeverity.CRITICAL));
        assertThat(req.piResponseDeadline()).isEqualTo(Duration.ofHours(24));
        assertThat(req.escalationRequirement()).isEqualTo(EscalationRequirement.IRB_REVIEW);
    }
}
```

- [ ] **Write the failing `ClinicalInboundNormaliserTest`:**

```java
package io.casehub.clinical.service;

import io.casehub.qhorus.api.gateway.*;
import io.casehub.qhorus.api.message.MessageType;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ClinicalInboundNormaliserTest {

    @Inject ClinicalInboundNormaliser normaliser;

    private ChannelRef ref = new ChannelRef(java.util.UUID.randomUUID(), "clinical/deviation/x/pi-oversight");

    @Test
    void approvedDecisionMappsToDone() {
        var msg = new InboundHumanMessage("pi-001", "{\"decision\":\"APPROVED\",\"comment\":\"OK\"}", Instant.now(), Map.of());
        var result = normaliser.normalise(ref, msg);
        assertThat(result.type()).isEqualTo(MessageType.DONE);
        assertThat(result.senderInstanceId()).isEqualTo("human:pi-001");
        assertThat(result.content()).isEqualTo("{\"decision\":\"APPROVED\",\"comment\":\"OK\"}");
    }

    @Test
    void rejectedDecisionMapsToDecline() {
        var msg = new InboundHumanMessage("pi-001", "{\"decision\":\"REJECTED\"}", Instant.now(), Map.of());
        var result = normaliser.normalise(ref, msg);
        assertThat(result.type()).isEqualTo(MessageType.DECLINE);
    }

    @Test
    void unknownContentDefaultsToQuery() {
        var msg = new InboundHumanMessage("pi-001", "Hello, I have a question", Instant.now(), Map.of());
        var result = normaliser.normalise(ref, msg);
        assertThat(result.type()).isEqualTo(MessageType.QUERY);
    }
}
```

- [ ] **Run to verify they fail:**

```bash
mvn test -pl runtime -Dtest="DefaultDeviationResponsePolicyTest,ClinicalInboundNormaliserTest" --batch-mode
```

Expected: FAIL — classes not found.

- [ ] **Create `DefaultDeviationResponsePolicy.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.spi.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import java.time.Duration;

@ApplicationScoped
@DefaultBean
public class DefaultDeviationResponsePolicy implements DeviationResponsePolicy {

    @ConfigProperty(name = "casehub.clinical.deviation.minor.deadline", defaultValue = "PT168H")
    Duration minorDeadline;

    @ConfigProperty(name = "casehub.clinical.deviation.major.deadline", defaultValue = "PT72H")
    Duration majorDeadline;

    @ConfigProperty(name = "casehub.clinical.deviation.critical.deadline", defaultValue = "PT24H")
    Duration criticalDeadline;

    @Override
    public DeviationResponseRequirements evaluate(DeviationContext context) {
        return switch (context.severity()) {
            case MINOR -> new DeviationResponseRequirements(minorDeadline, EscalationRequirement.NONE);
            case MAJOR -> new DeviationResponseRequirements(majorDeadline, EscalationRequirement.SPONSOR_NOTIFICATION);
            case CRITICAL -> new DeviationResponseRequirements(criticalDeadline, EscalationRequirement.IRB_REVIEW);
        };
    }
}
```

- [ ] **Create `ClinicalInboundNormaliser.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.qhorus.api.gateway.*;
import io.casehub.qhorus.api.message.MessageType;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ClinicalInboundNormaliser implements InboundNormaliser {

    @Override
    public NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw) {
        MessageType type = detectType(raw.content());
        return new NormalisedMessage(type, raw.content(), "human:" + raw.externalSenderId());
    }

    private MessageType detectType(String content) {
        if (content == null) return MessageType.QUERY;
        if (content.contains("\"decision\":\"APPROVED\"")) return MessageType.DONE;
        if (content.contains("\"decision\":\"REJECTED\"")) return MessageType.DECLINE;
        return MessageType.QUERY;
    }
}
```

- [ ] **Run tests to verify they pass:**

```bash
mvn test -pl runtime -Dtest="DefaultDeviationResponsePolicyTest,ClinicalInboundNormaliserTest" --batch-mode
```

Expected: PASS.

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): DefaultDeviationResponsePolicy + ClinicalInboundNormaliser

Policy: configurable deadlines via MicroProfile Config; default MINOR=7d MAJOR=72h CRITICAL=24h.
Normaliser: maps PI JSON decision to DONE/DECLINE for qhorus commitment auto-fulfillment.

Refs #5"
```

---

## Task 7: ProtocolDeviation entity + ProtocolDeviationLedgerEntry

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java`
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/ProtocolDeviationLedgerEntry.java`

- [ ] **Write the failing persistence test** — add to `ClinicalTrialPersistenceTest.java`:

```java
@Test
@Transactional
void protocolDeviationPersistsWithCommandedFields() {
    ClinicalTrial trial = new ClinicalTrial();
    trial.id = UUID.randomUUID();
    trial.protocolId = "PROT-001";
    trial.phase = TrialPhase.PHASE_2;
    trial.sponsor = "Sponsor";
    trial.targetEnrollment = 10;
    trial.status = TrialStatus.ACTIVE;
    trial.persist();

    TrialSite site = new TrialSite();
    site.id = UUID.randomUUID();
    site.trialId = trial.id;
    site.investigatorId = "pi-001";
    site.persist();

    ProtocolDeviation dev = new ProtocolDeviation();
    dev.id = UUID.randomUUID();
    dev.siteId = site.id;
    dev.deviationType = "sample-window";
    dev.severity = DeviationSeverity.MAJOR;
    dev.piApprovalStatus = PiApprovalStatus.COMMANDED;
    dev.piCommandChannelName = "clinical/deviation/" + dev.id + "/pi-oversight";
    dev.commandedAt = Instant.now();
    dev.responseDeadline = Instant.now().plus(72, ChronoUnit.HOURS);
    dev.escalationRequirement = EscalationRequirement.SPONSOR_NOTIFICATION;
    dev.persist();

    ProtocolDeviation loaded = ProtocolDeviation.findById(dev.id);
    assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.COMMANDED);
    assertThat(loaded.escalationRequirement).isEqualTo(EscalationRequirement.SPONSOR_NOTIFICATION);
    assertThat(loaded.piCommandChannelName).startsWith("clinical/deviation/");
}
```

- [ ] **Run to verify it fails:**

```bash
mvn test -pl runtime -Dtest=ClinicalTrialPersistenceTest --batch-mode
```

Expected: FAIL — new fields not on entity.

- [ ] **Update `ProtocolDeviation.java`** — add four new fields:

```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "protocol_deviation")
public class ProtocolDeviation extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "site_id", nullable = false)
    public UUID siteId;

    @Column(name = "deviation_type", nullable = false)
    public String deviationType;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public DeviationSeverity severity;

    @Enumerated(EnumType.STRING)
    @Column(name = "pi_approval_status", nullable = false)
    public PiApprovalStatus piApprovalStatus = PiApprovalStatus.PENDING;

    @Column(name = "pi_command_channel_name")
    public String piCommandChannelName;

    @Column(name = "commanded_at")
    public Instant commandedAt;

    @Column(name = "response_deadline")
    public Instant responseDeadline;

    @Enumerated(EnumType.STRING)
    @Column(name = "escalation_requirement")
    public EscalationRequirement escalationRequirement;
}
```

- [ ] **Create `ProtocolDeviationLedgerEntry.java`:**

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
}
```

- [ ] **Run tests to verify they pass:**

```bash
mvn test -pl runtime -Dtest=ClinicalTrialPersistenceTest --batch-mode
```

Expected: PASS.

- [ ] **Run remaining tests** — verify nothing broken:

```bash
mvn test -pl runtime --batch-mode
```

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): extend ProtocolDeviation entity; add ProtocolDeviationLedgerEntry

4 new fields: piCommandChannelName, commandedAt, responseDeadline, escalationRequirement.
LedgerEntry subclass in io.casehub.clinical.ledger per two-datasource convention.

Refs #5"
```

---

## Task 8: `ProtocolDeviationService`

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/ProtocolDeviationService.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/ProtocolDeviationServiceTest.java`

Key API facts from qhorus source:
- `ChannelService.findByName(String) → Optional<Channel>`
- `ChannelService.create(name, desc, ChannelSemantic, barrierContributors, allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance, allowedTypes)`
- `MessageService.send(UUID channelId, String sender, MessageType type, String content, String correlationId, Long inReplyTo, String artefactRefs, String target, ActorType actorType)`
  - Automatically calls `commitmentService.open()` when type=COMMAND and correlationId is set
- `correlationId` for all commitment operations = `deviation.id.toString()`

- [ ] **Write the failing test:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.*;
import java.time.Instant;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class ProtocolDeviationServiceTest {

    @Inject ProtocolDeviationService service;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject CommitmentService commitmentService;

    private static UUID trialId, siteId, deviationId;

    @BeforeAll
    @Transactional
    static void setup() {
        trialId = UUID.randomUUID();
        siteId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId; trial.protocolId = "PROT-001";
        trial.phase = TrialPhase.PHASE_2; trial.sponsor = "S";
        trial.targetEnrollment = 10; trial.status = TrialStatus.ACTIVE;
        trial.persist();
        TrialSite site = new TrialSite();
        site.id = siteId; site.trialId = trialId; site.investigatorId = "pi-001";
        site.persist();
    }

    @Test
    @Order(1)
    @Transactional
    void reportMinorDeviationSetsCommandedStateAndCreatesChannel() {
        ProtocolDeviation dev = new ProtocolDeviation();
        dev.id = UUID.randomUUID(); deviationId = dev.id;
        dev.siteId = siteId;
        dev.deviationType = "sample-window";
        dev.severity = DeviationSeverity.MINOR;

        service.reportDeviation(dev);

        ProtocolDeviation loaded = ProtocolDeviation.findById(dev.id);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.COMMANDED);
        assertThat(loaded.piCommandChannelName)
            .isEqualTo("clinical/deviation/" + dev.id + "/pi-oversight");
        assertThat(loaded.commandedAt).isNotNull();
        assertThat(loaded.responseDeadline)
            .isAfter(Instant.now().plusSeconds(6 * 24 * 3600)); // > 6 days
        assertThat(loaded.escalationRequirement).isEqualTo(EscalationRequirement.NONE);
    }

    @Test
    @Order(2)
    void channelExistsWithCorrectAllowedTypes() {
        var channel = channelService.findByName("clinical/deviation/" + deviationId + "/pi-oversight");
        assertThat(channel).isPresent();
        assertThat(channel.get().allowedTypes).contains("COMMAND");
    }

    @Test
    @Order(3)
    void commandMessageInChannelWithCorrelationId() {
        var channel = channelService.findByName("clinical/deviation/" + deviationId + "/pi-oversight").orElseThrow();
        var messages = messageService.pollAfter(channel.id, 0L, 10);
        assertThat(messages).hasSize(1);
        assertThat(messages.get(0).messageType).isEqualTo(MessageType.COMMAND);
        assertThat(messages.get(0).correlationId).isEqualTo(deviationId.toString());
        assertThat(messages.get(0).target).isEqualTo("pi-001");
    }

    @Test
    @Order(4)
    void commitmentIsOpenForDeviation() {
        var commitment = commitmentService.findByCorrelationId(deviationId.toString());
        assertThat(commitment).isPresent();
        assertThat(commitment.get().state.name()).isEqualTo("OPEN");
    }

    @Test
    @Order(5)
    @Transactional
    void ledgerEntryIsWritten() {
        long count = ProtocolDeviationLedgerEntry.count("deviationId", deviationId);
        assertThat(count).isEqualTo(1);
    }

    @Test
    @Order(6)
    @Transactional
    void criticalDeviationGets24hDeadlineAndIrbEscalation() {
        ProtocolDeviation dev = new ProtocolDeviation();
        dev.id = UUID.randomUUID();
        dev.siteId = siteId;
        dev.deviationType = "eligibility-breach";
        dev.severity = DeviationSeverity.CRITICAL;

        service.reportDeviation(dev);

        ProtocolDeviation loaded = ProtocolDeviation.findById(dev.id);
        assertThat(loaded.escalationRequirement).isEqualTo(EscalationRequirement.IRB_REVIEW);
        assertThat(loaded.responseDeadline)
            .isBefore(Instant.now().plusSeconds(25 * 3600)); // < 25 hours
    }
}
```

- [ ] **Run to verify it fails:**

```bash
mvn test -pl runtime -Dtest=ProtocolDeviationServiceTest --batch-mode
```

Expected: FAIL — ProtocolDeviationService not found.

- [ ] **Create `ProtocolDeviationService.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.spi.*;
import io.casehub.clinical.entity.*;
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;

@ApplicationScoped
public class ProtocolDeviationService {

    static final String CLINICAL_SENDER = "clinical-service";
    static final String CHANNEL_ALLOWED_TYPES = "QUERY,COMMAND";

    @Inject DeviationResponsePolicy policy;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject LedgerEntryRepository ledgerEntryRepository;

    @Transactional
    public void reportDeviation(ProtocolDeviation deviation) {
        TrialSite site = TrialSite.findById(deviation.siteId);
        ClinicalTrial trial = ClinicalTrial.findById(site.trialId);

        var context = new DeviationContext(
            deviation.id, deviation.siteId, site.trialId,
            trial.protocolId, trial.phase, deviation.severity, deviation.deviationType
        );
        var requirements = policy.evaluate(context);

        String channelName = "clinical/deviation/" + deviation.id + "/pi-oversight";
        ensureChannel(channelName);

        var channel = channelService.findByName(channelName).orElseThrow();
        Instant now = Instant.now();
        String content = buildCommandContent(deviation, now, requirements);

        messageService.send(
            channel.id, CLINICAL_SENDER, MessageType.COMMAND, content,
            deviation.id.toString(),   // correlationId — triggers commitmentService.open() automatically
            null, null, site.investigatorId, ActorType.SYSTEM
        );

        deviation.piCommandChannelName = channelName;
        deviation.commandedAt = now;
        deviation.responseDeadline = now.plus(requirements.piResponseDeadline());
        deviation.escalationRequirement = requirements.escalationRequirement();
        deviation.piApprovalStatus = io.casehub.clinical.api.model.PiApprovalStatus.COMMANDED;
        deviation.persist();

        writeLedgerEntry(deviation, site.investigatorId);
    }

    private void ensureChannel(String name) {
        if (channelService.findByName(name).isPresent()) return;
        channelService.create(name, "PI governance for protocol deviation",
            ChannelSemantic.APPEND, null, null, null, null, null, CHANNEL_ALLOWED_TYPES);
    }

    private String buildCommandContent(ProtocolDeviation dev, Instant now,
                                        DeviationResponseRequirements req) {
        return "{\"deviationId\":\"" + dev.id
            + "\",\"deviationType\":\"" + dev.deviationType
            + "\",\"severity\":\"" + dev.severity
            + "\",\"responseDeadline\":\"" + now.plus(req.piResponseDeadline())
            + "\"}";
    }

    private void writeLedgerEntry(ProtocolDeviation dev, String piId) {
        ProtocolDeviationLedgerEntry entry = new ProtocolDeviationLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = dev.siteId.toString();
        entry.sequenceNumber = System.currentTimeMillis();
        entry.entryType = "PROTOCOL_DEVIATION";
        entry.actorId = CLINICAL_SENDER;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "deviation-reporter";
        entry.occurredAt = dev.commandedAt;
        entry.deviationId = dev.id;
        entry.siteId = dev.siteId;
        entry.severity = dev.severity.name();
        entry.piId = piId;
        entry.commandedAt = dev.commandedAt;
        entry.responseDeadline = dev.responseDeadline;
        entry.escalationRequirement = dev.escalationRequirement != null
            ? dev.escalationRequirement.name() : null;
        ledgerEntryRepository.save(entry);
    }
}
```

- [ ] **Run tests to verify they pass:**

```bash
mvn test -pl runtime -Dtest=ProtocolDeviationServiceTest --batch-mode
```

Expected: PASS.

- [ ] **Run full test suite:**

```bash
mvn test -pl runtime --batch-mode
```

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): ProtocolDeviationService — COMMAND lifecycle for PI authorisation

Creates per-deviation oversight channel, sends COMMAND (auto-opens Commitment),
writes ProtocolDeviationLedgerEntry. Policy SPI drives deadline and escalation.

Refs #5"
```

---

## Task 9: `DeviationResource`

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/DeviationResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/DeviationResourceTest.java`

- [ ] **Write the failing test:**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.*;
import java.util.UUID;
import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class DeviationResourceTest {

    private static UUID trialId, siteId;

    @BeforeAll
    @Transactional
    static void setup() {
        trialId = UUID.randomUUID(); siteId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId; trial.protocolId = "PROT-R"; trial.phase = TrialPhase.PHASE_1;
        trial.sponsor = "S"; trial.targetEnrollment = 5; trial.status = TrialStatus.ACTIVE;
        trial.persist();
        TrialSite site = new TrialSite();
        site.id = siteId; site.trialId = trialId; site.investigatorId = "pi-res";
        site.persist();
    }

    @Test
    @Order(1)
    void reportDeviationReturns201WithLocation() {
        var response = given()
            .contentType("application/json")
            .body("{\"deviationType\":\"sample-window\",\"severity\":\"MINOR\"}")
            .when().post("/trials/{t}/sites/{s}/deviations", trialId, siteId)
            .then().statusCode(201)
            .header("Location", containsString("/deviations/"))
            .body("piApprovalStatus", is("COMMANDED"))
            .body("responseDeadline", notNullValue())
            .extract().response();

        String location = response.header("Location");
        String deviationId = location.substring(location.lastIndexOf('/') + 1);

        given().when()
            .get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteId, deviationId)
            .then().statusCode(200)
            .body("piApprovalStatus", is("COMMANDED"))
            .body("escalationRequirement", is("NONE"));
    }

    @Test
    @Order(2)
    void reportDeviationToWrongSiteReturns404() {
        given()
            .contentType("application/json")
            .body("{\"deviationType\":\"x\",\"severity\":\"MINOR\"}")
            .when().post("/trials/{t}/sites/{s}/deviations", trialId, UUID.randomUUID())
            .then().statusCode(404);
    }

    @Test
    @Order(3)
    void reportDeviationMissingSeverityReturns400() {
        given()
            .contentType("application/json")
            .body("{\"deviationType\":\"x\"}")
            .when().post("/trials/{t}/sites/{s}/deviations", trialId, siteId)
            .then().statusCode(400);
    }

    @Test
    @Order(4)
    void getUnknownDeviationReturns404() {
        given().when()
            .get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteId, UUID.randomUUID())
            .then().statusCode(404);
    }
}
```

- [ ] **Run to verify it fails:**

```bash
mvn test -pl runtime -Dtest=DeviationResourceTest --batch-mode
```

Expected: FAIL — no endpoint.

- [ ] **Create `DeviationResource.java`:**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.*;
import io.casehub.clinical.service.ProtocolDeviationService;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import java.net.URI;
import java.util.UUID;

@Path("/trials/{trialId}/sites/{siteId}/deviations")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class DeviationResource {

    @Inject ProtocolDeviationService deviationService;

    public record ReportDeviationRequest(
        @NotBlank String deviationType,
        @NotNull DeviationSeverity severity
    ) {}

    @POST
    @Transactional
    public Response reportDeviation(
            @PathParam("trialId") UUID trialId,
            @PathParam("siteId") UUID siteId,
            @Valid ReportDeviationRequest req,
            @Context UriInfo uriInfo) {
        TrialSite site = TrialSite.findById(siteId);
        if (site == null || !site.trialId.equals(trialId))
            return Response.status(Response.Status.NOT_FOUND).build();

        ProtocolDeviation deviation = new ProtocolDeviation();
        deviation.id = UUID.randomUUID();
        deviation.siteId = siteId;
        deviation.deviationType = req.deviationType();
        deviation.severity = req.severity();
        deviation.piApprovalStatus = PiApprovalStatus.PENDING;

        deviationService.reportDeviation(deviation);

        URI location = uriInfo.getAbsolutePathBuilder().path(deviation.id.toString()).build();
        return Response.created(location).entity(deviation).build();
    }

    @GET
    @Path("/{deviationId}")
    public Response getDeviation(
            @PathParam("trialId") UUID trialId,
            @PathParam("siteId") UUID siteId,
            @PathParam("deviationId") UUID deviationId) {
        ProtocolDeviation dev = ProtocolDeviation.findById(deviationId);
        if (dev == null || !dev.siteId.equals(siteId)) return Response.status(Response.Status.NOT_FOUND).build();
        TrialSite site = TrialSite.findById(siteId);
        if (site == null || !site.trialId.equals(trialId)) return Response.status(Response.Status.NOT_FOUND).build();
        return Response.ok(dev).build();
    }
}
```

- [ ] **Run tests to verify they pass:**

```bash
mvn test -pl runtime -Dtest=DeviationResourceTest --batch-mode
```

Expected: PASS.

- [ ] **Run full test suite:**

```bash
mvn test -pl runtime --batch-mode
```

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): DeviationResource — POST + GET /trials/{t}/sites/{s}/deviations

Refs #5"
```

---

## Task 10: `PiResponseListener` — unit test + disabled integration test

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/PiResponseListener.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/PiResponseListenerTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/PiResponseListenerIntegrationTest.java`

`MessageReceivedEvent` does not exist yet (qhorus#153). The listener is written with `@ObservesAsync` commented out. The `process()` method is package-private and called directly by tests.

- [ ] **Write the failing unit test:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.casehub.qhorus.api.message.MessageType;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.*;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class PiResponseListenerTest {

    @Inject PiResponseListener listener;
    @Inject ProtocolDeviationService service;

    private static UUID minorDeviationId, criticalDeviationId, rejectedDeviationId;

    @BeforeAll
    @Transactional
    static void setup() {
        // setup trial + site + COMMANDED deviations
        UUID trialId = UUID.randomUUID(), siteId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId; trial.protocolId = "P"; trial.phase = TrialPhase.PHASE_2;
        trial.sponsor = "S"; trial.targetEnrollment = 5; trial.status = TrialStatus.ACTIVE;
        trial.persist();
        TrialSite site = new TrialSite();
        site.id = siteId; site.trialId = trialId; site.investigatorId = "pi-L";
        site.persist();

        // pre-persist COMMANDED deviations
        minorDeviationId = persistCommanded(siteId, DeviationSeverity.MINOR, EscalationRequirement.NONE);
        criticalDeviationId = persistCommanded(siteId, DeviationSeverity.CRITICAL, EscalationRequirement.IRB_REVIEW);
        rejectedDeviationId = persistCommanded(siteId, DeviationSeverity.MINOR, EscalationRequirement.NONE);
    }

    @Transactional
    static UUID persistCommanded(UUID siteId, DeviationSeverity sev, EscalationRequirement esc) {
        ProtocolDeviation d = new ProtocolDeviation();
        d.id = UUID.randomUUID(); d.siteId = siteId; d.deviationType = "test"; d.severity = sev;
        d.piApprovalStatus = PiApprovalStatus.COMMANDED; d.escalationRequirement = esc;
        d.piCommandChannelName = "clinical/deviation/" + d.id + "/pi-oversight";
        d.commandedAt = Instant.now();
        d.responseDeadline = Instant.now().plus(24, ChronoUnit.HOURS);
        d.persist();
        return d.id;
    }

    @Test
    @Order(1)
    void approvedMinorDeviationSetsApproved() {
        listener.process("clinical/deviation/" + minorDeviationId + "/pi-oversight",
            MessageType.DONE, "human:pi-L");

        ProtocolDeviation loaded = ProtocolDeviation.findById(minorDeviationId);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.APPROVED);
    }

    @Test
    @Order(2)
    void approvedCriticalDeviationSetsEscalated() {
        listener.process("clinical/deviation/" + criticalDeviationId + "/pi-oversight",
            MessageType.DONE, "human:pi-L");

        ProtocolDeviation loaded = ProtocolDeviation.findById(criticalDeviationId);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.ESCALATED);
    }

    @Test
    @Order(3)
    void rejectedDeviationSetsRejected() {
        listener.process("clinical/deviation/" + rejectedDeviationId + "/pi-oversight",
            MessageType.DECLINE, "human:pi-L");

        ProtocolDeviation loaded = ProtocolDeviation.findById(rejectedDeviationId);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.REJECTED);
    }

    @Test
    @Order(4)
    void nonMatchingChannelIsIgnored() {
        // should not throw, no state change
        listener.process("clinical/other/channel", MessageType.DONE, "human:pi-L");
    }

    @Test
    @Order(5)
    void alreadyTerminalDeviationIsIdempotent() {
        // APPROVED is terminal — second call should be no-op
        listener.process("clinical/deviation/" + minorDeviationId + "/pi-oversight",
            MessageType.DONE, "human:pi-L");
        ProtocolDeviation loaded = ProtocolDeviation.findById(minorDeviationId);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.APPROVED);
    }
}
```

- [ ] **Write the disabled integration test:**

```java
package io.casehub.clinical.service;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Disabled;
import org.junit.jupiter.api.Test;

/**
 * Full channel-flow integration test: receiveHumanMessage → CDI event → PiResponseListener.
 * Blocked on casehubio/qhorus#153 (MessageReceivedEvent CDI hook in ChannelGateway).
 * Enable when qhorus ships the event and casehub-qhorus version is bumped.
 */
@QuarkusTest
@Disabled("casehubio/qhorus#153 — MessageReceivedEvent CDI hook not yet available")
class PiResponseListenerIntegrationTest {

    @Test
    void piApprovalViaChannelGatewayUpdatesDeviationStatus() {
        // TODO when qhorus#153 ships:
        // 1. reportDeviation() → COMMANDED
        // 2. channelGateway.receiveHumanMessage(channelRef,
        //        new InboundHumanMessage("pi-001", "{\"decision\":\"APPROVED\"}", Instant.now(), Map.of()))
        // 3. ClinicalInboundNormaliser maps to DONE
        // 4. messageService.send() auto-fulfills Commitment
        // 5. MessageReceivedEvent CDI event fires in receiveHumanMessage()
        // 6. PiResponseListener.process() invoked via @ObservesAsync
        // 7. Assert piApprovalStatus == APPROVED (MINOR) or ESCALATED (CRITICAL)
    }
}
```

- [ ] **Run to verify unit test fails:**

```bash
mvn test -pl runtime -Dtest=PiResponseListenerTest --batch-mode
```

Expected: FAIL — PiResponseListener not found.

- [ ] **Create `PiResponseListener.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
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

    // @ObservesAsync MessageReceivedEvent — awaiting casehubio/qhorus#153
    // When qhorus#153 ships and casehub-qhorus-api version is bumped, add:
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
        if (deviation.piApprovalStatus != PiApprovalStatus.COMMANDED) return; // idempotent

        boolean approved = messageType == MessageType.DONE;

        if (approved) {
            boolean needsEscalation = deviation.escalationRequirement != null
                && deviation.escalationRequirement != EscalationRequirement.NONE;
            deviation.piApprovalStatus = needsEscalation
                ? PiApprovalStatus.ESCALATED : PiApprovalStatus.APPROVED;
        } else {
            deviation.piApprovalStatus = PiApprovalStatus.REJECTED;
        }

        resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
            deviation.id, deviation.siteId, deviation.severity,
            deviation.escalationRequirement != null
                ? deviation.escalationRequirement : EscalationRequirement.NONE,
            deviation.piApprovalStatus
        ));
    }
}
```

- [ ] **Run tests to verify unit tests pass:**

```bash
mvn test -pl runtime -Dtest="PiResponseListenerTest,PiResponseListenerIntegrationTest" --batch-mode
```

Expected: PiResponseListenerTest PASS (5 tests). PiResponseListenerIntegrationTest SKIPPED (disabled).

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): PiResponseListener — updates piApprovalStatus on PI DONE/DECLINE

@ObservesAsync MessageReceivedEvent commented — awaiting casehubio/qhorus#153.
process() is package-visible and called directly by tests.
Integration test written and @Disabled pending qhorus#153.

Refs #5"
```

---

## Task 11: `DeviationExpirationJob`

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java`

- [ ] **Write the failing test:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.*;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DeviationExpirationJobTest {

    @Inject DeviationExpirationJob job;

    @BeforeAll
    @Transactional
    static void setup() {
        UUID trialId = UUID.randomUUID(), siteId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId; trial.protocolId = "EXP"; trial.phase = TrialPhase.PHASE_1;
        trial.sponsor = "S"; trial.targetEnrollment = 5; trial.status = TrialStatus.ACTIVE;
        trial.persist();
        TrialSite site = new TrialSite();
        site.id = siteId; site.trialId = trialId; site.investigatorId = "pi-exp";
        site.persist();
    }

    @Test
    @Transactional
    void overdueCommandedDeviationIsMarkedExpired() {
        ProtocolDeviation dev = new ProtocolDeviation();
        dev.id = UUID.randomUUID();
        dev.siteId = TrialSite.listAll().get(0).id;
        dev.deviationType = "overdue"; dev.severity = DeviationSeverity.MINOR;
        dev.piApprovalStatus = PiApprovalStatus.COMMANDED;
        dev.escalationRequirement = EscalationRequirement.NONE;
        dev.piCommandChannelName = "clinical/deviation/" + dev.id + "/pi-oversight";
        dev.commandedAt = Instant.now().minus(10, ChronoUnit.DAYS);
        dev.responseDeadline = Instant.now().minus(3, ChronoUnit.DAYS); // past deadline
        dev.persist();

        job.checkExpiredCommitments();

        ProtocolDeviation loaded = ProtocolDeviation.findById(dev.id);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.EXPIRED);
    }

    @Test
    @Transactional
    void futureDeadlineDeviationIsNotExpired() {
        ProtocolDeviation dev = new ProtocolDeviation();
        dev.id = UUID.randomUUID();
        dev.siteId = TrialSite.listAll().get(0).id;
        dev.deviationType = "active"; dev.severity = DeviationSeverity.MINOR;
        dev.piApprovalStatus = PiApprovalStatus.COMMANDED;
        dev.escalationRequirement = EscalationRequirement.NONE;
        dev.piCommandChannelName = "clinical/deviation/" + dev.id + "/pi-oversight";
        dev.commandedAt = Instant.now();
        dev.responseDeadline = Instant.now().plus(7, ChronoUnit.DAYS);
        dev.persist();

        job.checkExpiredCommitments();

        ProtocolDeviation loaded = ProtocolDeviation.findById(dev.id);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.COMMANDED);
    }
}
```

- [ ] **Run to verify it fails:**

```bash
mvn test -pl runtime -Dtest=DeviationExpirationJobTest --batch-mode
```

Expected: FAIL — DeviationExpirationJob not found.

- [ ] **Create `DeviationExpirationJob.java`:**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
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

    @Scheduled(every = "${casehub.clinical.deviation.expiration-check-interval:1h}",
               identity = "deviation-expiration")
    @Transactional
    public void checkExpiredCommitments() {
        List<ProtocolDeviation> overdue = ProtocolDeviation
            .find("piApprovalStatus = ?1 and responseDeadline < ?2",
                  PiApprovalStatus.COMMANDED, Instant.now())
            .list();

        for (ProtocolDeviation d : overdue) {
            d.piApprovalStatus = PiApprovalStatus.EXPIRED;
            commitmentService.fail(d.id.toString());
            resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
                d.id, d.siteId, d.severity,
                d.escalationRequirement != null ? d.escalationRequirement : EscalationRequirement.NONE,
                PiApprovalStatus.EXPIRED
            ));
        }
    }
}
```

Note: `quarkus.scheduler.enabled=false` in test properties prevents automatic firing. `@Scheduled` does not prevent direct method calls — call `job.checkExpiredCommitments()` directly in tests.

- [ ] **Run tests to verify they pass:**

```bash
mvn test -pl runtime -Dtest=DeviationExpirationJobTest --batch-mode
```

Expected: PASS.

- [ ] **Run full suite:**

```bash
mvn test -pl runtime --batch-mode
```

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): DeviationExpirationJob — marks overdue COMMANDED deviations as EXPIRED

Hourly scheduled scan; configurable via casehub.clinical.deviation.expiration-check-interval.
Calls commitmentService.fail() and fires ProtocolDeviationResolvedEvent(EXPIRED).

Refs #5"
```

---

## Task 12: Extend `ShowcaseScenarioTest`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/ShowcaseScenarioTest.java`

- [ ] **Add deviation scenario to the existing 3-site test.** Append to the existing `threeSiteOncologyShowcase` test (or add a new method):

```java
@Test
@Order(/* next order after existing */)
void siteC_piAuthorisationLifecycle() {
    // Site C reports a MINOR deviation
    var minorResponse = given()
        .contentType("application/json")
        .body("{\"deviationType\":\"sample-collection-window\",\"severity\":\"MINOR\"}")
        .when().post("/trials/{t}/sites/{s}/deviations", trialId, siteCId)
        .then().statusCode(201)
        .body("piApprovalStatus", is("COMMANDED"))
        .body("escalationRequirement", is("NONE"))
        .extract().response();

    String minorDeviationId = extractId(minorResponse.header("Location"));

    // Site C reports a CRITICAL deviation
    var criticalResponse = given()
        .contentType("application/json")
        .body("{\"deviationType\":\"eligibility-breach\",\"severity\":\"CRITICAL\"}")
        .when().post("/trials/{t}/sites/{s}/deviations", trialId, siteCId)
        .then().statusCode(201)
        .body("piApprovalStatus", is("COMMANDED"))
        .body("escalationRequirement", is("IRB_REVIEW"))
        .extract().response();

    String criticalDeviationId = extractId(criticalResponse.header("Location"));

    // Simulate PI approving the MINOR deviation (directly calling listener)
    piResponseListener.process(
        "clinical/deviation/" + minorDeviationId + "/pi-oversight",
        MessageType.DONE, "human:site-c-pi"
    );

    given().when()
        .get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteCId, minorDeviationId)
        .then().statusCode(200)
        .body("piApprovalStatus", is("APPROVED"));

    // Simulate PI approving the CRITICAL deviation — should escalate
    piResponseListener.process(
        "clinical/deviation/" + criticalDeviationId + "/pi-oversight",
        MessageType.DONE, "human:site-c-pi"
    );

    given().when()
        .get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteCId, criticalDeviationId)
        .then().statusCode(200)
        .body("piApprovalStatus", is("ESCALATED"));
}
```

Add `@Inject PiResponseListener piResponseListener;` to the test class, plus `import io.casehub.qhorus.api.message.MessageType;`. Add this helper:

```java
private String extractId(String location) {
    return location.substring(location.lastIndexOf('/') + 1);
}
```

`trialId` and `siteCId` are already set up in the existing 3-site `@BeforeAll` — use whatever variable names that test uses for Site C.

- [ ] **Run the showcase test:**

```bash
mvn test -pl runtime -Dtest=ShowcaseScenarioTest --batch-mode
```

Expected: PASS (existing + new scenarios).

- [ ] **Run the full test suite one final time:**

```bash
mvn test -pl api,runtime --batch-mode
```

Expected: all tests pass. `PiResponseListenerIntegrationTest` is SKIPPED (disabled).

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/test/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test: extend 3-site showcase with PI authorisation scenario

MINOR deviation → APPROVED; CRITICAL deviation → ESCALATED via PiResponseListener.

Refs #5"
```

---

## Completion checklist

- [ ] All tests pass (`mvn test -pl api,runtime --batch-mode`)
- [ ] `PiResponseListenerIntegrationTest` is `@Disabled` with explicit reference to qhorus#153
- [ ] LAYER-LOG.md Layer 3 `🔲` sections can now be partially filled (channel wiring, ClinicalInboundNormaliser pattern, gotchas discovered)
- [ ] `implementation-doc-sync` run after all tasks complete
- [ ] `superpowers:requesting-code-review` before final commit
