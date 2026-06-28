# Session Handover — 2026-06-28

## Last Session

Branch `issue-93-demo-ui-backend` closed: #93 (epic), #94 (Quinoa scaffold), #95 (DemoDataSeeder), #96 (TrialDashboardResource), #97 (datasets module — deferred to Plan 2), #103 (DemoActionResource + DemoCurrentPrincipal). Java backend for the demo UI is complete — 7 dashboard endpoints, 2 demo action endpoints, data seeder with Merkle-verified service-layer replay, dev-mode tenant context. Spec went through 6 rounds of adversarial review; implementation through 5 per-task reviews + 1 whole-branch review with 2 Important fixes.

Quinoa dependency removed from pom.xml — the deployment processor runs at build time regardless of config, and `npm install` fails because `@casehubio/pages-*` packages aren't published to any registry. Plan 2 (TypeScript UI) re-adds it after packages are linked locally.

## Immediate Next Step

Write Plan 2: TypeScript UI — 14 casehub-pages DSL compositions (8 guided steps + 6 explore pages). Requires linking `@casehubio/pages-runtime` and `@casehubio/pages-ui` from the local casehub-pages repo before Quinoa can build.

## What's Left

- Plan 2: TypeScript UI (casehubio/clinical#93 — UI portion not yet filed as separate issues)
- Plan 3: Playwright smoke tests (casehubio/clinical#102)
- `casehubio/clinical#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- `casehubio/platform#115` — provide MissingTenancyExceptionMapper from casehub-platform-oidc · S · Low
- `casehubio/engine#571` — enrich CaseLifecycleEvent with case context snapshot · M · Med
- Pre-existing flaky tests: `ThreeSiteShowcaseTest`, `RegulatorySubmissionDeadlineLifecycleTest` (×2)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Plan 2: TypeScript UI (casehub-pages DSL) | L | Med | Depends on linking casehub-pages packages |
| #102 | Playwright smoke tests | S | Med | Depends on Plan 2 |
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 |
| #78 | CBR over adverse event history | L | High | Blocked: engine#477, #478, neural-text#20, platform#87 |

## References

- Spec: `specs/2026-06-27-clinical-demo-ui-design.md` (Rev 6 — final)
- Plan 1: `plans/2026-06-27-clinical-demo-ui-backend.md`
- Blog: `blog/2026-06-28-mdp01-the-demo-that-reviews-itself.md`
- casehub-pages epic: casehubio/casehub-pages#50 (11 enhancement issues)
- Garden: GE-20260628-cabe4f (@IfBuildProfile + @Alternative + @Priority technique)
