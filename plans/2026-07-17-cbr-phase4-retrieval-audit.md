# CBR Phase 4 — Retrieval Audit Trail Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #117 — feat: CBR Phase 4 — outcome recording + retrieval audit trail
**Issue group:** #117

**Goal:** Every CBR precedent consultation produces a tamper-evident ledger entry and FDA-structured explanation text.

**Architecture:** `ClinicalCbrService` gains a `retrieveWithAudit()` composition method that reuses the existing `retrieveSimilar()`, builds a `CbrRetrievalTrace`, renders an explanation via `ClinicalExplanationRenderer` (CDI-displaced `ExplanationRenderer`), and writes a `CbrRetrievalLedgerEntry` via `CbrRetrievalLedgerWriter`. Three REST endpoints switch to the audited path and return wrapper responses with `traceId` + `explanation`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (JOINED inheritance on qhorus datasource), Flyway, neocortex `CbrRetrievalTrace`/`ExplanationRenderer` SPIs

## Global Constraints

- `CbrRetrievalLedgerEntry` must live in `io.casehub.clinical.ledger` (never `io.casehub.clinical.entity` — Panache cannot span two PUs)
- Flyway migration V2029 on qhorus datasource (`db/migration/qhorus/`)
- `domainContentBytes()` required on every `LedgerEntry` subclass — `LedgerProcessor` build-time validator enforces this
- `explanationText` column uses `VARCHAR(10000)` not `TEXT` (H2 compatibility in `@QuarkusTest`)
- Field named `retrievalTraceId` not `traceId` (avoids shadowing `LedgerEntry.traceId` — the OTel trace correlation field on the base table)
- `@Transactional(REQUIRES_NEW)` on the writer — observer/isolation pattern, not primary writer pattern
- `ledgerEntryRepository.save(entry, "default")` — hardcoded datasource name matching all existing writers
- `actorType = ActorType.USER` — all current callers are REST endpoints with human principals
- Response wrapper records go in `api/` module; production classes in `runtime/`

---

### Task 1: Response Wrapper Records + AuditedRetrievalResult

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/model/AePrecedentSearchResponse.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/DeviationPrecedentSearchResponse.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/AmendmentPrecedentSearchResponse.java`
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/AuditedRetrievalResult.java`

**Interfaces:**
- Produces: `AePrecedentSearchResponse(String traceId, String explanation, List<AePrecedentResponse> precedents)`, `DeviationPrecedentSearchResponse(String traceId, String explanation, List<DeviationPrecedentResponse> precedents)`, `AmendmentPrecedentSearchResponse(String traceId, String explanation, List<AmendmentPrecedentResponse> precedents)`, `AuditedRetrievalResult<C extends CbrCase>(List<ScoredCbrCase<C>> cases, String traceId, String explanation)`

- [ ] **Step 1: Create AePrecedentSearchResponse**

```java
package io.casehub.clinical.api.model;

import java.util.List;

public record AePrecedentSearchResponse(
    String traceId,
    String explanation,
    List<AePrecedentResponse> precedents) {}
```

Use `ide_create_file` for `api/src/main/java/io/casehub/clinical/api/model/AePrecedentSearchResponse.java`.

- [ ] **Step 2: Create DeviationPrecedentSearchResponse**

```java
package io.casehub.clinical.api.model;

import java.util.List;

public record DeviationPrecedentSearchResponse(
    String traceId,
    String explanation,
    List<DeviationPrecedentResponse> precedents) {}
```

Use `ide_create_file` for `api/src/main/java/io/casehub/clinical/api/model/DeviationPrecedentSearchResponse.java`.

- [ ] **Step 3: Create AmendmentPrecedentSearchResponse**

```java
package io.casehub.clinical.api.model;

import java.util.List;

public record AmendmentPrecedentSearchResponse(
    String traceId,
    String explanation,
    List<AmendmentPrecedentResponse> precedents) {}
```

Use `ide_create_file` for `api/src/main/java/io/casehub/clinical/api/model/AmendmentPrecedentSearchResponse.java`.

- [ ] **Step 4: Create AuditedRetrievalResult**

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;

import java.util.List;

public record AuditedRetrievalResult<C extends CbrCase>(
    List<ScoredCbrCase<C>> cases,
    String traceId,
    String explanation) {}
```

Use `ide_create_file` for `runtime/src/main/java/io/casehub/clinical/cbr/AuditedRetrievalResult.java`.

- [ ] **Step 5: Build api module and verify**

Run: `mvn install -pl api --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/clinical/api/model/AePrecedentSearchResponse.java \
       api/src/main/java/io/casehub/clinical/api/model/DeviationPrecedentSearchResponse.java \
       api/src/main/java/io/casehub/clinical/api/model/AmendmentPrecedentSearchResponse.java \
       runtime/src/main/java/io/casehub/clinical/cbr/AuditedRetrievalResult.java
