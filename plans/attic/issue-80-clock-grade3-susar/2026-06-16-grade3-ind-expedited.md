# Grade 3 IND 15-Day Expedited Reporting Path — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `RegulatorySubmissionCaseService` to trigger IND 15-day expedited reporting for unexpected Grade 3 AEs (21 CFR 312.32(c)(1)(ii)), alongside the existing Grade 5 7-day path (21 CFR 312.32(c)(1)(i)).

**Architecture:** `ClinicalComplianceSupplement.regulatorySubmission()` becomes grade-aware (`regulatorySubmission(CtcaeGrade grade)`) — breaking the existing signature intentionally to force all callers to be explicit about which regulatory path they're on. `RegulatorySubmissionCaseService` replaces a `Set<CtcaeGrade>` constant with an `isIndReportable()` predicate and an `indReportingWindow()` helper. The computed `indReportingDeadline` (ISO-8601 string) is passed in the engine case context; the YAML capability agent receives it via `inputSchema`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, Mockito, AssertJ, Awaitility, `@QuarkusTest` (H2 in-memory)

---

## File Map

| File | Change |
|------|--------|
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java` | `regulatorySubmission()` → `regulatorySubmission(CtcaeGrade)` |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java` | Pass `ae.grade`; class javadoc |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java` | Predicate + helper; context key; stale comment |
| `runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java` | Class javadoc |
| `runtime/src/main/resources/clinical/regulatory-submission.yaml` | Add `indReportingDeadline` to inputSchema |
| `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java` | Rename test; add planRef assertions; add Grade 3 test |
| `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCaseServiceTest.java` | `persistAe` overload; 3 new tests |

---

## Task 1: Grade-aware `ClinicalComplianceSupplement` + fix the one caller

### Files
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java`

- [ ] **Step 1: Write failing tests for grade-specific planRef**

Replace the entire content of `RegulatorySubmissionLedgerWriterTest`:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.argThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.ledger.RegulatorySubmissionLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import java.time.Clock;
import java.time.Instant;
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
    @Mock Clock clock;
    @InjectMocks RegulatorySubmissionLedgerWriter writer;

    @Test
    void grade5_writes_entry_with_c1i_planRef_and_correct_fields() {
        final Instant now = Instant.parse("2026-06-15T12:00:00Z");
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any(), any())).thenReturn(Optional.empty());

        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_5;
        ae.tenantId = "test-tenant";

        writer.writeEntry(ae);

        verify(ledgerEntryRepository).save(
                argThat(entry -> {
                    RegulatorySubmissionLedgerEntry rsle = (RegulatorySubmissionLedgerEntry) entry;
                    return rsle.aeId.equals(ae.id)
                            && "GRADE_5".equals(rsle.grade)
                            && rsle.filedAt.equals(now)
                            && rsle.subjectId.equals(ae.enrollmentId)
                            && rsle.sequenceNumber == 1
                            && rsle.id != null
                            && rsle.compliance()
                                    .map(c -> c.planRef)
                                    .orElse("")
                                    .contains("(c)(1)(i)");
                }),
                eq("default"));
    }

    @Test
    void grade3_writes_entry_with_c1ii_planRef() {
        final Instant now = Instant.parse("2026-06-16T12:00:00Z");
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any(), any())).thenReturn(Optional.empty());

        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_3;
        ae.tenantId = "test-tenant";

        writer.writeEntry(ae);

        verify(ledgerEntryRepository).save(
                argThat(entry -> {
                    RegulatorySubmissionLedgerEntry rsle = (RegulatorySubmissionLedgerEntry) entry;
                    return "GRADE_3".equals(rsle.grade)
                            && rsle.compliance()
                                    .map(c -> c.planRef)
                                    .orElse("")
                                    .contains("(c)(1)(ii)");
                }),
                eq("default"));
    }
}
```

- [ ] **Step 2: Run to confirm both tests fail**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionLedgerWriterTest --batch-mode
```

Expected: `grade3_writes_entry_with_c1ii_planRef` fails (method not found or wrong planRef). `grade5_writes_entry_with_c1i_planRef_and_correct_fields` fails if `regulatorySubmission()` no longer compiles after the signature change — but first check which error appears.

