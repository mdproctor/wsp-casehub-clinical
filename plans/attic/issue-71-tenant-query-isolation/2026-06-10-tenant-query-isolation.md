# Tenant Query Isolation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `findByIdForTenant(UUID, CurrentPrincipal)` to all six domain entities and enforce it at every REST-facing lookup, plus fix child-entity tenant stamping to derive from the parent entity rather than `principal.tenancyId()`.

**Architecture:** Entity-level static helpers hold the isolation logic (HQL filter + cross-tenant-admin bypass). REST resources call these helpers instead of plain `findById`. Internal system services (`DeviationExpirer`, CDI observers, `TrialCaseLookup`) keep plain `findById` — they are system actors that legitimately need cross-tenant access. `TrialActivationService` gains a `CurrentPrincipal` injection and uses the helper for its user-supplied trial ID. `AdverseEventService` loses its `CurrentPrincipal` injection and instead derives tenant from the enrollment entity it already loads.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM 7 / Panache Active Record, JUnit 5 + RestAssured (`@QuarkusTest`), `FixedCurrentPrincipal` (`@ApplicationScoped @Alternative @Priority(1)`) from `casehub-platform-testing`.

**Spec:** `docs/specs/2026-06-09-tenant-query-isolation-design.md` in the project repo.

**Build commands:**
```bash
mvn install -pl api --batch-mode                          # install api module
mvn test -pl runtime --batch-mode                          # run all runtime tests
mvn test -pl runtime -Dtest=TrialResourceTest --batch-mode # run one test class
```

---

## File map

**Entities (all in `runtime/src/main/java/io/casehub/clinical/entity/`):**
- `ClinicalTrial.java` — add `findByIdForTenant`
- `TrialSite.java` — add `findByIdForTenant`
- `PatientEnrollment.java` — add `findByIdForTenant`
- `AdverseEvent.java` — add `findByIdForTenant`
- `ProtocolDeviation.java` — add `findByIdForTenant`
- `IrbApproval.java` — add `findByIdForTenant`

**Resources (all in `runtime/src/main/java/io/casehub/clinical/resource/`):**
- `TrialResource.java` — update `get()`, `updateSponsorConfig()`
- `SiteResource.java` — update `get()`, `add()` (read fix + write stamp fix)
- `PatientResource.java` — update `get()`, `enroll()` (read fix + write stamp fix), `reportAdverseEvent()`
- `DeviationResource.java` — update `getDeviation()`, `reportDeviation()` (read fix + write stamp fix)

**Services (in `runtime/src/main/java/io/casehub/clinical/service/`):**
- `TrialActivationService.java` — add `CurrentPrincipal` injection, update `markActive()`
- `AdverseEventService.java` — remove `CurrentPrincipal` injection, inline resolver methods, derive tenant from enrollment

**Test classes (all in `runtime/src/test/...`):**
- `resource/TrialResourceTest.java` — add isolation + bypass tests
- `service/TrialActivationTest.java` — add `FixedCurrentPrincipal`, isolation + bypass tests
- `resource/SiteResourceTest.java` — add isolation + bypass + write-invariant tests
- `resource/PatientResourceTest.java` — add isolation + bypass + write-invariant tests
- `resource/DeviationResourceTest.java` — add isolation + bypass + write-invariant tests
- `service/AdverseEventServiceTest.java` — add write-invariant test

---

## Task 1: Add `findByIdForTenant` to all 6 entities

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/ClinicalTrial.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/TrialSite.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/PatientEnrollment.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/IrbApproval.java`

These are additive-only changes. No existing code is touched. Tests come in later tasks.

- [ ] **Step 1: Add `findByIdForTenant` to `ClinicalTrial`**

Add the import and the method. The method goes at the end of the class body, before the closing brace.

Add this import to `ClinicalTrial.java`:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
```

Add this method:
```java
public static ClinicalTrial findByIdForTenant(UUID id, CurrentPrincipal principal) {
    if (principal.isCrossTenantAdmin()) return findById(id);
    return find("id = ?1 AND tenantId = ?2", id, principal.tenancyId()).firstResult();
}
```

- [ ] **Step 2: Add `findByIdForTenant` to `TrialSite`**

Add this import to `TrialSite.java`:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
```

Add this method:
```java
public static TrialSite findByIdForTenant(UUID id, CurrentPrincipal principal) {
    if (principal.isCrossTenantAdmin()) return findById(id);
    return find("id = ?1 AND tenantId = ?2", id, principal.tenancyId()).firstResult();
}
```

- [ ] **Step 3: Add `findByIdForTenant` to `PatientEnrollment`**

Add this import:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
```

Add this method:
```java
public static PatientEnrollment findByIdForTenant(UUID id, CurrentPrincipal principal) {
    if (principal.isCrossTenantAdmin()) return findById(id);
    return find("id = ?1 AND tenantId = ?2", id, principal.tenancyId()).firstResult();
}
```

- [ ] **Step 4: Add `findByIdForTenant` to `AdverseEvent`**

