---
layout: post
title: "The Transaction That Looked Honest"
date: 2026-08-08
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [transaction-boundary, workitem, dsmb, safety-signal, error-isolation, quarkus]
---

A batch job detects a safety signal. It should create a WorkItem for DSMB review. The obvious implementation: detect the signal, persist the record, create the WorkItem, send a Slack notification — all in one transaction. Clean, atomic, done.

Three things wrong with that.

The aggregation job already existed — `TrialSafetyAggregationJob` scans every active trial on a 24-hour cadence, computing grade-threshold rates and cross-site event clusters. When it finds a signal, it persists a `TrialSafetySignal` record, stores a CBR case, and fires a CDI event for the ledger audit trail. What it didn't do was create a WorkItem for human review. That's what #146 adds.

The straightforward approach puts WorkItem creation inside `upsertSignalRecord()`, which already runs in a `REQUIRES_NEW` transaction. WorkItem goes in, `workItemId` gets stamped on the signal record, notification fires, everything commits together.

The design review killed it. Three anti-patterns in one method call:

First, no error isolation. If `WorkItemService.create()` throws — capability validation, exclusion policy, constraint violation — the entire transaction rolls back. The signal record is gone. The spec claimed "signal record persisted even if WorkItem creation fails." The transaction boundary made that claim false.

Second, phantom notifications. `Connector.send()` fires a Slack message inside the transaction. If the transaction later rolls back on flush, the DSMB gets a notification for a WorkItem that never existed.

Third, pool exhaustion. The Slack HTTP call holds an Agroal DB connection for the duration of the external request. Under concurrent load, every connection in the pool waits on Slack's response time. This is the same anti-pattern that forced the `DurableSponsorNotifier` rewrite months ago.

The fix is structural: two-phase transaction split. Phase 1 persists the signal record in its own `REQUIRES_NEW` and returns. Phase 2, in a separate `REQUIRES_NEW` with a try-catch, creates the WorkItem and stamps `workItemId` back on the signal record. The notification fires only after Phase 2 commits. If Phase 2 fails, the signal record survives — the next 24-hour run picks it up and tries again.

Idempotency comes from `TrialSafetySignal.workItemId`. If a non-terminal WorkItem already exists for this signal, skip. If the previous WorkItem completed or expired, create a fresh one — the signal persists and warrants a new review. A unique constraint on `(trial_id, signal_type, tenant_id)` prevents duplicate signal records under concurrent runs.

The notification follows the `DefaultSafetyOfficerNotifier` pattern: `@All List<Connector>` resolved by connector ID, `ConnectorMessage` with destination/title/body, and a ledger entry for every attempt — success or failure. The ledger entry matters for GCP audit: the fact that the DSMB was or wasn't notified must be independently verifiable.

Two things this uncovered that aren't in any documentation. The aggregation job queries entities by a tenant ID from a config property, not from `CurrentPrincipal` — it runs on a Quartz thread with no request context. Integration tests that seed data with `principal.tenancyId()` get a silent mismatch: the query returns empty, the signal is never detected, and `findSignal()` returns null with no error. The fix is to stamp `"default"` explicitly on test entities.

The job was also excluded from `@QuarkusTest` via `quarkus.arc.exclude-types` — the standard pattern for preventing schedulers from firing during tests. But the integration test needs the real bean. Removing the exclusion and setting a 9999-hour interval is the workaround: the bean is injectable, the scheduler never fires.

This is the last piece of the batch safety signal pipeline. Layer 6 handles the acute path — real-time Grade 4+ signals via engine blackboard flags. The aggregation job handles the slow-burn: Grade 3 accumulation, cross-site event clustering, statistical trends invisible to event-driven detection. Both now route to human review.
