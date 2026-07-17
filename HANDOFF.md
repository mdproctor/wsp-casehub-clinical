*Updated: neocortex#149 closed — removed from cross-repo.*

# Session Handover — 2026-07-17

## Last Session

Closed #78 — CBR over adverse event history. Replaced `AeResolutionCbrWriter` with `ClinicalCaseOutcomeObserver` (CaseOutcomeObserver SPI) as sole AE CBR writer. Stores `PlanCbrCase` with plan traces from `PlanItemStore`. Added `treatmentArm` + `priorAeCount` features (11 total). Clinical-weighted queries (grade 3.0, eventType 2.5). `AePrecedentResponse` gains plan steps with `workerName`. Design review (14 issues) + code review (6 issues) — all resolved. 575/575 tests. 2 garden entries submitted. Filed casehubio/parent#375 (doc sync) and casehubio/engine#741 (humanTask routing).

## Immediate Next Step

Run `mvn quarkus:dev` and test the webui with `VITE_DEMO_MODE=false` — all config fixes landed and CBR precedent endpoints now return enriched responses with plan steps.

## What's Left

- **Live mode** — test webui against running Quarkus · M · Med
- `#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- **Governance endpoint** — wired but untested end-to-end with live SUSAR case · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Dev mode + live mode verification | M | Med | All config fixes landed |
| #99 | Guided Steps 3-4: AE Event + Governance | M | High | Deferred |
| #104 | Guided Steps 3-4: Deviation + PI Auth | M | High | Deferred |
| #117 | CBR Phase 4: outcome recording + audit trail | M | High | Builds on recordOutcome() |
| #86 | Wire ProtocolAmendmentAdvisor | M | High | Blocked: engine#101 |

## Cross-Repo

- engine#101 — blocks #86 (ProtocolAmendmentAdvisor wiring)
- engine#741 — filed: humanTask routing enrichment via CBR plan traces
- parent#375 — filed: update casehub-clinical.md for CaseOutcomeObserver
