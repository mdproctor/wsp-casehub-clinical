---
title: "Trajectory — The Shape of Escalation"
date: 2026-07-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, trajectory, dtw, clinical-trials, neocortex]
---

# casehub-clinical — Trajectory: The Shape of Escalation

**Date:** 2026-07-20
**Type:** phase-update

---

## What I was trying to achieve: predictive pattern matching from AE progression history

CBR Phases 1–5 gave clinical the ability to find similar past cases and adapt their plans. But similarity was point-in-time — "this Grade 3 cardiac AE looks like that Grade 3 cardiac AE." What it couldn't see was trajectory: a Grade 1 that escalated to Grade 3 in 48 hours is a very different clinical signal than a Grade 3 that appeared suddenly. The issue (#119) asked for trajectory monitoring — match the *shape* of escalation, not just the current state.

## What we believed going in: neocortex would need new temporal primitives

The issue listed `casehubio/neocortex — TemporalCbrCase type + trajectory similarity (DTW or similar)` as a dependency. I expected cross-repo work. But when I read the neocortex repo doc during work-start, the primitives were already there: `FeatureField.TimeSeries` with inner fields and timestamps, `SimilaritySpec.DtwSpec` with Sakoe-Chiba band warping, `TrendAnalyzer` computing slope, acceleration, and change points, and a `TrendEnrichmentCbrCaseMemoryStore` decorator that auto-enriches on store and retrieve.

The trajectory infrastructure existed. What clinical needed was application-layer code to use it.

## Lazy reconstruction — the design choice that simplified everything

The brainstorming question was: how does the developing trajectory get its data? New CDI events for every state change? Incremental CBR writes? Both add complexity — new event types, idempotency guards, partial trajectory consistency.

The answer was simpler. The engine's `PlanItemStore` already records every binding execution with timestamps. The AE entity carries the current state. Reconstruct the trajectory on demand from these existing sources. No new events, no incremental writes, always consistent with current state.

For proactive alerting, this means: at existing lifecycle moments (escalation started, SUSAR gate resolved, safety review completed), build the partial trajectory from what the PlanItemStore has recorded so far, query the CBR store with DTW similarity, and fire an alert if the matches predict a high-severity outcome. The evaluation hooks are single try-catch lines added to six existing services.

For retention: at case completion, `ClinicalCaseOutcomeObserver` builds the full trajectory and stores it as a CBR case with TimeSeries features. The library of past trajectories grows automatically.

## The cross-domain erasure trap

One bug cost an hour. `CbrCaseMemoryStore.eraseEntity(entityId, tenantId)` is not domain-scoped — it erases across all domains. When the outcome observer stored the same AE in `clinical-ae` (point-in-time features) and then `clinical-ae-trajectory` (TimeSeries), the second `storeIdempotent` call wiped the first domain's case. Silent failure — the integration test said "Expecting actual not to be empty" with no hint about the cause. Fix: suffix the entityId (`aeId + "-trajectory"`) so each domain has its own erasure scope.

## The design review that earned its keep

Claude's adversarial design review caught real problems. Grade was listed as a TimeSeries inner field — but `AdverseEvent.grade` never changes within a trajectory (it's set at report time). Including it produced meaningless flat-line DTW comparisons. The reviewer also caught that `PlanItemRecord` only has `createdAt`, not `completedAt` — temporal ordering is correct but inter-observation durations are approximate. And it found that the spec's lifecycle hook placements were wrong: `prepareAndMarkRequested()` runs before `persistCaseId()`, so `engineCaseId` is null at the hook point I'd specified. Five rounds, 19 issues, all resolved before a line of implementation code was written.

## What this enables

The system can now warn "this AE's escalation trajectory matches 3 past cases — 2 of them reached Grade 4 within 72 hours" before the progression completes. Site enrollment deceleration detection works the same way: weekly enrollment counts as TimeSeries, DTW against historical sites, alert when the pattern matches investigator disengagement.

Four deferred items filed as issues: alert notification wiring (#133), demo trajectory data (#134), AE grade regrading (#135), periodic enrollment storage (#136). The mechanism is in place; the consumers and the richer data come next.
