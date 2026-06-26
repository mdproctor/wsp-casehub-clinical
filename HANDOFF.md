# Session Handover — 2026-06-26

## Last Session

Branch `issue-92-s-xs-cleanup-batch` closed: #92 (worker-api migration already done — fixed one stale Javadoc @link), #91 (repository_dispatch trigger added to publish.yml), #90 (removed CurrentPrincipal exclude-types after verifying platform#111 shipped @Alternative @Priority(100) on OidcCurrentPrincipal from bytecode). CI now auto-rebuilds on upstream SNAPSHOT publishes.

## Immediate Next Step

Pick next issue. Check open issues on casehubio/clinical. #86 (LlmPlanningStrategy) and #78 (CBR) are both blocked on foundation work.

## What's Left

- `casehubio/clinical#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- `casehubio/platform#115` — provide MissingTenancyExceptionMapper from casehub-platform-oidc · S · Low
- `casehubio/engine#571` — enrich CaseLifecycleEvent with case context snapshot · M · Med
- Pre-existing flaky tests: `ThreeSiteShowcaseTest`, `RegulatorySubmissionDeadlineLifecycleTest` (×2) — CaseInstance tenant mismatch in async paths

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 |
| #78 | CBR over adverse event history | L | High | Blocked: engine#477, #478, neural-text#20, platform#87 |

## References

- Blog: `blog/2026-06-26-mdp01-the-exception-that-rolls-back.md` (previous session)
- Follow-ups: platform#115, engine#571
