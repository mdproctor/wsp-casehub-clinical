# Handoff — casehub-clinical
2026-05-12

## What changed this session

Epic 3 paused — engine sub-case execution wiring is not implemented (engine#195 was scaffold only). Documented the gap, blocked clinical#3 on engine#112, fixed the incorrect foundation gates entry in CLAUDE.md. Epic 3 workspace/project branches kept for future resumption.

Epic 4 (adverse event escalation) brainstormed, designed, and planned. Implementation plan written and ready.

**Key decisions:**
- SLA escalation is deployment config (EscalationPolicy SPI), not clinical code — domain service sets `claimDeadline` + `candidateGroups` only
- Clinical domain migrations renamed V100-V105 (Quarkus Flyway scans casehub-work's V1-V21 from transitive JARs)
- Two-datasource architecture: default for domain+work, `qhorus` for casehub-ledger
- `AdverseEventLedgerEntry` is a domain-owned LedgerEntry subclass (JOINED, V1004 migration)
- connectors notification deferred to clinical#11 (prerequisites listed there)

**Artifacts created:**
- Spec: `specs/2026-05-11-epic4-adverse-event-escalation-design.md`
- Plan: `plans/2026-05-11-epic4-adverse-event-escalation.md`
- Blog: `blog/2026-05-12-mdp01-production-ready-scaffold.md`

## Current state

Both repos on `epic-adverse-event-escalation` branch. No code written yet — design and plan only.

```
clinical/
├── api/          11 enums + constants (CtcaeGrade needs 7-day fix per plan Task 1)
└── runtime/      6 entities, V100-V105 migrations (after rename), 17 tests
```

Engine#112 open and commented — sub-case execution wiring needed before Epic 3 can resume.

## What's next

**Implement Epic 4 using subagent-driven-development.** Start with Task 1 (CtcaeGrade SLA fix) and execute the plan task by task. Plan is at `plans/2026-05-11-epic4-adverse-event-escalation.md`.

Before executing: invoke `superpowers:subagent-driven-development`.

Note: the Flyway migration rename (V1-V6 → V100-V105) is **Task 2 Step 2.2** in the plan — it must happen before casehub-work is added as a dependency.

## References

- Spec: `specs/2026-05-11-epic4-adverse-event-escalation-design.md`
- Plan: `plans/2026-05-11-epic4-adverse-event-escalation.md`
- Blog: `blog/2026-05-12-mdp01-production-ready-scaffold.md`
- Engine sub-case tracking: casehubio/engine#112 (open)
- Epic 3 blocked: casehubio/clinical#3
- Epic 4 issue: casehubio/clinical#4
- Connectors deferred: casehubio/clinical#11
- Flyway numbering protocol: casehubio/parent#17 (open — add to platform docs)
- Garden entries: GE-20260511-a28064 (Flyway), GE-20260511-3e5a75 (SLA pattern), GE-20260511-b6f903 (LedgerEntry fields)
