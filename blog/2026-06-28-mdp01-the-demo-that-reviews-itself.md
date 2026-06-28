---
layout: post
title: "The Demo That Reviews Itself"
date: 2026-06-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [demo-ui, spec-review, quinoa, casehub-pages]
series: issue-93-demo-ui-backend
---

# casehub-clinical — The Demo That Reviews Itself

**Date:** 2026-06-28
**Type:** phase-update

---

## What I was trying to achieve: a self-narrating demo UI backend

casehub-clinical has ten completed foundation layers, twenty REST endpoints, and a showcase test that proves the full trial lifecycle works. What it doesn't have is a way to show someone who isn't a developer. The goal: build a demo UI where an investor or CTO clones the repo, runs `mvn quarkus:dev`, opens a browser, and understands what AI Fusion actually means — trust-weighted agent routing, oversight gates, Merkle audit trails, and the qhorus accountability model.

## What I believed going in: the API surface was already there

I assumed the existing REST endpoints would feed the demo directly. They don't. The existing API is entity-lookup oriented — get one trial, one patient, one AE by ID with full ownership chain validation. A dashboard needs flattened lists, cross-site aggregations, and governance context that doesn't exist on any single entity. The data for "what the AI decided vs how the platform governed it" lives across three different sources: the AE entity, the `WorkerDecisionEntry` in the qhorus datasource, and the `ActorTrustScore` cache.

## Six rounds before a single line of code

The spec went through six rounds of adversarial review before implementation started. Each round caught something the previous one missed — and three of them would have caused build or runtime failures.

Round 1 identified that the spec described *what* the UI shows but never specified *how* it's built with the casehub-pages DSL primitives. The DSL code examples in the spec used the wrong function signatures — `columns({span: 3}, metric(...))` when the actual API is `columns([3, 3, 3, 3], [metric(...)], ...)`. Compile-time errors in the spec.

Round 2 caught that the 8-step guided narrative skipped the PI authorisation flow entirely. The SUSAR gate story shows "AI decided, human approved." The PI COMMAND → Commitment → Resolution story shows "the platform created a formal obligation on a named person with a traceable deadline." The second is the stronger sell. The narrative expanded from 6 to 8 steps.

Round 3 found that `ActionGateService.approve()` — the method I'd specified for SUSAR gate approval — doesn't exist. Gates are approved by completing a WorkItem. The actual flow is: find the gate WorkItem by callerRef prefix, claim it, complete it, and `ActionGateCompletionApplier` publishes the event. The same round caught that `channelService.receiveHumanMessage()` doesn't exist either — it's `channelGateway.receiveHumanMessage(ChannelRef, InboundHumanMessage)`.

Round 4 identified that `%dev.quarkus.arc.selected-alternatives` would have dropped three JPA alternatives from the non-profiled list — Quarkus profile properties replace, not merge. The fix: `@IfBuildProfile("dev") + @Alternative + @Priority(150)` on the bean itself, no config property needed.

Round 5 caught that the pre-seeded Grade 2 AE wouldn't produce any trust score data. `SusarCriteriaEvaluator` gates on `GRADE_4`/`GRADE_5`. Grade 2 AEs route directly to safety officers with no engine case, no attestation, no trust score update. The seeder needed Grade 4 unexpected AEs with full SUSAR gate lifecycles.

Round 6 fixed `getBytes()` → `getBytes(StandardCharsets.UTF_8)` for the deterministic UUID, and turned the DSMB rollup side effect into a feature — the seeded Grade 4 AEs leave `grade4Active` flags set, so when the demo adds a new Grade 4 AE at Site B, the trial case detects two sites with simultaneous signals and fires the DSMB review automatically.

## What we built

Seven dashboard endpoints on `TrialDashboardResource`: trial summary, patients, adverse events, deviations (all Panache queries), plus agents, governance context, and ledger entries (all cross-datasource aggregations joining the default and qhorus datasources). The governance endpoint powers the demo's hero layout — "What the AI decided" beside "How the platform governed it" — by joining AE entity fields with `WorkerDecisionEntry.trustScoreAtRouting` and current `ActorTrustScore`.

Two demo-only endpoints on `DemoActionResource`, guarded by `@IfBuildProfile("dev")`: PI approval (calls `channelGateway.receiveHumanMessage()` to fire the real qhorus CDI chain) and SUSAR gate approval (finds the gate WorkItem via `scanAll()` + callerRef filter, completes it via `WorkItemService.completeFromSystem()`, then calls `TrustScoreJob.runComputation()` for immediate Bayesian Beta recomputation).

`DemoDataSeeder` replays a complete trial scenario through real service calls — not direct inserts. This matters because `LedgerEntryRepository.save()` computes Merkle digests and updates the frontier automatically. Hand-inserting ledger entries would break the verification endpoint, and Step 8 of the demo — "The Proof" — calls that endpoint. The seeder creates three sites, ten patients, resolved events at each site, and runs three Grade 4 SUSAR lifecycles to produce real attestations and materialised trust scores.

## Where this leaves us

The Java backend is complete. The next piece is the TypeScript UI — fourteen casehub-pages DSL compositions that render the guided walkthrough and explore dashboard against these endpoints. The framework is casehub-pages (our own declarative dashboard runtime), embedded in clinical via Quinoa. `mvn quarkus:dev` will serve both the Java API and the TypeScript UI from a single process.

The spec review process surfaced something worth naming: every API the spec referenced that turned out to be wrong (`ActionGateService`, `channelService.receiveHumanMessage`, `sidebar()` nesting, `selected-alternatives` profile merging) would have been a runtime failure discovered during implementation. Catching them in the spec cost minutes of research each. Catching them in code would have cost debugging sessions.
