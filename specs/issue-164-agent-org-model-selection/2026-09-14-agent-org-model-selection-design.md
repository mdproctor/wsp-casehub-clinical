# Agent Organization + Model Selection — Design Spec

**Epic:** casehubio/clinical#164
**Child issues:** #165, #166, #167, #168, #169, #170
**Date:** 2026-09-14
**Branch:** issue-164-agent-org-model-selection

## Overview

Wire four platform SPIs into clinical to give agents organisational structure,
tier-appropriate model selection, and visible decision narratives:

1. **Decision narratives** — `ClinicalNarrativeSignalStrategy` + `narrative-timeline` panel
2. **Org structure** — functional team hierarchy via `OrgStructure.define()`
3. **Model tiers** — per-capability tier declarations (FLAGSHIP / STANDARD)
4. **Model registry** — tier-based model resolution via `ModelRegistry` + `RoutingAgentProvider`
5. **UI conversion** — dockWorkbench layout with new blocks-ui panels
6. **Narrative model signals** — model selection surfaced in decision narratives

All platform SPIs are shipped and ready (eidos #150, #172; platform #286/#287;
blocks #241). fsitrading #41 is the parallel epic using the same SPIs.

## Foundation SPIs Used

| SPI | Module | Key types |
|-----|--------|-----------|
| Org model | eidos `org-api` | `OrgStructure`, `OrgRegistry`, `OrganizationalUnit`, `AgentRelationship`, `RelationshipKind` |
| Model tier vocab | eidos `vocab` | `ModelTierTerm` (FLAGSHIP, STANDARD, FAST, EMBEDDING) |
| Model registry | `casehub-platform-api` | `ModelTier`, `ModelDescriptor`, `ModelRegistry`, `ModelQuery` |
| Model routing | `casehub-platform-agent-router` | `RoutingAgentProvider` (already on classpath — Layer 11) |
| Narrative signals | blocks `summarisation` | `AbstractNarrativeSignalStrategy`, `DecisionSignal`, `DecisionNarrativePipeline`, `EventStreamBus` |
| Narrative UI | blocks-ui | `narrative-timeline` (`blocks-narrative-timeline` element) |

## #165 — Decision Narratives

### ClinicalNarrativeSignalStrategy

`@ApplicationScoped @Alternative @Priority(1)` extending
`AbstractNarrativeSignalStrategy`.

```java
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
        if (ctx != null) {
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
    }

    @Override
    protected String extractTenancyId(DecisionSignal signal) {
        return null; // multi-tenant extraction deferred
    }
}
```

**CDI wiring:** Add to `quarkus.arc.selected-alternatives`. Requires
`casehub-blocks-summarisation` on classpath (provides
`AbstractNarrativeSignalStrategy`, `DecisionNarrativePipeline`,
`EventStreamBus<DecisionSignal>`).

### NarrativeResource

```
GET /api/narrative/{caseId}
```

`@RolesAllowed({SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR})`. Returns
`NarrativeState` from `DecisionNarrativePipeline.getState(caseId)`.

`NarrativeState` structure (from blocks):
```json
{
  "steps": [
    {
      "caseId": "uuid",
      "stepName": "susar-assessment",
      "signals": [
        { "signalType": "STEP_OUTCOME", "summary": "...", "keyFacts": {...}, "confidence": 0.95 },
        { "signalType": "ROUTING", "summary": "...", "keyFacts": {...}, "confidence": 0.9 }
      ],
      "from": "2026-09-14T10:00:00Z",
      "to": "2026-09-14T10:00:05Z"
    }
  ],
  "narrative": { "summary": "..." }
}
```

### WebSocket Push for Live Signals

`NarrativeSignalBroadcaster` — `@ApplicationScoped` bean observing
`DecisionSignal` emissions. On each signal, broadcasts to topic
`clinical:narrative:{caseId}` via the existing `EventBroadcaster`.

Same pattern as `ClinicalCascadeBroadcaster` which pushes cascade events
to `clinical:ae:{aeId}:cascade`.

The `narrative-timeline` component loads initial state via REST
(`DataSourceMixin`). Live updates use the pages push infrastructure:
when `NarrativeSignalBroadcaster` pushes a signal event to the WS
topic, the pages framework triggers the component to re-fetch from
REST — no client-side signal merging required. Same refresh-on-push
pattern used by existing clinical components.

## #166 — Org Structure

### Functional Team Hierarchy

```
Clinical Trial Operations (root)
├── Safety Team
│   ├── Members: safety-monitoring agent, susar-criteria agent, dsmb-safety-signal agent
│   ├── Supervised by: Safety Officer
│   └── Capabilities: safety-monitoring, susar-criteria
├── Site Team (one logical team per trial site)
│   ├── Members: eligibility-screening binding, pi-authorisation binding
│   ├── Supervised by: Site PI
│   └── Capabilities: eligibility-screening
├── Regulatory Team
│   ├── Members: regulatory-submission binding, protocol-amendment-advisor agent
│   ├── Supervised by: Sponsor
│   └── Capabilities: regulatory-submission, protocol-amendment-advisor
└── Trial Supervisor
    ├── Members: trial-supervision agent
    ├── Supervises: Safety Team, Site Team, Regulatory Team
    └── Capabilities: trial-supervision
```

**Supervision chain:** site coordinator → PI → safety officer → sponsor

**Escalation edges:**
- Site Team → Safety Team (AE escalation)
- Safety Team → Regulatory Team (regulatory reporting)

### Registration

In `ClinicalTrialCaseHub.augment()`, after existing worker registration.
Uses `"default"` as tenancyId (matches existing clinical pattern — proper
per-tenant org structures deferred until multi-tenancy is production-ready):

```java
@Inject OrgRegistry orgRegistry;

// in augment():
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

**Dependencies:** eidos `org-api` (compile) + `org-runtime` (runtime) on classpath.

## #167 — Model Tier Declarations

### Per-Capability Tier Mapping

| Capability | Tier | Rationale |
|-----------|------|-----------|
| `safety-monitoring` | FLAGSHIP | Life-safety reasoning, AE causality assessment |
| `susar-criteria` | FLAGSHIP | Serious/unexpected/related classification — regulatory consequence |
| `protocol-amendment-advisor` | FLAGSHIP | Deep ICH GCP interpretation |
| `trial-supervision` | FLAGSHIP | Cross-site pattern detection, safety signal synthesis |
| `eligibility-screening` | STANDARD | Structured criteria matching against I/E criteria |

### ClinicalAgentSupport Changes

New config property: `casehub.clinical.agent.<key>.tier` (optional).

`resolveModel()` enhanced:

```java
private String resolveModel(String configKey) {
    // 1. Explicit model override (existing — highest priority)
    Optional<String> explicitModel = config.getOptionalValue(
        "casehub.clinical.agent." + configKey + ".model", String.class);
    if (explicitModel.isPresent()) return explicitModel.get();

    // 2. Tier-based resolution (new)
    Optional<String> tierStr = config.getOptionalValue(
        "casehub.clinical.agent." + configKey + ".tier", String.class);
    ModelTier tier = tierStr.map(ModelTier::valueOf)
        .orElse(DEFAULT_TIERS.getOrDefault(configKey, ModelTier.FLAGSHIP));

    List<ModelDescriptor> models = modelRegistry.query(
        ModelQuery.builder().tier(tier).build());
    if (!models.isEmpty()) return models.get(0).apiModelId();

    // 3. Fallback to default model string
    return "sonnet";
}
```

`DEFAULT_TIERS` is a static map:
```java
private static final Map<String, ModelTier> DEFAULT_TIERS = Map.of(
    "safety-monitoring", ModelTier.FLAGSHIP,
    "susar-criteria", ModelTier.FLAGSHIP,
    "protocol-amendment", ModelTier.FLAGSHIP,
    "trial-supervision", ModelTier.FLAGSHIP,
    "eligibility-screening", ModelTier.STANDARD
);
```

## #168 — Model Registry Wiring

Clinical already uses Vertex AI for Claude (Layer 11). The model registry
needs a model source to populate `ModelDescriptor` entries.

**Config additions (`application.properties`):**
```properties
casehub.platform.model.vertex.enabled=true
casehub.platform.model.vertex.project-id=${VERTEX_PROJECT_ID}
casehub.platform.model.vertex.region=${VERTEX_REGION:us-central1}
```

`VertexCloudModelSource` auto-discovers available Claude models and
populates `ModelRegistry` with `ModelDescriptor` entries including tier
classification. `NoOpModelRegistry` (`@DefaultBean`) is displaced.

`RoutingAgentProvider` (already wired) starts resolving via `ModelRegistry`
once populated — no additional wiring needed.

Per-agent model config (`casehub.clinical.agent.<key>.model`) continues
to work as an escape hatch — explicit model overrides tier-based resolution.

## #169 — UI Conversion to dockWorkbench

### Layout Strategy

Convert all three views from tree/tabs/columns to dockWorkbench:

**Safety Workbench:**
- Centre: AE data table + detail panel (existing content)
- Right zone: narrative-timeline, routing-rationale, trust-workbench
- Bottom zone: audit-trail, SLA indicators, compliance summary

**Protocol Workbench:**
- Centre: deviation data table + detail panel (existing content)
- Right zone: conversation-viewer (PI deliberation channel), routing-rationale
- Bottom zone: audit-trail, commitment lifecycle

**Operations:**
- Centre: trial dashboard (existing content)
- Right zone: orchestration-workbench, trust-workbench
- Bottom zone: SLA health, compliance, GDPR erasure

### Panel Registration

In `webui/src/index.ts`, add `registerPanel()` calls alongside existing
`customElements.define()`:

```typescript
import "@casehubio/blocks-ui-narrative-timeline";
import "@casehubio/blocks-ui-orchestration-workbench";
import "@casehubio/blocks-ui-trust-workbench";
import "@casehubio/blocks-ui-conversation-viewer";
import "@casehubio/blocks-ui-routing-rationale";

registerPanel("narrative-timeline", "blocks-narrative-timeline");
registerPanel("orchestration-workbench", "blocks-orchestration-workbench");
registerPanel("trust-workbench", "blocks-trust-workbench");
registerPanel("conversation-viewer", "blocks-conversation-viewer");
registerPanel("routing-rationale", "blocks-routing-rationale");
```

### Panel Data Sources

| Panel | Data source |
|-------|------------|
| narrative-timeline | `{ endpoint: "/api/narrative/{caseId}" }` + WS topic `clinical:narrative:{caseId}` |
| orchestration-workbench | Engine case state via existing `/api/trials/{trialId}` endpoints |
| trust-workbench | Existing trust data endpoints (trust scores, attestations) |
| conversation-viewer | Qhorus channel messages via existing channel endpoints |
| routing-rationale | Existing routing decision record endpoints |

### narrative-timeline Package

Not currently in `.casehub-packages/`. Options:
1. Publish SNAPSHOT from blocks-ui repo, add to `.casehub-packages/`
2. Add `file:` reference to local blocks-ui clone (dev-only)

Use option 1 for CI compatibility — same as all other blocks-ui packages.

### org-diagram

Not yet available as a blocks-ui package. Wire when published. The org
structure data (#166) will be ready — the panel just needs the package.

## #170 — Model Selection in Decision Narratives

### ModelSelectionEvent

New CDI event fired by `ClinicalAgentSupport` after successful model
resolution:

```java
public record ModelSelectionEvent(
    UUID caseId,
    String capabilityName,
    String configKey,
    ModelTier tier,
    String modelId,
    String modelDisplayName
) {}
```

Fired after `modelRegistry.query()` resolves a model, before
`agentProvider.invoke()`.

### ClinicalNarrativeSignalStrategy Extension

Observes `ModelSelectionEvent` and emits a narrative signal:

```java
public void onModelSelection(@ObservesAsync ModelSelectionEvent event) {
    emit(new StepOutcome(
        event.caseId(), event.configKey(),
        Instant.now(), "MODEL_SELECTED", event.modelId(),
        null, Duration.ZERO));
}
```

The narrative-timeline renders: "Used Claude Opus (FLAGSHIP tier) for
SUSAR assessment — life-safety reasoning requires deep analysis."

Model cost/latency metadata included if available from `ModelDescriptor`.

## Dependency Order

```
#165 Decision narratives (independent)
#166 Org structure (independent)
#167 Model tier declarations (independent)
  ├── #168 Model registry wiring (depends on #167)
  │   └── #170 Model selection in narratives (depends on #165 + #168)
  └── #169 UI components (depends on #166 for org-diagram data)
```

#165, #166, #167 can be worked in parallel. #168 follows #167. #169
can start independently (the panels work without org data — org-diagram
is deferred until its package ships). #170 is the capstone requiring
both narratives and model registry.

## Testing Strategy

### Unit Tests
- `ClinicalNarrativeSignalStrategyTest` — verify signal emission for each case type, verify filtering, verify context snapshot extraction
- `ClinicalAgentSupport` model resolution — verify tier-based lookup, verify explicit model override wins, verify fallback chain
- `NarrativeSignalBroadcaster` — verify WebSocket topic naming and broadcast

### Integration Tests (`@QuarkusTest`)
- `NarrativeResourceTest` — GET `/api/narrative/{caseId}` returns `NarrativeState`, RBAC enforcement
- Org structure registration — verify units and relationships registered in `OrgRegistry` after case definition load
- Model registry — verify `VertexCloudModelSource` populates registry, verify `ModelQuery` by tier resolves correct models

### Wiring Tests
- CDI: verify `ClinicalNarrativeSignalStrategy` displaces default, `DecisionNarrativePipeline` injectable
- CDI: verify `OrgRegistry` injectable after eidos jars added
- CDI: verify `ModelRegistry` populated (not `NoOpModelRegistry`) when model source configured

## Dependencies to Add

### Maven (runtime/pom.xml)

```xml
<!-- Narrative signals (verify exact artifactId from blocks repo) -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-blocks-summarisation</artifactId>
</dependency>

<!-- Org model -->
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

### npm (webui/package.json)

```json
"@casehubio/blocks-ui-narrative-timeline": "*",
"@casehubio/blocks-ui-orchestration-workbench": "*",
"@casehubio/blocks-ui-trust-workbench": "*",
"@casehubio/blocks-ui-conversation-viewer": "*",
"@casehubio/blocks-ui-routing-rationale": "*"
```

## References

- fsitrading `FsiNarrativeSignalStrategy` — reference implementation for narrative strategy pattern
- fsitrading `OvernightIncidentCaseHub.augment()` — reference for CaseHub integration
- fsitrading `site.ts` (ops-centre layout) — reference for dockWorkbench panel wiring
- `ClinicalCascadeBroadcaster.java` — existing WebSocket push pattern for real-time events
- `ClinicalAgentSupport.java` — existing model resolution, injection point for tier-based changes
- `ClinicalPushEndpoint.java` — existing `/ws/push` WebSocket infrastructure
- eidos `OrgStructure` API — fluent DSL for org hierarchy definition
- platform `ModelTier`, `ModelRegistry`, `ModelQuery` — tier-based model resolution
- blocks `AbstractNarrativeSignalStrategy`, `DecisionSignal` — narrative signal SPI
- ICH E6(R3) GCP — clinical trial role definitions (PI, sponsor, safety officer)
