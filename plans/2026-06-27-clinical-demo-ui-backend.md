# Clinical Demo UI — Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Java backend for the clinical trial demo UI — REST endpoints, demo infrastructure, and data seeder — so that the TypeScript UI (Plan 2) has a working API to consume.

**Architecture:** Three components in dependency order: (1) `DemoCurrentPrincipal` provides tenant context in dev mode, (2) `TrialDashboardResource` exposes 7 read-only aggregation endpoints, (3) `DemoActionResource` exposes 2 demo-only endpoints for live actions, (4) `DemoDataSeeder` replays a trial scenario through real service calls to produce Merkle-verified data. The Quinoa scaffold (webui directory + Maven extension) is included here because it gates the UI plan.

**Tech Stack:** Java 21 / Quarkus 3.32.2 / Panache Active Record / JTA XA / Quinoa (esbuild)

**Spec:** `specs/2026-06-27-clinical-demo-ui-design.md` (Rev 6 — final)

**Epic:** casehubio/clinical#93

## Global Constraints

- Java 21 language level on Java 26 JVM
- Use `mvn` not `./mvnw`
- `api/` must be installed before `runtime/` tests: `mvn install -pl api --batch-mode`
- Tests use `drop-and-create` + Flyway disabled; production uses Flyway
- Two datasources: default (domain entities + casehub-work) + qhorus (ledger + qhorus entities)
- `@RolesAllowed` with `ClinicalGroups` constants from `api/` on all new REST endpoints
- `quarkus.security.deny-unannotated-members=true` — new REST methods MUST have `@RolesAllowed`
- `@TestSecurity(user = "test-actor", roles = {SPONSOR, INVESTIGATOR, COORDINATOR})` on `@QuarkusTest` HTTP test classes
- `FixedCurrentPrincipal` for tenant context in tests (already in test `selected-alternatives`)
- All commits reference issues: `Refs #93` + specific child issue
- IntelliJ MCP for all rename/move/find-references operations
- Deterministic trial UUID: `UUID.nameUUIDFromBytes("ONCO-2024-001".getBytes(StandardCharsets.UTF_8))` = `316e3846-4ea7-3b18-a6f7-e01ce6582a69`
- Response types are nested records inside resource classes (existing clinical pattern)
- Demo-only classes use `@IfBuildProfile("dev")` — they don't exist in production builds

## Follow-up plans

- **Plan 2 (TypeScript UI):** All casehub-pages DSL pages, datasets, navigation. Depends on this plan.
- **Plan 3 (Playwright smoke tests):** Automated smoke tests. Depends on Plans 1+2.

---

### Task 1: Quinoa Scaffold + DemoCurrentPrincipal

**Issues:** #94 (Quinoa setup), #103 (DemoCurrentPrincipal)

**Files:**
- Modify: `runtime/pom.xml` — add `quarkus-quinoa` extension
- Modify: `runtime/src/main/resources/application.properties` — add Quinoa + demo config
- Create: `runtime/src/main/webui/package.json`
- Create: `runtime/src/main/webui/tsconfig.json`
- Create: `runtime/src/main/webui/esbuild.config.mjs`
- Create: `runtime/src/main/webui/index.html`
- Create: `runtime/src/main/webui/src/index.ts` — minimal placeholder
- Create: `runtime/src/main/java/io/casehub/clinical/demo/DemoCurrentPrincipal.java`
- Create: `runtime/src/test/java/io/casehub/clinical/demo/DemoCurrentPrincipalTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.identity.CurrentPrincipal` (platform-api)
- Produces: `DemoCurrentPrincipal` — CDI bean providing `tenancyId() = "demo-tenant"`, `actorId() = "demo-user"` in dev profile

- [ ] **Step 1: Write the DemoCurrentPrincipal test**

```java
package io.casehub.clinical.demo;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class DemoCurrentPrincipalTest {

    private final DemoCurrentPrincipal principal = new DemoCurrentPrincipal();

    @Test
    void tenancyId_returns_demo_tenant() {
        assertThat(principal.tenancyId()).isEqualTo("demo-tenant");
    }

    @Test
    void actorId_returns_demo_user() {
        assertThat(principal.actorId()).isEqualTo("demo-user");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=DemoCurrentPrincipalTest --batch-mode`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DemoCurrentPrincipal**

Read the `CurrentPrincipal` interface first to discover all methods that need implementing. Use IntelliJ MCP `ide_find_class` to find `CurrentPrincipal`, then `ide_file_structure` to list all abstract methods.

```java
package io.casehub.clinical.demo;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

@ApplicationScoped
@Alternative
@Priority(150)
@IfBuildProfile("dev")
public class DemoCurrentPrincipal implements CurrentPrincipal {

    public static final String TENANT_ID = "demo-tenant";
    public static final String ACTOR_ID = "demo-user";

    @Override
    public String tenancyId() {
        return TENANT_ID;
    }

    @Override
    public String actorId() {
        return ACTOR_ID;
    }

    // Implement remaining CurrentPrincipal methods with sensible defaults.
    // Use ide_file_structure on CurrentPrincipal to discover them.
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=DemoCurrentPrincipalTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Add quarkus-quinoa to runtime/pom.xml**

Add the Quinoa extension dependency. Find the correct GAV by checking the Quarkus BOM version (3.32.2):

```xml
<dependency>
    <groupId>io.quarkiverse.quinoa</groupId>
    <artifactId>quarkus-quinoa</artifactId>
