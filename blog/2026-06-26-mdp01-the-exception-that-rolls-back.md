---
layout: post
title: "The Exception That Rolls Back Behind Your Back"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [jta, gdpr, transactions, quarkus]
---

This branch closed three issues on a single pass — a tenancy exception mapper (#89), a listener transaction fix (#87), and a GDPR patient-scoped erasure endpoint (#79). The mapper and the listener fix were straightforward. The GDPR work is where things got interesting.

## The JTA gotcha nobody warns you about

The spec review for #79 caught something I hadn't considered. `ConsentWithdrawalService.withdraw()` threw `ConsentAlreadyWithdrawnException` when called for an already-withdrawn enrollment. The new `GdprErasureService` needed to iterate all enrollments for a patient and withdraw each. The obvious pattern: catch the exception in the loop, skip, continue.

It doesn't work.

`ConsentAlreadyWithdrawnException` extends `RuntimeException`. When thrown from a `@Transactional(REQUIRED)` method that has joined an outer transaction, the Narayana interceptor calls `setRollbackOnly()` on the outer transaction *before* re-throwing. By the time your catch block executes, the transaction is already doomed. Every subsequent JPA write fails silently. The batch is lost.

The catch block runs. The exception is handled. The transaction is dead. There's no warning, no error until commit time. I've written hundreds of JTA methods and never hit this because I'd never composed a REQUIRED method inside a loop where the expected outcome was an exception.

The fix was to stop using exceptions for expected outcomes. `withdraw()` now returns `WithdrawalResult.WITHDRAWN` or `WithdrawalResult.ALREADY_WITHDRAWN`. No exception, no rollback marking, no dead transaction. The REST endpoint checks the return value and maps `ALREADY_WITHDRAWN` to 409 — same HTTP behaviour, correct JTA semantics.

`ConsentAlreadyWithdrawnException` is deleted. Good riddance — using exceptions for "already done" was always the wrong model. Return types compose; exceptions poison.

## Three listeners, three different transaction designs

The listener fix (#87) was more surgical than it sounds. All three `CaseLifecycleEvent` listeners had `@Transactional` on their `@ObservesAsync` observer methods, holding a JDBC connection during a reactive `.await()` call. Removing the annotation was step one. Step two was verifying each listener's write transaction boundaries still worked.

`SusarOversightListener` was trivial — its only write was already `REQUIRES_NEW`. The outer transaction was doing nothing.

`AeEscalationListener` needed its ledger writer promoted to `REQUIRES_NEW`. This actually *strengthened* the FDA gap design: the status commit and ledger commit are now independently durable, where before the ledger commit depended on the outer transaction not rolling back.

`ProtocolAmendmentListener` was the interesting one. The entity state updates and the ledger write had to remain atomic — same transaction, all-or-nothing. We extracted `ProtocolAmendmentStatusUpdater` with a single `REQUIRES_NEW` method that owns the complete switch (PROCEED, HALT, REFER_TO_DSMB, and the default error case). The spec review caught that splitting the switch between listener and updater would fragment the dispatch contract. The listener is now five lines: filter, extract, delegate.

## A dependency break as a bonus

The branch also inherited a pre-existing compilation failure: `WorkerResult`, `PlannedAction`, `Worker`, and `Capability` had moved from `casehub-engine-api` to `casehub-worker-api` in a recent engine SNAPSHOT. The types vanished from their old package with no compile error on the engine side — clinical just stopped building. `ActionRiskClassifier.classify()` also gained a `ClassificationContext` second parameter. `PlannedAction` now enforces non-null `actionType` at construction, which killed one test that was exercising an impossible condition.

The root cause of the #87 perf issue — `CaseLifecycleEvent` carrying no case context, forcing every listener to query the reactive repository — is filed as engine#571. That's the real fix. What we did here removes the symptom; the engine change eliminates the cause.
