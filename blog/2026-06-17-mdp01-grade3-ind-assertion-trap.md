---
layout: post
title: "Grade 3 IND reporting, and an assertion that almost passed for the wrong grade"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [clinical-trials, regulatory, testing, java]
---

The Grade 5 IND expedited reporting path has been live for a week — unexpected fatal adverse events now start a regulatory submission case within 24 hours of being reported. Grade 3 was the next step: unexpected severe events that aren't fatal carry a 15-day reporting window under 21 CFR 312.32(c)(1)(ii).

The extension is three methods deep. `isIndReportable()` determines which grades trigger filing. `indReportingWindow()` returns the duration. `ClinicalComplianceSupplement.regulatorySubmission(CtcaeGrade)` returns the grade-specific planRef with the correct CFR citation for the ledger entry. Miss any one of them and you either skip reporting silently or get `IllegalArgumentException` at runtime on the first Grade 3 event.

That part was straightforward. The test assertion wasn't.

21 CFR uses a hierarchy of citation levels. The 7-day path for fatal and life-threatening events is (c)(1)(i). The 15-day path for other serious events is (c)(1)(ii). The Grade 5 test checked `.contains("(c)(1)(i)")` for the planRef. This passes — and also passes when the planRef contains "(c)(1)(ii)", because the shorter citation is a substring of the longer one.

The Grade 5 test would have said nothing useful if Grade 5 had accidentally returned the Grade 3 planRef. The fix: assert both `.contains("(c)(1)(i)")` and `!.contains("(c)(1)(ii)")`. The pattern generalises: any hierarchical identifier where one form is a prefix of another — ISO codes, SemVer strings, CFR citations — needs both a positive and a negative assertion to discriminate.

One other thing worth naming. The existing test `grade4_unexpected_does_not_start_regulatory_case` asserts that Grade 4 (life-threatening) doesn't trigger IND reporting. That's wrong by the regulation — Grade 4 falls under (c)(1)(i) alongside Grade 5. The test was preserved with a gap issue filed separately.

A codebase that knows exactly where it's wrong is more useful than one that papers over it.
