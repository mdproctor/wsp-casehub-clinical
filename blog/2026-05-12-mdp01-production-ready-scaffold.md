---
layout: post
title: "What 'production-ready' actually shipped"
date: 2026-05-12
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [quarkus, casehub-engine, casehub-work, adverse-events, flyway]
---

Epic 3 was supposed to be the architectural centrepiece — each trial site becoming its own sub-case, trial-level DSMB rollup detecting cross-site safety signals from accumulated blackboard state. The HANDOFF said the engine's sub-case API was production-ready (casehubio/engine#195, closed).

Claude read the blackboard code. What engine#195 actually shipped was a data model scaffold: `SubCase.java`, `Stage.subCases`, `CasePlanModel`, a completion strategy interface. What it did not ship was anything that actually spawns a child case when a binding fires. No `SubCaseExecutionHandler`. No listener that detects child case completion and resumes the parent. The blackboard and the engine runtime — `CaseHubRuntimeImpl` — are currently disconnected. You can describe a sub-case hierarchy in the model; you cannot run one.

I wasn't going to wire a workaround. The whole point of the architecture is that it does this properly. So we documented the gap, commented on engine#112 with the specific missing pieces, corrected the foundation gates table in CLAUDE.md, and moved to find what was actually unblocked.

That turned out to be Epic 4: adverse event escalation. Grade 3-4 events (serious, per ICH E6(R3)) require reporting within 24 hours. Grade 5 (death) gets an internal 1-hour SLA. Grade 1-2 carry a 7-day window. ClinicalAgent, the peer-reviewed baseline we're comparing against, has no deadline tracking at all. casehub-work has `claimDeadline` on `WorkItem` and an `EscalationPolicy` SPI for what happens when the deadline passes.

The design work surfaced three things worth noting.

## The Flyway collision nobody warned us about

casehub-work ships Flyway migrations in its JAR at `classpath:db/migration`, V1 through V21. Quarkus Flyway scans transitive JARs when it processes that path — not just the application's own resources. Our clinical domain migrations were also at V1-V6. Adding the casehub-work dependency would immediately blow up with "Found more than one migration with version 1." We renamed the clinical domain migrations to V100-V105. This is now documented in CLAUDE.md; it will matter for AML and devtown when they add casehub-work.

## SLA escalation is deployment config, not application code

The natural instinct is to observe `WorkItemLifecycleEvent` for EXPIRED status and hardcode what happens — spawn a DSMB WorkItem, fire a Slack message. But the same clinical trial on a different deployment might escalate differently depending on the institution. The domain service sets `claimDeadline` and `candidateGroups`. The `EscalationPolicy` SPI wired at deployment handles the breach. Clinical's code doesn't change between deployments; the config does. This kept casehub-connectors out of this epic — it's casehubio/clinical#11, waiting until the SLA pattern is established.

## LedgerEntry subclass fields are entirely undocumented

`LedgerEntryRepository.save()` expects `id`, `subjectId`, `sequenceNumber`, `entryType`, `actorId`, `actorType`, `actorRole`, and `occurredAt` — all set by the caller. There's no builder, no factory, no documented list of required fields. The only usage example is a test helper method buried in `LedgerPrivacyWiringIT.java`. We found it by reading source rather than docs.

The design and implementation plan for Epic 4 are written. The next session picks up with execution.
