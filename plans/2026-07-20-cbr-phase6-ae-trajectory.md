# CBR Phase 6 — AE Trajectory Monitoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #119 — feat: CBR Phase 6 — AE progression trajectory monitoring
**Issue group:** #119

**Goal:** Add trajectory-based CBR to clinical — capture AE state progressions and site enrollment rates as TimeSeries features, match developing trajectories against past cases via DTW, and fire proactive alerts when matches predict high-severity outcomes.

**Architecture:** Lazy trajectory reconstruction from existing data sources (AE entity + engine PlanItemStore + PatientEnrollment entities). No new CDI events for data collection — trajectories are built on demand. Alert services fire `AeTrajectoryAlertEvent` and `SiteEnrollmentAlertEvent` as output events. Neocortex `FeatureField.TimeSeries` with `DtwSpec(SakoeChibaBand(3))` and `TrendSpec(SLOPE, ACCELERATION, CHANGE_POINTS)` provides the similarity and trend infrastructure.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-neocortex-memory-api (FeatureField, CbrSimilarityScorer, TrendAnalyzer, CbrCaseMemoryStore), casehub-engine-common (PlanItemStore, PlanItemRecord)

## Global Constraints

- All code uses IntelliJ MCP tools for creation and editing — never bash Edit/Write on .java files
- `project_path` = `/Users/mdproctor/claude/casehub/clinical` for all IDE calls
- API module records go in `api/src/main/java/io/casehub/clinical/api/`
- Runtime services go in `runtime/src/main/java/io/casehub/clinical/cbr/` (CBR) or `.../service/` (lifecycle hooks)
- Tests in `runtime/src/test/java/` mirroring source package
- `ClinicalActors.CLINICAL_SERVICE` = `"clinical-service"` (already exists)
- `CbrQuery` has separate `features` (similarity-scored) and `filters` (hard exclusion via `CbrFilter`)
- `PlanItemRecord` has only `createdAt` — no `completedAt` (temporal precision limitation, engine#763)
- `WarpingConstraint.SakoeChibaBand(int windowSize)` — windowSize >= 1
- `TrendSpec(Set<TrendType> types, ChronoUnit timeUnit)`
- `SimilaritySpec.DtwSpec(WarpingConstraint constraint)`
- Build: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
- Single test: `mvn test -pl runtime -Dtest=ClassName --batch-mode`

---

### Task 1: Foundation — API Events, Domain Constants, Schema Registration

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/AeTrajectoryAlertEvent.java`
- Create: `api/src/main/java/io/casehub/clinical/api/SiteEnrollmentAlertEvent.java`
- Create: `api/src/main/java/io/casehub/clinical/api/TrialStatusChangedEvent.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCbrDomains.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCbrSchemaInitializer.java`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/ClinicalCbrSchemaInitializerTest.java`

**Interfaces:**
- Produces: `AeTrajectoryAlertEvent(UUID aeId, UUID enrollmentId, UUID siteId, CtcaeGrade currentGrade, int matchCount, double topScore, String predictedOutcome, double predictedProbability, String traceId, String tenantId)`
- Produces: `SiteEnrollmentAlertEvent(UUID siteId, UUID trialId, int matchCount, double topScore, String predictedOutcome, double predictedProbability, String traceId, String tenantId)`
- Produces: `TrialStatusChangedEvent(UUID trialId, TrialStatus oldStatus, TrialStatus newStatus, String tenantId)`
- Produces: `ClinicalCbrDomains.AE_TRAJECTORY`, `ClinicalCbrDomains.SITE_ENROLLMENT`
- Produces: `ClinicalCbrSchemaInitializer.aeTrajectorySchema()`, `ClinicalCbrSchemaInitializer.siteEnrollmentSchema()`

- [ ] **Step 1: Write tests for new schema registration**

Add tests to the existing `ClinicalCbrSchemaInitializerTest`:

```java
@Test
void aeTrajectorySchema_hasTimeSeriesFieldWithDtwAndTrend() {
    final ArgumentCaptor<CbrFeatureSchema> captor = ArgumentCaptor.forClass(CbrFeatureSchema.class);
    verify(store, times(5)).registerSchema(captor.capture());
    CbrFeatureSchema schema = captor.getAllValues().stream()
            .filter(s -> "clinical-ae-trajectory".equals(s.caseType()))
            .findFirst().orElseThrow();

    assertEquals("clinical-ae-trajectory", schema.caseType());
    var fieldNames = schema.fields().stream().map(FeatureField::name).toList();
    assertTrue(fieldNames.contains("grade"));
    assertTrue(fieldNames.contains("eventType"));
    assertTrue(fieldNames.contains("aeTrajectory"));

    var tsField = schema.fields().stream()
            .filter(f -> f instanceof FeatureField.TimeSeries)
            .map(f -> (FeatureField.TimeSeries) f)
            .findFirst().orElseThrow();
    assertEquals("aeTrajectory", tsField.name());
    assertInstanceOf(SimilaritySpec.DtwSpec.class, tsField.similaritySpec());
    assertNotNull(tsField.trendSpec());
    assertTrue(tsField.trendSpec().types().contains(TrendType.SLOPE));
    assertTrue(tsField.trendSpec().types().contains(TrendType.ACCELERATION));
    assertTrue(tsField.trendSpec().types().contains(TrendType.CHANGE_POINTS));
}

@Test
void siteEnrollmentSchema_hasTimeSeriesFieldWithDtwAndTrend() {
    final ArgumentCaptor<CbrFeatureSchema> captor = ArgumentCaptor.forClass(CbrFeatureSchema.class);
    verify(store, times(5)).registerSchema(captor.capture());
    CbrFeatureSchema schema = captor.getAllValues().stream()
            .filter(s -> "clinical-site-enrollment".equals(s.caseType()))
            .findFirst().orElseThrow();

    assertEquals("clinical-site-enrollment", schema.caseType());
    var tsField = schema.fields().stream()
            .filter(f -> f instanceof FeatureField.TimeSeries)
            .map(f -> (FeatureField.TimeSeries) f)
            .findFirst().orElseThrow();
    assertEquals("enrollmentRate", tsField.name());
    assertEquals("ts", tsField.timestampField());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalCbrSchemaInitializerTest --batch-mode`
Expected: FAIL — `times(5)` fails (currently 3 schemas), missing fields

- [ ] **Step 3: Create API event records**

Use `ide_create_file` for each:

`AeTrajectoryAlertEvent.java`:
```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.util.UUID;

public record AeTrajectoryAlertEvent(
    UUID aeId, UUID enrollmentId, UUID siteId,
    CtcaeGrade currentGrade,
    int matchCount, double topScore,
    String predictedOutcome, double predictedProbability,
    String traceId, String tenantId) {}
```

`SiteEnrollmentAlertEvent.java`:
```java
package io.casehub.clinical.api;

import java.util.UUID;

public record SiteEnrollmentAlertEvent(
    UUID siteId, UUID trialId,
    int matchCount, double topScore,
    String predictedOutcome, double predictedProbability,
    String traceId, String tenantId) {}
```

`TrialStatusChangedEvent.java`:
```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.TrialStatus;
import java.util.UUID;

public record TrialStatusChangedEvent(
    UUID trialId, TrialStatus oldStatus, TrialStatus newStatus,
    String tenantId) {}
```

- [ ] **Step 4: Add domain constants to ClinicalCbrDomains**

Use `ide_insert_member` to add after `AMENDMENT`:

```java
public static final MemoryDomain AE_TRAJECTORY = new MemoryDomain("clinical-ae-trajectory");
public static final MemoryDomain SITE_ENROLLMENT = new MemoryDomain("clinical-site-enrollment");
```

- [ ] **Step 5: Add schema methods to ClinicalCbrSchemaInitializer**

Use `ide_replace_member` on `onStartup` to register 5 schemas:

```java
void onStartup(@Observes final StartupEvent event) {
    LOG.info("Registering CBR schemas: clinical-ae, clinical-deviation, clinical-amendment, clinical-ae-trajectory, clinical-site-enrollment");
    store.registerSchema(aeSchema());
    store.registerSchema(deviationSchema());
    store.registerSchema(amendmentSchema());
    store.registerSchema(aeTrajectorySchema());
    store.registerSchema(siteEnrollmentSchema());
}
```

Use `ide_insert_member` after `amendmentSchema()`:

```java
static CbrFeatureSchema aeTrajectorySchema() {
    return CbrFeatureSchema.of("clinical-ae-trajectory",
        FeatureField.numeric("grade", 1, 5),
        FeatureField.categorical("eventType"),
        FeatureField.categorical("trialPhase"),
        FeatureField.categorical("unexpected"),
        FeatureField.categorical("suspected"),
        FeatureField.timeSeries("aeTrajectory", "ts",
            new SimilaritySpec.DtwSpec(new WarpingConstraint.SakoeChibaBand(3)),
            new TrendSpec(Set.of(TrendType.SLOPE, TrendType.ACCELERATION, TrendType.CHANGE_POINTS), ChronoUnit.HOURS),
            FeatureField.numeric("ts", 0, 7776000),
            FeatureField.numeric("escalation", 0, 3),
            FeatureField.numeric("susar", 0, 3),
            FeatureField.numeric("regulatory", 0, 3)));
}

static CbrFeatureSchema siteEnrollmentSchema() {
    return CbrFeatureSchema.of("clinical-site-enrollment",
        FeatureField.categorical("trialPhase"),
        FeatureField.timeSeries("enrollmentRate", "ts",
            new SimilaritySpec.DtwSpec(new WarpingConstraint.SakoeChibaBand(3)),
            new TrendSpec(Set.of(TrendType.SLOPE, TrendType.ACCELERATION, TrendType.CHANGE_POINTS), ChronoUnit.WEEKS),
            FeatureField.numeric("ts", 0, 260),
            FeatureField.numeric("cumulativeCount", 0, 10000),
            FeatureField.numeric("periodCount", 0, 500)));
}
```

Add imports: `java.time.temporal.ChronoUnit`, `java.util.Set`, `TrendSpec`, `TrendType`, `SimilaritySpec`, `WarpingConstraint`.

- [ ] **Step 6: Install api, run tests**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalCbrSchemaInitializerTest --batch-mode`
Expected: PASS — all schema tests green

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on `ClinicalCbrSchemaInitializer.java` and all three event files.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/src/main/java/io/casehub/clinical/api/AeTrajectoryAlertEvent.java api/src/main/java/io/casehub/clinical/api/SiteEnrollmentAlertEvent.java api/src/main/java/io/casehub/clinical/api/TrialStatusChangedEvent.java runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCbrDomains.java runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCbrSchemaInitializer.java runtime/src/test/java/io/casehub/clinical/cbr/ClinicalCbrSchemaInitializerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#119): CBR Phase 6 foundation — trajectory events, domains, schemas"
```

---

### Task 2: AeTrajectoryBuilder

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/AeTrajectoryBuilder.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/AeTrajectoryBuilderTest.java`

**Interfaces:**
- Consumes: `PlanItemStore.findByCaseId(UUID caseId, String tenancyId)` → `List<PlanItemRecord>`
- Consumes: `AdverseEvent` entity fields: `reportedAt`, `escalationStatus`, `susarOversightStatus`, `regulatorySubmissionStatus`, `engineCaseId`, `susarOversightCaseId`, `regulatorySubmissionCaseId`
- Produces: `buildTrajectory(AdverseEvent ae, String tenantId)` → `List<Map<String, FeatureValue>>`
- Produces: `buildPartialTrajectory(AdverseEvent ae, String tenantId)` → `List<Map<String, FeatureValue>>`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.api.model.TaskStatus;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class AeTrajectoryBuilderTest {

    private PlanItemStore planItemStore;
    private AeTrajectoryBuilder builder;

    @BeforeEach
    void setUp() {
        planItemStore = mock(PlanItemStore.class);
        builder = new AeTrajectoryBuilder(planItemStore);
    }

    @Test
    void noEngineCases_singleObservationFromEntity() {
        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_1, null, null, null);
        var trajectory = builder.buildTrajectory(ae, "tenant-1");

        assertEquals(1, trajectory.size());
        var obs = trajectory.get(0);
        assertEquals(0.0, ((FeatureValue.NumberVal) obs.get("ts")).value());
        assertEquals(0.0, ((FeatureValue.NumberVal) obs.get("escalation")).value());
        assertEquals(0.0, ((FeatureValue.NumberVal) obs.get("susar")).value());
        assertEquals(0.0, ((FeatureValue.NumberVal) obs.get("regulatory")).value());
    }

    @Test
    void withEngineCaseOnly_buildsTrajectoryFromPlanItems() {
        UUID caseId = UUID.randomUUID();
        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_3, caseId, null, null);
        ae.escalationStatus = AeEscalationStatus.COMPLETED;

        Instant base = ae.reportedAt;
        when(planItemStore.findByCaseId(caseId, "tenant-1")).thenReturn(List.of(
            planItem(caseId, "safety-review", TaskStatus.COMPLETED, base.plusSeconds(3600)),
            planItem(caseId, "dsmb-escalation", TaskStatus.COMPLETED, base.plusSeconds(7200))
        ));

        var trajectory = builder.buildTrajectory(ae, "tenant-1");

        assertTrue(trajectory.size() >= 3);
        // First observation is the initial report at t=0
        assertEquals(0.0, ((FeatureValue.NumberVal) trajectory.get(0).get("ts")).value());
        // Subsequent observations have positive timestamps
        double prevTs = 0;
        for (int i = 1; i < trajectory.size(); i++) {
            double ts = ((FeatureValue.NumberVal) trajectory.get(i).get("ts")).value();
            assertTrue(ts >= prevTs, "Observations must be sorted by timestamp");
            prevTs = ts;
        }
    }

    @Test
    void multipleEngineCases_mergesAndSorts() {
        UUID escalationCaseId = UUID.randomUUID();
        UUID susarCaseId = UUID.randomUUID();
        UUID regCaseId = UUID.randomUUID();
        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_4, escalationCaseId, susarCaseId, regCaseId);
        ae.escalationStatus = AeEscalationStatus.COMPLETED;
        ae.susarOversightStatus = SusarOversightStatus.COMPLETED;
        ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.REQUESTED;

        Instant base = ae.reportedAt;
        when(planItemStore.findByCaseId(escalationCaseId, "tenant-1")).thenReturn(List.of(
            planItem(escalationCaseId, "safety-review", TaskStatus.COMPLETED, base.plusSeconds(1800))
        ));
        when(planItemStore.findByCaseId(susarCaseId, "tenant-1")).thenReturn(List.of(
            planItem(susarCaseId, "susar-assessment", TaskStatus.COMPLETED, base.plusSeconds(3600))
        ));
        when(planItemStore.findByCaseId(regCaseId, "tenant-1")).thenReturn(List.of(
            planItem(regCaseId, "ind-submission", TaskStatus.ACTIVE, base.plusSeconds(5400))
        ));

        var trajectory = builder.buildTrajectory(ae, "tenant-1");

        assertTrue(trajectory.size() >= 4);
        // Verify all observations have all 4 inner fields
        for (var obs : trajectory) {
            assertNotNull(obs.get("ts"));
            assertNotNull(obs.get("escalation"));
            assertNotNull(obs.get("susar"));
            assertNotNull(obs.get("regulatory"));
        }
    }

    @Test
    void partialTrajectory_sameAsFullForDevelopingCase() {
        UUID caseId = UUID.randomUUID();
        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_3, caseId, null, null);
        ae.escalationStatus = AeEscalationStatus.REQUESTED;

        when(planItemStore.findByCaseId(caseId, "tenant-1")).thenReturn(List.of(
            planItem(caseId, "safety-review", TaskStatus.ACTIVE, ae.reportedAt.plusSeconds(600))
        ));

        var full = builder.buildTrajectory(ae, "tenant-1");
        var partial = builder.buildPartialTrajectory(ae, "tenant-1");
        assertEquals(full, partial);
    }

    private AdverseEvent buildAe(CtcaeGrade grade, UUID engineCaseId, UUID susarCaseId, UUID regCaseId) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.grade = grade;
        ae.reportedAt = Instant.parse("2026-01-15T10:00:00Z");
        ae.enrollmentId = UUID.randomUUID();
        ae.engineCaseId = engineCaseId;
        ae.susarOversightCaseId = susarCaseId;
        ae.regulatorySubmissionCaseId = regCaseId;
        ae.escalationStatus = AeEscalationStatus.NONE;
        ae.susarOversightStatus = SusarOversightStatus.NONE;
        ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.NONE;
        ae.tenantId = "tenant-1";
        return ae;
    }

    private PlanItemRecord planItem(UUID caseId, String binding, TaskStatus status, Instant createdAt) {
        return new PlanItemRecord(caseId, UUID.randomUUID().toString(), binding, status, createdAt,
            null, null, "tenant-1", null, null, null);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=AeTrajectoryBuilderTest --batch-mode`
Expected: FAIL — `AeTrajectoryBuilder` class not found

- [ ] **Step 3: Implement AeTrajectoryBuilder**

Use `ide_create_file`:

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Duration;
import java.util.*;

@ApplicationScoped
public class AeTrajectoryBuilder {

    private final PlanItemStore planItemStore;

    @Inject
    public AeTrajectoryBuilder(PlanItemStore planItemStore) {
        this.planItemStore = planItemStore;
    }

    public List<Map<String, FeatureValue>> buildTrajectory(AdverseEvent ae, String tenantId) {
        return doBuild(ae, tenantId);
    }

    public List<Map<String, FeatureValue>> buildPartialTrajectory(AdverseEvent ae, String tenantId) {
        return doBuild(ae, tenantId);
    }

    private List<Map<String, FeatureValue>> doBuild(AdverseEvent ae, String tenantId) {
        List<Observation> observations = new ArrayList<>();
        observations.add(new Observation(0, statusOrdinal(AeEscalationStatus.NONE),
                statusOrdinal(SusarOversightStatus.NONE), statusOrdinal(RegulatorySubmissionStatus.NONE)));

        List<PlanItemRecord> allRecords = new ArrayList<>();
        if (ae.engineCaseId != null) {
            allRecords.addAll(planItemStore.findByCaseId(ae.engineCaseId, tenantId));
        }
        if (ae.susarOversightCaseId != null) {
            allRecords.addAll(planItemStore.findByCaseId(ae.susarOversightCaseId, tenantId));
        }
        if (ae.regulatorySubmissionCaseId != null) {
            allRecords.addAll(planItemStore.findByCaseId(ae.regulatorySubmissionCaseId, tenantId));
        }

        allRecords.sort(Comparator.comparing(PlanItemRecord::createdAt));

        int escalation = ae.engineCaseId != null ? statusOrdinal(AeEscalationStatus.REQUESTED) : statusOrdinal(AeEscalationStatus.NONE);
        int susar = statusOrdinal(SusarOversightStatus.NONE);
        int regulatory = statusOrdinal(RegulatorySubmissionStatus.NONE);

        if (ae.engineCaseId != null) {
            observations.get(0).escalation = escalation;
        }

        for (PlanItemRecord record : allRecords) {
            long secondsSinceReport = Duration.between(ae.reportedAt, record.createdAt()).getSeconds();
            if (secondsSinceReport < 0) secondsSinceReport = 0;

            if (isSusarBinding(record.bindingName())) {
                susar = record.status().isTerminal()
                        ? statusOrdinal(SusarOversightStatus.COMPLETED) : statusOrdinal(SusarOversightStatus.REQUESTED);
            } else if (isRegulatoryBinding(record.bindingName())) {
                regulatory = record.status().isTerminal()
                        ? statusOrdinal(RegulatorySubmissionStatus.FILED) : statusOrdinal(RegulatorySubmissionStatus.REQUESTED);
            }
            if (record.status().isTerminal() && isEscalationBinding(record.bindingName())) {
                escalation = statusOrdinal(AeEscalationStatus.COMPLETED);
            }

            observations.add(new Observation(secondsSinceReport, escalation, susar, regulatory));
        }

        // Apply final entity state to last observation
        if (!observations.isEmpty()) {
            var last = observations.get(observations.size() - 1);
            last.escalation = statusOrdinal(ae.escalationStatus);
            last.susar = statusOrdinal(ae.susarOversightStatus);
            last.regulatory = statusOrdinal(ae.regulatorySubmissionStatus);
        }

        return observations.stream().map(Observation::toFeatureMap).toList();
    }

    private boolean isEscalationBinding(String name) {
        return name != null && (name.contains("safety-review") || name.contains("dsmb"));
    }

    private boolean isSusarBinding(String name) {
        return name != null && name.contains("susar");
    }

    private boolean isRegulatoryBinding(String name) {
        return name != null && (name.contains("regulatory") || name.contains("ind"));
    }

    private static int statusOrdinal(AeEscalationStatus status) {
        return switch (status) {
            case NONE -> 0; case REQUESTED -> 1; case COMPLETED -> 2; case FAILED -> 3;
        };
    }

    private static int statusOrdinal(SusarOversightStatus status) {
        return switch (status) {
            case NONE -> 0; case REQUESTED -> 1; case COMPLETED -> 2; case FAILED -> 3;
        };
    }

    private static int statusOrdinal(RegulatorySubmissionStatus status) {
        return switch (status) {
            case NONE -> 0; case REQUESTED -> 1; case FILED -> 2; case DEADLINE_MISSED -> 3;
        };
    }

    private static class Observation {
        long secondsSinceReport;
        int escalation;
        int susar;
        int regulatory;

        Observation(long secondsSinceReport, int escalation, int susar, int regulatory) {
            this.secondsSinceReport = secondsSinceReport;
            this.escalation = escalation;
            this.susar = susar;
            this.regulatory = regulatory;
        }

        Map<String, FeatureValue> toFeatureMap() {
            return Map.of(
                "ts", FeatureValue.number(secondsSinceReport),
                "escalation", FeatureValue.number(escalation),
                "susar", FeatureValue.number(susar),
                "regulatory", FeatureValue.number(regulatory));
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=AeTrajectoryBuilderTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Verify with ide_diagnostics, commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/cbr/AeTrajectoryBuilder.java runtime/src/test/java/io/casehub/clinical/cbr/AeTrajectoryBuilderTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#119): AeTrajectoryBuilder — lazy trajectory reconstruction from PlanItemStore"
```

---

### Task 3: SiteEnrollmentTrajectoryBuilder

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/SiteEnrollmentTrajectoryBuilder.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/SiteEnrollmentTrajectoryBuilderTest.java`

**Interfaces:**
- Consumes: `PatientEnrollment` Panache entity — `find("siteId = ?1 AND tenantId = ?2", siteId, tenantId)`
- Produces: `buildTrajectory(UUID siteId, UUID trialId, Instant trialActivatedAt, String tenantId)` → `List<Map<String, FeatureValue>>`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.FeatureValue;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class SiteEnrollmentTrajectoryBuilderTest {

    private SiteEnrollmentTrajectoryBuilder builder;
    private final UUID siteId = UUID.randomUUID();
    private final UUID trialId = UUID.randomUUID();
    private final Instant trialStart = Instant.parse("2026-01-01T00:00:00Z");

    @BeforeEach
    void setUp() {
        builder = new SiteEnrollmentTrajectoryBuilder();
    }

    @Test
    void emptyEnrollments_returnsEmptyTrajectory() {
        builder.setEnrollmentQuery((site, tenant) -> List.of());
        var trajectory = builder.buildTrajectory(siteId, trialId, trialStart, "tenant-1");
        assertTrue(trajectory.isEmpty());
    }

    @Test
    void threeWeeksOfEnrollments_threeObservations() {
        builder.setEnrollmentQuery((site, tenant) -> List.of(
            trialStart.plus(1, ChronoUnit.DAYS),
            trialStart.plus(2, ChronoUnit.DAYS),
            trialStart.plus(8, ChronoUnit.DAYS),
            trialStart.plus(15, ChronoUnit.DAYS),
            trialStart.plus(16, ChronoUnit.DAYS),
            trialStart.plus(17, ChronoUnit.DAYS)
        ));
        var trajectory = builder.buildTrajectory(siteId, trialId, trialStart, "tenant-1");

        assertEquals(3, trajectory.size());
        // Week 0: 2 enrollments
        assertEquals(0.0, ((FeatureValue.NumberVal) trajectory.get(0).get("ts")).value());
        assertEquals(2.0, ((FeatureValue.NumberVal) trajectory.get(0).get("periodCount")).value());
        assertEquals(2.0, ((FeatureValue.NumberVal) trajectory.get(0).get("cumulativeCount")).value());
        // Week 1: 1 enrollment
        assertEquals(1.0, ((FeatureValue.NumberVal) trajectory.get(1).get("ts")).value());
        assertEquals(1.0, ((FeatureValue.NumberVal) trajectory.get(1).get("periodCount")).value());
        assertEquals(3.0, ((FeatureValue.NumberVal) trajectory.get(1).get("cumulativeCount")).value());
        // Week 2: 3 enrollments
        assertEquals(2.0, ((FeatureValue.NumberVal) trajectory.get(2).get("ts")).value());
        assertEquals(3.0, ((FeatureValue.NumberVal) trajectory.get(2).get("periodCount")).value());
        assertEquals(6.0, ((FeatureValue.NumberVal) trajectory.get(2).get("cumulativeCount")).value());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=SiteEnrollmentTrajectoryBuilderTest --batch-mode`
Expected: FAIL — class not found

- [ ] **Step 3: Implement SiteEnrollmentTrajectoryBuilder**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import jakarta.enterprise.context.ApplicationScoped;

import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.function.BiFunction;

@ApplicationScoped
public class SiteEnrollmentTrajectoryBuilder {

    private BiFunction<UUID, String, List<Instant>> enrollmentQuery = SiteEnrollmentTrajectoryBuilder::defaultQuery;

    void setEnrollmentQuery(BiFunction<UUID, String, List<Instant>> query) {
        this.enrollmentQuery = query;
    }

    public List<Map<String, FeatureValue>> buildTrajectory(UUID siteId, UUID trialId,
                                                            Instant trialActivatedAt, String tenantId) {
        List<Instant> enrollmentDates = enrollmentQuery.apply(siteId, tenantId);
        if (enrollmentDates.isEmpty()) return List.of();

        Map<Long, Integer> weekCounts = new TreeMap<>();
        for (Instant date : enrollmentDates) {
            long week = Duration.between(trialActivatedAt, date).toDays() / 7;
            if (week < 0) week = 0;
            weekCounts.merge(week, 1, Integer::sum);
        }

        List<Map<String, FeatureValue>> observations = new ArrayList<>();
        int cumulative = 0;
        for (var entry : weekCounts.entrySet()) {
            cumulative += entry.getValue();
            observations.add(Map.of(
                "ts", FeatureValue.number(entry.getKey()),
                "periodCount", FeatureValue.number(entry.getValue()),
                "cumulativeCount", FeatureValue.number(cumulative)));
        }
        return observations;
    }

    private static List<Instant> defaultQuery(UUID siteId, String tenantId) {
        return PatientEnrollment.<PatientEnrollment>find("siteId = ?1 AND tenantId = ?2", siteId, tenantId)
                .stream()
                .filter(e -> e.enrolledAt != null)
                .map(e -> e.enrolledAt)
                .sorted()
                .toList();
    }
}
```

Note: `PatientEnrollment` may not have an `enrolledAt` field. Check the entity and use the appropriate timestamp field (creation time or status change time). If no explicit enrollment timestamp exists, the builder should fall back to entity creation time from Panache (`persistedAt` is not a Panache field — use `id` + creation order, or add `createdAt` if needed). During implementation, inspect the actual entity and adjust accordingly.

- [ ] **Step 4: Run tests, verify with ide_diagnostics, commit**

Run: `mvn test -pl runtime -Dtest=SiteEnrollmentTrajectoryBuilderTest --batch-mode`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/cbr/SiteEnrollmentTrajectoryBuilder.java runtime/src/test/java/io/casehub/clinical/cbr/SiteEnrollmentTrajectoryBuilderTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#119): SiteEnrollmentTrajectoryBuilder — enrollment rate by week"
```

---

### Task 4: AeTrajectoryAlertService

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/AeTrajectoryAlertService.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/AeTrajectoryAlertServiceTest.java`

**Interfaces:**
- Consumes: `AeTrajectoryBuilder.buildPartialTrajectory(ae, tenantId)`
- Consumes: `ClinicalCbrService.retrieveWithAudit(query, PlanCbrCase.class, subjectId, actorId)`
- Consumes: `ClinicalActors.CLINICAL_SERVICE`
- Produces: `evaluate(UUID aeId, String tenantId)` → `Optional<AeTrajectoryAlertEvent>`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.AeTrajectoryAlertEvent;
import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.neocortex.memory.cbr.*;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class AeTrajectoryAlertServiceTest {

    private AeTrajectoryBuilder trajectoryBuilder;
    private ClinicalCbrService cbrService;
    private Event<AeTrajectoryAlertEvent> alertEvents;
    private AeTrajectoryAlertService service;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        trajectoryBuilder = mock(AeTrajectoryBuilder.class);
        cbrService = mock(ClinicalCbrService.class);
        alertEvents = mock(Event.class);
        service = new AeTrajectoryAlertService(trajectoryBuilder, cbrService, alertEvents);
        service.setEntityFinder(AeTrajectoryAlertServiceTest::findAe);
    }

    @Test
    void noMatches_returnsEmpty() {
        when(trajectoryBuilder.buildPartialTrajectory(any(), eq("t1"))).thenReturn(List.of(Map.of()));
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
                .thenReturn(new AuditedRetrievalResult<>(List.of(), "trace-1", null));

        Optional<AeTrajectoryAlertEvent> result = service.evaluate(TEST_AE_ID, "t1");
        assertTrue(result.isEmpty());
        verify(alertEvents, never()).fireAsync(any());
    }

    @Test
    void matchesBelowProbabilityThreshold_returnsEmpty() {
        when(trajectoryBuilder.buildPartialTrajectory(any(), eq("t1"))).thenReturn(List.of(Map.of()));
        var match1 = scoredCase("COMPLETED", 0.7);
        var match2 = scoredCase("FAULTED", 0.65);
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
                .thenReturn(new AuditedRetrievalResult<>(List.of(match1, match2), "trace-1", null));

        // COMPLETED has score sum 0.7, FAULTED has 0.65. Probability = 0.7/1.35 ≈ 0.52 < 0.6 threshold
        Optional<AeTrajectoryAlertEvent> result = service.evaluate(TEST_AE_ID, "t1");
        assertTrue(result.isEmpty());
    }

    @Test
    void matchesAboveThreshold_firesEvent() {
        when(trajectoryBuilder.buildPartialTrajectory(any(), eq("t1"))).thenReturn(List.of(Map.of()));
        var match1 = scoredCase("FAULTED", 0.8);
        var match2 = scoredCase("FAULTED", 0.7);
        var match3 = scoredCase("COMPLETED", 0.5);
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
                .thenReturn(new AuditedRetrievalResult<>(List.of(match1, match2, match3), "trace-1", null));

        // FAULTED: 0.8+0.7=1.5, COMPLETED: 0.5. Probability = 1.5/2.0 = 0.75 > 0.6
        Optional<AeTrajectoryAlertEvent> result = service.evaluate(TEST_AE_ID, "t1");
        assertTrue(result.isPresent());
        assertEquals("FAULTED", result.get().predictedOutcome());
        assertEquals(3, result.get().matchCount());
        verify(alertEvents).fireAsync(any());
    }

    @Test
    void cbrRetrievalFailure_returnsEmptyAndLogs() {
        when(trajectoryBuilder.buildPartialTrajectory(any(), eq("t1"))).thenReturn(List.of(Map.of()));
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
                .thenThrow(new RuntimeException("CBR store unavailable"));

        Optional<AeTrajectoryAlertEvent> result = service.evaluate(TEST_AE_ID, "t1");
        assertTrue(result.isEmpty());
        verify(alertEvents, never()).fireAsync(any());
    }

    private static final UUID TEST_AE_ID = UUID.randomUUID();

    private static AdverseEvent findAe(UUID id) {
        if (!TEST_AE_ID.equals(id)) return null;
        AdverseEvent ae = new AdverseEvent();
        ae.id = id;
        ae.grade = CtcaeGrade.GRADE_3;
        ae.enrollmentId = UUID.randomUUID();
        ae.engineCaseId = UUID.randomUUID();
        ae.eventType = "NAUSEA";
        ae.escalationStatus = AeEscalationStatus.REQUESTED;
        ae.susarOversightStatus = SusarOversightStatus.NONE;
        ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.NONE;
        ae.tenantId = "t1";
        ae.reportedAt = java.time.Instant.now();
        return ae;
    }

    private ScoredCbrCase<PlanCbrCase> scoredCase(String outcome, double score) {
        var cbrCase = new PlanCbrCase("problem", "solution", outcome, 1.0, Map.of(), List.of());
        return new ScoredCbrCase<>(cbrCase, UUID.randomUUID().toString(), score);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=AeTrajectoryAlertServiceTest --batch-mode`
Expected: FAIL — class not found

- [ ] **Step 3: Implement AeTrajectoryAlertService**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.AeTrajectoryAlertEvent;
import io.casehub.clinical.api.ClinicalActors;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

@ApplicationScoped
public class AeTrajectoryAlertService {

    private static final Logger LOG = Logger.getLogger(AeTrajectoryAlertService.class);

    private final AeTrajectoryBuilder trajectoryBuilder;
    private final ClinicalCbrService cbrService;
    private final Event<AeTrajectoryAlertEvent> alertEvents;
    private Function<UUID, AdverseEvent> entityFinder = id -> AdverseEvent.findById(id);

    @ConfigProperty(name = "casehub.clinical.trajectory.alert.min-matches", defaultValue = "2")
    int minMatches;

    @ConfigProperty(name = "casehub.clinical.trajectory.alert.min-similarity", defaultValue = "0.5")
    double minSimilarity;

    @ConfigProperty(name = "casehub.clinical.trajectory.alert.min-probability", defaultValue = "0.6")
    double minProbability;

    @Inject
    public AeTrajectoryAlertService(AeTrajectoryBuilder trajectoryBuilder,
                                     ClinicalCbrService cbrService,
                                     Event<AeTrajectoryAlertEvent> alertEvents) {
        this.trajectoryBuilder = trajectoryBuilder;
        this.cbrService = cbrService;
        this.alertEvents = alertEvents;
    }

    void setEntityFinder(Function<UUID, AdverseEvent> finder) {
        this.entityFinder = finder;
    }

    public Optional<AeTrajectoryAlertEvent> evaluate(UUID aeId, String tenantId) {
        try {
            AdverseEvent ae = entityFinder.apply(aeId);
            if (ae == null) {
                LOG.debugf("AeTrajectoryAlertService: AE not found for aeId=%s", aeId);
                return Optional.empty();
            }
            var trajectory = trajectoryBuilder.buildPartialTrajectory(ae, tenantId);
            Map<String, FeatureValue> features = new LinkedHashMap<>();
            features.put("grade", FeatureValue.number(ae.grade != null ? ae.grade.ordinal() + 1 : 0));
            features.put("trialPhase", FeatureValue.string("UNKNOWN"));
            features.put("unexpected", FeatureValue.string(String.valueOf(ae.unexpected)));
            features.put("suspected", FeatureValue.string(String.valueOf(ae.suspected)));
            features.put("aeTrajectory", FeatureValue.structList(trajectory));

            CbrQuery query = CbrQuery.of(tenantId, ClinicalCbrDomains.AE_TRAJECTORY, Path.root(),
                            "clinical-ae-trajectory", features, 10)
                    .withMinSimilarity(minSimilarity)
                    .withFilter("eventType", CbrFilter.contains(ae.eventType != null ? ae.eventType : "UNKNOWN"));

            AuditedRetrievalResult<PlanCbrCase> result = cbrService.retrieveWithAudit(
                    query, PlanCbrCase.class, ae.enrollmentId, ClinicalActors.CLINICAL_SERVICE);

            if (result.cases().size() < minMatches) return Optional.empty();

            var prediction = predictOutcome(result.cases());
            if (prediction.probability < minProbability) return Optional.empty();

            UUID siteId = null; // resolve from enrollment if needed
            var event = new AeTrajectoryAlertEvent(
                    aeId, ae.enrollmentId, siteId, ae.grade,
                    result.cases().size(), result.cases().get(0).score(),
                    prediction.outcome, prediction.probability,
                    result.traceId(), tenantId);
            alertEvents.fireAsync(event);
            return Optional.of(event);
        } catch (Exception e) {
            LOG.warnf(e, "AeTrajectoryAlertService: evaluation failed for aeId=%s — continuing without alert", aeId);
            return Optional.empty();
        }
    }

    record Prediction(String outcome, double probability) {}

    Prediction predictOutcome(List<ScoredCbrCase<PlanCbrCase>> cases) {
        Map<String, Double> scoresByOutcome = cases.stream()
                .filter(c -> c.cbrCase().outcome() != null)
                .collect(Collectors.groupingBy(
                        c -> c.cbrCase().outcome(),
                        Collectors.summingDouble(ScoredCbrCase::score)));

        double totalScore = scoresByOutcome.values().stream().mapToDouble(Double::doubleValue).sum();
        if (totalScore == 0) return new Prediction("UNKNOWN", 0);

        var winner = scoresByOutcome.entrySet().stream()
                .max(Map.Entry.comparingByValue())
                .orElseThrow();
        return new Prediction(winner.getKey(), winner.getValue() / totalScore);
    }
}
```

- [ ] **Step 4: Run tests, verify, commit**

Run: `mvn test -pl runtime -Dtest=AeTrajectoryAlertServiceTest --batch-mode`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/cbr/AeTrajectoryAlertService.java runtime/src/test/java/io/casehub/clinical/cbr/AeTrajectoryAlertServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#119): AeTrajectoryAlertService — weighted majority voting, configurable thresholds"
```

---

### Task 5: SiteEnrollmentAlertService

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/SiteEnrollmentAlertService.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/SiteEnrollmentAlertServiceTest.java`

**Interfaces:**
- Consumes: `SiteEnrollmentTrajectoryBuilder.buildTrajectory(siteId, trialId, trialActivatedAt, tenantId)`
- Consumes: `ClinicalCbrService.retrieveWithAudit(...)`
- Produces: `evaluate(UUID siteId, UUID trialId, String tenantId)` → `Optional<SiteEnrollmentAlertEvent>`

Same pattern as Task 4. Key differences: looks up `ClinicalTrial` to get `trialActivatedAt`, uses `ClinicalCbrDomains.SITE_ENROLLMENT` domain, `siteId` as audit subject.

- [ ] **Step 1: Write failing tests** — same structure as AeTrajectoryAlertServiceTest
- [ ] **Step 2: Run tests to verify they fail**
- [ ] **Step 3: Implement SiteEnrollmentAlertService** — follows AeTrajectoryAlertService pattern exactly
- [ ] **Step 4: Run tests, verify, commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#119): SiteEnrollmentAlertService — enrollment deceleration detection"
```

---

### Task 6: Lifecycle Hooks + Retention

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java` — add hook after `ae.persist()`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java` — add hook after `persistCaseId()`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java` — add hook after `markCompleted()`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightCaseService.java` — add hook after `persistCaseId()`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SusarGateDecisionListener.java` — add hook in all 3 methods
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java` — add hook after enrollment persist
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCaseOutcomeObserver.java` — add trajectory retention
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/TrialCompletionSiteTrajectoryWriter.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/TrajectoryLifecycleHookTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/TrialCompletionSiteTrajectoryWriterTest.java`

**Interfaces:**
- Consumes: `AeTrajectoryAlertService.evaluate(aeId, tenantId)`
- Consumes: `SiteEnrollmentAlertService.evaluate(siteId, trialId, tenantId)`
- Consumes: `AeTrajectoryBuilder.buildTrajectory(ae, tenantId)`
- Consumes: `TrialStatusChangedEvent`

- [ ] **Step 1: Write lifecycle hook test**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.service.*;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {"trial-sponsor", "principal-investigator", "trial-coordinator"})
class TrajectoryLifecycleHookTest {

    @InjectMock AeTrajectoryAlertService aeAlertService;
    @InjectMock SiteEnrollmentAlertService siteAlertService;
    @Inject AdverseEventService adverseEventService;

    @BeforeEach
    void setUp() {
        when(aeAlertService.evaluate(any(), any())).thenReturn(Optional.empty());
        when(siteAlertService.evaluate(any(), any(), any())).thenReturn(Optional.empty());
    }

    @Test
    void reportAdverseEvent_callsTrajectoryAlertService() {
        // Setup: create trial, site, enrollment in @BeforeEach
        // Call adverseEventService.reportAdverseEvent(ae)
        // Verify: aeAlertService.evaluate(ae.id, tenantId) was called
        verify(aeAlertService, atLeastOnce()).evaluate(any(), any());
    }
}
```

- [ ] **Step 2: Add hooks to existing services**

Each hook is a single line added in a try-catch:

**AdverseEventService** — after `ae.persist()` and `ledgerWriter.writeReportEntry(ae)`, before the CDI event block:
```java
try { aeTrajectoryAlertService.evaluate(ae.id, ae.tenantId); } catch (Exception e) { LOG.warnf(e, "Trajectory alert evaluation failed for aeId=%s", ae.id); }
```

**AeEscalationCaseService** — after `persistCaseId(event.aeId(), caseId)`:
```java
try { aeTrajectoryAlertService.evaluate(event.aeId(), event.tenantId()); } catch (Exception e) { LOG.warnf(e, "Trajectory alert re-evaluation failed for aeId=%s", event.aeId()); }
```

**AeEscalationListener** — after `boolean firstCompletion = statusUpdater.markCompleted(aeId)`, inside the `if (firstCompletion)` block:
```java
try { aeTrajectoryAlertService.evaluate(aeId, resolveString(instance.getCaseContext().getPath("tenantId"))); } catch (Exception e) { LOG.warnf(e, "Trajectory alert evaluation failed for aeId=%s", aeId); }
```

**SusarOversightCaseService** — after `persistCaseId(event.aeId(), caseId)`:
```java
try { aeTrajectoryAlertService.evaluate(event.aeId(), event.tenantId()); } catch (Exception e) { LOG.warnf(e, "Trajectory alert re-evaluation failed for aeId=%s", event.aeId()); }
```

**SusarGateDecisionListener** — in each of `onApproved`, `onRejected`, `onExpired`, after the `ledgerWriter.writeEntry(...)` call:
```java
try { aeTrajectoryAlertService.evaluate(ae.id, ae.tenantId); } catch (Exception e) { LOG.warnf(e, "Trajectory alert evaluation failed for aeId=%s", ae.id); }
```

**PatientResource.enroll()** — after `enrollment.persist()`:
```java
try { siteEnrollmentAlertService.evaluate(siteId, site.trialId, enrollment.tenantId); } catch (Exception e) { /* silent — enrollment is advisory */ }
```

Each service needs `@Inject AeTrajectoryAlertService aeTrajectoryAlertService` or `@Inject SiteEnrollmentAlertService siteEnrollmentAlertService` added as a field.

- [ ] **Step 3: Extend ClinicalCaseOutcomeObserver for trajectory retention**

In `handleAeCase()`, after the existing `cbrService.storeIdempotent(...)` call, add:

```java
// Trajectory retention — store full trajectory as CBR case in ae-trajectory domain
List<Map<String, FeatureValue>> trajectory = trajectoryBuilder.buildTrajectory(ae, ae.tenantId);
if (!trajectory.isEmpty()) {
    Map<String, Object> trajFeatures = new LinkedHashMap<>(features);
    trajFeatures.put("aeTrajectory", trajectory);
    var trajCbrCase = new PlanCbrCase(problem, solution, event.outcomeLabel(), 1.0,
        FeatureValue.toFeatureMap(trajFeatures), planTraces);
    cbrService.storeIdempotent(trajCbrCase, "clinical-ae-trajectory", aeId.toString(),
        ClinicalCbrDomains.AE_TRAJECTORY, ae.tenantId,
        event.caseId() != null ? event.caseId().toString() : null);
}
```

Add `@Inject AeTrajectoryBuilder trajectoryBuilder` field.

- [ ] **Step 4: Create TrialCompletionSiteTrajectoryWriter**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.TrialStatusChangedEvent;
import io.casehub.clinical.api.model.TrialStatus;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;

@ApplicationScoped
public class TrialCompletionSiteTrajectoryWriter {

    private static final Logger LOG = Logger.getLogger(TrialCompletionSiteTrajectoryWriter.class);

    @Inject ClinicalCbrService cbrService;
    @Inject SiteEnrollmentTrajectoryBuilder trajectoryBuilder;

    @Transactional
    public void onTrialStatusChanged(@ObservesAsync TrialStatusChangedEvent event) {
        if (event.newStatus() != TrialStatus.COMPLETED && event.newStatus() != TrialStatus.TERMINATED) return;

        ClinicalTrial trial = ClinicalTrial.findById(event.trialId());
        if (trial == null) return;

        List<TrialSite> sites = TrialSite.find("trialId", event.trialId()).list();
        for (TrialSite site : sites) {
            try {
                storeForSite(site, trial, event.tenantId());
            } catch (Exception e) {
                LOG.warnf(e, "Failed to store enrollment trajectory for siteId=%s", site.id);
            }
        }
    }

    private void storeForSite(TrialSite site, ClinicalTrial trial, String tenantId) {
        var trajectory = trajectoryBuilder.buildTrajectory(site.id, trial.id, trial.activatedAt, tenantId);
        if (trajectory.isEmpty()) return;

        Map<String, Object> features = Map.of(
                "trialPhase", trial.phase != null ? trial.phase.name() : "UNKNOWN",
                "enrollmentRate", trajectory);

        var cbrCase = new PlanCbrCase(
                "Site " + site.siteName + " in " + (trial.phase != null ? trial.phase.name() : "UNKNOWN") + " trial",
                "Enrollment: " + trajectory.size() + " weeks tracked",
                "COMPLETED", 1.0,
                FeatureValue.toFeatureMap(features), List.of());

        cbrService.storeIdempotent(cbrCase, "clinical-site-enrollment", site.id.toString(),
                ClinicalCbrDomains.SITE_ENROLLMENT, tenantId, null);
    }
}
```

- [ ] **Step 5: Run full test suite, commit**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: PASS — all existing tests + new tests green

```bash
git -C /Users/mdproctor/claude/casehub/clinical add -A
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#119): trajectory lifecycle hooks + retention — wire alert evaluation into existing services"
```

---

### Task 7: REST API — Trajectory Endpoints

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialDashboardResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/TrajectoryEndpointTest.java`

**Interfaces:**
- Consumes: `AeTrajectoryBuilder.buildPartialTrajectory(ae, tenantId)`
- Consumes: `AeTrajectoryAlertService.evaluate(aeId, tenantId)`
- Consumes: `SiteEnrollmentTrajectoryBuilder.buildTrajectory(siteId, trialId, trialActivatedAt, tenantId)`
- Consumes: `TrendAnalyzer.analyze(observations, schema)`

- [ ] **Step 1: Define response records in TrialDashboardResource**

Add inner records:

```java
record TrajectoryObservation(long secondsSinceReport, String escalationStatus, String susarStatus, String regulatoryStatus) {}
record DimensionTrend(double slope, double acceleration, int changePoints) {}
record TrendSummary(Map<String, DimensionTrend> dimensions) {}
record AeTrajectoryResponse(UUID aeId, List<TrajectoryObservation> observations, TrendSummary trends) {}
record TrajectoryMatch(String caseId, double score, String outcome, List<TrajectoryObservation> trajectory) {}
record AeTrajectoryMatchResponse(List<TrajectoryMatch> matches, String traceId, String explanation) {}
record EnrollmentObservation(int weekNumber, int periodCount, int cumulativeCount) {}
record EnrollmentMatch(String caseId, double score, String outcome, List<EnrollmentObservation> trajectory) {}
record SiteEnrollmentTrajectoryResponse(List<EnrollmentObservation> observations, TrendSummary trends, List<EnrollmentMatch> matches) {}
```

- [ ] **Step 2: Add GET /adverse-events/{aeId}/trajectory endpoint**

```java
@GET
@Path("/adverse-events/{aeId}/trajectory")
@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR, ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
public Response aeTrajectory(@PathParam("trialId") UUID trialId, @PathParam("aeId") UUID aeId) {
    AdverseEvent ae = AdverseEvent.findByIdForTenant(aeId, principal);
    if (ae == null) return Response.status(Response.Status.NOT_FOUND).build();

    var observations = aeTrajectoryBuilder.buildPartialTrajectory(ae, principal.tenancyId());
    var obsResponses = observations.stream().map(this::toTrajectoryObservation).toList();

    var schema = ClinicalCbrSchemaInitializer.aeTrajectorySchema();
    var tsField = schema.fields().stream()
            .filter(f -> f instanceof FeatureField.TimeSeries).map(f -> (FeatureField.TimeSeries) f)
            .findFirst().orElse(null);
    TrendSummary trends = tsField != null ? computeTrends(observations, tsField) : new TrendSummary(Map.of());

    return Response.ok(new AeTrajectoryResponse(aeId, obsResponses, trends)).build();
}
```

- [ ] **Step 3: Add GET /adverse-events/{aeId}/trajectory/matches endpoint**

Similar to existing precedent endpoints — calls `ClinicalCbrService.retrieveWithAudit()` with trajectory query.

- [ ] **Step 4: Add GET /sites/{siteId}/enrollment-trajectory endpoint**

Calls `SiteEnrollmentTrajectoryBuilder`, returns observations + trend summary + CBR matches.

- [ ] **Step 5: Write endpoint tests, run full suite, commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#119): REST API — trajectory + matches + enrollment-trajectory endpoints"
```

---

## Verification

After all tasks complete:

- [ ] Run full test suite: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
- [ ] Run `ide_diagnostics` on all new/modified files
- [ ] Verify test count increased (baseline: 614 tests)
- [ ] Check `ide_build_project` compiles cleanly