Add this import:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
```

Add this method:
```java
public static AdverseEvent findByIdForTenant(UUID id, CurrentPrincipal principal) {
    if (principal.isCrossTenantAdmin()) return findById(id);
    return find("id = ?1 AND tenantId = ?2", id, principal.tenancyId()).firstResult();
}
```

- [ ] **Step 5: Add `findByIdForTenant` to `ProtocolDeviation`**

Add this import:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
```

Add this method:
```java
public static ProtocolDeviation findByIdForTenant(UUID id, CurrentPrincipal principal) {
    if (principal.isCrossTenantAdmin()) return findById(id);
    return find("id = ?1 AND tenantId = ?2", id, principal.tenancyId()).firstResult();
}
```

- [ ] **Step 6: Add `findByIdForTenant` to `IrbApproval`**

Add this import:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
```

Add this method:
```java
public static IrbApproval findByIdForTenant(UUID id, CurrentPrincipal principal) {
    if (principal.isCrossTenantAdmin()) return findById(id);
    return find("id = ?1 AND tenantId = ?2", id, principal.tenancyId()).firstResult();
}
```

- [ ] **Step 7: Compile to verify no errors**

```bash
mvn install -pl api --batch-mode && mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS with no compilation errors.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/entity/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(entity): add findByIdForTenant helper to all 6 domain entities

Refs #71"
```

---

## Task 2: TrialResource — read path + isolation + bypass tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java`

`TrialResource` already injects `@Inject CurrentPrincipal principal`. No injection changes needed.

- [ ] **Step 1: Write failing isolation tests in `TrialResourceTest`**

Add these imports to `TrialResourceTest.java`:
```java
import io.casehub.platform.testing.FixedCurrentPrincipal;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
```

Add these fields and methods to the class body:
```java
@Inject FixedCurrentPrincipal principal;

@AfterEach
void resetPrincipal() { principal.reset(); }

@Test
void get_returns_404_for_wrong_tenant() {
    String location = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-T-001\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID id = UUID.fromString(location.substring(location.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    given().when().get("/trials/{id}", id).then().statusCode(404);
}

@Test
void patch_sponsor_config_returns_404_for_wrong_tenant() {
    String location = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-T-002\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID id = UUID.fromString(location.substring(location.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    given()
        .contentType("application/json")
        .body("{\"connectorId\":\"slack\",\"destination\":\"https://example.com\"}")
        .when().patch("/trials/{id}/sponsor-config", id)
        .then().statusCode(404);
}
```

- [ ] **Step 2: Run to verify isolation tests fail**

```bash
mvn test -pl runtime -Dtest=TrialResourceTest --batch-mode
```

Expected: `get_returns_404_for_wrong_tenant` and `patch_sponsor_config_returns_404_for_wrong_tenant` FAIL with `expected: 404 but was: 200` (or 204). All other existing tests pass.

- [ ] **Step 3: Update `TrialResource.get()` to use the helper**

In `TrialResource.java`, change the `get()` method body:

```java
@GET
@Path("/{id}")
public Response get(@PathParam("id") UUID id) {
    ClinicalTrial trial = ClinicalTrial.findByIdForTenant(id, principal);
    if (trial == null) return Response.status(Response.Status.NOT_FOUND).build();
    return Response.ok(trial).build();
}
```

- [ ] **Step 4: Update `TrialResource.updateSponsorConfig()` to use the helper**

```java
@PATCH
@Path("/{id}/sponsor-config")
@Transactional
public Response updateSponsorConfig(@PathParam("id") UUID id, @NotNull @Valid SponsorConfigRequest req) {
    ClinicalTrial trial = ClinicalTrial.findByIdForTenant(id, principal);
    if (trial == null) return Response.status(Response.Status.NOT_FOUND).build();
    trial.sponsorNotificationConnectorId = req.connectorId();
    trial.sponsorNotificationDestination = req.destination();
    return Response.noContent().build();
}
```

- [ ] **Step 5: Run to verify isolation tests now pass**

```bash
mvn test -pl runtime -Dtest=TrialResourceTest --batch-mode
```

Expected: all tests including the two new isolation tests PASS.

- [ ] **Step 6: Write bypass test**

Add to `TrialResourceTest.java`:
```java
@Test
void get_succeeds_for_cross_tenant_admin() {
    String location = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-T-003\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID id = UUID.fromString(location.substring(location.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    principal.setCrossTenantAdmin(true);
    given().when().get("/trials/{id}", id).then().statusCode(200);
}
```

- [ ] **Step 7: Run to verify bypass test passes**

```bash
mvn test -pl runtime -Dtest=TrialResourceTest --batch-mode
```

Expected: all tests PASS including the new bypass test.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java \
  runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(resource): tenant-scope ClinicalTrial reads in TrialResource

Refs #71"
```

---

## Task 3: SiteResource — read path, write-path stamp fix, all tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/SiteResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/SiteResourceTest.java`

`SiteResource` already injects `@Inject CurrentPrincipal principal`. No injection changes needed.

