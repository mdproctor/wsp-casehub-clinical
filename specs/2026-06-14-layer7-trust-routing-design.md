# Layer 7 — Trust-Weighted Safety Agent Routing

**Date:** 2026-06-14  
**Issue:** casehubio/clinical#8  
**Branch:** issue-8-trust-weighted-safety-routing  
**Status:** Approved for implementation

---

## Context

Layer 7 closes the compliance gap: "reliable agents on high-risk decisions." Layers 1–8 established domain entities, SLA, audit trail, PI authorisation, engine orchestration, trial-level blackboard, and action-risk oversight gates. Layer 7 adds trust-weighted routing so that safety agent selection improves as evidence accumulates — closing the gap between "agent makes decisions" and "the right agent makes decisions."

Three independent capabilities:
1. Trust routing infrastructure (policy + attestation + engine wiring)
2. Regulatory-submission path for Grade 5 unexpected AEs (21 CFR 312.32(c)(1)(i))
3. `AeEscalationCompletedEvent.unexpected` API extension

---

## Architecture Overview

```
1. Trust Routing Infrastructure
   ClinicalTrustRoutingPolicyProvider  →  TrustWeightedAgentStrategy (engine-ledger)
   TrustAttestationStrategy SPI        →  ActorTrustScoreRepository
     WorkerDecisionEvent               ↗  (observation signal)
     AeEscalationCompletedEvent        ↗  (quality signal: safety-accuracy dimension)

2. Regulatory-Submission Path
   AdverseEventReportedEvent  →  RegulatorySubmissionCaseService  →  startCase()
                                  ClinicalRegulatorySubmissionCaseHub
                                  regulatory-submission.yaml

3. API Extension
   AeEscalationCompletedEvent  ← boolean unexpected (from AdverseEvent entity, via case context)
```

**New dependency:** `casehub-engine-ledger` — activates `TrustWeightedAgentStrategy` by classpath presence (GE-20260602-c68651).

**No new Flyway migrations for trust tables** — `ActorTrustScoreRepository` writes to ledger's existing trust score table. New migrations: V112 (AdverseEvent regulatory submission fields, default datasource); V2023 (RegulatorySubmissionLedgerEntry join table, qhorus datasource).

---

## §1 TrustAttestationStrategy SPI

### Interface (`api/spi/TrustAttestationStrategy.java`)

Pure Java, no framework dependencies. CDI displacement contract: `@DefaultBean @ApplicationScoped` default; a future ML-based strategy displaces by being `@ApplicationScoped` without `@DefaultBean`.

```java
public interface TrustAttestationStrategy {
    /** Observation signal — called after every worker decision. */
    void onWorkerDecision(String workerId, String capabilityTag, String tenancyId);

    /** Quality signal — called when an AE escalation outcome is known. */
    void onAeEscalationOutcome(String workerId, String capabilityTag,
                                AeOutcomeSignal signal, String tenancyId);
}
```

### Signal Enum (`api/spi/AeOutcomeSignal.java`)

```java
public enum AeOutcomeSignal {
    CORRECT,    // decision validated — increments alpha (successes) in Bayesian Beta
    INCORRECT,  // decision wrong — increments beta (failures)
    UNCERTAIN   // no quality information — no upsert, observation already counted
}
```

### Outcome Mapping (AeEscalationCompletedEvent → AeOutcomeSignal)

| Condition | Signal |
|-----------|--------|
| SUSAR gate approved + dsmbEscalated when required | `CORRECT` |
| SUSAR gate rejected or expired | `INCORRECT` |
| No SUSAR gate, no unexpected finding | `UNCERTAIN` |

### DefaultTrustAttestationStrategy (`runtime/service/DefaultTrustAttestationStrategy.java`)

`@DefaultBean @ApplicationScoped`. Injects `ActorTrustScoreRepository`.

- `onWorkerDecision()`: calls `repository.upsert(workerId, CAPABILITY, capabilityTag, ...)` — increments `observations`, keeps alpha unchanged (decision seen, not yet scored)
- `onAeEscalationOutcome(CORRECT)`: upserts `safety-accuracy` dimension, increments alpha
- `onAeEscalationOutcome(INCORRECT)`: upserts `safety-accuracy` dimension, increments beta
- `onAeEscalationOutcome(UNCERTAIN)`: no-op

