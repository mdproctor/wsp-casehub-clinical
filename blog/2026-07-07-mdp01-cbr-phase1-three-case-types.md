---
title: "CBR Phase 1 — Three Case Types, One Clinical Harness"
date: 2026-07-07
author: mdp
tags: [cbr, neocortex, case-based-reasoning, clinical]
---

Wired casehub-neocortex's CBR store into clinical today. The goal was straightforward — store resolution precedents so future AE/deviation/amendment decisions can reference what happened before — but the design choice that made it interesting was using all three CBR case types to demonstrate the platform's retrieval breadth.

**FeatureVectorCbrCase for adverse events.** Nine-dimensional feature vector: grade (numeric 1-5), eventType, trialPhase, unexpected, suspected as problem features; safetyReviewOutcome, dsmbEscalated, indReportFiled, susarOversight as outcome features. The query weights outcome features at zero — "find AEs that looked like this one" regardless of how they were resolved. The scored results show whether similar AEs were escalated, filed IND reports, or triggered SUSAR oversight. CbrQuery's per-field weighting makes this a single API call, not separate queries.

**PlanCbrCase for deviations.** Same feature vector approach for similarity matching, plus a plan trace capturing the engine case's binding sequence. The deviation-review case has two bindings (pi-oversight, irb-committee), so the trace is short but semantically meaningful — it shows whether the deviation went through PI approval only or escalated to IRB review. The two-event observer pattern (ProtocolDeviationResolvedEvent + IrbApprovalResolvedEvent) ensures CRITICAL deviations get their IRB decision recorded even though it arrives days after the PI decision.

**TextualCbrCase for amendments.** Pure text — the proposedChange is the problem, the advisor recommendation is the solution. No structured features in Phase 1; InMemoryCbrCaseMemoryStore returns all amendments with score 1.0 (effectively a recent-amendments list). Phase 3 will activate semantic text similarity via Qdrant embeddings, making this a true "find similar proposals" query. The Phase 1 limitation is documented honestly in the spec and the REST endpoint response.

The session also fixed two pre-existing flaky tests. The interesting one: `ThreeSiteShowcaseTest` failed because three concurrent `@ObservesAsync` observers updated the same `AdverseEvent` entity without `@DynamicUpdate`. Hibernate's full-row UPDATE silently overwrote earlier observers' status changes. Symptom looked like a timing issue; root cause was JPA's default update strategy. Garden entry GE-20260707-76a6f4.

**What ships:** 30 new tests, three REST precedent endpoints, explore page panels, and the CBR foundation that Phase 2+ will build on (CBR-informed agent routing, AE trajectory monitoring, semantic text matching).
