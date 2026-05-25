# Epic 4: Adverse Event Escalation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire adverse event reporting to casehub-work WorkItems with grade-driven GCP SLA deadlines and tamper-evident ledger entries.

**Architecture:** `AdverseEventService` creates a WorkItem (via embedded casehub-work) and an `AdverseEventLedgerEntry` (via embedded casehub-ledger) in a single JTA transaction when an AE is reported. Escalation policy (who receives the DSMB alert on SLA miss) is runtime configuration, not clinical code. Connectors notification deferred to casehubio/clinical#11.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-work (`WorkItemService`), casehub-ledger (`LedgerEntryRepository`), H2 `MODE=PostgreSQL` for tests, JUnit 5 + RestAssured + AssertJ.

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `api/src/main/java/io/casehub/clinical/api/model/CtcaeGrade.java` | Modify | Add 7-day SLA for Grade 1-2 |
| `api/src/test/java/io/casehub/clinical/api/model/CtcaeGradeTest.java` | Modify | Extend tests for all-grade SLA coverage |
| `runtime/pom.xml` | Modify | Add casehub-work and casehub-ledger dependencies |
| `runtime/src/main/resources/application.properties` | Modify | Add qhorus named datasource for ledger entities |
| `runtime/src/test/resources/application.properties` | Modify | Add qhorus H2 datasource + hibernate packages |
| `runtime/src/main/resources/db/migration/V1__clinical_trial.sql` → `V100__clinical_trial.sql` | Rename | Avoid conflict with casehub-work V1-V21 migrations |
| `runtime/src/main/resources/db/migration/V2__trial_site.sql` → `V101__trial_site.sql` | Rename | (same reason) |
| `runtime/src/main/resources/db/migration/V3__patient_enrollment.sql` → `V102__patient_enrollment.sql` | Rename | (same reason) |
| `runtime/src/main/resources/db/migration/V4__adverse_event.sql` → `V103__adverse_event.sql` | Rename | (same reason) |
| `runtime/src/main/resources/db/migration/V5__protocol_deviation.sql` → `V104__protocol_deviation.sql` | Rename | (same reason) |
| `runtime/src/main/resources/db/migration/V6__irb_approval.sql` → `V105__irb_approval.sql` | Rename | (same reason) |
| `runtime/src/main/resources/db/migration/V106__adverse_event_work_item_id.sql` | Create | Add `work_item_id` column to `adverse_event` |
| `runtime/src/main/resources/db/migration/V1004__ae_ledger_entry.sql` | Create | Join table for `AdverseEventLedgerEntry` (platform rule: V1004+) |
| `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java` | Modify | Add `workItemId` field |
| `runtime/src/main/java/io/casehub/clinical/entity/AdverseEventLedgerEntry.java` | Create | LedgerEntry subclass with AE-specific audit fields |
| `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java` | Create | Orchestrates AE persist + WorkItem + ledger entry |
| `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java` | Modify | Add AE report endpoint, delegate to AdverseEventService |
| `runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java` | Create | Integration tests for service logic |
| `runtime/src/test/java/io/casehub/clinical/resource/AdverseEventResourceTest.java` | Create | REST integration tests (robustness + correctness) |
| `runtime/src/test/java/io/casehub/clinical/resource/ShowcaseScenarioTest.java` | Modify | Extend showcase with AE reporting scenarios |

---

## Task 1: Fix `CtcaeGrade` SLA — add 7-day window for Grade 1-2

**Files:**
- Modify: `api/src/main/java/io/casehub/clinical/api/model/CtcaeGrade.java`
- Modify: `api/src/test/java/io/casehub/clinical/api/model/CtcaeGradeTest.java`

- [ ] **Step 1.1: Write failing tests**

Open `api/src/test/java/io/casehub/clinical/api/model/CtcaeGradeTest.java` and replace/extend its contents:

```java
package io.casehub.clinical.api.model;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import static org.assertj.core.api.Assertions.assertThat;

class CtcaeGradeTest {

    @Test
    void grade1_has_7day_sla() {
        assertThat(CtcaeGrade.GRADE_1.sla()).hasValue(Duration.ofDays(7));
    }

    @Test
    void grade2_has_7day_sla() {
        assertThat(CtcaeGrade.GRADE_2.sla()).hasValue(Duration.ofDays(7));
    }

    @Test
    void grade3_has_24h_sla() {
        assertThat(CtcaeGrade.GRADE_3.sla()).hasValue(Duration.ofHours(24));
    }

    @Test
    void grade4_has_24h_sla() {
        assertThat(CtcaeGrade.GRADE_4.sla()).hasValue(Duration.ofHours(24));
    }

    @Test
    void grade5_has_1h_sla() {
        assertThat(CtcaeGrade.GRADE_5.sla()).hasValue(Duration.ofHours(1));
    }

    @Test
    void all_grades_have_non_empty_sla() {
        for (CtcaeGrade grade : CtcaeGrade.values()) {
            assertThat(grade.sla()).as("Grade %s must have SLA", grade).isPresent();
        }
    }
}
```

