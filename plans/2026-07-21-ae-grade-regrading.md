# AE Grade Regrading Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #135 — feat: AE grade regrading — support grade changes over time
**Issue group:** #135

**Goal:** Record adverse event grade transitions over time, trigger
compliance re-evaluation on upgrades, and add grade as a DTW-matchable
trajectory dimension.

**Architecture:** New `AeGradeChange` entity records grade history.
`AdverseEventService.regradeAdverseEvent()` updates the current grade,
writes ledger, and fires `AeGradeChangedEvent` after commit. Five
listeners handle upgrade re-evaluation (escalation, SUSAR, regulatory,
trajectory, safety officer). `AeTrajectoryBuilder` merges grade change
timestamps into the time series as a fifth dimension.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, Flyway, casehub-ledger,
casehub-engine PlanItemStore, casehub-neocortex CBR

## Global Constraints

- Flyway default datasource: V127 (next available)
- Flyway qhorus datasource: V2030 (next available)
- `AeGradeChangeLedgerEntry` lives in `io.casehub.clinical.ledger` (qhorus PU)
- `AeGradeChange` entity lives in `io.casehub.clinical.entity` (default PU)
- `AeGradeChangedEvent` record lives in `io.casehub.clinical.api` (api module)
- Tests use `drop-and-create` + Flyway disabled; `InMemoryLedgerEntryRepository`
- `@InjectMock` replaces CDI beans — stub in `@BeforeEach`
- After-commit event firing via `TransactionSynchronizationRegistry`
- All REST endpoints under `/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events`
- `@RolesAllowed({INVESTIGATOR, COORDINATOR})` for regrade/history endpoints
- IntelliJ MCP mandatory for all .java edits — `project_path=/Users/mdproctor/claude/casehub/clinical`

---

### Task 1: Domain Model — AeGradeChange entity, AeGradeChangedEvent, Flyway migration

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/entity/AeGradeChange.java`
- Create: `api/src/main/java/io/casehub/clinical/api/AeGradeChangedEvent.java`
- Create: `runtime/src/main/resources/db/migration/default/V127__ae_grade_change.sql`
- Test: `runtime/src/test/java/io/casehub/clinical/entity/AeGradeChangeTest.java`

**Interfaces:**
- Produces: `AeGradeChange` entity with `findByAdverseEventId(UUID)`,
  `findLatestByAdverseEventId(UUID)` static queries
- Produces: `AeGradeChangedEvent` record with `isUpgrade()`, `isDowngrade()`

- [ ] **Step 1: Write AeGradeChange entity query tests**

Create `runtime/src/test/java/io/casehub/clinical/entity/AeGradeChangeTest.java`:

```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static io.casehub.clinical.api.ClinicalGroups.*;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {SPONSOR, INVESTIGATOR, COORDINATOR})
class AeGradeChangeTest {

    private UUID aeId;

    @BeforeEach
    @Transactional
    void setup() {
        AeGradeChange.deleteAll();
        aeId = UUID.randomUUID();
    }

    @Test
    @Transactional
    void findByAdverseEventId_returnsOrderedHistory() {
        Instant t1 = Instant.parse("2026-01-01T00:00:00Z");
        Instant t2 = Instant.parse("2026-01-02T00:00:00Z");
        Instant t3 = Instant.parse("2026-01-03T00:00:00Z");

        persistChange(aeId, null, CtcaeGrade.GRADE_1, t1);
        persistChange(aeId, CtcaeGrade.GRADE_1, CtcaeGrade.GRADE_3, t3);
        persistChange(aeId, CtcaeGrade.GRADE_1, CtcaeGrade.GRADE_2, t2);

        List<AeGradeChange> history = AeGradeChange.findByAdverseEventId(aeId);
        assertEquals(3, history.size());
        assertNull(history.get(0).previousGrade);
        assertEquals(CtcaeGrade.GRADE_2, history.get(1).newGrade);
        assertEquals(CtcaeGrade.GRADE_3, history.get(2).newGrade);
    }

    @Test
    @Transactional
    void findByAdverseEventId_emptyForUnknownId() {
        assertTrue(AeGradeChange.findByAdverseEventId(UUID.randomUUID()).isEmpty());
    }

    @Test
    @Transactional
    void findLatestByAdverseEventId_returnsMostRecent() {
        Instant t1 = Instant.parse("2026-01-01T00:00:00Z");
        Instant t2 = Instant.parse("2026-01-02T00:00:00Z");

        persistChange(aeId, null, CtcaeGrade.GRADE_1, t1);
        persistChange(aeId, CtcaeGrade.GRADE_1, CtcaeGrade.GRADE_3, t2);

        AeGradeChange latest = AeGradeChange.findLatestByAdverseEventId(aeId);
        assertNotNull(latest);
        assertEquals(CtcaeGrade.GRADE_3, latest.newGrade);
    }

    @Test
    @Transactional
    void findLatestByAdverseEventId_nullForUnknownId() {
        assertNull(AeGradeChange.findLatestByAdverseEventId(UUID.randomUUID()));
    }

    private void persistChange(UUID adverseEventId, CtcaeGrade prev, CtcaeGrade next, Instant at) {
        AeGradeChange gc = new AeGradeChange();
        gc.id = UUID.randomUUID();
        gc.adverseEventId = adverseEventId;
        gc.previousGrade = prev;
        gc.newGrade = next;
        gc.changedAt = at;
        gc.changedBy = "test";
        gc.reason = "test reason";
        gc.persist();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=AeGradeChangeTest --batch-mode
```
Expected: compilation failure — `AeGradeChange` class does not exist.

- [ ] **Step 3: Create AeGradeChangedEvent record in api module**

Create `api/src/main/java/io/casehub/clinical/api/AeGradeChangedEvent.java`:

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;

import java.time.Instant;
import java.util.UUID;

public record AeGradeChangedEvent(
    UUID aeId,
    UUID enrollmentId,
    UUID siteId,
    CtcaeGrade previousGrade,
    CtcaeGrade newGrade,
    Instant changedAt,
    String changedBy,
    String tenantId
) {
    public boolean isUpgrade() {
        return previousGrade == null || newGrade.ordinal() > previousGrade.ordinal();
    }

    public boolean isDowngrade() {
        return previousGrade != null && newGrade.ordinal() < previousGrade.ordinal();
    }
}
```

- [ ] **Step 4: Write AeGradeChangedEvent unit tests in api module**

Create `api/src/test/java/io/casehub/clinical/api/AeGradeChangedEventTest.java`:

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class AeGradeChangedEventTest {

    @Test
    void isUpgrade_grade1To3_true() {
        var event = event(CtcaeGrade.GRADE_1, CtcaeGrade.GRADE_3);
        assertTrue(event.isUpgrade());
        assertFalse(event.isDowngrade());
    }

    @Test
    void isDowngrade_grade3To1_true() {
        var event = event(CtcaeGrade.GRADE_3, CtcaeGrade.GRADE_1);
        assertFalse(event.isUpgrade());
        assertTrue(event.isDowngrade());
    }

    @Test
    void sameGrade_neitherUpgradeNorDowngrade() {
        var event = event(CtcaeGrade.GRADE_2, CtcaeGrade.GRADE_2);
        assertFalse(event.isUpgrade());
        assertFalse(event.isDowngrade());
    }

    @Test
    void nullPreviousGrade_isUpgrade() {
        var event = event(null, CtcaeGrade.GRADE_1);
        assertTrue(event.isUpgrade());
        assertFalse(event.isDowngrade());
    }

    private AeGradeChangedEvent event(CtcaeGrade prev, CtcaeGrade next) {
        return new AeGradeChangedEvent(
            UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(),
            prev, next, Instant.now(), "test", "default");
    }
}
```

- [ ] **Step 5: Create AeGradeChange entity**

Create `runtime/src/main/java/io/casehub/clinical/entity/AeGradeChange.java`:

```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

@Entity
@Table(name = "ae_grade_change")
public class AeGradeChange extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "adverse_event_id", nullable = false)
    public UUID adverseEventId;

