# Live LLM Agent Execution via AgentProvider — Design Spec

**Issue:** casehubio/clinical#160
**Date:** 2026-09-11
**Branch:** issue-160-live-llm-agent-execution

## Problem

casehub-clinical is an agentic harness — but no agent actually runs. Every agent decision
is pre-seeded by `DemoDataSeeder`. The platform's `AgentProvider` SPI exists, `ClaudeAgentProvider`
wraps the Claude CLI (Vertex-authenticated), and `LlmProtocolAmendmentAdvisor` is already wired
to call `agentProvider.invoke()` — but `casehub-platform-agent-claude` is not on the classpath,
so `NoOpAgentProvider` is active.

This design activates real LLM agents making clinical decisions governed by the CaseHub platform —
trust routing, oversight gates, and Merkle audit govern real agent outputs, not stubs.

## Architecture Overview

### Agent Execution Path

```
AgentProvider (platform SPI)
  └── RoutingAgentProvider (casehub-platform-agent-router)
        └── ClaudeAgentProvider (casehub-platform-agent-claude, backend key "claude")
              └── ClaudeAgentClient (spring-ai-community claude-agent-sdk)
                    └── Claude CLI subprocess (Vertex-authenticated)
```

### Per-Agent Wiring Pattern (D6)

Each agent follows the `ProtocolAmendmentAdvisor` pattern from issue-86:

```
SPI interface (io.casehub.clinical.api.spi)
  ├── @DefaultBean stub — safe default, active when no LLM available
  └── @ApplicationScoped LLM impl — displaces stub via CDI priority
        └── ClinicalAgentSupport.invoke() — shared prompt/parse/metrics utility
```

### Agent Inventory

| Agent | SPI | Stub default | LLM fallback (D7) | Integration |
|-------|-----|-------------|-------------------|-------------|
| Protocol amendment | `ProtocolAmendmentAdvisor` (exists) | PROCEED | PROCEED | `ProtocolAmendmentCaseHub` worker (exists) |
| Eligibility screening | `EligibilityCriteriaEvaluator` (new) | All MET | All MARGINAL → IRB gate | `EligibilityScreeningService` |
| Safety monitoring | `SusarEvaluatorFunction` (exists) | Rule-based | susarRequired=true → escalate | `ClinicalSusarOversightCaseHub` worker (exists) |
| DSMB analysis | `SafetySignalAnalyzer` (new) | Pass-through | FLAG_FOR_REVIEW → DSMB WorkItem | `TrialSafetyAggregationJob` |
| Trial supervision | `TrialSupervisionAdvisor` (new) | REVIEW_REQUIRED | REVIEW_REQUIRED → WorkItem | `ClinicalTrialCaseHub.augment()` + `trial-coordination.yaml` |

## Section 1: Bootstrap — Dependencies and Configuration

### Dependencies

Add to `runtime/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-agent-router</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-agent-claude</artifactId>
  <scope>runtime</scope>
</dependency>
```

This activates `RoutingAgentProvider` which discovers `ClaudeAgentProvider` (backend key `"claude"`).
The existing `LlmProtocolAmendmentAdvisor` immediately displaces `DefaultProtocolAmendmentAdvisor`
with zero code changes.

### Configuration

Production `application.properties`:

```properties
casehub.platform.agent.default-backend=claude
casehub.clinical.agent.eligibility.model=sonnet
casehub.clinical.agent.eligibility.timeout=PT30S
casehub.clinical.agent.safety.model=sonnet
casehub.clinical.agent.safety.timeout=PT30S
casehub.clinical.agent.dsmb.model=sonnet
casehub.clinical.agent.dsmb.timeout=PT60S
casehub.clinical.agent.supervision.model=sonnet
casehub.clinical.agent.supervision.timeout=PT60S
casehub.clinical.agent.amendment.model=sonnet
casehub.clinical.agent.amendment.timeout=PT30S
```

Each capability reads its model from `casehub.clinical.agent.<key>.model` (D2). The backend
(Claude/OpenAI/Gemini) is platform-configured via `casehub.platform.agent.default-backend`.
Neither model nor backend is hardcoded — deployments configure both per environment.

Test profile: no agent dependencies activated. `@InjectMock AgentProvider` controls all responses.

### Jandex indexing