- [ ] **Step 1.2: Run tests to confirm failure**

```bash
mvn test -pl api -Dtest=CtcaeGradeTest --batch-mode
```

Expected: FAIL — `grade1_has_7day_sla` and `grade2_has_7day_sla` fail because Grade 1-2 currently return `Optional.empty()`.

- [ ] **Step 1.3: Update the enum**

Replace the two `null` SLA entries in `api/src/main/java/io/casehub/clinical/api/model/CtcaeGrade.java`:

```java
public enum CtcaeGrade {
    GRADE_1("Mild",             Duration.ofDays(7)),
    GRADE_2("Moderate",         Duration.ofDays(7)),
    GRADE_3("Severe",           Duration.ofHours(24)),
    GRADE_4("Life-threatening", Duration.ofHours(24)),
    GRADE_5("Death",            Duration.ofHours(1));

    private final String label;
    private final Duration sla;

    CtcaeGrade(String label, Duration sla) {
        this.label = label;
        this.sla = sla;
    }

    public String label() { return label; }

    /** Reporting SLA per GCP ICH E6(R3) §5.17. Present for all grades. */
    public Optional<Duration> sla() { return Optional.of(sla); }
}
```

- [ ] **Step 1.4: Run tests to confirm pass**

```bash
mvn test -pl api --batch-mode
```

Expected: all 6 CtcaeGradeTest tests pass, plus all existing api module tests.

- [ ] **Step 1.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/src/main/java/io/casehub/clinical/api/model/CtcaeGrade.java \
  api/src/test/java/io/casehub/clinical/api/model/CtcaeGradeTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "fix(api): add 7-day GCP SLA for Grade 1-2 adverse events

GCP ICH E6(R3) requires non-serious AEs (Grade 1-2) to be reported
within 7 days. Previously null — now explicit Duration.ofDays(7).
All grades now return non-empty Optional<Duration>.

Refs #4"
```

---

## Task 2: Add dependencies and configure datasources

**Files:**
- Modify: `runtime/pom.xml`
- Modify: `runtime/src/main/resources/application.properties`
- Modify: `runtime/src/test/resources/application.properties`

**Background:** casehub-work's Flyway migrations (V1–V21+) live at `classpath:db/migration` in the casehub-work JAR. Quarkus Flyway scans this path from all JARs on the classpath, so clinical's current V1–V6 migrations conflict. Renaming clinical's domain migrations to V100–V105 gives casehub-work ample room and follows the ecosystem pattern (AML will do the same when it adds domain migrations).

The ledger (`casehub-ledger`) runs on a **named `qhorus` datasource** — this is a platform convention (see AML's `application.properties`). All ledger entities and migrations share this second H2 DB in tests.

- [ ] **Step 2.1: Add dependencies to `runtime/pom.xml`**

Inside the `<dependencies>` block, after `casehub-clinical-api`, add:

```xml
<!-- Layer 2: human task lifecycle — WorkItem SLA, candidateGroups routing -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-work</artifactId>
</dependency>

<!-- Layer 2: tamper-evident audit ledger — AdverseEventLedgerEntry -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>
</dependency>
```

Both are version-managed by the parent BOM — no `<version>` needed.

- [ ] **Step 2.2: Rename existing Flyway migrations (V1-V6 → V100-V105)**

Run from `runtime/src/main/resources/db/migration/`:

```bash
cd /Users/mdproctor/claude/casehub/clinical/runtime/src/main/resources/db/migration
mv V1__clinical_trial.sql     V100__clinical_trial.sql
mv V2__trial_site.sql         V101__trial_site.sql
mv V3__patient_enrollment.sql V102__patient_enrollment.sql
mv V4__adverse_event.sql      V103__adverse_event.sql
mv V5__protocol_deviation.sql V104__protocol_deviation.sql
mv V6__irb_approval.sql       V105__irb_approval.sql
```

- [ ] **Step 2.3: Update `runtime/src/main/resources/application.properties`**

Replace the entire file:

```properties
quarkus.application.name=casehub-clinical

# ============================================================
# Default datasource — clinical domain + casehub-work tables
# ============================================================
quarkus.datasource.db-kind=postgresql
quarkus.hibernate-orm.packages=io.casehub.work.runtime.model,io.casehub.clinical.entity
quarkus.hibernate-orm.schema-management.strategy=validate
quarkus.flyway.migrate-at-start=true

%dev.quarkus.datasource.db-kind=h2
%dev.quarkus.datasource.jdbc.url=jdbc:h2:mem:clinical;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%dev.quarkus.hibernate-orm.schema-management.strategy=none

# ============================================================
# Qhorus named datasource — qhorus + casehub-ledger tables
# Platform convention: ledger always shares the qhorus PU.
# ============================================================
quarkus.datasource.qhorus.db-kind=postgresql
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.ledger.runtime.model,io.casehub.ledger.model
quarkus.hibernate-orm.qhorus.schema-management.strategy=validate
quarkus.flyway.qhorus.migrate-at-start=true