git commit -m "feat(#117): add CBR retrieval response wrappers and AuditedRetrievalResult

Refs casehubio/clinical#117"
```

---

### Task 2: ClinicalExplanationRenderer

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalExplanationRenderer.java`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/ClinicalExplanationRendererTest.java`

**Interfaces:**
- Consumes: `ExplanationRenderer` SPI (neocortex), `CbrRetrievalTrace` record (neocortex)
- Produces: `ClinicalExplanationRenderer implements ExplanationRenderer` — `@ApplicationScoped`, displaces `DefaultExplanationRenderer` (`@DefaultBean`)

- [ ] **Step 1: Write failing tests**

Create test file via `ide_create_file` at `runtime/src/test/java/io/casehub/clinical/cbr/ClinicalExplanationRendererTest.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.*;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.platform.api.path.Path;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ClinicalExplanationRendererTest {

    private ClinicalExplanationRenderer renderer;

    @BeforeEach
    void setUp() {
        renderer = new ClinicalExplanationRenderer();
    }

    @Test
    void render_aeTrace_producesStructuredExplanation() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of("grade", FeatureValue.of(3)), 10)
            .withMinSimilarity(0.3);

        var trace = new CbrRetrievalTrace("trace-1", query, List.of(
            new CbrRetrievalTrace.TracedCase("case-1", 0.92, false,
                Map.of("grade", 1.0, "eventType", 0.95, "trialPhase", 0.80), 0.85),
            new CbrRetrievalTrace.TracedCase("case-2", 0.78, false,
                Map.of("grade", 0.8, "eventType", 0.70), 0.60)
        ), Instant.now());

        String result = renderer.render(trace);

        assertThat(result).contains("Adverse event precedent consultation");
        assertThat(result).contains("2 prior cases retrieved");
        assertThat(result).contains("score 0.92");
        assertThat(result).contains("grade=1.00");
        assertThat(result).contains("clinical-ae");
    }

    @Test
    void render_deviationTrace_usesDeviationLabel() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-deviation"),
            Path.root(), "clinical-deviation", Map.of(), 10);

        var trace = new CbrRetrievalTrace("trace-2", query, List.of(
            new CbrRetrievalTrace.TracedCase("case-1", 0.75, false, Map.of(), null)
        ), Instant.now());

        String result = renderer.render(trace);

        assertThat(result).contains("Protocol deviation precedent consultation");
        assertThat(result).contains("1 prior case retrieved");
    }

    @Test
    void render_amendmentTrace_usesAmendmentLabel() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-amendment"),
            Path.root(), "clinical-amendment", Map.of(), 10);

        var trace = new CbrRetrievalTrace("trace-3", query, List.of(), Instant.now());

        String result = renderer.render(trace);

        assertThat(result).contains("Protocol amendment precedent consultation");
        assertThat(result).contains("0 prior cases retrieved");
    }

    @Test
    void render_emptyResults_noNpeOrDivideByZero() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);

        var trace = new CbrRetrievalTrace("trace-4", query, List.of(), Instant.now());

        String result = renderer.render(trace);

        assertThat(result).contains("0 prior cases retrieved");
        assertThat(result).doesNotContain("Top precedent");
    }

    @Test
    void render_nullConfidence_handledGracefully() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);

        var trace = new CbrRetrievalTrace("trace-5", query, List.of(
            new CbrRetrievalTrace.TracedCase("case-1", 0.88, false, Map.of(), null),
            new CbrRetrievalTrace.TracedCase("case-2", 0.72, false, Map.of(), 0.90)
        ), Instant.now());

        String result = renderer.render(trace);

        assertThat(result).contains("1 has no recorded confidence");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalExplanationRendererTest --batch-mode`
Expected: FAIL — `ClinicalExplanationRenderer` does not exist

- [ ] **Step 3: Implement ClinicalExplanationRenderer**

Create via `ide_create_file` at `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalExplanationRenderer.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.CbrRetrievalTrace;
import io.casehub.neocortex.memory.cbr.ExplanationRenderer;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;
import java.util.stream.Collectors;

@ApplicationScoped
public class ClinicalExplanationRenderer implements ExplanationRenderer {

