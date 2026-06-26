# Session Handover — 2026-06-26

## Last Session

Branch `issue-89-tenancy-perf-gdpr` closed: #89 MissingTenancyExceptionMapper (HTTP 400), #87 listener TX fix (remove @Transactional from 3 observers, REQUIRES_NEW on ledger writers, extract ProtocolAmendmentStatusUpdater), #79 GDPR Art.17 patient-scoped erasure (idempotent withdraw() with WithdrawalResult, erasure receipts enabled, GdprErasureResource at DELETE /api/gdpr/erasure/patients/{patientId}). Fixed pre-existing worker-api migration (engine#543). Garden entry GE-20260626-aa69fa (JTA REQUIRED + RuntimeException rollback gotcha). Filed platform#115 and engine#571.

## Immediate Next Step

Pick next issue. Check open issues on casehubio/clinical.

## What's Left

- `casehubio/clinical#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- `casehubio/platform#115` — provide MissingTenancyExceptionMapper from casehub-platform-oidc · S · Low
- `casehubio/engine#571` — enrich CaseLifecycleEvent with case context snapshot · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 |

## References

- Blog: `blog/2026-06-26-mdp01-the-exception-that-rolls-back.md`
- Spec: `docs/specs/2026-06-25-tenancy-perf-gdpr-design.md` (project)
- Garden: GE-20260626-aa69fa (JTA rollback gotcha)
- Follow-ups: platform#115, engine#571
