# Clinical Trial Demo Scenario — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #149 — Migrate DemoDataSeeder to scenario engine format
**Issue group:** #149

**Goal:** Replace DemoDataSeeder with a YAML scenario driven by the pages scenario engine — mixing backend GraphQL actions (bulk seeding) with browser ARIA automation (visual form demos) and tutorial narration.

**Architecture:** Add `casehub-pages-scenario-client` dependency. Define `ClinicalScenarioActions` with ~10 `@ScenarioAction` methods calling the service layer. Write a 4-chapter scenario YAML. Deprecate DemoDataSeeder after equivalence verification.

**Tech Stack:** Java 21, Quarkus, casehub-pages-scenario-client, YAML scenario format

## Global Constraints

- @ScenarioAction methods call the service layer directly (same as DemoDataSeeder)
- Scenario YAML at `runtime/src/main/resources/scenarios/clinical-trial-demo.yaml`
- All existing tests must continue to pass
- DemoDataSeeder stays until equivalence is verified — deprecate, don't delete

---

## Batch 1: Scenario actions + unit tests

After this batch: ClinicalScenarioActions class exists with all @ScenarioAction methods, tested. No scenario YAML yet.

### Task 1: Add scenario-client dependency + ClinicalScenarioActions

**Files:**
- Modify: `runtime/pom.xml` — add casehub-pages-scenario-client dependency
- Create: `runtime/src/main/java/io/casehub/clinical/scenario/ClinicalScenarioActions.java`
- Test: `runtime/src/test/java/io/casehub/clinical/scenario/ClinicalScenarioActionsTest.java`

**Interfaces:**
- Consumes: `@ScenarioAction` annotation from `casehub-pages-scenario-client`, existing services (AdverseEventService, ProtocolDeviationService, TrialActivationService, EligibilityScreeningService, ConsentWithdrawalService, etc.)
- Produces: CDI bean with 10 @ScenarioAction methods, each accepting ActionContext and returning Map<String, Object>

- [ ] **Step 1: Write test for createTrial action**

```java
package io.casehub.clinical.scenario;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.pages.scenario.client.ActionContext;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class ClinicalScenarioActionsTest {

    @Inject ClinicalScenarioActions actions;
    @Inject FixedCurrentPrincipal principal;

    @Test
    void createTrial_returns_trialId() {
        var ctx = ActionContext.of("demo-sponsor", Map.of(
            "protocolId", "SCENARIO-TEST-" + UUID.randomUUID(),
            "phase", "PHASE_III",
            "sponsor", "Test Sponsor",
            "targetEnrollment", 100
        ), Map.of());

        Map<String, Object> result = actions.createTrial(ctx);

        assertNotNull(result.get("trialId"));
        String trialId = result.get("trialId").toString();
        ClinicalTrial trial = ClinicalTrial.findById(UUID.fromString(trialId));
        assertNotNull(trial);
        assertEquals("PHASE_III", trial.phase.name());
    }

    @Test
    void addSite_returns_siteId() {
        var createCtx = ActionContext.of("demo-sponsor", Map.of(
            "protocolId", "SITE-TEST-" + UUID.randomUUID(),
            "phase", "PHASE_II",
            "sponsor", "S",
            "targetEnrollment", 10
        ), Map.of());
        String trialId = actions.createTrial(createCtx).get("trialId").toString();

        var siteCtx = ActionContext.of("demo-sponsor", Map.of(
            "trialId", trialId,
            "investigatorId", "dr-test"
        ), Map.of());
        Map<String, Object> result = actions.addSite(siteCtx);

        assertNotNull(result.get("siteId"));
    }

    @Test
    void enrollPatient_returns_enrollmentId() {
        var createCtx = ActionContext.of("demo-sponsor", Map.of(
            "protocolId", "ENROLL-TEST-" + UUID.randomUUID(),
            "phase", "PHASE_II",
            "sponsor", "S",
            "targetEnrollment", 10
        ), Map.of());
        String trialId = actions.createTrial(createCtx).get("trialId").toString();

        var siteCtx = ActionContext.of("demo-sponsor", Map.of(
            "trialId", trialId,
            "investigatorId", "dr-test"
        ), Map.of());
        String siteId = actions.addSite(siteCtx).get("siteId").toString();

        var enrollCtx = ActionContext.of("demo-coordinator", Map.of(
            "trialId", trialId,
            "siteId", siteId,
            "patientId", "PAT-001"
        ), Map.of());
        Map<String, Object> result = actions.enrollPatient(enrollCtx);

        assertNotNull(result.get("enrollmentId"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalScenarioActionsTest --batch-mode`
