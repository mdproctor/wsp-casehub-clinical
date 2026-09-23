# Regulatory Reporting and Audit Trail Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #162 — Regulatory reporting and audit trail export
**Issue group:** #162

**Goal:** Generate regulatory-grade export artifacts (IND safety, audit
trail, compliance) from existing ledger and entity data, with dual JSON/PDF
output via Accept header content negotiation.

**Architecture:** Three services behind one `ReportResource` — `IndSafetyReportService`
(queries AEs, ledger entries, work items), plus stub delegates for compliance
and audit trail endpoints (real services arrive with casehubio/ledger#211).
`PdfGenerator` SPI (platform-pdf) renders Qute HTML templates to PDF/A-2B.

**Tech Stack:** Java 21 / Quarkus 3.32.2, casehub-platform-pdf (`PdfGenerator`
SPI, openhtmltopdf, PDF/A-2B), Quarkus Qute (HTML templates), existing clinical
entities + 22 LedgerEntry subclasses.

## Global Constraints

- `PdfGenerator.generateFromHtml(String html, PdfOptions options)` is the PDF SPI — in `casehub-platform-api`, implemented by `OpenHtmlToPdfGenerator` in `casehub-platform-pdf`
- `PdfOptions.defaults()` returns null fields + `PdfAConformance.PDFA_2_B`
- Report endpoints require `@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.MONITOR})` — SPONSOR = `"trial-sponsor"`, MONITOR = `"safety-monitor"`
- `@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.MONITOR})` on all report test classes
- Tenancy: use `CurrentPrincipal.tenancyId()` for all tenant-scoped queries
- AdverseEvent has no `findByTrialId` — chain: trial → sites (by trialId) → enrollments (by siteIds) → AEs (by enrollmentIds)
- LedgerEntryRepository has no `findByTenancyId` — iterate enrollment (subject) IDs and call `findBySubjectId()` per enrollment
- ICSR criteria: `grade >= GRADE_3 OR unexpected == true OR regulatorySubmissionStatus != NONE`
- `casehub-ledger-reporting` does not exist yet (casehubio/ledger#211) — stub/mock compliance, audit, and Merkle endpoints

---

## Batch 1: Report Models + IND Safety Service

### Task 1: Report model records + IndSafetyReportService

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/ReportMetadata.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/TrialReportContext.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/PeriodSummary.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/EscalationStep.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/SusarDecision.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/LedgerTraceEntry.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/IndividualCaseSafetyReport.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/SlaComplianceSummary.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/model/IndSafetyReport.java`
- Create: `runtime/src/main/java/io/casehub/clinical/report/IndSafetyReportService.java`
- Test: `runtime/src/test/java/io/casehub/clinical/report/IndSafetyReportServiceTest.java`

**Interfaces:**
- Consumes: `AdverseEvent` entity (plain JPA, `EntityManager` queries), `ClinicalTrial` entity, `TrialSite` entity, `PatientEnrollment` entity, `LedgerEntryRepository.findBySubjectId(UUID, String)`, `CtcaeGrade` enum (compareTo for >= GRADE_3), `RegulatorySubmissionStatus` enum, `AeEscalationStatus` enum, `AeOutcome` enum
- Produces: `IndSafetyReport` record consumed by `ReportResource` (Task 2), `ReportMetadata` and other model records reused across report types

- [ ] **Step 1: Write report model records**

Create all model records. These are plain data carriers with no logic.

```java
// ReportMetadata.java
package io.casehub.clinical.report.model;

import java.time.Instant;

public record ReportMetadata(
        String reportType,
        String tenancyId,
        Instant generatedAt,
        Instant periodStart,
        Instant periodEnd) {}
```

```java
// TrialReportContext.java
package io.casehub.clinical.report.model;

import io.casehub.clinical.api.model.TrialPhase;
import java.util.UUID;

public record TrialReportContext(
        UUID trialId,
        String protocolId,
        TrialPhase phase,
        String sponsor,
        int totalSites,
        int totalEnrolled) {}
```

```java
// PeriodSummary.java
package io.casehub.clinical.report.model;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.util.Map;

public record PeriodSummary(
        int totalAdverseEvents,
        Map<CtcaeGrade, Integer> byGrade,
        int susarCount,
        int indReportsFiled,
        int indReportsBreached,
        int seriousUnexpectedCount) {}
```

```java
// EscalationStep.java
package io.casehub.clinical.report.model;

import java.time.Instant;

public record EscalationStep(
        String status,
        Instant occurredAt,
        String actorId) {}
```

```java
// SusarDecision.java
package io.casehub.clinical.report.model;

import java.time.Instant;

public record SusarDecision(
        String gateOutcome,
        Instant decidedAt) {}
```

```java
// LedgerTraceEntry.java
package io.casehub.clinical.report.model;

import java.time.Instant;

public record LedgerTraceEntry(
        String entryType,
        Instant occurredAt,
        String actorId,
        String digest,
        long sequenceNumber) {}
```

```java
// IndividualCaseSafetyReport.java
package io.casehub.clinical.report.model;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import org.jspecify.annotations.Nullable;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

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
        List<LedgerTraceEntry> auditTrail) {}
```

```java
// SlaComplianceSummary.java
package io.casehub.clinical.report.model;

public record SlaComplianceSummary(
        int totalObligations,
        int metWithinSla,
        int breached,
        double complianceRate) {}
```

```java
// IndSafetyReport.java
package io.casehub.clinical.report.model;

import java.util.List;

public record IndSafetyReport(
        ReportMetadata metadata,
        TrialReportContext trial,
        PeriodSummary period,
        List<IndividualCaseSafetyReport> icsrs,
        SlaComplianceSummary slaCompliance) {}
```

- [ ] **Step 2: Write unit test for IndSafetyReportService**

```java
package io.casehub.clinical.report;

import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import jakarta.persistence.EntityManager;
import jakarta.persistence.TypedQuery;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class IndSafetyReportServiceTest {

    private EntityManager em;
    private LedgerEntryRepository ledgerRepo;
    private IndSafetyReportService service;

    private final UUID trialId = UUID.randomUUID();
    private final String tenancyId = "default";
    private final Instant from = Instant.parse("2026-01-01T00:00:00Z");
    private final Instant to = Instant.parse("2026-06-30T23:59:59Z");

    @BeforeEach
    void setup() {
        em = mock(EntityManager.class);
        ledgerRepo = mock(LedgerEntryRepository.class);
        service = new IndSafetyReportService(em, ledgerRepo);
    }

    @Test
    void filtersIcsrsByGrade3OrHigher() {
        var aes = List.of(
                ae(CtcaeGrade.GRADE_1, false, RegulatorySubmissionStatus.NONE),
                ae(CtcaeGrade.GRADE_3, false, RegulatorySubmissionStatus.NONE),
                ae(CtcaeGrade.GRADE_4, false, RegulatorySubmissionStatus.NONE));
        mockTrialLookup(aes);

        var report = service.generate(trialId, from, to, tenancyId);

        assertThat(report.icsrs()).hasSize(2);
        assertThat(report.icsrs()).allMatch(
                icsr -> icsr.grade().compareTo(CtcaeGrade.GRADE_3) >= 0);
    }

    @Test
    void includesUnexpectedAeRegardlessOfGrade() {
        var aes = List.of(
                ae(CtcaeGrade.GRADE_1, true, RegulatorySubmissionStatus.NONE),
                ae(CtcaeGrade.GRADE_2, false, RegulatorySubmissionStatus.NONE));
        mockTrialLookup(aes);

        var report = service.generate(trialId, from, to, tenancyId);

        assertThat(report.icsrs()).hasSize(1);
        assertThat(report.icsrs().getFirst().unexpected()).isTrue();
    }

    @Test
    void includesAeWithRegulatorySubmission() {
        var aes = List.of(
                ae(CtcaeGrade.GRADE_1, false, RegulatorySubmissionStatus.FILED),
                ae(CtcaeGrade.GRADE_2, false, RegulatorySubmissionStatus.NONE));
        mockTrialLookup(aes);

        var report = service.generate(trialId, from, to, tenancyId);

        assertThat(report.icsrs()).hasSize(1);
        assertThat(report.icsrs().getFirst().regulatoryStatus())
                .isEqualTo(RegulatorySubmissionStatus.FILED);
    }

    @Test
    void computesPeriodSummaryByGrade() {
        var aes = List.of(
                ae(CtcaeGrade.GRADE_1, false, RegulatorySubmissionStatus.NONE),
                ae(CtcaeGrade.GRADE_3, false, RegulatorySubmissionStatus.NONE),
                ae(CtcaeGrade.GRADE_3, false, RegulatorySubmissionStatus.NONE),
                ae(CtcaeGrade.GRADE_5, true, RegulatorySubmissionStatus.FILED));
        mockTrialLookup(aes);

        var report = service.generate(trialId, from, to, tenancyId);

        assertThat(report.period().totalAdverseEvents()).isEqualTo(4);
        assertThat(report.period().byGrade().get(CtcaeGrade.GRADE_3)).isEqualTo(2);
        assertThat(report.period().byGrade().get(CtcaeGrade.GRADE_5)).isEqualTo(1);
    }

    @Test
    void filtersByPeriod() {
        var before = ae(CtcaeGrade.GRADE_4, false, RegulatorySubmissionStatus.NONE);
        before.reportedAt = Instant.parse("2025-12-01T00:00:00Z");
        var during = ae(CtcaeGrade.GRADE_4, false, RegulatorySubmissionStatus.NONE);
        during.reportedAt = Instant.parse("2026-03-15T00:00:00Z");
        mockTrialLookup(List.of(before, during));

        var report = service.generate(trialId, from, to, tenancyId);

        assertThat(report.period().totalAdverseEvents()).isEqualTo(1);
    }

    @Test
    void computesSlaCompliance() {
        var met = ae(CtcaeGrade.GRADE_3, false, RegulatorySubmissionStatus.FILED);
        met.regulatorySubmissionStatus = RegulatorySubmissionStatus.FILED;
        var breached = ae(CtcaeGrade.GRADE_4, false, RegulatorySubmissionStatus.DEADLINE_MISSED);
        breached.regulatorySubmissionStatus = RegulatorySubmissionStatus.DEADLINE_MISSED;
        mockTrialLookup(List.of(met, breached));

        var report = service.generate(trialId, from, to, tenancyId);

        assertThat(report.slaCompliance().totalObligations()).isEqualTo(2);
        assertThat(report.slaCompliance().metWithinSla()).isEqualTo(1);
        assertThat(report.slaCompliance().breached()).isEqualTo(1);
        assertThat(report.slaCompliance().complianceRate()).isEqualTo(0.5);
    }

    @Test
    void setsMetadataCorrectly() {
        mockTrialLookup(List.of());

        var report = service.generate(trialId, from, to, tenancyId);

        assertThat(report.metadata().reportType()).isEqualTo("ind-safety");
        assertThat(report.metadata().periodStart()).isEqualTo(from);
        assertThat(report.metadata().periodEnd()).isEqualTo(to);
        assertThat(report.metadata().tenancyId()).isEqualTo(tenancyId);
    }

    private AdverseEvent ae(CtcaeGrade grade, boolean unexpected,
                            RegulatorySubmissionStatus regStatus) {
        var ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = grade;
        ae.unexpected = unexpected;
        ae.suspected = true;
        ae.regulatorySubmissionStatus = regStatus;
        ae.escalationStatus = AeEscalationStatus.NONE;
        ae.outcome = AeOutcome.ONGOING;
        ae.eventType = "headache";
        ae.reportedAt = Instant.parse("2026-03-15T10:00:00Z");
        ae.slaDeadline = ae.reportedAt.plus(grade.sla().orElse(Duration.ofDays(7)));
        return ae;
    }

    @SuppressWarnings("unchecked")
    private void mockTrialLookup(List<AdverseEvent> aes) {
        var trial = new ClinicalTrial();
        trial.id = trialId;
        trial.protocolId = "PROTO-001";
        trial.phase = TrialPhase.PHASE_3;
        trial.sponsor = "Acme Pharma";
        trial.tenantId = tenancyId;

        var trialQuery = mock(TypedQuery.class);
        when(em.createNamedQuery("ClinicalTrial.findByIdAndTenantId", ClinicalTrial.class))
                .thenReturn(trialQuery);
        when(trialQuery.setParameter(any(String.class), any())).thenReturn(trialQuery);
        when(trialQuery.getSingleResult()).thenReturn(trial);

        var site = new TrialSite();
        site.id = UUID.randomUUID();
        site.trialId = trialId;
        site.tenantId = tenancyId;

        var siteQuery = mock(TypedQuery.class);
        when(em.createQuery(contains("TrialSite"), eq(TrialSite.class)))
                .thenReturn(siteQuery);
        when(siteQuery.setParameter(any(String.class), any())).thenReturn(siteQuery);
        when(siteQuery.getResultList()).thenReturn(List.of(site));

        var enrollment = new PatientEnrollment();
        enrollment.id = UUID.randomUUID();
        enrollment.siteId = site.id;
        enrollment.tenantId = tenancyId;

        var enrollQuery = mock(TypedQuery.class);
        when(em.createQuery(contains("PatientEnrollment"), eq(PatientEnrollment.class)))
                .thenReturn(enrollQuery);
        when(enrollQuery.setParameter(any(String.class), any())).thenReturn(enrollQuery);
        when(enrollQuery.getResultList()).thenReturn(List.of(enrollment));

        var aeQuery = mock(TypedQuery.class);
        when(em.createQuery(contains("AdverseEvent"), eq(AdverseEvent.class)))
                .thenReturn(aeQuery);
        when(aeQuery.setParameter(any(String.class), any())).thenReturn(aeQuery);
        when(aeQuery.getResultList()).thenReturn(aes);

        when(ledgerRepo.findBySubjectId(any(), eq(tenancyId))).thenReturn(List.of());
    }

    private static String contains(String substring) {
        return argThat(s -> s != null && s.contains(substring));
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=IndSafetyReportServiceTest --batch-mode`
Expected: FAIL — `IndSafetyReportService` does not exist

- [ ] **Step 4: Implement IndSafetyReportService**

```java
package io.casehub.clinical.report;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.report.model.*;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

@ApplicationScoped
public class IndSafetyReportService {

    private final EntityManager em;
    private final LedgerEntryRepository ledgerRepo;

    @Inject
    public IndSafetyReportService(EntityManager em,
                                   LedgerEntryRepository ledgerRepo) {
        this.em = em;
        this.ledgerRepo = ledgerRepo;
    }

    public IndSafetyReport generate(UUID trialId, Instant from, Instant to,
                                     String tenancyId) {
        var trial = em.createNamedQuery("ClinicalTrial.findByIdAndTenantId",
                        ClinicalTrial.class)
                .setParameter("id", trialId)
                .setParameter("tenantId", tenancyId)
                .getSingleResult();

        var sites = em.createQuery(
                        "SELECT s FROM TrialSite s WHERE s.trialId = :trialId AND s.tenantId = :tenantId",
                        TrialSite.class)
                .setParameter("trialId", trialId)
                .setParameter("tenantId", tenancyId)
                .getResultList();

        List<UUID> siteIds = sites.stream().map(s -> s.id).toList();
        if (siteIds.isEmpty()) {
            return emptyReport(trial, from, to, tenancyId, sites);
        }

        var enrollments = em.createQuery(
                        "SELECT e FROM PatientEnrollment e WHERE e.siteId IN :siteIds AND e.tenantId = :tenantId",
                        PatientEnrollment.class)
                .setParameter("siteIds", siteIds)
                .setParameter("tenantId", tenancyId)
                .getResultList();

        List<UUID> enrollmentIds = enrollments.stream().map(e -> e.id).toList();
        if (enrollmentIds.isEmpty()) {
            return emptyReport(trial, from, to, tenancyId, sites);
        }

        var allAes = em.createQuery(
                        "SELECT a FROM AdverseEvent a WHERE a.enrollmentId IN :enrollmentIds AND a.tenantId = :tenantId",
                        AdverseEvent.class)
                .setParameter("enrollmentIds", enrollmentIds)
                .setParameter("tenantId", tenancyId)
                .getResultList();

        var periodAes = allAes.stream()
                .filter(ae -> ae.reportedAt != null
                        && !ae.reportedAt.isBefore(from)
                        && !ae.reportedAt.isAfter(to))
                .toList();

        var metadata = new ReportMetadata("ind-safety", tenancyId,
                Instant.now(), from, to);

        var trialContext = new TrialReportContext(trial.id, trial.protocolId,
                trial.phase, trial.sponsor, sites.size(),
                enrollments.size());

        var period = buildPeriodSummary(periodAes);
        var icsrs = buildIcsrs(periodAes, tenancyId);
        var slaCompliance = buildSlaCompliance(periodAes);

        return new IndSafetyReport(metadata, trialContext, period,
                icsrs, slaCompliance);
    }

    private PeriodSummary buildPeriodSummary(List<AdverseEvent> aes) {
        Map<CtcaeGrade, Integer> byGrade = new EnumMap<>(CtcaeGrade.class);
        int susarCount = 0;
        int filed = 0;
        int breached = 0;
        int seriousUnexpected = 0;

        for (var ae : aes) {
            byGrade.merge(ae.grade, 1, Integer::sum);
            if (ae.susarOversightStatus != null
                    && ae.susarOversightStatus.name().contains("CONFIRMED")) {
                susarCount++;
            }
            if (ae.regulatorySubmissionStatus == RegulatorySubmissionStatus.FILED) {
                filed++;
            }
            if (ae.regulatorySubmissionStatus == RegulatorySubmissionStatus.DEADLINE_MISSED) {
                breached++;
            }
            if (ae.grade.compareTo(CtcaeGrade.GRADE_3) >= 0 && ae.unexpected) {
                seriousUnexpected++;
            }
        }

        return new PeriodSummary(aes.size(), byGrade, susarCount,
                filed, breached, seriousUnexpected);
    }

    private List<IndividualCaseSafetyReport> buildIcsrs(List<AdverseEvent> aes,
                                                         String tenancyId) {
        return aes.stream()
                .filter(ae -> ae.grade.compareTo(CtcaeGrade.GRADE_3) >= 0
                        || ae.unexpected
                        || ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.NONE)
                .map(ae -> buildIcsr(ae, tenancyId))
                .toList();
    }

    private IndividualCaseSafetyReport buildIcsr(AdverseEvent ae, String tenancyId) {
        List<LedgerEntry> ledgerEntries = ledgerRepo.findBySubjectId(
                ae.enrollmentId, tenancyId);

        List<LedgerTraceEntry> auditTrail = ledgerEntries.stream()
                .map(e -> new LedgerTraceEntry(
                        e.getClass().getSimpleName(),
                        e.occurredAt(),
                        e.actorId(),
                        e.digest(),
                        e.sequenceNumber()))
                .toList();

        return new IndividualCaseSafetyReport(
                ae.id, ae.enrollmentId, ae.grade, ae.eventType,
                ae.reportedAt, ae.slaDeadline, ae.unexpected, ae.suspected,
                ae.outcome, ae.regulatorySubmissionStatus,
                List.of(), null, auditTrail);
    }

    private SlaComplianceSummary buildSlaCompliance(List<AdverseEvent> aes) {
        var withObligations = aes.stream()
                .filter(ae -> ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.NONE)
                .toList();

        int total = withObligations.size();
        int met = (int) withObligations.stream()
                .filter(ae -> ae.regulatorySubmissionStatus == RegulatorySubmissionStatus.FILED
                        || ae.regulatorySubmissionStatus == RegulatorySubmissionStatus.PENDING)
                .count();
        int breached = (int) withObligations.stream()
                .filter(ae -> ae.regulatorySubmissionStatus == RegulatorySubmissionStatus.DEADLINE_MISSED)
                .count();

        double rate = total > 0 ? (double) met / total : 1.0;

        return new SlaComplianceSummary(total, met, breached, rate);
    }

    private IndSafetyReport emptyReport(ClinicalTrial trial, Instant from,
                                         Instant to, String tenancyId,
                                         List<TrialSite> sites) {
        return new IndSafetyReport(
                new ReportMetadata("ind-safety", tenancyId, Instant.now(), from, to),
                new TrialReportContext(trial.id, trial.protocolId, trial.phase,
                        trial.sponsor, sites.size(), 0),
                new PeriodSummary(0, Map.of(), 0, 0, 0, 0),
                List.of(),
                new SlaComplianceSummary(0, 0, 0, 1.0));
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=IndSafetyReportServiceTest --batch-mode`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/report/
git add runtime/src/test/java/io/casehub/clinical/report/
git commit -m "feat(#162): add IndSafetyReportService + report model records

IND safety report aggregates AEs by trial, filters ICSRs by grade >= 3
or unexpected or regulatory submission, computes period summary and SLA
compliance. Chains trial → sites → enrollments → AEs for trial-level
queries. Includes per-ICSR ledger audit trail.

Refs #162"
```

---

## Batch 2: REST Endpoint + Content Negotiation

### Task 2: ReportResource with 4 endpoints + content negotiation

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/report/ReportResource.java`
- Test: `runtime/src/test/java/io/casehub/clinical/report/ReportResourceTest.java`

**Interfaces:**
- Consumes: `IndSafetyReportService.generate(UUID trialId, Instant from, Instant to, String tenancyId)` → `IndSafetyReport` (from Task 1), `PdfGenerator.generateFromHtml(String, PdfOptions)` → `Optional<byte[]>` (from platform-pdf), `CurrentPrincipal.tenancyId()` → `String`, `io.quarkus.qute.Template` (Qute injection)
- Produces: 4 REST endpoints at `/api/reports/*` — JSON or PDF based on Accept header

- [ ] **Step 1: Write integration test for ReportResource**

```java
package io.casehub.clinical.report;

import io.casehub.clinical.api.ClinicalGroups;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.MONITOR})
class ReportResourceTest {

    private static final UUID TRIAL_ID = UUID.fromString("316e3846-4ea7-3b18-a6f7-e01ce6582a69");

    @Test
    void indSafetyReturnsJson() {
        given()
            .queryParam("trialId", TRIAL_ID)
            .queryParam("from", "2026-01-01")
            .queryParam("to", "2026-06-30")
            .accept("application/json")
        .when()
            .get("/api/reports/ind-safety")
        .then()
            .statusCode(200)
            .contentType("application/json")
            .body("metadata.reportType", equalTo("ind-safety"))
            .body("period", notNullValue())
            .body("icsrs", notNullValue());
    }

    @Test
    void indSafetyReturnsPdf() {
        given()
            .queryParam("trialId", TRIAL_ID)
            .queryParam("from", "2026-01-01")
            .queryParam("to", "2026-06-30")
            .accept("application/pdf")
        .when()
            .get("/api/reports/ind-safety")
        .then()
            .statusCode(200)
            .contentType("application/pdf")
            .header("Content-Disposition", containsString("ind-safety"));
    }

    @Test
    void auditTrailReturnsJson() {
        given()
            .queryParam("trialId", TRIAL_ID)
            .accept("application/json")
        .when()
            .get("/api/reports/audit-trail")
        .then()
            .statusCode(200);
    }

    @Test
    void complianceReturnsJson() {
        given()
            .queryParam("trialId", TRIAL_ID)
            .accept("application/json")
        .when()
            .get("/api/reports/compliance")
        .then()
            .statusCode(200);
    }

    @Test
    void merkleVerificationReturnsJson() {
        given()
            .queryParam("trialId", TRIAL_ID)
        .when()
            .get("/api/reports/merkle-verification")
        .then()
            .statusCode(200)
            .contentType("application/json");
    }

    @Test
    @TestSecurity(user = "investigator", roles = {ClinicalGroups.INVESTIGATOR})
    void rejectsInvestigatorRole() {
        given()
            .queryParam("trialId", TRIAL_ID)
            .queryParam("from", "2026-01-01")
            .queryParam("to", "2026-06-30")
        .when()
            .get("/api/reports/ind-safety")
        .then()
            .statusCode(403);
    }

    @Test
    @TestSecurity(user = "anonymous")
    void rejectsUnauthenticated() {
        given()
            .queryParam("trialId", TRIAL_ID)
            .queryParam("from", "2026-01-01")
            .queryParam("to", "2026-06-30")
        .when()
            .get("/api/reports/ind-safety")
        .then()
            .statusCode(anyOf(is(401), is(403)));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=ReportResourceTest --batch-mode`
Expected: FAIL — `ReportResource` does not exist

- [ ] **Step 3: Create Qute template for IND safety report**

Create `runtime/src/main/resources/templates/reports/ind-safety.html`:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8"/>
<style>
  body { font-family: 'Liberation Sans', sans-serif; font-size: 10pt; margin: 2cm; }
  h1 { font-size: 16pt; border-bottom: 2pt solid #333; padding-bottom: 4pt; }
  h2 { font-size: 13pt; margin-top: 1.5em; color: #333; }
  h3 { font-size: 11pt; margin-top: 1em; }
  table { border-collapse: collapse; width: 100%; margin: 0.5em 0; }
  th, td { border: 1px solid #999; padding: 4pt 6pt; text-align: left; font-size: 9pt; }
  th { background-color: #f0f0f0; font-weight: bold; }
  .summary-grid { display: flex; gap: 1em; margin: 0.5em 0; }
  .summary-card { border: 1px solid #ccc; padding: 8pt; flex: 1; text-align: center; }
  .summary-card .value { font-size: 18pt; font-weight: bold; }
  .summary-card .label { font-size: 8pt; color: #666; }
  .footer { margin-top: 2em; border-top: 1pt solid #999; padding-top: 4pt; font-size: 8pt; color: #666; }
</style>
</head>
<body>
<h1>IND Periodic Safety Report</h1>
<p><strong>Protocol:</strong> {report.trial.protocolId} &mdash;
   <strong>Phase:</strong> {report.trial.phase} &mdash;
   <strong>Sponsor:</strong> {report.trial.sponsor}</p>
<p><strong>Period:</strong> {report.metadata.periodStart} to {report.metadata.periodEnd}</p>

<h2>Summary</h2>
<div class="summary-grid">
  <div class="summary-card"><div class="value">{report.period.totalAdverseEvents}</div><div class="label">Total AEs</div></div>
  <div class="summary-card"><div class="value">{report.period.susarCount}</div><div class="label">SUSARs</div></div>
  <div class="summary-card"><div class="value">{report.period.indReportsFiled}</div><div class="label">IND Filed</div></div>
  <div class="summary-card"><div class="value">{report.period.indReportsBreached}</div><div class="label">Deadlines Missed</div></div>
</div>

<h3>AEs by Grade</h3>
<table>
  <tr><th>Grade</th><th>Count</th></tr>
  {#each report.period.byGrade.entrySet}
  <tr><td>{it.key}</td><td>{it.value}</td></tr>
  {/each}
</table>

<h2>Individual Case Safety Reports ({report.icsrs.size})</h2>
{#each report.icsrs}
<h3>ICSR: {it.aeId}</h3>
<table>
  <tr><th>Field</th><th>Value</th></tr>
  <tr><td>Patient</td><td>{it.patientEnrollmentId}</td></tr>
  <tr><td>Grade</td><td>{it.grade}</td></tr>
  <tr><td>Event Type</td><td>{it.eventType}</td></tr>
  <tr><td>Reported</td><td>{it.reportedAt}</td></tr>
  <tr><td>SLA Deadline</td><td>{it.slaDeadline}</td></tr>
  <tr><td>Unexpected</td><td>{it.unexpected}</td></tr>
  <tr><td>Suspected</td><td>{it.suspected}</td></tr>
  <tr><td>Outcome</td><td>{it.outcome}</td></tr>
  <tr><td>Regulatory Status</td><td>{it.regulatoryStatus}</td></tr>
</table>
{#if it.auditTrail.size > 0}
<p><strong>Audit Trail ({it.auditTrail.size} entries)</strong></p>
<table>
  <tr><th>Type</th><th>Occurred</th><th>Actor</th><th>Seq</th></tr>
  {#each it.auditTrail}
  <tr><td>{it.entryType}</td><td>{it.occurredAt}</td><td>{it.actorId}</td><td>{it.sequenceNumber}</td></tr>
  {/each}
</table>
{/if}
{/each}

<h2>SLA Compliance</h2>
<table>
  <tr><th>Metric</th><th>Value</th></tr>
  <tr><td>Total Obligations</td><td>{report.slaCompliance.totalObligations}</td></tr>
  <tr><td>Met Within SLA</td><td>{report.slaCompliance.metWithinSla}</td></tr>
  <tr><td>Breached</td><td>{report.slaCompliance.breached}</td></tr>
  <tr><td>Compliance Rate</td><td>{report.slaCompliance.complianceRate}</td></tr>
</table>

<div class="footer">
  Generated: {report.metadata.generatedAt} | Report Type: {report.metadata.reportType}
</div>
</body>
</html>
```

- [ ] **Step 4: Implement ReportResource**

```java
package io.casehub.clinical.report;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.report.model.IndSafetyReport;
import io.casehub.platform.api.pdf.PdfGenerator;
import io.casehub.platform.api.pdf.PdfOptions;
import io.casehub.platform.spi.CurrentPrincipal;
import io.quarkus.qute.Template;
import io.quarkus.qute.Location;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.Context;
import jakarta.ws.rs.core.HttpHeaders;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneOffset;
import java.util.Map;
import java.util.UUID;

@Path("/api/reports")
@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.MONITOR})
public class ReportResource {

    @Inject IndSafetyReportService indSafetyService;
    @Inject PdfGenerator pdfGenerator;
    @Inject CurrentPrincipal principal;

    @Location("reports/ind-safety")
    Template indSafetyTemplate;

    @GET
    @Path("/ind-safety")
    @Produces({MediaType.APPLICATION_JSON, "application/pdf"})
    public Response indSafety(@QueryParam("trialId") UUID trialId,
                               @QueryParam("from") String from,
                               @QueryParam("to") String to,
                               @Context HttpHeaders headers) {
        Instant fromInstant = LocalDate.parse(from).atStartOfDay().toInstant(ZoneOffset.UTC);
        Instant toInstant = LocalDate.parse(to).atTime(23, 59, 59).toInstant(ZoneOffset.UTC);

        var report = indSafetyService.generate(trialId, fromInstant, toInstant,
                principal.tenancyId());

        if (wantsPdf(headers)) {
            String html = indSafetyTemplate.data("report", report).render();
            var options = new PdfOptions("IND Safety Report — " + from + " to " + to,
                    "CaseHub Clinical", Instant.now(), "ind-safety", null);
            byte[] pdf = pdfGenerator.generateFromHtml(html, options).orElseThrow();
            return Response.ok(pdf, "application/pdf")
                    .header("Content-Disposition",
                            "attachment; filename=\"ind-safety-" + from + "-" + to + ".pdf\"")
                    .build();
        }

        return Response.ok(report, MediaType.APPLICATION_JSON_TYPE).build();
    }

    @GET
    @Path("/audit-trail")
    @Produces({MediaType.APPLICATION_JSON, "application/pdf"})
    public Response auditTrail(@QueryParam("trialId") UUID trialId,
                                @Context HttpHeaders headers) {
        return Response.ok(Map.of(
                "status", "stub",
                "message", "Audit trail export — pending casehubio/ledger#211",
                "trialId", trialId.toString()
        )).build();
    }

    @GET
    @Path("/compliance")
    @Produces({MediaType.APPLICATION_JSON, "application/pdf"})
    public Response compliance(@QueryParam("trialId") UUID trialId,
                                @Context HttpHeaders headers) {
        return Response.ok(Map.of(
                "status", "stub",
                "message", "EU AI Act Art.12 compliance report — pending casehubio/ledger#211",
                "trialId", trialId.toString()
        )).build();
    }

    @GET
    @Path("/merkle-verification")
    @Produces(MediaType.APPLICATION_JSON)
    public Response merkleVerification(@QueryParam("trialId") UUID trialId) {
        return Response.ok(Map.of(
                "status", "stub",
                "message", "Merkle verification bundle — pending casehubio/ledger#211",
                "trialId", trialId.toString()
        )).build();
    }

    private static boolean wantsPdf(HttpHeaders headers) {
        var accept = headers.getAcceptableMediaTypes();
        return accept.stream().anyMatch(mt ->
                mt.getType().equals("application") && mt.getSubtype().equals("pdf"));
    }
}
```

- [ ] **Step 5: Add quarkus-qute dependency if needed**

Check `runtime/pom.xml` for `quarkus-qute`. If not present, add:

```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-qute</artifactId>
</dependency>
```

Also ensure `casehub-platform-pdf` is on the classpath (for `PdfGenerator`):

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-pdf</artifactId>
  <scope>runtime</scope>
</dependency>
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=ReportResourceTest --batch-mode`
Expected: PASS — all 7 tests green

- [ ] **Step 7: Run full test suite to verify no regressions**

Run: `mvn test -pl runtime --batch-mode`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/report/ReportResource.java
git add runtime/src/main/resources/templates/reports/ind-safety.html
git add runtime/src/test/java/io/casehub/clinical/report/ReportResourceTest.java
git add runtime/pom.xml
git commit -m "feat(#162): add ReportResource with 4 endpoints + IND PDF template

GET /api/reports/ind-safety (JSON + PDF via Accept header)
GET /api/reports/audit-trail (stub — pending ledger#211)
GET /api/reports/compliance (stub — pending ledger#211)
GET /api/reports/merkle-verification (stub — pending ledger#211)

IND safety PDF rendered via Qute template + PdfGenerator SPI.
SPONSOR and MONITOR roles required per ICH E6(R3).

Refs #162"
```

---

## References

- `specs/issue-162-regulatory-reporting-export/2026-09-22-regulatory-reporting-export-design.md` — design spec
- `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java` — AE entity with grade, SLA, regulatory fields
- `runtime/src/main/java/io/casehub/clinical/entity/ClinicalTrial.java` — trial metadata
- `runtime/src/main/java/io/casehub/clinical/resource/PatientComplianceResource.java` — reference JAX-RS + ledger pattern
- `api/src/main/java/io/casehub/clinical/api/ClinicalGroups.java` — SPONSOR, MONITOR constants
- `api/src/main/java/io/casehub/clinical/api/model/CtcaeGrade.java` — grade enum with SLA durations
- `casehub-platform-api/pdf/PdfGenerator.java` — PDF SPI
- `casehub-platform/platform-pdf/OpenHtmlToPdfGenerator.java` — PDF/A-2B implementation
- casehubio/clinical#162 — focal issue
- casehubio/ledger#211 — ledger-reporting module (compliance, audit, verification)
- 21 CFR 312.32 — FDA IND safety reporting requirements
