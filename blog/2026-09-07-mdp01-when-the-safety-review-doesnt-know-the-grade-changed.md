---
title: "When the Safety Review Doesn't Know the Grade Changed"
date: 2026-09-07
author: mdp
entry_type: note
subtype: diary
series: issue-147-escalation-regrade
projects: [casehubio/clinical]
tags: [adverse-events, escalation, engine-cases, audit-trail, TOCTOU]
---

The AE regrading spec (#135) introduced grade changes — Grade 1 nausea becomes Grade 3 dehydration. The spec handled the easy case: when no engine case exists, start one. But it explicitly punted on the hard case: what happens when a Grade 3 AE already has an active escalation case and the grade upgrades to 4?

The existing code returned `null` and moved on. The `requiresDsmbEscalation` flag — frozen at `false` when the Grade 3 case was created — never got a second look. A Grade 4 AE walked past the DSMB gate without triggering it.

I initially thought this might be a configuration question — maybe different sponsors want different behaviour. But the `AdverseEventEscalationPolicy` SPI already IS the configuration point. It says Grade 4 needs DSMB. Making the mechanism separately configurable would let organisations say "yes, our policy requires DSMB at Grade 4, but please don't actually do it when upgrading from Grade 3." That's a compliance contradiction, not a policy choice.

The fix: always start a fresh case. The old case continues if active — its safety review is still clinically valid work, just at a stale grade. The new case gets the correct Grade 4 requirements from the policy.

The interesting engineering problem was the completion handler. Two cases for the same AE mean two eventual `GoalReached` events. The old `boolean markCompleted(UUID aeId)` couldn't distinguish which case was finishing. We replaced it with a `CompletionResult` enum — COMPLETED, SUPERSEDED, ALREADY_COMPLETED, NOT_FOUND — with case-ID discrimination inside the `REQUIRES_NEW` transaction.

The decision review caught a race condition I'd missed. Between `prepareAndMarkForRegrade()` (phase 1, which commits) and `persistCaseId()` (phase 3, which commits the new case ID), the old `engineCaseId` is still on the entity. If the old case completes in this window, the discrimination matches the wrong case and corrupts state. The fix was simple: null `engineCaseId` in phase 1 before returning. Any old-case completion during the window sees a null mismatch and is correctly treated as superseded.

The review also flagged that superseded completions were being silently dropped — no ledger entry, no audit record. In a GxP system, a completed safety review has independent audit value regardless of whether the grade has since changed. We added `writeSupersededCompletionEntry` with a distinct `actorRole` so an auditor can see "this review was performed at Grade 3 but was superseded by a Grade 4 case."

One thing this work highlighted: the three-phase pattern (prepare in `@Transactional`, `startCase()` outside any tx, persist in `@Transactional`) creates subtle TOCTOU windows whenever a discrimination field lives on the entity. The null-in-phase-1 technique closes the window for `engineCaseId`, but any future feature that adds a similar discrimination field will need the same treatment.
