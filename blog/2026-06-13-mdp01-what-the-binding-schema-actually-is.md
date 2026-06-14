---
layout: post
title: "What the Binding Schema Actually Is"
date: 2026-06-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [casehub-engine, layer-8, susar, gdpr, gate-listener]
---

The previous entry ended with "the clean path" — a dedicated SUSAR oversight case hub that doesn't touch the ae-escalation case. This is the entry where we built it.

The approach works exactly as expected. Start a separate case only when Phase 1 confirms the AE meets Grade 4/5 unexpected/suspected criteria. Use a dedicated YAML case definition (`susar-oversight.yaml`) with its own goal and binding. The binding filter is `.aeId != null and .susarAssessmentComplete == null` — false on the empty first-context event because `aeId` hasn't been set yet, true on the second when the full initial context lands. The timing problem that killed the programmatic binding is structurally unreachable.

Then we hit the first unexpected wall.

The YAML I'd drafted had a `worker:` object nested under the binding:

```yaml
- name: susar-assessment
  on:
    contextChange:
      filter: ".aeId != null and .susarAssessmentComplete == null"
  worker:
    capability: safety-monitoring
    inputSchema: "{ aeId: .aeId }"
```

Claude decompiled `io.casehub.model.Binding` from the engine JAR to check the schema. There is no `worker:` field. The `@JsonPropertyOrder` is `{"name", "on", "when", "capability", "subCase", "humanTask", "conflictResolverStrategy"}`. The mapper runs with `FAIL_ON_UNKNOWN_PROPERTIES=DISABLED`, so the entire `worker:` block is silently dropped. The binding's `getCapability()` returns null. `convertBinding()` throws `IllegalArgumentException: must have capability, subCase, or humanTask`.

The field is `capability: safety-monitoring` directly on the binding — a String reference to a named entry in `spec.capabilities`. The `inputSchema` belongs on the capability definition, not the binding. The Java function also needs programmatic registration via `getDefinition()` override — the YAML alone can't bind a Java function. `ClinicalSusarOversightCaseHub` now overrides `getDefinition()` and adds `Worker.builder().name("susar-criteria-evaluator").capabilities(...).function(susarEvaluator).build()` to the case definition, same pattern as the Layer 8 plan would have used on the ae-escalation hub.

The gate listener turned out to have its own problem.

`SusarGateDecisionListener` needs to discriminate: when `casehub.action.gate.rejected` fires, was this our SUSAR gate or some other gate? The obvious approach is to read `CaseInstanceCache.get(event.caseId()).getPendingActionGate().plannedAction().actionType()`. Claude read `ActionGateRejectedHandler` from the engine bytecode. Both `ActionGateRejectedHandler` and `ActionGateExpiredHandler` are `@ConsumeEvent(blocking = true)` — competing consumers on the same addresses as the clinical listener. They call `instance.setPendingActionGate(null)` synchronously before publishing downstream. Execution order against the clinical consumer is non-deterministic on the Vert.x worker pool. If the engine handler runs first, `getPendingActionGate()` is null by the time we read it. The ledger write is silently skipped. For an FDA audit trail that's not acceptable.

The fix: `SusarOversightCaseService` persists `ae.susarOversightCaseId = caseId` in Phase 3, before the gate WorkItem is created. `SusarGateDecisionListener` calls `AdverseEvent.findBySusarOversightCaseId(event.caseId())`. Non-null means ours. The DB write happens-before the gate; the discriminator is race-free and survives JVM restart.

The GDPR work (`ConsentWithdrawalService`) was more straightforward. Patient withdrawal pseudonymizes the external `patientId`, calls `LedgerErasureService.erase()` to tokenize the actorId across all ledger entries, and wipes patient-specific memories. One wrinkle: enabling `casehub.ledger.identity.tokenisation.enabled=true` globally in test properties caused existing tests to fail — `PiResponseListenerTest` asserts `entry.actorId == "human:pi-L"` and gets a UUID token instead. The property belongs scoped to specific tests, not the shared config.

The ComplianceSupplement work was clean once we found the right method name. It's `entry.attach()`, not `addSupplement()`. One factory class, six static methods, one call per ledger writer before save.

Three years of casual assumptions about how a YAML case binding routes to a Java function — resolved in an afternoon of reading bytecode.