- [ ] **Step 3: Change `ClinicalComplianceSupplement.regulatorySubmission()` to grade-aware**

In `ClinicalComplianceSupplement.java`, replace:

```java
public static ComplianceSupplement regulatorySubmission() {
    ComplianceSupplement s = new ComplianceSupplement();
    s.planRef = "21 CFR 312.32(c)(1)(i) — IND expedited safety reporting, unexpected fatal/life-threatening AE";
    s.algorithmRef = "RegulatorySubmissionCaseService (rule-based Grade 5 + unexpected criteria)";
    s.humanOverrideAvailable = true;
    return s;
}
```

with:

```java
public static ComplianceSupplement regulatorySubmission(CtcaeGrade grade) {
    ComplianceSupplement s = new ComplianceSupplement();
    s.planRef = switch (grade) {
        case GRADE_5 -> "21 CFR 312.32(c)(1)(i) — IND 7-day expedited safety reporting, unexpected fatal AE";
        case GRADE_3 -> "21 CFR 312.32(c)(1)(ii) — IND 15-day expedited safety reporting, unexpected serious AE";
        default -> throw new IllegalArgumentException("no IND planRef for grade: " + grade);
    };
    s.algorithmRef = "RegulatorySubmissionCaseService (rule-based CTCAE grade + unexpected criteria)";
    s.humanOverrideAvailable = true;
    return s;
}
```

Add import at the top of `ClinicalComplianceSupplement.java`:

```java
import io.casehub.clinical.api.model.CtcaeGrade;
```

- [ ] **Step 4: Fix the one caller — `RegulatorySubmissionLedgerWriter.writeEntry()`**

In `RegulatorySubmissionLedgerWriter.java`, change the `attach` call:

```java
// BEFORE
entry.attach(ClinicalComplianceSupplement.regulatorySubmission());

// AFTER
entry.attach(ClinicalComplianceSupplement.regulatorySubmission(ae.grade));
```

Also update the class javadoc — change:
```
 * Writes tamper-evident record when Grade 5 + unexpected AE triggers IND expedited
 * safety reporting obligation (21 CFR 312.32(c)(1)(i)).
```
to:
```
 * Writes tamper-evident record when an unexpected Grade 3 (15-day, 21 CFR 312.32(c)(1)(ii))
 * or Grade 5 (7-day, 21 CFR 312.32(c)(1)(i)) AE triggers IND expedited safety reporting.
```

- [ ] **Step 5: Run tests**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionLedgerWriterTest --batch-mode
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java \
  runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: grade-aware ClinicalComplianceSupplement.regulatorySubmission(CtcaeGrade)

(c)(1)(i) for Grade 5 (fatal, 7-day); (c)(1)(ii) for Grade 3 (serious, 15-day).
One caller updated: RegulatorySubmissionLedgerWriter.writeEntry() passes ae.grade.

Refs #81"
```

---

## Task 2: Extend `RegulatorySubmissionCaseService` for Grade 3

### Files
- Modify: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCaseServiceTest.java`

- [ ] **Step 1: Write three failing tests**

Add to `RegulatorySubmissionCaseServiceTest.java`.

First, add these imports if not already present:
```java
import java.time.Duration;
import org.mockito.ArgumentCaptor;
import static org.assertj.core.api.Assertions.assertThat;
```

Add a `persistAe` overload with a fixed `reportedAt`:
```java
@Transactional
UUID persistAe(CtcaeGrade grade, boolean unexpected, Instant reportedAt) {
    AdverseEvent ae = new AdverseEvent();
    ae.id = UUID.randomUUID();
    ae.enrollmentId = UUID.randomUUID();
    ae.grade = grade;
    ae.unexpected = unexpected;
    ae.suspected = true;
    ae.actuality = EventActuality.ACTUAL;
    ae.outcome = AeOutcome.ONGOING;
    ae.occurredAt = Instant.now();
    ae.reportedAt = reportedAt;
    ae.tenantId = "test-tenant";
    ae.persist();
    return ae.id;
}
```