    @Enumerated(EnumType.STRING)
    @Column(name = "previous_grade")
    public CtcaeGrade previousGrade;

    @Enumerated(EnumType.STRING)
    @Column(name = "new_grade", nullable = false)
    public CtcaeGrade newGrade;

    @Column(name = "changed_at", nullable = false)
    public Instant changedAt;

    @Column(name = "changed_by", nullable = false)
    public String changedBy;

    @Column(length = 500)
    public String reason;

    public static List<AeGradeChange> findByAdverseEventId(UUID aeId) {
        return list("adverseEventId = ?1 order by changedAt asc", aeId);
    }

    public static AeGradeChange findLatestByAdverseEventId(UUID aeId) {
        return find("adverseEventId = ?1 order by changedAt desc", aeId).firstResult();
    }
}
```

- [ ] **Step 6: Create Flyway migration V127**

Create `runtime/src/main/resources/db/migration/default/V127__ae_grade_change.sql`:

```sql
CREATE TABLE ae_grade_change (
    id UUID PRIMARY KEY,
    adverse_event_id UUID NOT NULL,
    previous_grade VARCHAR(20),
    new_grade VARCHAR(20) NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE NOT NULL,
    changed_by VARCHAR(255) NOT NULL,
    reason VARCHAR(500),
    CONSTRAINT fk_ae_grade_change_ae FOREIGN KEY (adverse_event_id)
        REFERENCES adverse_event(id)
);
CREATE INDEX idx_ae_grade_change_ae_id ON ae_grade_change(adverse_event_id);

INSERT INTO ae_grade_change (id, adverse_event_id, previous_grade, new_grade, changed_at, changed_by, reason)
SELECT gen_random_uuid(), id, NULL, grade, reported_at, 'migration', 'Retroactive initial grade entry'
FROM adverse_event
WHERE NOT EXISTS (
    SELECT 1 FROM ae_grade_change gc WHERE gc.adverse_event_id = adverse_event.id
);
```

- [ ] **Step 7: Run all tests — verify green**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=AeGradeChangeTest,AeGradeChangedEventTest --batch-mode
```
Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/src/main/java/io/casehub/clinical/api/AeGradeChangedEvent.java api/src/test/java/io/casehub/clinical/api/AeGradeChangedEventTest.java runtime/src/main/java/io/casehub/clinical/entity/AeGradeChange.java runtime/src/main/resources/db/migration/default/V127__ae_grade_change.sql runtime/src/test/java/io/casehub/clinical/entity/AeGradeChangeTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#135): AeGradeChange entity, AeGradeChangedEvent, V127 migration

Refs #135"
```

---

### Task 2: Ledger — AeGradeChangeLedgerEntry, writer, ComplianceSupplement

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/AeGradeChangeLedgerEntry.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeLedgerWriter.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2030__ae_grade_change_ledger_entry.sql`
- Test: `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeLedgerWriterTest.java`

**Interfaces:**
- Consumes: `AeGradeChange` entity (Task 1)
- Produces: `AeGradeChangeLedgerWriter.writeGradeChangeEntry(AdverseEvent, CtcaeGrade previousGrade, String reason)`
- Produces: `ClinicalComplianceSupplement.gradeChange()`

- [ ] **Step 1: Write ledger writer test**

Create `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeLedgerWriterTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.ledger.AeGradeChangeLedgerEntry;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class AeGradeChangeLedgerWriterTest {

    private LedgerEntryRepository repo;
    private AeGradeChangeLedgerWriter writer;

    @BeforeEach
    void setUp() {
        repo = mock(LedgerEntryRepository.class);
        Clock clock = Clock.fixed(Instant.parse("2026-07-21T12:00:00Z"), ZoneOffset.UTC);
        when(repo.findLatestBySubjectId(any(), eq("default"))).thenReturn(Optional.empty());
        writer = new AeGradeChangeLedgerWriter();
        writer.ledgerEntryRepository = repo;
        writer.clock = clock;
    }

    @Test
    void writeGradeChangeEntry_writesCorrectFields() {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_3;

        writer.writeGradeChangeEntry(ae, CtcaeGrade.GRADE_1, "Patient condition worsened");

        var captor = ArgumentCaptor.forClass(AeGradeChangeLedgerEntry.class);
        verify(repo).save(captor.capture(), eq("default"));
        var entry = captor.getValue();
        assertEquals("GRADE_1", entry.previousGrade);
        assertEquals("GRADE_3", entry.newGrade);
        assertEquals("Patient condition worsened", entry.reason);
        assertNotNull(entry.id);
        assertEquals(1, entry.sequenceNumber);
        assertNotNull(entry.supplements);
        assertFalse(entry.supplements.isEmpty());
    }

    @Test
    void writeGradeChangeEntry_nullPreviousGrade_storedAsNull() {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_2;

        writer.writeGradeChangeEntry(ae, null, "Initial report");

        var captor = ArgumentCaptor.forClass(AeGradeChangeLedgerEntry.class);
        verify(repo).save(captor.capture(), eq("default"));
        assertNull(captor.getValue().previousGrade);
    }

    @Test
    void writeGradeChangeEntry_sequenceNumberIncrements() {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_3;

        var existing = new AeGradeChangeLedgerEntry();
        existing.sequenceNumber = 3;
        when(repo.findLatestBySubjectId(ae.id, "default")).thenReturn(Optional.of(existing));

        writer.writeGradeChangeEntry(ae, CtcaeGrade.GRADE_1, "reason");

        var captor = ArgumentCaptor.forClass(AeGradeChangeLedgerEntry.class);
        verify(repo).save(captor.capture(), eq("default"));
        assertEquals(4, captor.getValue().sequenceNumber);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=AeGradeChangeLedgerWriterTest --batch-mode
```
Expected: compilation failure — classes don't exist.

- [ ] **Step 3: Create AeGradeChangeLedgerEntry**

Create `runtime/src/main/java/io/casehub/clinical/ledger/AeGradeChangeLedgerEntry.java`:

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import java.nio.charset.StandardCharsets;

@Entity
@Table(name = "ae_grade_change_ledger_entry")
@DiscriminatorValue("AE_GRADE_CHANGE")
public class AeGradeChangeLedgerEntry extends JpaLedgerEntry {

    @Column(name = "previous_grade")
    public String previousGrade;

    @Column(name = "new_grade", nullable = false)
    public String newGrade;

    @Column(length = 500)
    public String reason;

