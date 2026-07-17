# CBR Phase 4 — Retrieval Audit Trail + Clinical Explanation

**Issue:** casehubio/clinical#117
**Epic:** casehubio/clinical#115 (CBR roadmap)
**Date:** 2026-07-17
**Depends on:** #116 (Phase 1 — closed), #78 (CaseOutcomeObserver — closed)

## Scope

Two deliverables remain from #117 — outcome recording was completed in #78.

1. **Retrieval traceability** — every CBR precedent consultation that could influence a clinical decision produces a tamper-evident ledger entry recording what was queried, what was retrieved, and why.
2. **Clinical explanation renderer** — FDA-structured "why this precedent" text for audit and clinician transparency.

## Out of Scope

- Outcome recording — complete in #78 (`ClinicalCaseOutcomeObserver` + `CbrCaseMemoryStore.recordOutcome()`)
- Changes to neocortex SPIs — all required SPIs exist (`CbrRetrievalTrace`, `ExplanationRenderer`, `CbrOutcome`, `CbrCaseMemoryStore.recordOutcome()`)
- CBR retrieval logic — stays in neocortex per boundary rules

## Design Principles

- **Factor responsibilities, don't pile them.** Retrieval stays pure. Explanation rendering is a CDI-displaced SPI implementation. Ledger writing follows the existing writer pattern. Composition is a thin orchestration method.
- **Reuse existing infrastructure.** `ClinicalCbrService.retrieveSimilar()` untouched — new `retrieveWithAudit()` composes on top. `ExplanationRenderer` SPI from neocortex displaced via CDI, not reimplemented.
- **Deterministic explanations.** No LLM involvement in explanation rendering — structured text assembly from `CbrRetrievalTrace` data. Reproducible and itself not requiring audit.

## Architecture

### Component Responsibilities

| Component | Responsibility | Package |
|---|---|---|
| `ClinicalExplanationRenderer` | FDA-structured explanation text from `CbrRetrievalTrace` | `io.casehub.clinical.cbr` |
| `CbrRetrievalLedgerEntry` | Tamper-evident record of a precedent consultation | `io.casehub.clinical.ledger` |
| `CbrRetrievalLedgerWriter` | Constructs and persists the ledger entry | `io.casehub.clinical.service` |
| `ClinicalCbrService.retrieveWithAudit()` | Composition: retrieve → trace → explain → audit → return | `io.casehub.clinical.cbr` |
| `AuditedRetrievalResult<C>` | Return type carrying cases + traceId + explanation | `io.casehub.clinical.cbr` |
| Response wrappers | REST response DTOs with traceId + explanation + precedent list | `io.casehub.clinical.api.model` |

### Pipeline

```
CbrQuery
  → ClinicalCbrService.retrieveWithAudit()
    → retrieveSimilar(query, caseType)           // existing, untouched
    → buildTrace(query, results)                  // generates traceId (UUID) + timestamp (Clock)
    → explanationRenderer.render(trace)           // CDI-injected ExplanationRenderer
    → ledgerWriter.record(trace, explanation, …)  // REQUIRES_NEW ledger write
    → return AuditedRetrievalResult(cases, traceId, explanation)
```

#### Failure Handling

1. **`render()` failure:** Catch the exception, set `explanation = null`, continue to `record()`. The retrieval happened and must be audited even without explanation text. Log a warning.
2. **`record()` failure:** Propagate the exception — the caller does **not** receive results. Returning unaudited AI decision-support results is a compliance violation (EU AI Act Art.12). The retrieval is lost but the alternative (unaudited results in production) is worse.
3. **Outer TX rollback after `record()` commits:** The ledger entry persists (`REQUIRES_NEW` already committed). This is intentional — the AI system queried the case base, and that fact is audit-worthy regardless of whether the caller consumed the result.

This ensures every retrieval attempt that progresses past `buildTrace()` produces exactly one ledger entry. `buildTrace()` failure (UUID/Clock infrastructure error) propagates naturally — no trace means nothing meaningful to audit.

#### Empty Results

When `retrieveSimilar()` returns an empty list, the full pipeline still executes: trace records zero results, renderer produces appropriate text ("0 prior cases retrieved"), writer creates a ledger entry recording the empty consultation, and the caller receives `AuditedRetrievalResult(emptyList, traceId, explanation)`. An empty retrieval is still a consultation worth auditing — it records that the AI system was queried and found no relevant precedents. The renderer handles zero results without divide-by-zero on score statistics or NPE on "top precedent" fields.