```properties
quarkus.index-dependency.agent-router.group-id=io.casehub
quarkus.index-dependency.agent-router.artifact-id=casehub-platform-agent-router
quarkus.index-dependency.agent-claude.group-id=io.casehub
quarkus.index-dependency.agent-claude.artifact-id=casehub-platform-agent-claude
```

## Section 2: ClinicalAgentSupport — Shared Utility

`io.casehub.clinical.agent.ClinicalAgentSupport` — `@ApplicationScoped`, injects `AgentProvider`
and `@ConfigMapping` for per-capability model config.

### API

```java
public <T> ClinicalAgentResult<T> invoke(ClinicalAgentRequest<T> request)
```

**`ClinicalAgentRequest<T>`:** system prompt, user prompt, response class (`T`), fallback value,
config key for model lookup, correlation ID.

**`ClinicalAgentResult<T>`:** parsed response (`T`), raw text, `InvocationMetrics`, whether
fallback was used, failure reason.

**`InvocationMetrics`:** Sourced from two places:
- From `AgentSessionConfig` (input): `model` (the model string passed to the config)
- From `AgentEvent.InvocationComplete` (output): `inputTokens`, `outputTokens`,
  `thinkingTokens`, `cacheReadTokens`, `cacheWriteTokens`, `totalCostUsd`, `durationMs`,
  `apiDurationMs`, `sessionId`, `numTurns`, `isError`

Note: `InvocationComplete` does not carry a `model` field — the model is sourced from
the `AgentSessionConfig` the caller built.

### Invoke flow

