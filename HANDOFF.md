# HANDOFF — casehub-clinical

## Last Session

Designed and partially implemented #164 (Agent Organization + Model Selection). Completed 4 of 6 tasks: ClinicalNarrativeSignalStrategy (#165), org structure registration (#166), model tier declarations (#167), model registry wiring (#168). Fixed pre-existing SNAPSHOT Clock regression (qhorus ClockProducer vs ClinicalClockProducer). Key CDI learning: don't index casehub-blocks (blast radius) — use manual NarrativeCdiProducer instead.

## Immediate Next Step

Batch 4: Convert Safety/Protocol/Operations workbenches from tree/tabs/columns to dockWorkbench layout and wire 4 blocks-ui panels. Plan at `plans/2026-09-14-agent-org-model-selection.md`, Task 5.

## References

- `specs/issue-164-agent-org-model-selection/2026-09-14-agent-org-model-selection-design.md`
- `plans/2026-09-14-agent-org-model-selection.md`
- `specs/issue-164-agent-org-model-selection/decisions.md`
- `JOURNAL.md`

## Known Flakes

- **PiResponseListenerIntegrationTest** — pre-existing, passes on retry
- **AeEscalationLifecycleTest** — pre-existing async engine lifecycle flake
- **DsmbRollupTest** — pre-existing async engine lifecycle flake
- **CbrRetrievalAuditIntegrationTest** — pre-existing CBR state contamination flake
- **ClinicalCaseOutcomeObserverIntegrationTest** — pre-existing CBR state contamination flake