Uses `ActorTrustScoreRepository.upsert()` — NOT `updateGlobalTrustScore()` which is a silent no-op for new actors (GE-20260531-769f9c).

### Thin CDI Observers (`runtime/service/`)

**`WorkerDecisionAttestationObserver @ApplicationScoped`** — `@ObservesAsync WorkerDecisionEvent`, delegates to `TrustAttestationStrategy.onWorkerDecision(event.workerId(), event.capabilityTag(), event.tenancyId())`.

**`AeEscalationAttestationObserver @ApplicationScoped`** — `@ObservesAsync AeEscalationCompletedEvent`, derives `AeOutcomeSignal` from event fields, delegates to `TrustAttestationStrategy.onAeEscalationOutcome(...)`. Must know the workerId that handled the capability — reads from `AeEscalationCompletedEvent.workerId()` (to be added alongside `unexpected`) or falls back to `ClinicalCapabilities.SAFETY_MONITORING` as a synthetic workerId if the engine doesn't expose it. **See Open Questions — verify engine writes workerId to case context or event before implementing this observer.**

---

## §2 ClinicalTrustRoutingPolicyProvider

`@ApplicationScoped` implements `TrustRoutingPolicyProvider`.

```java
@Override
public TrustRoutingPolicy forCapability(String capability) {
    return switch (capability) {
        case SAFETY_MONITORING ->
            // Strict: safety-critical, tight margin → Phase 3 escalation near threshold
            new TrustRoutingPolicy(0.75, 20, 0.05, 0.7,
                    Map.of(SAFETY_ACCURACY, 0.70), false);

        case ELIGIBILITY_SCREENING ->
            // Moderate: reversible decision, wider margin acceptable
            new TrustRoutingPolicy(0.70, 15, 0.10, 0.6,
                    Map.of(ELIGIBILITY_PRECISION, 0.65), false);

        case PROTOCOL_REVIEW ->
            // Conservative: high minimum observations before trust kicks in
            new TrustRoutingPolicy(0.65, 25, 0.08, 0.6,
                    Map.of(PROTOCOL_ADHERENCE, 0.60), false);

        default ->
            TrustRoutingPolicy.DEFAULT;  // availability routing fallback
    };
}
```

**Parameter rationale:**
- `threshold`: minimum trust score to route to this agent for this capability
- `minimumObservations`: Phase 0 → Phase 2 transition point (bootstrap → trust-weighted)
- `borderlineMargin`: score within this distance of threshold → Phase 3 (`EscalateToOversight`)
- `blendFactor`: weight of capability-specific score vs global score
- `qualityFloors`: per-dimension minimums enforced independently of global score
- `bootstrapEscalationRequired=false`: in bootstrap phase, fall back to availability (not escalation)

**Maturity model phases:**
- Phase 0 (`isBootstrap(decisionCount) == true`): `TrustWeightedAgentStrategy` falls back to availability routing — Gastown parity
- Phase 2 (`passesThresholdCheck(score) == true`): trust-weighted selection
- Phase 3 (`isBorderline(score) == true`): `EscalateToOversight` assignment → `casehub.agent.routing.escalation` event → clinical observer can create a human-review WorkItem

---

## §3 AeEscalationCompletedEvent.unexpected

**Existing record** (`api/AeEscalationCompletedEvent.java`) — add one field:

```java
public record AeEscalationCompletedEvent(
    UUID aeId,
    CtcaeGrade grade,
    UUID siteId,
    String safetyReviewOutcome,
    boolean dsmbEscalated,
    Instant completedAt,
    boolean unexpected)   // new — derived from AdverseEvent entity via case context
{}
```

**`AeEscalationListener`** reads `unexpected` from `instance.getCaseContext().getPath("unexpected")` (already propagated by `AeEscalationCaseService` in Layer 8). No DB query needed.

This is a breaking change to the record constructor — all call sites must be updated. Only `AeEscalationListener` constructs the event; existing tests that mock the event must add the `unexpected` argument.

---

## §4 Regulatory-Submission Path

