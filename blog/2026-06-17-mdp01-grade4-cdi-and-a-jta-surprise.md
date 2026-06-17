---
layout: post
title: "Grade 4, a CDI draw, and a JTA surprise"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [quarkus, cdi, jta, ind-reporting, ctcae, regulatory]
---

Three issues today — XS, S, S — and I expected them to be quick. Grade 4 was. The other two had interesting surprises underneath.

**Grade 4 IND path.** 21 CFR 312.32(c)(1)(i) requires 7-day expedited reporting for unexpected ADRs that are *fatal or life-threatening*. Fatal is Grade 5. Life-threatening is Grade 4. Clinical had Grade 5 wired and Grade 3 (15-day, a different subclause), but Grade 4 was sitting there correctly excluded from the existing tests. Three locations needed to change atomically per PP-20260617-3167e3: the `isIndReportable()` predicate, `indReportingWindow()`, and the compliance supplement's CFR citation. Straightforward once you know the rule — the switch case idiom for Grade 4 and 5 sharing a 7-day deadline is clean Java:

```java
case GRADE_4, GRADE_5 -> Duration.ofDays(7);
```

Code review caught stale Javadoc in five adjacent files that still described Grade 5 only. Easy fix, worth catching.

**SusarOversightLifecycleTest.** The test had been accepting `REQUESTED or COMPLETED` as valid final states because the Quartz-gated SUSAR function worker never runs in `@QuarkusTest` — the scheduler jar isn't Jandex-indexed, so `NoOpWorkerExecutionManager` wins and no gate is ever created. The fix was already documented in GE-20260614-b97659: drive the two observable behaviors directly. We inject `SusarGateDecisionListener` and `SusarOversightListener`, call `onApproved()` and `onCaseLifecycle()` directly from the test, and the full approved path runs synchronously. The test now asserts `COMPLETED` unconditionally. `SusarOversightApprovedLifecycleTest` had been doing this all along — the original test was just lagging behind the pattern.

**The CDI ambiguity.** `mvn install` had been failing during production augmentation with 40 `AmbiguousResolutionException` errors for `CurrentPrincipal`. I assumed it was `MockCurrentPrincipal` versus `FixedCurrentPrincipal` — the title of issue #85 even said so. Wrong.

When we traced the actual error, three beans were competing:

- `MockCurrentPrincipal @DefaultBean @ApplicationScoped` — from `casehub-platform`, the fallback mock
- `QhorusInboundCurrentPrincipal @ApplicationScoped` — from `casehub-qhorus`, reads the X-Tenancy-ID header
- `TenantScopedPrincipal @RequestScoped` — from `casehub-work`, reads from a `TenantHolder` thread-local

`@DefaultBean` is supposed to suppress itself when a non-default implementation is present. It does — but only when *one* non-default bean exists. With two competing non-default beans, `@DefaultBean` is suppressed, but the two winners still conflict with each other. CDI doesn't resolve that. The error output lists all three as ambiguous candidates, which made it look like a three-way fight, but the real issue was the two-way draw between `QhorusInboundCurrentPrincipal` and `TenantScopedPrincipal`.

`QhorusInboundCurrentPrincipal` is the right bean for clinical — explicitly designed to displace `@DefaultBean` in consumer repos, per qhorus#276. `TenantScopedPrincipal` belongs in casehub-work's own deployment, not in every repo that depends on casehub-work. It should be `@Alternative`. I filed `casehubio/work#268` for the proper fix. For now, `%prod.quarkus.arc.exclude-types=io.casehub.work.runtime.service.TenantScopedPrincipal` in clinical's production `application.properties` resolves it.

**The JTA surprise.** One other thing surfaced while closing the branch. `AeEscalationLifecycleTest` had been failing intermittently in the full suite — passing alone, failing when the whole suite ran. The test creates an `AdverseEvent` in `@BeforeEach` without stamping `ae.tenantId`, so it gets the H2 column default `"default"`. `FixedCurrentPrincipal.tenancyId()` returns a UUID. When `ClinicalMemoryService.queryPatientContext()` runs, `InMemoryMemoryStore.query()` calls `MemoryPermissions.assertTenant()`, detects the mismatch, and throws `SecurityException`. `ClinicalMemoryService` catches it — you can see the `WARN: returning empty` in the log — but the test still errors. The exact JTA mechanism was hard to pin down precisely; what we know is that stamping `ae.tenantId = principal.tenancyId()` in `@BeforeEach` eliminated the exception entirely and the test stabilised. Added a protocol for it: PP-20260617-318449.

The work#268 issue is the loose end. If `TenantScopedPrincipal` becomes `@Alternative`, clinical's `exclude-types` entry comes out. Until then, every new casehub-clinical deployment needs to know to add it — which is exactly what CLAUDE.md is for.