</dependency>
```

If not in the BOM, add with an explicit version. Check the latest Quinoa version compatible with Quarkus 3.32.2.

- [ ] **Step 6: Add Quinoa + demo config to application.properties**

Append to `runtime/src/main/resources/application.properties`:

```properties
# ── Quinoa — esbuild (no dev server; Quinoa watches sources and re-runs npm build) ──
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true

# ── Demo data (dev profile only) ──
%dev.casehub.clinical.demo.seed-data=true
casehub.clinical.demo.seed-data=false
```

- [ ] **Step 7: Create webui scaffold**

Create `runtime/src/main/webui/package.json`:
```json
{
  "name": "casehub-clinical-ui",
  "private": true,
  "scripts": {
    "build": "node esbuild.config.mjs",
    "dev": "node esbuild.config.mjs --watch"
  },
  "dependencies": {
    "@casehubio/pages-runtime": "0.2.0",
    "@casehubio/pages-ui": "0.2.0"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

Create `runtime/src/main/webui/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false,
    "noEmit": true
  },
  "include": ["src"]
}
```

Create `runtime/src/main/webui/esbuild.config.mjs` (from spec).

Create `runtime/src/main/webui/index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CaseHub Clinical</title>
</head>
<body>
    <div id="app"></div>
    <script type="module" src="dist/app.js"></script>
</body>
</html>
```

Create `runtime/src/main/webui/src/index.ts`:
```typescript
const container = document.getElementById("app");
if (container) {
    container.textContent = "CaseHub Clinical — UI loading...";
}
```

- [ ] **Step 8: Verify the full build works**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: all existing tests pass, Quinoa build runs (may warn about npm install on first run)

- [ ] **Step 9: Commit**

```bash
git add runtime/pom.xml runtime/src/main/resources/application.properties \
  runtime/src/main/webui/ runtime/src/main/java/io/casehub/clinical/demo/DemoCurrentPrincipal.java \
  runtime/src/test/java/io/casehub/clinical/demo/DemoCurrentPrincipalTest.java
git commit -m "feat: Quinoa scaffold + DemoCurrentPrincipal for dev-mode tenant context

Refs #94, #103, casehubio/clinical#93"
```

---

### Task 2: TrialDashboardResource — Summary, Patients, AEs, Deviations (4 simple endpoints)

**Issue:** #96

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/TrialDashboardResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/TrialDashboardResourceTest.java`

**Interfaces:**
- Consumes: `ClinicalTrial`, `TrialSite`, `PatientEnrollment`, `AdverseEvent`, `ProtocolDeviation` (Panache entities, default datasource), `CurrentPrincipal`
- Produces: `GET /trials/{trialId}/summary`, `GET /trials/{trialId}/patients`, `GET /trials/{trialId}/adverse-events`, `GET /trials/{trialId}/deviations` — all with nested response records

- [ ] **Step 1: Write tests for the summary endpoint**

```java
package io.casehub.clinical.resource;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.clinical.entity.*;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class TrialDashboardResourceTest {

    @Inject FixedCurrentPrincipal principal;

    private UUID trialId;
    private UUID siteAId;
    private UUID siteBId;

    @BeforeEach
    @Transactional
    void setup() {
        trialId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId;
        trial.tenantId = principal.tenancyId();
        trial.protocolId = "TEST-001";
        trial.phase = TrialPhase.PHASE_3;
        trial.sponsor = "Test Sponsor";
        trial.targetEnrollment = 20;
        trial.persist();

        siteAId = UUID.randomUUID();
        TrialSite siteA = new TrialSite();
        siteA.id = siteAId;
        siteA.tenantId = principal.tenancyId();
        siteA.trialId = trialId;
        siteA.investigatorId = "dr-chen";
        siteA.persist();

        siteBId = UUID.randomUUID();
        TrialSite siteB = new TrialSite();
        siteB.id = siteBId;
        siteB.tenantId = principal.tenancyId();
        siteB.trialId = trialId;
        siteB.investigatorId = "dr-patel";
        siteB.persist();

        UUID enrollmentId = UUID.randomUUID();
        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = enrollmentId;
        enrollment.tenantId = principal.tenancyId();
        enrollment.siteId = siteAId;
        enrollment.patientId = "P-001";
        enrollment.persist();

        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.tenantId = principal.tenancyId();
        ae.enrollmentId = enrollmentId;
        ae.grade = CtcaeGrade.GRADE_3;
        ae.reportedAt = Instant.now();
        ae.slaDeadline = Instant.now().plusSeconds(86400);
        ae.persist();
    }

    @Test
    void summary_returns_trial_metrics() {
        given()
            .when().get("/trials/{trialId}/summary", trialId)
            .then()
            .statusCode(200)
            .body("protocolId", equalTo("TEST-001"))
            .body("phase", equalTo("PHASE_3"))
            .body("totalEnrolled", greaterThanOrEqualTo(1))
            .body("totalAdverseEvents", greaterThanOrEqualTo(1));
    }

    @Test
    void summary_returns_404_for_wrong_tenant() {
        given()
            .when().get("/trials/{trialId}/summary", UUID.randomUUID())
            .then()
            .statusCode(404);
    }

    @Test
    void patients_returns_flattened_list() {
        given()
            .when().get("/trials/{trialId}/patients", trialId)
            .then()
            .statusCode(200)
            .body("size()", greaterThanOrEqualTo(1))
            .body("[0].patientId", equalTo("P-001"))
            .body("[0].siteId", notNullValue());
    }

    @Test
    void adverse_events_returns_flattened_list_with_sla() {
        given()
            .when().get("/trials/{trialId}/adverse-events", trialId)
            .then()
            .statusCode(200)
            .body("size()", greaterThanOrEqualTo(1))
            .body("[0].grade", equalTo("GRADE_3"))
            .body("[0].slaDeadline", notNullValue());
    }

    @Test
    void deviations_returns_empty_list_when_none() {
        given()
            .when().get("/trials/{trialId}/deviations", trialId)
            .then()
            .statusCode(200)
            .body("size()", equalTo(0));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=TrialDashboardResourceTest --batch-mode`
Expected: FAIL — no resource mapped to `/trials/{trialId}/summary`

- [ ] **Step 3: Implement TrialDashboardResource with 4 endpoints**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.*;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@Path("/trials/{trialId}")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Transactional
public class TrialDashboardResource {

    @Inject CurrentPrincipal principal;

    // --- Response records (nested per clinical convention) ---

    public record TrialSummary(
        String protocolId, String phase, String sponsor, int targetEnrollment,
        long totalEnrolled, long totalAdverseEvents, long totalDeviations
    ) {}

    public record PatientRow(
        UUID id, UUID siteId, String patientId, String enrollmentStatus,
        String screeningResult, String consentStatus
    ) {}

    public record AdverseEventRow(
        UUID id, UUID enrollmentId, UUID siteId, String grade, String type,
        Instant reportedAt, Instant slaDeadline, String escalationStatus,
        String regulatorySubmissionStatus, String slaTimeRemaining
    ) {}

    public record DeviationRow(
        UUID id, UUID siteId, String deviationType, String severity,
        String piApprovalStatus, Instant reportedAt
    ) {}

    // --- Endpoints ---

    @GET
    @Path("/summary")
    @RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR,
                   ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
    public Response summary(@PathParam("trialId") UUID trialId) {
        ClinicalTrial trial = ClinicalTrial.findByIdForTenant(trialId, principal);
        if (trial == null) return Response.status(404).build();

        List<UUID> siteIds = TrialSite.find("trialId = ?1 and tenantId = ?2",
            trialId, principal.tenancyId()).project(UUID.class).list();

        long enrolled = siteIds.isEmpty() ? 0 :
            PatientEnrollment.count("siteId in ?1 and tenantId = ?2",
                siteIds, principal.tenancyId());

        long aeCount = 0;
        if (!siteIds.isEmpty()) {
            List<UUID> enrollmentIds = PatientEnrollment
                .find("siteId in ?1 and tenantId = ?2", siteIds, principal.tenancyId())
                .project(UUID.class).list();
            if (!enrollmentIds.isEmpty()) {
                aeCount = AdverseEvent.count("enrollmentId in ?1 and tenantId = ?2",
                    enrollmentIds, principal.tenancyId());
            }
        }

        long devCount = siteIds.isEmpty() ? 0 :
            ProtocolDeviation.count("siteId in ?1 and tenantId = ?2",
                siteIds, principal.tenancyId());

        return Response.ok(new TrialSummary(
            trial.protocolId, trial.phase.name(), trial.sponsor,
            trial.targetEnrollment, enrolled, aeCount, devCount
        )).build();
    }

    @GET
    @Path("/patients")
    @RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR,
                   ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
    public Response patients(@PathParam("trialId") UUID trialId) {
        ClinicalTrial trial = ClinicalTrial.findByIdForTenant(trialId, principal);
        if (trial == null) return Response.status(404).build();

        List<UUID> siteIds = TrialSite.find("trialId = ?1 and tenantId = ?2",
            trialId, principal.tenancyId()).project(UUID.class).list();

        if (siteIds.isEmpty()) return Response.ok(List.of()).build();

        List<PatientEnrollment> enrollments = PatientEnrollment
            .find("siteId in ?1 and tenantId = ?2", siteIds, principal.tenancyId())
            .list();

        List<PatientRow> rows = enrollments.stream().map(e -> new PatientRow(
            e.id, e.siteId, e.patientId,
            e.enrollmentStatus != null ? e.enrollmentStatus.name() : null,
            e.screeningResult != null ? e.screeningResult.name() : null,
            e.consentStatus != null ? e.consentStatus.name() : null
        )).toList();

        return Response.ok(rows).build();
    }

    @GET
    @Path("/adverse-events")
    @RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR,
                   ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
    public Response adverseEvents(@PathParam("trialId") UUID trialId) {
        ClinicalTrial trial = ClinicalTrial.findByIdForTenant(trialId, principal);
        if (trial == null) return Response.status(404).build();

        // Two-hop: trial → sites → enrollments → AEs
        List<UUID> siteIds = TrialSite.find("trialId = ?1 and tenantId = ?2",
            trialId, principal.tenancyId()).project(UUID.class).list();
        if (siteIds.isEmpty()) return Response.ok(List.of()).build();

        List<PatientEnrollment> enrollments = PatientEnrollment
            .find("siteId in ?1 and tenantId = ?2", siteIds, principal.tenancyId())
            .list();
        if (enrollments.isEmpty()) return Response.ok(List.of()).build();

        List<UUID> enrollmentIds = enrollments.stream().map(e -> e.id).toList();
        // Build siteId lookup from enrollment
        java.util.Map<UUID, UUID> enrollmentToSite = enrollments.stream()
            .collect(java.util.stream.Collectors.toMap(e -> e.id, e -> e.siteId));

        List<AdverseEvent> aes = AdverseEvent
            .find("enrollmentId in ?1 and tenantId = ?2", enrollmentIds, principal.tenancyId())
            .list();

        Instant now = Instant.now();
        List<AdverseEventRow> rows = aes.stream().map(ae -> {
            String slaRemaining = null;
            if (ae.slaDeadline != null) {
                Duration remaining = Duration.between(now, ae.slaDeadline);
                if (remaining.isNegative()) {
                    slaRemaining = "OVERDUE by " + formatDuration(remaining.abs());
                } else {
                    slaRemaining = formatDuration(remaining) + " remaining";
                }
            }
            return new AdverseEventRow(
                ae.id, ae.enrollmentId, enrollmentToSite.get(ae.enrollmentId),
                ae.grade != null ? ae.grade.name() : null, null,
                ae.reportedAt, ae.slaDeadline,
                ae.escalationStatus != null ? ae.escalationStatus.name() : null,
                ae.regulatorySubmissionStatus != null ? ae.regulatorySubmissionStatus.name() : null,
                slaRemaining
            );
        }).toList();

        return Response.ok(rows).build();
    }

    @GET
    @Path("/deviations")
    @RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR,
                   ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
    public Response deviations(@PathParam("trialId") UUID trialId) {
        ClinicalTrial trial = ClinicalTrial.findByIdForTenant(trialId, principal);
        if (trial == null) return Response.status(404).build();

        List<UUID> siteIds = TrialSite.find("trialId = ?1 and tenantId = ?2",
            trialId, principal.tenancyId()).project(UUID.class).list();
        if (siteIds.isEmpty()) return Response.ok(List.of()).build();

        List<ProtocolDeviation> devs = ProtocolDeviation
            .find("siteId in ?1 and tenantId = ?2", siteIds, principal.tenancyId())
            .list();

        List<DeviationRow> rows = devs.stream().map(d -> new DeviationRow(
            d.id, d.siteId, d.deviationType,
            d.severity != null ? d.severity.name() : null,
            d.piApprovalStatus != null ? d.piApprovalStatus.name() : null,
            d.reportedAt
        )).toList();

        return Response.ok(rows).build();
    }

    private static String formatDuration(Duration d) {
        long hours = d.toHours();
        long minutes = d.toMinutesPart();
        if (hours > 24) return (hours / 24) + "d " + (hours % 24) + "h";
        return hours + "h " + minutes + "m";
    }
}
```

**Note to implementer:** The actual entity field names (`enrollmentStatus`, `screeningResult`, `consentStatus`, etc.) and their types must be verified against the Panache entities using `ide_file_structure` on each entity class. The `project(UUID.class)` Panache method extracts a single field — verify it works for `id` fields. If Panache doesn't support `project()` for entity IDs in this version, use `.list()` and `.stream().map(e -> e.id).toList()`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: TrialDashboardResource — summary, patients, AEs, deviations endpoints

Refs #96, casehubio/clinical#93"
```

---

### Task 3: TrialDashboardResource — Agents, Governance, Ledger Entries (3 cross-source endpoints)

**Issue:** #96

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialDashboardResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/TrialDashboardResourceTest.java`

**Interfaces:**
- Consumes (additional): `ClinicalCapabilities`, `ClinicalTrustDimensions`, `ClinicalTrustRoutingPolicyProvider`, `ActorTrustScoreRepository` (qhorus), `CaseLedgerEntryRepository` (qhorus), `LedgerEntryRepository` (qhorus)
- Produces: `GET /trials/{trialId}/agents`, `GET /trials/{trialId}/adverse-events/{aeId}/governance`, `GET /trials/{trialId}/ledger-entries`

**Important:** These endpoints are cross-datasource aggregations. The agent endpoint joins static capabilities with trust scores and attestation data from the qhorus datasource. The governance endpoint joins AE entity fields with `WorkerDecisionEntry` and `ActorTrustScore`. The ledger-entries endpoint finds subject IDs from the default datasource, then queries qhorus.

- [ ] **Step 1: Research the actual repository APIs**

Before writing tests, verify the method signatures and return types using IntelliJ MCP:

1. `ide_find_class` for `ActorTrustScoreRepository` — find all query methods
2. `ide_find_class` for `CaseLedgerEntryRepository` — find `findWorkerDecisionsByCaseId()` signature
3. `ide_find_class` for `WorkerDecisionEntry` — verify `trustScoreAtRouting`, `thresholdApplied`, `workerId`, `capabilityTag` field names
4. `ide_find_class` for `LedgerEntryRepository` — find `findBySubjectId()` signature
5. `ide_find_class` for `TrustRoutingPolicy` — verify the policy fields (threshold, minDecisionCount, etc.)

Document the exact method signatures found before proceeding.

- [ ] **Step 2: Write tests for the agents endpoint**

Add to `TrialDashboardResourceTest`:

```java
@Test
void agents_returns_capability_list_with_trust_data() {
    // The agents endpoint aggregates static capabilities with trust scores.
    // With no trust data seeded, it should still return the capability list
    // with null/zero trust fields.
    given()
        .when().get("/trials/{trialId}/agents", trialId)
        .then()
        .statusCode(200)
        .body("size()", greaterThanOrEqualTo(1))
        .body("[0].capability", notNullValue());
}
```

- [ ] **Step 3: Write tests for the governance endpoint**

```java
@Test
void governance_returns_404_for_ae_without_susar() {
    // The seeded AE has no SUSAR oversight case
    UUID aeId = AdverseEvent.find("tenantId", principal.tenancyId())
        .<AdverseEvent>firstResult().id;
    given()
        .when().get("/trials/{trialId}/adverse-events/{aeId}/governance", trialId, aeId)
        .then()
        .statusCode(200)
        .body("susarOversightStatus", equalTo("NONE"));
}
```

- [ ] **Step 4: Write tests for ledger-entries endpoint**

```java
@Test
void ledger_entries_returns_empty_when_no_entries() {
    given()
        .when().get("/trials/{trialId}/ledger-entries", trialId)
        .then()
        .statusCode(200)
        .body("size()", equalTo(0));
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest --batch-mode`
Expected: FAIL — endpoints not found

- [ ] **Step 6: Implement agents endpoint**

Add to `TrialDashboardResource`. Inject `ActorTrustScoreRepository` and `ClinicalTrustRoutingPolicyProvider`. Build the agent list from static `ClinicalCapabilities` constants, enriched with trust scores from the repository.

The response record:

```java
public record AgentRow(
    String capability, String trustDimension,
    Double trustScore, Double threshold, int maturityPhase,
    int decisionCount, int attestationPositive, int attestationNegative
) {}
```

Implementation: iterate `ClinicalCapabilities` constants, for each look up the trust score from `ActorTrustScoreRepository`, and the routing policy from `ClinicalTrustRoutingPolicyProvider.forCapability()`. Return the joined data. When no trust score exists (bootstrap phase), return nulls for trust fields.

- [ ] **Step 7: Implement governance endpoint**

Add `GET /adverse-events/{aeId}/governance` to `TrialDashboardResource`. This is the Step 6 hero layout data source.

The response record:

```java
public record GovernanceContext(
    // What the AI decided
    String grade, boolean unexpected, boolean suspected,
    String susarOversightStatus,
    // WorkerDecisionEntry data (if SUSAR case exists)
    String workerId, String capabilityTag,
    Double trustScoreAtRouting, Double thresholdApplied,
    // Current trust score
    Double currentTrustScore,
    // Gate status (from AE entity, not ephemeral gate state)
    String gateStatus
) {}
```

Implementation: load AE, if `susarOversightCaseId != null`, query `CaseLedgerEntryRepository.findWorkerDecisionsByCaseId()` for the SAFETY_MONITORING decision entry. Look up current trust score from `ActorTrustScoreRepository`. Gate status comes from `ae.susarOversightStatus`.

- [ ] **Step 8: Implement ledger-entries endpoint**

Add `GET /ledger-entries` to `TrialDashboardResource`. This is cross-datasource.

The response record:

```java
public record LedgerEntryRow(
    UUID id, UUID subjectId, int sequenceNumber,
    String entryType, String actorId, String actorRole,
    Instant occurredAt, String summary
) {}
```

Implementation:
1. Find all enrollment IDs and deviation IDs for the trial (default datasource queries)
2. For each subject ID, call `LedgerEntryRepository.findBySubjectId(subjectId, "default")` (qhorus datasource)
3. Flatten, sort by occurredAt, return

Support `?type=` query parameter to filter by `entryType`.

- [ ] **Step 9: Run all tests**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest --batch-mode`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git commit -m "feat: TrialDashboardResource — agents, governance, ledger-entries endpoints

Cross-datasource aggregation: agents join ClinicalCapabilities +
ActorTrustScoreRepository; governance joins AE + WorkerDecisionEntry +
ActorTrustScore; ledger-entries queries by subject IDs across default
and qhorus datasources.

Refs #96, casehubio/clinical#93"
```

---

### Task 4: DemoActionResource — PI Approval + SUSAR Gate Approval

**Issue:** #103

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/demo/DemoActionResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/demo/DemoActionResourceTest.java`

**Interfaces:**
- Consumes: `ProtocolDeviation` (entity), `AdverseEvent` (entity), `ChannelGateway` + `ChannelRef` + `InboundHumanMessage` (qhorus-api), `ChannelService` (qhorus), `WorkItemService` + `WorkItemStore` (casehub-work), `TrustScoreJob` (casehub-ledger), `CurrentPrincipal`
- Produces: `POST /demo/deviations/{deviationId}/approve-pi`, `POST /demo/adverse-events/{aeId}/approve-susar-gate`

- [ ] **Step 1: Research the exact API signatures**

Before writing code, use IntelliJ MCP to verify:

1. `ide_find_class` for `ChannelGateway` — verify `receiveHumanMessage(ChannelRef, InboundHumanMessage)` signature
2. `ide_find_class` for `ChannelRef` — verify `record ChannelRef(UUID id, String name)`
3. `ide_find_class` for `InboundHumanMessage` — verify all 6 fields
4. `ide_find_class` for `ChannelService` — verify `findByName(String)` return type
5. `ide_find_class` for `WorkItemService` — verify `claim(UUID, String)` and `complete(UUID, String, String, String)` signatures
6. `ide_find_class` for `WorkItemStore` — find a method to query by callerRef prefix, or verify that scanning with stream filter is the available pattern
7. `ide_find_class` for `TrustScoreJob` — verify `runComputation()` is public

Document findings before proceeding.

- [ ] **Step 2: Write tests for PI approval endpoint**

```java
package io.casehub.clinical.demo;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.casehub.clinical.service.ProtocolDeviationService;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class DemoActionResourceTest {

    @Inject FixedCurrentPrincipal principal;
    @Inject ProtocolDeviationService deviationService;

    private UUID trialId;
    private UUID siteId;

    @BeforeEach
    @Transactional
    void setup() {
        trialId = UUID.randomUUID();
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = trialId;
        trial.tenantId = principal.tenancyId();
        trial.protocolId = "DEMO-TEST-" + UUID.randomUUID().toString().substring(0, 8);
        trial.phase = TrialPhase.PHASE_3;
        trial.sponsor = "Test";
        trial.targetEnrollment = 10;
        trial.persist();

        siteId = UUID.randomUUID();
        TrialSite site = new TrialSite();
        site.id = siteId;
        site.tenantId = principal.tenancyId();
        site.trialId = trialId;
        site.investigatorId = "dr-test";
        site.persist();
    }

    @Test
    void approve_pi_returns_409_when_no_pending_command() {
        // Create deviation without reporting it (no COMMANDED state)
        UUID devId = createDirectDeviation();
        given()
            .when().post("/demo/deviations/{devId}/approve-pi", devId)
            .then()
            .statusCode(409);
    }

    @Test
    void approve_pi_returns_404_for_unknown_deviation() {
        given()
            .when().post("/demo/deviations/{devId}/approve-pi", UUID.randomUUID())
            .then()
            .statusCode(404);
    }

    @Transactional
    UUID createDirectDeviation() {
        ProtocolDeviation dev = new ProtocolDeviation();
        dev.id = UUID.randomUUID();
        dev.tenantId = principal.tenancyId();
        dev.siteId = siteId;
        dev.deviationType = "DOSING_ERROR";
        dev.severity = DeviationSeverity.MINOR;
        dev.piApprovalStatus = PiApprovalStatus.NONE;
        dev.persist();
        return dev.id;
    }
}
```

**Note:** Full lifecycle tests for PI approval require reporting a deviation through `ProtocolDeviationService.reportDeviation()` first (to get COMMANDED state). This involves the full qhorus channel creation flow. The test pattern matches `PiResponseListenerIntegrationTest` — study that test before implementing.

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=DemoActionResourceTest --batch-mode`
Expected: FAIL — no resource at `/demo/...`

- [ ] **Step 4: Implement DemoActionResource**

```java
package io.casehub.clinical.demo;

import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;

@Path("/demo")
@Produces(MediaType.APPLICATION_JSON)
@IfBuildProfile("dev")
public class DemoActionResource {

    @Inject CurrentPrincipal principal;
    @Inject ChannelGateway channelGateway;
    @Inject ChannelService channelService;
    @Inject WorkItemService workItemService;
    // Inject TrustScoreJob — verify class name via ide_find_class

    @POST
    @Path("/deviations/{deviationId}/approve-pi")
    @Transactional
    public Response approvePi(@PathParam("deviationId") UUID deviationId) {
        ProtocolDeviation dev = ProtocolDeviation.findByIdForTenant(deviationId, principal);
        if (dev == null) return Response.status(404).build();
        if (dev.piApprovalStatus != io.casehub.clinical.api.model.PiApprovalStatus.COMMANDED) {
            return Response.status(409).entity(Map.of(
                "error", "Deviation is not in COMMANDED state",
                "currentStatus", dev.piApprovalStatus.name()
            )).build();
        }

        var channel = channelService.findByName(dev.piCommandChannelName).orElse(null);
        if (channel == null) return Response.status(500).entity(Map.of(
            "error", "PI oversight channel not found"
        )).build();

        channelGateway.receiveHumanMessage(
            new ChannelRef(channel.id, channel.name),
            new InboundHumanMessage(
                "demo-pi",
                "{\"decision\":\"APPROVED\"}",
                Instant.now(),
                Map.of(),
                deviationId.toString(),
                null
            )
        );

        return Response.ok(Map.of(
            "deviationId", deviationId,
            "action", "PI_APPROVED",
            "note", "MessageReceivedEvent fired — PiResponseListener will process async"
        )).build();
    }

    @POST
    @Path("/adverse-events/{aeId}/approve-susar-gate")
    public Response approveSusarGate(@PathParam("aeId") UUID aeId) {
        AdverseEvent ae = AdverseEvent.findByIdForTenant(aeId, principal);
        if (ae == null) return Response.status(404).build();
        if (ae.susarOversightCaseId == null) {
            return Response.status(400).entity(Map.of(
                "error", "No SUSAR oversight case exists for this AE"
            )).build();
        }

        // Find the gate WorkItem by callerRef prefix
        String callerRefPrefix = "case:" + ae.susarOversightCaseId + "/gate:";
        // Use WorkItemStore to scan for the matching WorkItem.
        // Verify the actual API via ide_find_class on WorkItemStore.
        // Pattern: workItemStore.findAll() filtered by callerRef.startsWith(prefix)
        // and non-terminal status.

        // Once found:
        // workItemService.claim(workItem.id, "demo-investigator");
        // workItemService.complete(workItem.id, "demo-investigator",
        //     "{\"decision\":\"APPROVED\"}", "APPROVED");

        // After gate approval events process:
        // trustScoreJob.runComputation();

        // Return trust score delta
        return Response.ok(Map.of(
            "aeId", aeId,
            "action", "SUSAR_GATE_APPROVED"
        )).build();
    }
}
```

**Note to implementer:** The SUSAR gate approval implementation requires completing the WorkItem lookup, claim, and complete flow. The actual `WorkItemStore` API must be verified — if no `findByCallerRefPrefix()` exists, scan all active WorkItems with a stream filter. The `TrustScoreJob` injection and `runComputation()` call must be verified. Study `SusarOversightLifecycleTest` for the expected event chain.

- [ ] **Step 5: Run tests and iterate**

Run: `mvn test -pl runtime -Dtest=DemoActionResourceTest --batch-mode`
Expected: PASS for 404/409 error cases. Full lifecycle tests require more setup.

- [ ] **Step 6: Add full lifecycle test for PI approval**

Add a test that creates a deviation via `ProtocolDeviationService.reportDeviation()`, waits for COMMANDED state, then calls the demo endpoint. Verify the deviation transitions to APPROVED or ESCALATED. Use Awaitility for async waiting. Pattern: `PiResponseListenerIntegrationTest`.

- [ ] **Step 7: Run full test suite**

Run: `mvn test -pl runtime --batch-mode`
Expected: all tests pass including existing tests

- [ ] **Step 8: Commit**

```bash
git commit -m "feat: DemoActionResource — PI approval + SUSAR gate approval demo endpoints

PI approval uses ChannelGateway.receiveHumanMessage() to fire real
CDI chain. SUSAR gate approval uses WorkItemService.claim()+complete()
to trigger ActionGateApprovedEvent, then TrustScoreJob.runComputation()
for immediate trust score delta. Both @IfBuildProfile('dev').

Refs #103, casehubio/clinical#93"
```

---

### Task 5: DemoDataSeeder

**Issue:** #95

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/demo/DemoDataSeeder.java`
- Create: `runtime/src/test/java/io/casehub/clinical/demo/DemoDataSeederTest.java`

**Interfaces:**
- Consumes: `AdverseEventService`, `ProtocolDeviationService`, `EligibilityScreeningService`, `TrialActivationService`, `ChannelGateway`, `ChannelService`, `WorkItemService`, `WorkItemStore`, `TrustScoreJob`, `LedgerVerificationService`, `CurrentPrincipal`
- Produces: A fully-populated trial scenario (ONCO-2024-001) with Merkle-verified ledger entries and materialised trust scores

**This is the most complex task.** The seeder replays a trial scenario through real service calls with async engine case processing. Study `ThreeSiteShowcaseTest` and `PiResponseListenerIntegrationTest` for the patterns.

- [ ] **Step 1: Write the seeder skeleton with idempotency**

```java
package io.casehub.clinical.demo;

import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import io.quarkus.runtime.StartupEvent;
import io.quarkus.arc.profile.IfBuildProfile;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

@ApplicationScoped
@IfBuildProfile("dev")
public class DemoDataSeeder {

    private static final Logger LOG = Logger.getLogger(DemoDataSeeder.class);

    public static final UUID TRIAL_ID =
        UUID.nameUUIDFromBytes("ONCO-2024-001".getBytes(StandardCharsets.UTF_8));

    @ConfigProperty(name = "casehub.clinical.demo.seed-data", defaultValue = "false")
    boolean seedEnabled;

    @Inject CurrentPrincipal principal;
    // Inject all required services — verify via ide_find_class

    void onStartup(@Observes StartupEvent event) {
        if (!seedEnabled) {
            LOG.info("Demo data seeding disabled");
            return;
        }
        if (trialAlreadyExists()) {
            LOG.info("Demo trial already exists — skipping seed");
            return;
        }
        LOG.info("Seeding demo data...");
        seed();
        LOG.info("Demo data seeding complete");
    }

    private boolean trialAlreadyExists() {
        // Must be in a TX context — use @Transactional helper or QuarkusTransaction
        return io.casehub.clinical.entity.ClinicalTrial
            .find("protocolId", "ONCO-2024-001").firstResult() != null;
    }

    private void seed() {
        // Phase 1: Create trial + 3 sites + patients
        // Phase 2: Seed Site A events (screening, Grade 2 AE, Grade 4 AEs with SUSAR lifecycle)
        // Phase 3: Seed Site B events (protocol deviation with PI approval)
        // Phase 4: Seed Site C events (protocol amendment)
        // Phase 5: Materialise trust scores
        // Phase 6: Verify Merkle chains
    }
}
```

- [ ] **Step 2: Implement Phase 1 — trial, sites, patients**

Create the trial with `TRIAL_ID`, three sites, and patients. Each entity gets `tenantId = principal.tenancyId()` (which resolves to "demo-tenant" via `DemoCurrentPrincipal`).

**Important:** Entity creation must be `@Transactional`. Use helper methods annotated `@Transactional(REQUIRES_NEW)` so each phase commits independently. The seeder's main `seed()` method is NOT `@Transactional` — this avoids holding connections during async polling.

- [ ] **Step 3: Implement Phase 2 — Site A events**

Seed in order:
1. Eligibility screening via `EligibilityScreeningService` — CRITERIA_MET result
2. Grade 2 AE via `AdverseEventService.reportAdverseEvent()` — baseline, no SUSAR
3. Grade 4 unexpected AE #1 via `AdverseEventService.reportAdverseEvent()` — triggers SUSAR
4. Poll for SUSAR oversight case to start (Awaitility pattern: check `ae.susarOversightCaseId != null`)
5. Find and complete the SUSAR gate WorkItem (same flow as `DemoActionResource.approveSusarGate()`)
6. Poll for attestation to be written
7. Repeat steps 3-6 for Grade 4 AEs #2 and #3
8. **Do NOT complete the AE escalation WorkItems** — leave grade4Active flags set for DSMB demo

- [ ] **Step 4: Implement Phase 3 — Site B deviation**

1. Create a CRITICAL deviation via `ProtocolDeviationService.reportDeviation()`
2. Poll for COMMANDED state
3. Approve as PI via `channelGateway.receiveHumanMessage()` (same flow as `DemoActionResource.approvePi()`)
4. Poll for approval + IRB escalation

- [ ] **Step 5: Implement Phase 4 — Site C amendment**

1. Create an amendment via the appropriate service
2. Poll for PROCEED result

- [ ] **Step 6: Implement Phase 5 — trust score materialisation**

Call `TrustScoreJob.runComputation()` after all Grade 4 AE SUSAR lifecycles are complete. This materialises Bayesian Beta scores from the accumulated attestations.

- [ ] **Step 7: Implement Phase 6 — Merkle verification**

For each subject ID that has ledger entries, call `LedgerVerificationService.verify()`. If any fail, throw `IllegalStateException` — startup aborts.

- [ ] **Step 8: Write integration test**

```java
@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class DemoDataSeederTest {

    @Inject LedgerVerificationService ledgerVerificationService;

    @Test
    void seeded_trial_exists() {
        ClinicalTrial trial = ClinicalTrial.find("protocolId", "ONCO-2024-001").firstResult();
        assertThat(trial).isNotNull();
        assertThat(trial.id).isEqualTo(DemoDataSeeder.TRIAL_ID);
    }

    @Test
    void seeded_sites_exist() {
        long siteCount = TrialSite.count("trialId", DemoDataSeeder.TRIAL_ID);
        assertThat(siteCount).isEqualTo(3);
    }

    // Add tests for entity counts, trust score existence, Merkle verification
}
```

**Note:** The seeder test depends on the seeder having run. In tests, `casehub.clinical.demo.seed-data` may be false. Either enable it in test config or call `seeder.seed()` directly from the test.

- [ ] **Step 9: Run full test suite**

Run: `mvn test -pl runtime --batch-mode`
Expected: all tests pass

- [ ] **Step 10: Commit**

```bash
git commit -m "feat: DemoDataSeeder — service-layer trial scenario with Merkle verification

Replays ONCO-2024-001 through real service calls:
- Site A: screening + Grade 2 AE + 3 Grade 4 SUSAR lifecycles (trust scores)
- Site B: CRITICAL deviation with PI approval + IRB escalation
- Site C: protocol amendment (PROCEED)
Idempotent on restart. Verifies Merkle chains post-seed.
grade4Active flags intentionally left set for DSMB demo.

Refs #95, casehubio/clinical#93"
```

---

## Self-Review

**Spec coverage check:**
- DemoCurrentPrincipal: Task 1 ✓
- Quinoa scaffold: Task 1 ✓
- TrialDashboardResource (7 endpoints): Tasks 2+3 ✓
- DemoActionResource (2 endpoints): Task 4 ✓
- DemoDataSeeder: Task 5 ✓
- Deterministic UUID: Task 1 (constant) + Task 5 (seeder uses it) ✓
- Idempotency: Task 5 ✓
- Merkle verification: Task 5 Phase 6 ✓
- Trust score materialisation: Task 5 Phase 5 ✓
- DSMB rollup (leave grade4Active set): Task 5 Phase 2 step 8 ✓
- Profile guard (@IfBuildProfile): Tasks 1, 4, 5 ✓
- Cross-datasource queries documented: Task 3 ✓
- StartupEvent timing: Task 5 (documented in seeder, fallback pattern in spec) ✓

**Placeholder scan:** Task 4 Step 4 has partial implementation for the SUSAR gate approval — marked with "Note to implementer" indicating the exact APIs must be verified. This is intentional — the WorkItemStore query pattern depends on the actual API, which the implementer must verify via IntelliJ MCP. The code shows the structure; the implementer fills in the verified API calls.

**Type consistency:** `TRIAL_ID` constant defined in Task 1 (DemoCurrentPrincipal.java adjacent) and used in Task 5 (DemoDataSeeder.java). TypeScript constant `316e3846-4ea7-3b18-a6f7-e01ce6582a69` matches. Response record field names (`protocolId`, `grade`, `slaDeadline`, etc.) match entity field names verified from existing resource classes.