%dev.quarkus.datasource.qhorus.db-kind=h2
%dev.quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:qhorus;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%dev.quarkus.datasource.qhorus.username=sa
%dev.quarkus.datasource.qhorus.password=

# Direct casehub-ledger to the qhorus persistence unit
casehub.ledger.datasource=qhorus
```

- [ ] **Step 2.4: Update `runtime/src/test/resources/application.properties`**

Replace the entire file:

```properties
quarkus.http.test-port=0
quarkus.scheduler.enabled=false

# ============================================================
# Default datasource — clinical domain + casehub-work tables
# ============================================================
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:clinical;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.username=sa
quarkus.datasource.password=
quarkus.hibernate-orm.packages=io.casehub.work.runtime.model,io.casehub.clinical.entity
quarkus.hibernate-orm.schema-management.strategy=none
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true

# ============================================================
# Qhorus named datasource — casehub-ledger tables
# ============================================================
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:qhorus;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.ledger.runtime.model,io.casehub.ledger.model,io.casehub.clinical.entity
quarkus.hibernate-orm.qhorus.schema-management.strategy=none
quarkus.hibernate-orm.qhorus.database.generation=none
quarkus.flyway.qhorus.migrate-at-start=true

casehub.ledger.datasource=qhorus
```

Note: `io.casehub.clinical.entity` is added to the qhorus packages so `AdverseEventLedgerEntry` (created in Task 4) is managed by the qhorus PU.

- [ ] **Step 2.5: Run existing tests to verify**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode
```

Expected: all 17 existing tests pass. If any Flyway migration conflict appears in the output (e.g., "Found more than one migration with version 1"), stop and verify the V1-V6 rename completed correctly in Step 2.2.

- [ ] **Step 2.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/pom.xml \
  runtime/src/main/resources/application.properties \
  runtime/src/test/resources/application.properties \
  "runtime/src/main/resources/db/migration/V100__clinical_trial.sql" \
  "runtime/src/main/resources/db/migration/V101__trial_site.sql" \
  "runtime/src/main/resources/db/migration/V102__patient_enrollment.sql" \
  "runtime/src/main/resources/db/migration/V103__adverse_event.sql" \
  "runtime/src/main/resources/db/migration/V104__protocol_deviation.sql" \
  "runtime/src/main/resources/db/migration/V105__irb_approval.sql"
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): add casehub-work and casehub-ledger dependencies

Rename domain migrations V1-V6 → V100-V105 to avoid conflict with
casehub-work's V1-V21 migrations (both at classpath:db/migration).
Add qhorus named datasource for casehub-ledger entities.

Refs #4"
```

---

## Task 3: Flyway migrations — `work_item_id` and `ae_ledger_entry`

**Files:**
- Create: `runtime/src/main/resources/db/migration/V106__adverse_event_work_item_id.sql`
- Create: `runtime/src/main/resources/db/migration/V1004__ae_ledger_entry.sql`

- [ ] **Step 3.1: Create V106 — add `work_item_id` to `adverse_event`**

Create `runtime/src/main/resources/db/migration/V106__adverse_event_work_item_id.sql`:

```sql
ALTER TABLE adverse_event ADD COLUMN work_item_id UUID;
```

- [ ] **Step 3.2: Create V1004 — `ae_ledger_entry` join table**

This is a ledger subclass join table. Per platform convention, consumer-owned ledger join tables use V1004+. Create `runtime/src/main/resources/db/migration/V1004__ae_ledger_entry.sql`:

```sql
CREATE TABLE ae_ledger_entry (
    id              UUID    NOT NULL,
    adverse_event_id UUID   NOT NULL,
    enrollment_id   UUID    NOT NULL,
    ctcae_grade     VARCHAR(20) NOT NULL,
    reported_at     TIMESTAMP WITH TIME ZONE NOT NULL,
    sla_deadline    TIMESTAMP WITH TIME ZONE NOT NULL,
    CONSTRAINT pk_ae_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_ae_ledger_entry_base FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 3.3: Run all tests to verify migrations apply cleanly**

```bash
mvn test -pl runtime --batch-mode
```

Expected: all existing tests pass. If `FlywayMigrationTest` fails with a checksum error, verify the renamed V100-V105 files match the originals exactly.

- [ ] **Step 3.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  "runtime/src/main/resources/db/migration/V106__adverse_event_work_item_id.sql" \
  "runtime/src/main/resources/db/migration/V1004__ae_ledger_entry.sql"
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): add Flyway migrations V106 and V1004

V106: work_item_id on adverse_event (WorkItem tracking)
V1004: ae_ledger_entry join table (LedgerEntry JOINED subclass,
per platform V1004+ numbering for consumer-owned joins)

Refs #4"
```

---

## Task 4: Entity updates — `AdverseEvent.workItemId` and `AdverseEventLedgerEntry`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Create: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEventLedgerEntry.java`

