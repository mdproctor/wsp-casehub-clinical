# Regulatory Reporting and Audit Trail Export — Design Spec

**Epic:** casehubio/clinical#162
**Date:** 2026-09-22
**Branch:** issue-162-regulatory-reporting-export

## Overview

Generate regulatory-grade export artifacts from the existing ledger,
compliance supplements, and work item history. Three report types, four
REST endpoints, dual output format (JSON + PDF via Accept header).

The data already exists — 22 LedgerEntry subclasses, ComplianceSupplement
attachments, LedgerVerificationService, LedgerProvExportService, WorkItemStore.
This issue adds the rendering and export layer.

## Architecture

Three-layer split (D3):

```
casehub-platform-pdf (exists)
  └── PdfGenerator SPI — generateFromHtml(html, PdfOptions)
      └── OpenHtmlToPdfGenerator (openhtmltopdf, PDF/A-2B, Liberation fonts)

casehub-ledger-reporting (new — casehubio/ledger#211)
  ├── ReportRenderer — Qute template → HTML → PdfGenerator → PDF
  ├── ReportMediaType — Accept header content negotiation utility
  ├── ComplianceReportService — EU AI Act Art.12 report
  ├── AuditTrailExportService — Merkle chain + PROV-O aggregation
  └── MerkleVerificationBundleService — offline verification bundle

casehub-clinical (this issue)
  ├── IndSafetyReportService — IND safety report (21 CFR 312.32)
  ├── ReportResource — 4 REST endpoints at /api/reports/*
  └── Qute templates — IND-specific report templates
```

Clinical depends on `casehub-ledger-reporting` for compliance/audit reports
and `casehub-platform-pdf` (transitive) for PDF rendering. Clinical only
implements the IND safety report — the other two reports delegate to
ledger-reporting services.

## Report Types

### 1. IND Safety Report (21 CFR 312.32)

**Endpoint:** `GET /api/reports/ind-safety?trialId={id}&from={date}&to={date}`

**Data sources:**
- `AdverseEvent` entity — grade, reportedAt, unexpected, suspected, outcome,
  escalationStatus, regulatorySubmissionStatus
- `AdverseEventLedgerEntry` — tamper-evident record of AE reporting
- `AeEscalationLedgerEntry` — escalation chain
- `SusarDecisionLedgerEntry` — SUSAR gate decisions
- `IndReportFiledLedgerEntry` — IND filing records
- `IndReportBreachLedgerEntry` — missed deadlines
- `RegulatorySubmissionLedgerEntry` — regulatory submission obligations
- WorkItem history — SLA compliance for regulatory work items

**Report model:**

```java
public record IndSafetyReport(
    ReportMetadata metadata,
    TrialSummary trial,
    PeriodSummary period,
    List<IndividualCaseSafetyReport> icsrs,
    SlaComplianceSummary slaCompliance
) {}

public record PeriodSummary(
    int totalAdverseEvents,
    Map<CtcaeGrade, Integer> byGrade,
    int susarCount,
    int indReportsFiled,
    int indReportsBreached,
    int seriousUnexpectedCount
) {}

public record IndividualCaseSafetyReport(
    UUID aeId,
    UUID patientEnrollmentId,
    CtcaeGrade grade,
    String eventType,
    Instant reportedAt,
    Instant slaDeadline,
    boolean unexpected,
    boolean suspected,
    AeOutcome outcome,
    RegulatorySubmissionStatus regulatoryStatus,
    List<EscalationStep> escalationChain,
    @Nullable SusarDecision susarDecision,
    List<LedgerTraceEntry> auditTrail
) {}

public record SlaComplianceSummary(
    int totalObligations,
    int metWithinSla,
    int breached,
    double complianceRate
) {}
```

**ICSR criteria:** Include AEs where `grade >= GRADE_3` OR `unexpected == true`
OR `regulatorySubmissionStatus != NONE` within the reporting period.

