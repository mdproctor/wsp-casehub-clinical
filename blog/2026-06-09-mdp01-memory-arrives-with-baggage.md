---
layout: post
title: "Memory arrives with baggage"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [casehub-platform, memory, multi-tenancy, snapshot]
---

CaseMemoryStore integration requires one thing before the first write can happen: a tenantId. No tenantId, no tenant isolation, no point writing facts that can't be scoped to the right trial operator. So the two issues — multi-tenancy foundation (#69) and the memory integration (#33) — had to land together.

The multi-tenancy part is mechanical: `tenant_id NOT NULL DEFAULT 'default'` across six domain tables (V116), the field on all six entities, and three CDI events gaining a `String tenantId` component. Four REST resources inject `CurrentPrincipal` and stamp the entity at persist time. `AdverseEventService` does the same before persisting the AE — it sees the HTTP request context, so injecting `CurrentPrincipal` is clean there.

The interesting constraint comes later. `AeEscalationListener`, `PiResponseListener`, and `DeviationExpirer` all need to write to memory, but none of them run in a request context. `AeEscalationListener` is a CDI `@ObservesAsync` handler; `DeviationExpirer.expireOne()` runs on a Quartz thread. Both lack an active CDI request scope, so `@RequestScoped` proxies throw `ContextNotActiveException` when accessed. The fix is to never inject `CurrentPrincipal` into `ClinicalMemoryService` at all — tenantId flows as an explicit `String` parameter from the caller, sourced from wherever the caller has it: the entity field, a case context key set earlier in the lifecycle. All writes in non-request-context paths are wrapped in try/catch and degrade to WARN until platform#79 fixes `MemoryPermissions.assertTenant` for these contexts. The case runs exactly as before in the interim.

One design question came up during `querySiteContext`: should the 180-day window for recent timeline breaches be applied in Java after the query, or pushed into the store? The natural instinct is post-filter — query 20 entries, filter to the last six months. But `MemoryQuery.forEntity()` returns 20 results in `CHRONOLOGICAL` order (oldest first). A site with 30 entries gets its 20 oldest returned; the recent breaches at positions 21–30 are invisible to the filter. Using `.withSince(Instant.now().minus(180, DAYS)).withLimit(50)` in the query pushes the time window into the store, which is where it belongs. The context object just counts what comes back.

The SNAPSHOT broke mid-session. `casehub-ledger` updated `LedgerEntryRepository` to add a `String tenantId` second parameter to every method — `save`, `findBySubjectId`, `findLatestBySubjectId`, all of them. `casehub-ledger-memory`'s `InMemoryLedgerEntryRepository` was not updated simultaneously; it still implements the old one-arg signatures. Selecting it in tests causes `AbstractMethodError` when production code calls the new two-arg API. And `casehub-qhorus`'s `LedgerWriteService` had been compiled against the old interface — it calls `findLatestBySubjectId(UUID)`, which no longer exists on the runtime interface, giving `NoSuchMethodError` in any test that goes through `ProtocolDeviationService.reportDeviation()`.

The bridge is `ClinicalTestLedgerRepository`, a test-scope class that implements the new two-arg interface and ignores the tenantId parameter entirely (single-tenant test context). It replaces `InMemoryLedgerEntryRepository` in `selected-alternatives`. The production ledger writers now pass `"default"` as tenantId until real per-entity propagation follows. Not elegant, but contained — tracked in #74 for removal once the snapshots are consistent again. The five tests that still fail all go through qhorus's `LedgerWriteService`, which requires a qhorus snapshot update to fix, not a clinical change.

What's actually working: patient AE history accumulates across cases and flows into the engine's `initialContext` before the first agent runs. A JQ binding can now key off `.patientContext.hasPriorGrade3OrAbove` to route a Grade 3 patient with prior history differently from a first presentation. Site compliance context follows the same path. The infrastructure is there — platform#79 is what activates it for real workloads.