- [ ] **Step 1: Write failing tests in `SiteResourceTest`**

Add these imports:
```java
import io.casehub.platform.testing.FixedCurrentPrincipal;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
```

Add these fields and tests:
```java
@Inject FixedCurrentPrincipal principal;

@AfterEach
void resetPrincipal() { principal.reset(); }

@Test
void get_site_returns_404_for_wrong_tenant() {
    String trialLoc = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-S-001\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    String siteLoc = given()
        .contentType("application/json")
        .body("{\"investigatorId\":\"pi-iso\"}")
        .when().post("/trials/{id}/sites", trialId).then().statusCode(201).extract().header("Location");
    UUID siteId = UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    given().when().get("/trials/{t}/sites/{s}", trialId, siteId).then().statusCode(404);
}

@Test
void add_site_returns_404_when_trial_belongs_to_different_tenant() {
    String trialLoc = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-S-002\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    given()
        .contentType("application/json")
        .body("{\"investigatorId\":\"pi-iso\"}")
        .when().post("/trials/{id}/sites", trialId)
        .then().statusCode(404);
}

@Test
void site_inherits_trial_tenantId_not_principal_tenantId() {
    String trialLoc = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-S-003\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    // cross-tenant admin adds a site to a different tenant's trial
    principal.setTenancyId("admin-tenant");
    principal.setCrossTenantAdmin(true);
    String siteLoc = given()
        .contentType("application/json")
        .body("{\"investigatorId\":\"pi-iso\"}")
        .when().post("/trials/{id}/sites", trialId).then().statusCode(201).extract().header("Location");
    UUID siteId = UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));

    // default tenant can find the site (site.tenantId = trial.tenantId = default tenant)
    principal.reset();
    given().when().get("/trials/{t}/sites/{s}", trialId, siteId).then().statusCode(200);

    // admin's own tenant (no bypass) cannot find it — proves site.tenantId ≠ "admin-tenant"
    principal.setTenancyId("admin-tenant");
    given().when().get("/trials/{t}/sites/{s}", trialId, siteId).then().statusCode(404);
}
```

- [ ] **Step 2: Run to verify all three tests fail**

```bash
mvn test -pl runtime -Dtest=SiteResourceTest --batch-mode
```

Expected: `get_site_returns_404_for_wrong_tenant`, `add_site_returns_404_when_trial_belongs_to_different_tenant`, and `site_inherits_trial_tenantId_not_principal_tenantId` all FAIL.

- [ ] **Step 3: Update `SiteResource.add()` — read path + write stamp fix**

Replace the current `add()` method body in `SiteResource.java`:

```java
@POST
@Transactional
public Response add(@PathParam("trialId") UUID trialId,
                    @Valid AddSiteRequest req,
                    @Context UriInfo uriInfo) {
    ClinicalTrial trial = ClinicalTrial.findByIdForTenant(trialId, principal);
    if (trial == null)
        return Response.status(Response.Status.NOT_FOUND).build();

    TrialSite site = new TrialSite();
    site.id = UUID.randomUUID();
    site.tenantId = trial.tenantId;
    site.trialId = trialId;
    site.investigatorId = req.investigatorId();
    site.status = SiteStatus.PENDING;
    site.persist();

    URI location = uriInfo.getAbsolutePathBuilder().path(site.id.toString()).build();
    return Response.created(location).build();
}
```

- [ ] **Step 4: Update `SiteResource.get()` — read path fix**

Replace the current `get()` method body:

```java
@GET
@Path("/{siteId}")
public Response get(@PathParam("trialId") UUID trialId,
                    @PathParam("siteId") UUID siteId) {
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();
    return Response.ok(site).build();
}
```

- [ ] **Step 5: Run to verify isolation + write-invariant tests pass**

```bash
mvn test -pl runtime -Dtest=SiteResourceTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 6: Write bypass test**

Add to `SiteResourceTest.java`:
```java
@Test
void get_site_succeeds_for_cross_tenant_admin() {
    String trialLoc = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-S-004\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    String siteLoc = given()
        .contentType("application/json")
        .body("{\"investigatorId\":\"pi-iso\"}")
        .when().post("/trials/{id}/sites", trialId).then().statusCode(201).extract().header("Location");
    UUID siteId = UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    principal.setCrossTenantAdmin(true);
    given().when().get("/trials/{t}/sites/{s}", trialId, siteId).then().statusCode(200);
}
```

- [ ] **Step 7: Run to verify bypass test passes**

```bash
mvn test -pl runtime -Dtest=SiteResourceTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/resource/SiteResource.java \
  runtime/src/test/java/io/casehub/clinical/resource/SiteResourceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(resource): tenant-scope TrialSite reads and stamp fix in SiteResource

Refs #71"
```

---

## Task 4: PatientResource — read path, write-path stamp fix, all tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/PatientResourceTest.java`

`PatientResource` already injects `@Inject CurrentPrincipal principal`.

- [ ] **Step 1: Write failing tests in `PatientResourceTest`**