    @Override
    public String render(CbrRetrievalTrace trace) {
        int count = trace.results().size();
        String domain = trace.query().domain().name();
        String label = domainLabel(domain);

        StringBuilder sb = new StringBuilder();
        sb.append(label).append(": ");
        sb.append(count).append(count == 1 ? " prior case retrieved" : " prior cases retrieved");
        sb.append(" (min similarity ").append(String.format("%.2f", trace.query().minSimilarity())).append(").");

        if (!trace.results().isEmpty()) {
            var top = trace.results().getFirst();
            sb.append("\nTop precedent: score ").append(String.format("%.2f", top.score()));
            if (top.confidence() != null) {
                sb.append(", confidence ").append(String.format("%.2f", top.confidence()));
            }
            sb.append(".");

            if (!top.featureSimilarities().isEmpty()) {
                String breakdown = top.featureSimilarities().entrySet().stream()
                    .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                    .map(e -> e.getKey() + "=" + String.format("%.2f", e.getValue()))
                    .collect(Collectors.joining(", "));
                sb.append(" Feature alignment: ").append(breakdown).append(".");
            }

            long withConfidence = trace.results().stream()
                .filter(r -> r.confidence() != null && r.confidence() >= 0.70)
                .count();
            long withoutConfidence = trace.results().stream()
                .filter(r -> r.confidence() == null)
                .count();

            sb.append("\nConfidence band: ");
            sb.append(withConfidence).append(" of ").append(count);
            sb.append(withConfidence == 1 ? " precedent has" : " precedents have");
            sb.append(" confidence >= 0.70.");
            if (withoutConfidence > 0) {
                sb.append(" ").append(withoutConfidence);
                sb.append(withoutConfidence == 1 ? " has" : " have");
                sb.append(" no recorded confidence.");
            }
        }

        sb.append("\nQuery domain: ").append(domain);
        sb.append(". Retrieval mode: ").append(trace.query().retrievalMode()).append(".");

        return sb.toString();
    }

