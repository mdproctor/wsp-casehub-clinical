# Grade 3 IND 15-Day Expedited Reporting Path — Design

**Issue:** casehubio/clinical#81  
**Branch:** issue-80-clock-grade3-susar  
**Date:** 2026-06-16  
**Regulation:** 21 CFR 312.32(c)(1)(ii)

---

## Context

`RegulatorySubmissionCaseService` currently handles Grade 5 + unexpected AEs only
(21 CFR 312.32(c)(1)(i) — 7-day fatal/life-threatening path). Grade 3 (severe,
unexpected) triggers a distinct regulatory obligation: 15-day expedited safety
reporting under 21 CFR 312.32(c)(1)(ii). The compliance supplement is wrong if
applied to Grade 3 — it cites (c)(1)(i) in the planRef.

---

## Scope

- Grade 3 + unexpected → IND 15-day expedited path (21 CFR 312.32(c)(1)(ii))
- No Flyway migrations; no new entity fields
- `indReportingDeadline` passed in engine case context only (not stored on entity)
- Grade 4 gap filed separately as #82

---

## Changes

### `RegulatorySubmissionCaseService`

Replace `REPORTABLE_GRADES = Set.of(GRADE_5)` with a predicate method:

```java
private static boolean isIndReportable(CtcaeGrade grade) {
    return grade == CtcaeGrade.GRADE_3 || grade == CtcaeGrade.GRADE_5;
}

private static Duration indReportingWindow(CtcaeGrade grade) {
    return switch (grade) {
        case GRADE_5 -> Duration.ofDays(7);
        case GRADE_3 -> Duration.ofDays(15);
        default -> throw new IllegalArgumentException("no IND reporting window for " + grade);
    };
}
```

`prepareAndMark()` filter changes from:
```java
if (!REPORTABLE_GRADES.contains(ae.grade) || !ae.unexpected) return null;
```
to:
```java
if (!isIndReportable(ae.grade) || !ae.unexpected) return null;
```

Case context gains two keys:
```java
"indReportingDeadline", ae.reportedAt.plus(indReportingWindow(ae.grade)).toString(),
"cfrSubclause",         ae.grade == CtcaeGrade.GRADE_5 ? "c_1_i" : "c_1_ii"
```

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

One caller to update: `RegulatorySubmissionLedgerWriter.writeEntry(ae)` passes `ae.grade`.

### `regulatory-submission.yaml`

Add `indReportingDeadline` to inputSchema so the capability agent knows the deadline:

```yaml
inputSchema: "{ grade: .grade, unexpected: .unexpected, aeId: .aeId, indReportingDeadline: .indReportingDeadline }"
```

### Tests

**`RegulatorySubmissionCaseServiceTest`** — three new tests:
1. `grade3_unexpected_starts_regulatory_case()` — `regulatorySubmissionStatus = PENDING`, `regulatorySubmissionCaseId` non-null
2. `grade3_expected_does_not_start_regulatory_case()` — status stays `NONE`
3. `grade3_case_context_includes_15_day_deadline()` — capture `startCase()` arg, assert `indReportingDeadline = reportedAt + 15 days`

Existing test `grade4_unexpected_does_not_start_regulatory_case` remains valid — Grade 4 is deferred to #82.

**`RegulatorySubmissionLedgerWriterTest`**:
- Rename existing test to `grade5_writes_entry_with_correct_planRef_c1i()`
- Add `grade3_writes_entry_with_correct_planRef_c1ii()` — Grade 3 AE → supplement planRef contains "(c)(1)(ii)"

---

## Files Modified

| File | Change |
|------|--------|
| `runtime/.../service/RegulatorySubmissionCaseService.java` | Predicate + helpers + context keys |
| `runtime/.../service/ClinicalComplianceSupplement.java` | Grade-aware factory method |
| `runtime/.../service/RegulatorySubmissionLedgerWriter.java` | Pass `ae.grade` to supplement |
| `runtime/.../resources/clinical/regulatory-submission.yaml` | Add `indReportingDeadline` to inputSchema |
| `runtime/.../service/RegulatorySubmissionCaseServiceTest.java` | 3 new tests |
| `runtime/.../service/RegulatorySubmissionLedgerWriterTest.java` | Grade-specific planRef tests |

No migrations. No new entity fields.
