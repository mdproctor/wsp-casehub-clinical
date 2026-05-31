---
layout: post
title: "When caught exceptions commit"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [gcp, cdi, ledger, jta, compliance]
---

Issues #45 and #46 extended the observer fallback pattern to `SafetyOfficerNotificationListener` and `SponsorNotificationListener`. Issue #48 extends it to `AeEscalationListener` and `IrbDecisionListener`. Those two have a different failure topology.

The notification listeners are simple: the entire body is in a try/catch, the catch writes a failure ledger entry, done. No REQUIRES_NEW has committed before the try block starts, so if anything throws, there's no orphaned state — just a missing notification record, which the fallback covers.

`AeEscalationListener` is not like that. Before the ledger write, it calls `AeStatusUpdater.markCompleted(aeId)` — `@Transactional(REQUIRES_NEW)`, commits independently. If that succeeds and then `writeCompletionEntry` throws, the AE escalation status is COMPLETED in the database with no corresponding ledger entry. The fallback covers that gap. So far so similar.

The wrinkle is `completedEvents.fireAsync()`. If the ledger write commits and `fireAsync` throws — and the observer's catch block swallows it — what happens?

The outer `@Transactional` interceptor rolls back only when an exception escapes the method. If the catch returns normally, the interceptor commits. `writeCompletionEntry` result is in the DB. And so is the REQUIRES_NEW failure entry that the catch block just wrote. Two entries for one event: one recording success, one recording failure. An auditor sees a contradiction.

The fix is a boolean flag set immediately after the critical write:

```java
boolean ledgerWritten = false;
try {
    ledgerWriter.writeCompletionEntry(aeId, enrollmentId, grade, ...);
    ledgerWritten = true;
    completedEvents.fireAsync(new AeEscalationCompletedEvent(...));
} catch (Exception e) {
    if (!ledgerWritten) {
        try { ledgerWriter.writeObserverFailureEntry(aeId, enrollmentId, grade); }
        catch (Exception writeEx) { LOG.errorf(writeEx, "AUDIT GAP: ..."); }
    } else {
        LOG.errorf(e, "downstream fireAsync failed — ledger entry exists, no fallback");
    }
}
```

If `fireAsync` throws, `ledgerWritten` is already true. No second REQUIRES_NEW write. No double-recording.

The initial design spec for `IrbDecisionListener` missed this. The body had `approval.persist()`, case signaling, `ledgerWriter.writeDecisionEntry`, and `fireAsync` in a single try/catch with a fallback and no flag. Code review flagged it as critical: same topology, same risk. `writeDecisionEntry` commits via the outer TX; if `fireAsync` throws after that, both entries commit. I added `ledgerDecisionWritten` and the same conditional.

The other structural change was moving context resolution outside the try block in `AeEscalationListener`. The issue: `enrollmentId`, `grade`, and `siteId` are all needed to write a meaningful failure entry. But `markCompleted` commits before they're resolved. If resolution throws, status is COMPLETED with no ledger entry and nothing to write a failure entry against — we don't have the subject identifiers. Putting resolution outside the try block doesn't close that gap; it just makes the scoping explicit. The exception propagates to the dispatcher, gets logged, and the gap exists. Filed as part of #52.

Code review found one more issue: `AeEscalationLedgerWriter.writeObserverFailureEntry` was calling `clock.instant()` twice — once for `occurredAt`, once for `completedAt`, with several field assignments between. Claude flagged it during review: two different timestamps for two fields that should represent the same moment. The fix is one `Instant now = clock.instant()` at the top, shared by both. Small thing, but an FDA audit record where `occurredAt ≠ completedAt` on a failure entry would invite questions.

All four clinical listeners now have the full pattern. 13 new tests on the branch, pre-existing test suite intact.
