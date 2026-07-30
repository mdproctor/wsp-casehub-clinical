# Protocol Amendment LLM Advisor — Design Spec

**Issue:** casehubio/clinical#86
**Date:** 2026-07-30
**Branch:** issue-86-protocol-amendment-llm

## Problem

`ProtocolAmendmentAdvisor` SPI has a stub implementation (`DefaultProtocolAmendmentAdvisor`)
that always returns PROCEED. The stub was introduced in Layer 9 (#10 showcase) as a
placeholder. Now that `AgentProvider` (casehub-platform-agent-api) is available, the
advisor can be backed by an LLM to produce meaningful recommendations based on trial context.

## Design

### Context Enrichment

`ProtocolAmendmentCaseHub` worker lambda populates `trialBlackboardSnapshot` before
constructing `ProtocolAmendmentContext`. Currently passes `Map.of()`.

**Snapshot contents:**

| Key | Type | Source |
|-----|------|--------|
| `trialPhase` | String | `ClinicalTrial.phase.name()` |
| `trialStatus` | String | `ClinicalTrial.status.name()` |
| `sponsor` | String | `ClinicalTrial.sponsor` |
| `totalAdverseEvents` | long | Count of all AEs across trial sites |
| `grade3PlusCount` | long | Count of Grade 3/4/5 AEs |
| `hasGrade5` | boolean | Any Grade 5 AE exists |
| `priorAmendmentCount` | int | Count of prior `ProtocolAmendment` for this trial |

Query path: `trialId → TrialSite.trialId → PatientEnrollment.siteId → AdverseEvent.enrollmentId`.

No API change — `ProtocolAmendmentContext.trialBlackboardSnapshot` is already `Map<String, Object>`.

### LlmProtocolAmendmentAdvisor

New class: `io.casehub.clinical.service.LlmProtocolAmendmentAdvisor`

- `@ApplicationScoped` (not `@DefaultBean`) — displaces `DefaultProtocolAmendmentAdvisor` automatically
- Injects `AgentProvider`
- Implements `ProtocolAmendmentAdvisor.advise(ProtocolAmendmentContext)`

**Flow:**

1. Build system prompt — clinical domain expert for protocol amendment analysis (GCP/ICH-aware)
2. Build user prompt from `ProtocolAmendmentContext`:
   - Proposed change text
   - Trial phase and status
   - AE safety summary (total, Grade 3+, Grade 5 presence)
   - Prior amendment count
3. Call `agentProvider.invoke(AgentSessionConfig.of(systemPrompt, userPrompt))`
4. Collect `TextDelta` events into a string
5. Parse JSON response: `{"recommendation": "REFER_TO_DSMB", "reasoning": "..."}`
6. Map to `AmendmentRecommendation` enum value

**System prompt guidance for the LLM:**

- Return exactly one of: `PROCEED`, `REFER_TO_DSMB`, `HALT`
- HALT: Grade 5 AEs present, proposed change impacts safety endpoints, or trial integrity at risk
- REFER_TO_DSMB: Grade 4 rate elevated, proposed change affects safety monitoring, cross-site safety signals
- PROCEED: administrative changes, low safety signal burden, early-phase trials with expected AE profiles

**Fallback chain:**

| Condition | Result | Log level |
|-----------|--------|-----------|
| `AgentProvider` returns empty (NoOp active) | PROCEED | WARN |
| LLM response not parseable as JSON | PROCEED | WARN (includes raw response) |
| Recommendation value not in enum | PROCEED | WARN |
| Exception during invocation | PROCEED | ERROR |

PROCEED is the safe clinical default — a failed advisor should not block amendment processing.

### CDI Displacement

`LlmProtocolAmendmentAdvisor @ApplicationScoped` displaces `DefaultProtocolAmendmentAdvisor @DefaultBean`
by standard Quarkus ArC priority. No `selected-alternatives` entry needed. The stub remains for
environments without `AgentProvider` on the classpath — it is never removed.

In tests, `@InjectMock AgentProvider` controls LLM responses without displacing the advisor itself.

## Testing

### Unit tests — `LlmProtocolAmendmentAdvisorTest`

Mock `AgentProvider`. No Quarkus, no DB.

| Test | Input | Expected |
|------|-------|----------|
| Valid PROCEED response | `{"recommendation": "PROCEED", "reasoning": "..."}` | PROCEED |
| Valid REFER_TO_DSMB response | `{"recommendation": "REFER_TO_DSMB", "reasoning": "..."}` | REFER_TO_DSMB |
| Valid HALT response | `{"recommendation": "HALT", "reasoning": "..."}` | HALT |
| Empty response (NoOp) | `Multi.empty()` | PROCEED (fallback) |
| Malformed JSON | `"not json"` | PROCEED (fallback) |
| Unknown recommendation | `{"recommendation": "UNKNOWN", ...}` | PROCEED (fallback) |
| Prompt contains context | any | Assert prompt includes trial phase, AE counts, proposed change |

### Integration test — `ProtocolAmendmentAdvisorIntegrationTest`

`@QuarkusTest` with `@InjectMock AgentProvider`.

| Test | What it verifies |
|------|------------------|
| CDI displacement | Injected `ProtocolAmendmentAdvisor` is `LlmProtocolAmendmentAdvisor` |
| Enriched context | Worker lambda populates snapshot with trial phase, AE counts |
| End-to-end | Propose amendment → case starts → advisor called → recommendation recorded |

## Files Changed

| File | Change |
|------|--------|
| `LlmProtocolAmendmentAdvisor.java` | **New** |
| `ProtocolAmendmentCaseHub.java` | Enrich `trialBlackboardSnapshot` in worker lambda |
| `ProtocolAmendmentAdvisor.java` | Update javadoc |
| `DefaultProtocolAmendmentAdvisor.java` | Update javadoc |
| `LlmProtocolAmendmentAdvisorTest.java` | **New** |
| `ProtocolAmendmentAdvisorIntegrationTest.java` | **New** |

No new dependencies. No migrations. No API changes.
