# Layer 6 Design: Trial-Level Blackboard Aggregation — Cross-Site DSMB Rollup

**Date:** 2026-05-25
**Branch:** epic-3-multi-site-sub-case
**Issue:** casehubio/clinical#3

---

## What This Layer Adds

A trial-level engine case that accumulates Grade 4+ adverse event signals from
independent per-site AE escalation processes and triggers a DSMB committee review
when two or more sites are simultaneously affected — without any site-level service
knowing about other sites.

This closes the compliance gap ClinicalAgent cannot close: it runs a linear pipeline
for one site and has no concept of cross-site safety patterns or trial-level
coordination.

---

## What "Multi-Site Sub-Case" Actually Means Here

The original label for this layer was "multi-site sub-case orchestration." After
design analysis, sub-cases (parent-child engine relationships) are the wrong tool:
sites are long-running domain entities, not bounded work units that start and
complete. The engine's sub-case model (engine#112) is correct for bounded delegation;
it is not correct for modelling investigational sites.

The correct architecture is a trial-level case whose blackboard accumulates signals
from independent per-AE-event processes. The tutorial story is stronger for it: the
engine detects a cross-site pattern from accumulated state, not from a structural
hierarchy. Sub-case composition belongs in a later layer (patient batch screening).

---

## Architecture

```
ClinicalTrial (domain entity)
  └── engineCaseId : UUID                    NEW (V110 migration)

POST /trials/{id}/activate                   NEW endpoint
  └── TrialActivationService.activate()      NEW service
        ├── validates PLANNING → ACTIVE transition
        ├── ClinicalTrialCaseHub.startCase(initialContext)
        └── persists trial.engineCaseId

ClinicalTrialCaseHub                         NEW (extends YamlCaseHub)
  └── trial-coordination.yaml               NEW case definition

AeEscalationCaseService                      EXTENDED
  └── after starting AE case, if grade ≥ 4:
        runtime.signal(trialCaseId, "grade4Active.<siteId>", true)

AeEscalationCompletedEvent                   EXTENDED (add siteId field)

TrialSafetySignalService                     NEW
  └── observes AeEscalationCompletedEvent
        └── if grade ≥ 4:
              runtime.signal(trialCaseId, "grade4Active.<siteId>", false)

deviation-review.yaml                        FIXED (when: → filter:)
```

---

## Domain Model Changes

### `ClinicalTrial` — one new field

```java
@Column(name = "engine_case_id")
public UUID engineCaseId;   // null until trial transitions to ACTIVE
```

### V110 migration (`db/migration/default/V110__trial_engine_case_id.sql`)

```sql
ALTER TABLE clinical_trial ADD COLUMN engine_case_id UUID;
```

### `TrialSite` — unchanged

Sites remain pure domain entities. No engine case per site.

---

## Trial Activation

`POST /trials/{id}/activate` added to `TrialResource`. Delegates to
`TrialActivationService`, which:

1. Loads the trial; rejects if status ≠ `PLANNING` (400)
2. Sets `status = ACTIVE`
3. Starts the trial case: `caseHub.startCase(Map.of("trialId", id.toString(), "protocolId", trial.protocolId, "grade4Active", Map.of()))`
4. Persists `trial.engineCaseId = caseId`

The guard `if (trial.engineCaseId != null) return` in signal-sending code handles
the case where a signal fires before the trial is activated (correct: silently
drop).

---

## `trial-coordination.yaml`

```yaml
dsl: "0.1"
version: "1.0.0"
name: trial-coordination
namespace: clinical
title: Clinical Trial Coordination — cross-site safety monitoring

spec:

  goals:
    - name: dsmb-review-complete
      kind: success
      condition: ".dsmbReview != null"

  bindings:
    - name: dsmb-rollup
      on:
        contextChange:
          filter: "[.grade4Active // {} | to_entries[] | select(.value == true)] | length >= 2"
      humanTask:
        title: "DSMB review — simultaneous Grade 4+ events at multiple sites"
        expiresIn: PT48H
        candidateGroups: [dsmb]
        inputMapping: "{ trialId: .trialId, activeSites: [.grade4Active // {} | to_entries[] | select(.value == true) | .key] }"
        outputMapping: "{ dsmbReview: . }"
```

**No completion condition.** The trial case runs for the trial's lifetime. Goals
and completion conditions belong in the showcase layer (Layer 10). The DSMB binding
re-fires sequentially each time the condition becomes true again after the previous
WorkItem reaches a terminal state (PlanItem dedup allows sequential re-fire once
the prior PlanItem is terminal).

---

## Signal Flow

### Grade 4+ START

```
AdverseEventReportedEvent (grade, siteId, enrollmentId, aeId)
  → AeEscalationCaseService.onAdverseEventReported()        [existing observer]
      → start AE escalation case                            [existing]
      → if grade.value() >= 4:                              [NEW]
            look up trial.engineCaseId via siteId → TrialSite → ClinicalTrial
            if engineCaseId != null:
              runtime.signal(engineCaseId, "grade4Active." + siteId, true)
```

### Grade 4+ CLEAR

