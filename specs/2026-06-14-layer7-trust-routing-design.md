# Layer 7 — Trust-Weighted Safety Agent Routing

**Date:** 2026-06-14 (revised 2026-06-15 r3)
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
     → writes WorkerDecisionEntry to ledger
       (subjectId = caseId, actorId = workerId, capabilityTag)
       FIRES ONLY for capability-based agent workers — NOT humanTask bindings

   SusarAgentAttestationWriter (@ApplicationScoped)
     → @ConsumeEvent("casehub.action.gate.approved")   → ENDORSED
     → @ConsumeEvent("casehub.action.gate.rejected")   → CHALLENGED
     → @ConsumeEvent("casehub.action.gate.expired")    → CHALLENGED
     Discriminates via AdverseEvent.findBySusarOversightCaseId(event.caseId())
     Finds WorkerDecisionEntry via CaseLedgerEntryRepository.findWorkerDecisionsByCaseId(ae.susarOversightCaseId)
     Writes LedgerAttestation anchored to WorkerDecisionEntry

   TrustScoreJob (casehub-ledger, 24h schedule, gated by config)
     → reads WorkerDecisionEntry records grouped by actorId
     → reads LedgerAttestation for those entries
     → PerActorTrustComputer.computeForActor() → Bayesian Beta scores
     → writes ActorTrustScoreRepository
     → publishes TrustScoreFullPayload → TrustScoreCache.onFull()

2. Regulatory-Submission Path
   AdverseEventReportedEvent  →  RegulatorySubmissionCaseService  →  startCase()
                                  ClinicalRegulatorySubmissionCaseHub
                                  regulatory-submission.yaml

3. API Extension
   AeEscalationCompletedEvent  ← boolean unexpected (from case context, propagated in Layer 8)
```

**Why SUSAR gate events, not AeEscalationCompletedEvent:**
`ae-escalation.yaml` has ONLY `humanTask` bindings (senior safety monitor, DSMB). `WorkerDecisionEventCapture` fires on `WorkerDecisionEvent` — only published after AGENT worker completions, never after human task completions. There are zero `WorkerDecisionEntry` records in the AE escalation case. The safety-monitoring AGENT works in the SUSAR oversight case (`susar-oversight.yaml`, `capability: safety-monitoring`). The SUSAR gate outcome (APPROVED/REJECTED/EXPIRED) is the correct quality signal.

**Batch trust model:** Attestations accumulate in `ledger_attestation` between `TrustScoreJob` cycles. Routing quality improves on the next cycle (default: 24h, configurable via `casehub.ledger.trust-score.schedule`). Phase 0 (bootstrap) falls back to availability during this period.

**New dependency:** `casehub-engine-ledger` — activates `TrustWeightedAgentStrategy`, `WorkerDecisionEventCapture`, `TrustScoreCache`, and `CaseLedgerEntryRepository` by classpath presence (GE-20260602-c68651).

---

## §1 SusarAgentAttestationWriter

No SPI layer. The attestation logic is two lines: `APPROVED → ENDORSED`, `REJECTED/EXPIRED → CHALLENGED`. A future ML-based attestation strategy displaces the whole writer via `@DefaultBean` — same displacement contract as `SusarCriteriaEvaluator`, without introducing an interface that adds no architectural value.

### SusarAgentAttestationWriter (`runtime/service/`)

`@ApplicationScoped` — plain, no `@DefaultBean`. `@DefaultBean` displacement requires a shared interface type; without one, a future competing class would be a different type and both would register as separate `@ConsumeEvent` consumers (no CDI ambiguity to resolve, no displacement). Future replacement via `quarkus.arc.exclude-types` if needed.

Three `@ConsumeEvent` methods, same three gate addresses as `SusarGateDecisionListener`. Same discrimination: `AdverseEvent.findBySusarOversightCaseId(event.caseId())`.

`attestorType` follows the `SusarDecisionLedgerWriter` pattern exactly (line 44: `decidedBy != null ? ActorType.HUMAN : ActorType.SYSTEM`): HUMAN when a named human actor approved/rejected, SYSTEM when the service itself acts (expiry). This matters for EigenTrust — `TrustScoreJob.runEigenTrustPass()` builds a social trust graph from attestations keyed by attestorId and type; recording a human clinical reviewer as SYSTEM misrepresents the social trust signal.

```java
@ApplicationScoped
public class SusarAgentAttestationWriter {