    @Column(name = "changed_by")
    public String changedBy;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
            previousGrade != null ? previousGrade : "",
            newGrade,
            reason != null ? reason : "",
            changedBy != null ? changedBy : ""
        ).getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 4: Add ClinicalComplianceSupplement.gradeChange()**

Add to `ClinicalComplianceSupplement.java` after the `cbrRetrieval()` method:

```java
public static ComplianceSupplement gradeChange() {
    ComplianceSupplement s = new JpaComplianceSupplement();
    s.planRef = "ICH E6(R3) §5.17 — adverse event grade reassessment audit trail";
    s.algorithmRef = "AdverseEventService.regradeAdverseEvent (clinician-initiated grade change)";
    s.humanOverrideAvailable = true;
    return s;
}
```

- [ ] **Step 5: Create AeGradeChangeLedgerWriter**

Create `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeLedgerWriter.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ClinicalActors;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.ledger.AeGradeChangeLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Clock;
import java.util.UUID;

@ApplicationScoped
public class AeGradeChangeLedgerWriter {

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Clock clock;

    public void writeGradeChangeEntry(AdverseEvent ae, CtcaeGrade previousGrade, String reason) {
        AeGradeChangeLedgerEntry entry = new AeGradeChangeLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = ae.id;
        entry.sequenceNumber = nextSequenceNumber(ae.id);
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "AdverseEventRegrader";
        entry.occurredAt = clock.instant();
        entry.previousGrade = previousGrade != null ? previousGrade.name() : null;
        entry.newGrade = ae.grade.name();
        entry.reason = reason;
        entry.changedBy = ClinicalActors.CLINICAL_SERVICE;
        entry.attach(ClinicalComplianceSupplement.gradeChange());
        ledgerEntryRepository.save(entry, "default");
    }

    private int nextSequenceNumber(UUID aeId) {
        return ledgerEntryRepository.findLatestBySubjectId(aeId, "default")
            .map(e -> e.sequenceNumber + 1)
            .orElse(1);
    }
}
```

- [ ] **Step 6: Create Flyway migration V2030**

Create `runtime/src/main/resources/db/migration/qhorus/V2030__ae_grade_change_ledger_entry.sql`:

```sql
CREATE TABLE ae_grade_change_ledger_entry (
    id UUID PRIMARY KEY REFERENCES ledger_entry(id),
    previous_grade VARCHAR(20),
    new_grade VARCHAR(20) NOT NULL,
    reason VARCHAR(500),
    changed_by VARCHAR(255)
);
```

- [ ] **Step 7: Run test — verify green**

```bash
mvn test -pl runtime -Dtest=AeGradeChangeLedgerWriterTest --batch-mode
```
Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/ledger/AeGradeChangeLedgerEntry.java runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeLedgerWriter.java runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java runtime/src/main/resources/db/migration/qhorus/V2030__ae_grade_change_ledger_entry.sql runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#135): AeGradeChangeLedgerEntry, writer, ComplianceSupplement

Refs #135"
```

---

### Task 3: Service Layer — regradeAdverseEvent, initial grade in reportAdverseEvent, memory store

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/memory/ClinicalMemoryService.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/AdverseEventRegradeTest.java`

**Interfaces:**
- Consumes: `AeGradeChange` entity (Task 1), `AeGradeChangeLedgerWriter` (Task 2)
- Produces: `AdverseEventService.regradeAdverseEvent(UUID aeId, CtcaeGrade newGrade, String changedBy, String reason)`
- Produces: `ClinicalMemoryService.storeAeRegrade(UUID aeId, UUID enrollmentId, UUID siteId, UUID trialId, CtcaeGrade previousGrade, CtcaeGrade newGrade, String tenantId)`

- [ ] **Step 1: Write regradeAdverseEvent unit test**

Create `runtime/src/test/java/io/casehub/clinical/service/AdverseEventRegradeTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AeGradeChangedEvent;
import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.spi.AdverseEventEscalationPolicy;
import io.casehub.clinical.cbr.AeTrajectoryAlertService;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.AeGradeChange;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.memory.ClinicalMemoryService;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.InjectMock;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static io.casehub.clinical.api.ClinicalGroups.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {SPONSOR, INVESTIGATOR, COORDINATOR})
class AdverseEventRegradeTest {

    @Inject AdverseEventService service;
    @Inject FixedCurrentPrincipal principal;
    @InjectMock AeGradeChangeLedgerWriter gradeChangeLedgerWriter;
    @InjectMock ClinicalMemoryService memoryService;

    private UUID aeId;

    @BeforeEach
    @Transactional
    void setup() {
        AeGradeChange.deleteAll();
        AdverseEvent.deleteAll();
        PatientEnrollment.deleteAll();
        TrialSite.deleteAll();

        UUID trialId = UUID.randomUUID();
        UUID siteId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();

        TrialSite site = new TrialSite();
        site.id = siteId;
        site.trialId = trialId;
        site.investigatorId = "inv-1";
        site.tenantId = principal.tenancyId();
        site.persist();

        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = enrollmentId;
        enrollment.siteId = siteId;
        enrollment.patientId = "P-001";
        enrollment.tenantId = principal.tenancyId();
        enrollment.persist();

        aeId = UUID.randomUUID();
        AdverseEvent ae = new AdverseEvent();
        ae.id = aeId;
        ae.enrollmentId = enrollmentId;
        ae.grade = CtcaeGrade.GRADE_1;
        ae.occurredAt = Instant.now().minus(Duration.ofHours(2));
        ae.reportedAt = Instant.now().minus(Duration.ofHours(1));
        ae.slaDeadline = ae.reportedAt.plus(Duration.ofDays(7));
        ae.tenantId = principal.tenancyId();
        ae.persist();
    }

    @Test
    void regrade_updatesGradeAndCreatesHistory() {
        service.regradeAdverseEvent(aeId, CtcaeGrade.GRADE_3, "dr-smith", "Condition worsened");

        AdverseEvent ae = AdverseEvent.findById(aeId);
        assertEquals(CtcaeGrade.GRADE_3, ae.grade);

        List<AeGradeChange> history = AeGradeChange.findByAdverseEventId(aeId);
        assertEquals(1, history.size());
        assertEquals(CtcaeGrade.GRADE_1, history.get(0).previousGrade);
        assertEquals(CtcaeGrade.GRADE_3, history.get(0).newGrade);
        assertEquals("dr-smith", history.get(0).changedBy);
        assertEquals("Condition worsened", history.get(0).reason);
    }

    @Test
    void regrade_sameGrade_noOp() {
        service.regradeAdverseEvent(aeId, CtcaeGrade.GRADE_1, "dr-smith", "No change");

        List<AeGradeChange> history = AeGradeChange.findByAdverseEventId(aeId);
        assertTrue(history.isEmpty());
        verify(gradeChangeLedgerWriter, never()).writeGradeChangeEntry(any(), any(), any());
    }

    @Test
    void regrade_upgrade_tightensSla() {
        AdverseEvent aeBefore = AdverseEvent.findById(aeId);
        Instant oldDeadline = aeBefore.slaDeadline;

        service.regradeAdverseEvent(aeId, CtcaeGrade.GRADE_3, "dr-smith", "Escalated");

        AdverseEvent ae = AdverseEvent.findById(aeId);
        assertTrue(ae.slaDeadline.isBefore(oldDeadline));
    }

    @Test
    void regrade_downgrade_doesNotRelaxSla() {
        service.regradeAdverseEvent(aeId, CtcaeGrade.GRADE_3, "dr-smith", "Up");
        AdverseEvent aeAfterUpgrade = AdverseEvent.findById(aeId);
        Instant tightDeadline = aeAfterUpgrade.slaDeadline;

        service.regradeAdverseEvent(aeId, CtcaeGrade.GRADE_1, "dr-smith", "Down");

        AdverseEvent ae = AdverseEvent.findById(aeId);
        assertEquals(tightDeadline, ae.slaDeadline);
    }

    @Test
    void regrade_writesLedgerEntry() {
        service.regradeAdverseEvent(aeId, CtcaeGrade.GRADE_3, "dr-smith", "Worsened");

        verify(gradeChangeLedgerWriter).writeGradeChangeEntry(any(), eq(CtcaeGrade.GRADE_1), eq("Worsened"));
    }

    @Test
    void regrade_storesMemory() {
        service.regradeAdverseEvent(aeId, CtcaeGrade.GRADE_3, "dr-smith", "Worsened");

        verify(memoryService).storeAeRegrade(eq(aeId), any(), any(), any(),
            eq(CtcaeGrade.GRADE_1), eq(CtcaeGrade.GRADE_3), eq(principal.tenancyId()));
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=AdverseEventRegradeTest --batch-mode
```
Expected: compilation failure — `regradeAdverseEvent` and `storeAeRegrade` don't exist.