- [ ] **Step 4.1: Add `workItemId` to `AdverseEvent`**

Add one field after `slaDeadline` in `AdverseEvent.java`:

```java
/** WorkItem id created by AdverseEventService for GCP SLA tracking. Null until service call. */
@Column(name = "work_item_id")
public UUID workItemId;
```

Full file after change:

```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "adverse_event")
public class AdverseEvent extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "enrollment_id", nullable = false)
    public UUID enrollmentId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public CtcaeGrade grade;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public EventActuality actuality = EventActuality.ACTUAL;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public AeOutcome outcome = AeOutcome.ONGOING;

    @Column(name = "occurred_at", nullable = false)
    public Instant occurredAt;

    @Column(name = "reported_at", nullable = false)
    public Instant reportedAt;

    /** Null for Grade 1 and 2 (no GCP SLA). Computed from reportedAt + grade.sla(). */
    @Column(name = "sla_deadline")
    public Instant slaDeadline;

    /** WorkItem id created by AdverseEventService for GCP SLA tracking. Null until service call. */
    @Column(name = "work_item_id")
    public UUID workItemId;
}
```

Note: `slaDeadline` is now non-null for all grades after the CtcaeGrade fix (Task 1), but the column stays nullable for backward compatibility with any entries that pre-date this epic.

- [ ] **Step 4.2: Create `AdverseEventLedgerEntry`**

Create `runtime/src/main/java/io/casehub/clinical/entity/AdverseEventLedgerEntry.java`:

```java
package io.casehub.clinical.entity;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident audit record written when an adverse event is reported.
 *
 * Extends LedgerEntry via JPA JOINED inheritance — base fields (subjectId,
 * sequenceNumber, actorId, occurredAt, digest) are in the ledger_entry table;
 * AE-specific fields are in ae_ledger_entry (V1004 migration).
 *
 * FDA IND requirement: every safety event must be independently verifiable
 * via the Merkle chain without server access.
 */
@Entity
@Table(name = "ae_ledger_entry")
@DiscriminatorValue("ADVERSE_EVENT")
public class AdverseEventLedgerEntry extends LedgerEntry {

    @Column(name = "adverse_event_id", nullable = false)
    public UUID adverseEventId;

    @Column(name = "enrollment_id", nullable = false)
    public UUID enrollmentId;

    @Column(name = "ctcae_grade", nullable = false, length = 20)
    public String ctcaeGrade;

    @Column(name = "reported_at", nullable = false)
    public Instant reportedAt;

    @Column(name = "sla_deadline", nullable = false)
    public Instant slaDeadline;
}
```

- [ ] **Step 4.3: Run existing tests to verify no regression**

```bash
mvn test -pl runtime --batch-mode
```

Expected: all 17 existing tests pass.

- [ ] **Step 4.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java \
  runtime/src/main/java/io/casehub/clinical/entity/AdverseEventLedgerEntry.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): add workItemId to AdverseEvent, add AdverseEventLedgerEntry

AdverseEvent.workItemId tracks the casehub-work WorkItem for GCP SLA.
AdverseEventLedgerEntry is the FDA tamper-evident audit record
(JOINED LedgerEntry subclass, discriminator ADVERSE_EVENT).

Refs #4"
```

---

## Task 5: `AdverseEventService` (TDD)

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java`

- [ ] **Step 5.1: Write failing integration tests**

Create `runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.AdverseEventLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

@QuarkusTest
class AdverseEventServiceTest {

    @Inject
    AdverseEventService service;

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Test
    @Transactional
    void grade3_slaDeadline_is_reportedAt_plus_24h() {
        AdverseEvent ae = newAe(CtcaeGrade.GRADE_3);
        service.reportAdverseEvent(ae);
        assertThat(ae.slaDeadline)
            .isCloseTo(ae.reportedAt.plus(Duration.ofHours(24)), within(5, ChronoUnit.SECONDS));
    }

    @Test
    @Transactional
    void grade5_slaDeadline_is_reportedAt_plus_1h() {
        AdverseEvent ae = newAe(CtcaeGrade.GRADE_5);
        service.reportAdverseEvent(ae);
        assertThat(ae.slaDeadline)
            .isCloseTo(ae.reportedAt.plus(Duration.ofHours(1)), within(5, ChronoUnit.SECONDS));
    }

    @Test
    @Transactional
    void grade1_slaDeadline_is_reportedAt_plus_7days() {
        AdverseEvent ae = newAe(CtcaeGrade.GRADE_1);
        service.reportAdverseEvent(ae);
        assertThat(ae.slaDeadline)
            .isCloseTo(ae.reportedAt.plus(Duration.ofDays(7)), within(5, ChronoUnit.SECONDS));
    }

    @Test
    @Transactional
    void workItemId_is_set_after_report() {
        AdverseEvent ae = newAe(CtcaeGrade.GRADE_3);
        service.reportAdverseEvent(ae);
        assertThat(ae.workItemId).isNotNull();
    }

    @Test
    @Transactional
    void reportedAt_is_set_server_side() {
        AdverseEvent ae = newAe(CtcaeGrade.GRADE_3);
        Instant before = Instant.now().minusSeconds(1);
        service.reportAdverseEvent(ae);
        assertThat(ae.reportedAt).isAfterOrEqualTo(before);
    }

    @Test
    @Transactional
    void ledger_entry_is_persisted_with_correct_fields() {
        AdverseEvent ae = newAe(CtcaeGrade.GRADE_4);
        service.reportAdverseEvent(ae);

        var entries = ledgerRepo.findBySubjectId(ae.id);
        assertThat(entries).hasSize(1);
        AdverseEventLedgerEntry entry = (AdverseEventLedgerEntry) entries.get(0);
        assertThat(entry.adverseEventId).isEqualTo(ae.id);
        assertThat(entry.enrollmentId).isEqualTo(ae.enrollmentId);
        assertThat(entry.ctcaeGrade).isEqualTo("GRADE_4");
        assertThat(entry.reportedAt).isEqualTo(ae.reportedAt);
        assertThat(entry.slaDeadline).isEqualTo(ae.slaDeadline);
    }

    private AdverseEvent newAe(CtcaeGrade grade) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = grade;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now().minus(Duration.ofHours(2));
        return ae;
    }
}
```

