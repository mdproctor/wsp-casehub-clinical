---
entry_type: note
projects: [clinical]
tags: [gcp, compliance, ledger, cdi, quarkus, testing, async-observer]
date: 2026-06-03
---
# The silent skip

ICH E6(R3) §5.17 requires that the fact of non-notification — and the reason for it — be independently verifiable in the trial's audit trail. Claude and I had the failure path covered: when a notification attempt throws an unexpected exception, a REQUIRES_NEW fallback writer records a `delivered=false` entry in the Merkle chain. What we hadn't covered were the deliberate exits.

Both `SafetyOfficerNotificationListener` and `SponsorNotificationListener` have several early-return paths: no siteId in the event, site not found, trial not found, connector config missing. These aren't crashes — they're data-quality exits, valid by design. The existing observer failure pattern doesn't fire for them. An FDA auditor looking at the trail would see an AE reported, a safety officer notification absent, and no explanation why.

The fix was to write a `delivered=false` entry for each path, with the `actorRole` encoding the specific reason. `"safety-officer-notifier-skipped-site-not-found"` tells the auditor why; a generic `"skipped"` doesn't. We added `writeSkippedEntry` and `writeSkippedSponsorEntry` as REQUIRES_NEW methods on each ledger writer, each wrapped in its own try-catch so a failed audit write can't trigger the outer observer failure handler — which would write a different, misleading entry type.

There was a schema complication. `safety_officer_notification_ledger_entry` had `site_id NOT NULL`. A skip triggered by a missing siteId has no site to record. V1013 makes both `site_id` and `enrollment_id` nullable — absent by definition when the event carries no site.

Then we hit a test flake. `AdverseEventServiceTest` was asserting `findBySubjectId(ae.id)` expected exactly one entry. Sometimes it found two: the `AdverseEventLedgerEntry` we were after, and a `SafetyOfficerNotificationLedgerEntry` committed by the listener in the background. When `reportAdverseEvent()` fires `AdverseEventReportedEvent` async, the safety officer listener picks it up, finds no TrialSite (the test doesn't set one up), and commits a skip entry for the same subject ID. `@ObservesAsync` delivery timing determines whether the listener fires before or after the assertion — intermittent, and easy to misread as an unrelated flake.

The fix: filter `findBySubjectId` results by subclass type in tests that care about a specific entry class. A more honest assertion anyway.

The documentation side was quick. `ARC42STORIES.MD` had two gaps from the self-assessment. Layer 5 described the IRB gate YAML binding in prose but showed no syntax — a reader replicating the layer had to open the file to understand `contextChange.filter` and parallel `humanTask` firing. The snippet is now inline in the key wiring section. Layer 3 was missing `SafetyOfficerNotificationLedgerEntry` from the key files; that's corrected with the V1013 migration story as context.