### AdverseEvent entity extension

Two new fields (V112 migration, default datasource):
- `regulatorySubmissionStatus RegulatorySub​missionStatus` (enum: `NONE`, `PENDING`, `FILED`) — default `NONE`
- `regulatorySubmissionCaseId UUID` — nullable

### Case Hub (`runtime/service/ClinicalRegulatorySubmissionCaseHub`)

```java
@ApplicationScoped
public class ClinicalRegulatorySubmissionCaseHub extends YamlCaseHub {
    public ClinicalRegulatorySubmissionCaseHub() { super("clinical/regulatory-submission.yaml"); }
}
```

No Java function worker needed — routes to `regulatory-submission` capability (availability routing in Phase 0; external agent in production).

### YAML (`regulatory-submission.yaml`)

```yaml
dsl: "0.1"
version: "1.0.0"
name: regulatory-submission
namespace: clinical
title: IND Expedited Safety Report Filing

spec:
  capabilities:
    - name: regulatory-submission
      inputSchema: "{ grade: .grade, unexpected: .unexpected, aeId: .aeId }"

  goals:
    - name: submission-complete
      kind: success
      condition: ".submissionFiled != null"

  bindings:
    - name: file-ind-report
      on:
        contextChange:
          filter: ".grade != null and .submissionFiled == null"
      capability: regulatory-submission
```

### Service (`runtime/service/RegulatorySubmissionCaseService`)

Three-phase observer on `AdverseEventReportedEvent` — same pattern as `SusarOversightCaseService`:

- **Phase 1** (`@Transactional`): load `AdverseEvent`; check `grade == GRADE_5 && unexpected`; idempotency guard (`regulatorySubmissionStatus != NONE → return null`); set `regulatorySubmissionStatus = PENDING`; build case context `{aeId, grade, unexpected, siteId, tenantId}`
- **Phase 2**: `regulatorySubmissionCaseHub.startCase(ctx).join()` outside any transaction
- **Phase 3** (`@Transactional`): persist `regulatorySubmissionCaseId`

Initial context keys: `aeId`, `grade`, `unexpected`, `siteId`, `tenantId`.

### Ledger (`runtime/ledger/RegulatorySubmissionLedgerEntry`)

`@DiscriminatorValue("RegulatorySubmission")`, `@Entity`, V2023 migration (qhorus datasource). Fields: `aeId UUID`, `grade String`, `filedAt Instant`. `domainContentBytes()`: `String.join("|", aeId, grade)`.

`RegulatorySubmissionLedgerWriter @ApplicationScoped` — writes entry when case starts (in Phase 1 or Phase 3).

---

## §5 casehub-engine-ledger Dependency

Add to `runtime/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-ledger</artifactId>
</dependency>
```

`casehub-engine-ledger` activates `TrustWeightedAgentStrategy @ApplicationScoped` by classpath presence — no `selected-alternatives` or explicit registration needed. The strategy is discovered by `CaseContextChangedEventHandler` via `@Inject AgentRoutingStrategy agentRoutingStrategy`.

**Test `application.properties`:** No change to `quarkus.arc.exclude-types`. `TrustScoreJob` (nightly EigenTrust computation in `casehub-engine-ledger`) should be excluded in tests: `io.casehub.engine.ledger.scheduler.TrustScoreJob`.