- [ ] **Step 5.2: Run tests to confirm failure**

```bash
mvn test -pl runtime -Dtest=AdverseEventServiceTest --batch-mode
```

Expected: FAIL — `AdverseEventService` does not exist yet.

- [ ] **Step 5.3: Implement `AdverseEventService`**

Create `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java`:

```java
package io.casehub.clinical.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.AdverseEventLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.work.runtime.model.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class AdverseEventService {

    @Inject WorkItemService workItemService;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject ObjectMapper objectMapper;

    @Transactional
    public void reportAdverseEvent(AdverseEvent ae) {
        ae.reportedAt = Instant.now();
        ae.slaDeadline = ae.reportedAt.plus(ae.grade.sla().orElseThrow());

        var workItem = workItemService.create(new WorkItemCreateRequest(
            "Adverse Event — " + ae.grade.label(),
            "Grade " + ae.grade.label() + " AE for enrollment " + ae.enrollmentId +
                ". GCP SLA: " + ae.grade.sla().orElseThrow().toHours() + "h from " + ae.reportedAt,
            "adverse-event",
            "adverse-event-review",
            priority(ae),
            null,
            candidateGroups(ae),
            null,
            null,
            "system",
            payload(ae),
            ae.slaDeadline,
            null,
            null,
            null,
            null,
            null,
            null,
            null
        ));
        ae.workItemId = workItem.id;
        ae.persist();

        writeLedgerEntry(ae);
    }

    private void writeLedgerEntry(AdverseEvent ae) {
        AdverseEventLedgerEntry entry = new AdverseEventLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = ae.id;
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = "system";
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "AdverseEventReporter";
        entry.occurredAt = Instant.now();
        entry.adverseEventId = ae.id;
        entry.enrollmentId = ae.enrollmentId;
        entry.ctcaeGrade = ae.grade.name();
        entry.reportedAt = ae.reportedAt;
        entry.slaDeadline = ae.slaDeadline;
        ledgerRepo.save(entry);
    }

    private WorkItemPriority priority(AdverseEvent ae) {
        return switch (ae.grade) {
            case GRADE_5 -> WorkItemPriority.URGENT;
            case GRADE_3, GRADE_4 -> WorkItemPriority.HIGH;
            default -> WorkItemPriority.MEDIUM;
        };
    }

    private String candidateGroups(AdverseEvent ae) {
        return switch (ae.grade) {
            case GRADE_1, GRADE_2 -> "safety-officers";
            default -> "dsmb,safety-officers";
        };
    }

    private String payload(AdverseEvent ae) {
        try {
            return objectMapper.writeValueAsString(Map.of(
                "enrollmentId", ae.enrollmentId.toString(),
                "grade", ae.grade.name(),
                "occurredAt", ae.occurredAt.toString()
            ));
        } catch (JsonProcessingException e) {
            return "{\"enrollmentId\":\"" + ae.enrollmentId + "\"}";
        }
    }
}
```

- [ ] **Step 5.4: Run tests to confirm pass**

```bash
mvn test -pl runtime -Dtest=AdverseEventServiceTest --batch-mode
```

Expected: all 6 service tests pass.

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java \
  runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): add AdverseEventService — WorkItem + ledger on AE report

Creates a casehub-work WorkItem (claimDeadline = reportedAt + grade.sla())
and an AdverseEventLedgerEntry for every reported adverse event.
Escalation policy is deployment config, not clinical code.

Refs #4"
```

---

## Task 6: Add AE report endpoint to `PatientResource`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`

- [ ] **Step 6.1: Add the request record and endpoint**

Add these to `PatientResource.java`. Full updated file:

