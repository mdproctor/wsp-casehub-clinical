# HANDOFF — casehub-clinical

## Last Session

Completed all 6 tasks for #164 (Agent Organization + Model Selection). Batch 4: converted Safety, Protocol, Operations workbenches from columns/tabs to dockWorkbench with registerPanel pattern — wired orchestration-workbench, trust-workbench, conversation-viewer, routing-rationale panels. Added portal resolutions for all casehub-packages to fix transitive dependency failures. Batch 5: created ModelSelectionEvent record, added onModelSelection(@ObservesAsync) to ClinicalNarrativeSignalStrategy emitting MODEL_SELECTED StepOutcome signals, wired CDI event firing from ClinicalAgentSupport after tier-based model resolution.

## Immediate Next Step

All plan tasks complete. Ready for `work end` — code review, squash, merge, close issues #165-#170, close epic #164.

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

## Pre-existing Build Issues

- **webui esbuild** — `emitPagesEvent`/`onPagesEvent` not exported from `pages-component` dist; pre-existing, not caused by this branch
- **webui typecheck** — `DataSetId` brand type errors in `patient-detail.ts`, `trial-detail.ts`, `trial-list.ts`; pre-existing
- **webui datasets.test.ts** — 4 failures related to DEMO_MODE/TRIAL_ID constants; pre-existing
