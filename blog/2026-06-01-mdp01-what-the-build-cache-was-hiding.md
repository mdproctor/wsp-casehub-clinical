---
layout: post
title: "What the Build Cache Was Hiding"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [quarkus, cdi, engine, testing]
---

I brought Claude in to run the full build and see where clinical stood. Twelve
failures, three distinct causes. Two were quick: the engine had a new
`findByUuid(UUID, String tenancyId)` signature, and the `pi-oversight` channel
was rejecting EVENT messages because qhorus#153 had shipped and nobody had
updated the allowed types. We fixed both and moved on.

The CDI puzzle took longer.

`IrbCommitteePolicySpiTest` was failing to start Quarkus. The test used
`@TestProfile` with `getEnabledAlternatives()` returning only the test IRB
policy bean. I'd assumed `getEnabledAlternatives()` was additive — that it
would add the test bean on top of whatever `quarkus.arc.selected-alternatives`
had configured in `application.properties`. It isn't. It replaces the entire
set. The moment the profile returned `Set.of(TestIrbCommitteeAssignmentPolicy.class)`,
`MemoryPlanItemStore` was no longer selected, and `HumanTaskRecoveryService`
started trying to query `WorkAdapterPlanItemEntity` — which doesn't exist in
the test Hibernate context.

The fix was clean: rewrite the test to use `@InjectMock` on the SPI field.
No profile, no alternatives manipulation, standard config left intact.

That rewrite invalidated the Quarkus augmentation cache. Any source change
triggers a full re-augmentation, and when Quarkus rebuilt the CDI graph from
scratch it surfaced something that had been hiding: `MockGroupMembershipProvider`
(from `casehub-platform`) and `NoOpGroupMembershipProvider` (from `casehub-work`)
are both `@DefaultBean @ApplicationScoped` implementations of
`GroupMembershipProvider`. Two `@DefaultBean` beans of the same type cause
`AmbiguousResolutionException` at augmentation time.

I tried the obvious fix: add `io.casehub.platform.mock.MockGroupMembershipProvider`
to `quarkus.arc.exclude-types` in both `application.properties` files. Nothing
happened. The bean remained registered. Glob patterns. Nothing. Other beans on
the same exclusion list were suppressed correctly — this one wasn't.

Claude traced the reason. The `casehub-platform` JAR ships `META-INF/jandex.idx`.
Beans from Jandex-indexed JARs are processed at a different augmentation phase
than the one `exclude-types` hooks into. For pre-indexed JARs, the property is
silently ignored — no error, no warning, just no effect. The same property works
fine for application source beans and for JARs without Jandex indices.

The fix uses CDI's own resolution rules. A concrete `@ApplicationScoped` bean
that is NOT `@DefaultBean` suppresses all `@DefaultBean` candidates of the same
type. We added `ClinicalGroupMembershipProvider` to test/support — a single
no-op implementation, test-scoped. CDI picks it as the most-specific candidate
and the ambiguity disappears.

With CDI resolved, the build ran. Twelve new failures, all with the same
`AbstractMethodError`: `CaseHubRuntimeImpl does not define or inherit
'abstract CompletionStage startCase(CaseDefinition, Object)'`. The
`CaseHubRuntime` interface had a new method. The implementation didn't have it.

I checked the JAR timestamps in `~/.m2`. The `casehub-engine-api-0.2-SNAPSHOT.jar`
had a June 1 timestamp — 01:41 that morning. `casehub-engine-0.2-SNAPSHOT.jar`
was from 00:58. Both newer than anything from our previous session. IntelliJ
had been building sibling projects in the background. When those builds finished,
`mvn install` pushed fresh SNAPSHOT JARs into the local cache. Maven resolved
them automatically on the next build. The API JAR updated; the implementation
JAR was from an earlier build of the same cycle and didn't implement the new
method.

That's not a clinical problem to fix. We documented it in #53 and committed
what we had. Twelve tests still failing, all against the same engine
incompatibility. Everything else is green.