- [ ] **Step 3: Add ClinicalMemoryService.storeAeRegrade()**

Add to `ClinicalMemoryService.java` after `storeAeReport()`:

```java
public void storeAeRegrade(final UUID aeId, final UUID enrollmentId, final UUID siteId,
                            final UUID trialId, final CtcaeGrade previousGrade,
                            final CtcaeGrade newGrade, final String tenantId) {
    final Map<String, String> attrs = Map.of(
        MemoryAttributeKeys.ACTOR_ID, ACTOR,
        MemoryAttributeKeys.OUTCOME, "REGRADED",
        ClinicalMemoryAttributes.GRADE, newGrade.name());
    final String text = "AE " + aeId + " regraded " + previousGrade.name() + " → " + newGrade.name()
        + " for enrollment " + enrollmentId + " at site " + siteId;

    try {
        store.store(new MemoryInput("patient:" + enrollmentId, ClinicalMemoryDomains.PATIENT,
            tenantId, null, text, attrs));
    } catch (Exception e) {
        LOG.warnf(e, "ClinicalMemoryService: storeAeRegrade failed for aeId=%s — ignored", aeId);
    }
    try {
        store.store(new MemoryInput("site:" + siteId, ClinicalMemoryDomains.SITE,
            tenantId, null, text, attrs));
    } catch (Exception e) {
        LOG.warnf(e, "ClinicalMemoryService: storeAeRegrade (site) failed for siteId=%s — ignored", siteId);
    }
    if (trialId != null && siteId != null) {
        try {
            store.store(new MemoryInput("trial:" + trialId, ClinicalMemoryDomains.DRUG,
                tenantId, null, text,
                Map.of(MemoryAttributeKeys.ACTOR_ID, ACTOR,
                    MemoryAttributeKeys.OUTCOME, "REGRADED",
                    ClinicalMemoryAttributes.GRADE, newGrade.name(),
                    ClinicalMemoryAttributes.SITE_ID, siteId.toString())));
        } catch (Exception e) {
            LOG.warnf(e, "ClinicalMemoryService: storeAeRegrade (drug) failed for trialId=%s — ignored", trialId);
        }
    }
}
```

- [ ] **Step 4: Add regradeAdverseEvent() to AdverseEventService**

Add to `AdverseEventService.java` after `reportAdverseEvent()`. Inject `AeGradeChangeLedgerWriter` and `Event<AeGradeChangedEvent>`:

New fields:
```java
@Inject AeGradeChangeLedgerWriter gradeChangeLedgerWriter;
@Inject Event<AeGradeChangedEvent> gradeChangedEvents;
```

New method:
```java
@Transactional
public void regradeAdverseEvent(UUID aeId, CtcaeGrade newGrade, String changedBy, String reason) {
    AdverseEvent ae = AdverseEvent.findById(aeId);
    if (ae == null) return;
    if (newGrade == ae.grade) return;

    CtcaeGrade previousGrade = ae.grade;

    AeGradeChange change = new AeGradeChange();
    change.id = UUID.randomUUID();
    change.adverseEventId = aeId;
    change.previousGrade = previousGrade;
    change.newGrade = newGrade;
    change.changedAt = Instant.now();
    change.changedBy = changedBy;
    change.reason = reason;
    change.persist();

    ae.grade = newGrade;

    if (newGrade.ordinal() > previousGrade.ordinal()) {
        Instant newDeadline = Instant.now().plus(newGrade.sla().orElseThrow());
        if (newDeadline.isBefore(ae.slaDeadline)) {
            ae.slaDeadline = newDeadline;
        }
    }

    gradeChangeLedgerWriter.writeGradeChangeEntry(ae, previousGrade, reason);

    PatientEnrollment enrollment = PatientEnrollment.findById(ae.enrollmentId);
    UUID siteId = enrollment != null ? enrollment.siteId : null;
    TrialSite site = siteId != null ? TrialSite.findById(siteId) : null;
    UUID trialId = site != null ? site.trialId : null;

    memoryService.storeAeRegrade(aeId, ae.enrollmentId, siteId, trialId,
        previousGrade, newGrade, ae.tenantId);

    var event = new AeGradeChangedEvent(
        aeId, ae.enrollmentId, siteId, previousGrade, newGrade,
        change.changedAt, changedBy, ae.tenantId);
    txSync.registerInterposedSynchronization(new Synchronization() {
        @Override public void beforeCompletion() {}
        @Override public void afterCompletion(int status) {
            if (status == Status.STATUS_COMMITTED) {
                gradeChangedEvents.fireAsync(event);
            }
        }
    });
}
```

Add import for `AeGradeChangedEvent`, `AeGradeChange`, `AeGradeChangeLedgerWriter`.

- [ ] **Step 5: Add initial grade history to reportAdverseEvent()**

In `AdverseEventService.reportAdverseEvent()`, after `ae.persist();` (line ~79), add:

```java
AeGradeChange initial = new AeGradeChange();
initial.id = UUID.randomUUID();
initial.adverseEventId = ae.id;
initial.previousGrade = null;
initial.newGrade = ae.grade;
initial.changedAt = ae.reportedAt;
initial.changedBy = "system";
initial.reason = "Initial report";
initial.persist();
```

- [ ] **Step 6: Run tests — verify green**

```bash
mvn test -pl runtime -Dtest=AdverseEventRegradeTest --batch-mode
```
Expected: all pass.

- [ ] **Step 7: Run existing AdverseEventServiceTest — verify no regressions**

```bash
mvn test -pl runtime -Dtest=AdverseEventServiceTest --batch-mode
```
Expected: all pass (initial grade history insert is additive).

- [ ] **Step 8: Commit**

Commit all modified/created files with message:
```
feat(#135): regradeAdverseEvent service method + initial grade history

Refs #135
```