    private static String domainLabel(String domain) {
        return switch (domain) {
            case "clinical-ae" -> "Adverse event precedent consultation";
            case "clinical-deviation" -> "Protocol deviation precedent consultation";
            case "clinical-amendment" -> "Protocol amendment precedent consultation";
            default -> "Precedent consultation (" + domain + ")";
        };
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=ClinicalExplanationRendererTest --batch-mode`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/cbr/ClinicalExplanationRenderer.java \
       runtime/src/test/java/io/casehub/clinical/cbr/ClinicalExplanationRendererTest.java
git commit -m "feat(#117): ClinicalExplanationRenderer — FDA-structured explanation text

Displaces DefaultExplanationRenderer via CDI. Handles AE, deviation,
amendment domains with confidence band statistics.

Refs casehubio/clinical#117"
```

---

### Task 3: CbrRetrievalLedgerEntry + Flyway Migration

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/CbrRetrievalLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2029__cbr_retrieval_ledger_entry.sql`

**Interfaces:**
- Produces: `CbrRetrievalLedgerEntry extends JpaLedgerEntry` with fields `retrievalTraceId`, `queryDomain`, `queryFeaturesSummary`, `retrievedCaseCount`, `topScore`, `explanationText`

- [ ] **Step 1: Create Flyway migration**

Create `runtime/src/main/resources/db/migration/qhorus/V2029__cbr_retrieval_ledger_entry.sql`:

```sql
-- Ledger entry for CBR precedent consultation audit trail
CREATE TABLE cbr_retrieval_ledger_entry (
    id                      UUID PRIMARY KEY REFERENCES ledger_entry(id),
    retrieval_trace_id      VARCHAR(36)    NOT NULL,
    query_domain            VARCHAR(50)    NOT NULL,
    query_features_summary  VARCHAR(2000)  NOT NULL,
    retrieved_case_count    INT            NOT NULL,
    top_score               DOUBLE PRECISION NOT NULL,
    explanation_text        VARCHAR(10000)
);
```

- [ ] **Step 2: Create CbrRetrievalLedgerEntry**

Create via `ide_create_file` at `runtime/src/main/java/io/casehub/clinical/ledger/CbrRetrievalLedgerEntry.java`:

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry;
import jakarta.persistence.*;
import java.nio.charset.StandardCharsets;

@Entity
@Table(name = "cbr_retrieval_ledger_entry")
@DiscriminatorValue("CBR_RETRIEVAL")
public class CbrRetrievalLedgerEntry extends JpaLedgerEntry {

    @Column(name = "retrieval_trace_id", nullable = false, length = 36)
    public String retrievalTraceId;

    @Column(name = "query_domain", nullable = false, length = 50)
    public String queryDomain;

    @Column(name = "query_features_summary", nullable = false, length = 2000)
    public String queryFeaturesSummary;

    @Column(name = "retrieved_case_count", nullable = false)
    public int retrievedCaseCount;

    @Column(name = "top_score", nullable = false)
    public double topScore;

    @Column(name = "explanation_text", length = 10000)
    public String explanationText;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                retrievalTraceId    != null ? retrievalTraceId    : "",
                queryDomain         != null ? queryDomain         : "",
                queryFeaturesSummary != null ? queryFeaturesSummary : "",
                String.valueOf(retrievedCaseCount),
                String.valueOf(topScore),
                explanationText     != null ? explanationText     : "")
                .getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 3: Build and verify**

Run: `mvn compile -pl runtime --batch-mode`
Expected: BUILD SUCCESS — JPA entity compiles, migration file on classpath

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/ledger/CbrRetrievalLedgerEntry.java \
       runtime/src/main/resources/db/migration/qhorus/V2029__cbr_retrieval_ledger_entry.sql
git commit -m "feat(#117): CbrRetrievalLedgerEntry + V2029 Flyway migration

JOINED inheritance on qhorus datasource. retrievalTraceId avoids
shadowing LedgerEntry.traceId (OTel). VARCHAR(10000) for H2 compat.

Refs casehubio/clinical#117"
```

---

### Task 4: CbrRetrievalLedgerWriter + ClinicalComplianceSupplement

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/CbrRetrievalLedgerWriter.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/CbrRetrievalLedgerWriterTest.java`

**Interfaces:**
- Consumes: `CbrRetrievalLedgerEntry` (Task 3), `CbrRetrievalTrace` (neocortex), `LedgerEntryRepository` (ledger), `ClinicalComplianceSupplement` (existing)
- Produces: `CbrRetrievalLedgerWriter.record(CbrRetrievalTrace, String explanation, UUID subjectId, String actorId)` — `@Transactional(REQUIRES_NEW)`

- [ ] **Step 1: Add cbrRetrieval() to ClinicalComplianceSupplement**

Use `ide_insert_member` to add after the `regulatorySubmissionBreach` method in `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`:

```java
public static ComplianceSupplement cbrRetrieval() {
    ComplianceSupplement s = new JpaComplianceSupplement();
    s.planRef = "EU AI Act Art.12 — record-keeping for high-risk AI decision support";
    s.algorithmRef = "ClinicalCbrService (CBR precedent retrieval, weighted feature similarity)";
    s.humanOverrideAvailable = true;
    return s;
}
```

- [ ] **Step 2: Write failing tests for CbrRetrievalLedgerWriter**

Create test file via `ide_create_file` at `runtime/src/test/java/io/casehub/clinical/service/CbrRetrievalLedgerWriterTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.ledger.CbrRetrievalLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.path.Path;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class CbrRetrievalLedgerWriterTest {

    private LedgerEntryRepository repository;
    private CbrRetrievalLedgerWriter writer;
    private Clock clock;

    @BeforeEach
    void setUp() {
        repository = mock(LedgerEntryRepository.class);
        clock = Clock.fixed(Instant.parse("2026-07-17T10:00:00Z"), ZoneOffset.UTC);
        writer = new CbrRetrievalLedgerWriter();
        writer.ledgerEntryRepository = repository;
        writer.clock = clock;

        when(repository.findLatestBySubjectId(any(), eq("default"))).thenReturn(Optional.empty());
    }

    @Test
    void record_writesEntryWithCorrectFields() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of("grade", FeatureValue.of(3)), 10);

        var trace = new CbrRetrievalTrace("trace-abc", query, List.of(
            new CbrRetrievalTrace.TracedCase("case-1", 0.92, false, Map.of(), 0.85)
        ), Instant.now());

        UUID subjectId = UUID.randomUUID();
        writer.record(trace, "Explanation text", subjectId, "clinician-1");

        ArgumentCaptor<CbrRetrievalLedgerEntry> captor = ArgumentCaptor.forClass(CbrRetrievalLedgerEntry.class);
        verify(repository).save(captor.capture(), eq("default"));

        CbrRetrievalLedgerEntry entry = captor.getValue();
        assertThat(entry.retrievalTraceId).isEqualTo("trace-abc");
        assertThat(entry.queryDomain).isEqualTo("clinical-ae");
        assertThat(entry.queryFeaturesSummary).contains("grade=");
        assertThat(entry.retrievedCaseCount).isEqualTo(1);
        assertThat(entry.topScore).isEqualTo(0.92);
        assertThat(entry.explanationText).isEqualTo("Explanation text");
        assertThat(entry.subjectId).isEqualTo(subjectId);
        assertThat(entry.actorId).isEqualTo("clinician-1");
        assertThat(entry.actorType).isEqualTo(ActorType.USER);
        assertThat(entry.actorRole).isEqualTo("cbr-retrieval-auditor");
        assertThat(entry.entryType).isEqualTo(LedgerEntryType.EVENT);
        assertThat(entry.occurredAt).isEqualTo(clock.instant());
        assertThat(entry.sequenceNumber).isEqualTo(1);
    }

