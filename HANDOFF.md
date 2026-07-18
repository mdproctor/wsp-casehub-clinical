# Session Handover — 2026-07-18

## Last Session

Closed #127, #125, #126, #123 on branch `issue-127-webui-s-fixes`. Three of four were already resolved — CSV field alignment (#125), GDPR erasure dialog (#126), and most of #127 (JSONata, CBR demo mode). Real work: CSV column headers aligned to backend record field names across 6 datasets, SiteRow gained `siteName`, and Work Queue migrated from `table()` to `<work-item-inbox>` with WorkIdentity and type-discriminated navigation (#123). Hygiene recovered 1 blog + 3 specs from closed branches. Filed #131 for CDI Clock ambiguity (qhorus SNAPSHOT drift).

## Immediate Next Step

Fix #131 — CDI Clock ambiguity blocks `mvn install` in production mode. Either exclude qhorus `ClockProducer` or remove `ClinicalClockProducer`.

## What's Left

- `#131` — CDI Clock ambiguity (production augmentation failure) · S · Low
- `#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- **Live mode** — test webui against running Quarkus (all datasets now field-aligned) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #131 | Fix CDI Clock ambiguity | S | Low | Blocks production build |
| — | Dev mode + live mode verification | M | Med | All field alignment done |
| #99 | Guided Steps 3-4: AE Event + Governance | M | High | Deferred |
| #104 | Guided Steps 3-4: Deviation + PI Auth | M | High | Deferred |
| #86 | Wire ProtocolAmendmentAdvisor | M | High | Blocked: engine#101 |

## Cross-Repo

- engine#101 — blocks #86 (ProtocolAmendmentAdvisor wiring)
- engine#741 — filed: humanTask routing enrichment via CBR plan traces
- parent#375 — filed: update casehub-clinical.md for CaseOutcomeObserver