---

### Task 4: REST Endpoints — regrade and grade-history

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/AeRegradeResourceTest.java`

**Interfaces:**
- Consumes: `AdverseEventService.regradeAdverseEvent()` (Task 3),
  `AeGradeChange.findByAdverseEventId()` (Task 1)
- Produces: `POST /{enrollmentId}/adverse-events/{aeId}/regrade`,
  `GET /{enrollmentId}/adverse-events/{aeId}/grade-history`

- [ ] **Step 1: Write REST endpoint tests**

Create `runtime/src/test/java/io/casehub/clinical/resource/AeRegradeResourceTest.java`:

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.AeGradeChange;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.UUID;

import static io.casehub.clinical.api.ClinicalGroups.*;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {SPONSOR, INVESTIGATOR, COORDINATOR})
class AeRegradeResourceTest {

    @Inject FixedCurrentPrincipal principal;

    private UUID trialId, siteId, enrollmentId, aeId;

    @BeforeEach
    @Transactional
    void setup() {
        AeGradeChange.deleteAll();
        AdverseEvent.deleteAll();
        PatientEnrollment.deleteAll();
        TrialSite.deleteAll();
        ClinicalTrial.deleteAll();

        trialId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId;
        trial.protocolId = "PROTO-001";
        trial.tenantId = principal.tenancyId();
        trial.persist();

        siteId = UUID.randomUUID();
        TrialSite site = new TrialSite();
        site.id = siteId;
        site.trialId = trialId;
        site.investigatorId = "inv-1";
        site.tenantId = principal.tenancyId();
        site.persist();

        enrollmentId = UUID.randomUUID();
        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = enrollmentId;
        enrollment.siteId = siteId;
        enrollment.patientId = "P-001";
        enrollment.tenantId = principal.tenancyId();
        enrollment.persist();

        aeId = UUID.randomUUID();
        AdverseEvent ae = new AdverseEvent();
        ae.id = aeId;
        ae.enrollmentId = enrollmentId;
        ae.grade = CtcaeGrade.GRADE_1;
        ae.occurredAt = Instant.now().minus(Duration.ofHours(2));
        ae.reportedAt = Instant.now().minus(Duration.ofHours(1));
        ae.slaDeadline = ae.reportedAt.plus(Duration.ofDays(7));
        ae.tenantId = principal.tenancyId();
        ae.persist();
    }

    @Test
    void regrade_updatesGrade() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"grade\": \"GRADE_3\", \"reason\": \"Condition worsened\"}")
            .post("/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/regrade",
                trialId, siteId, enrollmentId, aeId)
            .then()
            .statusCode(200)
            .body("grade", equalTo("GRADE_3"));
    }

    @Test
    void regrade_nonexistentAe_returns404() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"grade\": \"GRADE_3\", \"reason\": \"test\"}")
            .post("/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/regrade",
                trialId, siteId, enrollmentId, UUID.randomUUID())
            .then()
            .statusCode(404);
    }

    @Test
    void gradeHistory_returnsOrderedEntries() {
        // insert some history
        given()
            .contentType(ContentType.JSON)
            .body("{\"grade\": \"GRADE_2\", \"reason\": \"Moderate\"}")
            .post("/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/regrade",
                trialId, siteId, enrollmentId, aeId);

        given()
            .contentType(ContentType.JSON)
            .body("{\"grade\": \"GRADE_3\", \"reason\": \"Severe\"}")
            .post("/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/regrade",
                trialId, siteId, enrollmentId, aeId);

        given()
            .get("/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/grade-history",
                trialId, siteId, enrollmentId, aeId)
            .then()
            .statusCode(200)
            .body("$.size()", equalTo(2))
            .body("[0].newGrade", equalTo("GRADE_2"))
            .body("[1].newGrade", equalTo("GRADE_3"));
    }

    @Test
    void gradeHistory_nonexistentAe_returns404() {
        given()
            .get("/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events/{aeId}/grade-history",
                trialId, siteId, enrollmentId, UUID.randomUUID())
            .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=AeRegradeResourceTest --batch-mode
```
Expected: 404 — endpoints don't exist yet.

- [ ] **Step 3: Add regrade and grade-history endpoints to PatientResource**

Add a request record inside `PatientResource`:

```java
public record RegradeRequest(CtcaeGrade grade, String reason) {}
```

Add a response record for grade history:

```java
public record GradeHistoryEntry(UUID id, String previousGrade, String newGrade,
                                 Instant changedAt, String changedBy, String reason) {}
```

Add regrade endpoint after `getAdverseEvent()`:

```java
@POST
@Path("/{enrollmentId}/adverse-events/{aeId}/regrade")
@RolesAllowed({ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
public Response regradeAdverseEvent(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @PathParam("enrollmentId") UUID enrollmentId,
        @PathParam("aeId") UUID aeId,
        @Valid RegradeRequest req) {
    PatientEnrollment enrollment = PatientEnrollment.findByIdForTenant(enrollmentId, principal);
    if (enrollment == null || !enrollment.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();
    AdverseEvent ae = AdverseEvent.findByIdForTenant(aeId, principal);
    if (ae == null || !ae.enrollmentId.equals(enrollmentId))
        return Response.status(Response.Status.NOT_FOUND).build();

    adverseEventService.regradeAdverseEvent(aeId, req.grade(), principal.name(), req.reason());

    ae = AdverseEvent.findById(aeId);
    return Response.ok(ae).build();
}
```

Add grade-history endpoint:

```java
@GET
@Path("/{enrollmentId}/adverse-events/{aeId}/grade-history")
@RolesAllowed({ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
public Response getGradeHistory(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @PathParam("enrollmentId") UUID enrollmentId,
        @PathParam("aeId") UUID aeId) {
    PatientEnrollment enrollment = PatientEnrollment.findByIdForTenant(enrollmentId, principal);
    if (enrollment == null || !enrollment.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();
    AdverseEvent ae = AdverseEvent.findByIdForTenant(aeId, principal);
    if (ae == null || !ae.enrollmentId.equals(enrollmentId))
        return Response.status(Response.Status.NOT_FOUND).build();

    var history = AeGradeChange.findByAdverseEventId(aeId).stream()
        .map(gc -> new GradeHistoryEntry(gc.id,
            gc.previousGrade != null ? gc.previousGrade.name() : null,
            gc.newGrade.name(), gc.changedAt, gc.changedBy, gc.reason))
        .toList();
    return Response.ok(history).build();
}
```

- [ ] **Step 4: Run tests — verify green**

```bash
mvn test -pl runtime -Dtest=AeRegradeResourceTest --batch-mode
```
Expected: all pass.

- [ ] **Step 5: Commit**

```
feat(#135): REST endpoints for AE regrade and grade history

Refs #135
```

---