Add these imports:
```java
import io.casehub.platform.testing.FixedCurrentPrincipal;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
```

Add these fields and tests:
```java
@Inject FixedCurrentPrincipal principal;

@AfterEach
void resetPrincipal() { principal.reset(); }

@Test
void get_enrollment_returns_404_for_wrong_tenant() {
    UUID siteId = createTrialAndSite();
    String patientLoc = given()
        .contentType("application/json")
        .body("{\"patientId\":\"PAT-ISO-001\"}")
        .when().post("/trials/{t}/sites/{s}/patients",
            findTrialId(siteId), siteId)
        .then().statusCode(201).extract().header("Location");

    principal.setTenancyId("other-tenant");
    given().when().get(patientLoc).then().statusCode(404);
}

@Test
void report_ae_returns_404_for_wrong_tenant_enrollment() {
    UUID siteId = createTrialAndSite();
    UUID trialId = findTrialId(siteId);
    String patientLoc = given()
        .contentType("application/json")
        .body("{\"patientId\":\"PAT-ISO-002\"}")
        .when().post("/trials/{t}/sites/{s}/patients", trialId, siteId)
        .then().statusCode(201).extract().header("Location");
    UUID enrollmentId = UUID.fromString(patientLoc.substring(patientLoc.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    given()
        .contentType("application/json")
        .body("{\"grade\":\"GRADE_1\",\"occurredAt\":\"2026-01-01T10:00:00Z\"}")
        .when().post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
            trialId, siteId, enrollmentId)
        .then().statusCode(404);
}

@Test
void enrollment_inherits_site_tenantId_not_principal_tenantId() {
    UUID siteId = createTrialAndSite();
    UUID trialId = findTrialId(siteId);

    // cross-tenant admin enrolls a patient under a different tenant's site
    principal.setTenancyId("admin-tenant");
    principal.setCrossTenantAdmin(true);
    String patientLoc = given()
        .contentType("application/json")
        .body("{\"patientId\":\"PAT-ISO-INHERIT\"}")
        .when().post("/trials/{t}/sites/{s}/patients", trialId, siteId)
        .then().statusCode(201).extract().header("Location");
    UUID enrollmentId = UUID.fromString(patientLoc.substring(patientLoc.lastIndexOf('/') + 1));

    // default tenant can find it (enrollment.tenantId = site.tenantId = default tenant)
    principal.reset();
    given().when().get(patientLoc).then().statusCode(200);

    // admin's own tenant (no bypass) cannot find it
    principal.setTenancyId("admin-tenant");
    given().when().get(patientLoc).then().statusCode(404);
}
```

Add these private helpers to the class (used by the new tests):
```java
private UUID createTrialAndSite() {
    String trialLoc = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-P-" + java.util.UUID.randomUUID() + "\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    String siteLoc = given()
        .contentType("application/json")
        .body("{\"investigatorId\":\"pi-iso\"}")
        .when().post("/trials/{id}/sites", trialId).then().statusCode(201).extract().header("Location");
    return UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));
}

private UUID findTrialId(UUID siteId) {
    // derive trialId from siteId — sites were created under the same trial in createTrialAndSite
    // Re-create a trial+site for independence; use a stored trialId per test if needed.
    // Since tests create their own graphs, pass trialId explicitly where needed.
    throw new UnsupportedOperationException("Use the overload that creates trial+site together");
}
```

Actually the helper is cleaner if it returns both IDs. Replace the two helpers with:
```java
private UUID[] createTrialAndSiteIds() {
    String trialLoc = given()
        .contentType("application/json")
        .body("{\"protocolId\":\"ISO-P-" + java.util.UUID.randomUUID() + "\",\"phase\":\"PHASE_I\",\"sponsor\":\"T\",\"targetEnrollment\":5}")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
    UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

    String siteLoc = given()
        .contentType("application/json")
        .body("{\"investigatorId\":\"pi-iso\"}")
        .when().post("/trials/{id}/sites", trialId).then().statusCode(201).extract().header("Location");
    UUID siteId = UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));
    return new UUID[]{trialId, siteId};
}
```

