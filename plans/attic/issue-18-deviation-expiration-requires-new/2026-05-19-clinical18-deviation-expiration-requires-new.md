# DeviationExpirer REQUIRES_NEW — Per-Deviation Transaction Isolation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace DeviationExpirationJob's single @Transactional batch loop with a DeviationExpirer helper bean that runs each deviation in an independent REQUIRES_NEW sub-transaction, so one deviation's JPA failure cannot roll back others.

**Architecture:** A new `DeviationExpirer @ApplicationScoped` owns two transaction boundaries: `findOverdueIds()` (REQUIRED — short read) and `expireOne(UUID)` (REQUIRES_NEW — per-deviation isolation). `DeviationExpirationJob` becomes a non-transactional orchestrator. Tests remove `@Transactional` from test methods and use a `TestDeviationPersister` helper that commits deviations before the job runs.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache Active Record, CDI `@Transactional(REQUIRES_NEW)`, JUnit 5 with `@QuarkusTest`, AssertJ.

**Issues:** Refs casehubio/clinical#18

---

## File Map

| Action | File |
|--------|------|
| CREATE | `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java` |
| MODIFY | `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java` |
| CREATE | `runtime/src/test/java/io/casehub/clinical/service/TestDeviationPersister.java` |
| MODIFY | `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java` |
| CREATE | `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirerTest.java` |

---

## Task 1: Test Infrastructure — `TestDeviationPersister` + `DeviationExpirerTest` (failing)

Tests for the new `DeviationExpirer` bean fail until it exists. The `TestDeviationPersister` helper commits deviations in their own transaction so REQUIRES_NEW can see them.

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/TestDeviationPersister.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirerTest.java`

- [ ] **Step 1.1: Create `TestDeviationPersister`**

This is a CDI bean in `src/test/java` that commits a deviation before returning. Tests inject it instead of persisting inline with `@Transactional` on the test method.

Create `runtime/src/test/java/io/casehub/clinical/service/TestDeviationPersister.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

@ApplicationScoped
class TestDeviationPersister {

    @Transactional
    UUID persistCommanded(UUID siteId, DeviationSeverity severity,
                          EscalationRequirement esc, Instant deadline) {
        ProtocolDeviation d = new ProtocolDeviation();
        d.id = UUID.randomUUID();
        d.siteId = siteId;
        d.deviationType = "test";
        d.severity = severity;
        d.piApprovalStatus = PiApprovalStatus.COMMANDED;
        d.escalationRequirement = esc;
        d.piCommandChannelName = "clinical/deviation/" + d.id + "/pi-oversight";
        d.commandedAt = Instant.now().minus(10, ChronoUnit.DAYS);
        d.responseDeadline = deadline;
        d.persist();
        return d.id;
    }
}
```

- [ ] **Step 1.2: Write `DeviationExpirerTest` (all failing)**

Create `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirerTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.*;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class DeviationExpirerTest {

    @Inject DeviationExpirer expirer;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject TestDeviationPersister persister;

    private UUID siteId;

    @BeforeAll
    @Transactional
    void setup() {
        UUID trialId = UUID.randomUUID();
        siteId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId; trial.protocolId = "EXPR"; trial.phase = TrialPhase.PHASE_I;
        trial.sponsor = "S"; trial.targetEnrollment = 5; trial.status = TrialStatus.ACTIVE;
        trial.persist();
        TrialSite site = new TrialSite();
        site.id = siteId; site.trialId = trialId; site.investigatorId = "pi-expr";
        site.persist();
    }

    @Test
    void expireOne_skipsNonExistentDeviation() {
        expirer.expireOne(UUID.randomUUID()); // must not throw
    }

    @Test
    void expireOne_skipsAlreadyTerminalDeviation() {
        UUID devId = persister.persistCommanded(siteId, DeviationSeverity.MINOR,
            EscalationRequirement.NONE, Instant.now().minus(1, ChronoUnit.DAYS));

        expirer.expireOne(devId); // first call — commits
        expirer.expireOne(devId); // second call — guard skips (EXPIRED ≠ COMMANDED)

        assertThat(ledgerRepo.findBySubjectId(devId)).hasSize(1); // not 2
    }

    @Test
    void expireOne_expiresCommandedDeviation() {
        UUID devId = persister.persistCommanded(siteId, DeviationSeverity.MINOR,
            EscalationRequirement.NONE, Instant.now().minus(1, ChronoUnit.DAYS));

        expirer.expireOne(devId);

        ProtocolDeviation loaded = ProtocolDeviation.findById(devId);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.EXPIRED);

        var entries = ledgerRepo.findBySubjectId(devId);
        assertThat(entries).hasSize(1);
        assertThat(((ProtocolDeviationLedgerEntry) entries.get(0)).terminalStatus).isEqualTo("EXPIRED");
    }
}
```

- [ ] **Step 1.3: Run and confirm tests fail**

```bash
mvn test -pl runtime -Dtest=DeviationExpirerTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: FAIL — `DeviationExpirer` does not exist yet.

