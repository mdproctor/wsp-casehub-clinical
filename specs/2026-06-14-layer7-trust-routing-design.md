# Layer 7 — Trust-Weighted Safety Agent Routing

**Date:** 2026-06-14 (revised 2026-06-15)
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
                                          ↕ TrustScoreCache (engine-ledger, @Startup)
                                          ↕ ActorTrustScoreRepository (ledger)

   WorkerDecisionEventCapture (engine-ledger, @ObservesAsync WorkerDecisionEvent)
     → writes WorkerDecisionEntry to ledger (subjectId = caseId, actorId = workerId)

   TrustAttestationStrategy SPI  (clinical api/)
   AeEscalationAttestationObserver (@ObservesAsync AeEscalationCompletedEvent)
     → deriveVerdict(event) → ENDORSED | CHALLENGED | empty
     → findWorkerDecisionsByCaseId(engineCaseId) → WorkerDecisionEntry
     → saveAttestation(LedgerAttestation anchored to WorkerDecisionEntry)

   TrustScoreJob (casehub-ledger, 24h schedule, gated by config)
     → reads all WorkerDecisionEntry records grouped by actorId
     → reads LedgerAttestation records for those entries
     → PerActorTrustComputer.computeForActor() → Bayesian Beta scores
     → writes ActorTrustScoreRepository
     → publishes TrustScoreFullPayload → TrustScoreCache.onFull() → routing uses updated scores

2. Regulatory-Submission Path
   AdverseEventReportedEvent  →  RegulatorySubmissionCaseService  →  startCase()
                                  ClinicalRegulatorySubmissionCaseHub
                                  regulatory-submission.yaml

3. API Extension
   AeEscalationCompletedEvent  ← boolean unexpected (from AdverseEvent entity, via case context)
```

**Batch trust model:** Attestations accumulate in `ledger_attestation` between `TrustScoreJob` cycles. Routing quality improves on the next cycle (default: 24h, configurable via `casehub.ledger.trust-score.schedule`). This is by design — the batch model is TrustScoreJob's architecture, not a clinical choice. Routing in Phase 0 (bootstrap) falls back to availability during this period.

**New dependency:** `casehub-engine-ledger` — activates `TrustWeightedAgentStrategy`, `WorkerDecisionEventCapture`, `TrustScoreCache`, and `CaseLedgerEntryRepository` by classpath presence (GE-20260602-c68651).

---

## §1 TrustAttestationStrategy SPI

### The Correct Attestation Mechanism

`WorkerDecisionEventCapture @ApplicationScoped @ObservesAsync WorkerDecisionEvent` (in `casehub-engine-ledger`) already writes a `WorkerDecisionEntry` (a `LedgerEntry` subclass) to the ledger for every worker decision. `TrustScoreJob` reads these entries grouped by `actorId`, loads `LedgerAttestation` records anchored to them, and passes both to `PerActorTrustComputer.computeForActor()`. That is the scoring pipeline.

Writing directly to `ActorTrustScoreRepository` from clinical code is wrong: `TrustScoreJob` calls `trustRepo.upsert()` for every actor on its next cycle, overwriting any direct writes. The correct hook is `LedgerAttestation` — anchored to a specific `WorkerDecisionEntry` via `ledgerEntryId`.

### Interface (`api/spi/TrustAttestationStrategy.java`)

Pure Java, no framework dependencies. Single method — only quality signals are clinical's responsibility; observation counting is handled by `WorkerDecisionEventCapture`.

```java
public interface TrustAttestationStrategy {
    /**
     * Derives a verdict from the outcome of an AE escalation case.
     * Returns empty to skip attestation (uncertain outcome — no quality signal).
     */
    Optional<AttestationVerdict> deriveVerdict(AeEscalationCompletedEvent event);
}
```

CDI displacement contract: `DefaultTrustAttestationStrategy @DefaultBean @ApplicationScoped`; a future ML-based strategy is `@ApplicationScoped` (no `@DefaultBean`) and displaces the default automatically.

### DefaultTrustAttestationStrategy (`runtime/service/`)

```java
@DefaultBean @ApplicationScoped
public class DefaultTrustAttestationStrategy implements TrustAttestationStrategy {
    @Override
    public Optional<AttestationVerdict> deriveVerdict(AeEscalationCompletedEvent event) {
        // SUSAR gate approved + DSMB path followed when required → decision was sound
        if (event.dsmbEscalated() && "ESCALATED".equals(event.safetyReviewOutcome())) {
            return Optional.of(AttestationVerdict.ENDORSED);
        }
        // Gate rejected or expired → agent's SUSAR criteria assessment was challenged
        if ("REJECTED".equals(event.safetyReviewOutcome())
                || "EXPIRED".equals(event.safetyReviewOutcome())) {
            return Optional.of(AttestationVerdict.CHALLENGED);
        }
        // No quality information available — uncertain outcome, skip
        return Optional.empty();
    }
}
```

### AeEscalationAttestationObserver (`runtime/service/`)

`@ApplicationScoped @ObservesAsync AeEscalationCompletedEvent`:

1. Call `strategy.deriveVerdict(event)` — if empty, return immediately
2. Load `AdverseEvent` by `event.aeId()` → get `engineCaseId` and `tenantId` (one `@Transactional` read)
3. Call `caseLedgerEntryRepository.findWorkerDecisionsByCaseId(engineCaseId)` — filter by `capabilityTag == "safety-monitoring"` — get the specific `WorkerDecisionEntry`
4. If no entry found (case not yet recorded or wrong capability), log WARN and return — no attestation
5. Create `LedgerAttestation` (runtime entity):
   - `id = UUID.randomUUID()`
   - `ledgerEntryId = workerDecisionEntry.id` — anchors attestation to the specific decision
   - `subjectId = event.aeId()`
   - `attestorId = ClinicalActors.CLINICAL_SERVICE`
   - `attestorType = ActorType.SYSTEM`
   - `attestorRole = "safety-outcome-reviewer"`
   - `verdict = verdict` (ENDORSED or CHALLENGED)
   - `capabilityTag = ClinicalCapabilities.SAFETY_MONITORING`
   - `trustDimension = ClinicalTrustDimensions.SAFETY_ACCURACY`
   - `confidence = 1.0`
   - `occurredAt = event.completedAt()`
6. Call `ledgerEntryRepository.saveAttestation(attestation, tenantId)`

The `ledger_attestation` table is created by `V1000__ledger_base_schema.sql` in `casehub-ledger` — already exists, no migration needed.

---

## §2 ClinicalTrustRoutingPolicyProvider

`@ApplicationScoped` implements `TrustRoutingPolicyProvider`.

**CDI displacement:** `DefaultTrustRoutingPolicyProvider` (in `casehub-engine-ledger`) is `@DefaultBean @ApplicationScoped`. `ClinicalTrustRoutingPolicyProvider` needs only `@ApplicationScoped` — no `@Alternative`, no `selected-alternatives` entry, no `@Priority`. The `@DefaultBean` mechanism yields automatically to any non-default implementation.

```java
@ApplicationScoped
public class ClinicalTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {

    @Override
    public TrustRoutingPolicy forCapability(String capability) {
        return switch (capability) {
            case SAFETY_MONITORING ->
                // Strict: safety-critical, tight borderline margin → Phase 3 escalation near threshold
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
                TrustRoutingPolicy.DEFAULT;  // availability routing fallback for other capabilities
        };
    }
}
```

**Parameter rationale:**
- `threshold`: minimum trust score to route to this agent for this capability
- `minimumObservations`: Phase 0 → Phase 2 transition (bootstrap → trust-weighted)
- `borderlineMargin`: score within this distance of threshold → Phase 3 (`EscalateToOversight`)
- `blendFactor`: weight of capability score vs global score
- `qualityFloors`: per-dimension minimums — an agent with high global trust but poor safety-accuracy is still blocked
- `bootstrapEscalationRequired = false`: Phase 0 falls back to availability, not human escalation

**Maturity model phases:**
- Phase 0 (`isBootstrap(decisionCount) == true`): `TrustWeightedAgentStrategy` falls back to availability routing — Gastown parity, no trust data needed
- Phase 2 (`passesThresholdCheck(score) == true`): trust-weighted selection
- Phase 3 (`isBorderline(score) == true`): `EscalateToOversight` → `casehub.agent.routing.escalation` event

---

## §3 AeEscalationCompletedEvent.unexpected

Add one field — derived from case context at fire time:

```java
public record AeEscalationCompletedEvent(
    UUID aeId,
    CtcaeGrade grade,
    UUID siteId,
    String safetyReviewOutcome,
    boolean dsmbEscalated,
    Instant completedAt,
    boolean unexpected)   // new — read from case context "unexpected" key (propagated in Layer 8)
{}
```

`AeEscalationListener` reads `unexpected` from `instance.getCaseContext().getPath("unexpected")` — already in case context from Layer 8. No DB read needed; default to `false` if absent.

This is a **breaking change to the record constructor** — `AeEscalationListener` is the only place that constructs the event; all test mocks must add `unexpected` argument.

---

## §4 Regulatory-Submission Path

### AdverseEvent entity extension (V112, default datasource)

Two new fields:
- `regulatorySubmissionStatus RegulatorySubmissionStatus` (enum: `NONE`, `PENDING`, `FILED`) — default `NONE`
- `regulatorySubmissionCaseId UUID` — nullable

### Case Hub + Service

**`ClinicalRegulatorySubmissionCaseHub extends YamlCaseHub`** — no Java function worker; routes to `regulatory-submission` capability (availability routing in Phase 0 showcase).

**`regulatory-submission.yaml`:**
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

**`RegulatorySubmissionCaseService`** — three-phase `@ObservesAsync AdverseEventReportedEvent` (concurrent with `AeEscalationCaseService`, same trigger pattern as `SusarOversightCaseService`):

- **Phase 1** (`@Transactional`): load `AdverseEvent`; check `grade == GRADE_5 && unexpected`; idempotency guard (`regulatorySubmissionStatus != NONE → return null`); set `regulatorySubmissionStatus = PENDING`; write `RegulatorySubmissionLedgerEntry` (in same TX); build case context `{aeId, grade, unexpected, siteId, tenantId}`
- **Phase 2**: `caseHub.startCase(ctx).join()` outside any transaction
- **Phase 3** (`@Transactional`): persist `regulatorySubmissionCaseId`

**Ledger write in Phase 1** (same transaction as status update): the regulatory filing obligation is established when the check passes and `PENDING` is set. `RegulatorySubmissionLedgerEntry` has `aeId`, `grade`, `filedAt` — `domainContentBytes()`: `String.join("|", aeId, grade)`. V2023 migration (qhorus datasource).

### Concurrent observer chain for Grade 5 + unexpected AE:
```
AdverseEventReportedEvent
  ├── AeEscalationCaseService      → clinical review (senior monitor + DSMB)
  ├── SusarOversightCaseService    → SUSAR criteria gate
  └── RegulatorySubmissionCaseService → IND 7-day reporting obligation (new)
