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
    → buildTrace(query, results)                  // pure mapping → CbrRetrievalTrace
    → explanationRenderer.render(trace)           // CDI-injected ExplanationRenderer
    → ledgerWriter.record(trace, explanation, …)  // REQUIRES_NEW ledger write
    → return AuditedRetrievalResult(cases, traceId, explanation)
```

### Data Model

#### `CbrRetrievalLedgerEntry extends JpaLedgerEntry`

| Field | Type | Purpose |
|---|---|---|
| `traceId` | `String` | UUID correlating ledger entry with REST response |
| `queryDomain` | `String` | "clinical-ae", "clinical-deviation", or "clinical-amendment" |
| `queryFeaturesSummary` | `String` | Compact query features: "grade=3,eventType=Neutropenia,trialPhase=PHASE_III" |
| `retrievedCaseCount` | `int` | Number of precedents returned |
| `topScore` | `double` | Highest similarity score in result set |
| `explanationText` | `String` (TEXT) | Full FDA explanation from `ClinicalExplanationRenderer` |

- `subjectId`: the entity being queried (aeId / deviationId / amendmentId)
- `actorId`: the human or agent who triggered the consultation
- `domainContentBytes()`: joins all fields for Merkle hash
- Flyway V2029 on qhorus datasource

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
Top precedent: score 0.92, confidence 0.85. Feature alignment: grade=1.00, eventType=0.95, trialPhase=0.80.
3 of 4 precedents resolved successfully. 1 faulted.
Query domain: clinical-ae. Retrieval mode: HYBRID.
```

Computes outcome/confidence statistics from `TracedCase.confidence()` across the result set.

### CbrRetrievalLedgerWriter

`@ApplicationScoped` in `io.casehub.clinical.service`. Follows existing 10-writer pattern.

```java
@Transactional(TxType.REQUIRES_NEW)
public void record(CbrRetrievalTrace trace, String explanation,
                   UUID subjectId, String actorId, String tenantId)
```

- Constructs `CbrRetrievalLedgerEntry`
- Serialises query features to summary string
- Attaches `ClinicalComplianceSupplement.cbrRetrieval()` (new factory method)
- `REQUIRES_NEW` so audit persists regardless of caller transaction state

### ClinicalCbrService Enhancement

One new method. Existing `retrieveSimilar()` and `storeIdempotent()` untouched.

```java
public <C extends CbrCase> AuditedRetrievalResult<C> retrieveWithAudit(
        CbrQuery query, Class<C> caseType,
        UUID subjectId, String actorId)
```

New injection points: `ExplanationRenderer`, `CbrRetrievalLedgerWriter`. Existing `CbrCaseMemoryStore` injection unchanged.

### Endpoint Updates

Three endpoints switch from `retrieveSimilar()` to `retrieveWithAudit()`:

1. `TrialDashboardResource.aePrecedents()` — passes `aeId` as `subjectId`, `principal.actorId()` as actor
2. `TrialDashboardResource.deviationPrecedents()` — passes `devId` as `subjectId`
3. `ProtocolAmendmentResource.amendmentPrecedents()` — passes `amendmentId` as `subjectId`

Response type changes from bare `List<XxxPrecedentResponse>` to `XxxPrecedentSearchResponse` wrapper.

### ClinicalComplianceSupplement

New `cbrRetrieval()` factory method returning EU AI Act Art.12 traceability classification. Follows existing 6 factory methods.

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