- [ ] **Step 1.4: Commit failing tests**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/TestDeviationPersister.java \
  runtime/src/test/java/io/casehub/clinical/service/DeviationExpirerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "test: DeviationExpirerTest — failing tests for per-deviation REQUIRES_NEW isolation — Refs #18"
```

---

## Task 2: Restructure `DeviationExpirationJobTest` (failing assertions)

Remove `@Transactional` from the three test methods. Inject `TestDeviationPersister` to commit deviations before the job runs. The assertions remain the same — they now run outside a test transaction, reading committed data.

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java`

- [ ] **Step 2.1: Rewrite `DeviationExpirationJobTest`**

Replace the entire file:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.casehub.clinical.ledger.ProtocolDeviationLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.*;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class DeviationExpirationJobTest {

    @Inject DeviationExpirationJob job;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject TestDeviationPersister persister;

    private UUID siteId;

    @BeforeAll
    @Transactional
    void setup() {
        UUID trialId = UUID.randomUUID();
        siteId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId; trial.protocolId = "EXP"; trial.phase = TrialPhase.PHASE_I;
        trial.sponsor = "S"; trial.targetEnrollment = 5; trial.status = TrialStatus.ACTIVE;
        trial.persist();
        TrialSite site = new TrialSite();
        site.id = siteId; site.trialId = trialId; site.investigatorId = "pi-exp";
        site.persist();
    }

    @Test
    void overdueCommandedDeviationIsMarkedExpired() {
        UUID devId = persister.persistCommanded(siteId, DeviationSeverity.MINOR,
            EscalationRequirement.NONE, Instant.now().minus(3, ChronoUnit.DAYS));

        job.checkExpiredCommitments();

        ProtocolDeviation loaded = ProtocolDeviation.findById(devId);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.EXPIRED);

        var entries = ledgerRepo.findBySubjectId(devId);
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
    void twoOverdueDeviationsEachGetIndependentLedgerEntry() {
        UUID devId1 = persister.persistCommanded(siteId, DeviationSeverity.MINOR,
            EscalationRequirement.NONE, Instant.now().minus(1, ChronoUnit.DAYS));
        UUID devId2 = persister.persistCommanded(siteId, DeviationSeverity.MINOR,
            EscalationRequirement.NONE, Instant.now().minus(1, ChronoUnit.DAYS));

        job.checkExpiredCommitments();

        assertThat(ledgerRepo.findBySubjectId(devId1)).hasSize(1);
        assertThat(ledgerRepo.findBySubjectId(devId2)).hasSize(1);
    }

    @Test
    void futureDeadlineDeviationIsNotExpired() {
        UUID devId = persister.persistCommanded(siteId, DeviationSeverity.MINOR,
            EscalationRequirement.NONE, Instant.now().plus(7, ChronoUnit.DAYS));

        job.checkExpiredCommitments();

        ProtocolDeviation loaded = ProtocolDeviation.findById(devId);
        assertThat(loaded.piApprovalStatus).isEqualTo(PiApprovalStatus.COMMANDED);
    }
}
```

- [ ] **Step 2.2: Run and confirm Task 2 tests fail**

```bash
mvn test -pl runtime -Dtest=DeviationExpirationJobTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: FAIL — `DeviationExpirer` doesn't exist yet (job still references old code).

- [ ] **Step 2.3: Commit failing test restructure**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "test: restructure DeviationExpirationJobTest — remove @Transactional, use TestDeviationPersister — Refs #18"
```

---

## Task 3: Implement `DeviationExpirer`

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java`