    private static final Logger LOG = Logger.getLogger(SusarAgentAttestationWriter.class);

    @Inject CaseLedgerEntryRepository caseLedgerEntryRepository;
    @Inject LedgerEntryRepository ledgerEntryRepository;

    @ConsumeEvent(value = "casehub.action.gate.approved", blocking = true)
    @Transactional
    public void onApproved(ActionGateApprovedEvent event) {
        writeAttestation(event.caseId(), AttestationVerdict.ENDORSED, event.approvedBy(), Instant.now());
    }

    @ConsumeEvent(value = "casehub.action.gate.rejected", blocking = true)
    @Transactional
    public void onRejected(ActionGateRejectedEvent event) {
        writeAttestation(event.caseId(), AttestationVerdict.CHALLENGED, event.rejectedBy(), Instant.now());
    }

    @ConsumeEvent(value = "casehub.action.gate.expired", blocking = true)
    @Transactional
    public void onExpired(ActionGateExpiredEvent event) {
        writeAttestation(event.caseId(), AttestationVerdict.CHALLENGED, ClinicalActors.CLINICAL_SERVICE, Instant.now());
    }

    private void writeAttestation(UUID caseId, AttestationVerdict verdict, String attestorId, Instant now) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(caseId);
        if (ae == null) return; // not a SUSAR oversight gate

        caseLedgerEntryRepository.findWorkerDecisionsByCaseId(ae.susarOversightCaseId)
                .stream()
                .filter(e -> ClinicalCapabilities.SAFETY_MONITORING.equals(e.capabilityTag))
                .findFirst()
                .ifPresentOrElse(
                        entry -> {
                            LedgerAttestation attestation = new LedgerAttestation();
                            attestation.id = UUID.randomUUID();
                            attestation.ledgerEntryId = entry.id;
                            attestation.subjectId = ae.susarOversightCaseId;
                            attestation.attestorId = attestorId != null ? attestorId : ClinicalActors.CLINICAL_SERVICE;
                            // Mirror SusarDecisionLedgerWriter pattern: HUMAN when named actor present, SYSTEM otherwise
                            attestation.attestorType = ClinicalActors.CLINICAL_SERVICE.equals(attestorId) || attestorId == null
                                    ? ActorType.SYSTEM : ActorType.HUMAN;
                            attestation.attestorRole = "safety-gate-outcome";
                            attestation.verdict = verdict;
                            attestation.capabilityTag = ClinicalCapabilities.SAFETY_MONITORING;
                            attestation.trustDimension = ClinicalTrustDimensions.SAFETY_ACCURACY;
                            attestation.confidence = 1.0;
                            attestation.occurredAt = now;
                            ledgerEntryRepository.saveAttestation(attestation, ae.tenantId);
                        },
                        () -> LOG.warnf("SusarAgentAttestationWriter: no WorkerDecisionEntry for caseId=%s — attestation skipped", caseId)
                );
    }
}
```

**Test isolation note:** `SusarAgentAttestationWriter` fires in any `@QuarkusTest` that triggers SUSAR gate events. The null-AE path returns immediately; the absent-WorkerDecisionEntry path logs WARN and returns. No test failures expected; do NOT add this to `quarkus.arc.exclude-types`.

**Why gate events, not AeEscalationCompletedEvent:**
- The safety-monitoring agent runs in `susar-oversight.yaml` via `capability: safety-monitoring` binding
- `WorkerDecisionEventCapture` fires for this worker and writes `WorkerDecisionEntry` with `subjectId = ae.susarOversightCaseId`
- The gate APPROVED/REJECTED/EXPIRED outcome IS the quality signal for the agent's decision
- `SusarGateDecisionListener` already has the same discriminator (`findBySusarOversightCaseId`) — this writer follows the same pattern

**`LedgerAttestation.subjectId = ae.susarOversightCaseId`** (the case the decision belongs to, not the AE entity ID — consistent with `WorkerDecisionEntry.subjectId`).

**`ledger_attestation` table** is created by `V1000__ledger_base_schema.sql` in `casehub-ledger` — already exists, no migration needed.

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
                // Strict: safety-critical, tight borderline margin → Phase 3 near threshold
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
                TrustRoutingPolicy.DEFAULT;  // availability routing for other capabilities
        };
    }
}
```

