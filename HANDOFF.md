# Handoff — casehub-clinical
2026-05-23

## What happened this session

**Epic 6 complete — Layer 5 casehub-engine introduced.** 115 tests, 0 failures.

- IRB gate: `ClinicalDeviationCaseHub` + `deviation-review.yaml` — CRITICAL deviation + PI approval → 72h IRB WorkItem → case completes on all four terminal outcomes (APPROVED/REJECTED/DEFERRED/EXPIRED). `IrbDecisionListener` bridges WorkItem lifecycle to `IrbApproval` entity + ledger.
- AE escalation: `ClinicalAdverseEventCaseHub` + `ae-escalation.yaml` — `AdverseEventEscalationPolicy` SPI replaces hardcoded routing. Grade 3: senior monitor gate. Grade 4+: senior monitor + DSMB in parallel. Same YAML, context determines paths.
- Engine CDI wiring: `casehub-platform` + `casehub-platform-expression` required (undocumented). Quartz cron incompatibility with casehub-work scheduler beans fixed. `on.contextChange.filter` not `when` for binding conditions (engine#335).
- PR #32 opened → casehubio/clinical. Issue #6 closed. 11 blog entries published to mdproctor.github.io. ADR 0001 promoted to project. Garden: 3 entries.

## Current state

- **Project repo:** `main` (rebased from Epic 6 branch), 115 tests pass
- **Workspace:** `main`
- Blog: `2026-05-23-mdp01-layer-5-engine-arrives.md`
- ADR: `adr/0001-layer-5-case-granularity-per-event-not-per-site.md`

## Stale workspace branches (low priority)

*Unchanged — `git show HEAD~1:HANDOFF.md`* (epic-deviation-resolution-ledger, epic-multi-site-sub-case, issue-18-deviation-expiration-requires-new, issue-24-minor-cleanups due deletion 2026-06-05)

Also: `issue-6-irb-gate` (marked closed, deletion due 2026-06-06)

## Key open items

| Issue | What | Priority |
|-------|------|----------|
| casehubio/clinical#11 | Adverse event safety officer notification via connectors | After connectors pattern |
| casehubio/clinical#26 | `engine_case_id` on domain entities (Layer 6) | Layer 6 |
| casehubio/clinical#27 | `AdverseEvent.escalationStatus` domain field | Layer 6 |
| casehubio/clinical#28 | `CaseLifecycleEvent` internal package coupling | Engine team |
| casehubio/clinical#29 | IRB committee assignment configurable | Layer 6 |
| casehubio/parent#49 | Sync casehub-clinical.md + PLATFORM.md for Layer 5 | Next parent session |
| casehubio/engine#335 | `when` field ignored on contextChange bindings | Engine team |

## What's next

**Layer 6 (engine#112) remains blocked.** Per-site sub-case orchestration needs engine sub-case execution wiring.

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #11 | AE safety officer notification via connectors | M | Med | After connectors pattern established |
| Layer 6 | Multi-site sub-case orchestration + DSMB rollup | XL | High | Blocked on engine#112 |
| Layer 7 | Trust routing + ClinicalAgent comparison | XL | High | After Layer 6 |

**Immediate next step:** Work on clinical#11 (AE safety officer notification) or check engine#112 status for Layer 6 unblock.