**Ledger traceability:** Each ICSR includes its `auditTrail` — the chain of
LedgerEntry records for that AE, providing tamper-evident provenance for
every data point in the report.

### 2. Audit Trail Export

**Endpoint:** `GET /api/reports/audit-trail?trialId={id}`

**Delegates to:** `AuditTrailExportService` from casehub-ledger-reporting.

Clinical's `ReportResource` calls `auditTrailExportService.generateForTenancy(tenancyId)`
and wraps the result with clinical-specific context (trial name, sites).

**Output (JSON):** Full Merkle chain per subject (patient enrollment), PROV-O
JSON-LD provenance graphs, verification summary.

**Output (PDF):** Summary page with verification status per subject, followed
by per-subject decision history tables.

### 3. EU AI Act Art.12 Compliance Report

**Endpoint:** `GET /api/reports/compliance?trialId={id}`

**Delegates to:** `ComplianceReportService` from casehub-ledger-reporting.

Clinical's `ReportResource` calls `complianceReportService.generateForTenancy(tenancyId)`
and wraps with clinical-specific headers (trial protocol, sponsor).

**Content:** AI system transparency log — all agent decisions with compliance
supplements (planRef, algorithmRef, humanOverrideAvailable), risk classification
from `ClinicalActionRiskClassifier`, LLM invocation metrics (model, tokens, cost)
from `InvocationMetrics` attached via `ClinicalComplianceSupplement.withMetrics()`.

### 4. Merkle Verification Bundle

**Endpoint:** `GET /api/reports/merkle-verification?trialId={id}`

**Always returns JSON** (not PDF — this is a machine-readable verification package).

**Delegates to:** `MerkleVerificationBundleService` from casehub-ledger-reporting.

**Bundle contents:**
- All ledger entries with their hashes per subject
- MMR structure per subject chain
- Standalone Python verification script (embedded as string)
- Human-readable verification instructions (Markdown)
- Verification summary (chains valid/invalid count)

## REST API

### ReportResource

```java
@Path("/api/reports")
@Produces({MediaType.APPLICATION_JSON, "application/pdf"})
@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.MONITOR})
public class ReportResource {

    @GET @Path("/ind-safety")
    public Response indSafety(
        @QueryParam("trialId") UUID trialId,
        @QueryParam("from") String from,
        @QueryParam("to") String to,
        @Context HttpHeaders headers);

    @GET @Path("/audit-trail")
    public Response auditTrail(
        @QueryParam("trialId") UUID trialId,
        @Context HttpHeaders headers);

    @GET @Path("/compliance")
    public Response compliance(
        @QueryParam("trialId") UUID trialId,
        @Context HttpHeaders headers);

    @GET @Path("/merkle-verification")
    @Produces(MediaType.APPLICATION_JSON)
    public Response merkleVerification(
        @QueryParam("trialId") UUID trialId);
}
```

**Roles:** `SPONSOR` and `MONITOR` — regulatory reports are sponsor/monitor
responsibility per ICH E6(R3). Investigators and coordinators do not generate
regulatory exports.

**Content negotiation:** `ReportMediaType.respond()` from ledger-reporting
handles Accept header routing. Merkle verification is JSON-only.

## Content Negotiation Flow

```
Client sends Accept: application/pdf
  → ReportResource receives request
  → Service builds report model (POJO)
  → ReportMediaType.wantsPdf(headers) → true
  → ReportRenderer.renderPdf("ind-safety", model)
    → Qute resolves templates/reports/ind-safety.html
    → Qute renders HTML with model data
    → PdfGenerator.generateFromHtml(html, options) → PDF bytes
  → Response with Content-Disposition: attachment; filename="ind-safety-2026-09.pdf"
```

## Qute Templates

Clinical provides IND-specific templates at `runtime/src/main/resources/templates/reports/`:

- `ind-safety.html` — IND periodic safety report
  - Header: trial protocol ID, sponsor, reporting period
  - Aggregate summary table (AEs by grade, SUSARs, IND filings)
  - Individual case safety reports (one section per qualifying AE)
  - SLA compliance section
  - Footer: generation timestamp, Merkle root hash

