# Layer 5 Design — casehub-engine: IRB Gate and AE Escalation
**Branch:** `issue-6-irb-gate` · **Issue:** casehubio/clinical#6 · **Date:** 2026-05-22

---

## Goal

Introduce `casehub-engine` to clinical for the first time (Layer 5). Replace fixed-pipeline service code with engine case definitions that open different gates based on accumulated context. Two compliance gaps closed:

- **IRB gate** — CRITICAL deviation + PI approval suspends the case in WAITING until an IRB committee decides within 72h (GCP/ICH E6(R3) §5.17, 21 CFR Part 312).
- **AE severity routing** — Grade 3+ adverse event routes to senior safety monitor; Grade 4+ additionally opens a DSMB escalation gate in parallel. Routing is SPI-driven, not hardcoded.

---

## Architecture

### Two case flows

```
CRITICAL DEVIATION FLOW (IRB gate)
─────────────────────────────────────────────────────────────────
[Layer 3]  ProtocolDeviationService → qhorus COMMAND → PI
[Layer 3]  PiResponseListener fires ProtocolDeviationResolvedEvent(IRB_REVIEW, APPROVED)
[Layer 5]  IrbDeviationCaseService @ObservesAsync → creates IrbApproval(PENDING) → startCase()
[engine]   deviation-review.yaml: irb-consultation humanTask fires
[adapter]  HumanTaskScheduleHandler → 72h WorkItem → irb-committee group
           IRB committee claims and resolves WorkItem
[adapter]  WorkItemLifecycleAdapter → CONTEXT_CHANGED → outputMapping writes irbConsultation
[Layer 5]  IrbDecisionListener @ObservesAsync WorkItemLifecycleEvent
           → reads payload.deviationId → updates IrbApproval.decision
           → updates ProtocolDeviation.piApprovalStatus
           → writes IrbApprovalLedgerEntry
           → fires IrbApprovalResolvedEvent
           (EXPIRED path: signals case directly, then same updates)

ADVERSE EVENT ESCALATION FLOW
─────────────────────────────────────────────────────────────────
           AdverseEventService.reportAdverseEvent(ae)
[Layer 5]  calls AdverseEventEscalationPolicy.evaluate(aeContext)
           Grade 1/2 (engineCaseRequired=false):
             → direct WorkItem creation, SPI-provided candidateGroups [Layer 2 preserved]
           Grade 3+ (engineCaseRequired=true):
             → fires AdverseEventReportedEvent (no WorkItem created here)
[Layer 5]  AeEscalationCaseService @ObservesAsync → re-evaluates policy → startCase(aeContext)
[engine]   ae-escalation.yaml:
             safety-review humanTask fires    (requiresSeniorMonitor=true)
             dsmb-escalation humanTask fires  (requiresDsmbEscalation=true, Grade 4+)
[adapter]  HumanTaskScheduleHandler → WorkItem(s) → senior-safety-monitors / dsmb groups
           Reviewers complete WorkItems
[adapter]  WorkItemLifecycleAdapter → CONTEXT_CHANGED → outputMapping per binding
[engine]   Goals satisfied → case COMPLETED → CaseLifecycleEvent(eventType=CaseCompleted)
[Layer 5]  AeEscalationListener @ObservesAsync CaseLifecycleEvent
           → discriminates AE case via CaseContext.aeId
           → writes AeEscalationLedgerEntry
           → fires AeEscalationCompletedEvent
```

### Key structural decisions

