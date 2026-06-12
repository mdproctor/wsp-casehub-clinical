# ClinicalActionRiskClassifier — Design Spec
**Issue:** casehubio/clinical#47
**Date:** 2026-06-11 (revised ×4, implementation note added 2026-06-12)
**Branch:** issue-47-action-risk-classifier

---

## What We're Building

An `ActionRiskClassifier` implementation for casehub-clinical that gates consequential
agent actions before the engine commits the worker's output to the case blackboard.
Also adds the first agent worker binding to `ae-escalation.yaml` — a SUSAR criteria
evaluator that proposes regulatory filing when Grade 4/5 AEs meet unexpectedness and
assumed-causality criteria.

This is **Layer 8** of the clinical harness layer progression (trust routing is Layer 7,
issue #8). LAYER-LOG and ARC42STORIES §9.4 must be updated when this layer closes.

This is not a port of AML Layer 9. AML's oversight gate is an isolated parallel harness
(separate YAML, coordinator, REST endpoint). Clinical integrates the gate inside the
existing `ae-escalation.yaml`. The architectures differ deliberately: SUSAR assessment
is an intrinsic part of AE escalation, not a separate concern.

Foundation shared opportunities (NLI-based classifier, shared `GatePolicy` enum) are
tracked in casehubio/engine#472 and deferred pending casehub-neural-text readiness.

---

## Action Taxonomy

`ClinicalActionType` enum — pure Java, no framework deps. Lives in `api/model/`.
All five types are ALWAYS-gated: all are regulatory obligations, not configurable policy.

| Constant | actionType | candidateGroups | reversible | Regulatory basis |
|---|---|---|---|---|
| `SUSAR_CRITERIA_DECISION` | `susar.criteria.decision` | `["qualified-investigator"]` | false | ICH E2A §I.A — classification of unexpected suspected serious reaction: starts the 7-day reporting clock |
| `SUSAR_REGULATORY_FILING` | `susar.regulatory.filing` | `["qualified-investigator"]` | false | 21 CFR 312.32(c) — actual FDA/EMA submission; a separate act from classification |
| `PATIENT_WITHDRAWAL` | `patient.withdrawal` | `["principal-investigator"]` | false | GCP §4.8.3 — consent lifecycle, irreversible |
| `DOSE_MODIFICATION` | `dose.modification` | `["principal-investigator"]` | true | GCP — physician approval required; reversible (dosage can be restored) |
| `PROTOCOL_DEVIATION_RECORDING` | `protocol.deviation.recording` | `["principal-investigator", "irb-committee"]` | false | GCP §4.5 — PI accountability |

**`SUSAR_CRITERIA_DECISION` vs `SUSAR_REGULATORY_FILING` — two distinct actions:**
Classification (criteria decision) starts the 7-day clock and is performed by a safety
evaluator. Filing is the formal regulatory submission act. The approving clinician may
differ; the timing obligations differ. They cannot be collapsed.

**`candidateGroups` semantics (GE-20260607-326c7e):** fewer entries = more restrictive
in chain resolution. SUSAR types require a single specialist — most restrictive.
Deviation recording accepts either PI or IRB committee — broadest pool.

**`expiresIn`:** `null` for all types — regulatory deadline policy is post-GA config,
per AML precedent.

**`scope`:** `"casehubio/clinical/oversight"`.

---

## SUSAR Criteria — Regulatory Basis

A SUSAR (Suspected Unexpected Serious Adverse Reaction) satisfies all three of:

1. **Serious** — fatal, life-threatening, requires hospitalisation, causes disability,
   or is a congenital anomaly (ICH E2A §I.A)
2. **Unexpected** — not in the Investigator Brochure expected adverse reaction list
3. **Suspected** — causal relationship with the IMP is not ruled out

**Grade threshold:** This implementation covers the 7-day expedited path (Grade 4 =
life-threatening, Grade 5 = fatal). Grade 3 unexpected AEs require a 15-day expedited
report under 21 CFR 312.32(c)(1)(ii) — this path is explicitly deferred (out of scope).

**Causality default:** ICH E2A §I.A.1 states reporters should assume causal relationship
unless explicitly stated otherwise. This implementation treats all AEs as IMP-suspected
by default. The `AdverseEvent.suspected` field (added by this issue) allows explicit
exclusion: `suspected = false` means a non-IMP event (e.g. road traffic accident) that
must not be gated as a SUSAR. Default is `true`. Conservative default is
regulatory-correct: a false negative (ungated real SUSAR) is the more dangerous error.

**Summary:** `SusarCriteriaEvaluator` gates when:
`grade ∈ {GRADE_4, GRADE_5} AND unexpected == true AND suspected != false`

---

## Data Model Changes Required

The `unexpected` and `suspected` fields must propagate from the data model through the
case context. `AdverseEventReportedEvent` is NOT modified — it is a lightweight
notification signal. The entity is already loaded in
`AeEscalationCaseService.prepareAndMarkRequested()` (`AdverseEvent.findById(event.aeId())`),
so both fields are read directly from `ae.unexpected` and `ae.suspected` there.

### `AdverseEvent` entity

```java
@Column(nullable = false)
public boolean unexpected = false;

@Column(nullable = false)
public boolean suspected = true;   // conservative per ICH E2A §I.A.1
```

### Flyway migration

`V111__adverse_event_unexpectedness.sql`:
```sql
ALTER TABLE adverse_event ADD COLUMN unexpected BOOLEAN NOT NULL DEFAULT FALSE;
ALTER TABLE adverse_event ADD COLUMN suspected  BOOLEAN NOT NULL DEFAULT TRUE;
```

### `PatientResource.ReportAdverseEventRequest`

Add optional `Boolean unexpected` and `Boolean suspected` fields (null → entity default).
`PatientResource.reportAdverseEvent()` sets `ae.unexpected` and `ae.suspected` from the
request before calling `adverseEventService.reportAdverseEvent(ae)`. The service receives
the entity already populated and needs no change for this feature.

### `AeEscalationCaseService.prepareAndMarkRequested()`

```java
ctx.put("unexpected", ae.unexpected);
ctx.put("suspected",  ae.suspected);
```

No change to `AdverseEventReportedEvent` — entity is already in scope at this call site.

---

## Components

### `ClinicalActionType` — `api/src/main/java/io/casehub/clinical/api/model/`

Pure Java enum. Carries all gate data per constant (reason, candidateGroups, reversible,
scope, expiresIn). Exposes `actionType()` (dot-notation string) and `fromActionType(String)`
(Optional, null-safe, no throw). No framework annotations.

### `ClinicalActionRiskClassifier` — `runtime/src/main/java/io/casehub/clinical/routing/`

`@ApplicationScoped @RiskClassifier`. Implements `ActionRiskClassifier`.

- Unknown or null `actionType` → `Autonomous` (owns only clinical types)
- Known type → `GateRequired` from enum constant data (all are ALWAYS)
- No switch statements — delegates to `ClinicalActionType.fromActionType()`
  (descriptor+handler protocol, PP-20260609)

Engine's `ChainedReactiveActionRiskClassifier` discovers this bean automatically.
Classifier exception fail-safe: engine applies `GateRequired("Classifier error — manual
review required", reversible=true, null groups)`.

### `SusarEvaluatorFunction` — `api/src/main/java/io/casehub/clinical/api/spi/`

```java
public interface SusarEvaluatorFunction
    extends Function<Map<String, Object>, WorkerResult> {}
```

Named interface in `api/` for two reasons:
1. **CDI disambiguation:** injecting raw `Function<Map<String,Object>,WorkerResult>` risks
   `AmbiguousResolutionException` due to generic erasure if any other `Function` bean
   exists in the CDI context. A named interface provides a distinct type.
2. **Displacement contract:** future ML agent implements `SusarEvaluatorFunction` as a
   plain `@ApplicationScoped` bean (no `@DefaultBean`) and displaces the default without
   `@Alternative` or `selected-alternatives` config.

`Worker.builder().function(susarEvaluator)` still works — `SusarEvaluatorFunction`
extends `Function<Map<String,Object>,WorkerResult>`, satisfying the builder's overload.

### `SusarCriteriaEvaluator` — `runtime/src/main/java/io/casehub/clinical/service/`

`@DefaultBean @ApplicationScoped` implementing `SusarEvaluatorFunction`.

**Logic:**
```
grade ∈ {GRADE_4, GRADE_5}
  AND context["unexpected"] == true
  AND context["suspected"] != false   (absent key → default true per conservative rule)
→ WorkerResult.of(
    Map.of("susarRequired", true),
    PlannedAction.of(
      "SUSAR criteria met — clinician sign-off required before regulatory clock starts",
      ClinicalActionType.SUSAR_CRITERIA_DECISION.actionType(),
      Map.of("aeId", aeId, "grade", grade, "unexpected", true)))
otherwise →
  WorkerResult.of(Map.of("susarRequired", false))
```

Output map always contains `susarRequired` (true or false) so the YAML outputMapping
`.susarRequired` never produces null.

### `ClinicalAdverseEventCaseHub` — `runtime/src/main/java/io/casehub/clinical/service/`

Overrides `getDefinition()` to inject `SusarCriteriaEvaluator` and register it as an
in-process worker:

```java
@ApplicationScoped
public class ClinicalAdverseEventCaseHub extends YamlCaseHub {

    @Inject SusarEvaluatorFunction susarEvaluator;   // named interface, not concrete class

    private volatile CaseDefinition augmentedDefinition;

    public ClinicalAdverseEventCaseHub() { super("clinical/ae-escalation.yaml"); }

    @Override
    public CaseDefinition getDefinition() {
        if (augmentedDefinition == null) {
            synchronized (this) {
                if (augmentedDefinition == null) {
                    CaseDefinition def = super.getDefinition();
                    def.getWorkers().add(Worker.builder()
                        .name("susar-criteria-evaluator")
                        .capabilities(List.of(Capability.builder()
                            .name("safety-monitoring").inputSchema(".").outputSchema(".").build()))
                        .function(susarEvaluator)
                        .build());
                    augmentedDefinition = def;
                }
            }
        }
        return augmentedDefinition;
    }
}
```

CDI displacement: a future ML agent implements `SusarEvaluatorFunction @ApplicationScoped`
(no `@DefaultBean`). Quarkus ArC selects the non-default bean at the
`@Inject SusarEvaluatorFunction` site automatically.

### `ae-escalation.yaml` — new `susar-assessment` binding

```yaml
- name: susar-assessment
  on:
    contextChange:
      filter: ".susarAssessmentComplete == null"
  worker:
    capability: safety-monitoring
    input: "{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId, unexpected: (.unexpected // false), suspected: (.suspected // true) }"
    output: "{ susarAssessmentComplete: true, susarRequired: .susarRequired }"
```

Fires once per case. `susarRequired` is a **hook key** — its purpose is to trigger a
future `regulatory-submission` binding (out of scope for #47). It does not self-execute.

**Gate flow:** worker returns `PlannedAction` → engine creates gate WorkItem via
`ActionGateWorkItemHandler` (already on classpath from Layer 5) → **the worker's output
application is deferred** (not the entire case — `safety-review` and `dsmb-escalation`
humanTask bindings fire in parallel and proceed independently; the 24h SLA is not
blocked) → clinician approves WorkItem → engine re-fires
`WorkflowExecutionCompleted(plannedAction=null)` (GE-20260607-66daf2) → worker output
committed → `susarAssessmentComplete: true` on blackboard.

---

## Files

**New:**
- `api/src/main/java/io/casehub/clinical/api/spi/SusarEvaluatorFunction.java`
- `api/src/main/java/io/casehub/clinical/api/model/ClinicalActionType.java`
- `runtime/src/main/java/io/casehub/clinical/routing/ClinicalActionRiskClassifier.java`
- `runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java`
- `runtime/src/main/resources/db/migration/default/V111__adverse_event_unexpectedness.sql`
- `api/src/test/java/io/casehub/clinical/api/model/ClinicalActionTypeTest.java`
- `runtime/src/test/java/io/casehub/clinical/routing/ClinicalActionRiskClassifierTest.java`
- `runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java`
- `runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java`

**Modified:**
- `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java` — add `unexpected`, `suspected` fields
- `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java` — add optional request fields; set `ae.unexpected` and `ae.suspected` before calling service
- `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java` — add `ctx.put("unexpected")` / `ctx.put("suspected")`
- `runtime/src/main/java/io/casehub/clinical/service/ClinicalAdverseEventCaseHub.java` — override `getDefinition()`
- `runtime/src/main/resources/clinical/ae-escalation.yaml` — add `susar-assessment` binding

---

## Tests

**Unit (no Quarkus):**

`ClinicalActionTypeTest`
- Each constant: correct actionType string, candidateGroups size, reversible, scope
- `fromActionType()` round-trips all 5 constants
- Unknown string → empty; null → empty

`ClinicalActionRiskClassifierTest` — instantiated directly
- Each known action type → `GateRequired` with correct fields
- Unknown actionType → `Autonomous`; null → `Autonomous`

`SusarCriteriaEvaluatorTest` — instantiated directly
- Grade 4 + unexpected=true + suspected=true → PlannedAction with `"susar.criteria.decision"`
- Grade 5 + unexpected=true + (suspected key absent → default true) → PlannedAction
- Grade 4 + unexpected=false → `WorkerResult.of(Map.of("susarRequired", false))`
- Grade 3 + unexpected=true → no PlannedAction (7-day path only; Grade 3 is deferred)
- Grade 4 + unexpected=true + suspected=false → no PlannedAction (explicit non-IMP exclusion)
- All no-gate paths → output map contains `susarRequired: false` (never null)

**Integration (`@QuarkusTest`):**

`SusarActionGateLifecycleTest`

Setup: persist trial + site + enrollment; create `AdverseEvent` with
`unexpected = true`, `suspected = true`, `grade = GRADE_5`, `tenantId` stamped from
principal (required by `TrialActivationService.findByIdForTenant` pattern). Entity must
exist in DB before case starts.

Positive: start AE escalation case with Grade 5 + unexpected=true + suspected=true →
await gate WorkItem creation (Awaitility 10s) → approve WorkItem → await
`susarAssessmentComplete: true` on blackboard → assert `susarRequired: true`.

Negative: Grade 3 + unexpected=true → no gate WorkItem; `susarAssessmentComplete: true`,
`susarRequired: false` written directly.

Negative: Grade 5 + unexpected=false → no gate WorkItem; assert `susarAssessmentComplete: true`
and `susarRequired: false` written directly to blackboard by worker outputMapping.

---

## Constraints and Notes

- `ActionGateWorkItemHandler` on classpath from Layer 5 (`casehub-engine-work-adapter`)
- `pendingActionGate` in-memory only in v1 (engine#433) — gate lost on server restart;
  acceptable for this layer
- Layer 3 PI COMMAND and ActionRiskClassifier gate are complementary: classifier gates
  the agent's proposed action before recording; PI COMMAND governs formal accountability
  after recording
- `IrbGateLifecycleTest` ConditionTimeout failures (Task #2) are unrelated to this branch
- `AdverseEventReportedEvent` is NOT modified — `unexpected` and `suspected` are read
  from the entity directly in `prepareAndMarkRequested()` which already loads it
- `AeEscalationCompletedEvent` also lacks `unexpected` — the LAYER-LOG Layer 7 note
  identified this gap; it remains for Layer 7 (trust routing) to add to that event.
  The entity field added here is the shared source of truth for both layers.

**Idempotency guard dependency — `PlanningStrategyLoopControl` must be active:**
The `.susarAssessmentComplete == null` filter is the only guard preventing re-dispatch
while the gate is pending. During the gate pending window this filter remains `true` —
`susarAssessmentComplete` is not written until after approval. Without dedup,
every `CONTEXT_CHANGED` event (e.g., safety-review humanTask completing) would
re-dispatch the SUSAR worker and create a second gate WorkItem.

Re-dispatch is prevented by `PlanningStrategyLoopControl @Alternative @Priority(10)`
from `casehub-engine-blackboard`. Its `filterToDispatchable()` method keeps only bindings
whose PlanItem is in `PENDING` status. When the SUSAR worker is selected,
`indexSelectedForCompletion()` marks its PlanItem RUNNING synchronously — blocking
all subsequent re-dispatch for that binding.

Clinical already requires `casehub-engine-blackboard` for DSMB blackboard signalling,
so `PlanningStrategyLoopControl` is active and this constraint is automatically
satisfied. **Never add this binding to a case running on `ChoreographyLoopControl`
(the default, no-dedup loop control) without additional output-key guards.**

**Gate rejection / expiry — regulatory audit gap (tracked as #76):**
`ActionGateCompletionApplier` fires `ActionGateRejectedEvent` (Vert.x bus:
`"casehub.action.gate.rejected"`) on WorkItem REJECTED/CANCELLED, and
`ActionGateExpiredEvent` (`"casehub.action.gate.expired"`) on EXPIRED.
Clinical has no consumer of either event. When the SUSAR gate is rejected or expires:
- `susarAssessmentComplete` is never written to the blackboard
- The plan item is blocked from re-dispatch (RUNNING → not PENDING → filtered by
  `filterToDispatchable()`)
- The case can still complete via `safety-review-complete AND dsmb-complete` goals
- No audit record of the gate decision exists — auditor cannot distinguish
  "gate still pending" from "clinician explicitly declined SUSAR escalation"

The fix direction (deferred to #76) follows `IrbDecisionListener`'s EXPIRED pattern:
a `SusarGateRejectionListener` consumes the event bus and signals the case with
`susarAssessmentComplete: true, susarRequired: false`. The open design question is
how to identify that a given `gateId` corresponds to the SUSAR gate specifically
(requires investigation of the `pendingActionGate` in-memory store from engine#433).

---

## Out of Scope

- Grade 3 unexpected AE → 15-day expedited path (21 CFR 312.32(c)(1)(ii)) — deferred
- `regulatory-submission` binding reacting to `susarRequired: true` — future issue
- `SUSAR_REGULATORY_FILING` worker binding — future issue
- SUSAR gate rejection / expiry handler (`SusarGateRejectionListener`) — tracked #76
- NLI-based classifier (`casehub-engine-ai`) — casehubio/engine#472
- Shared `GatePolicy` enum in `casehub-engine-api` — casehubio/engine#472
- Worker bindings for `PATIENT_WITHDRAWAL`, `DOSE_MODIFICATION`,
  `PROTOCOL_DEVIATION_RECORDING` — follow when agents arrive
- `AeEscalationCompletedEvent.unexpected` — Layer 7 responsibility