### Data Model

#### `CbrRetrievalLedgerEntry extends JpaLedgerEntry`

| Field | Type | Purpose |
|---|---|---|
| `traceId` | `String` | UUID correlating ledger entry with REST response |
| `queryDomain` | `String` | "clinical-ae", "clinical-deviation", or "clinical-amendment" |
| `queryFeaturesSummary` | `String` | Compact query features: "grade=3,eventType=Neutropenia,trialPhase=PHASE_III" |
| `retrievedCaseCount` | `int` | Number of precedents returned |
| `topScore` | `double` | Highest similarity score in result set |
| `explanationText` | `String` (VARCHAR(10000)) | Full FDA explanation from `ClinicalExplanationRenderer` (nullable — null when render fails) |

- `@DiscriminatorValue("CBR_RETRIEVAL")` on the JPA entity
- `subjectId`: the entity being queried (aeId / deviationId / amendmentId)
- `actorId`: the human or agent who triggered the consultation
- `domainContentBytes()` field order (load-bearing for Merkle chain integrity):
  ```java
  String.join("|", traceId, queryDomain, queryFeaturesSummary,
      String.valueOf(retrievedCaseCount), String.valueOf(topScore),
      explanationText != null ? explanationText : "")
      .getBytes(StandardCharsets.UTF_8)
  ```
  Including `explanationText` in the hash means the Merkle chain covers the rendered explanation — any post-hoc modification breaks the chain. This requires explanation text to be deterministic (guaranteed by the structured text assembly design).
- Flyway V2029 on qhorus datasource — uses `VARCHAR(10000)` for `explanation_text` to follow existing migration patterns (all clinical ledger migrations use `VARCHAR(N)`, not `TEXT`, for H2 compatibility in `@QuarkusTest`)

#### `AuditedRetrievalResult<C extends CbrCase>`

```java
public record AuditedRetrievalResult<C extends CbrCase>(
    List<ScoredCbrCase<C>> cases,
    String traceId,
    String explanation) {}
```

#### Response Wrappers

```java
public record AePrecedentSearchResponse(
    String traceId, String explanation,
    List<AePrecedentResponse> precedents) {}

public record DeviationPrecedentSearchResponse(
    String traceId, String explanation,
    List<DeviationPrecedentResponse> precedents) {}

public record AmendmentPrecedentSearchResponse(
    String traceId, String explanation,
    List<AmendmentPrecedentResponse> precedents) {}
```

### ClinicalExplanationRenderer

`@ApplicationScoped`, displaces `DefaultExplanationRenderer` (`@DefaultBean`).

Reads `trace.query().domain().name()` to branch on AE / deviation / amendment. Differences are minor — feature name labelling.

Output structure (AE example):
```
Adverse event precedent consultation: 4 prior cases retrieved (min similarity 0.30).
Top precedent: score 0.92. Feature alignment: grade=1.00, eventType=0.95, trialPhase=0.80.
Confidence band: 3 of 4 precedents have confidence >= 0.70. 1 has no recorded confidence.
Query domain: clinical-ae. Retrieval mode: HYBRID.
```

Computes score distribution and confidence band statistics from `TracedCase` fields across the result set. `TracedCase.confidence()` is a nullable `Double` — a trust indicator updated by `CbrOutcome.adjustConfidence()`, not an outcome status. `null` means no outcome was ever recorded. The renderer does not report outcome statistics because `TracedCase` does not carry outcome data (only the underlying `CbrCase` does, and the renderer SPI receives `CbrRetrievalTrace`, not the case objects).

### CbrRetrievalLedgerWriter

`@ApplicationScoped` in `io.casehub.clinical.service`. Follows the observer/isolation variant of the ledger writer pattern — the same `REQUIRES_NEW` semantics used by `DeviationLedgerWriter.writeSponsorNotifiedEntry()`, `AeEscalationLedgerWriter.writeCompletionEntry()`, and other observer-path writes. This is distinct from the primary writer pattern (`writeCommandEntry`, `writeReportEntry`) which participates in the caller's transaction.

```java
@Transactional(TxType.REQUIRES_NEW)
public void record(CbrRetrievalTrace trace, String explanation,
                   UUID subjectId, String actorId, String tenantId)
```

