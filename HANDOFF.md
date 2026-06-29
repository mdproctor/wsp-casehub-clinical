# Session Handover — 2026-06-29

## Last Session

Dev-mode config alignment (#106) — `mvn quarkus:dev` now starts with H2, all 14 demo pages render live data. Key discovery: `quarkus.arc.selected-alternatives` ignores `%dev.` profile overrides; workaround uses profile-specific `exclude-types`. Also fixed all casehub-pages DSL calling conventions and integrated native `action-button`/`alert` components.

## Immediate Next Step

Pick the next issue from the backlog below. #102 (Playwright smoke tests) is now unblocked since dev mode runs.

## What's Left

- `casehubio/clinical#102` — Playwright smoke tests · S · Med · unblocked by #106
- Missing `/trials/{id}/sites` GET endpoint — Step 1 enrollment chart + sites table need it
- `casehubio/clinical#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- Pre-existing flaky tests: `ThreeSiteShowcaseTest`, `RegulatorySubmissionDeadlineLifecycleTest` (×2)
- Steps 4, 7, 8, audit-trail still use html() scripts via MutationObserver (action-button needs dynamic IDs or GET support)
- DemoDataSeeder SUSAR oversight case start times out — partial demo data

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #102 | Playwright smoke tests | S | Med | Unblocked by #106 |
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 |
| #78 | CBR over adverse event history | L | High | Blocked: foundation |

## References

- Garden: GE-20260629-e6460e (arc selected-alternatives profile override)
- Garden: GE-20260629-59c7e6 (esbuild template minification)
- Garden: GE-20260629-6f1d64 (Maven parent scope override)
- Blog: `blog/2026-06-29-mdp01-the-profile-that-didnt.md`
