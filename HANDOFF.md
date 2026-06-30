# Session Handover — 2026-06-30

## Last Session

Demo UI polish (#102, #108, #109) — sites list endpoint, three custom web components replacing inline scripts + MutationObserver hack, Playwright smoke tests. Design review (8 rounds, $22) improved the spec significantly before implementation. All three issues closed, two follow-up issues filed (#110, #111).

## Immediate Next Step

Pre-existing build failure: `ProtocolAmendmentCaseHub.java` — casehub-engine SNAPSHOT made `YamlCaseHub.getDefinition()` final and removed `capabilities()` from `Worker.Builder`. Fix this before any new work.

## What's Left

- `casehubio/clinical#110` — Java cleanup: redundant HashMap import + weak test assertion · XS · Low
- `casehubio/clinical#111` — Playwright test hardening: error filtering, waitForTimeout, test isolation · S · Low
- `casehubio/clinical#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- Pre-existing flaky tests: `ThreeSiteShowcaseTest`, `RegulatorySubmissionDeadlineLifecycleTest` (×2)
- DemoDataSeeder SUSAR oversight case start times out — partial demo data
- Pre-existing build failure: `ProtocolAmendmentCaseHub.java` — engine SNAPSHOT broke `getDefinition()` override + `capabilities()` API

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Fix ProtocolAmendmentCaseHub build failure | S | Med | Engine SNAPSHOT API change — urgent |
| #110 | Java cleanup (import + assertion) | XS | Low | |
| #111 | Playwright test hardening | S | Low | |
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 |
| #78 | CBR over adverse event history | L | High | Blocked: foundation |

## References

- Spec: `specs/2026-06-29-demo-ui-polish-design.md` (also promoted to project `docs/specs/`)
- Plan: `plans/attic/issue-102-playwright-smoke-tests/2026-06-30-demo-ui-polish.md`
- Blog: `blog/2026-06-30-mdp01-the-mutationobserver-that-shouldnt-have-been.md`
- Garden: GE-20260629-e6460e (arc selected-alternatives profile override)