And rewrite the three new tests to use it:
```java
@Test
void get_enrollment_returns_404_for_wrong_tenant() {
    UUID[] ids = createTrialAndSiteIds();
    UUID trialId = ids[0], siteId = ids[1];

    String patientLoc = given()
        .contentType("application/json")
        .body("{\"patientId\":\"PAT-ISO-001\"}")
        .when().post("/trials/{t}/sites/{s}/patients", trialId, siteId)
        .then().statusCode(201).extract().header("Location");

    principal.setTenancyId("other-tenant");
    given().when().get(patientLoc).then().statusCode(404);
}

@Test
void report_ae_returns_404_for_wrong_tenant_enrollment() {
    UUID[] ids = createTrialAndSiteIds();
    UUID trialId = ids[0], siteId = ids[1];

    String patientLoc = given()
        .contentType("application/json")
        .body("{\"patientId\":\"PAT-ISO-002\"}")
        .when().post("/trials/{t}/sites/{s}/patients", trialId, siteId)
        .then().statusCode(201).extract().header("Location");
    UUID enrollmentId = UUID.fromString(patientLoc.substring(patientLoc.lastIndexOf('/') + 1));

    principal.setTenancyId("other-tenant");
    given()
        .contentType("application/json")
        .body("{\"grade\":\"GRADE_1\",\"occurredAt\":\"2026-01-01T10:00:00Z\"}")
        .when().post("/trials/{t}/sites/{s}/patients/{e}/adverse-events",
            trialId, siteId, enrollmentId)
        .then().statusCode(404);
}

@Test
void enrollment_inherits_site_tenantId_not_principal_tenantId() {
    UUID[] ids = createTrialAndSiteIds();
    UUID trialId = ids[0], siteId = ids[1];

    principal.setTenancyId("admin-tenant");
    principal.setCrossTenantAdmin(true);
    String patientLoc = given()
        .contentType("application/json")
        .body("{\"patientId\":\"PAT-ISO-INHERIT\"}")
        .when().post("/trials/{t}/sites/{s}/patients", trialId, siteId)
        .then().statusCode(201).extract().header("Location");

    principal.reset();
    given().when().get(patientLoc).then().statusCode(200);

    principal.setTenancyId("admin-tenant");
    given().when().get(patientLoc).then().statusCode(404);
}
```

- [ ] **Step 2: Run to verify the three new tests fail**

```bash
mvn test -pl runtime -Dtest=PatientResourceTest --batch-mode
```

Expected: the three new tests FAIL. Existing tests pass.

- [ ] **Step 3: Update `PatientResource.enroll()` — read path + write stamp fix**

Replace the `enroll()` method body in `PatientResource.java`:

```java
@POST
@Transactional
public Response enroll(@PathParam("trialId") UUID trialId,
                       @PathParam("siteId") UUID siteId,
                       @Valid EnrollPatientRequest req,
                       @Context UriInfo uriInfo) {
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();

    PatientEnrollment enrollment = new PatientEnrollment();
    enrollment.id = UUID.randomUUID();
    enrollment.tenantId = site.tenantId;
    enrollment.siteId = siteId;
    enrollment.patientId = req.patientId();
    enrollment.consentStatus = ConsentStatus.PENDING;
    enrollment.enrollmentStatus = EnrollmentStatus.CANDIDATE;
    enrollment.persist();

    URI location = uriInfo.getAbsolutePathBuilder().path(enrollment.id.toString()).build();
    return Response.created(location).build();
}
```

- [ ] **Step 4: Update `PatientResource.get()` — read path fix**

Replace the `get()` method body:

```java
@GET
@Path("/{enrollmentId}")
public Response get(@PathParam("trialId") UUID trialId,
                    @PathParam("siteId") UUID siteId,
                    @PathParam("enrollmentId") UUID enrollmentId) {
    PatientEnrollment enrollment = PatientEnrollment.findByIdForTenant(enrollmentId, principal);
    if (enrollment == null || !enrollment.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();
    return Response.ok(enrollment).build();
}
```

- [ ] **Step 5: Update `PatientResource.reportAdverseEvent()` — read path fix**

Replace the `reportAdverseEvent()` method body:

```java
@POST
@Path("/{enrollmentId}/adverse-events")
public Response reportAdverseEvent(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @PathParam("enrollmentId") UUID enrollmentId,
        @Valid ReportAdverseEventRequest req,
        @Context UriInfo uriInfo) {
    PatientEnrollment enrollment = PatientEnrollment.findByIdForTenant(enrollmentId, principal);
    if (enrollment == null || !enrollment.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();

    AdverseEvent ae = new AdverseEvent();
    ae.id = UUID.randomUUID();
    ae.enrollmentId = enrollmentId;
    ae.grade = req.grade();
    ae.actuality = req.actuality() != null ? req.actuality() : EventActuality.ACTUAL;
    ae.outcome = AeOutcome.ONGOING;
    ae.occurredAt = req.occurredAt();

    adverseEventService.reportAdverseEvent(ae);

    URI location = uriInfo.getAbsolutePathBuilder().path(ae.id.toString()).build();
    return Response.created(location).entity(ae).build();
}
```

- [ ] **Step 6: Run to verify all new tests pass**

```bash
mvn test -pl runtime -Dtest=PatientResourceTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 7: Write bypass test**

Add to `PatientResourceTest.java`:
```java
@Test
void get_enrollment_succeeds_for_cross_tenant_admin() {
    UUID[] ids = createTrialAndSiteIds();
    UUID trialId = ids[0], siteId = ids[1];

    String patientLoc = given()
        .contentType("application/json")
        .body("{\"patientId\":\"PAT-BYPASS-001\"}")
        .when().post("/trials/{t}/sites/{s}/patients", trialId, siteId)
        .then().statusCode(201).extract().header("Location");

    principal.setTenancyId("other-tenant");
    principal.setCrossTenantAdmin(true);
    given().when().get(patientLoc).then().statusCode(200);
}
```

- [ ] **Step 8: Run to verify bypass test passes**

```bash
mvn test -pl runtime -Dtest=PatientResourceTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java \
  runtime/src/test/java/io/casehub/clinical/resource/PatientResourceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(resource): tenant-scope PatientEnrollment/TrialSite reads and stamp fix in PatientResource