- Constructs `CbrRetrievalLedgerEntry`
- Sets `entryType = LedgerEntryType.EVENT` — a retrieval is an observed event (something happened), not a state-changing command or attestation
- Sets `actorRole = "cbr-retrieval-auditor"` — the system role that wrote the entry; the human/agent who triggered the consultation is in `actorId`
- Sets `actorId = ClinicalActors.CLINICAL_SERVICE`, `actorType = ActorType.SYSTEM`
- Serialises query features to summary string
- Attaches `ClinicalComplianceSupplement.cbrRetrieval()` (new factory method)
- `REQUIRES_NEW` so the audit entry persists regardless of caller transaction state — the AI system queried the case base, and that fact must be recorded even if the caller's endpoint transaction rolls back

### ClinicalCbrService Enhancement

One new method. Existing `retrieveSimilar()` and `storeIdempotent()` untouched.

```java
public <C extends CbrCase> AuditedRetrievalResult<C> retrieveWithAudit(
        CbrQuery query, Class<C> caseType,
        UUID subjectId, String actorId)
```

New injection points: `ExplanationRenderer`, `CbrRetrievalLedgerWriter`, `Clock` (standard testability pattern — every clinical ledger writer injects `Clock`). Existing `CbrCaseMemoryStore` injection unchanged.

### Endpoint Updates

Three endpoints switch from `retrieveSimilar()` to `retrieveWithAudit()`:

1. `TrialDashboardResource.aePrecedents()` — passes `aeId` as `subjectId`, `principal.actorId()` as actor
2. `TrialDashboardResource.deviationPrecedents()` — passes `devId` as `subjectId`
3. `ProtocolAmendmentResource.amendmentPrecedents()` — passes `amendmentId` as `subjectId`

Response type changes from bare `List<XxxPrecedentResponse>` to `XxxPrecedentSearchResponse` wrapper.

### ClinicalComplianceSupplement

New `cbrRetrieval()` factory method returning EU AI Act Art.12 traceability classification. Follows existing 11 factory methods.

```java
s.planRef = "EU AI Act Art.12 — record-keeping for high-risk AI decision support";
s.algorithmRef = "ClinicalCbrService (CBR precedent retrieval, weighted feature similarity)";
s.humanOverrideAvailable = true;
```

## File Inventory

### New Files

| File | Module | Type |
|---|---|---|
| `ClinicalExplanationRenderer.java` | runtime | Production |
| `CbrRetrievalLedgerEntry.java` | runtime | Production |
| `CbrRetrievalLedgerWriter.java` | runtime | Production |
| `AuditedRetrievalResult.java` | runtime | Production |
| `AePrecedentSearchResponse.java` | api | Production |
| `DeviationPrecedentSearchResponse.java` | api | Production |
| `AmendmentPrecedentSearchResponse.java` | api | Production |
| `V2029__cbr_retrieval_ledger_entry.sql` | runtime | Migration |
| `ClinicalExplanationRendererTest.java` | runtime | Test |
| `CbrRetrievalLedgerWriterTest.java` | runtime | Test |
| `ClinicalCbrServiceAuditTest.java` | runtime | Test |
| `CbrRetrievalAuditIntegrationTest.java` | runtime | Test |

### Modified Files

| File | Change |
|---|---|
| `ClinicalCbrService.java` | Add `retrieveWithAudit()` method + new injections |
| `TrialDashboardResource.java` | Switch to `retrieveWithAudit()`, return wrapper responses |
| `ProtocolAmendmentResource.java` | Switch to `retrieveWithAudit()`, return wrapper response |
| `ClinicalComplianceSupplement.java` | Add `cbrRetrieval()` factory method |
| Test `application.properties` | No changes expected — writer uses existing `InMemoryLedgerEntryRepository` |

## Testing Strategy

| Test | What it verifies |
|---|---|
| `ClinicalExplanationRendererTest` | Correct FDA-structured text for AE/deviation/amendment domains; empty results; null confidence handling |
| `CbrRetrievalLedgerWriterTest` | Entry construction, `domainContentBytes()`, compliance supplement attachment, sequence numbering |
| `ClinicalCbrServiceAuditTest` | `retrieveWithAudit()` pipeline: calls retrieve → builds trace → renders → writes ledger → returns result with traceId |
| `CbrRetrievalAuditIntegrationTest` | `@QuarkusTest` end-to-end: store a case, retrieve with audit, verify ledger entry persisted, verify explanation in response |
| Existing `TrialDashboardResourceTest` | Updated assertions for wrapper response shape (`traceId`, `explanation`, `precedents`) |