**Maturity model phases:**
- Phase 0 (`isBootstrap(decisionCount) == true`): availability routing — Gastown parity, no trust data needed
- Phase 2 (`passesThresholdCheck(score) == true`): trust-weighted selection
- Phase 3 (`isBorderline(score) == true`): `EscalateToOversight` → `casehub.agent.routing.escalation` event

---

## §3 AeEscalationCompletedEvent.unexpected

Add one field — derived from case context at fire time. This makes the event complete: `unexpected` is a material fact about the AE that belongs in the completion event regardless of which Layer 7 component consumes it now.

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

`AeEscalationListener` reads `unexpected` from `instance.getCaseContext().getPath("unexpected")` — already in case context from Layer 8; default to `false` if absent.

**Breaking change:** All call sites that construct `AeEscalationCompletedEvent` must add `unexpected` argument. Known call sites:
- `AeEscalationListener` (production constructor)
- `AeEscalationListenerTest` — mock construction, add `false` default
- `TrialSafetySignalServiceTest` (line 90: `new AeEscalationCompletedEvent(UUID.randomUUID(), grade, siteId, "REVIEWED", true, Instant.now())`) — add `false`
- `AeEscalationListenerMemoryTest` — check for captors

---

## §4 Regulatory-Submission Path

### AdverseEvent entity extension (V112, default datasource)

Two new fields:
- `regulatorySubmissionStatus RegulatorySubmissionStatus` (enum: `NONE`, `PENDING`, `FILED`) — default `NONE`
- `regulatorySubmissionCaseId UUID` — nullable

`RegulatorySubmissionStatus` lives in `api/src/main/java/io/casehub/clinical/api/model/` — consistent with `SusarOversightStatus`, `AeEscalationStatus`, and all other status enums.

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

**`RegulatorySubmissionCaseService`** — three-phase `@ObservesAsync AdverseEventReportedEvent` (concurrent with `AeEscalationCaseService` and `SusarOversightCaseService`):

- **Phase 1** (`@Transactional`): load `AdverseEvent`; check `grade == GRADE_5 && unexpected`; idempotency guard (`regulatorySubmissionStatus != NONE → return null`); set `regulatorySubmissionStatus = PENDING`; write `RegulatorySubmissionLedgerEntry` (in same TX); build case context `{aeId, grade, unexpected, siteId, tenantId}`
- **Phase 2**: `caseHub.startCase(ctx).join()` outside any transaction
- **Phase 3** (`@Transactional`): persist `regulatorySubmissionCaseId`

**Ledger write in Phase 1** — same transaction as the status update. `RegulatorySubmissionLedgerEntry` has `aeId`, `grade`, `filedAt`. `domainContentBytes()`: `String.join("|", aeId, grade)`. V2023 migration (qhorus datasource).

`RegulatorySubmissionLedgerWriter` must call `entry.attach(ClinicalComplianceSupplement.regulatorySubmission())` before `ledgerEntryRepository.save()`. The `LedgerProcessor` build-time validator requires this on any subclass with `@Column` fields — CDI deployment fails without it. Add `regulatorySubmission()` to `ClinicalComplianceSupplement`:

```java
public static ComplianceSupplement regulatorySubmission() {
    ComplianceSupplement s = new ComplianceSupplement();
    s.planRef = "21 CFR 312.32(c)(1)(i) — IND expedited safety reporting, unexpected fatal/life-threatening AE";
    s.algorithmRef = "RegulatorySubmissionCaseService (rule-based Grade 5 + unexpected criteria)";
    s.humanOverrideAvailable = true;
    return s;
}
```

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
- `WorkerDecisionEventCapture @ApplicationScoped @ObservesAsync WorkerDecisionEvent` — writes `WorkerDecisionEntry` to ledger for every agent worker decision
- `TrustScoreCache @Startup @ApplicationScoped @PostConstruct` — auto-hydrates at startup; refreshes via `TrustScoreFullPayload` / `TrustScoreDeltaPayload` events
- `DefaultTrustRoutingPolicyProvider @DefaultBean @ApplicationScoped` — displaced by `ClinicalTrustRoutingPolicyProvider`
- `CaseLedgerEntryRepository @ApplicationScoped @Transactional` — needed by `SusarAgentAttestationWriter`; uses `@LedgerPersistenceUnit` EntityManager (qhorus datasource)

**JPA packages:** `WorkerDecisionEntry` and `CaseLedgerEntry` are in `io.casehub.ledger.model` — **already listed** in `quarkus.hibernate-orm.qhorus.packages`. No change needed.

### Flyway migrations (REQUIRED)

`casehub-engine-ledger` ships Flyway migrations at `classpath:db/engine-ledger/migration`:
- `V2000__case_ledger_entry.sql`
- `V2001__worker_decision_entry.sql`

These create `WorkerDecisionEntry` and `CaseLedgerEntry` tables on the qhorus datasource. Without them, `WorkerDecisionEventCapture` fails at startup.

**Both** production and test `application.properties`:
```properties
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus,classpath:db/engine-ledger/migration
```

### Index dependencies (test application.properties)