Refs #71"
```

---

## Task 5: DeviationResource — read path, write-path stamp fix, all tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/DeviationResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/DeviationResourceTest.java`

`DeviationResource` already injects `@Inject CurrentPrincipal principal`.

- [ ] **Step 1: Write failing tests in `DeviationResourceTest`**

Add these imports:
```java
import io.casehub.platform.testing.FixedCurrentPrincipal;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
```

Add fields and tests:
```java
@Inject FixedCurrentPrincipal principal;

@AfterEach
void resetPrincipal() { principal.reset(); }

@Test
void get_deviation_returns_404_for_wrong_tenant() {
    UUID[] ids = createTrialAndSite();
    UUID trialId = ids[0], siteId = ids[1];

    var resp = given()
        .contentType("application/json")
        .body("{\"deviationType\":\"consent-gap\",\"severity\":\"MINOR\"}")
        .when().post("/trials/{t}/sites/{s}/deviations", trialId, siteId)
        .then().statusCode(201).extract();
    String location = resp.header("Location");
    String deviationId = location.substring(location.lastIndexOf('/') + 1);

    principal.setTenancyId("other-tenant");
    given().when().get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteId, deviationId)
        .then().statusCode(404);
}

@Test
void deviation_inherits_site_tenantId_not_principal_tenantId() {
    UUID[] ids = createTrialAndSite();
    UUID trialId = ids[0], siteId = ids[1];

    principal.setTenancyId("admin-tenant");
    principal.setCrossTenantAdmin(true);
    var resp = given()
        .contentType("application/json")
        .body("{\"deviationType\":\"sample-window\",\"severity\":\"MINOR\"}")
        .when().post("/trials/{t}/sites/{s}/deviations", trialId, siteId)
        .then().statusCode(201).extract();
    String location = resp.header("Location");
    String deviationId = location.substring(location.lastIndexOf('/') + 1);

    principal.reset();
    given().when().get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteId, deviationId)
        .then().statusCode(200);

    principal.setTenancyId("admin-tenant");
    given().when().get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteId, deviationId)
        .then().statusCode(404);
}
```

- [ ] **Step 2: Run to verify both new tests fail**

```bash
mvn test -pl runtime -Dtest=DeviationResourceTest --batch-mode
```

Expected: `get_deviation_returns_404_for_wrong_tenant` and `deviation_inherits_site_tenantId_not_principal_tenantId` FAIL.

- [ ] **Step 3: Update `DeviationResource.reportDeviation()` — read path + write stamp fix**

Replace the `reportDeviation()` method body in `DeviationResource.java`:

```java
@POST
@Transactional
public Response reportDeviation(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @Valid ReportDeviationRequest req,
        @Context UriInfo uriInfo) {
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();

    ProtocolDeviation deviation = new ProtocolDeviation();
    deviation.id = UUID.randomUUID();
    deviation.tenantId = site.tenantId;
    deviation.siteId = siteId;
    deviation.deviationType = req.deviationType();
    deviation.severity = req.severity();
    deviation.piApprovalStatus = PiApprovalStatus.PENDING;

    deviationService.reportDeviation(deviation);

    URI location = uriInfo.getAbsolutePathBuilder().path(deviation.id.toString()).build();
    return Response.created(location).entity(deviation).build();
}
```

- [ ] **Step 4: Update `DeviationResource.getDeviation()` — read path fix**

Replace the `getDeviation()` method body:

```java
@GET
@Path("/{deviationId}")
public Response getDeviation(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @PathParam("deviationId") UUID deviationId) {
    ProtocolDeviation dev = ProtocolDeviation.findByIdForTenant(deviationId, principal);
    if (dev == null || !dev.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();
    return Response.ok(dev).build();
}
```

- [ ] **Step 5: Run to verify all tests pass**

```bash
mvn test -pl runtime -Dtest=DeviationResourceTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 6: Write bypass test**

Add to `DeviationResourceTest.java`:
```java
@Test
void get_deviation_succeeds_for_cross_tenant_admin() {
    UUID[] ids = createTrialAndSite();
    UUID trialId = ids[0], siteId = ids[1];

    var resp = given()
        .contentType("application/json")
        .body("{\"deviationType\":\"consent-gap\",\"severity\":\"MINOR\"}")
        .when().post("/trials/{t}/sites/{s}/deviations", trialId, siteId)
        .then().statusCode(201).extract();
    String location = resp.header("Location");
    String deviationId = location.substring(location.lastIndexOf('/') + 1);

    principal.setTenancyId("other-tenant");
    principal.setCrossTenantAdmin(true);
    given().when().get("/trials/{t}/sites/{s}/deviations/{d}", trialId, siteId, deviationId)
        .then().statusCode(200);
}
```

- [ ] **Step 7: Run to verify bypass test passes**

```bash
mvn test -pl runtime -Dtest=DeviationResourceTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/resource/DeviationResource.java \
  runtime/src/test/java/io/casehub/clinical/resource/DeviationResourceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(resource): tenant-scope ProtocolDeviation/TrialSite reads and stamp fix in DeviationResource