### Task 5: Upgrade Re-evaluation Listeners

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeEscalationListener.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeSusarListener.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeRegulatoryListener.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeTrajectoryListener.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeSafetyOfficerListener.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java` — add `startEscalationForRegrade()`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightCaseService.java` — add `reevaluateForRegrade()`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java` — add `reevaluateForRegrade()`
- Test: `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeEscalationListenerTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeSusarListenerTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeRegulatoryListenerTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeTrajectoryListenerTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeSafetyOfficerListenerTest.java`

**Interfaces:**
- Consumes: `AeGradeChangedEvent` (Task 1), all existing case services
- Produces: Five listeners observing `@ObservesAsync AeGradeChangedEvent`
- Produces: `AeEscalationCaseService.startEscalationForRegrade(UUID aeId, UUID enrollmentId, UUID siteId, CtcaeGrade grade, String tenantId)`
- Produces: `SusarOversightCaseService.reevaluateForRegrade(UUID aeId, UUID siteId, String tenantId)`
- Produces: `RegulatorySubmissionCaseService.reevaluateForRegrade(UUID aeId, UUID siteId, String tenantId)`

- [ ] **Step 1: Write escalation listener test**

Create `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeEscalationListenerTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AeGradeChangedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.mockito.Mockito.*;

class AeGradeChangeEscalationListenerTest {

    private AeEscalationCaseService escalationService;
    private AeGradeChangeEscalationListener listener;

    @BeforeEach
    void setUp() {
        escalationService = mock(AeEscalationCaseService.class);
        listener = new AeGradeChangeEscalationListener();
        listener.escalationService = escalationService;
    }

    @Test
    void upgrade_grade1To3_callsStartEscalation() {
        var event = event(CtcaeGrade.GRADE_1, CtcaeGrade.GRADE_3);
        listener.onGradeChanged(event);
        verify(escalationService).startEscalationForRegrade(
            event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(), event.tenantId());
    }

    @Test
    void upgrade_grade3To4_callsStartEscalation() {
        var event = event(CtcaeGrade.GRADE_3, CtcaeGrade.GRADE_4);
        listener.onGradeChanged(event);
        verify(escalationService).startEscalationForRegrade(
            event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(), event.tenantId());
    }

    @Test
    void downgrade_grade3To1_skips() {
        var event = event(CtcaeGrade.GRADE_3, CtcaeGrade.GRADE_1);
        listener.onGradeChanged(event);
        verifyNoInteractions(escalationService);
    }

    @Test
    void sameGrade_skips() {
        var event = event(CtcaeGrade.GRADE_2, CtcaeGrade.GRADE_2);
        listener.onGradeChanged(event);
        verifyNoInteractions(escalationService);
    }

    private AeGradeChangedEvent event(CtcaeGrade prev, CtcaeGrade next) {
        return new AeGradeChangedEvent(UUID.randomUUID(), UUID.randomUUID(),
            UUID.randomUUID(), prev, next, Instant.now(), "dr-test", "default");
    }
}
```

- [ ] **Step 2: Write SUSAR, regulatory, trajectory, and safety officer listener tests**

Each follows the same pattern as the escalation listener test — verify the listener gates on `isUpgrade()` and delegates to the correct service method. The trajectory listener should fire on both upgrades and downgrades.

Create similar test classes:
- `AeGradeChangeSusarListenerTest` — verify calls `susarOversightCaseService.reevaluateForRegrade()` on upgrade, skips on downgrade
- `AeGradeChangeRegulatoryListenerTest` — verify calls `regulatorySubmissionCaseService.reevaluateForRegrade()` on upgrade, skips on downgrade
- `AeGradeChangeTrajectoryListenerTest` — verify calls `aeTrajectoryAlertService.evaluate()` on both upgrade and downgrade
- `AeGradeChangeSafetyOfficerListenerTest` — verify calls `safetyOfficerNotificationListener.onGradeUpgrade()` on upgrade, skips on downgrade

- [ ] **Step 3: Run tests — verify they fail**

```bash
mvn test -pl runtime -Dtest="AeGradeChange*ListenerTest" --batch-mode
```
Expected: compilation failures.

- [ ] **Step 4: Add startEscalationForRegrade() to AeEscalationCaseService**

Add a new public method to `AeEscalationCaseService.java` that follows the same three-phase pattern as `onAdverseEventReported()` but accepts regrade parameters directly:

```java
public void startEscalationForRegrade(UUID aeId, UUID enrollmentId, UUID siteId,
                                       CtcaeGrade grade, String tenantId) {
    try {
        Map<String, Object> initialContext = prepareAndMarkForRegrade(aeId, enrollmentId, siteId, grade, tenantId);
        if (initialContext == null) return;
        UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().join();
        persistCaseId(aeId, caseId);
        try { aeTrajectoryAlertService.evaluate(aeId, tenantId); } catch (Exception te) { LOG.warnf(te, "Trajectory alert re-evaluation failed for aeId=%s", aeId); }
        if (SEVERE_GRADES.contains(grade)) {
            trialSafetySignalService.signalGrade4Active(siteId);
        }
    } catch (Exception e) {
        LOG.errorf(e, "AeEscalationCaseService: regrade escalation failed for aeId=%s — marking FAILED", aeId);
        try { markFailed(aeId); } catch (Exception ex) {
            LOG.errorf(ex, "AeEscalationCaseService: markFailed also failed for aeId=%s", aeId);
        }
    }
}
```

Add the `prepareAndMarkForRegrade` helper (same logic as `prepareAndMarkRequested` but with direct params):

```java
@Transactional
Map<String, Object> prepareAndMarkForRegrade(UUID aeId, UUID enrollmentId, UUID siteId,
                                              CtcaeGrade grade, String tenantId) {
    AdverseEvent ae = AdverseEvent.findById(aeId);
    if (ae == null) { LOG.warnf("AE not found for regrade escalation aeId=%s", aeId); return null; }
    if (ae.engineCaseId != null) {
        if (SEVERE_GRADES.contains(grade)) {
            trialSafetySignalService.signalGrade4Active(siteId);
        }
        return null;
    }
    if (ae.workItemId != null) {
        LOG.infof("Cancelling Grade 1/2 WorkItem %s for aeId=%s — engine case taking over", ae.workItemId, aeId);
        ae.workItemId = null;
    }
    ae.escalationStatus = AeEscalationStatus.REQUESTED;

    AdverseEventEscalationRequirements requirements = policy.evaluate(
        new AdverseEventContext(aeId, enrollmentId, siteId, grade));

    Map<String, Object> ctx = new HashMap<>();
    ctx.put("aeId", aeId.toString());
    ctx.put("enrollmentId", enrollmentId.toString());
    ctx.put("siteId", siteId != null ? siteId.toString() : "");
    ctx.put("grade", grade.name());
    ctx.put("requiresSeniorMonitor", requirements.requiresSeniorMonitor());
    ctx.put("requiresDsmbEscalation", requirements.requiresDsmbEscalation());
    ctx.put("tenantId", tenantId);
    var patientCtx = memoryService.queryPatientContext(enrollmentId, tenantId);
    var siteCtx = siteId != null ? memoryService.querySiteContext(siteId, tenantId) : null;
    ctx.put("patientContext", patientCtx.toContextMap());
    if (siteCtx != null) ctx.put("siteContext", siteCtx.toContextMap());
    ctx.put("unexpected", ae.unexpected);
    ctx.put("suspected", ae.suspected);

    var plan = planRetriever.retrieve(ae);
    if (plan.hasRecommendation()) {
        ctx.put("escalationPlanRecommendation", plan.toContextMap());
    }
    return ctx;
}
```

- [ ] **Step 5: Add reevaluateForRegrade() to SusarOversightCaseService**

```java
public void reevaluateForRegrade(UUID aeId, UUID siteId, String tenantId) {
    AdverseEvent ae = AdverseEvent.findById(aeId);
    if (ae == null) return;
    if (ae.susarOversightStatus != SusarOversightStatus.NONE) return;
    if (!ae.unexpected || !ae.suspected) return;

    var event = new AdverseEventReportedEvent(
        aeId, ae.enrollmentId, siteId, ae.grade, ae.reportedAt, tenantId);
    Map<String, Object> ctx = prepareAndMark(event);
    if (ctx == null) return;
    try {
        UUID caseId = susarOversightCaseHub.startCase(ctx).toCompletableFuture().join();
        persistCaseId(aeId, caseId);
    } catch (Exception e) {
        LOG.errorf(e, "SUSAR regrade evaluation failed for aeId=%s", aeId);
        markFailed(aeId);
    }
}
```

- [ ] **Step 6: Add reevaluateForRegrade() to RegulatorySubmissionCaseService**

```java
public void reevaluateForRegrade(UUID aeId, UUID siteId, String tenantId) {
    AdverseEvent ae = AdverseEvent.findById(aeId);
    if (ae == null) return;
    if (ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.NONE) return;
    if (!isIndReportable(ae.grade) || !ae.unexpected) return;

    var event = new AdverseEventReportedEvent(
        aeId, ae.enrollmentId, siteId, ae.grade, ae.reportedAt, tenantId);
    Map<String, Object> ctx = prepareAndMark(event);
    if (ctx == null) return;
    try {
        UUID caseId = regulatorySubmissionCaseHub.startCase(ctx).toCompletableFuture().join();
        persistCaseId(aeId, caseId);
    } catch (Exception e) {
        LOG.errorf(e, "Regulatory regrade evaluation failed for aeId=%s", aeId);
        resetToNone(aeId);
    }
}
```

- [ ] **Step 7: Create all five listener classes**

Each is `@ApplicationScoped` with `@ObservesAsync AeGradeChangedEvent`:

**AeGradeChangeEscalationListener:**
```java
@ApplicationScoped
public class AeGradeChangeEscalationListener {
    @Inject AeEscalationCaseService escalationService;