Ledger-reporting provides compliance/audit templates (not clinical's concern).

All templates extend a base template from ledger-reporting with consistent
header/footer structure and regulatory document styling.

## Dependencies

### New Maven dependencies (runtime/pom.xml)

```xml
<!-- Ledger reporting — compliance, audit, verification reports -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-reporting</artifactId>
</dependency>
```

`casehub-platform-pdf` is a transitive dependency of `casehub-ledger-reporting`.

**Dependency on ledger#211:** Clinical#162 depends on casehubio/ledger#211
shipping `casehub-ledger-reporting`. For initial development, stub the
ledger-reporting interfaces with in-memory implementations in clinical's
test scope.

### Quarkus extensions

`quarkus-qute` — may need explicit addition if not already transitive.

## Testing Strategy

### Unit tests

- `IndSafetyReportServiceTest` — verify AE aggregation logic, ICSR
  filtering (grade >= 3, unexpected, regulatory status), period filtering,
  SLA compliance calculation
- `ReportResourceTest` — verify content negotiation, role-based access,
  query parameter validation

### Integration tests (`@QuarkusTest`)

- `IndSafetyReportIntegrationTest` — seed trial + AEs + ledger entries
  in `@BeforeEach`, call endpoint, verify JSON response structure
- PDF rendering test — call with `Accept: application/pdf`, verify
  response is valid PDF (starts with `%PDF`, non-empty)
- RBAC test — verify `@RolesAllowed` enforcement (SPONSOR/MONITOR only)

### Stub tests (until ledger#211 ships)

- Mock `ComplianceReportService`, `AuditTrailExportService`,
  `MerkleVerificationBundleService` with `@InjectMock`
- Verify clinical endpoints correctly delegate and wrap responses

## File Layout

```
runtime/src/main/java/io/casehub/clinical/
  report/
    IndSafetyReportService.java
    ReportResource.java
    model/
      IndSafetyReport.java
      IndividualCaseSafetyReport.java
      PeriodSummary.java
      SlaComplianceSummary.java
      TrialReportContext.java

runtime/src/main/resources/templates/reports/
  ind-safety.html

runtime/src/test/java/io/casehub/clinical/
  report/
    IndSafetyReportServiceTest.java
    ReportResourceTest.java
    IndSafetyReportIntegrationTest.java
```

## Phasing

**Phase 1 (this branch):** Clinical-side implementation with stubs for
ledger-reporting services. IND safety report fully functional (queries
existing entities and ledger entries directly). Audit trail, compliance,
and Merkle endpoints return stubbed/mocked responses that demonstrate the
API contract.

**Phase 2 (after ledger#211):** Replace stubs with real ledger-reporting
services. Add Qute templates for compliance/audit PDF rendering. Wire
content negotiation end-to-end.

## References

- `PdfGenerator` SPI — `casehub-platform-api/pdf/PdfGenerator.java`
- `OpenHtmlToPdfGenerator` — `casehub-platform/platform-pdf/`
- `LedgerVerificationService` — `casehub-ledger/runtime/service/`
- `LedgerProvExportService` — `casehub-ledger/runtime/service/`
- `ComplianceSupplement` — `casehub-ledger/`
- `PatientComplianceResource` — existing per-patient audit/compliance endpoints
- `ClinicalComplianceSupplement` — factory for clinical supplements
- `ClinicalIndReportingBreachPolicy` — IND SLA enforcement
- `AdverseEvent` entity — grade, SLA, regulatory fields
- 22 LedgerEntry subclasses in `io.casehub.clinical.ledger`
- casehubio/ledger#211 — ledger-reporting module
- casehubio/platform#383 (closed) — platform-pdf SPI already exists
- 21 CFR 312.32 — FDA IND safety reporting
- EU AI Act Article 12 — transparency and logging
- W3C PROV-O — https://www.w3.org/TR/prov-dm/
