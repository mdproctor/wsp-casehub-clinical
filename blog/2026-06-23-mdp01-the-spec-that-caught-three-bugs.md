---
layout: post
title: "The Spec That Caught Three Bugs Before Any Code Was Written"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [security, oidc, rbac, quarkus, spec-review]
series: issue-88-wire-platform-oidc
---

The spec for clinical#88 — wiring `casehub-platform-oidc` and `@RolesAllowed` across nineteen endpoints — went through four review rounds. The implementation went through zero rework. That ratio is the point.

## The wrong property that looked right

`quarkus.security.jaxrs.deny-unannotated-endpoints=true` does not exist. The `quarkus.security.jaxrs` namespace is real — it contains `deny-jax-rs`, from `JaxRsSecurityConfig` in the REST common deployment jar. The property I specified looks plausible, passes a plausibility check against the namespace, and Quarkus accepts it without complaint at startup. Zero warnings. Zero errors. The security backstop is silently absent while appearing to be configured.

The actual property is `quarkus.security.deny-unannotated-members`, defined in the security deployment `SecurityConfig` interface. I found it by decompiling the `quarkus-security-deployment-3.32.2.jar` — the only way to confirm, because the property naming doesn't follow the pattern you'd predict from the namespace layout.

Even once you have the right property, its behaviour is narrower than the name implies. `DenyUnannotatedPredicate` — verified from bytecode — returns `true` only when a class has no class-level security annotation AND at least one method already has one. A new resource class with zero annotations anywhere is not covered. The protection catches the most common mistake (forgetting `@RolesAllowed` on a new method to an existing class) but does not catch the second most common one (adding a new class with no annotations at all).

## The layer that destroys the application

The first-pass spec specified `quarkus.http.auth.permission.*.policy=deny` on `/*` as a catch-all. HTTP permission policies operate at the Vert.x filter layer — before JAX-RS routing, before CDI security interceptors. A `deny` policy at that layer rejects every request regardless of authentication status, role, or `@RolesAllowed` annotation. The application returns 403 to everything. `@RolesAllowed` becomes unreachable dead code.

It also breaks dev mode. `quarkus.security.auth.enabled-in-dev-mode=false` suppresses the CDI authorization controller, not the Vert.x HTTP filter. Two security layers with distinct suppression mechanisms — and the documentation doesn't make the distinction obvious.

## GCP topology as a first-principles exercise

The RBAC mapping started as "which roles should access which endpoints?" and improved significantly when I stopped copying patterns and started reading the regulations.

ICH E6(R3) §5.17 assigns the sponsor aggregate regulatory reporting, not individual site-level data entry. That means SPONSOR does not belong on `POST /adverse-events`, `POST /patients`, `POST /screen`, or `POST /deviations`. The sponsor receives notifications through `SponsorNotificationDeliveryService` — they monitor and receive reports, they don't enter clinical data.

GDPR Art.17 consent withdrawal is PI authority, not sponsor authority. The PI is the data processor at the site. The sponsor organisation receives notification of the withdrawal, but does not execute it.

These distinctions matter because they're structural: a sponsor with site-level write access is an architectural misstatement about who has regulatory authority over site data. Getting the topology wrong is not a permission gap to be fixed later — it's a compliance claim that is false from the start.

## The exception mapper that can't be tested

`OidcCurrentPrincipal.tenancyId()` throws `IllegalStateException` when a JWT lacks the `tenancyId` claim. The spec proposed a string-matching `ExceptionMapper<IllegalStateException>` with a `throw e` re-throw for non-matching cases. Two problems: `IllegalStateException` is thrown by half of Quarkus and Hibernate, making the string match fragile; and `throw e` inside `toResponse()` is undefined at the JAX-RS spec boundary.

The deeper problem: it can't be tested the way you'd expect. In `@QuarkusTest`, `FixedCurrentPrincipal` is the active CDI `CurrentPrincipal` via `selected-alternatives`. `OidcCurrentPrincipal.tenancyId()` is never called — `@TestSecurity` controls the Quarkus security layer, but the business logic identity goes through `FixedCurrentPrincipal`. An HTTP test asserting 400 would pass for the wrong reason.

The right fix is upstream: `MissingTenancyClaimException` in `casehub-platform-oidc`, typed mapper in clinical. Filed as platform#111 and deferred to clinical#89.

## What shipped

Four commits. `ClinicalGroups` in `api/` with four GCP-derived groups — `trial-sponsor`, `principal-investigator`, `trial-coordinator`, `safety-monitor`. `@RolesAllowed` on every endpoint, `@TestSecurity` on every HTTP test. `RbacBoundaryTest` covering every denied role-endpoint combination: MONITOR denied all writes, COORDINATOR denied governance and trial management, INVESTIGATOR denied sponsor-only operations, SPONSOR denied site-level data entry. Twenty-seven boundary tests, all green.

The spec took four rounds to converge. The implementation took one pass. That's the correct allocation of effort for a security-critical change.