```

---

## §5 casehub-engine-ledger Dependency

### Add to runtime/pom.xml

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-ledger</artifactId>
</dependency>
```

**What activates by classpath presence:**
- `TrustWeightedAgentStrategy @ApplicationScoped @Priority(...)` — displaces default availability routing
- `WorkerDecisionEventCapture @ApplicationScoped @ObservesAsync WorkerDecisionEvent` — writes `WorkerDecisionEntry` to ledger
- `TrustScoreCache @Startup @ApplicationScoped` — auto-hydrates from `ActorTrustScoreRepository` at startup; refreshes via `TrustScoreFullPayload` / `TrustScoreDeltaPayload` events from `TrustScoreJob`
- `DefaultTrustRoutingPolicyProvider @DefaultBean @ApplicationScoped` — displaced by `ClinicalTrustRoutingPolicyProvider`
- `CaseLedgerEntryRepository @ApplicationScoped @Transactional` — needed by `AeEscalationAttestationObserver`

**JPA packages:** `WorkerDecisionEntry` and `CaseLedgerEntry` are in `io.casehub.ledger.model` — **already listed** in `quarkus.hibernate-orm.qhorus.packages`. No change needed.

### Flyway migrations (REQUIRED)

`casehub-engine-ledger` ships Flyway migrations at `classpath:db/engine-ledger/migration`:
- `V2000__case_ledger_entry.sql`
- `V2001__worker_decision_entry.sql`

These create qhorus-datasource tables for `CaseLedgerEntry` and `WorkerDecisionEntry`. Without them, `WorkerDecisionEventCapture` fails at startup.

**Production `application.properties`:**
```properties
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus,classpath:db/engine-ledger/migration
```

**Test `application.properties`:**
```properties
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus,classpath:db/engine-ledger/migration
```

### Index dependencies (test application.properties)

