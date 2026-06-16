# Grade 3 IND 15-Day Expedited Reporting Path — Design

**Issue:** casehubio/clinical#81  
**Branch:** issue-80-clock-grade3-susar  
**Date:** 2026-06-16  
**Regulation:** 21 CFR 312.32(c)(1)(ii)

---

## Regulatory structure (21 CFR 312.32)

| Subclause | Trigger | Window | Grades |
|-----------|---------|--------|--------|
| (c)(1)(i) | Unexpected fatal or life-threatening ADR | 7 calendar days | Grade 4, Grade 5 |
| (c)(1)(ii) | Unexpected serious ADR, not fatal/life-threatening | 15 calendar days | Grade 3 |

Currently: Grade 5 implemented. Grade 4 gap filed as #82. This spec: Grade 3 only.

---

## Context

`RegulatorySubmissionCaseService` filters on `REPORTABLE_GRADES = Set.of(GRADE_5)`.
`ClinicalComplianceSupplement.regulatorySubmission()` is hardcoded to (c)(1)(i) planRef.
Both need to become grade-aware to support Grade 3 → 15-day (c)(1)(ii).

---

## Scope

- Grade 3 + unexpected → IND 15-day expedited path (21 CFR 312.32(c)(1)(ii))
- No Flyway migrations; no new entity fields
- `indReportingDeadline` passed in engine case context only (not stored on entity)
- Grade 4 deferred to #82
- IND deadline enforcement via WorkItem claimDeadline deferred to #83

---

## Changes

### `RegulatorySubmissionCaseService`

Replace `REPORTABLE_GRADES = Set.of(GRADE_5)` with a predicate + window helper:

```java
private static boolean isIndReportable(CtcaeGrade grade) {
    return grade == CtcaeGrade.GRADE_3 || grade == CtcaeGrade.GRADE_5;
}

// When #82 adds GRADE_4: case GRADE_4 -> Duration.ofDays(7)
private static Duration indReportingWindow(CtcaeGrade grade) {
    return switch (grade) {
        case GRADE_5 -> Duration.ofDays(7);
        case GRADE_3 -> Duration.ofDays(15);
        default -> throw new IllegalArgumentException("no IND reporting window for " + grade);
    };
}
```

`prepareAndMark()` filter becomes:
```java
if (!isIndReportable(ae.grade) || !ae.unexpected) return null;
```

Case context adds `indReportingDeadline` (CFR subclause is derivable from grade — not included):
```java
"indReportingDeadline", ae.reportedAt.plus(indReportingWindow(ae.grade)).toString()
```

`ae.reportedAt` is the entity field set by `AdverseEventService` — used directly, no Clock injection needed.

Stale inline comment updated: "Only Grade 5 + unexpected triggers IND expedited safety reporting" →
"Grade 3 (15-day, §(c)(1)(ii)) and Grade 5 (7-day, §(c)(1)(i)) + unexpected trigger IND expedited safety reporting."

### `ClinicalComplianceSupplement`

Signature change: `regulatorySubmission()` → `regulatorySubmission(CtcaeGrade grade)`:

```java
public static ComplianceSupplement regulatorySubmission(CtcaeGrade grade) {
    ComplianceSupplement s = new ComplianceSupplement();
    s.planRef = switch (grade) {
        case GRADE_5 -> "21 CFR 312.32(c)(1)(i) — IND 7-day expedited safety reporting, unexpected fatal AE";
        case GRADE_3 -> "21 CFR 312.32(c)(1)(ii) — IND 15-day expedited safety reporting, unexpected serious AE";
        default -> throw new IllegalArgumentException("no IND planRef for " + grade);
    };
    s.algorithmRef = "RegulatorySubmissionCaseService (rule-based CTCAE grade + unexpected criteria)";
    s.humanOverrideAvailable = true;
    return s;
}
```

One caller: `RegulatorySubmissionLedgerWriter.writeEntry(ae)` → passes `ae.grade`.

### `RegulatorySubmissionLedgerWriter`

Class javadoc updated: "Grade 5 + unexpected AE triggers IND expedited safety reporting obligation" →
"Unexpected Grade 3 (15-day) or Grade 5 (7-day) AE triggers IND expedited safety reporting obligation."

Call site: `ClinicalComplianceSupplement.regulatorySubmission()` → `ClinicalComplianceSupplement.regulatorySubmission(ae.grade)`.

### `RegulatorySubmissionLedgerEntry`

Class javadoc updated: "Grade 5 + unexpected criteria are confirmed" →
"Grade 3 or Grade 5 + unexpected criteria are confirmed."

### `regulatory-submission.yaml`

Add `indReportingDeadline` to inputSchema so the capability agent knows the deadline:

```yaml
inputSchema: "{ grade: .grade, unexpected: .unexpected, aeId: .aeId, indReportingDeadline: .indReportingDeadline }"
```

### Tests

**`RegulatorySubmissionCaseServiceTest`** — three new tests:

1. `grade3_unexpected_starts_regulatory_case()` — `regulatorySubmissionStatus = PENDING`,
   `regulatorySubmissionCaseId` non-null.

2. `grade3_expected_does_not_start_regulatory_case()` — status stays `NONE`.

3. `grade3_case_context_includes_15_day_deadline()` — use `persistAe(grade, unexpected, fixedReportedAt)`
   overload with a fixed `Instant`; capture `startCase()` arg via `ArgumentCaptor<Map>`; assert
   `Instant.parse(capturedContext.get("indReportingDeadline"))
    .equals(fixedReportedAt.plus(Duration.ofDays(15)))`. No timing ambiguity.

`persistAe()` gets an overload: `persistAe(CtcaeGrade grade, boolean unexpected, Instant reportedAt)`.

Existing `grade4_unexpected_does_not_start_regulatory_case` stays valid — Grade 4 deferred to #82.

**`RegulatorySubmissionLedgerWriterTest`**:

- Rename existing test: `writes_entry_with_correct_fields()` →
  `grade5_writes_entry_with_c1i_planRef_and_correct_fields()`; add planRef assertion:
  `rsle.getSupplements().get(0).planRef.contains("(c)(1)(i)")`.
- Add `grade3_writes_entry_with_c1ii_planRef()`: Grade 3 AE → supplement planRef contains `"(c)(1)(ii)"`.

---

## Files Modified

| File | Change |
|------|--------|
| `runtime/.../service/RegulatorySubmissionCaseService.java` | `isIndReportable()` predicate, `indReportingWindow()` helper, context key, stale comment |
| `runtime/.../service/ClinicalComplianceSupplement.java` | Grade-aware `regulatorySubmission(CtcaeGrade)` |
| `runtime/.../service/RegulatorySubmissionLedgerWriter.java` | Pass `ae.grade` to supplement; class javadoc |
| `runtime/.../ledger/RegulatorySubmissionLedgerEntry.java` | Class javadoc |
| `runtime/.../resources/clinical/regulatory-submission.yaml` | Add `indReportingDeadline` to inputSchema |
| `runtime/.../service/RegulatorySubmissionCaseServiceTest.java` | 3 new tests; `persistAe` overload |
| `runtime/.../service/RegulatorySubmissionLedgerWriterTest.java` | Grade-specific planRef tests |

No migrations. No new entity fields.

---

## Deferred

| Issue | What |
|-------|------|
| #82 | Grade 4 + unexpected → IND 7-day (c)(1)(i); `indReportingWindow()` gains `GRADE_4 -> Duration.ofDays(7)` |
| #83 | IND reporting deadline enforced as WorkItem `claimDeadline` with auto-escalation |