- [ ] **Step 3.1: Create `DeviationExpirer`**

Create `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.runtime.message.CommitmentService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

/**
 * Handles per-deviation expiration with independent transaction isolation.
 *
 * <p>{@link #findOverdueIds()} reads all COMMANDED deviations past their deadline in a
 * short REQUIRED transaction. {@link #expireOne(UUID)} processes each deviation in a
 * dedicated REQUIRES_NEW transaction — if that deviation's writes fail, only its
 * sub-transaction rolls back; other already-committed expirations are unaffected.
 */
@ApplicationScoped
public class DeviationExpirer {

    @Inject CommitmentService commitmentService;
    @Inject Event<ProtocolDeviationResolvedEvent> resolvedEvent;
    @Inject DeviationLedgerWriter ledgerWriter;

    @Transactional
    public List<UUID> findOverdueIds() {
        return ProtocolDeviation
            .find("piApprovalStatus = ?1 and responseDeadline < ?2",
                  PiApprovalStatus.COMMANDED, Instant.now())
            .<ProtocolDeviation>list()
            .stream().map(d -> d.id).toList();
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void expireOne(UUID deviationId) {
        ProtocolDeviation d = ProtocolDeviation.findById(deviationId);
        if (d == null || d.piApprovalStatus != PiApprovalStatus.COMMANDED) return;

        d.piApprovalStatus = PiApprovalStatus.EXPIRED;
        commitmentService.fail(d.id.toString());
        ledgerWriter.writeResolutionEntry(d, PiApprovalStatus.EXPIRED,
            "system", ActorType.SYSTEM, "deviation-expiration-job");
        resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
            d.id, d.siteId, d.severity,
            d.escalationRequirement != null ? d.escalationRequirement : EscalationRequirement.NONE,
            PiApprovalStatus.EXPIRED
        ));
    }
}
```

- [ ] **Step 3.2: Run `DeviationExpirerTest` and confirm it passes**

```bash
mvn test -pl runtime -Dtest=DeviationExpirerTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 3.3: Commit `DeviationExpirer`**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "feat: DeviationExpirer — REQUIRES_NEW per-deviation transaction isolation — Refs #18"
```

---

## Task 4: Refactor `DeviationExpirationJob`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java`

- [ ] **Step 4.1: Replace `DeviationExpirationJob`**

Replace the entire file:

```java
package io.casehub.clinical.service;

import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.UUID;

/**
 * Hourly scan that expires COMMANDED deviations whose responseDeadline has passed.
 *
 * <p>Each deviation is processed in an independent REQUIRES_NEW transaction via
 * {@link DeviationExpirer#expireOne(UUID)}. A failure expiring one deviation rolls back
 * only that deviation's sub-transaction — previously committed expirations are not affected.
 * Failed deviations remain COMMANDED and are retried on the next scheduled run.
 *
 * <p>The scheduler is disabled in tests ({@code quarkus.scheduler.enabled=false}).
 * Call {@link #checkExpiredCommitments()} directly in tests.
 */
@ApplicationScoped
public class DeviationExpirationJob {

    @Inject DeviationExpirer expirer;

    @Scheduled(every = "${casehub.clinical.deviation.expiration-check-interval:1h}",
               identity = "deviation-expiration")
    public void checkExpiredCommitments() {
        for (UUID id : expirer.findOverdueIds()) {
            try {
                expirer.expireOne(id);
            } catch (Exception e) {
                org.jboss.logging.Logger.getLogger(DeviationExpirationJob.class)
                    .errorf(e, "Failed to expire deviation %s — will retry next run", id);
            }
        }
    }
}
```

- [ ] **Step 4.2: Run `DeviationExpirationJobTest` and confirm all pass**

```bash
mvn test -pl runtime -Dtest=DeviationExpirationJobTest --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 4.3: Run full test suite**

```bash
mvn test --batch-mode -f /Users/mdproctor/claude/casehub/clinical/pom.xml
```

Expected: all previously passing tests still pass (68 + new tests).

- [ ] **Step 4.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m \
  "refactor: DeviationExpirationJob delegates to DeviationExpirer — Closes #18"
```