```properties
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

### TrustScoreJob — no exclusion needed

`TrustScoreJob` is in `casehub-ledger` (not `casehub-engine-ledger`). It is gated by two config flags — both default to `false`:
```properties
# These are NOT set in clinical by default — TrustScoreJob.computeTrustScores() is a no-op
# casehub.ledger.trust-score.enabled=false
# casehub.ledger.trust-score.materialization.enabled=false
```

For the showcase (and production deployment), enable both to activate the 24h trust score cycle. No exclusion needed in tests — the job is a no-op without the config flags.

### Batch model latency

`TrustScoreCache` is hydrated at startup from `ActorTrustScoreRepository` and refreshed only when `TrustScoreJob` completes (publishing `TrustScoreFullPayload`). Default job interval: 24h. This means:
- An AE outcome attestation written at time T is ingested by the next `TrustScoreJob` cycle
- The routing cache reflects the updated scores after that cycle
- Between T and T+24h, routing uses the previous scores

This is the designed batch model — not a deficiency. For the showcase, enable `TrustScoreJob` and run `runComputation()` directly in integration tests if immediate score updates are needed.

---

## §6 Testing Strategy

### Unit tests (no Quarkus)

| Test class | What it tests |
|---|---|
| `ClinicalTrustRoutingPolicyProviderTest` | `forCapability()` returns correct policy for SAFETY_MONITORING, ELIGIBILITY_SCREENING, PROTOCOL_REVIEW; returns `TrustRoutingPolicy.DEFAULT` (non-null) for unconfigured capabilities; `isBootstrap(19)==true`, `isBootstrap(20)==false` for safety-monitoring |
| `DefaultTrustAttestationStrategyTest` | `dsmbEscalated + ESCALATED → ENDORSED`; `REJECTED → CHALLENGED`; `EXPIRED → CHALLENGED`; other outcomes → empty (Mockito not needed — pure logic) |

### Integration tests (`@QuarkusTest`)

| Test class | What it tests |
|---|---|
| `TrustRoutingPolicyProviderIntegrationTest` | CDI deployment: `ClinicalTrustRoutingPolicyProvider` resolves; policies non-null for all 8 capabilities; `DEFAULT` returned for unconfigured capabilities (not custom policy) |
| `RegulatorySubmissionCaseServiceTest` | Grade 5 + unexpected → case starts, `regulatorySubmissionStatus = PENDING`, ledger entry written; Grade 4 + unexpected → no case; Grade 5 + expected → no case; idempotency guard prevents double-start |
| `AeEscalationListenerTest` (extend) | `unexpected = true` in case context → `AeEscalationCompletedEvent.unexpected == true`; `unexpected = false` → `false` |
| `AeEscalationAttestationObserverTest` | Call observer directly with mocked `TrustAttestationStrategy` returning ENDORSED; assert `LedgerEntryRepository.saveAttestation()` called with correct `ledgerEntryId`, `verdict`, `capabilityTag`, `trustDimension`; strategy returning empty → `saveAttestation` not called |

### Not tested (documented limitation)

Full `TrustWeightedAgentStrategy` end-to-end routing in `@QuarkusTest` — Quartz function worker execution is unreliable in tests (see CLAUDE.md). Test routing policy + attestation strategy at unit level; engine-ledger integration is tested by the engine-ledger test suite.

---

## §7 Files Created/Modified

### New files

| File | Purpose |
|---|---|
| `api/src/main/java/io/casehub/clinical/api/spi/TrustAttestationStrategy.java` | SPI interface — `Optional<AttestationVerdict> deriveVerdict(event)` |
| `runtime/src/main/java/io/casehub/clinical/service/DefaultTrustAttestationStrategy.java` | Default outcome → verdict mapping (`@DefaultBean @ApplicationScoped`) |
| `runtime/src/main/java/io/casehub/clinical/service/AeEscalationAttestationObserver.java` | `@ObservesAsync AeEscalationCompletedEvent` → write `LedgerAttestation` |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalTrustRoutingPolicyProvider.java` | Per-capability policies (`@ApplicationScoped`, displaces `@DefaultBean`) |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalRegulatorySubmissionCaseHub.java` | `YamlCaseHub` subclass |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java` | Three-phase `@ObservesAsync` observer |
| `runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java` | `@DiscriminatorValue("RegulatorySubmission")`; `aeId`, `grade`, `filedAt`; V2023 |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java` | Writer bean (Phase 1 write) |
| `runtime/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java` | Enum: `NONE, PENDING, FILED` |
| `runtime/src/main/resources/clinical/regulatory-submission.yaml` | Case definition |
| `runtime/src/main/resources/db/migration/default/V112__ae_regulatory_submission.sql` | `regulatory_submission_status`, `regulatory_submission_case_id` on `adverse_event` |
| `runtime/src/main/resources/db/migration/qhorus/V2023__regulatory_submission_ledger_entry.sql` | Join table |

### Modified files

| File | Change |
|---|---|
| `api/src/main/java/io/casehub/clinical/api/AeEscalationCompletedEvent.java` | Add `boolean unexpected` |
| `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java` | Propagate `unexpected` from case context to event |
| `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java` | Add `regulatorySubmissionStatus`, `regulatorySubmissionCaseId` |
| `runtime/pom.xml` | Add `casehub-engine-ledger` |
| `runtime/src/main/resources/application.properties` | Add `classpath:db/engine-ledger/migration` to qhorus Flyway locations |
| `runtime/src/test/resources/application.properties` | Same Flyway change; add `engine-ledger` to `quarkus.index-dependency` |

---

## Compliance Gaps Closed

| Gap | Closed by |
|---|---|
| Agents selected by availability only — no trust differentiation | `ClinicalTrustRoutingPolicyProvider` + `TrustWeightedAgentStrategy` activated |
| Trust scores never improve — no attestation mechanism | `AeEscalationAttestationObserver` writes `LedgerAttestation`; `TrustScoreJob` ingests |
| Grade 5 unexpected AE — no regulatory submission obligation tracked | `RegulatorySubmissionCaseService` concurrent with AE escalation |
| `AeEscalationCompletedEvent` missing `unexpected` qualifier | API extension, derived from case context |