Refs #71"
```

---

## Task 6: TrialActivationService — read path + isolation + bypass tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/TrialActivationService.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/TrialActivationTest.java`

- [ ] **Step 1: Write failing isolation and bypass tests in `TrialActivationTest`**

Add these imports to `TrialActivationTest.java`:
```java
import io.casehub.platform.testing.FixedCurrentPrincipal;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
```

Add fields and tests:
```java
@Inject FixedCurrentPrincipal principal;

@AfterEach
void resetPrincipal() { principal.reset(); }

@Test
void activating_wrong_tenant_trial_returns_404() {
    UUID trialId = createTrial();
    principal.setTenancyId("other-tenant");
    given().when().post("/trials/" + trialId + "/activate").then().statusCode(404);
}

@Test
void activating_cross_tenant_trial_succeeds_for_admin() {
    UUID trialId = createTrial();
    principal.setTenancyId("other-tenant");
    principal.setCrossTenantAdmin(true);
    given().when().post("/trials/" + trialId + "/activate").then().statusCode(204);
}
```

- [ ] **Step 2: Run to verify isolation test fails and bypass test passes**

```bash
mvn test -pl runtime -Dtest=TrialActivationTest --batch-mode
```

Expected: `activating_wrong_tenant_trial_returns_404` FAILS (returns 204 instead of 404). `activating_cross_tenant_trial_succeeds_for_admin` PASSES (but for the wrong reason — no tenant isolation yet). All existing tests pass.

- [ ] **Step 3: Add `CurrentPrincipal` injection to `TrialActivationService`**

Add this import to `TrialActivationService.java`:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.inject.Inject;
```

Add this field after the existing `@Inject ClinicalTrialCaseHub caseHub;`:
```java
@Inject CurrentPrincipal principal;
```

- [ ] **Step 4: Update `markActive()` to use `findByIdForTenant`**

Replace the `markActive()` method body:

```java
@Transactional
Map<String, Object> markActive(UUID trialId) {
    ClinicalTrial trial = ClinicalTrial.findByIdForTenant(trialId, principal);
    if (trial == null) throw new TrialNotFoundException(trialId);
    if (trial.status != TrialStatus.PLANNING) throw new TrialNotInPlanningStatusException(trial.status);
    trial.status = TrialStatus.ACTIVE;

    Map<String, Object> ctx = new HashMap<>();
    ctx.put("trialId", trialId.toString());
    ctx.put("protocolId", trial.protocolId);
    ctx.put("grade4Active", new HashMap<>());
    return ctx;
}
```

`persistCaseId()` is unchanged — it uses plain `findById(trialId)`, which is safe because `trialId` was already tenant-validated in phase 1.

- [ ] **Step 5: Run to verify both tests pass**

```bash
mvn test -pl runtime -Dtest=TrialActivationTest --batch-mode
```

Expected: all tests PASS including `activating_wrong_tenant_trial_returns_404`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/TrialActivationService.java \
  runtime/src/test/java/io/casehub/clinical/service/TrialActivationTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): tenant-scope ClinicalTrial read in TrialActivationService

Refs #71"
```

---

## Task 7: AdverseEventService — refactoring + write-invariant test

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java`

`AdverseEventService` currently injects `CurrentPrincipal` and uses `principal.tenancyId()` to stamp the AE tenant. After this task: tenant is derived from the enrollment entity, and `CurrentPrincipal` injection is removed.

- [ ] **Step 1: Write failing write-invariant test in `AdverseEventServiceTest`**

Add this test (no new imports needed — the class already has the relevant imports):

```java
@Test
@Transactional
void ae_tenantId_is_derived_from_enrollment_not_principal() {
    AdverseEvent ae = newAe(CtcaeGrade.GRADE_1);
    PatientEnrollment enrollment = PatientEnrollment.findById(ae.enrollmentId);
    service.reportAdverseEvent(ae);
    // enrollment.tenantId = "default" (entity field default, set in newAe())
    // principal.tenancyId() = "278776f9-..." (FixedCurrentPrincipal default)
    // These differ, so the assertion catches which source is used
    assertThat(ae.tenantId).isEqualTo(enrollment.tenantId);
}
```

- [ ] **Step 2: Run to verify the test fails**

```bash
mvn test -pl runtime -Dtest=AdverseEventServiceTest --batch-mode
```

Expected: `ae_tenantId_is_derived_from_enrollment_not_principal` FAILS because `ae.tenantId` = `"278776f9-..."` but `enrollment.tenantId` = `"default"`.

- [ ] **Step 3: Refactor `AdverseEventService.reportAdverseEvent()`**

In `AdverseEventService.java`, replace the body of `reportAdverseEvent()` as follows. The key changes are: load enrollment once at the top (replacing the two private methods), derive both `siteId` and `tenantId` from it.

Current beginning of `reportAdverseEvent()`:
```java
ae.reportedAt = Instant.now();
ae.slaDeadline = ae.reportedAt.plus(ae.grade.sla().orElseThrow());
ae.tenantId = principal.tenancyId();

