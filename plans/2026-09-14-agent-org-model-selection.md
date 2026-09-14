# Agent Organization + Model Selection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #164 — Agent Organization + Model Selection
**Issue group:** #164, #165, #166, #167, #168, #169, #170

**Goal:** Wire eidos org model, model tier vocabulary, LLM model registry,
and decision narratives into clinical so agents have organisational
structure, tier-appropriate model selection, and visible decision trails.

**Architecture:** Four platform SPIs wired into clinical following
fsitrading's reference patterns. `ClinicalNarrativeSignalStrategy`
extends `AbstractNarrativeSignalStrategy` (casehub-blocks, already on
classpath). Org structure registered in `ClinicalTrialCaseHub.augment()`
via eidos `OrgStructure.define()` (new dependency). Model tiers declared
per capability in `ClinicalAgentSupport` with `ModelRegistry` query
resolution. UI converts to `dockWorkbench` with `registerPanel()` for
blocks-ui panels.

**Tech Stack:** Java 21 / Quarkus 3.32.2, casehub-blocks (already on
classpath), eidos org-api + org-runtime (new deps), casehub-platform-api
ModelRegistry (already on classpath), TypeScript / casehub-pages DSL /
blocks-ui components.

## Global Constraints

- casehub-blocks (`AbstractNarrativeSignalStrategy`, `DecisionNarrativePipeline`) is already on the classpath — no new Maven dependency for narratives
- eidos `org-api` and `org-runtime` are NOT on the classpath — must be added
- `ModelRegistry`, `ModelTier`, `ModelQuery`, `ModelDescriptor` are in `casehub-platform-api` — already on classpath
- `RoutingAgentProvider` is in `casehub-platform-agent-router` — already on classpath
- Model tier enum values are `FLAGSHIP`, `STANDARD`, `FAST`, `EMBEDDING` — NOT the informal names from fsitrading issues (POWERFUL, BALANCED)
- `narrative-timeline` and `org-diagram` blocks-ui packages are NOT in `.casehub-packages/` — must be published from blocks-ui repo first or deferred
- `orchestration-workbench`, `trust-workbench`, `conversation-viewer`, `routing-rationale` ARE in `.casehub-packages/`
- `@Alternative @Priority(1)` beans require addition to `quarkus.arc.selected-alternatives`
- WebSocket push uses existing `EventBroadcaster` → `TopicRegistry` → `/ws/push` infrastructure
- All REST endpoints require `@RolesAllowed` with `ClinicalGroups` constants
- Tests use `@TestSecurity(user = "test-actor", roles = {SPONSOR, INVESTIGATOR, COORDINATOR})`
- Use `"default"` as tenancyId (existing clinical pattern)

---

## Batch 1: Decision Narratives (#165)

### Task 1: ClinicalNarrativeSignalStrategy + NarrativeResource

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/narrative/ClinicalNarrativeSignalStrategy.java`
- Create: `runtime/src/main/java/io/casehub/clinical/narrative/NarrativeSignalBroadcaster.java`
- Create: `runtime/src/main/java/io/casehub/clinical/resource/NarrativeResource.java`
- Modify: `runtime/src/main/resources/application.properties` — add selected-alternatives
- Test: `runtime/src/test/java/io/casehub/clinical/narrative/ClinicalNarrativeSignalStrategyTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/resource/NarrativeResourceTest.java`

**Interfaces:**
- Consumes: `AbstractNarrativeSignalStrategy` (casehub-blocks), `DecisionNarrativePipeline` (casehub-blocks), `EventBroadcaster` (casehub-pages-push), `StepOutcomeEvent` (engine SPI)
- Produces: `ClinicalNarrativeSignalStrategy` (CDI bean), `NarrativeResource` (REST at `/api/narrative/{caseId}`), `NarrativeSignalBroadcaster` (WebSocket push)

- [ ] **Step 1: Write unit test for ClinicalNarrativeSignalStrategy**

```java
package io.casehub.clinical.narrative;