    @Test
    void record_nullExplanation_writesEntryWithNullExplanation() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);

        var trace = new CbrRetrievalTrace("trace-null", query, List.of(), Instant.now());

        writer.record(trace, null, UUID.randomUUID(), "clinician-1");

        ArgumentCaptor<CbrRetrievalLedgerEntry> captor = ArgumentCaptor.forClass(CbrRetrievalLedgerEntry.class);
        verify(repository).save(captor.capture(), eq("default"));

        assertThat(captor.getValue().explanationText).isNull();
    }

    @Test
    void record_emptyResults_topScoreIsZero() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);

        var trace = new CbrRetrievalTrace("trace-empty", query, List.of(), Instant.now());

        writer.record(trace, "No cases found", UUID.randomUUID(), "clinician-1");

        ArgumentCaptor<CbrRetrievalLedgerEntry> captor = ArgumentCaptor.forClass(CbrRetrievalLedgerEntry.class);
        verify(repository).save(captor.capture(), eq("default"));

        assertThat(captor.getValue().retrievedCaseCount).isZero();
        assertThat(captor.getValue().topScore).isEqualTo(0.0);
    }

    @Test
    void record_attachesComplianceSupplement() {
        CbrQuery query = CbrQuery.of("tenant-1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);

        var trace = new CbrRetrievalTrace("trace-cs", query, List.of(), Instant.now());

        writer.record(trace, "text", UUID.randomUUID(), "clinician-1");

        ArgumentCaptor<CbrRetrievalLedgerEntry> captor = ArgumentCaptor.forClass(CbrRetrievalLedgerEntry.class);
        verify(repository).save(captor.capture(), eq("default"));

        assertThat(captor.getValue().supplements).isNotEmpty();
    }

    @Test
    void domainContentBytes_isStable() {
        var entry = new CbrRetrievalLedgerEntry();
        entry.retrievalTraceId = "trace-1";
        entry.queryDomain = "clinical-ae";
        entry.queryFeaturesSummary = "grade=3";
        entry.retrievedCaseCount = 2;
        entry.topScore = 0.92;
        entry.explanationText = "Explanation";

        byte[] first = entry.domainContentBytes();
        byte[] second = entry.domainContentBytes();

        assertThat(first).isEqualTo(second);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=CbrRetrievalLedgerWriterTest --batch-mode`
Expected: FAIL — `CbrRetrievalLedgerWriter` does not exist

- [ ] **Step 4: Implement CbrRetrievalLedgerWriter**

Create via `ide_create_file` at `runtime/src/main/java/io/casehub/clinical/service/CbrRetrievalLedgerWriter.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.ledger.CbrRetrievalLedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.neocortex.memory.cbr.CbrRetrievalTrace;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.time.Clock;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class CbrRetrievalLedgerWriter {

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Clock clock;

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void record(CbrRetrievalTrace trace, String explanation,
                       UUID subjectId, String actorId) {
        var entry = new CbrRetrievalLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = subjectId;
        entry.sequenceNumber = nextSequenceNumber(subjectId);
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.USER;
        entry.actorRole = "cbr-retrieval-auditor";
        entry.occurredAt = clock.instant();
        entry.retrievalTraceId = trace.traceId();
        entry.queryDomain = trace.query().domain().name();
        entry.queryFeaturesSummary = summariseFeatures(trace.query().features());
        entry.retrievedCaseCount = trace.results().size();
        entry.topScore = trace.results().isEmpty() ? 0.0
            : trace.results().getFirst().score();
        entry.explanationText = explanation;
        entry.attach(ClinicalComplianceSupplement.cbrRetrieval());
        ledgerEntryRepository.save(entry, "default");
    }

    private int nextSequenceNumber(UUID subjectId) {
        return ledgerEntryRepository.findLatestBySubjectId(subjectId, "default")
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }

    private static String summariseFeatures(Map<String, FeatureValue> features) {
        if (features.isEmpty()) return "";
        return features.entrySet().stream()
            .map(e -> e.getKey() + "=" + e.getValue().displayValue())
            .collect(Collectors.joining(","));
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=CbrRetrievalLedgerWriterTest --batch-mode`
Expected: PASS (5 tests)

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/service/CbrRetrievalLedgerWriter.java \
       runtime/src/test/java/io/casehub/clinical/service/CbrRetrievalLedgerWriterTest.java \
       runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java
git commit -m "feat(#117): CbrRetrievalLedgerWriter + cbrRetrieval() compliance supplement

REQUIRES_NEW observer/isolation pattern. LedgerEntryType.EVENT,
ActorType.USER, actorRole=cbr-retrieval-auditor.

Refs casehubio/clinical#117"
```

---

### Task 5: ClinicalCbrService.retrieveWithAudit() + Endpoint Updates

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCbrService.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialDashboardResource.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/ProtocolAmendmentResource.java`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/ClinicalCbrServiceAuditTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/CbrRetrievalAuditIntegrationTest.java`

**Interfaces:**
- Consumes: `AuditedRetrievalResult<C>` (Task 1), `ClinicalExplanationRenderer` (Task 2), `CbrRetrievalLedgerWriter` (Task 4), `AePrecedentSearchResponse` / `DeviationPrecedentSearchResponse` / `AmendmentPrecedentSearchResponse` (Task 1)
- Produces: `ClinicalCbrService.retrieveWithAudit(CbrQuery, Class<C>, UUID subjectId, String actorId) → AuditedRetrievalResult<C>`

- [ ] **Step 1: Write failing unit tests for retrieveWithAudit**

Create via `ide_create_file` at `runtime/src/test/java/io/casehub/clinical/cbr/ClinicalCbrServiceAuditTest.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.service.CbrRetrievalLedgerWriter;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

class ClinicalCbrServiceAuditTest {

    private CbrCaseMemoryStore store;
    private ExplanationRenderer renderer;
    private CbrRetrievalLedgerWriter writer;
    private ClinicalCbrService service;

    @BeforeEach
    void setUp() {
        store = mock(CbrCaseMemoryStore.class);
        renderer = mock(ExplanationRenderer.class);
        writer = mock(CbrRetrievalLedgerWriter.class);
        Clock clock = Clock.fixed(Instant.parse("2026-07-17T10:00:00Z"), ZoneOffset.UTC);
        service = new ClinicalCbrService(store, renderer, writer, clock);
    }

    @Test
    void retrieveWithAudit_callsRetrieveThenRenderThenWrite() {
        CbrQuery query = CbrQuery.of("t1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);
        var scored = new ScoredCbrCase<>(mock(PlanCbrCase.class), "c1", 0.9);
        when(store.retrieveSimilar(query, PlanCbrCase.class)).thenReturn(List.of(scored));
        when(renderer.render(any())).thenReturn("explanation-text");

        UUID subjectId = UUID.randomUUID();
        var result = service.retrieveWithAudit(query, PlanCbrCase.class, subjectId, "actor-1");

        assertThat(result.cases()).hasSize(1);
        assertThat(result.traceId()).isNotNull();
        assertThat(result.explanation()).isEqualTo("explanation-text");

        verify(renderer).render(any(CbrRetrievalTrace.class));
        verify(writer).record(any(CbrRetrievalTrace.class), eq("explanation-text"),
            eq(subjectId), eq("actor-1"));
    }

    @Test
    void retrieveWithAudit_renderThrows_explanationNullButLedgerWritten() {
        CbrQuery query = CbrQuery.of("t1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);
        when(store.retrieveSimilar(query, PlanCbrCase.class)).thenReturn(List.of());
        when(renderer.render(any())).thenThrow(new RuntimeException("render failed"));

        var result = service.retrieveWithAudit(query, PlanCbrCase.class, UUID.randomUUID(), "actor-1");

        assertThat(result.explanation()).isNull();
        verify(writer).record(any(), isNull(), any(), eq("actor-1"));
    }

    @Test
    void retrieveWithAudit_writerThrows_propagatesToCaller() {
        CbrQuery query = CbrQuery.of("t1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);
        when(store.retrieveSimilar(query, PlanCbrCase.class)).thenReturn(List.of());
        when(renderer.render(any())).thenReturn("text");
        doThrow(new RuntimeException("ledger write failed")).when(writer).record(any(), any(), any(), any());

        assertThatThrownBy(() ->
            service.retrieveWithAudit(query, PlanCbrCase.class, UUID.randomUUID(), "actor-1"))
            .hasMessageContaining("ledger write failed");
    }

    @Test
    void retrieveWithAudit_emptyResults_stillWritesLedgerEntry() {
        CbrQuery query = CbrQuery.of("t1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);
        when(store.retrieveSimilar(query, PlanCbrCase.class)).thenReturn(List.of());
        when(renderer.render(any())).thenReturn("0 cases");

        service.retrieveWithAudit(query, PlanCbrCase.class, UUID.randomUUID(), "actor-1");

        verify(writer).record(any(), eq("0 cases"), any(), any());
    }

    @Test
    void retrieveSimilar_unchanged_noAudit() {
        CbrQuery query = CbrQuery.of("t1", new MemoryDomain("clinical-ae"),
            Path.root(), "clinical-ae", Map.of(), 10);
        when(store.retrieveSimilar(query, PlanCbrCase.class)).thenReturn(List.of());

        service.retrieveSimilar(query, PlanCbrCase.class);

        verifyNoInteractions(renderer);
        verifyNoInteractions(writer);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=ClinicalCbrServiceAuditTest --batch-mode`
Expected: FAIL — `ClinicalCbrService` constructor doesn't accept renderer/writer/clock yet

- [ ] **Step 3: Update ClinicalCbrService with retrieveWithAudit**

Use `ide_edit_member` to replace the `ClinicalCbrService` class declaration + constructor, and `ide_insert_member` to add the new method. The updated class:

Replace the constructor and add imports via `ide_edit_member` on `ClinicalCbrService`:

New constructor:
```java
@Inject
public ClinicalCbrService(CbrCaseMemoryStore store,
                           ExplanationRenderer explanationRenderer,
                           CbrRetrievalLedgerWriter ledgerWriter,
                           Clock clock) {
    this.store = store;
    this.explanationRenderer = explanationRenderer;
    this.ledgerWriter = ledgerWriter;
    this.clock = clock;
}
```

Add fields after existing `store` field:
```java
private final ExplanationRenderer explanationRenderer;
private final CbrRetrievalLedgerWriter ledgerWriter;
private final Clock clock;
```

Add `retrieveWithAudit` method after `retrieveSimilar`:
```java
public <C extends CbrCase> AuditedRetrievalResult<C> retrieveWithAudit(
        CbrQuery query, Class<C> caseType,
        UUID subjectId, String actorId) {
    List<ScoredCbrCase<C>> cases = retrieveSimilar(query, caseType);
    CbrRetrievalTrace trace = buildTrace(query, cases);

    String explanation;
    try {
        explanation = explanationRenderer.render(trace);
    } catch (Exception e) {
        LOG.warnf(e, "ExplanationRenderer failed for trace %s — recording with null explanation", trace.traceId());
        explanation = null;
    }

    ledgerWriter.record(trace, explanation, subjectId, actorId);
    return new AuditedRetrievalResult<>(cases, trace.traceId(), explanation);
}

private <C extends CbrCase> CbrRetrievalTrace buildTrace(CbrQuery query, List<ScoredCbrCase<C>> cases) {
    List<CbrRetrievalTrace.TracedCase> tracedCases = cases.stream()
        .map(sc -> new CbrRetrievalTrace.TracedCase(
            sc.caseId(), sc.score(), sc.reranked(),
            sc.featureSimilarities(),
            sc.cbrCase().confidence()))
        .toList();
    return new CbrRetrievalTrace(
        UUID.randomUUID().toString(), query, tracedCases, clock.instant());
}
```

Add `LOG` field if not present:
```java
private static final Logger LOG = Logger.getLogger(ClinicalCbrService.class);
```

Add required imports: `java.util.UUID`, `java.time.Clock`, `io.casehub.neocortex.memory.cbr.CbrRetrievalTrace`, `io.casehub.neocortex.memory.cbr.ExplanationRenderer`, `io.casehub.clinical.service.CbrRetrievalLedgerWriter`, `org.jboss.logging.Logger`

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=ClinicalCbrServiceAuditTest --batch-mode`
Expected: PASS (5 tests)

- [ ] **Step 5: Also verify existing ClinicalCbrServiceTest still passes**

Run: `mvn test -pl runtime -Dtest=ClinicalCbrServiceTest --batch-mode`
Expected: PASS — but this test will need updating because the constructor changed.

If the test fails because the constructor now requires 4 args, update the test setup to pass `mock(ExplanationRenderer.class)`, `mock(CbrRetrievalLedgerWriter.class)`, and a fixed `Clock`.

- [ ] **Step 6: Update ClinicalCaseOutcomeObserver constructor call**

`ClinicalCaseOutcomeObserver` injects `ClinicalCbrService`. The service's constructor changed — CDI handles this automatically (all new params are injectable). But `ClinicalCaseOutcomeObserverTest` creates `ClinicalCbrService` manually via `mock()` — verify it still works (it should, since mock doesn't call the constructor).

Run: `mvn test -pl runtime -Dtest=ClinicalCaseOutcomeObserverTest --batch-mode`
Expected: PASS

- [ ] **Step 7: Update TrialDashboardResource.aePrecedents()**

Use `ide_replace_member` on `TrialDashboardResource.aePrecedents` to replace the last section (lines ~576-581) where `cbrService.retrieveSimilar(...)` is called. The new body ending:

```java
var result = cbrService.retrieveWithAudit(query, PlanCbrCase.class, aeId, principal.actorId());
List<AePrecedentResponse> precedents = result.cases().stream()
    .map(this::mapToAeResponse)
    .toList();
return Response.ok(new AePrecedentSearchResponse(result.traceId(), result.explanation(), precedents)).build();
```

- [ ] **Step 8: Update TrialDashboardResource.deviationPrecedents()**

Similarly, replace the retrieval section in `deviationPrecedents`:

```java
var result = cbrService.retrieveWithAudit(query, PlanCbrCase.class, devId, principal.actorId());
List<DeviationPrecedentResponse> precedents = result.cases().stream()
    .map(this::mapToDeviationResponse)
    .toList();
return Response.ok(new DeviationPrecedentSearchResponse(result.traceId(), result.explanation(), precedents)).build();
```

- [ ] **Step 9: Update ProtocolAmendmentResource.amendmentPrecedents()**

Replace the retrieval section:

```java
var result = cbrService.retrieveWithAudit(query, TextualCbrCase.class, amendmentId, principal.actorId());
List<AmendmentPrecedentResponse> precedents = result.cases().stream()
    .map(this::mapToAmendmentResponse)
    .toList();
return Response.ok(new AmendmentPrecedentSearchResponse(result.traceId(), result.explanation(), precedents)).build();
```

- [ ] **Step 10: Run full test suite**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: All tests pass. If any `TrialDashboardResourceTest` or `PrecedentEndpointTest` assertions fail on response shape, update them to expect the wrapper response (extract `precedents` list from the wrapper).

- [ ] **Step 11: Write integration test**

Create via `ide_create_file` at `runtime/src/test/java/io/casehub/clinical/cbr/CbrRetrievalAuditIntegrationTest.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR})
class CbrRetrievalAuditIntegrationTest {

    @Inject ClinicalCbrService cbrService;
    @Inject CbrCaseMemoryStore store;
    @Inject FixedCurrentPrincipal principal;

    @BeforeEach
    @Transactional
    void setUp() {
        store.erase(new io.casehub.neocortex.memory.EraseRequest(
            principal.tenancyId(), ClinicalCbrDomains.AE, null, null));
    }

    @Test
    void retrieveWithAudit_producesTraceIdAndExplanation() {
        var cbrCase = new PlanCbrCase(
            "Grade 3 Neutropenia", "Safety review: CONTINUE", "COMPLETED", 1.0,
            FeatureValue.toFeatureMap(Map.of("grade", 3, "eventType", "Neutropenia")),
            java.util.List.of());

        store.store(cbrCase, "clinical-ae", "ae-" + UUID.randomUUID(),
            ClinicalCbrDomains.AE, principal.tenancyId(), null, Path.root());

        CbrQuery query = CbrQuery.of(principal.tenancyId(), ClinicalCbrDomains.AE,
            Path.root(), "clinical-ae",
            FeatureValue.toFeatureMap(Map.of("grade", 3, "eventType", "Neutropenia")), 10)
            .withVectorWeight(0.0);

        var result = cbrService.retrieveWithAudit(query, PlanCbrCase.class,
            UUID.randomUUID(), principal.actorId());

        assertThat(result.traceId()).isNotNull();
        assertThat(result.explanation()).contains("precedent consultation");
        assertThat(result.cases()).isNotEmpty();
    }
}
```

- [ ] **Step 12: Run integration test**

Run: `mvn test -pl runtime -Dtest=CbrRetrievalAuditIntegrationTest --batch-mode`
Expected: PASS

- [ ] **Step 13: Run full test suite**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: All tests pass

- [ ] **Step 14: Commit**

```bash
git add -A
git commit -m "feat(#117): retrieveWithAudit pipeline + endpoint updates

ClinicalCbrService.retrieveWithAudit() composes retrieve → trace →
explain → audit → return. Three endpoints switch to audited retrieval
with wrapper responses including traceId and explanation.

Refs casehubio/clinical#117"
```