UUID siteId = resolveSiteId(ae.enrollmentId);
UUID trialId = resolveTrialId(siteId);
```

Replace with:
```java
ae.reportedAt = Instant.now();
ae.slaDeadline = ae.reportedAt.plus(ae.grade.sla().orElseThrow());

PatientEnrollment enrollment = PatientEnrollment.findById(ae.enrollmentId);
UUID siteId = enrollment != null ? enrollment.siteId : null;
ae.tenantId = enrollment != null ? enrollment.tenantId : "default";

TrialSite site = siteId != null ? TrialSite.findById(siteId) : null;
UUID trialId = site != null ? site.trialId : null;
```

The rest of the method body stays unchanged.

- [ ] **Step 4: Remove the two private resolver methods**

Delete both private methods from `AdverseEventService.java`:
```java
private UUID resolveSiteId(UUID enrollmentId) { ... }
private UUID resolveTrialId(UUID siteId) { ... }
```

- [ ] **Step 5: Remove `CurrentPrincipal` injection**

Delete this field from `AdverseEventService.java`:
```java
@Inject CurrentPrincipal principal;
```

Delete this import:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
```

- [ ] **Step 6: Compile to check for errors**

```bash
mvn compile -pl runtime --batch-mode
```

Expected: BUILD SUCCESS. If there are any unresolved references to `principal` or the deleted methods, fix them now.

- [ ] **Step 7: Run to verify the write-invariant test and all existing tests pass**

```bash
mvn test -pl runtime -Dtest=AdverseEventServiceTest --batch-mode
```

Expected: all tests PASS including `ae_tenantId_is_derived_from_enrollment_not_principal`.

- [ ] **Step 8: Run the full test suite**

```bash
mvn test -pl runtime --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java \
  runtime/src/test/java/io/casehub/clinical/service/AdverseEventServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "refactor(service): derive ae.tenantId from enrollment entity, remove CurrentPrincipal from AdverseEventService

Refs #71"
```

---

## Task 8: Full suite verification and issue close

- [ ] **Step 1: Run the complete test suite**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 2: Verify the 12 read-path call sites are all updated**

Check each resource file — confirm every `findById` used for user-supplied IDs in REST handlers has been replaced with `findByIdForTenant`:

- `TrialResource.java`: `get()` and `updateSponsorConfig()` ✓
- `SiteResource.java`: `add()` (parent trial check) and `get()` ✓
- `PatientResource.java`: `enroll()`, `get()` (both lookups), `reportAdverseEvent()` (both lookups) ✓
- `DeviationResource.java`: `reportDeviation()` and `getDeviation()` (both lookups) ✓

Count: 2 + 2 + 5 + 3 = 12. ✓

- [ ] **Step 3: Commit the workspace branch**

```bash
git -C /Users/mdproctor/claude/public/casehub/clinical add -A
git -C /Users/mdproctor/claude/public/casehub/clinical commit -m "docs: tenant isolation impl complete — Refs #71"
```

---

## Self-review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `findByIdForTenant` on 6 entities | Task 1 |
| `TrialResource` read path (GET, PATCH) | Task 2 |
| `SiteResource` read path + write stamp | Task 3 |
| `PatientResource` read path + write stamp | Task 4 |
| `DeviationResource` read path + write stamp | Task 5 |
| `TrialActivationService` injection + `markActive` | Task 6 |
| `AdverseEventService` refactoring + tenant from enrollment | Task 7 |
| Isolation tests (wrong tenant → 404) | Tasks 2–6 |
| Bypass tests (cross-tenant admin → 200) | Tasks 2–6 |
| Write-invariant tests (child inherits parent tenant) | Tasks 3–4–5–7 |
| `AdverseEvent` + `IrbApproval` helpers (forward-safety) | Task 1 (included in entity helpers batch) |
| `FixedCurrentPrincipal` injection in all 5 test classes | Tasks 2–6 |
| `persistCaseId` unchanged | Task 6 — verified explicitly |
| System-actor paths unchanged | Spec "deliberately unchanged" — no plan tasks touch them |

**Placeholder scan:** No TBDs, TODOs, or "similar to Task N" references. All code blocks are complete.

**Type consistency:** `findByIdForTenant(UUID id, CurrentPrincipal principal)` is the signature throughout. `TrialSite.findByIdForTenant`, `PatientEnrollment.findByIdForTenant`, `ProtocolDeviation.findByIdForTenant` — all consistent. `enrollment.tenantId`, `site.tenantId`, `trial.tenantId` — all consistent field names from the entity classes as read in the spec context.
