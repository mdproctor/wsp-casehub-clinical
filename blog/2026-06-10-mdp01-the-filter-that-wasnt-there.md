---
layout: post
title: "The Filter That Wasn't There"
date: 2026-06-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [tenant-isolation, hibernate, multi-tenancy, panache]
---

V116 added `tenant_id` to all six domain entities in casehub-clinical. Writes were correct from the start — resources stamp `principal.tenancyId()` on creation. Reads were completely unscoped. Any caller who guessed a UUID got the entity, regardless of tenant.

The obvious fix was Hibernate's `@Filter`. Enable it in a `@ServerRequestFilter` at the start of every HTTP request, every query in that request scopes to the current tenant. Clean infrastructure, no per-call-site changes. That was the design I went in with.

It was wrong.

I read the Hibernate 7 source to settle the question. `SingleIdLoadPlan` — Hibernate's mechanism for primary key entity loading — builds its `JdbcSelect` in the constructor, at session factory startup. That SQL is cached and immutable. When `em.find()` fires at runtime, Hibernate executes the pre-built SQL directly and never touches the session's filter state. `@Filter` only affects queries compiled at runtime: HQL, JPQL, Criteria API. `findById()` bypasses all of that.

To use `@Filter` for `findById()`, you'd convert every call to `find("id = ?1", id).firstResult()` — a HQL query that does go through the runtime compiler. Same call-site change count as adding the tenant predicate explicitly, but with the intent hidden in infrastructure. That's worse, not better.

So: static entity helpers.

```java
public static ClinicalTrial findByIdForTenant(UUID id, CurrentPrincipal principal) {
    if (principal.isCrossTenantAdmin()) return findById(id);
    return find("id = ?1 AND tenantId = ?2", id, principal.tenancyId()).firstResult();
}
```

Six entities, same pattern. REST resources and `TrialActivationService` call this. System actors — `DeviationExpirer`, CDI observers, internal service enrichment — keep plain `findById`. They're cross-tenant by design and the Panache call makes that visible at the call site.

A secondary problem surfaced during spec design: write paths were wrong too. `SiteResource.add` was stamping `site.tenantId = principal.tenancyId()`. For a cross-tenant admin adding a site to tenant A's trial, that produces `site.tenantId = "admin-tenant"`. The site becomes invisible to tenant A — reachable by no one. The fix: derive from the parent. `site.tenantId = trial.tenantId`. The trial was already loaded by `findByIdForTenant`, so the actual tenant is right there.

`AdverseEventService` had the same issue. It was stamping `ae.tenantId = principal.tenancyId()` — the only use of `CurrentPrincipal` in that service. The service now loads the enrollment once at the top of `reportAdverseEvent` and derives both `siteId` and `tenantId` from that single instance. Cleaner than the two private resolver methods it replaced.

Testing the write-path invariant requires a non-obvious structure. An isolation test (wrong tenant → 404) only proves the read guard works. It doesn't prove what tenant was actually stamped. The invariant test does it differently: create as cross-tenant admin (bypass on), confirm default tenant can read it (200), then confirm admin's own tenant without bypass cannot (404). The negative assertion is the structural proof that the child inherited the parent's tenant, not the caller's.

`Hibernate @TenantId` was the other approach I considered. It breaks `DeviationExpirer` — which queries all tenants as a scheduler — and `@ObservesAsync` observers, which run without request scope. The resolver is per-session, not per-query. No bypass mechanism exists for system actors.