- **Single routing decision point**: `AdverseEventEscalationPolicy` governs both Layer 2 `candidateGroups` (Grade 1/2) and Layer 5 case context keys (Grade 3+). No parallel routing systems.
- **`outputMapping` flat pattern** (engine#314 — nested `{..}` unsupported): `"{ irbConsultation: . }"` and `"{ safetyReview: . }"` map the full WorkItem resolution. Avoid nested field extraction in outputMapping.
- **`IrbDecisionListener` handles EXPIRED explicitly**: `WorkItemLifecycleAdapter` calls `markFaulted()` on expired PlanItems — no outputMapping fires. Listener signals the case directly via `caseHub.signal(caseId, "irbConsultation", Map.of("decision", "EXPIRED", ...))`.
- **`AeEscalationListener` observes case completion**: discriminates AE escalation cases from deviation review cases by checking `CaseContext.aeId` — present for AE cases, absent for deviation cases. No external state map needed.
- **`engine_case_id` on domain entities deferred to Layer 6** (clinical#26) — devtown precedent (devtown#10). `IrbDecisionListener` routes via payload `deviationId`; `AeEscalationListener` routes via `CaseContext.aeId`.
- **Grade 5 SLA (1h internal)**: `AeEscalationCaseService` passes `ae.slaDeadline` as case budget deadline; `HumanTaskScheduleHandler` uses `earliestOf(bindingExpiry, caseBudget)` to enforce the tighter 1h deadline without changing the YAML.

---

## Component Inventory

### `api/` module — new and modified

| Component | Kind | Notes |
|---|---|---|
| `AdverseEventEscalationPolicy` | SPI interface | Replaces hardcoded candidateGroups routing |
| `AdverseEventEscalationRequirements` | record | `engineCaseRequired`, `candidateGroups`, `requiresSeniorMonitor`, `requiresDsmbEscalation` |
| `AdverseEventContext` | record | `aeId`, `enrollmentId`, `siteId`, `grade`, `aeType` — mirrors `DeviationContext` |
| `AdverseEventReportedEvent` | CDI event record | `aeId`, `enrollmentId`, `siteId`, `grade`, `reportedAt` |
| `IrbApprovalResolvedEvent` | CDI event record | `approvalId`, `deviationId`, `siteId`, `decision`, `decidedAt` — follows `ProtocolDeviationResolvedEvent` |
| `AeEscalationCompletedEvent` | CDI event record | `aeId`, `grade`, `safetyReviewOutcome`, `dsmbEscalated`, `completedAt` |
| `IrbDecision.EXPIRED` | enum addition | Terminal state when 72h IRB WorkItem expires |

### `runtime/` module — new classes

| Component | Kind | Notes |
|---|---|---|
| `DefaultAdverseEventEscalationPolicy` | `@DefaultBean` | CTCAE grades as org-overridable baseline |
| `ClinicalDeviationCaseHub` | `YamlCaseHub` | Loads `clinical/deviation-review.yaml` |
| `IrbDeviationCaseService` | CDI observer | `@ObservesAsync ProtocolDeviationResolvedEvent`; creates `IrbApproval(PENDING)`; starts case |
| `IrbDecisionListener` | CDI observer | `@ObservesAsync WorkItemLifecycleEvent`; updates `IrbApproval` + `ProtocolDeviation`; ledger + event |
| `IrbApprovalLedgerWriter` | service | Writes `IrbApprovalLedgerEntry` |
| `IrbApprovalLedgerEntry` | ledger subclass | In `io.casehub.clinical.ledger`; JOINED inheritance; FDA tamper-evident IRB record |
| `ClinicalAdverseEventCaseHub` | `YamlCaseHub` | Loads `clinical/ae-escalation.yaml` |
| `AeEscalationCaseService` | CDI observer | `@ObservesAsync AdverseEventReportedEvent`; evaluates policy; starts case |
| `AeEscalationListener` | CDI observer | `@ObservesAsync CaseLifecycleEvent`; discriminates by `CaseContext.aeId`; ledger + event |
| `AeEscalationLedgerWriter` | service | Writes `AeEscalationLedgerEntry` |
| `AeEscalationLedgerEntry` | ledger subclass | In `io.casehub.clinical.ledger`; JOINED inheritance; safety review outcome record |

### `runtime/` module — modified classes

| Component | Change |
|---|---|
| `AdverseEventService` | Inject `AdverseEventEscalationPolicy`; replace hardcoded `candidateGroups()`; Grade 3+ fires `AdverseEventReportedEvent` instead of creating WorkItem; `ae.workItemId` null for Grade 3+ (intentional) |

### Resources

| File | Notes |
|---|---|
| `clinical/deviation-review.yaml` | 1 binding, 1 goal |
| `clinical/ae-escalation.yaml` | 2 bindings, 2 goals |

### Test classes

| Class | Scope | Notes |
|---|---|---|
| `IrbGateLifecycleTest` | `@QuarkusTest` | Full IRB lifecycle: APPROVED, EXPIRED paths |
| `AeEscalationLifecycleTest` | `@QuarkusTest` | Grade 3 (1 gate) and Grade 4 (2 parallel gates) paths |
| `WorkItemCompletionCapture` | test | `@ObservesAsync WorkItemLifecycleEvent` capture; copied from devtown |
| `WorkItemQueries` | test | `WorkItemStore.scanAll()` wrapper; copied from devtown |

---

## Case Definitions

### `clinical/deviation-review.yaml`

```yaml
dsl: "0.1"
version: "1.0.0"
name: deviation-review
namespace: clinical
title: Protocol Deviation Review — IRB consultation gate

spec:
  goals:
    - name: irb-decided
      kind: success
      condition: ".irbConsultation != null"

  completion:
    success:
      allOf: [irb-decided]

  bindings:
    - name: irb-consultation
      on: { contextChange: {} }
      when: ".irbConsultationRequired == true and .irbConsultation == null"
      humanTask:
        title: "IRB consultation required — protocol deviation"
        expiresIn: PT72H
        candidateGroups: [irb-committee]
        inputSchema: "{ deviationId: .deviationId, severity: .severity }"
        outputMapping: "{ irbConsultation: . }"
```

Initial context keys: `deviationId`, `siteId`, `severity`, `escalationRequirement`, `irbConsultationRequired: true`.

`inputSchema` uses the mini-DSL (GE-0167 — not JQ). `outputMapping` is JQ (flat pattern, engine#314).

`irbConsultation` is set to the full WorkItem resolution object. `IrbDecisionListener` reads `resolution.decision` for domain updates. For EXPIRED WorkItems (no outputMapping), listener signals the case manually with `{ irbConsultation: { decision: EXPIRED, ... } }`.

All four terminal outcomes (APPROVED, REJECTED, DEFERRED, EXPIRED) satisfy `.irbConsultation != null` and complete the case. DEFERRED completes the case — re-submission requires a new case instance when additional information is provided.

### `clinical/ae-escalation.yaml`

```yaml
dsl: "0.1"
version: "1.0.0"
name: ae-escalation
namespace: clinical
title: Adverse Event Safety Escalation — adaptive severity routing

spec:
  goals:
    - name: safety-review-complete
      kind: success
      condition: ".safetyReview != null"

    - name: dsmb-complete
      kind: success
      condition: ".requiresDsmbEscalation == false or .dsmbEscalation != null"

  completion:
    success:
      allOf: [safety-review-complete, dsmb-complete]

  bindings:
    - name: safety-review
      on: { contextChange: {} }
      when: ".requiresSeniorMonitor == true and .safetyReview == null"
      humanTask:
        title: "Senior safety monitor review — adverse event"
        expiresIn: PT24H
        candidateGroups: [senior-safety-monitors]
        inputSchema: "{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId }"
        outputMapping: "{ safetyReview: . }"

    - name: dsmb-escalation
      on: { contextChange: {} }
      when: ".requiresDsmbEscalation == true and .dsmbEscalation == null"
      humanTask:
        title: "DSMB escalation — Grade 4+ adverse event"
        expiresIn: PT24H
        candidateGroups: [dsmb]
        inputSchema: "{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId }"
        outputMapping: "{ dsmbEscalation: . }"
```

Initial context keys (set by `AeEscalationCaseService` from SPI result): `aeId`, `enrollmentId`, `grade`, `requiresSeniorMonitor`, `requiresDsmbEscalation`.

Grade 3: one gate opens. Grade 4+: two gates open simultaneously. Same YAML — context drives which bindings fire. This is the adaptive routing the engine provides.

---

## SPI: `AdverseEventEscalationPolicy`

```java
// api/
public interface AdverseEventEscalationPolicy {
    AdverseEventEscalationRequirements evaluate(AdverseEventContext context);
}
```

`DefaultAdverseEventEscalationPolicy` (`@DefaultBean`):

| Grade | `engineCaseRequired` | `candidateGroups` | `requiresSeniorMonitor` | `requiresDsmbEscalation` |
|---|---|---|---|---|
| 1, 2 | false | `safety-officers` | — | — |
| 3 | true | — | true | false |
| 4, 5 | true | — | true | true |

Orgs override the SPI entirely. The SPI is the org flexibility point — thresholds, team assignments, and scope rules are opaque to the engine case definition.

---

## Migrations

### V109 — default datasource (`db/migration/default/`)

```sql
-- Links IrbApproval to its originating deviation (required for IrbDecisionListener lookup)
ALTER TABLE irb_approval
    ADD COLUMN deviation_id UUID REFERENCES protocol_deviation(id);
```

Nullable — existing stub rows have no deviation. All new rows set it at `IrbApproval` creation in `IrbDeviationCaseService`.

### V1009 — qhorus datasource (`db/migration/qhorus/`)

`irb_approval_ledger_entry` join table (JOINED inheritance from `ledger_entry`). Follows the pattern of V1005 (`ae_ledger_entry`) and V1006 (`protocol_deviation_ledger_entry`).

Fields: `id` (FK to `ledger_entry`), `irb_approval_id`, `deviation_id`, `irb_decision`, `committee_id`, `decided_at`.

### V1010 — qhorus datasource (`db/migration/qhorus/`)

`ae_escalation_ledger_entry` join table. Records safety review completion for FDA audit.

Fields: `id` (FK to `ledger_entry`), `ae_id`, `enrollment_id`, `grade`, `safety_review_outcome`, `dsmb_escalated` (boolean), `completed_at`.

---

## Engine Dependencies (`runtime/pom.xml`)

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-scheduler-quartz</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-work-adapter</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-testing</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-persistence-memory</artifactId>
  <scope>test</scope>
</dependency>
```

Engine uses the existing default datasource — no new datasource needed. `JpaPlanItemStore` (from `casehub-engine-work-adapter`) shares the default datasource that already holds casehub-work + clinical domain tables.

---

## Configuration Changes

### Production `application.properties`

```properties
# Add engine JPA store selections alongside existing JpaLedgerEntryRepository
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.persistence.jpa.JpaPlanItemStore,\
  io.casehub.persistence.jpa.JpaSubCaseGroupRepository
```

### Test `application.properties`

```properties
# Index engine external jars — beans not auto-discovered without this
quarkus.index-dependency.engine-testing.group-id=io.casehub
quarkus.index-dependency.engine-testing.artifact-id=casehub-engine-testing
quarkus.index-dependency.engine-scheduler-quartz.group-id=io.casehub
quarkus.index-dependency.engine-scheduler-quartz.artifact-id=casehub-engine-scheduler-quartz
quarkus.index-dependency.engine-work-adapter.group-id=io.casehub
quarkus.index-dependency.engine-work-adapter.artifact-id=casehub-engine-work-adapter
quarkus.index-dependency.engine-blackboard.group-id=io.casehub
quarkus.index-dependency.engine-blackboard.artifact-id=casehub-engine-blackboard
quarkus.index-dependency.engine-persistence-memory.group-id=io.casehub
quarkus.index-dependency.engine-persistence-memory.artifact-id=casehub-engine-persistence-memory

# CDI: exclude ambiguous work provider; use memory stores in tests
quarkus.arc.exclude-types=\
  io.casehub.connectors.twilio.TwilioSmsConnector,\
  io.casehub.connectors.whatsapp.WhatsAppConnector,\
  io.casehub.work.runtime.service.JpaWorkloadProvider

quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.persistence.memory.MemoryPlanItemStore,\
  io.casehub.persistence.memory.MemorySubCaseGroupRepository

# Engine hibernate packages on default datasource (existing line extended)
quarkus.hibernate-orm.packages=\
  io.casehub.work.runtime,io.casehub.clinical.entity
```

---

## Test Architecture

### Pattern (from devtown `HumanApprovalLifecycleTest`)

Tests cannot rely on `@ObservesAsync` CDI delivery to externally-packaged observers (engine#315). Pattern:
1. `WorkItemCompletionCapture` — captures `@ObservesAsync WorkItemLifecycleEvent` delivery
2. `WorkItemQueries.scanAll()` — reads WorkItems from `WorkItemStore`
3. `WorkItemLifecycleAdapter` — injected and called directly with `WorkItemLifecycleEvent.of(...)`

### `IrbGateLifecycleTest` checkpoints

```
1. Fire ProtocolDeviationResolvedEvent(IRB_REVIEW, APPROVED)
   → IrbApproval(PENDING) created with deviation_id set
2. await: IRB WorkItem exists (callerRef contains caseId) [engine#312 — 5s timeout]
3. Complete WorkItem: {"decision":"APPROVED","committeeId":"irb-001","decidedAt":"..."}
   → verify WorkItemCompletionCapture received event
   → manually invoke: lifecycleAdapter.onWorkItemLifecycle(COMPLETED, wi)
4. await: IrbApproval.decision == APPROVED
   await: ProtocolDeviation.piApprovalStatus updated
   verify: IrbApprovalResolvedEvent fired
5. await: case COMPLETED (CaseStatus.COMPLETED)

Additional: EXPIRED path
   → let WorkItem expire (or simulate via lifecycleAdapter with EXPIRED status)
   → verify: IrbApproval.decision == EXPIRED, IrbApprovalResolvedEvent(EXPIRED) fired
```

### `AeEscalationLifecycleTest` checkpoints

```
Grade 3 path:
1. reportAdverseEvent(GRADE_3) → AdverseEventReportedEvent fired, no workItemId on ae
2. await: one WorkItem → senior-safety-monitors; assert dsmb WorkItem absent
3. Complete WorkItem → lifecycleAdapter invoked
4. await: case COMPLETED; AeEscalationCompletedEvent fired (dsmbEscalated=false)

Grade 4 path:
1. reportAdverseEvent(GRADE_4) → AdverseEventReportedEvent fired
2. await: two WorkItems → senior-safety-monitors + dsmb (parallel)
3. Complete both → lifecycleAdapter invoked for each
4. await: case COMPLETED; AeEscalationCompletedEvent fired (dsmbEscalated=true)

Grade 1/2 regression path:
1. reportAdverseEvent(GRADE_1) → no AdverseEventReportedEvent; workItemId set on ae
2. assert: no engine case started; WorkItem candidateGroups=safety-officers
```

---

## Gotchas (from garden)

| ID | Issue | Impact | Mitigation |
|---|---|---|---|
| GE-20260521-a0f5a6 | `HumanTaskScheduleHandler` PENDING guard — PlanItem pre-marked RUNNING | WorkItem may not be created | `await().atMost(5s).untilAsserted()` in tests; engine#312 filed |
| engine#315 | `@ObservesAsync` CDI delivery to external jar observers unreliable in tests | `WorkItemLifecycleAdapter` not triggered in test | Use `WorkItemCompletionCapture` + direct adapter invocation |
| GE-0167 | `inputSchema`/`outputSchema` use mini-DSL, not JQ | Dot-path syntax only; no JQ filters | Use `{ key: .path }` format only |
| engine#314 | Nested `{..}` in `outputMapping` unsupported | Data extraction fails silently | Flat pattern: `{ key: . }` — resolution mapped wholesale |
| JpaWorkloadProvider | CDI ambiguity between work module and engine bridge | `AmbiguousResolutionException` at startup | Exclude `JpaWorkloadProvider` in test `application.properties` |
| Grade 3+ AEs | `ae.workItemId` is null after Layer 5 | Tests asserting `workItemId != null` for Grade 3+ break | Intentional — update affected tests |

---

## Deferred Issues

| Issue | Filed |
|---|---|
| `engine_case_id` on `ProtocolDeviation` + `AdverseEvent` (Layer 6) | casehubio/clinical#26 |
| `AdverseEvent.escalationStatus` domain field | casehubio/clinical#27 |

---

## Platform Coherence

- **Application tier rule** ✅ — all new code is domain-specific clinical logic; nothing in foundation
- **CDI async pattern** ✅ — all observers use `@ObservesAsync`; ledger writes are async
- **Named datasource rule** ✅ — ledger subclasses on `qhorus`; domain tables on default
- **Flyway version ranges** ✅ — V109 (default, clinical domain); V1009/V1010 (qhorus, ledger subclass joins)
- **SPI default populated** ✅ — `DefaultAdverseEventEscalationPolicy` is a vocabulary SPI; no-op would break routing
- **`outputMapping` flat pattern** ✅ — engine#314 constraint respected throughout
- **LedgerEntry subclasses in `io.casehub.clinical.ledger`** ✅ — never in `io.casehub.clinical.entity`
- **Auth retrofit readiness** ✅ — no auth logic in service or domain layers
- **Case definition three-layer architecture** ✅ — YAML → schema model → canonical API; no YAML bypassed