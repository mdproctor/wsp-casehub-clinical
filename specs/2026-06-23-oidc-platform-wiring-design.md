# Design: Wire casehub-platform-oidc — RBAC for Clinical REST Resources

**Issue:** casehubio/clinical#88
**Date:** 2026-06-23
**Branch:** issue-88-wire-platform-oidc

---

## Context

`casehub-platform-oidc` ships `OidcCurrentPrincipal @RequestScoped`, which implements `CurrentPrincipal` backed by `SecurityIdentity` and `JsonWebToken`. It provides `tenancyId()`, `actorId()`, and `groups()` from the JWT. Adding it to the classpath activates `@RolesAllowed` enforcement across Quarkus REST resources.

Without it on the classpath, `CurrentPrincipal.groups()` always returns an empty set and `@RolesAllowed` annotations are inert.

Reference implementation: `casehubio/life#40` (life repo, commits `42064e3` + `b727392`). Clinical follows the same wiring pattern with clinical-specific groups and endpoint topology.

---

## Permission Topology

### Groups

```java
// api/src/main/java/io/casehub/clinical/api/ClinicalGroups.java
public final class ClinicalGroups {
    public static final String SPONSOR      = "trial-sponsor";
    public static final String INVESTIGATOR = "principal-investigator";
    public static final String COORDINATOR  = "trial-coordinator";
    public static final String MONITOR      = "safety-monitor";
    private ClinicalGroups() {}
}
```

### Regulatory basis

| Group | GCP / FDA role | Regulatory basis |
|-------|---------------|-----------------|
| `trial-sponsor` | Sponsor (IND holder) | ICH E6(R3) §5.1 — takes responsibility for trial initiation and management |
| `principal-investigator` | PI (site conductor) | ICH E6(R3) §4.1, Form FDA 1572 — personally accountable for site conduct |
| `trial-coordinator` | CRC (operational staff) | ICH E6(R3) §4.1.5 — acts under PI supervision; no independent regulatory authority |
| `safety-monitor` | DSMB member / Safety Officer | ICH E6(R3) §5.18.3 — independence mandate; zero write access by regulatory design |

### No RBAC-differentiated thresholds in `ClinicalActionRiskClassifier`

`ClinicalActionRiskClassifier` gates are driven by CTCAE grade, unexpected/suspected flags, and 21 CFR 312.32 reporting requirements — objective clinical severity signals independent of who triggered them. A sponsor cannot negotiate away a Grade 4 AE's 24-hour SLA. No role-based threshold overrides. This is intentionally a stronger compliance story than life's financial-threshold model.

---

## Endpoint Annotation Map

> ⚠️ **SPEC REVIEW — PAY SPECIAL ATTENTION HERE**
>
> The per-endpoint role assignments below were derived from GCP (ICH E6(R3)) and FDA (21 CFR Part 312) regulatory role definitions. Each assignment is a claim about what that regulation requires or permits. A reviewer should ask for every row:
>
> 1. Does the regulatory basis cited actually require this restriction?
> 2. Is there a GCP scenario where the denied role legitimately needs this access?
> 3. Are there missing endpoints (new ones added since this spec was written) not in this table?
>
> Particular rows to scrutinise:
> - `POST /withdraw-consent` — PI only, SPONSOR excluded. GDPR Art.17 erasure is executed by the data processor at the site (PI). The sponsor organisation receives notification but does not perform site-level personal data erasure. Correct?
> - `POST /amendments` — COORDINATOR excluded (protocol governance, not operational). Correct?
> - `POST /sites` — INVESTIGATOR excluded. ICH E6(R3) §5.6: the sponsor selects and contracts investigator sites. A PI does not commission their own site. Correct?
> - `POST /trials/activate` — INVESTIGATOR excluded. IND submission is sponsor authority per 21 CFR 312.30. Correct?
> - `POST /patients`, `POST /screen`, `POST /adverse-events`, `POST /deviations` — SPONSOR excluded. ICH E6(R3) §5.17 assigns the sponsor aggregate regulatory reporting, not individual site-level data entry. Site-level data capture is PI/coordinator territory. Sponsors receive notifications via `SponsorNotificationDeliveryService`. Correct?

