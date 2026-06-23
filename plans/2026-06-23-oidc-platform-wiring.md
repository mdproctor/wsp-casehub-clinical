# Wire casehub-platform-oidc — RBAC for Clinical REST Resources

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `casehub-platform-oidc` into casehub-clinical so that `@RolesAllowed` is enforced on all 19 REST endpoints with four clinical-trial-specific groups.

**Architecture:** Add `casehub-platform-oidc` as compile dependency — `OidcCurrentPrincipal` becomes the sole active `CurrentPrincipal`. Four GCP-derived groups (`trial-sponsor`, `principal-investigator`, `trial-coordinator`, `safety-monitor`) control access. `@TestSecurity` from `quarkus-test-security` controls `SecurityIdentity` in tests; `FixedCurrentPrincipal` continues to handle business logic identity. `quarkus.security.deny-unannotated-members=true` ensures new methods on annotated classes fail closed.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-oidc 0.2-SNAPSHOT, quarkus-test-security

## Global Constraints

- Spec: `specs/2026-06-23-oidc-platform-wiring-design.md`
- Issue: casehubio/clinical#88
- No auth logic in domain or service layers — `@RolesAllowed` on REST resources only
- No RBAC-differentiated thresholds in `ClinicalActionRiskClassifier`
- `casehub-platform-oidc` ships `META-INF/jandex.idx` — no `quarkus.index-dependency` needed
- `MissingTenancyClaimExceptionMapper` is NOT a deliverable — blocked on casehubio/platform#111; tracked in casehubio/clinical#89
- Build: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`

---

### Task 1: ClinicalGroups Constants + Dependencies + Configuration

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/ClinicalGroups.java`
- Modify: `runtime/pom.xml`
- Modify: `runtime/src/main/resources/application.properties`
- Modify: `runtime/src/test/resources/application.properties`

**Interfaces:**
- Consumes: nothing
- Produces: `ClinicalGroups.SPONSOR`, `.INVESTIGATOR`, `.COORDINATOR`, `.MONITOR` string constants used by all subsequent tasks

- [ ] **Step 1: Create `ClinicalGroups.java` in `api/`**

```java
package io.casehub.clinical.api;

public final class ClinicalGroups {
    public static final String SPONSOR      = "trial-sponsor";
    public static final String INVESTIGATOR = "principal-investigator";
    public static final String COORDINATOR  = "trial-coordinator";
    public static final String MONITOR      = "safety-monitor";
    private ClinicalGroups() {}
}
```

File: `api/src/main/java/io/casehub/clinical/api/ClinicalGroups.java`

- [ ] **Step 2: Add `casehub-platform-oidc` compile dependency to `runtime/pom.xml`**

Insert after the `casehub-platform-config` dependency block (after line 58):

```xml
    <!-- OidcCurrentPrincipal @RequestScoped — becomes sole active CurrentPrincipal;
         brings quarkus-oidc transitively. clinical#88. -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-oidc</artifactId>
    </dependency>
```

- [ ] **Step 3: Add `quarkus-test-security` test dependency to `runtime/pom.xml`**

Insert after the `quarkus-junit` test dependency (after line 166):

```xml
    <!-- @TestSecurity — controls SecurityIdentity in @QuarkusTest without a real OIDC server (clinical#88) -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-test-security</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 4: Update production `application.properties`**

Replace the existing `%prod.quarkus.arc.exclude-types` block (lines 51–63) with:

```properties
# ── CurrentPrincipal resolution (clinical#88) ─────────────────────────────────
# OidcCurrentPrincipal (@RequestScoped, casehub-platform-oidc) is the sole active
# CurrentPrincipal. Tenant identity comes from the JWT tenancyId claim.
#
# QhorusInboundCurrentPrincipal: excluded as LOCAL WORKAROUND pending
#   casehubio/platform#111 (OidcCurrentPrincipal needs @Alternative @Priority(100)).
#   Remove this exclusion once platform#111 ships.
#
# TenantScopedPrincipal (casehub-work @RequestScoped): excluded — belongs to
#   casehub-work deployments; upstream fix tracked in casehubio/work#268.
#
# CasehubWorkloadProvider: REMOVED from this list — class deleted in engine#378.
%prod.quarkus.arc.exclude-types=\
  io.casehub.qhorus.runtime.identity.QhorusInboundCurrentPrincipal,\
  io.casehub.work.runtime.service.TenantScopedPrincipal
