---
layout: post
title: "The Global Alternative"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [quarkus, cdi, alternative, test-coverage, signaling]
---

Three coverage gaps left over from the engine SPI cleanup branch. The issues
were clear enough going in: add lifecycle assertions to the grade4 test (it
had fewer checks than grade3 despite being the more serious path), prove that
`IrbCommitteeAssignmentPolicy` is actually invoked rather than just present,
and add grade4/5 signaling coverage for `AeEscalationCaseService`.

\#44 was mechanical — three `assertThat` lines mirroring what the grade3 test
already had: `REQUESTED` synchronously after Phase 1, `engineCaseId` non-null
after the engine case starts, `COMPLETED` after `AeEscalationListener` fires.
The grade3 pattern existed; grade4 just hadn't been brought up to the same
standard.

\#43 took longer. The existing test `irb_approval_committeeId_matches_policy_default`
asserted `approval.committeeId == "irb-committee"`. That only proves the hardcoded
default string is there — it doesn't prove `committeePolicy.evaluate()` was ever
called. If someone wired the policy injection point incorrectly and the call was
bypassed entirely, the assertion would still pass. I wanted a test that fails when
the policy is not called: an `@Alternative` implementation returning a different
committee ID, with the assertion checking that different value.

`@QuarkusTestProfile.getEnabledAlternatives()` is the right mechanism. The alternative
bean gets `@Alternative @ApplicationScoped` and the profile activates it. I added
`@Priority(1)` as well — it seemed harmless, a ranking hint for when multiple
alternatives compete. It wasn't harmless. In CDI 2.0, `@Priority` on an `@Alternative`
is a global activation flag, not a priority ranking. The alternative became active
across every `@QuarkusTest` in the build. `IrbGateLifecycleTest` — which has no
profile and expects the `@DefaultBean` — started seeing `"test-irb-committee-xyz"`
where it expected `"irb-committee"`. No exception, no helpful error, just the wrong
value in an unrelated test.

Once that clicked, the fix was one-line: remove `@Priority`. The profile's
`getEnabledAlternatives()` handles activation on its own; `@Priority` is unnecessary
and dangerous in this context.

\#42 was a design question. `signalTrialGrade4Active` was private in
`AeEscalationCaseService`, called after Phase 3 when a Grade 4+ AE starts. The
coverage gap: Phase 1 of that service calls `AdverseEvent.findById()` — Panache static,
breaks `@InjectMocks`. The lifecycle tests exercise the full flow but don't create a
`TrialSite` or `ClinicalTrial`, so `trialCaseLookup.findTrialEngineCase()` returns null
and the signal never fires. Three options: `quarkus-panache-mock`, extending the
lifecycle test with a real trial setup, or moving the method.

Moving it was right. `TrialSafetySignalService` already owned the clear path —
`onAeEscalationCompleted` clears `grade4Active.<siteId>` when an AE escalation case
closes. Adding `signalGrade4Active(siteId)` to the same service meant all grade4
blackboard flag operations live in one place. `AeEscalationCaseService` sheds its
`CaseHubRuntime` and `TrialCaseLookup` injections entirely. The new method is pure
Mockito testable alongside the existing tests. We added `@InjectSpy TrialSafetySignalService`
to the lifecycle test to verify the routing decision without needing a real trial in the
database — grade3 should never touch the signal method, grade4 should call it exactly once.

During code review, Claude flagged that `IrbCommitteePolicySpiTest.irb_approval_reflects_alternative_policy_committee_id` was annotated `@Transactional` at the method level, which wraps `startCase().join()` inside an open transaction — exactly the Agroal pool deadlock pattern the three-phase separation exists to prevent. In the test environment it doesn't deadlock because the in-memory engine doesn't use the database pool, but it's architecturally wrong and would fail on a real persistence backend. A `@Transactional` helper method for the Panache read fixed it.

136 tests passing, three more than the branch started with. The CDI 2.0 rule about `@Priority` on alternatives is now in CLAUDE.md under Ecosystem Conventions.