Add three new tests:
```java
@Test
void grade3_unexpected_starts_regulatory_case() {
    UUID aeId = persistAe(CtcaeGrade.GRADE_3, true);
    AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_3);

    service.onAdverseEventReported(event);

    await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
        AdverseEvent ae = findAe(aeId);
        assertThat(ae.regulatorySubmissionStatus).isEqualTo(RegulatorySubmissionStatus.PENDING);
        assertThat(ae.regulatorySubmissionCaseId).isNotNull();
    });
}

@Test
void grade3_expected_does_not_start_regulatory_case() {
    UUID aeId = persistAe(CtcaeGrade.GRADE_3, false);
    AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_3);

    service.onAdverseEventReported(event);

    AdverseEvent ae = findAe(aeId);
    assertThat(ae.regulatorySubmissionStatus).isEqualTo(RegulatorySubmissionStatus.NONE);
    assertThat(ae.regulatorySubmissionCaseId).isNull();
}

@Test
@SuppressWarnings("unchecked")
void grade3_case_context_includes_15_day_ind_deadline() {
    final Instant fixedReportedAt = Instant.parse("2026-06-16T09:00:00Z");
    UUID aeId = persistAe(CtcaeGrade.GRADE_3, true, fixedReportedAt);
    AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_3);

    service.onAdverseEventReported(event);

    await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
        ArgumentCaptor<Map> captor = ArgumentCaptor.forClass(Map.class);
        verify(regulatorySubmissionCaseHub).startCase(captor.capture());
        Map<String, Object> ctx = captor.getValue();
        assertThat(ctx).containsKey("indReportingDeadline");
        assertThat(Instant.parse((String) ctx.get("indReportingDeadline")))
                .isEqualTo(fixedReportedAt.plus(Duration.ofDays(15)));
    });
}
```

- [ ] **Step 2: Run to confirm all three tests fail**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionCaseServiceTest --batch-mode
```

Expected: 3 new tests fail. Existing 5 tests pass.

- [ ] **Step 3: Extend `RegulatorySubmissionCaseService`**

In `RegulatorySubmissionCaseService.java`:

Add import:
```java
import java.time.Duration;
```

Replace the constant:
```java
// REMOVE:
private static final Set<CtcaeGrade> REPORTABLE_GRADES = Set.of(CtcaeGrade.GRADE_5);

// ADD:
private static boolean isIndReportable(final CtcaeGrade grade) {
    return grade == CtcaeGrade.GRADE_3 || grade == CtcaeGrade.GRADE_5;
}

// When #82 adds GRADE_4: add case GRADE_4 -> Duration.ofDays(7)
private static Duration indReportingWindow(final CtcaeGrade grade) {
    return switch (grade) {
        case GRADE_5 -> Duration.ofDays(7);
        case GRADE_3 -> Duration.ofDays(15);
        default -> throw new IllegalArgumentException("no IND reporting window for grade: " + grade);
    };
}
```

In `prepareAndMark()`, change the filter:
```java
// BEFORE:
if (!REPORTABLE_GRADES.contains(ae.grade) || !ae.unexpected) {
    return null;
}

// AFTER:
if (!isIndReportable(ae.grade) || !ae.unexpected) {
    return null;
}
```

Also in `prepareAndMark()`, update the inline comment and the context map:
```java
// BEFORE (comment):
// Only Grade 5 + unexpected triggers IND expedited safety reporting

// AFTER (comment):
// Grade 3 (15-day, §(c)(1)(ii)) and Grade 5 (7-day, §(c)(1)(i)) + unexpected trigger IND expedited safety reporting

// BEFORE (return statement):
return Map.of(
        "aeId", ae.id.toString(),
        "grade", ae.grade.name(),
        "unexpected", ae.unexpected,
        "siteId", event.siteId().toString(),
        "tenantId", ae.tenantId);

// AFTER (return statement):
return Map.of(
        "aeId", ae.id.toString(),
        "grade", ae.grade.name(),
        "unexpected", ae.unexpected,
        "siteId", event.siteId().toString(),
        "tenantId", ae.tenantId,
        "indReportingDeadline", ae.reportedAt.plus(indReportingWindow(ae.grade)).toString());
```

- [ ] **Step 4: Run all service tests**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionCaseServiceTest --batch-mode
```

