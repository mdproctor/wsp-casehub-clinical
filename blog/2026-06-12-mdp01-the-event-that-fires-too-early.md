---
layout: post
title: "The Event That Fires Too Early"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [casehub-engine, layer-8, action-risk-classifier, debugging]
---

Layer 8 was supposed to be the straightforward one.

The specification was clear: implement `ClinicalActionRiskClassifier`, wire it with a `SusarCriteriaEvaluator` worker, and create the SUSAR oversight gate for Grade 4/5 adverse events. The AML reference implementation had done something equivalent in Layer 9. We had four rounds of spec review, identified all the gotchas in advance. This was execution, not exploration.

The classifier itself took minutes. `ClinicalActionType` as a pure Java enum carrying all gate policy per constant — reason, candidateGroups, reversible, scope — with `ClinicalActionRiskClassifier @RiskClassifier` delegating to it through `fromActionType()`. Clean. The engine's `ChainedReactiveActionRiskClassifier` discovers `@RiskClassifier` beans automatically via CDI qualifier; no registration needed. That part worked on the first compile.

The evaluator took longer but not much. `SusarCriteriaEvaluator` needed to determine whether an AE was Grade 4/5, unexpected, and IMP-suspected. I had wanted to read those values from the engine's context — the initial context set when the AE escalation case starts contains exactly those fields. That turned out to be where the session got interesting.

---

The first sign something was wrong was that the gate WorkItem never appeared.

The binding fired — we could see "Agent selected: worker='susar-criteria-evaluator'" in the logs. The Quartz job executed. But the `WorkerResult` came back with no `PlannedAction`. No gate. No WorkItem. The worker was reporting that the AE didn't meet SUSAR criteria even though we'd explicitly set `unexpected=true` and `grade=GRADE_4` in the test.

Logging `context.get("grade")` inside the evaluator function printed null. `context.get("unexpected")` printed null. The context map that the engine passed to the worker was empty.

There's an existing garden entry (GE-20260417-4a3c22) that documented the same symptom — null fields arriving in worker lambdas. Its root cause was listed as "not fully determined." We were about to determine it.

The engine's `WorkerScheduleEventHandler` builds the input data for the Quartz job at scheduling time:

```java
// Line 81 — reads LIVE context, not event snapshot:
Map<String, Object> inputData = this.evalJqAsMap(
    instance.getCaseContext().asJsonNode(),
    capability.getInputSchema());
```

That's the live `CaseInstance.getCaseContext()`. Not the snapshot that `CaseContextChangedEventHandler` evaluated the binding filter against — the live object, read milliseconds later.

The engine fires `CaseContextChangedEvent` before the initial context is applied to the live instance. The snapshot in the event is the correct full context — that's what the binding filters evaluate against. But by the time `WorkerScheduleEventHandler` processes the corresponding `WorkerScheduleEvent`, the live instance context hasn't been updated yet. It's empty.

The YAML humanTask bindings don't hit this problem because their filters require affirmative Boolean fields: `.requiresSeniorMonitor == true` evaluates to false against empty context, so they don't fire on the early event. They wait for the second event, when the context is fully populated. A programmatic worker binding with `.susarAssessmentComplete == null` fires vacuously — null equals null — and gets dispatched with an empty input map.

Worse: `PlanningStrategyLoopControl` marks the plan item RUNNING immediately after the first dispatch. Every subsequent `CaseContextChangedEvent` (including the one with the full context) finds the item already RUNNING and skips it. The worker got one shot, took it with empty data, and the case moved on.

---

The fix I didn't want to write was the right one: load the entity from the database.

The `AdverseEvent` entity is committed before `startCase()` is called. It always exists, it always has the correct values. `SusarCriteriaEvaluator.apply()` now takes the `aeId` from whatever context it does get, loads the entity with `@Transactional`, and reads `ae.grade`, `ae.unexpected`, `ae.suspected` directly. No reliance on the engine's context propagation path.

The worker binding itself — connecting the evaluator to the `ae-escalation.yaml` case — is filed as clinical#77. Adding a programmatic `ContextChangeTrigger` binding to an existing `YamlCaseHub` hits the same timing problem: any filter that could match the empty initial context will fire too early. The AML Layer 9 pattern avoids this by using a dedicated oversight case hub with its own YAML, where the initial context is set correctly before anything fires. That's the clean path.

What shipped in this layer is the classifier and the evaluator — the decision machinery. Five regulatory gate types encoded as enum constants with their approval groups and reversibility. A default evaluator that loads the entity and applies ICH E2A criteria. An ADR recording why the named CDI interface for displacement landed in `runtime/service/` rather than `api/spi/` (the api module can't depend on the engine — `WorkerResult` is engine-api).

The oversight case hub comes next. This time I know what the engine actually does.
