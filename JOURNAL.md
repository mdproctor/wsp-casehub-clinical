# Design Journal — issue-164-agent-org-model-selection

## 2026-09-14 — Session 1: Design + Backend Implementation

**Completed:** Batches 1–3 (4 of 6 tasks)

### Design
- Brainstormed and specced the full epic (#164) — 5 decisions, all quick-pick following fsitrading #41 reference patterns
- Key decisions: dockWorkbench UI conversion, functional org teams, FLAGSHIP/STANDARD tier split, all 7 case types for narrative signals, REST + WebSocket push for narrative data

### Implementation
- **#165 ClinicalNarrativeSignalStrategy** — `@Alternative @Priority(1)` extending `AbstractNarrativeSignalStrategy`, filters all 7 clinical case types, emits StepOutcome/RoutingDecision/TrustAssessment/CbrRetrieval signals. `NarrativeResource` at `GET /api/narrative/{caseId}` serves cached `DecisionNarrative` list. `NarrativeSignalBroadcaster` pushes to WebSocket topics.
- **#166 ClinicalOrgRegistrar** — functional team hierarchy: Safety Team, Site Team, Regulatory Team under Clinical Trial Operations root. Supervision and escalation edges. Registered in `ClinicalTrialCaseHub.augment()`.
- **#167 ClinicalModelTierResolver** — FLAGSHIP for safety-monitoring, susar-criteria, protocol-amendment, trial-supervision. STANDARD for eligibility-screening. Explicit model config overrides tier. Falls back to "sonnet" when registry empty.
- **#168 Model registry config** — Vertex model source properties (disabled in tests). `NarrativeCdiProducer` manually constructs `DecisionNarrativePipeline` + `EventStreamBus<DecisionSignal>`.

### CDI Issues Resolved
- **Clock ambiguity (pre-existing SNAPSHOT regression):** qhorus SNAPSHOT now ships `ClockProducer` in Jandex-indexed jar. `exclude-types` can't suppress it. Removed clinical's redundant `ClinicalClockProducer`.
- **casehub-blocks indexing blast radius:** `index-dependency` discovers ALL beans including conflicting `ClockProducer` and `LlmAgentRoutingStrategy`. Fixed with manual CDI producers instead.
- **eidos-org runtime transitive deps:** `OrgBootstrap` injects eidos-core SPIs clinical doesn't use. Fixed by using only org-api + org-memory, calling `OrgStructure.define()` directly.

### Remaining (Batches 4–5)
- **#169** — Convert Safety/Protocol/Operations workbenches to dockWorkbench + wire 4 blocks-ui panels (orchestration-workbench, trust-workbench, conversation-viewer, routing-rationale)
- **#170** — `ModelSelectionEvent` + extend `ClinicalNarrativeSignalStrategy` to surface model selection in decision narratives
