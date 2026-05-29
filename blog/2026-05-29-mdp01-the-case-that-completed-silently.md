---
layout: post
title: "The Case That Completed Silently"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [quarkus, casehub-engine, cdi, async-observer, three-phase-pattern]
---

The engine SNAPSHOT update from last week deleted `CasehubWorkloadProvider` — the CDI bridge that made `WorkloadProvider` injectable in `@QuarkusTest`. The deletion made sense architecturally: it was an adapter from the work module's human-task routing into the engine's agent routing, and the engine PR that removed it also replaced the underlying routing SPI entirely. What it left behind was a CDI startup failure for every integration test in clinical. No `JpaWorkloadProvider` (excluded deliberately — it requires a full schema) and no bridge. We added a `StubWorkloadProvider` — `@DefaultBean`, returns zero — which is honestly the right shape for tests anyway. External routing decisions have no business in a unit environment.

More interesting was what we found when the lifecycle tests actually started running.

After refactoring `AeEscalationCaseService` into the three-phase `startCase()` pattern — the same pattern `TrialActivationService` uses, now applied a second time for AE escalation and a third for IRB deviation — we added an assertion that `ae.escalationStatus` reaches `COMPLETED` when the engine case finishes. It should: `AeEscalationListener` observes `CaseLifecycleEvent` and writes the status on `CaseCompleted`. The await timed out every time.

The case was unambiguously in `CaseStatus.COMPLETED` — we could poll the repository and see it. The CDI event just never arrived.

This took a while to understand. The sequence in the engine is: goal satisfied → `GoalReachedEventHandler` fires `GoalReached` via CDI async → reactive chain calls `evaluateCompletion()` → Vert.x event bus publish → a separate handler fires `CaseCompleted`. Two async hops. In a single-threaded test environment with in-memory persistence, the second hop either doesn't fire at all or fires after the test's assertion window closes. `GoalReached` fires at the first hop and arrives reliably.

The fix: accept both event types in the observer. Since `GoalReached` fires once per goal (a two-goal case fires it twice), we extracted the status write to `AeStatusUpdater` with `@Transactional(REQUIRES_NEW)` and an idempotency guard. `REQUIRES_NEW` ensures the write commits even if the outer observer's broader transaction fails — which matters because ledger writes happen in the same method and we don't want a ledger failure to leave `escalationStatus` stuck at `REQUESTED`.

There's a filed engine issue for the root cause. For now, the observer is honest about what it actually observes.

On `IrbCommitteeAssignmentPolicy`: the original issue offered two options — a `@ConfigProperty` default or a full SPI mirroring `DeviationResponsePolicy`. I wanted the SPI. The config approach would have a single string with no type safety, no per-site routing, and no way to test the assignment logic independently. The SPI contract is `evaluate(IrbCommitteeContext) → IrbCommitteeAssignment` and the default returns `"irb-committee"` — identical behaviour, but the callsite is now explicit about the policy being extensible. One caveat: the YAML `candidateGroups` field is still a hardcoded string. Making that dynamic requires the engine to support context-evaluated `candidateGroups` in `humanTask` bindings, which it currently doesn't. Filed engine#387.