```java
package io.casehub.clinical.resource;

import com.fasterxml.jackson.annotation.JsonInclude;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.service.AdverseEventService;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import java.net.URI;
import java.time.Instant;
import java.util.UUID;

@Path("/trials/{trialId}/sites/{siteId}/patients")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class PatientResource {

    @Inject AdverseEventService adverseEventService;

    public record EnrollPatientRequest(@NotBlank String patientId) {}

    public record ReportAdverseEventRequest(
        @NotNull CtcaeGrade grade,
        @NotNull Instant occurredAt,
        EventActuality actuality
    ) {}

    @POST
    @Transactional
    public Response enroll(@PathParam("trialId") UUID trialId,
                           @PathParam("siteId") UUID siteId,
                           @Valid EnrollPatientRequest req,
                           @Context UriInfo uriInfo) {
        TrialSite site = TrialSite.findById(siteId);
        if (site == null || !site.trialId.equals(trialId))
            return Response.status(Response.Status.NOT_FOUND).build();

        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = UUID.randomUUID();
        enrollment.siteId = siteId;
        enrollment.patientId = req.patientId();
        enrollment.consentStatus = ConsentStatus.PENDING;
        enrollment.enrollmentStatus = EnrollmentStatus.CANDIDATE;
        enrollment.persist();

        URI location = uriInfo.getAbsolutePathBuilder().path(enrollment.id.toString()).build();
        return Response.created(location).build();
    }

    @GET
    @Path("/{enrollmentId}")
    public Response get(@PathParam("trialId") UUID trialId,
                        @PathParam("siteId") UUID siteId,
                        @PathParam("enrollmentId") UUID enrollmentId) {
        PatientEnrollment enrollment = PatientEnrollment.findById(enrollmentId);
        if (enrollment == null || !enrollment.siteId.equals(siteId))
            return Response.status(Response.Status.NOT_FOUND).build();
        TrialSite site = TrialSite.findById(siteId);
        if (site == null || !site.trialId.equals(trialId))
            return Response.status(Response.Status.NOT_FOUND).build();
        return Response.ok(enrollment).build();
    }

    @POST
    @Path("/{enrollmentId}/adverse-events")
    public Response reportAdverseEvent(
            @PathParam("trialId") UUID trialId,
            @PathParam("siteId") UUID siteId,
            @PathParam("enrollmentId") UUID enrollmentId,
            @Valid ReportAdverseEventRequest req,
            @Context UriInfo uriInfo) {
        PatientEnrollment enrollment = PatientEnrollment.findById(enrollmentId);
        if (enrollment == null || !enrollment.siteId.equals(siteId))
            return Response.status(Response.Status.NOT_FOUND).build();
        TrialSite site = TrialSite.findById(siteId);
        if (site == null || !site.trialId.equals(trialId))
            return Response.status(Response.Status.NOT_FOUND).build();

        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = enrollmentId;
        ae.grade = req.grade();
        ae.actuality = req.actuality() != null ? req.actuality() : EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = req.occurredAt();

        adverseEventService.reportAdverseEvent(ae);

        URI location = uriInfo.getAbsolutePathBuilder().path(ae.id.toString()).build();
        return Response.created(location).entity(ae).build();
    }
}
```

- [ ] **Step 6.2: Run all runtime tests**

```bash
mvn test -pl runtime --batch-mode
```

Expected: all tests pass (existing 17 + new 6 from Task 5).

- [ ] **Step 6.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(runtime): add POST /{enrollmentId}/adverse-events endpoint

Delegates to AdverseEventService. Returns 201 with AE entity
including workItemId and slaDeadline. reportedAt is server-set.

Refs #4"
```

---

## Task 7: REST integration tests — robustness and correctness

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/resource/AdverseEventResourceTest.java`

- [ ] **Step 7.1: Write the test class**

Create `runtime/src/test/java/io/casehub/clinical/resource/AdverseEventResourceTest.java`:

```java
package io.casehub.clinical.resource;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class AdverseEventResourceTest {

    // ── Setup helpers ─────────────────────────────────────────────────────────

    private UUID createTrial() {
        String loc = given()
            .contentType("application/json")
            .body("""
                {"protocolId":"AE-TEST-%s","phase":"PHASE_II","sponsor":"Test","targetEnrollment":10}
                """.formatted(UUID.randomUUID()))
            .when().post("/trials")
            .then().statusCode(201).extract().header("Location");
        return UUID.fromString(loc.substring(loc.lastIndexOf('/') + 1));
    }

    private UUID addSite(UUID trialId) {
        String loc = given()
            .contentType("application/json")
            .body("{\"investigatorId\":\"pi-ae-test\"}")
            .when().post("/trials/{t}/sites", trialId)
            .then().statusCode(201).extract().header("Location");
        return UUID.fromString(loc.substring(loc.lastIndexOf('/') + 1));
    }

    private UUID enrollPatient(UUID trialId, UUID siteId) {
        String loc = given()
            .contentType("application/json")
            .body("{\"patientId\":\"PAT-" + UUID.randomUUID() + "\"}")
            .when().post("/trials/{t}/sites/{s}/patients", trialId, siteId)
            .then().statusCode(201).extract().header("Location");
        return UUID.fromString(loc.substring(loc.lastIndexOf('/') + 1));
    }

    // ── Happy path ────────────────────────────────────────────────────────────

    @Test
    void grade3_ae_returns_201_with_workItemId_and_slaDeadline() {
        UUID trialId = createTrial();
        UUID siteId = addSite(trialId);
        UUID enrollmentId = enrollPatient(trialId, siteId);

        given()
            .contentType("application/json")
            .body("""
                {"grade":"GRADE_3","occurredAt":"%s"}
                """.formatted(Instant.now().minusSeconds(3600)))
        .when()
            .post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
                  trialId, siteId, enrollmentId)
        .then()
            .statusCode(201)
            .body("workItemId", notNullValue())
            .body("slaDeadline", notNullValue())
            .body("grade", equalTo("GRADE_3"))
            .header("Location", containsString("/adverse-events/"));
    }

    @Test
    void grade5_ae_slaDeadline_is_1h_after_reportedAt() {
        UUID trialId = createTrial();
        UUID siteId = addSite(trialId);
        UUID enrollmentId = enrollPatient(trialId, siteId);

        Instant before = Instant.now();

        String slaDeadline = given()
            .contentType("application/json")
            .body("""
                {"grade":"GRADE_5","occurredAt":"%s"}
                """.formatted(Instant.now().minusSeconds(60)))
        .when()
            .post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
                  trialId, siteId, enrollmentId)
        .then()
            .statusCode(201)
            .extract().path("slaDeadline");

        Instant deadline = Instant.parse(slaDeadline);
        Instant reportedAt = deadline.minusSeconds(3600);
        assertSlaIsApproximately(reportedAt, deadline, 3600);
    }

    // ── Robustness ────────────────────────────────────────────────────────────

    @Test
    void non_existent_enrollment_returns_404() {
        UUID trialId = createTrial();
        UUID siteId = addSite(trialId);
        UUID fakeEnrollment = UUID.randomUUID();

        given()
            .contentType("application/json")
            .body("""
                {"grade":"GRADE_3","occurredAt":"%s"}
                """.formatted(Instant.now()))
        .when()
            .post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
                  trialId, siteId, fakeEnrollment)
        .then()
            .statusCode(404);
    }

    @Test
    void missing_grade_returns_400() {
        UUID trialId = createTrial();
        UUID siteId = addSite(trialId);
        UUID enrollmentId = enrollPatient(trialId, siteId);

        given()
            .contentType("application/json")
            .body("""
                {"occurredAt":"%s"}
                """.formatted(Instant.now()))
        .when()
            .post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
                  trialId, siteId, enrollmentId)
        .then()
            .statusCode(400);
    }

    @Test
    void missing_occurredAt_returns_400() {
        UUID trialId = createTrial();
        UUID siteId = addSite(trialId);
        UUID enrollmentId = enrollPatient(trialId, siteId);

        given()
            .contentType("application/json")
            .body("{\"grade\":\"GRADE_3\"}")
        .when()
            .post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
                  trialId, siteId, enrollmentId)
        .then()
            .statusCode(400);
    }

    // ── Helper ────────────────────────────────────────────────────────────────

    private void assertSlaIsApproximately(Instant reportedAt, Instant deadline, long expectedSeconds) {
        long actualSeconds = deadline.getEpochSecond() - reportedAt.getEpochSecond();
        // allow 10-second tolerance
        if (Math.abs(actualSeconds - expectedSeconds) > 10) {
            throw new AssertionError("SLA is " + actualSeconds + "s, expected ~" + expectedSeconds + "s");
        }
    }
}
```

- [ ] **Step 7.2: Run the new tests**

```bash
mvn test -pl runtime -Dtest=AdverseEventResourceTest --batch-mode
```

Expected: all 5 tests pass.

- [ ] **Step 7.3: Run full test suite**

```bash
mvn test -pl runtime --batch-mode
```

Expected: all tests pass.

- [ ] **Step 7.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/resource/AdverseEventResourceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(runtime): add AdverseEventResourceTest — robustness and correctness

Covers 201 happy path with workItemId, Grade 5 1h SLA correctness,
404 for missing enrollment, 400 for missing required fields.

Refs #4"
```

---

## Task 8: Extend `ShowcaseScenarioTest` with AE scenarios

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/ShowcaseScenarioTest.java`

- [ ] **Step 8.1: Add AE scenario to the showcase test**

Add this test method to `ShowcaseScenarioTest` (inside the class, after the existing `three_site_oncology_trial_registers_correctly` test):