import io.casehub.blocks.summarisation.narrative.AbstractNarrativeSignalStrategy;
import io.casehub.blocks.summarisation.narrative.DecisionNarrativePipeline;
import io.casehub.blocks.summarisation.narrative.DecisionSignal;
import io.casehub.blocks.summarisation.narrative.EventStreamBus;
import io.casehub.api.spi.StepOutcomeEvent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class ClinicalNarrativeSignalStrategyTest {

    private EventStreamBus<DecisionSignal> signalBus;
    private DecisionNarrativePipeline pipeline;
    private ClinicalNarrativeSignalStrategy strategy;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setup() {
        signalBus = mock(EventStreamBus.class);
        pipeline = mock(DecisionNarrativePipeline.class);
        strategy = new ClinicalNarrativeSignalStrategy(signalBus, pipeline);
    }

    @Test
    void emitsStepOutcomeForClinicalCaseType() {
        var event = stepOutcome("susar-oversight", "susar-assessment",
            "susar-criteria-evaluator", Map.of());

        strategy.onStepOutcome(event);

        var captor = ArgumentCaptor.forClass(DecisionSignal.class);
        verify(signalBus, atLeastOnce()).publish(captor.capture());
        var signals = captor.getAllValues();
        assertTrue(signals.stream().anyMatch(s -> s instanceof DecisionSignal.StepOutcome));
    }

    @Test
    void ignoresNonClinicalCaseType() {
        var event = stepOutcome("overnight-incident", "classify",
            "classifier", Map.of());

        strategy.onStepOutcome(event);

        verify(signalBus, never()).publish(any());
    }

    @Test
    void emitsRoutingDecisionWhenRoutedAgentPresent() {
        var event = stepOutcome("susar-oversight", "susar-assessment",
            "evaluator", Map.of("routedAgentId", "agent-1", "trustScore", 0.85));

        strategy.onStepOutcome(event);

        var captor = ArgumentCaptor.forClass(DecisionSignal.class);
        verify(signalBus, atLeastOnce()).publish(captor.capture());
        assertTrue(captor.getAllValues().stream()
            .anyMatch(s -> s instanceof DecisionSignal.RoutingDecision));
    }

    @Test
    void emitsForAllSevenCaseTypes() {
        var types = List.of("trial-coordination", "susar-oversight",
            "ae-escalation", "eligibility-screening", "protocol-amendment",
            "deviation-review", "regulatory-submission");

        for (String type : types) {
            reset(signalBus);
            strategy.onStepOutcome(stepOutcome(type, "step", "worker", Map.of()));
            verify(signalBus, atLeastOnce()).publish(any());
        }
    }

    private StepOutcomeEvent stepOutcome(String caseType, String binding,
            String worker, Map<String, Object> context) {
        return new StepOutcomeEvent(UUID.randomUUID(), "default", caseType,
            binding, null, worker, StepOutcomeEvent.Outcome.SUCCESS,
            context, Duration.ofSeconds(1));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=ClinicalNarrativeSignalStrategyTest --batch-mode`
Expected: FAIL — `ClinicalNarrativeSignalStrategy` does not exist

- [ ] **Step 3: Implement ClinicalNarrativeSignalStrategy**

```java
package io.casehub.clinical.narrative;

import io.casehub.blocks.summarisation.narrative.AbstractNarrativeSignalStrategy;
import io.casehub.blocks.summarisation.narrative.DecisionNarrativePipeline;
import io.casehub.blocks.summarisation.narrative.DecisionSignal;
import io.casehub.blocks.summarisation.narrative.DecisionSignal.*;
import io.casehub.blocks.summarisation.narrative.EventStreamBus;
import io.casehub.api.spi.StepOutcomeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import jakarta.annotation.Priority;

import java.time.Instant;
import java.util.List;
import java.util.Set;

@ApplicationScoped
@Alternative
@Priority(1)
public class ClinicalNarrativeSignalStrategy extends AbstractNarrativeSignalStrategy {

    private static final Set<String> CASE_TYPES = Set.of(
        "trial-coordination", "susar-oversight", "ae-escalation",
        "eligibility-screening", "protocol-amendment",
        "deviation-review", "regulatory-submission"
    );

    @Inject
    public ClinicalNarrativeSignalStrategy(
            EventStreamBus<DecisionSignal> signalBus,
            DecisionNarrativePipeline pipeline) {
        super(signalBus, pipeline);
    }

    @Override
    public void onStepOutcome(Object event) {
        if (!(event instanceof StepOutcomeEvent step)) return;
        if (!CASE_TYPES.contains(step.caseType())) return;

        emit(new StepOutcome(step.caseId(), step.bindingName(),
            Instant.now(), step.outcome().name(), step.workerName(),
            null, step.executionDuration()));

        var ctx = step.contextSnapshot();
        if (ctx == null) return;

        String routedAgent = (String) ctx.get("routedAgentId");
        if (routedAgent != null) {
            double score = ctx.containsKey("trustScore")
                ? ((Number) ctx.get("trustScore")).doubleValue() : 0.0;
            emit(new RoutingDecision(step.caseId(), step.bindingName(),
                Instant.now(), routedAgent, "trust-weighted", score,
                List.of(step.workerName()), null));
        }

        if (ctx.containsKey("trustScore")) {
            double ts = ((Number) ctx.get("trustScore")).doubleValue();
            emit(new TrustAssessment(step.caseId(), step.bindingName(),
                Instant.now(), step.workerName(), ts, 0.0, true));
        }

        if (ctx.containsKey("cbrRetrievedCount")) {
            int count = ((Number) ctx.get("cbrRetrievedCount")).intValue();
            double sim = ctx.containsKey("cbrTopSimilarity")
                ? ((Number) ctx.get("cbrTopSimilarity")).doubleValue() : 0.0;
            emit(new CbrRetrieval(step.caseId(), step.bindingName(),
                Instant.now(), count, sim, null, "clinical"));
        }
    }

    @Override
    protected String extractTenancyId(DecisionSignal signal) {
        return null;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=ClinicalNarrativeSignalStrategyTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Write NarrativeSignalBroadcaster**

```java
package io.casehub.clinical.narrative;

import io.casehub.blocks.summarisation.narrative.DecisionSignal;
import io.casehub.blocks.summarisation.narrative.EventStreamBus;
import io.casehub.pages.push.EventBroadcaster;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class NarrativeSignalBroadcaster {

    private static final Logger LOG = Logger.getLogger(NarrativeSignalBroadcaster.class);

    private final EventBroadcaster eventBroadcaster;

    @Inject
    public NarrativeSignalBroadcaster(EventBroadcaster eventBroadcaster) {
        this.eventBroadcaster = eventBroadcaster;
    }

    public void broadcast(DecisionSignal signal) {
        String topic = "clinical:narrative:" + signal.caseId();
        try {
            eventBroadcaster.broadcast(topic, signal);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to broadcast narrative signal for caseId=%s", signal.caseId());
        }
    }
}
```

- [ ] **Step 6: Write NarrativeResource with test**

Test:
```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.ClinicalGroups;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class NarrativeResourceTest {

    @Test
    void returnsNarrativeStateForCase() {
        given()
            .pathParam("caseId", UUID.randomUUID().toString())
            .when().get("/api/narrative/{caseId}")
            .then()
            .statusCode(200)
            .body("steps", notNullValue());
    }

    @Test
    @TestSecurity(user = "anonymous")
    void rejectsUnauthenticated() {
        given()
            .pathParam("caseId", UUID.randomUUID().toString())
            .when().get("/api/narrative/{caseId}")
            .then()
            .statusCode(anyOf(is(401), is(403)));
    }
}
```

Resource:
```java
package io.casehub.clinical.resource;

import io.casehub.blocks.summarisation.narrative.DecisionNarrativePipeline;
import io.casehub.clinical.api.ClinicalGroups;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.UUID;

@Path("/api/narrative")
@Produces(MediaType.APPLICATION_JSON)
public class NarrativeResource {

    @Inject DecisionNarrativePipeline pipeline;

    @GET
    @Path("/{caseId}")
    @RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR,
                   ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
    public Response getNarrative(@PathParam("caseId") UUID caseId) {
        var state = pipeline.getState(caseId);
        return Response.ok(state).build();
    }
}
```

- [ ] **Step 7: Add CDI wiring to application.properties**

Add `ClinicalNarrativeSignalStrategy` to `quarkus.arc.selected-alternatives`.

- [ ] **Step 8: Run full test suite**

Run: `mvn test -pl runtime -Dtest=ClinicalNarrativeSignalStrategyTest,NarrativeResourceTest --batch-mode`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/narrative/
git add runtime/src/main/java/io/casehub/clinical/resource/NarrativeResource.java
git add runtime/src/test/java/io/casehub/clinical/narrative/
git add runtime/src/test/java/io/casehub/clinical/resource/NarrativeResourceTest.java
git add runtime/src/main/resources/application.properties
git commit -m "feat(#165): add ClinicalNarrativeSignalStrategy + NarrativeResource

Wire AbstractNarrativeSignalStrategy for all 7 clinical case types.
Emit StepOutcome, RoutingDecision, TrustAssessment, CbrRetrieval signals.
NarrativeResource serves GET /api/narrative/{caseId}.
NarrativeSignalBroadcaster pushes live signals over WebSocket.

Refs #164"
```

---

## Batch 2: Org Structure + Model Tiers (#166, #167)

### Task 2: Org structure registration in ClinicalTrialCaseHub

**Files:**
- Modify: `runtime/pom.xml` — add eidos org-api + org-runtime dependencies
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalTrialCaseHub.java` — register org structure in `augment()`
- Modify: `runtime/src/main/resources/application.properties` — Jandex index for eidos jars
- Test: `runtime/src/test/java/io/casehub/clinical/service/ClinicalOrgStructureTest.java`

**Interfaces:**
- Consumes: `OrgStructure` (eidos org-api), `OrgRegistry` (eidos org-api)
- Produces: Registered org units and supervision relationships in `OrgRegistry`

- [ ] **Step 1: Add eidos Maven dependencies to runtime/pom.xml**

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-eidos-org-api</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-eidos-org-runtime</artifactId>
  <scope>runtime</scope>
</dependency>
```

Add Jandex index entry in both production and test application.properties:
```properties
quarkus.index-dependency.eidos-org-runtime.group-id=io.casehub
quarkus.index-dependency.eidos-org-runtime.artifact-id=casehub-eidos-org-runtime
```

- [ ] **Step 2: Write test for org structure registration**

```java
package io.casehub.clinical.service;

import io.casehub.eidos.org.api.OrgRegistry;
import io.casehub.eidos.org.api.OrganizationalUnit;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {"SPONSOR", "INVESTIGATOR", "COORDINATOR"})
class ClinicalOrgStructureTest {

    @Inject OrgRegistry orgRegistry;

    @Test
    void registersRootUnit() {
        var root = orgRegistry.findUnit("clinical-ops");
        assertNotNull(root);
        assertEquals("Clinical Trial Operations", root.name());
    }

    @Test
    void registersSafetyTeamWithCapabilities() {
        var safety = orgRegistry.findUnit("safety-team");
        assertNotNull(safety);
        assertTrue(safety.capabilities().contains("safety-monitoring"));
        assertTrue(safety.capabilities().contains("susar-criteria"));
    }

    @Test
    void registersSiteTeam() {
        var site = orgRegistry.findUnit("site-team");
        assertNotNull(site);
        assertTrue(site.capabilities().contains("eligibility-screening"));
    }

    @Test
    void registersRegulatoryTeam() {
        var reg = orgRegistry.findUnit("regulatory-team");
        assertNotNull(reg);
        assertTrue(reg.capabilities().contains("regulatory-submission"));
        assertTrue(reg.capabilities().contains("protocol-amendment-advisor"));
    }

    @Test
    void registersSupervisionRelationships() {
        var supervisors = orgRegistry.supervisors("safety-team");
        assertFalse(supervisors.isEmpty());
    }

    @Test
    void registersEscalationEdges() {
        var escalation = orgRegistry.escalationPath("site-team");
        assertFalse(escalation.isEmpty());
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalOrgStructureTest --batch-mode`
Expected: FAIL — no org units registered

- [ ] **Step 4: Implement org structure in ClinicalTrialCaseHub.augment()**

Add `@Inject OrgRegistry orgRegistry;` field and registration call at end of `augment()`:

```java
OrgStructure.define("default")
    .unit("clinical-ops").name("Clinical Trial Operations").kind("department").add()
    .unit("safety-team").name("Safety Team").kind("functional")
        .parentUnit("clinical-ops")
        .capability("safety-monitoring").capability("susar-criteria").add()
    .unit("site-team").name("Site Team").kind("functional")
        .parentUnit("clinical-ops")
        .capability("eligibility-screening").add()
    .unit("regulatory-team").name("Regulatory Team").kind("functional")
        .parentUnit("clinical-ops")
        .capability("regulatory-submission").capability("protocol-amendment-advisor").add()
    .supervises("trial-supervisor", "safety-team").scope("safety-monitoring").add()
    .supervises("trial-supervisor", "site-team").scope("eligibility-screening").add()
    .supervises("trial-supervisor", "regulatory-team").scope("regulatory-submission").add()
    .escalatesTo("site-team", "safety-team").add()
    .escalatesTo("safety-team", "regulatory-team").add()
    .build()
    .registerAll(orgRegistry);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=ClinicalOrgStructureTest --batch-mode`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/pom.xml
git add runtime/src/main/java/io/casehub/clinical/service/ClinicalTrialCaseHub.java
git add runtime/src/main/resources/application.properties
git add runtime/src/test/resources/application.properties
git add runtime/src/test/java/io/casehub/clinical/service/ClinicalOrgStructureTest.java
git commit -m "feat(#166): register clinical trial agent org structure

Define functional team hierarchy: Safety Team, Site Team, Regulatory Team
under Clinical Trial Operations root. Supervision chain from trial
supervisor to all teams. Escalation edges: site→safety→regulatory.

Refs #164"
```

### Task 3: Model tier declarations in ClinicalAgentSupport

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java` — add tier-based model resolution
- Test: `runtime/src/test/java/io/casehub/clinical/agent/ClinicalAgentSupportModelTierTest.java`

**Interfaces:**
- Consumes: `ModelRegistry` (casehub-platform-api), `ModelQuery` (casehub-platform-api), `ModelTier` (casehub-platform-api), `ModelDescriptor` (casehub-platform-api)
- Produces: Modified `resolveModel(String configKey)` — returns model ID string resolved via tier-based `ModelQuery` when no explicit override

- [ ] **Step 1: Write unit test for tier-based model resolution**

```java
package io.casehub.clinical.agent;

import io.casehub.platform.api.model.ModelDescriptor;
import io.casehub.platform.api.model.ModelQuery;
import io.casehub.platform.api.model.ModelRegistry;
import io.casehub.platform.api.model.ModelTier;
import org.eclipse.microprofile.config.Config;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class ClinicalAgentSupportModelTierTest {

    private Config config;
    private ModelRegistry modelRegistry;

    @BeforeEach
    void setup() {
        config = mock(Config.class);
        modelRegistry = mock(ModelRegistry.class);
    }

    @Test
    void safetyMonitoringDefaultsToFlagship() {
        when(config.getOptionalValue(eq("casehub.clinical.agent.safety-monitoring.model"), eq(String.class)))
            .thenReturn(Optional.empty());
        when(config.getOptionalValue(eq("casehub.clinical.agent.safety-monitoring.tier"), eq(String.class)))
            .thenReturn(Optional.empty());

        var descriptor = mock(ModelDescriptor.class);
        when(descriptor.apiModelId()).thenReturn("claude-opus-4");
        when(modelRegistry.query(any(ModelQuery.class))).thenReturn(List.of(descriptor));

        // resolveModel should query with FLAGSHIP tier
        var captor = org.mockito.ArgumentCaptor.forClass(ModelQuery.class);
        String resolved = resolveModelWithMocks("safety-monitoring", config, modelRegistry);

        assertEquals("claude-opus-4", resolved);
    }

    @Test
    void eligibilityScreeningDefaultsToStandard() {
        when(config.getOptionalValue(eq("casehub.clinical.agent.eligibility-screening.model"), eq(String.class)))
            .thenReturn(Optional.empty());
        when(config.getOptionalValue(eq("casehub.clinical.agent.eligibility-screening.tier"), eq(String.class)))
            .thenReturn(Optional.empty());

        var descriptor = mock(ModelDescriptor.class);
        when(descriptor.apiModelId()).thenReturn("claude-sonnet-5");
        when(modelRegistry.query(any(ModelQuery.class))).thenReturn(List.of(descriptor));

        String resolved = resolveModelWithMocks("eligibility-screening", config, modelRegistry);
        assertEquals("claude-sonnet-5", resolved);
    }

    @Test
    void explicitModelOverridesTier() {
        when(config.getOptionalValue(eq("casehub.clinical.agent.safety-monitoring.model"), eq(String.class)))
            .thenReturn(Optional.of("claude-haiku-4-5"));

        String resolved = resolveModelWithMocks("safety-monitoring", config, modelRegistry);
        assertEquals("claude-haiku-4-5", resolved);
        verify(modelRegistry, never()).query(any());
    }

    @Test
    void explicitTierConfigOverridesDefault() {
        when(config.getOptionalValue(eq("casehub.clinical.agent.safety-monitoring.model"), eq(String.class)))
            .thenReturn(Optional.empty());
        when(config.getOptionalValue(eq("casehub.clinical.agent.safety-monitoring.tier"), eq(String.class)))
            .thenReturn(Optional.of("STANDARD"));

        var descriptor = mock(ModelDescriptor.class);
        when(descriptor.apiModelId()).thenReturn("claude-sonnet-5");
        when(modelRegistry.query(any(ModelQuery.class))).thenReturn(List.of(descriptor));

        String resolved = resolveModelWithMocks("safety-monitoring", config, modelRegistry);
        assertEquals("claude-sonnet-5", resolved);
    }

    @Test
    void fallsBackToSonnetWhenRegistryEmpty() {
        when(config.getOptionalValue(anyString(), eq(String.class))).thenReturn(Optional.empty());
        when(modelRegistry.query(any(ModelQuery.class))).thenReturn(List.of());

        String resolved = resolveModelWithMocks("safety-monitoring", config, modelRegistry);
        assertEquals("sonnet", resolved);
    }

    private String resolveModelWithMocks(String configKey, Config config, ModelRegistry registry) {
        // This calls the static resolution logic extracted from ClinicalAgentSupport
        return ClinicalAgentSupport.resolveModel(configKey, config, registry);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=ClinicalAgentSupportModelTierTest --batch-mode`
Expected: FAIL — `resolveModel` does not accept `ModelRegistry`

- [ ] **Step 3: Modify ClinicalAgentSupport to add tier-based resolution**

Add `ModelRegistry` injection and modify `resolveModel()`:

```java
@Inject ModelRegistry modelRegistry;

private static final Map<String, ModelTier> DEFAULT_TIERS = Map.of(
    "safety-monitoring", ModelTier.FLAGSHIP,
    "susar-criteria", ModelTier.FLAGSHIP,
    "protocol-amendment", ModelTier.FLAGSHIP,
    "trial-supervision", ModelTier.FLAGSHIP,
    "eligibility-screening", ModelTier.STANDARD
);

static String resolveModel(String configKey, Config config, ModelRegistry modelRegistry) {
    Optional<String> explicitModel = config.getOptionalValue(
        "casehub.clinical.agent." + configKey + ".model", String.class);
    if (explicitModel.isPresent()) return explicitModel.get();

    Optional<String> tierStr = config.getOptionalValue(
        "casehub.clinical.agent." + configKey + ".tier", String.class);
    ModelTier tier = tierStr.map(ModelTier::valueOf)
        .orElse(DEFAULT_TIERS.getOrDefault(configKey, ModelTier.FLAGSHIP));

    List<ModelDescriptor> models = modelRegistry.query(
        ModelQuery.builder().tier(tier).build());
    if (!models.isEmpty()) return models.get(0).apiModelId();

    return "sonnet";
}
```

Update the instance method to delegate to the static method.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=ClinicalAgentSupportModelTierTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Run existing ClinicalAgentSupportTest to verify no regressions**

Run: `mvn test -pl runtime -Dtest=ClinicalAgentSupportTest --batch-mode`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java
git add runtime/src/test/java/io/casehub/clinical/agent/ClinicalAgentSupportModelTierTest.java
git commit -m "feat(#167): add model tier declarations per agent capability

FLAGSHIP: safety-monitoring, susar-criteria, protocol-amendment, trial-supervision.
STANDARD: eligibility-screening.
Tier-based resolution via ModelRegistry.query() with explicit model and
tier config overrides. Falls back to 'sonnet' when registry empty.

Refs #164"
```

---

## Batch 3: Model Registry Wiring (#168)

### Task 4: Model registry configuration

**Files:**
- Modify: `runtime/src/main/resources/application.properties` — model source config
- Test: `runtime/src/test/java/io/casehub/clinical/agent/ModelRegistryWiringTest.java`

**Interfaces:**
- Consumes: `ModelRegistry` (casehub-platform-api), `VertexCloudModelSource` (casehub-platform), config properties
- Produces: Populated `ModelRegistry` with Claude model descriptors from Vertex AI

- [ ] **Step 1: Write wiring test**

```java
package io.casehub.clinical.agent;

import io.casehub.platform.api.model.ModelRegistry;
import io.casehub.platform.api.model.ModelTier;
import io.casehub.platform.api.model.ModelQuery;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {"SPONSOR"})
class ModelRegistryWiringTest {

    @Inject ModelRegistry modelRegistry;

    @Test
    void modelRegistryIsNotNoOp() {
        assertNotNull(modelRegistry);
        var all = modelRegistry.all();
        assertNotNull(all);
    }

    @Test
    void canQueryByTier() {
        var flagships = modelRegistry.query(
            ModelQuery.builder().tier(ModelTier.FLAGSHIP).build());
        assertNotNull(flagships);
    }
}
```

- [ ] **Step 2: Run test to verify baseline**

Run: `mvn test -pl runtime -Dtest=ModelRegistryWiringTest --batch-mode`
Expected: either PASS (NoOpModelRegistry returns empty) or FAIL (if assertion checks non-empty). Adjust assertions based on test environment — in CI without Vertex credentials, the registry may be empty.

- [ ] **Step 3: Add model source config to application.properties**

```properties
# Model registry — Vertex AI model source
casehub.platform.model.vertex.enabled=${VERTEX_MODEL_DISCOVERY:false}
casehub.platform.model.vertex.project-id=${VERTEX_PROJECT_ID:}
casehub.platform.model.vertex.region=${VERTEX_REGION:us-central1}
```

In test application.properties, ensure model source is disabled (tests use NoOp or InMemory):
```properties
casehub.platform.model.vertex.enabled=false
```

- [ ] **Step 4: Run full batch 2+3 tests**

Run: `mvn test -pl runtime -Dtest=ClinicalOrgStructureTest,ClinicalAgentSupportModelTierTest,ModelRegistryWiringTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/resources/application.properties
git add runtime/src/test/resources/application.properties
git add runtime/src/test/java/io/casehub/clinical/agent/ModelRegistryWiringTest.java
git commit -m "feat(#168): wire model registry for tier-based routing

Configure VertexCloudModelSource for production model discovery.
Disabled in tests (NoOpModelRegistry). ClinicalAgentSupport tier-based
resolution (#167) uses ModelRegistry.query() at runtime.

Refs #164"
```

---

## Batch 4: UI Conversion (#169)

### Task 5: Convert Safety Workbench to dockWorkbench + wire new panels

**Files:**
- Modify: `runtime/src/main/webui/package.json` — add blocks-ui panel dependencies
- Modify: `runtime/src/main/webui/src/index.ts` — add registerPanel calls
- Modify: `runtime/src/main/webui/src/views/safety-workbench.ts` — convert to dockWorkbench
- Modify: `runtime/src/main/webui/src/views/protocol-workbench.ts` — convert to dockWorkbench
- Modify: `runtime/src/main/webui/src/views/operations.ts` — convert to dockWorkbench

**Interfaces:**
- Consumes: `registerPanel()` from `@casehubio/pages-runtime`, `dockWorkbench()` / `hostPanel()` from `@casehubio/pages-ui`, blocks-ui panel components
- Produces: Converted layouts with narrative-timeline, orchestration-workbench, trust-workbench, conversation-viewer, routing-rationale panels

- [ ] **Step 1: Add npm dependencies to package.json**

Add to `dependencies`:
```json
"@casehubio/blocks-ui-orchestration-workbench": "*",
"@casehubio/blocks-ui-trust-workbench": "*",
"@casehubio/blocks-ui-conversation-viewer": "*",
"@casehubio/blocks-ui-routing-rationale": "*"
```

Add to `resolutions`:
```json
"@casehubio/blocks-ui-orchestration-workbench": "portal:./.casehub-packages/packages/orchestration-workbench",
"@casehubio/blocks-ui-trust-workbench": "portal:./.casehub-packages/packages/trust-workbench",
"@casehubio/blocks-ui-conversation-viewer": "portal:./.casehub-packages/packages/conversation-viewer",
"@casehubio/blocks-ui-routing-rationale": "portal:./.casehub-packages/packages/routing-rationale"
```

Note: `narrative-timeline` and `org-diagram` are NOT in `.casehub-packages/`.
Add them when published from blocks-ui repo. The layouts reserve panel slots
for them with TODO comments.

- [ ] **Step 2: Add registerPanel calls to index.ts**

```typescript
import "@casehubio/blocks-ui-orchestration-workbench";
import "@casehubio/blocks-ui-trust-workbench";
import "@casehubio/blocks-ui-conversation-viewer";
import "@casehubio/blocks-ui-routing-rationale";

registerPanel("orchestration-workbench", "blocks-orchestration-workbench");
registerPanel("trust-workbench", "blocks-trust-workbench");
registerPanel("conversation-viewer", "blocks-conversation-viewer");
registerPanel("routing-rationale", "blocks-routing-rationale");
```

- [ ] **Step 3: Convert safety-workbench.ts to dockWorkbench**

Replace existing `columns([5, 7], ...)` layout with `dockWorkbench()`:

```typescript
import { dockWorkbench, hostPanel } from "@casehubio/pages-ui";
import type { DockPanelConfig } from "@casehubio/pages-ui";

const aeDetail: DockPanelConfig = {
  key: "ae-detail", label: "AE Detail", icon: "alert-triangle",
  defaultOpen: true, content: /* existing AE detail tabs content */
};
const routingRationale: DockPanelConfig = {
  key: "routing", label: "Routing Rationale", icon: "git-branch",
  defaultOpen: false, content: hostPanel("routing-rationale", {})
};
const trustWorkbench: DockPanelConfig = {
  key: "trust", label: "Trust Workbench", icon: "shield",
  defaultOpen: false, content: hostPanel("trust-workbench", {})
};
// narrative-timeline reserved — add when package available
const auditTrail: DockPanelConfig = {
  key: "audit", label: "Audit Trail", icon: "file-text",
  defaultOpen: false, content: /* existing audit content */
};

export const safetyWorkbench = dockWorkbench({
  centre: /* existing AE data table */,
  right: [aeDetail, routingRationale, trustWorkbench],
  bottom: [auditTrail]
});
```

Preserve all existing data tables, tabs, and custom components — move them
into dockWorkbench zones rather than rewriting. The conversion is structural
(layout container), not functional (component behaviour).

- [ ] **Step 4: Convert protocol-workbench.ts to dockWorkbench**

Same pattern: existing content into `centre`, `conversation-viewer` and
`routing-rationale` into `right` zone.

- [ ] **Step 5: Convert operations.ts to dockWorkbench**

Existing trial dashboard into `centre`, `orchestration-workbench` and
`trust-workbench` into `right`, SLA/compliance into `bottom`.

- [ ] **Step 6: Build and verify**

Run: `cd runtime/src/main/webui && yarn install && yarn build && yarn typecheck`
Expected: Build succeeds, no type errors.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/webui/
git commit -m "feat(#169): convert workbenches to dockWorkbench + wire blocks-ui panels

Convert Safety Workbench, Protocol Workbench, Operations views from
tree/tabs/columns to dockWorkbench with registerPanel pattern.
Wire orchestration-workbench, trust-workbench, conversation-viewer,
routing-rationale panels. narrative-timeline and org-diagram deferred
until published to .casehub-packages.

Refs #164"
```

---

## Batch 5: Model Selection in Narratives (#170)

### Task 6: ModelSelectionEvent + narrative signal extension

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/agent/ModelSelectionEvent.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java` — fire ModelSelectionEvent after model resolution
- Modify: `runtime/src/main/java/io/casehub/clinical/narrative/ClinicalNarrativeSignalStrategy.java` — observe ModelSelectionEvent
- Test: `runtime/src/test/java/io/casehub/clinical/narrative/ModelSelectionNarrativeTest.java`

**Interfaces:**
- Consumes: `ClinicalAgentSupport` (modified), `ClinicalNarrativeSignalStrategy` (Batch 1), `ModelRegistry` (Batch 3)
- Produces: `ModelSelectionEvent` record, narrative signals for model selection decisions

- [ ] **Step 1: Write test for model selection narrative signal**

```java
package io.casehub.clinical.narrative;

import io.casehub.blocks.summarisation.narrative.DecisionSignal;
import io.casehub.blocks.summarisation.narrative.EventStreamBus;
import io.casehub.clinical.agent.ModelSelectionEvent;
import io.casehub.platform.api.model.ModelTier;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class ModelSelectionNarrativeTest {

    @Test
    @SuppressWarnings("unchecked")
    void emitsStepOutcomeForModelSelection() {
        var signalBus = mock(EventStreamBus.class);
        var pipeline = mock(io.casehub.blocks.summarisation.narrative.DecisionNarrativePipeline.class);
        var strategy = new ClinicalNarrativeSignalStrategy(signalBus, pipeline);

        var event = new ModelSelectionEvent(
            UUID.randomUUID(), "susar-criteria", "susar-criteria",
            ModelTier.FLAGSHIP, "claude-opus-4", "Claude Opus 4");

        strategy.onModelSelection(event);

        var captor = ArgumentCaptor.forClass(DecisionSignal.class);
        verify(signalBus, atLeastOnce()).publish(captor.capture());
        var signal = captor.getValue();
        assertInstanceOf(DecisionSignal.StepOutcome.class, signal);
        var outcome = (DecisionSignal.StepOutcome) signal;
        assertEquals("MODEL_SELECTED", outcome.status());
        assertEquals("claude-opus-4", outcome.workerId());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=ModelSelectionNarrativeTest --batch-mode`
Expected: FAIL — `ModelSelectionEvent` does not exist

- [ ] **Step 3: Create ModelSelectionEvent record**

```java
package io.casehub.clinical.agent;

import io.casehub.platform.api.model.ModelTier;
import java.util.UUID;

public record ModelSelectionEvent(
    UUID caseId,
    String capabilityName,
    String configKey,
    ModelTier tier,
    String modelId,
    String modelDisplayName
) {}
```

- [ ] **Step 4: Add onModelSelection to ClinicalNarrativeSignalStrategy**

```java
public void onModelSelection(@ObservesAsync ModelSelectionEvent event) {
    emit(new StepOutcome(
        event.caseId(), event.configKey(),
        Instant.now(), "MODEL_SELECTED", event.modelId(),
        null, Duration.ZERO));
}
```

- [ ] **Step 5: Fire ModelSelectionEvent from ClinicalAgentSupport**

After successful model resolution and before `agentProvider.invoke()`,
fire the event:

```java
@Inject Event<ModelSelectionEvent> modelSelectionEvent;

// in invoke(), after resolveModel():
if (request.caseId() != null) {
    modelSelectionEvent.fireAsync(new ModelSelectionEvent(
        request.caseId(), request.configKey(), request.configKey(),
        resolvedTier, resolvedModelId, resolvedModelId));
}
```

This requires `ClinicalAgentRequest` to carry `caseId` — verify it does,
or add it.

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=ModelSelectionNarrativeTest --batch-mode`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `mvn test -pl runtime --batch-mode`
Expected: PASS — all existing + new tests green

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/agent/ModelSelectionEvent.java
git add runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java
git add runtime/src/main/java/io/casehub/clinical/narrative/ClinicalNarrativeSignalStrategy.java
git add runtime/src/test/java/io/casehub/clinical/narrative/ModelSelectionNarrativeTest.java
git commit -m "feat(#170): surface model selection in decision narratives

Fire ModelSelectionEvent from ClinicalAgentSupport after tier-based model
resolution. ClinicalNarrativeSignalStrategy observes and emits
MODEL_SELECTED StepOutcome signal with model ID. Narratives show which
LLM model was used for each clinical agent decision.

Closes #165, Closes #166, Closes #167, Closes #168, Closes #169, Closes #170
Refs #164"
```

---

## References

- `specs/issue-164-agent-org-model-selection/2026-09-14-agent-org-model-selection-design.md` — design spec this plan implements
- `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java` — existing model resolution
- `runtime/src/main/java/io/casehub/clinical/service/ClinicalTrialCaseHub.java` — augment() integration point
- `runtime/src/main/java/io/casehub/clinical/push/ClinicalPushEndpoint.java` — existing WebSocket push
- `runtime/src/main/java/io/casehub/clinical/service/ClinicalCascadeBroadcaster.java` — existing broadcast pattern
- `runtime/src/main/webui/src/views/safety-workbench.ts` — current Safety Workbench layout
- fsitrading `FsiNarrativeSignalStrategy` — reference narrative strategy implementation
- fsitrading `site.ts` — reference dockWorkbench layout
- casehubio/clinical#164 — epic
- casehubio/fsitrading#41 — parallel epic