```properties
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

### TrustScoreJob — no exclusion needed

`TrustScoreJob` is in `casehub-ledger` (not `casehub-engine-ledger`). Gated by two config flags — both default to `false`:
```properties
# Set both true in production to activate 24h trust score cycle
# casehub.ledger.trust-score.enabled=true
# casehub.ledger.trust-score.materialization.enabled=true
```

### Batch model latency

`TrustScoreCache` hydrates at startup from `ActorTrustScoreRepository` and refreshes only after `TrustScoreJob` completes (24h default). Attestations written between cycles do not affect routing until the next cycle. This is by design. For integration tests where immediate score updates are needed, call `TrustScoreJob.runComputation()` directly.

---

## §6 Testing Strategy

### Unit tests (no Quarkus)

| Test class | What it tests |
|---|---|
| `ClinicalTrustRoutingPolicyProviderTest` | `forCapability()` returns correct policy for SAFETY_MONITORING, ELIGIBILITY_SCREENING, PROTOCOL_REVIEW; returns `TrustRoutingPolicy.DEFAULT` (non-null) for unconfigured capabilities; `isBootstrap(19)==true`, `isBootstrap(20)==false` for safety-monitoring |
| `SusarAgentAttestationWriterTest` | Mocked `CaseLedgerEntryRepository` + `LedgerEntryRepository`; approved gate → `saveAttestation()` called with `verdict=ENDORSED`, `capabilityTag=safety-monitoring`, `trustDimension=safety-accuracy`; rejected gate → `CHALLENGED`; null AE (not SUSAR gate) → `saveAttestation()` not called; no WorkerDecisionEntry → WARN logged, no attestation |

### Integration tests (`@QuarkusTest`)

| Test class | What it tests |
|---|---|
| `TrustRoutingPolicyProviderIntegrationTest` | CDI deployment: `ClinicalTrustRoutingPolicyProvider` resolves; returns non-null policies; `DEFAULT` returned for unconfigured capabilities |
| `RegulatorySubmissionCaseServiceTest` | Grade 5 + unexpected → case starts, `regulatorySubmissionStatus = PENDING`, ledger entry written; Grade 4 + unexpected → no case; Grade 5 + expected → no case; idempotency guard |
| `AeEscalationListenerTest` (extend) | `unexpected = true` in case context → `AeEscalationCompletedEvent.unexpected == true`; `unexpected = false` → `false`; existing tests updated to add `unexpected` argument |

### Not tested (documented limitation)

Full `TrustWeightedAgentStrategy` end-to-end routing in `@QuarkusTest` — Quartz function worker execution unreliable in tests. Test routing policy at unit level; engine-ledger integration tested by engine-ledger's own suite.

---

## §7 Files Created/Modified

### New files

| File | Purpose |
|---|---|
| `runtime/src/main/java/io/casehub/clinical/service/SusarAgentAttestationWriter.java` | `@DefaultBean @ApplicationScoped`; observes 3 gate events; writes `LedgerAttestation` anchored to `WorkerDecisionEntry` in SUSAR oversight case |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalTrustRoutingPolicyProvider.java` | Per-capability policies (`@ApplicationScoped`, displaces `@DefaultBean`) |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalRegulatorySubmissionCaseHub.java` | `YamlCaseHub` subclass |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCaseService.java` | Three-phase `@ObservesAsync` observer |
| `runtime/src/main/java/io/casehub/clinical/ledger/RegulatorySubmissionLedgerEntry.java` | `@DiscriminatorValue("RegulatorySubmission")`; `aeId`, `grade`, `filedAt`; V2023 |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java` | Writer bean (Phase 1 write) |
| `api/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java` | Enum: `NONE, PENDING, FILED` — in `api/` consistent with `SusarOversightStatus`, `AeEscalationStatus` |
| `runtime/src/main/resources/clinical/regulatory-submission.yaml` | Case definition |
| `runtime/src/main/resources/db/migration/default/V112__ae_regulatory_submission.sql` | `regulatory_submission_status`, `regulatory_submission_case_id` on `adverse_event` |
| `runtime/src/main/resources/db/migration/qhorus/V2023__regulatory_submission_ledger_entry.sql` | Join table |

### Modified files

| File | Change |
|---|---|
| `api/src/main/java/io/casehub/clinical/api/AeEscalationCompletedEvent.java` | Add `boolean unexpected` (7th field) |
| `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java` | Propagate `unexpected` from case context to event |
| `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java` | Add `regulatorySubmissionStatus`, `regulatorySubmissionCaseId` |
| `runtime/pom.xml` | Add `casehub-engine-ledger` |
| `runtime/src/main/resources/application.properties` | Add `classpath:db/engine-ledger/migration` to qhorus Flyway locations |
| `runtime/src/test/resources/application.properties` | Same Flyway change; add `engine-ledger` to `quarkus.index-dependency` |
| `runtime/src/test/java/...AeEscalationListenerTest.java` | Add `unexpected` arg to event constructors |
| `runtime/src/test/java/...TrialSafetySignalServiceTest.java` | Add `unexpected` arg to event constructors (line 90) |
| `runtime/src/test/java/...AeEscalationListenerMemoryTest.java` | Add `unexpected` arg to any event construction |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java` | Add `regulatorySubmission()` factory method (21 CFR 312.32(c)(1)(i)) |

---

## Compliance Gaps Closed

| Gap | Closed by |
|---|---|
| Agents selected by availability only — no trust differentiation | `ClinicalTrustRoutingPolicyProvider` + `TrustWeightedAgentStrategy` activated |
| Trust scores never improve — no quality attestation | `SusarAgentAttestationWriter` writes `LedgerAttestation` on SUSAR gate outcomes; `TrustScoreJob` ingests |
| Grade 5 unexpected AE — no regulatory submission obligation tracked | `RegulatorySubmissionCaseService` concurrent with AE escalation |
| `AeEscalationCompletedEvent` missing `unexpected` qualifier | API extension makes event complete |
