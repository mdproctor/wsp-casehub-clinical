# Session Handover — 2026-06-09

## Last Session

Implemented CaseMemoryStore integration (clinical#33) and multi-tenancy foundation (clinical#69) — both closed. V116 migration adds `tenant_id` to 6 domain tables; 3 CDI events and `SponsorNotificationRequest` gain `String tenantId`; all entity creation sites stamped. `ClinicalMemoryService` (PATIENT + SITE domains) wired into AE escalation and deviation lifecycle. `AeEscalationCaseService.prepareAndMarkRequested()` now injects `patientContext` + `siteContext` into engine `initialContext`; JQ-navigable. Branch closed, squashed to 1 commit on `casehubio/clinical:main`.

## Immediate Next Step

Run `/work` to start the next issue. Layer 7 (trust routing, #10) is the natural next epic. Check `docs/AGENTIC-HARNESS-GUIDE.md` for the current layer status before starting.

## What's Left

- casehubio/clinical#71 — query isolation: per-tenant Panache filters on all 6 entity types · M · Med
- casehubio/clinical#72 — DRUG memory domain: entityId convention + cross-tenant pharmacovigilance tradeoff · S · Med
- casehubio/clinical#73 — IRB memory domain: `IrbApproval.deviationType` gap (extra DB query vs schema change) · S · Med
- casehubio/clinical#74 — Remove `ClinicalTestLedgerRepository` workaround when casehub-ledger-memory updates to 2-arg `LedgerEntryRepository` API · XS · Low
- casehubio/clinical#75 — `toContextMap()` facts shape missing `grade` field (spec contract gap) · XS · Low
- casehubio/parent#208 — Sync `docs/repos/casehub-clinical.md` for memory store deps + `ClinicalMemoryService` · S · Low
- 5 tests fail via qhorus `LedgerWriteService` calling old 1-arg ledger API — unblocked by casehub-qhorus SNAPSHOT update (blocked externally, no clinical action)
- ARC42STORIES.MD §9.4 Layer 7+ entries for memory integration not yet added — will come via journal merge at next branch close

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #10 | Trust routing — Layer 7 | L | High | Next layer per harness guide |
| #8 | ClinicalAgent comparison / Layer 8 | M | Med | After trust routing |
| — | #74 — ledger-memory sync | XS | Low | External unblock required |

## References

- Spec: `specs/2026-06-08-casememorystore-design.md`
- Blog: `blog/2026-06-09-mdp01-memory-arrives-with-baggage.md`
- Last commit: `45d467d feat(memory): wire CaseMemoryStore + multi-tenancy foundation`