    public void onGradeChanged(@ObservesAsync AeGradeChangedEvent event) {
        if (!event.isUpgrade()) return;
        escalationService.startEscalationForRegrade(
            event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(), event.tenantId());
    }
}
```

**AeGradeChangeSusarListener:**
```java
@ApplicationScoped
public class AeGradeChangeSusarListener {
    @Inject SusarOversightCaseService susarOversightCaseService;

    public void onGradeChanged(@ObservesAsync AeGradeChangedEvent event) {
        if (!event.isUpgrade()) return;
        susarOversightCaseService.reevaluateForRegrade(event.aeId(), event.siteId(), event.tenantId());
    }
}
```

**AeGradeChangeRegulatoryListener:**
```java
@ApplicationScoped
public class AeGradeChangeRegulatoryListener {
    @Inject RegulatorySubmissionCaseService regulatorySubmissionCaseService;

    public void onGradeChanged(@ObservesAsync AeGradeChangedEvent event) {
        if (!event.isUpgrade()) return;
        regulatorySubmissionCaseService.reevaluateForRegrade(event.aeId(), event.siteId(), event.tenantId());
    }
}
```

**AeGradeChangeTrajectoryListener:**
```java
@ApplicationScoped
public class AeGradeChangeTrajectoryListener {
    @Inject AeTrajectoryAlertService aeTrajectoryAlertService;

    public void onGradeChanged(@ObservesAsync AeGradeChangedEvent event) {
        try {
            aeTrajectoryAlertService.evaluate(event.aeId(), event.tenantId());
        } catch (Exception e) {
            org.jboss.logging.Logger.getLogger(AeGradeChangeTrajectoryListener.class)
                .warnf(e, "Trajectory alert re-evaluation failed for aeId=%s", event.aeId());
        }
    }
}
```

**AeGradeChangeSafetyOfficerListener:**
```java
@ApplicationScoped
public class AeGradeChangeSafetyOfficerListener {
    @Inject SafetyOfficerNotifier notifier;
    @Inject SafetyOfficerNotificationLedgerWriter ledgerWriter;

