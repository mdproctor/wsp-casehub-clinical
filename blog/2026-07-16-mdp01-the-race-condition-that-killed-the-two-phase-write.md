---
layout: post
title: "The race condition that killed the two-phase write"
date: 2026-07-16
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub]
tags: [cbr, case-based-reasoning, caseoutcomeobserver, routing]
---

Phase 1 of CBR (#116) wired the basics — three CDI event writers, feature schemas, precedent endpoints. It worked. AE escalation completes, CDI event fires, `AeResolutionCbrWriter` stores a `FeatureVectorCbrCase`. Clinicians query the precedent endpoint and see past similar events.

The problem: `FeatureVectorCbrCase` has no plan trace. The routing strategy (`CbrAgentRoutingStrategy`) reads plan traces to score workers by historical outcomes. Without them, every AE precedent is invisible to routing. The CBR cycle was Retain and Retrieve but not Reuse — what the garden calls "degenerate CBR."

I'd planned a two-phase approach: the CDI writer stores features immediately at escalation completion, then `CaseOutcomeObserver` enriches the stored case with plan traces when the engine case closes. Two writes, erase-before-store handles the replacement.

The design review killed it. The temporal ordering is wrong. `CaseOutcomeObserver.onOutcome()` is a synchronous SPI call during engine case close. `AeResolutionCbrWriter` fires via async CDI — `CaseLifecycleEvent` → `AeEscalationListener` → `AeEscalationCompletedEvent` — at least two async hops *after* the engine begins closing. The writer's `storeIdempotent()` arrives late and overwrites the observer's enriched case, erasing the plan traces it just wrote. The two-phase write defeats itself.

The fix: delete `AeResolutionCbrWriter` entirely. `ClinicalCaseOutcomeObserver` becomes the sole AE CBR writer — one write, one lifecycle hook, complete data from the start. It loads the `AdverseEvent` entity for clinical features, queries `PlanItemStore` for the execution trace (which workers handled which bindings), and stores a `PlanCbrCase` with both. The deviation and amendment writers stay unchanged — their "workers" are PIs and IRB committees, domain entities known to the clinical model, not engine-assigned agents.

The routing story has an honest gap. AE escalation uses `humanTask` bindings, not `CapabilityTarget`. The engine's `CbrCaseRetainObserver` resolves capability names via `binding.target() instanceof CapabilityTarget` — humanTask targets produce an empty map. So the observer hardcodes a clinical binding→capability map (`safety-review` → `safety-monitoring`, `dsmb-escalation` → `data-safety-monitoring`). The plan traces are structurally correct and ready for when the engine adds humanTask routing enrichment, but automated routing isn't end-to-end today. The precedent display — showing clinicians which safety officer handled past similar events — works now.

Two new AE features round out the original #78 spec: `treatmentArm` (same drug exposure profile implies higher precedent relevance) and `priorAeCount` bucketed as NONE/ONE/MULTIPLE (the clinical distinction between first event, second event, and recurrent pattern). The precedent query now uses clinical-specific weights — grade at 3.0, eventType at 2.5, outcome features zeroed.