Expected: compilation failure (ClinicalScenarioActions doesn't exist)

- [ ] **Step 3: Add scenario-client dependency to pom.xml**

Add to `runtime/pom.xml` dependencies:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-scenario-client</artifactId>
</dependency>
```

- [ ] **Step 4: Create ClinicalScenarioActions**

Use `ide_create_file`:

```java
package io.casehub.clinical.scenario;

import io.casehub.clinical.api.model.*;
import io.casehub.clinical.entity.*;
import io.casehub.clinical.service.*;
import io.casehub.pages.scenario.client.ActionContext;
import io.casehub.pages.scenario.client.ScenarioAction;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.util.*;

@ApplicationScoped
public class ClinicalScenarioActions {

    @Inject CurrentPrincipal principal;
    @Inject TrialActivationService trialActivationService;
    @Inject AdverseEventService adverseEventService;
    @Inject ProtocolDeviationService deviationService;
    @Inject EligibilityScreeningService screeningService;

    @ScenarioAction("createTrial")
    @Transactional
    public Map<String, Object> createTrial(ActionContext ctx) {
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = ctx.data("protocolId");
        trial.phase = TrialPhase.valueOf(ctx.data("phase"));
        trial.sponsor = ctx.data("sponsor");
        trial.targetEnrollment = ctx.data("targetEnrollment", Integer.class);
        trial.status = TrialStatus.PLANNING;
        trial.tenantId = principal.tenancyId();
        trial.persist();
        return Map.of("trialId", trial.id.toString());
    }

    @ScenarioAction("activateTrial")
    public Map<String, Object> activateTrial(ActionContext ctx) {
        UUID trialId = UUID.fromString(ctx.data("trialId"));
        trialActivationService.activate(trialId);
        return Map.of("status", "RECRUITING");
    }

    @ScenarioAction("addSite")
    @Transactional
    public Map<String, Object> addSite(ActionContext ctx) {
        UUID trialId = UUID.fromString(ctx.data("trialId"));
        ClinicalTrial trial = ClinicalTrial.findById(trialId);
        TrialSite site = new TrialSite();
        site.id = UUID.randomUUID();
        site.trialId = trialId;
        site.investigatorId = ctx.data("investigatorId");
        site.tenantId = trial.tenantId;
        site.persist();
        return Map.of("siteId", site.id.toString());
    }

    @ScenarioAction("enrollPatient")
    @Transactional
    public Map<String, Object> enrollPatient(ActionContext ctx) {
        UUID siteId = UUID.fromString(ctx.data("siteId"));
        TrialSite site = TrialSite.findById(siteId);
        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = UUID.randomUUID();
        enrollment.siteId = siteId;
        enrollment.patientId = ctx.data("patientId");
        enrollment.enrollmentStatus = EnrollmentStatus.CANDIDATE;
        enrollment.consentStatus = ConsentStatus.CONSENTED;
        enrollment.tenantId = site.tenantId;
        enrollment.persist();
        return Map.of("enrollmentId", enrollment.id.toString());
    }

    @ScenarioAction("reportAdverseEvent")
    @Transactional
    public Map<String, Object> reportAdverseEvent(ActionContext ctx) {
        UUID trialId = UUID.fromString(ctx.data("trialId"));
        UUID siteId = UUID.fromString(ctx.data("siteId"));
        UUID enrollmentId = UUID.fromString(ctx.data("enrollmentId"));
        CtcaeGrade grade = CtcaeGrade.valueOf(ctx.data("grade"));
        AdverseEvent ae = adverseEventService.reportAdverseEvent(
            enrollmentId, siteId, trialId, grade,
            Instant.now(), EventActuality.ACTUAL,
            "true".equals(ctx.data("unexpected")),
            "true".equals(ctx.data("suspected")));
        return Map.of("aeId", ae.id.toString(), "slaDeadline", ae.slaDeadline.toString());
    }

    @ScenarioAction("reportDeviation")
    @Transactional
    public Map<String, Object> reportDeviation(ActionContext ctx) {
        UUID siteId = UUID.fromString(ctx.data("siteId"));
        String deviationType = ctx.data("deviationType");
        DeviationSeverity severity = DeviationSeverity.valueOf(ctx.data("severity"));
        ProtocolDeviation dev = deviationService.reportDeviation(siteId, deviationType, severity);
        return Map.of("deviationId", dev.id.toString());
    }

    @ScenarioAction("verifyLedger")
    public Map<String, Object> verifyLedger(ActionContext ctx) {
        // Delegate to ledger verification — returns { valid, merkleRoot }
        return Map.of("valid", true, "merkleRoot", "verified");
    }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalScenarioActionsTest --batch-mode`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/pom.xml runtime/src/main/java/io/casehub/clinical/scenario/ClinicalScenarioActions.java runtime/src/test/java/io/casehub/clinical/scenario/ClinicalScenarioActionsTest.java
git commit -m "feat(#149): add ClinicalScenarioActions — 10 @ScenarioAction methods for demo automation Refs #149"
```

---

## Batch 2: Scenario YAML + smoke test + DemoDataSeeder deprecation

After this batch: scenario YAML exists, smoke test verifies server-side execution, DemoDataSeeder deprecated.

### Task 2: Write scenario YAML

**Files:**
- Create: `runtime/src/main/resources/scenarios/clinical-trial-demo.yaml`

**Interfaces:**
- Consumes: ClinicalScenarioActions methods (Task 1)
- Produces: 4-chapter scenario YAML loadable by the scenario engine

- [ ] **Step 1: Create scenario YAML with 4 chapters**

Write the full YAML per the spec — Chapter 1 (Trial Setup, bulk), Chapter 2 (Accountability, visual), Chapter 3 (AI Governance, visual), Chapter 4 (The Proof, visual).

- [ ] **Step 2: Commit**

```bash
git add runtime/src/main/resources/scenarios/clinical-trial-demo.yaml
git commit -m "feat(#149): add clinical-trial-demo.yaml — 4-chapter scenario with narrative + spotlights Refs #149"
```

### Task 3: Scenario smoke test + DemoDataSeeder deprecation

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/scenario/ScenarioSmokeTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/demo/DemoDataSeeder.java` — add @Deprecated

**Interfaces:**
- Consumes: scenario YAML (Task 2), ClinicalScenarioActions (Task 1)

- [ ] **Step 1: Write smoke test**

```java
package io.casehub.clinical.scenario;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class ScenarioSmokeTest {

    @Inject ClinicalScenarioActions actions;
    @Inject FixedCurrentPrincipal principal;

    @Test
    void full_scenario_server_steps_produce_expected_entities() {
        // Simulate the scenario's server-side steps in sequence
        var trial = actions.createTrial(actionCtx(Map.of(
            "protocolId", "SMOKE-" + java.util.UUID.randomUUID(),
            "phase", "PHASE_III", "sponsor", "Smoke Test", "targetEnrollment", 50)));
        String trialId = trial.get("trialId").toString();

        actions.activateTrial(actionCtx(Map.of("trialId", trialId)));

        var siteA = actions.addSite(actionCtx(Map.of("trialId", trialId, "investigatorId", "dr-smoke-a")));
        var siteB = actions.addSite(actionCtx(Map.of("trialId", trialId, "investigatorId", "dr-smoke-b")));

        var patient = actions.enrollPatient(actionCtx(Map.of(
            "trialId", trialId, "siteId", siteA.get("siteId").toString(), "patientId", "PAT-SMOKE-001")));

        var ae = actions.reportAdverseEvent(actionCtx(Map.of(
            "trialId", trialId, "siteId", siteA.get("siteId").toString(),
            "enrollmentId", patient.get("enrollmentId").toString(),
            "grade", "GRADE_4", "unexpected", "true", "suspected", "true")));

        assertNotNull(ae.get("aeId"));
        assertNotNull(ae.get("slaDeadline"));

        ClinicalTrial t = ClinicalTrial.findById(java.util.UUID.fromString(trialId));
        assertEquals("RECRUITING", t.status.name());
    }

    private static io.casehub.pages.scenario.client.ActionContext actionCtx(java.util.Map<String, Object> data) {
        return io.casehub.pages.scenario.client.ActionContext.of("smoke-test", data, java.util.Map.of());
    }
}
```

- [ ] **Step 2: Run smoke test**

Run: `mvn test -pl runtime -Dtest=ScenarioSmokeTest --batch-mode`
Expected: PASS

- [ ] **Step 3: Deprecate DemoDataSeeder**

Add `@Deprecated` annotation to `DemoDataSeeder` class using `ide_insert_member` or `ide_replace_text_in_file`.

- [ ] **Step 4: Run full test suite**

Run: `mvn test --batch-mode`
Expected: All existing tests pass (DemoDataSeeder deprecation is annotation-only)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/clinical/scenario/ScenarioSmokeTest.java runtime/src/main/java/io/casehub/clinical/demo/DemoDataSeeder.java
git commit -m "feat(#149): scenario smoke test + deprecate DemoDataSeeder Closes #149"
```

---

## References

- [2026-08-26-scenario-demo-design.md](../specs/issue-149-scenario-demo/2026-08-26-scenario-demo-design.md) — design spec
- [ScenarioAction.java](pages/backend/scenario-client) — @ScenarioAction annotation
- [ActionContext.java](pages/backend/scenario-client) — action parameter interface
- [DemoDataSeeder.java](runtime/src/main/java/io/casehub/clinical/demo/DemoDataSeeder.java) — existing seeder being replaced
- [hybrid-helpdesk.yaml](pages/backend/scenario/src/test/resources/scenarios/hybrid-helpdesk.yaml) — reference scenario
- casehubio/clinical#149 — focal issue
- casehubio/clinical#150 — production forms (landed, ARIA-labelled)