    @Transactional
    public void onGradeChanged(@ObservesAsync AeGradeChangedEvent event) {
        if (!event.isUpgrade()) return;
        try {
            if (event.siteId() == null) {
                ledgerWriter.writeSkippedEntry(event.aeId(), event.enrollmentId(), null,
                    event.newGrade(), "safety-officer-regrade-skipped-no-site-id");
                return;
            }
            TrialSite site = TrialSite.findById(event.siteId());
            if (site == null) {
                ledgerWriter.writeSkippedEntry(event.aeId(), event.enrollmentId(), event.siteId(),
                    event.newGrade(), "safety-officer-regrade-skipped-site-not-found");
                return;
            }
            ClinicalTrial trial = ClinicalTrial.findById(site.trialId);
            if (trial == null || trial.safetyOfficerConnectorId == null || trial.safetyOfficerDestination == null) {
                ledgerWriter.writeSkippedEntry(event.aeId(), event.enrollmentId(), event.siteId(),
                    event.newGrade(), "safety-officer-regrade-skipped-no-config");
                return;
            }
            notifier.notify(new SafetyOfficerNotificationRequest(
                event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(),
                trial.safetyOfficerConnectorId, trial.safetyOfficerDestination));
        } catch (Exception e) {
            org.jboss.logging.Logger.getLogger(AeGradeChangeSafetyOfficerListener.class)
                .errorf(e, "Safety officer regrade notification failed for AE %s", event.aeId());
            try {
                ledgerWriter.writeObserverFailureEntry(event.aeId(), event.enrollmentId(),
                    event.siteId(), event.newGrade());
            } catch (Exception writeEx) {
                org.jboss.logging.Logger.getLogger(AeGradeChangeSafetyOfficerListener.class)
                    .errorf(writeEx, "AUDIT GAP: could not write failure entry for AE %s", event.aeId());
            }
        }
    }
}
```

- [ ] **Step 8: Run all listener tests — verify green**

```bash
mvn test -pl runtime -Dtest="AeGradeChange*ListenerTest" --batch-mode
```
Expected: all pass.

- [ ] **Step 9: Commit**

```
feat(#135): five AeGradeChanged upgrade re-evaluation listeners

Refs #135
```

---

### Task 6: Trajectory Integration — grade dimension in AeTrajectoryBuilder + schema

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/AeTrajectoryBuilder.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCbrSchemaInitializer.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/cbr/AeTrajectoryBuilderTest.java`

**Interfaces:**
- Consumes: `AeGradeChange.findByAdverseEventId()` (Task 1)
- Produces: `AeTrajectoryBuilder` output now includes `grade` dimension in time series
- Produces: `aeTrajectorySchema()` includes `grade` inner field with `TrendSpec`

- [ ] **Step 1: Write trajectory builder test with grade changes**

Add tests to `AeTrajectoryBuilderTest.java`:

```java
@Test
void buildTrajectory_withGradeChanges_includesGradeDimension() {
    AdverseEvent ae = buildAe(CtcaeGrade.GRADE_3);
    ae.reportedAt = Instant.parse("2026-01-01T00:00:00Z");

    // Insert grade history: initial GRADE_1, then upgraded to GRADE_3
    AeGradeChange initial = gradeChange(ae.id, null, CtcaeGrade.GRADE_1, ae.reportedAt);
    AeGradeChange upgrade = gradeChange(ae.id, CtcaeGrade.GRADE_1, CtcaeGrade.GRADE_3,
        ae.reportedAt.plus(Duration.ofHours(48)));

    mockGradeHistory(ae.id, List.of(initial, upgrade));

    var trajectory = builder.buildTrajectory(ae, "default");
    assertFalse(trajectory.isEmpty());

    // First observation should have grade=1
    assertEquals(1.0, trajectory.get(0).get("grade").numberValue());

    // After grade change, should have grade=3
    var lastObs = trajectory.get(trajectory.size() - 1);
    assertEquals(3.0, lastObs.get("grade").numberValue());
}

@Test
void buildTrajectory_noGradeHistory_usesCurrentGrade() {
    AdverseEvent ae = buildAe(CtcaeGrade.GRADE_2);
    ae.reportedAt = Instant.now();
    mockGradeHistory(ae.id, List.of());

    var trajectory = builder.buildTrajectory(ae, "default");
    assertFalse(trajectory.isEmpty());
    assertEquals(2.0, trajectory.get(0).get("grade").numberValue());
}
```

Helper methods to add:
```java
private AeGradeChange gradeChange(UUID aeId, CtcaeGrade prev, CtcaeGrade next, Instant at) {
    AeGradeChange gc = new AeGradeChange();
    gc.id = UUID.randomUUID();
    gc.adverseEventId = aeId;
    gc.previousGrade = prev;
    gc.newGrade = next;
    gc.changedAt = at;
    gc.changedBy = "test";
    return gc;
}
```

The `mockGradeHistory` will need the test to mock or stub the Panache static query. Since `AeTrajectoryBuilder` calls `AeGradeChange.findByAdverseEventId()`, and this is a Panache static method, the builder needs a `Function<UUID, List<AeGradeChange>>` setter (same pattern as `AeTrajectoryAlertService.setEntityFinder()`). Add `setGradeHistoryFinder(Function<UUID, List<AeGradeChange>> finder)` to `AeTrajectoryBuilder`.

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=AeTrajectoryBuilderTest --batch-mode
```
Expected: new tests fail — grade dimension not in output.

- [ ] **Step 3: Update AeTrajectoryBuilder — add grade to Observation**

Modify `Observation` to include `grade`:

```java
private static class Observation {
    long secondsSinceReport;
    int escalation;
    int susar;
    int regulatory;
    int grade;

    Observation(long secondsSinceReport, int escalation, int susar, int regulatory, int grade) {
        this.secondsSinceReport = secondsSinceReport;
        this.escalation = escalation;
        this.susar = susar;
        this.regulatory = regulatory;
        this.grade = grade;
    }

    Map<String, FeatureValue> toFeatureMap() {
        return Map.of(
            "ts", FeatureValue.number(secondsSinceReport),
            "escalation", FeatureValue.number(escalation),
            "susar", FeatureValue.number(susar),
            "regulatory", FeatureValue.number(regulatory),
            "grade", FeatureValue.number(grade));
    }
}
```

- [ ] **Step 4: Update doBuild() — sorted merge with grade changes**

Replace `doBuild()` to query grade history and merge with plan item records:

1. Add a field and setter for grade history lookup:
```java
private Function<UUID, List<AeGradeChange>> gradeHistoryFinder = AeGradeChange::findByAdverseEventId;

void setGradeHistoryFinder(Function<UUID, List<AeGradeChange>> finder) {
    this.gradeHistoryFinder = finder;
}
```

2. Rewrite `doBuild()` to do the sorted merge as specified in the design spec §Trajectory Integration.

The key changes:
- Query `gradeHistoryFinder.apply(ae.id)` for grade changes
- Initialize `currentGrade` from the first grade change record (or `ae.grade.ordinal() + 1` if no history)
- Create a merged timeline from grade changes and plan item records, sorted by timestamp
- Each observation carries the current value of all five dimensions
- Grade changes update `currentGrade`; plan items update escalation/susar/regulatory

- [ ] **Step 5: Update ClinicalCbrSchemaInitializer — add grade inner field**

In `aeTrajectorySchema()`, add `FeatureField.numeric("grade", 1, 5)` as the last inner field of the `timeSeries`:

```java
FeatureField.timeSeries("aeTrajectory", "ts",
    new SimilaritySpec.DtwSpec(new WarpingConstraint.SakoeChibaBand(3)),
    new TrendSpec(Set.of(TrendType.SLOPE, TrendType.ACCELERATION, TrendType.CHANGE_POINTS), ChronoUnit.HOURS),
    FeatureField.numeric("ts", 0, 7776000),
    FeatureField.numeric("escalation", 0, 3),
    FeatureField.numeric("susar", 0, 3),
    FeatureField.numeric("regulatory", 0, 3),
    FeatureField.numeric("grade", 1, 5))
```

- [ ] **Step 6: Run trajectory tests — verify green**

```bash
mvn test -pl runtime -Dtest=AeTrajectoryBuilderTest --batch-mode
```
Expected: all pass, including new grade tests and existing tests.

- [ ] **Step 7: Run full test suite — verify no regressions**

```bash
mvn test --batch-mode
```
Expected: all pass. Check for any tests that assert exact trajectory output shape (they now have 5 fields instead of 4).

- [ ] **Step 8: Commit**

```
feat(#135): grade dimension in AE trajectory builder + CBR schema

Refs #135
```

---

### Task 7: Dashboard — grade history in TrialDashboardResource

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialDashboardResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/TrialDashboardResourceTest.java`

**Interfaces:**
- Consumes: `AeGradeChange.findByAdverseEventId()` (Task 1)
- Produces: `AdverseEventRow` gains a `gradeHistory` field

- [ ] **Step 1: Add grade history to AdverseEventRow**

In `TrialDashboardResource`, add a `gradeHistory` field to the `AdverseEventRow` record:

```java
record GradeChangeRow(String previousGrade, String newGrade, Instant changedAt, String changedBy) {}
```

Add `List<GradeChangeRow> gradeHistory` to `AdverseEventRow`.

- [ ] **Step 2: Populate gradeHistory in the adverseEvents() method**

Where `AdverseEventRow` is constructed, query `AeGradeChange.findByAdverseEventId(ae.id)` and map to `GradeChangeRow` list.

- [ ] **Step 3: Add a test for gradeHistory in dashboard response**

In `TrialDashboardResourceTest`, add a test that creates an AE with grade history entries and verifies the dashboard response includes them.

- [ ] **Step 4: Run test — verify green**

```bash
mvn test -pl runtime -Dtest=TrialDashboardResourceTest --batch-mode
```

- [ ] **Step 5: Commit**

```
feat(#135): grade history in trial dashboard AE listing

Refs #135
```

---

## Post-Implementation

After all 7 tasks pass:

1. Run `mvn test --batch-mode` — full green
2. Run `ide_diagnostics` on modified files — no errors
3. Invoke `work-end` — handles code review, squash, merge, push
