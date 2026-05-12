# Handoff — casehub-clinical
2026-05-12

## What changed this session

Epic 4 (adverse event escalation) fully implemented. 30 tests passing (was 17).

**What was built:**
- `CtcaeGrade.sla()` now returns 7-day SLA for Grade 1-2 (was `Optional.empty()`)
- Flyway migrations renamed V1-V6 → V100-V105; V106 (work_item_id), V1005 (ae_ledger_entry) added
- `AdverseEvent.workItemId` field added
- `AdverseEventLedgerEntry` (JOINED LedgerEntry subclass) in `io.casehub.clinical.ledger`
- `AdverseEventService` — sets reportedAt server-side, computes slaDeadline, creates WorkItem, writes ledger entry in one JTA transaction
- `POST /{enrollmentId}/adverse-events` endpoint on `PatientResource`
- `AdverseEventResourceTest` (5 tests), `AdverseEventServiceTest` (6 tests), showcase extended with Grade 3 + Grade 5 AE scenarios

**Key discoveries (all now in CLAUDE.md):**
- Quarkus ArC ignores `beans.xml` `<alternatives>` — use `quarkus.arc.selected-alternatives`
- Panache entities can't span two PUs — `AdverseEventLedgerEntry` in `io.casehub.clinical.ledger`, not `.entity`
- casehub-work must be scanned at `io.casehub.work.runtime` (full package, not just `.model`)
- H2 multi-datasource JTA requires `quarkus.datasource.*.jdbc.transactions=xa` in test properties
- casehub-ledger owns V1000-V1004 (V1004 = actor_identity) — consumer joins start at V1005

## Current state

Both repos on `epic-adverse-event-escalation`. All 30 tests green.

```
clinical/
├── api/     CtcaeGrade with correct SLAs for all grades
└── runtime/ AdverseEventService, AdverseEventLedgerEntry, AE endpoint, 30 tests
```

Engine#112 open — sub-case execution wiring needed before Epic 3 can resume.

## What's next

Epic 4 is complete. Next options:

1. **Close Epic 4** — invoke `superpowers:finishing-a-development-branch` to merge or PR
2. **Epic 5** — protocol deviation PI authorisation (COMMAND lifecycle, already in foundation)
3. **Epic 3** — blocked on engine#112; check if that's been progressed

Before starting any new epic: invoke `session-start` for platform coherence check.

## References

- Blog: `blog/2026-05-12-mdp02-adverse-event-sla-wiring.md`
- Journal: `design/JOURNAL.md` (updated this session with Epic 4 discoveries)
- Epic 4 issue: casehubio/clinical#4
- Engine sub-case tracking: casehubio/engine#112 (open)
- Epic 3 blocked: casehubio/clinical#3
- Garden entries (this session): GE-20260512-ee7c07, GE-20260512-4d6f48, GE-20260512-d0fa82, GE-20260512-7f4aa4
