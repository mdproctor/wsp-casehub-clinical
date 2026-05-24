---
layout: post
title: "What the Test Suite Isn't Telling You"
date: 2026-05-24
type: phase-update
entry_type: note
subtype: log
projects: [clinical]
tags: [quarkus, CDI, casehub-engine, production]
---

When we ran the first production `quarkus:build` on casehub-clinical, the test suite was entirely green. 115 tests, no failures. The build gave us 43 CDI deployment problems.

Two separate issues. Both non-obvious.

## The memory store masks a missing production dependency

`casehub-engine` needs JPA implementations for its repository SPIs — `CaseInstanceRepository`, `EventLogRepository`, `SubCaseGroupRepository`, `CaseMetaModelRepository`. In tests, `casehub-engine-persistence-memory` satisfies all of them. It's test-scope only.

The production equivalent is `casehub-engine-persistence-hibernate`. It wasn't on the classpath. Nothing in the test suite can tell you this — the memory store covers every injection point cleanly. You only find out when `quarkus:build` runs and the JPA implementations simply aren't there.

Adding the dependency brought the failure count from 43 to 33.

## Reactive suppression belongs in production too

The remaining 33 failures were all unsatisfied reactive service beans: `ReactiveChannelService`, `ReactiveMessageStore`, `ReactiveInstanceService`. These come from casehub-qhorus and casehub-ledger transitive dependencies.

No reactive datasource was configured anywhere. The reactive driver wasn't on the classpath. Quarkus still indexes reactive beans from transitive JARs during augmentation, tries to wire them, and fails when their injection points have nothing to satisfy.

In test `application.properties`, `quarkus.datasource.reactive=false` suppresses this entire category. In production `application.properties`, it wasn't set. Tests completely clean; `quarkus:build` produces 33 failures that look like missing beans rather than a datasource configuration problem.

Both failure categories produce the same output: `UnsatisfiedResolutionException` with a type name. Neither error message points at a missing Maven dependency or a missing datasource flag.
