# Session Handover — 2026-06-29

## Last Session

Two branches closed this session:

**Plan 1 backend (#93):** 7 dashboard endpoints (TrialViewResource), 2 demo action endpoints (DemoActionResource), DemoCurrentPrincipal, DemoDataSeeder. 6 rounds of adversarial spec review, 5 per-task reviews + whole-branch review.

**Plan 2 TypeScript UI (#105):** 14 casehub-pages DSL page compositions — 8 guided walkthrough steps (trial overview, AI agents, deviation + PI auth, AE event + governance hero layout, resolution + Merkle proof) + 6 explore dashboard pages. Linked casehub-pages packages locally via `file:` protocol, re-added Quinoa.

**Blocked on #106:** `mvn quarkus:dev` fails with CDI errors. The project has never run in dev mode — only via `@QuarkusTest`. Dev mode needs the same CDI exclusions, Quartz RAM config, Flyway disabling, and reactive suppression as the test config. Three errors fixed (OidcCurrentPrincipal, connectors, Quartz); more remain (Mutiny.SessionFactory, possibly others).

## Immediate Next Step

**#106 is the priority.** Align `%dev.` profile in `application.properties` with the test config. Systematic — read both configs, diff them, apply all `%dev.` equivalents in one pass. Once dev mode starts, the demo UI is visible at `http://localhost:8080`.

## What's Left

- `casehubio/clinical#106` — dev-mode config alignment · S · Med · **PRIORITY — blocks seeing the demo**
- Plan 3: Playwright smoke tests (#102) · S · Med · depends on #106
- Missing `/trials/{id}/sites` endpoint — Step 1 enrollment chart + sites table are TODOs
- `casehubio/clinical#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- Pre-existing flaky tests: `ThreeSiteShowcaseTest`, `RegulatorySubmissionDeadlineLifecycleTest` (×2)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #106 | Dev-mode config alignment | S | Med | **PRIORITY** — blocks demo |
| #102 | Playwright smoke tests | S | Med | Depends on #106 |
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 |
| #78 | CBR over adverse event history | L | High | Blocked: foundation |

## References

- Spec: `specs/2026-06-27-clinical-demo-ui-design.md` (Rev 6 — final)
- Plan 1: `plans/2026-06-27-clinical-demo-ui-backend.md`
- Plan 2: `plans/2026-06-29-clinical-demo-ui-typescript.md`
- Blog: `blog/2026-06-28-mdp01-the-demo-that-reviews-itself.md`
- casehub-pages epic: casehubio/casehub-pages#50
- Garden: GE-20260601-ad6203 revised (ArC @Alternative validation variant)