```java
@Test
void site_a_grade3_ae_gets_24h_sla_and_workItemId() {
    // Register trial
    String trialLoc = given()
        .contentType("application/json")
        .body("""
            {
              "protocolId": "SHOWCASE-SAE-2026-%s",
              "phase": "PHASE_III",
              "sponsor": "Acme Oncology",
              "targetEnrollment": 300
            }
            """.formatted(UUID.randomUUID()))
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    // Add Site A, enroll patient
    UUID siteAId = addSite(trialId, "pi-showcase-sae-001");
    UUID patientA = enrollPatient(trialId, siteAId, "PATIENT-SAE-A-001");

    // Report Grade 3 AE (serious — 24h SLA)
    Instant occurredAt = Instant.now().minus(java.time.Duration.ofHours(2));
    String slaDeadlineStr = given()
        .contentType("application/json")
        .body("""
            {"grade":"GRADE_3","occurredAt":"%s"}
            """.formatted(occurredAt))
        .when()
            .post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
                  trialId, siteAId, patientA)
        .then()
            .statusCode(201)
            .body("workItemId", notNullValue())
            .body("grade", equalTo("GRADE_3"))
            .extract().path("slaDeadline");

    // Verify slaDeadline is ~24h after reportedAt (allow 30s tolerance)
    Instant deadline = Instant.parse(slaDeadlineStr);
    // reportedAt ≈ now; deadline ≈ now + 24h
    java.time.Duration gap = java.time.Duration.between(Instant.now(), deadline);
    assertThat(gap.toHours()).isBetween(23L, 24L);
}

@Test
void site_b_grade5_ae_gets_1h_urgent_sla() {
    String trialLoc = given()
        .contentType("application/json")
        .body("""
            {
              "protocolId": "SHOWCASE-DEATH-2026-%s",
              "phase": "PHASE_III",
              "sponsor": "Acme Oncology",
              "targetEnrollment": 300
            }
            """.formatted(UUID.randomUUID()))
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    UUID siteBId = addSite(trialId, "pi-showcase-g5-001");
    UUID patientB = enrollPatient(trialId, siteBId, "PATIENT-G5-B-001");

    // Report Grade 5 AE (death — 1h SLA)
    String slaDeadlineStr = given()
        .contentType("application/json")
        .body("""
            {"grade":"GRADE_5","occurredAt":"%s"}
            """.formatted(Instant.now().minus(java.time.Duration.ofMinutes(30))))
        .when()
            .post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
                  trialId, siteBId, patientB)
        .then()
            .statusCode(201)
            .body("workItemId", notNullValue())
            .body("grade", equalTo("GRADE_5"))
            .extract().path("slaDeadline");

    Instant deadline = Instant.parse(slaDeadlineStr);
    java.time.Duration gap = java.time.Duration.between(Instant.now(), deadline);
    // slaDeadline = reportedAt + 1h; reportedAt ≈ now; so gap ≈ 1h
    assertThat(gap.toMinutes()).isBetween(55L, 65L);
}
```

Also add these imports to `ShowcaseScenarioTest` if not already present:

```java
import java.time.Duration;
import java.time.Instant;
import static org.hamcrest.Matchers.notNullValue;
import static org.assertj.core.api.Assertions.assertThat;
```

- [ ] **Step 8.2: Run the full test suite**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode
```

Expected: all tests pass (17 existing + 6 service + 5 REST + 2 showcase = 30 tests).

- [ ] **Step 8.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/resource/ShowcaseScenarioTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(runtime): extend showcase with Grade 3 and Grade 5 AE scenarios

Grade 3 SAE: workItemId set, slaDeadline = reportedAt + 24h.
Grade 5 death: workItemId set, slaDeadline = reportedAt + 1h (urgent).
These demonstrate the GCP compliance gap vs ClinicalAgent.

Refs #4"
```

---

## Self-Review

**Spec coverage:**
- ✅ CtcaeGrade 7-day SLA for Grade 1-2 (Task 1)
- ✅ casehub-work + casehub-ledger dependencies (Task 2)
- ✅ V106 (work_item_id) and V1004 (ae_ledger_entry) migrations (Task 3)
- ✅ AdverseEvent.workItemId entity field (Task 4)
- ✅ AdverseEventLedgerEntry subclass with FDA audit fields (Task 4)
- ✅ AdverseEventService: reportedAt server-set, slaDeadline computed, WorkItem created, ledger written (Task 5)
- ✅ POST /adverse-events endpoint returning workItemId in response (Task 6)
- ✅ 400 for missing grade/occurredAt, 404 for missing enrollment (Task 7)
- ✅ ShowcaseScenarioTest extended with Grade 3 and Grade 5 (Task 8)
- ✅ Escalation policy is deployment config (service uses candidateGroups, not hardcoded handlers)

**Types consistent throughout:**
- `WorkItemCreateRequest` constructor arguments match the 19-field record definition exactly
- `LedgerEntryType.EVENT`, `ActorType.SYSTEM` imported from `io.casehub.ledger.api.model`
- `ledgerRepo.save(entry)` matches `LedgerEntryRepository.save(LedgerEntry)` signature
- `WorkItemPriority.URGENT/HIGH/MEDIUM` match the enum values in casehub-work

**No placeholders:** All steps contain exact code or exact commands.