Also add to `quarkus.index-dependency` in test `application.properties`:
```properties
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

---

## §6 Testing Strategy

### Unit tests (no Quarkus)

| Test class | What it tests |
|---|---|
| `ClinicalTrustRoutingPolicyProviderTest` | `forCapability()` returns correct policy per capability; `isBootstrap(19) == true`, `isBootstrap(20) == false` for safety-monitoring |
| `DefaultTrustAttestationStrategyTest` | `onWorkerDecision()` calls `upsert()` with correct args; `CORRECT` increments alpha; `INCORRECT` increments beta; `UNCERTAIN` is no-op (Mockito-mocked repository) |
| `AeOutcomeSignalMappingTest` | Outcome → signal mapping for all three conditions |
| `ClinicalActionTypeTest` | Existing — no change |

### Integration tests (`@QuarkusTest`)

| Test class | What it tests |
|---|---|
| `TrustRoutingPolicyProviderIntegrationTest` | CDI deployment: provider resolves; returns non-null policy for all 8 `ClinicalCapabilities` constants; policy parameters are within valid ranges |
| `RegulatorySubmissionCaseServiceTest` | Grade 5 + unexpected → case starts, `regulatorySubmissionStatus = PENDING`; Grade 4 + unexpected → no case; Grade 5 + expected → no case; idempotency guard prevents double-start |
| `AeEscalationListenerTest` (extend) | `unexpected = true` in entity → `AeEscalationCompletedEvent.unexpected == true`; `unexpected = false` → `unexpected == false` |
| `WorkerDecisionAttestationObserverTest` | Call observer directly; `DefaultTrustAttestationStrategy.onWorkerDecision()` called with correct args |

### Not tested (documented limitation)

Full `TrustWeightedAgentStrategy` end-to-end routing in `@QuarkusTest` — Quartz/CDI limitation means function workers don't execute in tests. Test routing policy + attestation strategy at unit level; trust routing is tested by the engine's own test suite.

---

## §7 Files Created/Modified

### New files

| File | Purpose |
|---|---|
| `api/src/main/java/io/casehub/clinical/api/spi/TrustAttestationStrategy.java` | SPI interface |
| `api/src/main/java/io/casehub/clinical/api/spi/AeOutcomeSignal.java` | Signal enum |
| `runtime/src/main/java/io/casehub/clinical/service/DefaultTrustAttestationStrategy.java` | Default Bayesian Beta impl |
| `runtime/src/main/java/io/casehub/clinical/service/WorkerDecisionAttestationObserver.java` | `@ObservesAsync WorkerDecisionEvent` → strategy |
| `runtime/src/main/java/io/casehub/clinical/service/AeEscalationAttestationObserver.java` | `@ObservesAsync AeEscalationCompletedEvent` → strategy |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalTrustRoutingPolicyProvider.java` | Per-capability policies |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalRegulatorySubmissionCaseHub.java` | YamlCaseHub subclass |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java` | Three-phase observer |
| `runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java` | Ledger subclass |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java` | Writer bean |
| `runtime/src/main/resources/clinical/regulatory-submission.yaml` | Case definition |
| `runtime/src/main/resources/db/migration/default/V112__ae_regulatory_submission.sql` | Entity fields |
| `runtime/src/main/resources/db/migration/qhorus/V2023__regulatory_submission_ledger_entry.sql` | Join table |

### Modified files

| File | Change |
|---|---|
| `api/src/main/java/io/casehub/clinical/api/AeEscalationCompletedEvent.java` | Add `boolean unexpected` |
| `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java` | Propagate `unexpected` to event |
| `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java` | Add `regulatorySubmissionStatus`, `regulatorySubmissionCaseId` |
| `runtime/pom.xml` | Add `casehub-engine-ledger` |
| `runtime/src/test/resources/application.properties` | Add engine-ledger index-dependency; exclude `TrustScoreJob` |

---

## Compliance Gaps Closed

| Gap | Closed by |
|---|---|
| Agents selected by availability only — no trust differentiation | `ClinicalTrustRoutingPolicyProvider` + `TrustWeightedAgentStrategy` activated |
| Trust scores never improve — no attestation mechanism | `TrustAttestationStrategy` SPI + Bayesian Beta observation + quality signals |
| Grade 5 unexpected AE — no regulatory submission obligation tracked | `RegulatorySubmissionCaseService` concurrent with AE escalation |
| `AeEscalationCompletedEvent` missing `unexpected` qualifier | API extension, derived from entity |

---

## Open Questions / Deferred

- `AeEscalationAttestationObserver` needs `workerId` from case context — verify engine writes it reliably; if not, fall back to `workerId = capabilityTag` as a placeholder
- `TrustScoreJob` exclusion: verify the class name in `casehub-engine-ledger` SNAPSHOT before adding to `exclude-types`
- Phase 3 (`EscalateToOversight`) handling: a clinical `AgentRoutingEscalationHandler` observer is needed to create a human-review WorkItem when routing cannot select a trusted agent — defer to a follow-up issue if not in scope for this branch
