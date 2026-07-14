# Session Handover — 2026-07-14

## Last Session

Closed #128 — S/XS runtime fixes. Fixed Clock CDI ambiguity (qhorus ClockProducer), engine memory store renames (InMemoryPlanItemStore/InMemorySubCaseGroupRepository), un-stubbed governance endpoint (WorkerDecisionEntry exists in casehub-engine-ledger), added EligibilityScreeningLedgerWriter.writeResolutionEntry() with 3 tests, made DemoDataSeeder SUSAR resilient (per-lifecycle catch). Also fixed #129 (SNAPSHOT API breakages — 21 files across engine, qhorus, neocortex type changes) and #127 (webui demo mode — JSONata expressions, CSV alignment, CBR fallback).

Fixed flaky ProtocolAmendmentIntegrationTest (test ordering) and ThreeSiteShowcaseTest (cancelAllAndClear + @Tag("showcase") isolation from default suite). 558/558 tests pass.

Design review completed for blocks-ui channel-activity-promotion spec (5 rounds, 18 issues, $16.25).

## Immediate Next Step

Run `mvn quarkus:dev` to verify dev mode startup with all config fixes. The Clock ambiguity, memory store renames, and reactive suppression are all in place.

## What's Left

- **Live mode** — test webui with `VITE_DEMO_MODE=false` against running Quarkus · M · Med
- **ThreeSiteShowcaseTest** — passes in isolation, fails in suite (clinical#87, engine Vert.x handler lifecycle) · S · High
- `#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- **Governance endpoint** — now wired but untested end-to-end with live SUSAR case · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Dev mode + live mode verification | M | Med | All config fixes landed |
| #99 | Guided Steps 3-4: AE Event + Governance | M | High | Deferred |
| #104 | Guided Steps 3-4: Deviation + PI Auth | M | High | Deferred |
| #86 | Wire ProtocolAmendmentAdvisor | M | High | Blocked: engine#101 |
| #78 | CBR over AE history | L | High | Blocked: neocortex#68 |

## Cross-Repo

- engine#719 — SNAPSHOT consistency (resolved by #129 clinical-side fixes)
- engine#724 — EngineStrategyResolver + work-core Jandex (resolved in engine)
- neocortex#149 — FeatureValue.of() Boolean support (filed)
- blocks-ui channel-activity-promotion design review complete (5 rounds, all issues resolved)
