# Design: Wire casehub-platform-oidc — RBAC for Clinical REST Resources

**Issue:** casehubio/clinical#88
**Date:** 2026-06-23
**Branch:** issue-88-wire-platform-oidc

---

## Context

`casehub-platform-oidc` ships `OidcCurrentPrincipal @RequestScoped`, which implements `CurrentPrincipal` backed by `SecurityIdentity` and `JsonWebToken`. It provides `tenancyId()`, `actorId()`, and `groups()` from the JWT. Adding it to the classpath activates `@RolesAllowed` enforcement across Quarkus REST resources.

Without it on the classpath, `CurrentPrincipal.groups()` always returns an empty set and `@RolesAllowed` annotations are inert.

Reference implementation: `casehubio/life#40` (life repo, commit `42064e3` + `b727392`). Clinical follows the same wiring pattern with clinical-specific groups and endpoint topology.

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
> - `POST /withdraw-consent` — COORDINATOR excluded (GDPR Art.17 data controller authority); correct?
> - `POST /amendments` — COORDINATOR excluded (protocol governance, not operational); correct?
> - `POST /sites` — INVESTIGATOR excluded (sponsor selects sites per ICH §5.6); correct? (A PI does not commission their own site.)
> - `POST /trials/activate` — INVESTIGATOR excluded (IND submission is sponsor authority per 21 CFR 312.30); correct?

| Resource | Method + Path | Groups |
|----------|---------------|--------|
| `TrialResource` | `POST /trials` | `SPONSOR` |
| | `GET /trials/{id}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `PATCH /trials/{id}/sponsor-config` | `SPONSOR` |
| | `POST /trials/{id}/activate` | `SPONSOR` |
| `SiteResource` | `POST /trials/{trialId}/sites` | `SPONSOR` |
| | `GET /trials/{trialId}/sites/{siteId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| `PatientResource` | `POST .../patients` (enroll) | `SPONSOR, INVESTIGATOR, COORDINATOR` |
| | `GET .../patients/{enrollmentId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `POST .../patients/{id}/screen` | `SPONSOR, INVESTIGATOR, COORDINATOR` |
| | `POST .../patients/{id}/adverse-events` | `SPONSOR, INVESTIGATOR, COORDINATOR` |
| | `GET .../adverse-events/{aeId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `GET .../ledger/verify` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `POST .../withdraw-consent` | `SPONSOR, INVESTIGATOR` |
| | `GET .../audit/prov` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| | `GET .../audit/entries/{entryId}/proof` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| `DeviationResource` | `POST .../deviations` | `SPONSOR, INVESTIGATOR, COORDINATOR` |
| | `GET .../deviations/{deviationId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |
| `ProtocolAmendmentResource` | `POST .../amendments` | `SPONSOR, INVESTIGATOR` |
| | `GET .../amendments/{amendmentId}` | `SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR` |

**Total: 19 endpoints across 5 REST resources.**

---

## CDI Wiring

### Problem

Adding `casehub-platform-oidc` puts `OidcCurrentPrincipal @RequestScoped` on the classpath. Clinical currently uses `QhorusInboundCurrentPrincipal @ApplicationScoped` (reads `X-Tenancy-ID` header) as its active `CurrentPrincipal`. Both are non-default, non-alternative beans → `AmbiguousResolutionException` at production augmentation.

### Resolution

Exclude `QhorusInboundCurrentPrincipal` from production. `OidcCurrentPrincipal` becomes the sole active `CurrentPrincipal`. Tenant identity moves from a raw `X-Tenancy-ID` header to the authenticated JWT token — correct direction for a system with real auth.

`OidcCurrentPrincipal` provides `tenancyId()`, `actorId()`, and `groups()` from the JWT, so no capability is lost.

### Updated `%prod.quarkus.arc.exclude-types`

```properties
# Identical to life#40 exclusion list
%prod.quarkus.arc.exclude-types=\
  io.casehub.platform.mock.MockCurrentPrincipal,\
  io.casehub.platform.mock.MockGroupMembershipProvider,\
  io.casehub.qhorus.runtime.identity.QhorusInboundCurrentPrincipal,\
  io.casehub.persistence.memory.DefaultTestPrincipal,\
  io.casehub.work.runtime.service.TenantScopedPrincipal
```

Note: `MockGroupMembershipProvider` was missing from clinical's prior exclusion list — added here.

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
<!-- OidcCurrentPrincipal @RequestScoped — displaces QhorusInboundCurrentPrincipal;
     brings quarkus-oidc transitively. CDI wiring: see clinical#88 spec. -->
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

These do not interfere. `FixedCurrentPrincipal` is the CDI `CurrentPrincipal`; `@TestSecurity` controls the Quarkus security layer that `OidcCurrentPrincipal` wraps.

### Existing HTTP-calling test classes

All test classes making HTTP calls (via RestAssured) get:

```java
@TestSecurity(username = "test-actor",
              roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
```

Full access — no existing test breaks. Non-HTTP tests (unit, CDI observers called directly) do not need `@TestSecurity`.

### New: `RbacBoundaryTest`

Dedicated test class for access control invariants:

- `MONITOR` cannot `POST` to any write endpoint → 403
- `COORDINATOR` cannot: `POST /trials`, `POST /activate`, `PATCH /sponsor-config`, `POST /sites`, `POST /amendments`, `POST /withdraw-consent` → 403
- Unauthenticated request to any protected endpoint → 401

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
| `runtime/src/main/resources/application.properties` | OIDC config + updated `exclude-types` |
| `runtime/src/test/resources/application.properties` | OIDC test config |
| `runtime/src/main/java/.../resource/TrialResource.java` | `@RolesAllowed` on 4 methods |
| `runtime/src/main/java/.../resource/SiteResource.java` | `@RolesAllowed` on 2 methods |
| `runtime/src/main/java/.../resource/PatientResource.java` | `@RolesAllowed` on 9 methods |
| `runtime/src/main/java/.../resource/DeviationResource.java` | `@RolesAllowed` on 2 methods |
| `runtime/src/main/java/.../resource/ProtocolAmendmentResource.java` | `@RolesAllowed` on 2 methods |
| Existing `*Test.java` HTTP-calling tests | Add `@TestSecurity` |
| `runtime/src/test/java/.../RbacBoundaryTest.java` | New — access control boundary tests |
