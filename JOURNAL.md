# Design Journal — issue-160-live-llm-agent-execution

## 2026-09-11 — Session 1: Design + Batch 1 Foundation

### What happened

Created epic #160 (live LLM agent execution), #161 (real-time event cascade), #162 (regulatory reporting). Brainstormed and designed the agent architecture for #160 — 7 decisions captured, decision review (light) and spec review (light) completed with substantive findings incorporated.

Key design decisions:
- Per-agent SPI pattern (interface + @DefaultBean stub + @ApplicationScoped LLM impl)
- Shared `ClinicalAgentSupport` utility with Jackson parsing and InvocationMetrics capture
- Configurable model/backend per capability via application.properties
- Conservative per-agent fallback policy (D7) — safety agents escalate on failure, not suppress
- InvocationComplete metrics serialized into existing ComplianceSupplement.detail field (no platform schema change)
- Grade 3 SUSAR scope remains deferred (clinical#76)

Spec review caught 7 issues: InvocationComplete has no model field (source from config), ComplianceSupplement is platform-owned (use detail field), eligibility integration gap (new evaluateAndScreen endpoint), SUSAR WorkerResult+PlannedAction mapping, Grade 3 scope boundary, trial supervision uses ClinicalTrialCaseHub.augment() not separate CaseHub, timeout strategy.

### What was built (Batch 1)

**Task 1: ClinicalAgentSupport** — shared utility in `io.casehub.clinical.agent` package. Records: `InvocationMetrics`, `ClinicalAgentRequest<T>`, `ClinicalAgentResult<T>`. `ClinicalAgentSupport.invoke()` handles: config-driven model/timeout resolution, AgentProvider invocation, TextDelta collection, InvocationComplete metrics capture, isError fallback check, JSON fence extraction, Jackson deserialization, exception fallback. 11 unit tests.

**Task 2: Bootstrap + refactor** — Added `casehub-platform-agent-router` + `casehub-platform-agent-claude` runtime dependencies. Added per-capability agent config properties. Refactored `LlmProtocolAmendmentAdvisor` to use `ClinicalAgentSupport` — replaced manual `extractJsonValue()` string parsing with Jackson. Added `ClinicalComplianceSupplement.withMetrics()`. 6 advisor tests updated.

**Pre-existing fix:** Migrated `PlanCbrCase`/`TextualCbrCase` to `FeatureVectorCbrCase` across ~30 files (neocortex SNAPSHOT rename).

### What's next (4 batches remaining)

- Batch 2: Eligibility screening agent (EligibilityCriteriaEvaluator SPI + LLM impl + REST endpoint)
- Batch 3: Safety monitoring agent (LlmSusarCriteriaEvaluator displacing rule-based evaluator)
- Batch 4: DSMB safety signal analysis agent (SafetySignalAnalyzer augmenting TrialSafetyAggregationJob)
- Batch 5: Trial supervision agent (TrialSupervisionAdvisor + ClinicalTrialCaseHub capability binding)

### Architecture note

The `RoutingAgentProvider` → `AgentBackend` discovery pattern (not direct AgentProvider → ClaudeAgentProvider) was confirmed during context gathering. `AgentSessionConfig` has a `model` field for backend routing. `InvocationComplete` does NOT have a model field — model is sourced from the config the caller built. This is documented in the spec and InvocationMetrics record.