1. Read model from config (`casehub.clinical.agent.<key>.model`, default `sonnet`)
2. Read timeout from config (`casehub.clinical.agent.<key>.timeout`, default `PT30S`)
3. Build `AgentSessionConfig` with model, timeout, and correlation ID set
4. Call `agentProvider.invoke(config)`
5. Collect `TextDelta` events into response text; capture `InvocationComplete` event
6. Check `InvocationComplete.isError()` — if true, treat as failure → fallback
7. Extract JSON block (handle markdown ` ```json ` fences)
8. Deserialize with Jackson `ObjectMapper.readValue(json, responseClass)`
9. On any failure (timeout, parse error, isError, exception): log at WARN/ERROR, return
   caller's specified fallback value

### Refactor existing LlmProtocolAmendmentAdvisor

Replace manual `extractJsonValue()` string manipulation with `ClinicalAgentSupport.invoke()`.
Same behavior, same test cases, robust Jackson parsing instead of fragile index manipulation.

## Section 3: Eligibility Screening Agent

### SPI

```java
// io.casehub.clinical.api.spi
public interface EligibilityCriteriaEvaluator {
    List<CriterionResult> evaluate(PatientEnrollment enrollment,
                                    List<String> protocolCriteria);
}
```

### DefaultEligibilityCriteriaEvaluator (@DefaultBean)

Returns all criteria as MET. Current behavior for pre-seeded data.

### LlmEligibilityCriteriaEvaluator (@ApplicationScoped)

**System prompt:** Clinical trial eligibility expert. Given patient clinical data and protocol
criteria text, evaluate each criterion. Return structured JSON with `{criterionId, met, marginal,
reasoning}` per criterion. Flag edge cases as MARGINAL rather than EXCLUDED — conservative.

**User prompt context:** Patient demographics, diagnosis, medical history, labs (`LabResult`),
vitals (`VitalSign`), concomitant medications (`ConcomitantMedication`). Protocol
inclusion/exclusion criteria text.

**Response record:**

```java
record EligibilityResponse(List<CriterionEvaluation> criteria) {}
record CriterionEvaluation(String criterionId, boolean met,
                            boolean marginal, String reasoning) {}
```

**Fallback:** All criteria returned as MARGINAL → triggers IRB consultation gate.

**Integration data flow:**

The current REST API (`PatientResource.screen()`) receives `ScreenPatientRequest` containing
pre-evaluated `List<CriterionResult>` and calls `EligibilityScreeningService.screen(enrollment, criteria)`.

New path: `EligibilityScreeningService` gains `evaluateAndScreen(PatientEnrollment, List<String> protocolCriteria)`:
1. Calls `EligibilityCriteriaEvaluator.evaluate(enrollment, protocolCriteria)` → `List<CriterionResult>`
2. Calls existing `screen(enrollment, criteria)` with the evaluated results

`PatientResource` gains a new endpoint `POST /api/patients/{id}/evaluate-and-screen` accepting
`EvaluateScreenRequest(List<String> protocolCriteria)` — protocol criteria text. The existing
`screen` endpoint continues to accept pre-evaluated criteria for backward compatibility.

`LlmEligibilityCriteriaEvaluator` maps `EligibilityResponse` to `List<CriterionResult>`
(existing `io.casehub.clinical.api.model` type) internally. The LLM's `reasoning` per
criterion is captured in the `ComplianceSupplement.detail` field via the `InvocationMetrics`
JSON blob — not lost, but not in the domain record.

`DemoDataSeeder` continues using the direct `screen()` path with hardcoded `CriterionResult`
values — the seeder does not call the LLM evaluator, keeping startup fast and deterministic.

No change to the downstream engine case or IRB gate.

## Section 4: Safety Monitoring Agent (SUSAR Evaluation)

### Existing interface

`SusarEvaluatorFunction` — `Function<Map<String,Object>, WorkerResult<Map<String,Object>>>`.
`SusarCriteriaEvaluator @DefaultBean` implements it with rule-based logic.

### LlmSusarCriteriaEvaluator (@ApplicationScoped)

Displaces `SusarCriteriaEvaluator` automatically via CDI priority.

**System prompt:** Clinical safety scientist specializing in SUSAR assessment per ICH E2A
and 21 CFR 312.32. Evaluate whether an adverse event meets SUSAR criteria: serious,
unexpected, and suspected causal relationship to study drug. Consider the full clinical
context — not just grade thresholds. Reason about causality (temporal relationship,
dose-response, dechallenge/rechallenge, known drug class effects) and expectedness
(against the Investigator's Brochure reference safety profile).

**User prompt context:** AE fields (grade, eventType, unexpected, suspected, actuality,
outcome, occurredAt, reportedAt). Patient context (enrollment, concomitant meds, study
drug administrations, prior AEs, vitals). Trial context (phase, protocol, drug class).

**Response record:**

```java
record SusarAssessmentResponse(boolean susarRequired,
                                String causalityAssessment,
                                String expectednessAssessment,
                                String reasoning,
                                float confidence) {}
```

**Fallback:** `susarRequired = true` → escalates to human safety officer review.

**Integration:** `LlmSusarCriteriaEvaluator` implements `SusarEvaluatorFunction`
(`Function<Map<String,Object>, WorkerResult<Map<String,Object>>>`). The `apply()` method:

1. Extracts `aeId` from the input map (same as existing `SusarCriteriaEvaluator`)
2. Loads `AdverseEvent` entity from DB (`@Transactional`), plus related patient context
3. Builds user prompt from the loaded clinical data
4. Calls `ClinicalAgentSupport.invoke()` → `SusarAssessmentResponse`
5. Maps response to `WorkerResult`:
   - Output map: `{"susarRequired": response.susarRequired(), "susarAssessmentComplete": true,
     "causalityAssessment": response.causalityAssessment(), "reasoning": response.reasoning()}`
   - When `susarRequired == true`: creates `PlannedAction.of(ClinicalActionType.SUSAR_CRITERIA_DECISION.reason(),
     ClinicalActionType.SUSAR_CRITERIA_DECISION.actionType(), actionCtx)` — same as existing evaluator.
     This triggers the oversight gate requiring qualified-investigator sign-off.
   - When `susarRequired == false`: returns `WorkerResult.of(outputMap)` with no `PlannedAction`

`ClinicalSusarOversightCaseHub` worker lambda calls the function unchanged.

**Value over rule-based:** The LLM adds causality reasoning (temporal relationship, dose-response,
dechallenge/rechallenge) and expectedness assessment beyond threshold rules. Maintains the same
Grade 4/5 scope boundary as the rule-based evaluator — Grade 3 unexpected AEs remain deferred
(clinical#76). The LLM does not expand regulatory scope.

## Section 5: DSMB Safety Signal Analysis Agent

### SPI

```java
// io.casehub.clinical.api.spi
public interface SafetySignalAnalyzer {
    SignalAnalysis analyze(TrialSafetyContext context);
}
```

### DefaultSafetySignalAnalyzer (@DefaultBean)

Returns raw rule-based signals with no additional analysis.

### LlmSafetySignalAnalyzer (@ApplicationScoped)

**System prompt:** Independent data safety monitoring analyst. You receive a summary of
adverse events across trial sites with rule-based signal detections. Assess whether
detected signals represent genuine safety concerns. Identify subtler patterns the rules
may have missed: temporal clustering, dose-response relationships, organ-system crossover.
Provide narrative assessment suitable for DSMB review. You are advisory — the DSMB makes
all binding decisions.

**User prompt context:** Per-site AE summaries (event type, grade distribution, counts, rates),
detected rule-based signals, trial phase and protocol, historical CBR cases for pattern
context, enrollment trajectory data.

**Response record:**

```java
record SignalAnalysis(List<AnalyzedSignal> signals,
                      String overallAssessment,
                      List<String> additionalPatterns) {}
record AnalyzedSignal(String signalType, SignalSeverity severity,
                      List<String> affectedSites, String narrative,
                      String recommendedAction) {}
enum SignalSeverity { LOW, MODERATE, HIGH, CRITICAL }
```

**Fallback:** FLAG_FOR_REVIEW — all rule-based signals passed to DSMB WorkItem with note
that LLM analysis was unavailable.

**Integration:** Called inside `TrialSafetyAggregationJob` after rule-based detection
completes. LLM analysis enriches `TrialSafetySignal` records with narrative before
`WorkItem` creation. Rule-based signals always fire regardless of LLM availability —
the LLM augments, never replaces. `InvocationComplete` metrics captured and included
in `ComplianceSupplement` attached to the DSMB signal ledger entry.

## Section 6: Trial Supervision Agent

### SPI

```java
// io.casehub.clinical.api.spi
public interface TrialSupervisionAdvisor {
    SupervisionAssessment assess(TrialSupervisionContext context);
}
```

### DefaultTrialSupervisionAdvisor (@DefaultBean)

Returns REVIEW_REQUIRED with no analysis — forces human review by default.

### LlmTrialSupervisionAdvisor (@ApplicationScoped)

**System prompt:** Clinical trial operations advisor. Assess trial-wide operational health
across all sites. Identify: enrollment trajectory anomalies (sites falling behind target),
protocol adherence degradation (rising deviation rates), site performance outliers (one site
with disproportionate AE rates or slow responses), cross-site patterns that warrant sponsor
attention. Recommend concrete operational interventions. You are advisory — operations teams
and PIs make binding decisions.

**User prompt context:** Per-site metrics (enrollment count vs target, AE rates, deviation
counts, PI response times), amendment history, active safety signals, trial phase timeline,
CBR precedent cases for similar trial profiles.

**Response record:**

```java
record SupervisionAssessment(TrialHealth overallHealth,
                              List<SupervisionFinding> findings,
                              String summary) {}
record SupervisionFinding(String findingType, List<String> affectedSites,
                           FindingSeverity severity, String narrative,
                           String recommendedAction) {}
enum TrialHealth { HEALTHY, WATCH, CONCERN, ACTION_REQUIRED }
enum FindingSeverity { LOW, MODERATE, HIGH, CRITICAL }
```

**Fallback:** REVIEW_REQUIRED → creates WorkItem for PI/operations review.

**Integration:** New capability binding added to `trial-coordination.yaml` alongside the
existing DSMB humanTask. Fires on `contextChange` trigger when trial-wide safety metrics
update (after `TrialSafetyAggregationJob` runs or enrollment changes). The worker is
registered via `ClinicalTrialCaseHub.augment()` — not a separate CaseHub class. This
avoids creating a duplicate `CaseInstance` per trial. The worker registration follows
`Worker.builder().function()` with capability `trial-supervision`.

## Section 7: Testing Strategy

### Unit tests (no Quarkus, no DB)

Mock `AgentProvider` via Mockito. One test class per agent:

- `ClinicalAgentSupportTest` — JSON extraction, Jackson parsing edge cases, markdown fence handling
- `LlmEligibilityCriteriaEvaluatorTest` — criterion evaluation, prompt verification
- `LlmSusarCriteriaEvaluatorTest` — SUSAR assessment, causality reasoning
- `LlmSafetySignalAnalyzerTest` — signal analysis, pattern detection
- `LlmTrialSupervisionAdvisorTest` — supervision assessment, finding generation

Each test class verifies:
- Prompt contains expected domain context
- Valid JSON response → correct parsed result
- Malformed JSON → domain-appropriate fallback (per D7)
- Empty response (NoOp) → fallback
- `InvocationComplete` metrics captured in result

### Integration tests (@QuarkusTest with @InjectMock AgentProvider)

Verify:
- CDI displacement: injected SPI is the LLM implementation, not the stub
- End-to-end: trigger → agent called → result flows through engine/ledger pipeline
- `ComplianceSupplement` includes `InvocationComplete` metrics

### Fallback escalation trade-off (D7)

Conservative fallbacks may create false escalations when the LLM is unavailable — eligibility
candidates routed to IRB unnecessarily, AEs escalated to safety officers that don't warrant
review. This is the correct trade-off for a clinical system: false escalations cost human
review time; false suppressions risk patient safety. Tests verify both the happy path (LLM
available, correct response) and the fallback path (LLM unavailable, conservative escalation).

### No real LLM calls in tests

`@InjectMock AgentProvider` controls all responses. Mock returns pre-crafted
`Multi.createFrom().items(TextDelta(...), InvocationComplete(...))` sequences.
Fast, deterministic, CI-friendly.

### Existing test updates

`LlmProtocolAmendmentAdvisor` tests updated to verify `ClinicalAgentSupport` refactor —
same test cases, different internal implementation.

## Audit Trail (D5)

`ClinicalAgentSupport` captures `InvocationComplete` metrics in `ClinicalAgentResult.metrics()`.

Ledger writers serialize `InvocationMetrics` as JSON into the existing `ComplianceSupplement.detail`
field (a platform-owned String). `ClinicalComplianceSupplement` factory methods gain an overload
accepting `InvocationMetrics` — the `detail` field carries serialized JSON like:
`{"model":"sonnet","inputTokens":1234,"outputTokens":567,"totalCostUsd":0.003,"durationMs":2100}`.

No new fields on `ComplianceSupplement` (platform-owned schema). No new `LedgerEntry` subclass
or migration. The `algorithmRef` field continues to identify the agent implementation class
(e.g., `"LlmSusarCriteriaEvaluator"`).

This satisfies EU AI Act Art.12 record-keeping: every AI agent decision has a traceable
record of which model ran, how much compute it used, what it cost, and how long it took.

## Files Changed

### New files

| File | Package | Purpose |
|------|---------|---------|
| `ClinicalAgentSupport.java` | `io.casehub.clinical.agent` | Shared invoke/parse/metrics utility |
| `ClinicalAgentRequest.java` | `io.casehub.clinical.agent` | Request record |
| `ClinicalAgentResult.java` | `io.casehub.clinical.agent` | Result record with InvocationMetrics |
| `InvocationMetrics.java` | `io.casehub.clinical.agent` | Metrics from InvocationComplete |
| `ClinicalAgentConfig.java` | `io.casehub.clinical.agent` | @ConfigMapping for per-capability model |
| `EligibilityCriteriaEvaluator.java` | `io.casehub.clinical.api.spi` | SPI interface |
| `DefaultEligibilityCriteriaEvaluator.java` | `io.casehub.clinical.service` | @DefaultBean stub |
| `LlmEligibilityCriteriaEvaluator.java` | `io.casehub.clinical.service` | LLM implementation |
| `EligibilityResponse.java` | `io.casehub.clinical.agent` | Response record |
| `LlmSusarCriteriaEvaluator.java` | `io.casehub.clinical.service` | LLM SUSAR evaluator |
| `SusarAssessmentResponse.java` | `io.casehub.clinical.agent` | Response record |
| `SafetySignalAnalyzer.java` | `io.casehub.clinical.api.spi` | SPI interface |
| `DefaultSafetySignalAnalyzer.java` | `io.casehub.clinical.service` | @DefaultBean stub |
| `LlmSafetySignalAnalyzer.java` | `io.casehub.clinical.service` | LLM implementation |
| `SignalAnalysis.java` | `io.casehub.clinical.agent` | Response records |
| `TrialSupervisionAdvisor.java` | `io.casehub.clinical.api.spi` | SPI interface |
| `DefaultTrialSupervisionAdvisor.java` | `io.casehub.clinical.service` | @DefaultBean stub |
| `LlmTrialSupervisionAdvisor.java` | `io.casehub.clinical.service` | LLM implementation |
| `SupervisionAssessment.java` | `io.casehub.clinical.agent` | Response records |
| `TrialSafetyContext.java` | `io.casehub.clinical.api.spi` | Input record for SafetySignalAnalyzer |
| `TrialSupervisionContext.java` | `io.casehub.clinical.api.spi` | Input record for TrialSupervisionAdvisor |
| `EvaluateScreenRequest.java` | `io.casehub.clinical.api.model` | REST request for evaluate-and-screen endpoint |

### Modified files

| File | Change |
|------|--------|
| `runtime/pom.xml` | Add agent-router + agent-claude dependencies |
| `application.properties` | Agent config properties, Jandex indexing |
| `LlmProtocolAmendmentAdvisor.java` | Refactor to use ClinicalAgentSupport |
| `EligibilityScreeningService.java` | Add evaluateAndScreen() method calling SPI |
| `PatientResource.java` | Add POST /api/patients/{id}/evaluate-and-screen endpoint |
| `TrialSafetyAggregationJob.java` | Call SafetySignalAnalyzer after rule-based detection |
| `trial-coordination.yaml` | Add trial-supervision capability binding |
| `ClinicalTrialCaseHub.java` | Register trial-supervision worker in augment() |
| `ClinicalComplianceSupplement.java` | Add overload accepting InvocationMetrics for detail field |

### Test files (new)

| File | Type |
|------|------|
| `ClinicalAgentSupportTest.java` | Unit |
| `LlmEligibilityCriteriaEvaluatorTest.java` | Unit |
| `LlmSusarCriteriaEvaluatorTest.java` | Unit |
| `LlmSafetySignalAnalyzerTest.java` | Unit |
| `LlmTrialSupervisionAdvisorTest.java` | Unit |
| `EligibilityAgentIntegrationTest.java` | Integration |
| `SusarAgentIntegrationTest.java` | Integration |
| `DsmbAgentIntegrationTest.java` | Integration |
| `TrialSupervisionIntegrationTest.java` | Integration |

### No new migrations

No database schema changes. Agent responses stored in existing case context fields.
ComplianceSupplement metrics stored as JSON in existing ledger entries.

## References

- [issue-86 spec](/Users/mdproctor/claude/casehub/clinical/docs/specs/issue-86-protocol-amendment-llm/2026-07-30-protocol-amendment-llm-advisor-design.md) — established the AgentProvider integration pattern
- [AgentProvider SPI](casehub-platform-agent-api) — `invoke(AgentSessionConfig)` → `Multi<AgentEvent>`
- [AgentEvent.InvocationComplete](casehub-platform-agent-api) — per-invocation metrics
- [ClaudeAgentProvider](casehub-platform-agent-claude) — Claude CLI wrapper with Vertex auth
- [SusarCriteriaEvaluator](runtime/src/main/java/.../service/SusarCriteriaEvaluator.java) — existing rule-based evaluator to displace
- [TrialSafetyAggregationJob](runtime/src/main/java/.../cbr/TrialSafetyAggregationJob.java) — DSMB integration point
- [trial-coordination.yaml](runtime/src/main/resources/.../trial-coordination.yaml) — trial supervision binding target
- ICH E2A — SUSAR assessment criteria
- 21 CFR 312.32 — FDA expedited safety reporting
- EU AI Act Art.12 — AI system record-keeping requirements
- Decision review R1-02, R1-03, R1-09 — Jackson parsing, InvocationComplete audit, per-agent fallback
- Spec review R1-02 — InvocationComplete has no model field; source from AgentSessionConfig
- Spec review R1-03 — ComplianceSupplement is platform-owned; use existing detail field
- Spec review R1-04 — Eligibility integration gap; new evaluateAndScreen() path
- Spec review R1-05 — SUSAR WorkerResult mapping + PlannedAction must be explicit
- Spec review R1-06 — Grade 3 scope deferred (clinical#76); LLM maintains Grade 4/5 boundary
- Spec review R1-08 — Trial supervision uses ClinicalTrialCaseHub.augment(), not separate CaseHub