| Resource | Method + Path | Groups |
|----------|---------------|--------|
| `TrialResource` | `POST /trials` | `SPONSOR` |
| | `GET /trials/{id}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `PATCH /trials/{id}/sponsor-config` | `SPONSOR` |
| | `POST /trials/{id}/activate` | `SPONSOR` |
| `SiteResource` | `POST /trials/{trialId}/sites` | `SPONSOR` |
| | `GET /trials/{trialId}/sites/{siteId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| `PatientResource` | `POST .../patients` (enroll) | `INVESTIGATOR, COORDINATOR` |
| | `GET .../patients/{enrollmentId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `POST .../patients/{id}/screen` | `INVESTIGATOR, COORDINATOR` |
| | `POST .../patients/{id}/adverse-events` | `INVESTIGATOR, COORDINATOR` |
| | `GET .../adverse-events/{aeId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `GET .../ledger/verify` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `POST .../withdraw-consent` | `INVESTIGATOR` |
| | `GET .../audit/prov` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `GET .../audit/entries/{entryId}/proof` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| `DeviationResource` | `POST .../deviations` | `INVESTIGATOR, COORDINATOR` |
| | `GET .../deviations/{deviationId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| `ProtocolAmendmentResource` | `POST .../amendments` | `SPONSOR, INVESTIGATOR` |
| | `GET .../amendments/{amendmentId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |

**Total: 19 endpoints across 5 REST resources.**

---

## CDI Wiring

### The platform gap (casehubio/platform#111)

`OidcCurrentPrincipal` is `@RequestScoped` with no `@Alternative` or `@Priority`. The platform contract — that adding `casehub-platform-oidc` to the classpath automatically displaces all other `CurrentPrincipal` implementations — is not fulfilled. Without `@Alternative @Priority(100)`, `OidcCurrentPrincipal` and `QhorusInboundCurrentPrincipal @ApplicationScoped` both exist as non-default, non-alternative `CurrentPrincipal` implementations and Quarkus ArC throws `AmbiguousResolutionException` at production augmentation.

The correct fix is `@Alternative @Priority(100)` on `OidcCurrentPrincipal` in casehub-platform (tracked: casehubio/platform#111). Until that ships, clinical uses the local workaround below.

### Resolution (local workaround pending platform#111)

Exclude `QhorusInboundCurrentPrincipal` from production. `OidcCurrentPrincipal` becomes the sole active `CurrentPrincipal`. Tenant identity moves from the raw `X-Tenancy-ID` header to the authenticated JWT `tenancyId` claim — the correct direction for a system with real auth.

`OidcCurrentPrincipal` provides `tenancyId()`, `actorId()`, and `groups()` from the JWT, so no capability is lost.

### Updated `%prod.quarkus.arc.exclude-types`

```properties
# ── CurrentPrincipal resolution (clinical#88) ─────────────────────────────────
# OidcCurrentPrincipal (@RequestScoped, casehub-platform-oidc) is the sole active
# CurrentPrincipal. Tenant identity comes from the JWT tenancyId claim.
#
# QhorusInboundCurrentPrincipal: excluded as a LOCAL WORKAROUND pending
#   casehubio/platform#111 (OidcCurrentPrincipal needs @Alternative @Priority(100)).
#   Remove this exclusion once platform#111 ships.
#
# TenantScopedPrincipal (casehub-work @RequestScoped): excluded — belongs to
#   casehub-work deployments; upstream fix tracked in casehubio/work#268.
#
# CasehubWorkloadProvider: REMOVED from this list — class was deleted in
#   engine#378; exclusion is no longer needed.
#
# Note: MockCurrentPrincipal (@DefaultBean) and MockGroupMembershipProvider
# (@DefaultBean) are NOT excluded here — both are automatically displaced by
# non-default beans already on the classpath (OidcCurrentPrincipal and
# ClinicalGroupMembershipProvider respectively). DefaultTestPrincipal
# (casehub-engine-persistence-memory) is test-scope only and never on the
# production classpath.
%prod.quarkus.arc.exclude-types=\
  io.casehub.qhorus.runtime.identity.QhorusInboundCurrentPrincipal,\
  io.casehub.work.runtime.service.TenantScopedPrincipal
```

---

## Security Posture — Global Deny Policy

`@RolesAllowed` annotations alone do not close the security boundary: any endpoint added without an annotation fails open (unauthenticated access). A global deny policy ensures missing annotations fail closed instead.

Add to production `application.properties`:

```properties
# Health/metrics exempt — must be accessible to load balancer health probes
quarkus.http.auth.permission.health.paths=/q/*
quarkus.http.auth.permission.health.policy=permit
# Catch-all deny — any endpoint without an explicit @RolesAllowed returns 403
# (not 200). This makes missing annotations fail closed.
quarkus.http.auth.permission.default.paths=/*
quarkus.http.auth.permission.default.policy=deny
```

---

## Missing `tenancyId` Claim — ExceptionMapper

`OidcCurrentPrincipal.tenancyId()` throws `IllegalStateException` when the JWT lacks the `tenancyId` custom claim. JAX-RS propagates unhandled `RuntimeException` as 500. A correctly-signed token from a newly-configured provider that lacks the custom claim should produce a clear 400 (bad token configuration), not a 500 with a stack trace.

Add to `runtime/`:

```java
@Provider
public class MissingTenancyClaimExceptionMapper
        implements ExceptionMapper<IllegalStateException> {
    @Override
    public Response toResponse(IllegalStateException e) {
        if (e.getMessage() != null && e.getMessage().contains("tenancyId")) {
            return Response.status(Response.Status.BAD_REQUEST)
                .entity("{\"error\":\"JWT missing required claim: tenancyId\"}")
                .type(MediaType.APPLICATION_JSON)
                .build();
        }
        throw e; // re-throw unrelated IllegalStateExceptions
    }
}
```

Add corresponding test case to `RbacBoundaryTest`: `@TestSecurity` with username and roles but no `tenancyId` claim exercised via a request that hits a `findByIdForTenant` call — expects 400.

---

## OIDC Configuration

### Production `application.properties`

```properties
# ============================================================
# OIDC — casehub-platform-oidc (clinical#88)
# Required env vars (do NOT hardcode values — empty strings bypass ConfigException):
#   QUARKUS_OIDC_AUTH_SERVER_URL   e.g. https://auth.example.com/realms/casehub
#   QUARKUS_OIDC_CLIENT_ID         e.g. casehub-clinical
# ============================================================
quarkus.oidc.application-type=service

# Dev profile — disable OIDC and all security enforcement.
# GE-20260622-580d45: quarkus.security.auth.enabled-in-dev-mode=false activates
# DevModeDisabledAuthorizationController — @RolesAllowed not enforced in dev.
%dev.quarkus.security.auth.enabled-in-dev-mode=false
%dev.quarkus.oidc.enabled=false
%dev.quarkus.keycloak.devservices.enabled=false
```

### Test `application.properties`

```properties
# ============================================================
# OIDC test config (clinical#88)
# GE-20260521-f50602: discovery-disabled requires jwks-path (lazy — never fetched with @TestSecurity)
# GE-20260601-08a351: devservices disabled — Keycloak container startup suppressed
# casehub-platform-oidc ships META-INF/jandex.idx — no quarkus.index-dependency needed.
# ============================================================
quarkus.oidc.auth-server-url=http://localhost:8180/realms/test
quarkus.oidc.discovery-enabled=false
quarkus.oidc.jwks-path=protocol/openid-connect/certs
quarkus.keycloak.devservices.enabled=false
```

---

## Dependencies

### `runtime/pom.xml` — compile

```xml
<!-- OidcCurrentPrincipal @RequestScoped — becomes sole active CurrentPrincipal;
     brings quarkus-oidc transitively. CDI wiring: clinical#88 spec.
     Note: casehub-platform-oidc ships META-INF/jandex.idx — no index-dependency needed. -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-oidc</artifactId>
</dependency>
```

### `runtime/pom.xml` — test

```xml
<!-- @TestSecurity — controls SecurityIdentity in @QuarkusTest without a real OIDC server -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-test-security</artifactId>
    <scope>test</scope>
</dependency>
```

---

## Test Infrastructure

### Two independent concerns

| Concern | Mechanism | Scope |
|---------|-----------|-------|
| Business logic identity (`tenancyId`, `actorId`) | `FixedCurrentPrincipal` via `selected-alternatives` | Unchanged — continues as before |
| `@RolesAllowed` enforcement | `@TestSecurity` on test class | New — controls `SecurityIdentity` |

These do not interfere. `FixedCurrentPrincipal` is the CDI `CurrentPrincipal`; `@TestSecurity` controls the Quarkus security layer that `OidcCurrentPrincipal` wraps in production.

### CurrentPrincipal injection — safety analysis

`@RequestScoped` `CurrentPrincipal` is injected in exactly six production locations:

| Injection point | Context | Safe? |
|----------------|---------|-------|
| `TrialResource` | HTTP request | ✅ |
| `SiteResource` | HTTP request | ✅ |
| `PatientResource` | HTTP request | ✅ |
| `DeviationResource` | HTTP request | ✅ |
| `ProtocolAmendmentResource` | HTTP request | ✅ |
| `TrialActivationService` | Called only from `TrialResource.activate()` | ✅ |

Background services deliberately do NOT inject `CurrentPrincipal` (documented in blog `2026-06-09-mdp01-memory-arrives-with-baggage.md`, issue-71 constraint): `AeEscalationListener`, `PiResponseListener`, `DeviationExpirer`, `SponsorNotificationRetryJob`. These services source `tenantId` from entity fields or case context, never from `CurrentPrincipal` injection.

### Existing HTTP-calling test classes

All test classes making HTTP calls (via RestAssured) get:

```java
@TestSecurity(username = "test-actor",
              roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
```

Full access — no existing test breaks. Non-HTTP tests (unit, CDI observers called directly) do not need `@TestSecurity`.

### New: `RbacBoundaryTest`

Dedicated test class for access control invariants:

**MONITOR (safety-monitor) — zero write access:**
- `POST` to any endpoint → 403

**COORDINATOR — excluded from governance and trial management:**
- `POST /trials` → 403
- `POST /trials/{id}/activate` → 403
- `PATCH /trials/{id}/sponsor-config` → 403
- `POST /trials/{id}/sites` → 403
- `POST .../amendments` → 403
- `POST .../withdraw-consent` → 403

**INVESTIGATOR — excluded from sponsor-only trial management:**
- `POST /trials` → 403
- `POST /trials/{id}/activate` → 403
- `PATCH /trials/{id}/sponsor-config` → 403
- `POST /trials/{id}/sites` → 403

**SPONSOR — excluded from site-level clinical data entry:**
- `POST .../patients` (enroll) → 403
- `POST .../screen` → 403
- `POST .../adverse-events` → 403
- `POST .../deviations` → 403
- `POST .../withdraw-consent` → 403

**Unauthenticated:**
- Any endpoint → 401

**Missing tenancyId claim:**
- Valid token with roles but missing `tenancyId` JWT claim → 400

---

## Out of Scope

- RBAC-differentiated thresholds in `ClinicalActionRiskClassifier` — gates are CTCAE-severity-driven
- OIDC server setup / Keycloak configuration — deployment concern
- Inbound agent messages through qhorus channels — these bypass REST; not affected
- Quartz scheduler jobs (`SponsorNotificationRetryJob`) — already excluded from tests; system principal used

---

## Files Changed

| File | Change |
|------|--------|
| `api/src/main/java/io/casehub/clinical/api/ClinicalGroups.java` | New — group string constants |
| `runtime/pom.xml` | Add `casehub-platform-oidc` (compile), `quarkus-test-security` (test) |
| `runtime/src/main/resources/application.properties` | OIDC config, global deny policy, updated `exclude-types` with documented removals |
| `runtime/src/test/resources/application.properties` | OIDC test config |
| `runtime/src/main/java/.../MissingTenancyClaimExceptionMapper.java` | New — maps missing tenancyId claim to 400 |
| `runtime/src/main/java/.../resource/TrialResource.java` | `@RolesAllowed` on 4 methods |
| `runtime/src/main/java/.../resource/SiteResource.java` | `@RolesAllowed` on 2 methods |
| `runtime/src/main/java/.../resource/PatientResource.java` | `@RolesAllowed` on 9 methods |
| `runtime/src/main/java/.../resource/DeviationResource.java` | `@RolesAllowed` on 2 methods |
| `runtime/src/main/java/.../resource/ProtocolAmendmentResource.java` | `@RolesAllowed` on 2 methods |
| Existing `*Test.java` HTTP-calling tests | Add `@TestSecurity` |
| `runtime/src/test/java/.../RbacBoundaryTest.java` | New — access control boundary + tenancyId-missing test |

## Cross-repo

| Action | Issue |
|--------|-------|
| Filed: `OidcCurrentPrincipal` needs `@Alternative @Priority(100)` | casehubio/platform#111 |