```

Append at end of file:

```properties

# ============================================================
# OIDC — casehub-platform-oidc (clinical#88)
# Required env vars (do NOT hardcode values):
#   QUARKUS_OIDC_AUTH_SERVER_URL   e.g. https://auth.example.com/realms/casehub
#   QUARKUS_OIDC_CLIENT_ID         e.g. casehub-clinical
# ============================================================
quarkus.oidc.application-type=service

# Deny unannotated members — any new method on a class with existing @RolesAllowed
# annotations fails closed. Does NOT cover entirely new resource classes with zero
# annotations (DenyUnannotatedPredicate requires at least one annotated method).
quarkus.security.deny-unannotated-members=true

# Dev profile — disable OIDC and security enforcement.
# GE-20260622-580d45: auth.enabled-in-dev-mode=false suppresses @RolesAllowed + deny-unannotated-members.
%dev.quarkus.security.auth.enabled-in-dev-mode=false
%dev.quarkus.oidc.enabled=false
%dev.quarkus.keycloak.devservices.enabled=false
```

- [ ] **Step 5: Update test `application.properties`**

Append at end of file:

```properties

# ============================================================
# OIDC test config (clinical#88)
# GE-20260521-f50602: discovery-disabled requires jwks-path (lazy — never fetched with @TestSecurity)
# GE-20260601-08a351: devservices disabled — Keycloak container startup suppressed
# casehub-platform-oidc ships META-INF/jandex.idx — no quarkus.index-dependency needed.
# @TestSecurity controls SecurityIdentity for @RolesAllowed; FixedCurrentPrincipal
# (selected-alternatives above) controls CurrentPrincipal for business logic.
# ============================================================
quarkus.oidc.auth-server-url=http://localhost:8180/realms/test
quarkus.oidc.discovery-enabled=false
quarkus.oidc.jwks-path=protocol/openid-connect/certs
quarkus.keycloak.devservices.enabled=false
```

- [ ] **Step 6: Verify api module compiles**

Run: `mvn install -pl api --batch-mode`
Expected: BUILD SUCCESS — `ClinicalGroups.java` compiles cleanly.

- [ ] **Step 7: Verify runtime module starts and all 431+ existing tests pass**

Run: `mvn test -pl runtime --batch-mode`
Expected: All tests pass. OIDC config is inert in tests (discovery-disabled, `@TestSecurity` not yet on test classes — Quarkus falls through to `FixedCurrentPrincipal`). No `@RolesAllowed` on resources yet, so `deny-unannotated-members` has no effect (no classes with any annotated methods).

- [ ] **Step 8: Commit**

```
feat(#88): add ClinicalGroups, casehub-platform-oidc dep, OIDC config

Refs #88
```

Stage: `api/src/main/java/io/casehub/clinical/api/ClinicalGroups.java`, `runtime/pom.xml`, `runtime/src/main/resources/application.properties`, `runtime/src/test/resources/application.properties`

---

### Task 2: Add `@TestSecurity` to All HTTP-Calling Test Classes

**Files:**
- Modify: 11 test files (listed below)

**Interfaces:**
- Consumes: `ClinicalGroups.SPONSOR`, `.INVESTIGATOR`, `.COORDINATOR` (from Task 1)
- Produces: all HTTP tests authenticated with full clinical access — no test breaks when `@RolesAllowed` is added in Task 3

The 11 test classes using RestAssured:

1. `runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java`
2. `runtime/src/test/java/io/casehub/clinical/resource/SiteResourceTest.java`
3. `runtime/src/test/java/io/casehub/clinical/resource/PatientResourceTest.java`
4. `runtime/src/test/java/io/casehub/clinical/resource/DeviationResourceTest.java`
5. `runtime/src/test/java/io/casehub/clinical/resource/AdverseEventResourceTest.java`
6. `runtime/src/test/java/io/casehub/clinical/resource/PatientAuditResourceTest.java`
7. `runtime/src/test/java/io/casehub/clinical/resource/ClinicalLayerComplianceTest.java`
8. `runtime/src/test/java/io/casehub/clinical/resource/ThreeSiteShowcaseTest.java`
9. `runtime/src/test/java/io/casehub/clinical/service/EligibilityScreeningIntegrationTest.java`
10. `runtime/src/test/java/io/casehub/clinical/service/ProtocolAmendmentIntegrationTest.java`
11. `runtime/src/test/java/io/casehub/clinical/service/TrialActivationTest.java`

- [ ] **Step 1: Add `@TestSecurity` to each test class**

Add these two imports to each file:

```java
import io.casehub.clinical.api.ClinicalGroups;
import io.quarkus.test.security.TestSecurity;
```

Add this annotation directly above the class declaration (after `@QuarkusTest`):

```java
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
```

Apply identically to all 11 classes. `MONITOR` is intentionally excluded — monitors are read-only and existing tests write data.

- [ ] **Step 2: Run full test suite**

Run: `mvn test -pl runtime --batch-mode`
Expected: All 431+ tests pass. `@TestSecurity` supplies a `SecurityIdentity` with all three write roles — no test breaks. `@RolesAllowed` is not on resources yet, so this is a no-op preparation.

- [ ] **Step 3: Commit**

```
chore(#88): add @TestSecurity to all HTTP-calling test classes

Refs #88
```

Stage all 11 modified test files.

---

### Task 3: Add `@RolesAllowed` to All 19 REST Endpoints

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/SiteResource.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/DeviationResource.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/ProtocolAmendmentResource.java`

**Interfaces:**
- Consumes: `ClinicalGroups.*` constants (from Task 1); `@TestSecurity` on test classes (from Task 2)
- Produces: all 19 endpoints enforcing role-based access control

- [ ] **Step 1: Annotate `TrialResource.java` (4 endpoints)**

Add imports:

```java
import io.casehub.clinical.api.ClinicalGroups;
import jakarta.annotation.security.RolesAllowed;
```

Add `@RolesAllowed` above each method:

| Method | Annotation |
|--------|------------|
| `register()` — `POST /trials` | `@RolesAllowed(ClinicalGroups.SPONSOR)` |
| `get()` — `GET /trials/{id}` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |
| `updateSponsorConfig()` — `PATCH` | `@RolesAllowed(ClinicalGroups.SPONSOR)` |
| `activate()` — `POST /activate` | `@RolesAllowed(ClinicalGroups.SPONSOR)` |

- [ ] **Step 2: Annotate `SiteResource.java` (2 endpoints)**

Add imports:

```java
import io.casehub.clinical.api.ClinicalGroups;
import jakarta.annotation.security.RolesAllowed;
```

| Method | Annotation |
|--------|------------|
| `add()` — `POST /sites` | `@RolesAllowed(ClinicalGroups.SPONSOR)` |
| `get()` — `GET /sites/{siteId}` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |

- [ ] **Step 3: Annotate `PatientResource.java` (9 endpoints)**

Add imports:

```java
import io.casehub.clinical.api.ClinicalGroups;
import jakarta.annotation.security.RolesAllowed;
```

| Method | Annotation |
|--------|------------|
| `enroll()` — `POST /patients` | `@RolesAllowed({ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})` |
| `get()` — `GET /patients/{id}` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |
| `screen()` — `POST /screen` | `@RolesAllowed({ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})` |
| `reportAdverseEvent()` — `POST /adverse-events` | `@RolesAllowed({ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})` |
| `getAdverseEvent()` — `GET /adverse-events/{aeId}` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |
| `verifyLedger()` — `GET /ledger/verify` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |
| `withdrawConsent()` — `POST /withdraw-consent` | `@RolesAllowed(ClinicalGroups.INVESTIGATOR)` |
| `getAuditProv()` — `GET /audit/prov` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |
| `getMerkleProof()` — `GET /audit/entries/{id}/proof` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |

- [ ] **Step 4: Annotate `DeviationResource.java` (2 endpoints)**

Add imports:

```java
import io.casehub.clinical.api.ClinicalGroups;
import jakarta.annotation.security.RolesAllowed;
```

| Method | Annotation |
|--------|------------|
| `reportDeviation()` — `POST /deviations` | `@RolesAllowed({ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})` |
| `getDeviation()` — `GET /deviations/{id}` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |

- [ ] **Step 5: Annotate `ProtocolAmendmentResource.java` (2 endpoints)**

Add imports:

```java
import io.casehub.clinical.api.ClinicalGroups;
import jakarta.annotation.security.RolesAllowed;
```

| Method | Annotation |
|--------|------------|
| `propose()` — `POST /amendments` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR})` |
| `get()` — `GET /amendments/{id}` | `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})` |

- [ ] **Step 6: Run full test suite**

Run: `mvn test -pl runtime --batch-mode`
Expected: All tests pass — `@TestSecurity` from Task 2 provides the required roles.

- [ ] **Step 7: Commit**

```
feat(#88): add @RolesAllowed to all 19 REST endpoints

Refs #88
```

Stage all 5 modified resource files.

---

### Task 4: RbacBoundaryTest — Access Control Invariants

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/resource/RbacBoundaryTest.java`

**Interfaces:**
- Consumes: `ClinicalGroups.*` (Task 1); `@RolesAllowed` annotations (Task 3)
- Produces: test coverage for all 403 boundaries + 401 unauthenticated

This is a dedicated `@QuarkusTest` class. Each test method uses a different `@TestSecurity` annotation at the method level to test a specific role's denied operations. The class has no class-level `@TestSecurity` — method-level overrides.

Test data setup: each test creates a trial (as SPONSOR) before testing the boundary. Use nested helper methods or `@TestSecurity`-annotated inner classes.

**Important design constraint:** Quarkus `@TestSecurity` works at the method level. Each test method can have its own `@TestSecurity`. For the setup (creating a trial), a helper method annotated with `@TestSecurity(user = "setup", roles = {SPONSOR, INVESTIGATOR, COORDINATOR})` provides the required entities. However, `@TestSecurity` is a CDI interceptor annotation — it only applies to the method directly annotated, not to methods called from within it. The approach: pre-create test data in a `@BeforeEach` method (which runs without security context in test mode — RestAssured calls within `@BeforeEach` will use the class-level `@TestSecurity` if present, or fail if none).

The practical pattern for boundary tests: use a class-level `@TestSecurity` with full roles for setup, then override at method level with restricted roles for the actual assertions. Each test method's `@TestSecurity` overrides the class level.

- [ ] **Step 1: Write the test class structure**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;

@QuarkusTest
@TestSecurity(user = "setup-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class RbacBoundaryTest {

    @Inject FixedCurrentPrincipal principal;

    private String trialId;
    private String siteId;
    private String enrollmentId;

    @BeforeEach
    void setup() {
        principal.reset();

        // Create trial (requires SPONSOR)
        String trialLocation = given()
            .contentType("application/json")
            .body("{\"protocolId\":\"RBAC-001\",\"phase\":\"PHASE_I\",\"sponsor\":\"RBAC-Sponsor\",\"targetEnrollment\":10}")
            .when().post("/trials")
            .then().statusCode(201).extract().header("Location");
        trialId = trialLocation.substring(trialLocation.lastIndexOf('/') + 1);

        // Create site (requires SPONSOR)
        String siteLocation = given()
            .contentType("application/json")
            .body("{\"investigatorId\":\"PI-001\"}")
            .when().post("/trials/" + trialId + "/sites")
            .then().statusCode(201).extract().header("Location");
        siteId = siteLocation.substring(siteLocation.lastIndexOf('/') + 1);

        // Enroll patient (requires INVESTIGATOR or COORDINATOR)
        String enrollLocation = given()
            .contentType("application/json")
            .body("{\"patientId\":\"RBAC-PAT-001\"}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/patients")
            .then().statusCode(201).extract().header("Location");
        enrollmentId = enrollLocation.substring(enrollLocation.lastIndexOf('/') + 1);
    }

    // --- MONITOR: zero write access (POST and PATCH) ---

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_post_trials() {
        given().contentType("application/json")
            .body("{\"protocolId\":\"X\",\"phase\":\"PHASE_I\",\"sponsor\":\"X\",\"targetEnrollment\":1}")
            .when().post("/trials")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_activate_trial() {
        given().when().post("/trials/" + trialId + "/activate")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_patch_sponsor_config() {
        given().contentType("application/json")
            .body("{\"connectorId\":\"x\",\"destination\":\"y\"}")
            .when().patch("/trials/" + trialId + "/sponsor-config")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_add_site() {
        given().contentType("application/json")
            .body("{\"investigatorId\":\"PI-X\"}")
            .when().post("/trials/" + trialId + "/sites")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_enroll_patient() {
        given().contentType("application/json")
            .body("{\"patientId\":\"PAT-X\"}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/patients")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_screen_patient() {
        given().contentType("application/json")
            .body("{\"criteria\":[{\"criterionId\":\"C1\",\"met\":true}]}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/patients/" + enrollmentId + "/screen")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_report_adverse_event() {
        given().contentType("application/json")
            .body("{\"grade\":\"GRADE_1\",\"occurredAt\":\"2026-01-01T00:00:00Z\"}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/patients/" + enrollmentId + "/adverse-events")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_report_deviation() {
        given().contentType("application/json")
            .body("{\"deviationType\":\"dosing\",\"severity\":\"MINOR\"}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/deviations")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_propose_amendment() {
        given().contentType("application/json")
            .body("{\"proposedChange\":\"change dosing\"}")
            .when().post("/trials/" + trialId + "/amendments")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_cannot_withdraw_consent() {
        given().when().post("/trials/" + trialId + "/sites/" + siteId + "/patients/" + enrollmentId + "/withdraw-consent")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "monitor-user", roles = {ClinicalGroups.MONITOR})
    void monitor_can_read_trial() {
        given().when().get("/trials/" + trialId)
            .then().statusCode(200);
    }

    // --- COORDINATOR: excluded from governance and trial management ---

    @Test
    @TestSecurity(user = "coord-user", roles = {ClinicalGroups.COORDINATOR})
    void coordinator_cannot_create_trial() {
        given().contentType("application/json")
            .body("{\"protocolId\":\"X\",\"phase\":\"PHASE_I\",\"sponsor\":\"X\",\"targetEnrollment\":1}")
            .when().post("/trials")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "coord-user", roles = {ClinicalGroups.COORDINATOR})
    void coordinator_cannot_activate_trial() {
        given().when().post("/trials/" + trialId + "/activate")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "coord-user", roles = {ClinicalGroups.COORDINATOR})
    void coordinator_cannot_patch_sponsor_config() {
        given().contentType("application/json")
            .body("{\"connectorId\":\"x\",\"destination\":\"y\"}")
            .when().patch("/trials/" + trialId + "/sponsor-config")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "coord-user", roles = {ClinicalGroups.COORDINATOR})
    void coordinator_cannot_add_site() {
        given().contentType("application/json")
            .body("{\"investigatorId\":\"PI-X\"}")
            .when().post("/trials/" + trialId + "/sites")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "coord-user", roles = {ClinicalGroups.COORDINATOR})
    void coordinator_cannot_propose_amendment() {
        given().contentType("application/json")
            .body("{\"proposedChange\":\"change dosing\"}")
            .when().post("/trials/" + trialId + "/amendments")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "coord-user", roles = {ClinicalGroups.COORDINATOR})
    void coordinator_cannot_withdraw_consent() {
        given().when().post("/trials/" + trialId + "/sites/" + siteId + "/patients/" + enrollmentId + "/withdraw-consent")
            .then().statusCode(403);
    }

    // --- INVESTIGATOR: excluded from sponsor-only trial management ---

    @Test
    @TestSecurity(user = "pi-user", roles = {ClinicalGroups.INVESTIGATOR})
    void investigator_cannot_create_trial() {
        given().contentType("application/json")
            .body("{\"protocolId\":\"X\",\"phase\":\"PHASE_I\",\"sponsor\":\"X\",\"targetEnrollment\":1}")
            .when().post("/trials")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "pi-user", roles = {ClinicalGroups.INVESTIGATOR})
    void investigator_cannot_activate_trial() {
        given().when().post("/trials/" + trialId + "/activate")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "pi-user", roles = {ClinicalGroups.INVESTIGATOR})
    void investigator_cannot_patch_sponsor_config() {
        given().contentType("application/json")
            .body("{\"connectorId\":\"x\",\"destination\":\"y\"}")
            .when().patch("/trials/" + trialId + "/sponsor-config")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "pi-user", roles = {ClinicalGroups.INVESTIGATOR})
    void investigator_cannot_add_site() {
        given().contentType("application/json")
            .body("{\"investigatorId\":\"PI-X\"}")
            .when().post("/trials/" + trialId + "/sites")
            .then().statusCode(403);
    }

    // --- SPONSOR: excluded from site-level clinical data entry ---

    @Test
    @TestSecurity(user = "sponsor-user", roles = {ClinicalGroups.SPONSOR})
    void sponsor_cannot_enroll_patient() {
        given().contentType("application/json")
            .body("{\"patientId\":\"PAT-X\"}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/patients")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "sponsor-user", roles = {ClinicalGroups.SPONSOR})
    void sponsor_cannot_screen_patient() {
        given().contentType("application/json")
            .body("{\"criteria\":[{\"criterionId\":\"C1\",\"met\":true}]}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/patients/" + enrollmentId + "/screen")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "sponsor-user", roles = {ClinicalGroups.SPONSOR})
    void sponsor_cannot_report_adverse_event() {
        given().contentType("application/json")
            .body("{\"grade\":\"GRADE_1\",\"occurredAt\":\"2026-01-01T00:00:00Z\"}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/patients/" + enrollmentId + "/adverse-events")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "sponsor-user", roles = {ClinicalGroups.SPONSOR})
    void sponsor_cannot_report_deviation() {
        given().contentType("application/json")
            .body("{\"deviationType\":\"dosing\",\"severity\":\"MINOR\"}")
            .when().post("/trials/" + trialId + "/sites/" + siteId + "/deviations")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "sponsor-user", roles = {ClinicalGroups.SPONSOR})
    void sponsor_cannot_withdraw_consent() {
        given().when().post("/trials/" + trialId + "/sites/" + siteId + "/patients/" + enrollmentId + "/withdraw-consent")
            .then().statusCode(403);
    }

    // --- UNAUTHENTICATED ---

    @Test
    @TestSecurity  // no user, no roles — unauthenticated
    void unauthenticated_gets_401() {
        given().when().get("/trials/" + trialId)
            .then().statusCode(401);
    }
}
```

File: `runtime/src/test/java/io/casehub/clinical/resource/RbacBoundaryTest.java`

- [ ] **Step 2: Run the new test class**

Run: `mvn test -pl runtime -Dtest=RbacBoundaryTest --batch-mode`
Expected: All tests pass — 403 for denied role/endpoint combinations, 401 for unauthenticated, 200 for monitor reading.

- [ ] **Step 3: Run full test suite**

Run: `mvn test -pl runtime --batch-mode`
Expected: All tests pass — no regressions.

- [ ] **Step 4: Commit**

```
test(#88): add RbacBoundaryTest — 403/401 boundary coverage for all 4 roles

Refs #88
```

Stage: `runtime/src/test/java/io/casehub/clinical/resource/RbacBoundaryTest.java`