`AeEscalationCompletedEvent` gains a `siteId` field. `AeEscalationListener`
already has `siteId` in the case context at completion time — it populates
the enriched event at source, avoiding a downstream DB lookup.

```
AeEscalationCompletedEvent (aeId, grade, siteId)           [siteId added]
  → TrialSafetySignalService.onAeEscalationCompleted()      [NEW]
      → if grade.value() >= 4:
            look up trial.engineCaseId via siteId → TrialSite → ClinicalTrial
            if engineCaseId != null:
              runtime.signal(engineCaseId, "grade4Active." + siteId, false)
```

### Lookup helper

Both services share a `TrialCaseIdLookup` helper (or private method) that does:
`TrialSite.findById(siteId).trialId → ClinicalTrial.findById(trialId).engineCaseId`.
No new repository needed — uses existing Panache entities.

### Concurrency

Concurrent signals to `grade4Active.siteA` and `grade4Active.siteB` use different
path-based locks in `SignalReceivedEventHandler` and write to independent sub-keys.
No race condition. Confirmed by reading engine source.

---

## `deviation-review.yaml` Fix

The `when:` field is silently ignored for `contextChange` bindings (engine source
confirmed; GE-20260523-fd8725). The condition must be in `on.contextChange.filter`.

```yaml
# Before
- name: irb-consultation
  on: { contextChange: {} }
  when: ".irbConsultationRequired == true and .irbConsultation == null"

# After
- name: irb-consultation
  on:
    contextChange:
      filter: ".irbConsultationRequired == true and .irbConsultation == null"
```

Behaviour is unchanged for the IRB case (the case always starts with
`irbConsultationRequired=true`, so the first fire was always correct). This fix
makes the condition explicit and correct for any future context where the
condition might not hold on first evaluation.

---

## Known Simplifications (documented, not debt)

**Boolean flag per site, not a count.** `grade4Active.<siteId>` is a boolean.
If a site has two simultaneous Grade 4+ AEs, the first CLEAR signal sets the
flag to `false` even though one AE is still active. For the 3-site showcase
(one AE per site at a time), this is correct. A production implementation would
recompute the flag from domain truth at signal time: query "does this site
currently have any unresolved Grade 4+ AEs?" and signal the computed boolean.
The mechanism (`signal()` with the same path) is identical — it's a one-line
change in `TrialSafetySignalService`.

**Case cancellation leaves stale flag.** If an AE escalation case is cancelled
rather than completing normally, `AeEscalationListener` does not fire
`AeEscalationCompletedEvent` and the CLEAR signal is never sent. Case cancellation
is not implemented in the current codebase. Note for a later layer.

---

## Testing Plan

### Unit tests (no Quarkus)

| Class | Covers |
|---|---|
| `TrialSafetySignalServiceTest` | Grade 4 clears signal; Grade 3 does not; null `engineCaseId` guard |
| `AeEscalationCaseServiceSignalTest` | Grade 4 sets signal with correct path; Grade 3 does not |
| `TrialCoordinationYamlTest` | YAML parses without error; binding has correct filter and humanTask |

### Integration tests (`@QuarkusTest`, H2)

| Class | Covers |
|---|---|
| `TrialActivationTest` | Endpoint transitions status, persists `engineCaseId`, trial case running |
| `DsmbRollupTest` | Full scenario: two sites Grade 4+ → DSMB WorkItem created; WorkItem completes → `dsmbReview` in context; CLEAR signal → flag false |
| `DsmbRollupNoFireTest` | Single site Grade 4+ → no DSMB WorkItem |
| `DsmbGradeGuardTest` | Two sites Grade 3 → no signal, no DSMB WorkItem |
| `DeviationReviewYamlFixTest` | Existing IRB gate tests still pass after `when:` → `filter:` fix |

**Test wiring:** existing engine test configuration in `application.properties`
covers all required alternatives (memory stores, Quartz exclusions, scheduler
start-mode). No new profile needed if no alternative changes are required.
`QuarkusTestProfile.getEnabledAlternatives()` replaces not appends
(GE-20260521-4de4f1) — any test using a profile must declare all required
alternatives explicitly.

---

## Files Changed

| File | Change |
|---|---|
| `runtime/src/main/java/.../entity/ClinicalTrial.java` | Add `engineCaseId` field |
| `runtime/src/main/resources/db/migration/default/V110__trial_engine_case_id.sql` | New migration |
| `runtime/src/main/java/.../service/ClinicalTrialCaseHub.java` | New |
| `runtime/src/main/resources/clinical/trial-coordination.yaml` | New |
| `runtime/src/main/java/.../service/TrialActivationService.java` | New |
| `runtime/src/main/java/.../service/TrialSafetySignalService.java` | New |
| `runtime/src/main/java/.../service/AeEscalationCaseService.java` | Add Grade 4 START signal |
| `runtime/src/main/java/.../resource/TrialResource.java` | Add `POST /trials/{id}/activate` |
| `api/src/main/java/.../api/AeEscalationCompletedEvent.java` | Add `siteId` field |
| `runtime/src/main/java/.../service/AeEscalationListener.java` | Populate `siteId` in event |
| `runtime/src/main/resources/clinical/deviation-review.yaml` | Fix `when:` → `filter:` |