Expected: `Tests run: 8, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCaseServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat: Grade 3 + unexpected triggers IND 15-day expedited reporting

isIndReportable() predicate covers GRADE_3 and GRADE_5. indReportingWindow()
returns 15 days for Grade 3 (21 CFR 312.32(c)(1)(ii)), 7 days for Grade 5
(21 CFR 312.32(c)(1)(i)). indReportingDeadline passed in engine case context.

Grade 4 deferred to #82. Enforcement via WorkItem claimDeadline deferred to #83.

Closes #81
Refs #10"
```

---

## Task 3: YAML inputSchema + stale javadocs

### Files
- Modify: `runtime/src/main/resources/clinical/regulatory-submission.yaml`
- Modify: `runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java`

- [ ] **Step 1: Update YAML inputSchema**

In `regulatory-submission.yaml`, change:
```yaml
# BEFORE:
    inputSchema: "{ grade: .grade, unexpected: .unexpected, aeId: .aeId }"

# AFTER:
    inputSchema: "{ grade: .grade, unexpected: .unexpected, aeId: .aeId, indReportingDeadline: .indReportingDeadline }"
```

- [ ] **Step 2: Update `RegulatorySubmissionLedgerEntry` class javadoc**

Change the class javadoc from:
```
 * Written in Phase 1 of RegulatorySubmissionCaseService when Grade 5 + unexpected
 * criteria are confirmed.
```
to:
```
 * Written in Phase 1 of RegulatorySubmissionCaseService when Grade 3 or Grade 5
 * + unexpected criteria are confirmed (21 CFR 312.32(c)(1)(i)/(c)(1)(ii)).
```

- [ ] **Step 3: Full suite run**

```bash
mvn install -pl api --batch-mode -q && mvn test --batch-mode
```

Expected: `Tests run: 376, Failures: 0, Errors: 0` (374 existing + 2 new writer tests + 3 new service tests = 379; minus the renamed test = net +4 tests... actually: 374 + 1 new grade5 planRef assertion + 1 new grade3 writer test + 3 new service tests = 374 + 4 new = 378 total, but the renamed test still runs so it should be 374 + 4 = 378).

Expected output ends with: `Tests run: 378, Failures: 0, Errors: 0` (approximately — exact count depends on test runner grouping).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/resources/clinical/regulatory-submission.yaml \
  runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "docs: sync javadocs and YAML for Grade 3 IND expedited path

regulatory-submission.yaml inputSchema gains indReportingDeadline.
Stale Grade 5-only javadocs updated in RegulatorySubmissionLedgerEntry
and RegulatorySubmissionLedgerWriter.

Refs #81"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `isIndReportable(grade)` predicate replacing `REPORTABLE_GRADES` | Task 2, Step 3 |
| `indReportingWindow(grade)` helper — Grade 3 → 15d, Grade 5 → 7d | Task 2, Step 3 |
| `indReportingDeadline` in engine case context | Task 2, Step 3 |
| `ClinicalComplianceSupplement.regulatorySubmission(CtcaeGrade)` | Task 1, Step 3 |
| One caller fix: `RegulatorySubmissionLedgerWriter` | Task 1, Step 4 |
| YAML `inputSchema` gains `indReportingDeadline` | Task 3, Step 1 |
| Test: Grade 3 + unexpected starts case | Task 2, Step 1 |
| Test: Grade 3 + expected does not start case | Task 2, Step 1 |
| Test: context includes correct 15-day deadline with fixed `reportedAt` | Task 2, Step 1 |
| Test: Grade 5 planRef still (c)(1)(i) | Task 1, Step 1 |
| Test: Grade 3 planRef is (c)(1)(ii) | Task 1, Step 1 |
| `RegulatorySubmissionLedgerEntry` javadoc | Task 3, Step 2 |
| `RegulatorySubmissionLedgerWriter` javadoc | Task 1, Step 4 |
| `prepareAndMark()` inline comment | Task 2, Step 3 |
| No migrations | ✅ confirmed — no migration files in any task |
| `grade4_unexpected_does_not_start_regulatory_case` left unchanged | ✅ no task touches it |

All spec items covered. No placeholders. Types and method signatures consistent across tasks.
