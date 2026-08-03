# CBR Feature Enrichment + Case Compaction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #132 — expand AeCbrFeatureBuilder to include site profile and trust scores
**Issue group:** #132, #144

**Goal:** Enrich AE CBR cases with site enrollment profile and agent trust scores (#132), then add a batch compaction job that merges similar CBR cases into weighted representatives (#144).

**Architecture:** #132 adds `AeCbrContext` record and 3 new features to the static `AeCbrFeatureBuilder`. Callers construct the record with site enrollment count, target enrollment, and agent trust score looked up from `ActorTrustScore`. #144 adds `CbrCompactionJob` that scans the `clinical-ae` domain, groups cases by 5 categorical merge-key features, and replaces groups of 3+ with a single weighted representative.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache Active Record, casehub-neocortex CBR API, casehub-ledger `ActorTrustScore`

## Global Constraints

- `AeCbrFeatureBuilder` stays a static utility — no CDI dependencies
- `ActorTrustScore` is in the qhorus PU — XA already configured on both datasources
- CBR store is schemaless — new features are additive, no Flyway migrations
- `eventType` is `categoricalList` — always `List.of(value)`, never plain `String`
- Compaction runs on `clinical-ae` domain only
- Scheduler jobs excluded in tests via `quarkus.arc.exclude-types`

---

### Task 1: AeCbrContext record + builder refactor

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/AeCbrContext.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/AeCbrFeatureBuilder.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/cbr/AeCbrFeatureBuilderTest.java`

**Interfaces:**
- Consumes: `AdverseEvent`, `PatientEnrollment`, `ClinicalTrial` entities
- Produces: `AeCbrContext` record, `AeCbrFeatureBuilder.buildFeatures(AeCbrContext)`, `buildQueryFeatures(AeCbrContext)`, `buildProblemSummary(AeCbrContext)`, `buildSolutionSummary(AeCbrContext)` — all return same types as before

- [ ] **Step 1: Write failing tests for AeCbrContext + new features**

Update `AeCbrFeatureBuilderTest`. Add test for 14-feature output via `AeCbrContext`:

```java
@Test
void buildFeatures_withContext_returns14Features() {
    AdverseEvent ae = new AdverseEvent();
    ae.grade = CtcaeGrade.GRADE_3;
    ae.eventType = "Neutropenia";
    ae.suspected = true;
    ae.unexpected = true;
    ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.FILED;
    ae.susarOversightStatus = SusarOversightStatus.COMPLETED;

    PatientEnrollment enrollment = new PatientEnrollment();
    enrollment.treatmentArm = "ARM_A";

    ClinicalTrial trial = new ClinicalTrial();
    trial.phase = TrialPhase.PHASE_III;

    var ctx = new AeCbrContext(ae, enrollment, trial, "CONTINUE_MONITORING",
        true, 2, 45, 100, 0.82);

    Map<String, Object> features = AeCbrFeatureBuilder.buildFeatures(ctx);

    assertThat(features).hasSize(14);
    assertThat(features.get("siteEnrollmentCount")).isEqualTo(45L);
    assertThat(features.get("siteTargetEnrollment")).isEqualTo(100);
    assertThat(features.get("agentTrustScore")).isEqualTo(0.82);
}

@Test
void buildQueryFeatures_excludesAgentTrustScore() {
    var ctx = new AeCbrContext(minimalAe(), null, null, null, false, 0, 10, 50, 0.75);
    Map<String, Object> features = AeCbrFeatureBuilder.buildQueryFeatures(ctx);

    assertThat(features).containsKey("siteEnrollmentCount");
    assertThat(features).containsKey("siteTargetEnrollment");
    assertThat(features).doesNotContainKey("agentTrustScore");
}

@Test
void buildFeatures_trustScoreAtBounds() {
    var ctx0 = new AeCbrContext(minimalAe(), null, null, null, false, 0, 0, 0, 0.0);
    assertThat(AeCbrFeatureBuilder.buildFeatures(ctx0).get("agentTrustScore")).isEqualTo(0.0);

    var ctx1 = new AeCbrContext(minimalAe(), null, null, null, false, 0, 0, 0, 1.0);
    assertThat(AeCbrFeatureBuilder.buildFeatures(ctx1).get("agentTrustScore")).isEqualTo(1.0);
}

private AdverseEvent minimalAe() {
    AdverseEvent ae = new AdverseEvent();
    ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.NONE;
    ae.susarOversightStatus = SusarOversightStatus.NONE;
    return ae;
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=AeCbrFeatureBuilderTest --batch-mode`
Expected: FAIL — `AeCbrContext` does not exist

- [ ] **Step 3: Create AeCbrContext record**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/clinical/cbr/AeCbrContext.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.PatientEnrollment;

public record AeCbrContext(AdverseEvent ae,
                            PatientEnrollment enrollment,
                            ClinicalTrial trial,
                            String safetyReviewOutcome,
                            boolean dsmbEscalated,
                            long priorAeCount,
                            long siteEnrollmentCount,
                            int siteTargetEnrollment,
                            double agentTrustScore) {}
```

- [ ] **Step 4: Refactor AeCbrFeatureBuilder to accept AeCbrContext**

Use `ide_edit_member` on `AeCbrFeatureBuilder` to replace each method:

`buildFeatures(AeCbrContext ctx)` — adds 3 new features at the end:
```java
features.put("siteEnrollmentCount", ctx.siteEnrollmentCount());
features.put("siteTargetEnrollment", ctx.siteTargetEnrollment());
features.put("agentTrustScore", ctx.agentTrustScore());
```

`buildQueryFeatures(AeCbrContext ctx)` — adds site features only:
```java
features.put("siteEnrollmentCount", ctx.siteEnrollmentCount());
features.put("siteTargetEnrollment", ctx.siteTargetEnrollment());
```

`buildProblemSummary(AeCbrContext ctx)` and `buildSolutionSummary(AeCbrContext ctx)` — extract fields from `ctx` instead of direct params.

Keep the old 6-param `buildFeatures()` as a delegate to the new method (creates `AeCbrContext` with 0/0/0.5 defaults for the 3 new fields) — avoids breaking the existing caller in `DemoDataSeeder` until Task 2 updates it.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=AeCbrFeatureBuilderTest --batch-mode`
Expected: PASS

- [ ] **Step 6: Update schema**

Use `ide_edit_member` on `ClinicalCbrSchemaInitializer.aeSchema()` to add 4 fields:
```java
FeatureField.numeric("siteEnrollmentCount", 0, 10000),
FeatureField.numeric("siteTargetEnrollment", 0, 10000),
FeatureField.numeric("agentTrustScore", 0, 1),
FeatureField.numeric("mergeCount", 1, 100000)
```

Update `ClinicalCbrSchemaInitializerTest` to assert 15 fields.

- [ ] **Step 7: Run schema test**

Run: `mvn test -pl runtime -Dtest=ClinicalCbrSchemaInitializerTest --batch-mode`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/cbr/AeCbrContext.java runtime/src/main/java/io/casehub/clinical/cbr/AeCbrFeatureBuilder.java runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCbrSchemaInitializer.java runtime/src/test/java/io/casehub/clinical/cbr/AeCbrFeatureBuilderTest.java runtime/src/test/java/io/casehub/clinical/cbr/ClinicalCbrSchemaInitializerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#132): AeCbrContext record + 3 new features in AeCbrFeatureBuilder

Refs #132"
```

---

### Task 2: Wire trust score + site enrollment into ClinicalCaseOutcomeObserver

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCaseOutcomeObserver.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/cbr/ClinicalCaseOutcomeObserverTest.java`

**Interfaces:**
- Consumes: `AeCbrContext` (from Task 1), `ActorTrustScore` (casehub-ledger JPA entity), `PlanTrace`
- Produces: updated `EntityResolver` with `countEnrollmentsAtSite(UUID)` and `findAgentTrustScore(String)` methods

- [ ] **Step 1: Write failing tests for new EntityResolver methods**

In `ClinicalCaseOutcomeObserverTest`, update the `EntityResolver` mock to include:

```java
@Test
void handleAeCase_storesCbrWithSiteEnrollmentAndTrust() {
    // setup: ae, enrollment, site, trial
    when(resolver.countEnrollmentsAtSite(site.id)).thenReturn(45L);
    when(resolver.findAgentTrustScore("agent-1")).thenReturn(0.82);
    // trigger onOutcome
    // verify: features map passed to storeIdempotent contains
    //   siteEnrollmentCount=45, siteTargetEnrollment=100, agentTrustScore=0.82
}

@Test
void handleAeCase_noPlanTrace_trustDefaultsToHalf() {
    // setup: no safety-monitoring PlanTrace in planItemStore
    when(resolver.countEnrollmentsAtSite(any())).thenReturn(0L);
    // trigger onOutcome
    // verify: agentTrustScore=0.5 in features
}

@Test
void handleAeCase_noTrustScore_defaultsToHalf() {
    when(resolver.findAgentTrustScore(anyString())).thenReturn(0.5);
    // verify: agentTrustScore=0.5
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=ClinicalCaseOutcomeObserverTest --batch-mode`
Expected: FAIL — `countEnrollmentsAtSite` and `findAgentTrustScore` don't exist on `EntityResolver`

- [ ] **Step 3: Extend EntityResolver interface**

Use `ide_edit_member` to add to the `EntityResolver` interface inside `ClinicalCaseOutcomeObserver`:

```java
long countEnrollmentsAtSite(UUID siteId);
double findAgentTrustScore(String actorId);
```

Update `PanacheEntityResolver`:

```java
@Override
public long countEnrollmentsAtSite(UUID siteId) {
    return PatientEnrollment.count("siteId", siteId);
}

@Override
public double findAgentTrustScore(String actorId) {
    try {
        var em = Arc.container().instance(EntityManager.class,
            new io.quarkus.hibernate.orm.PersistenceUnit.PersistenceUnitLiteral("qhorus")).get();
        var results = em.createNamedQuery("ActorTrustScore.findCapabilityDimensionByKeys",
                io.casehub.ledger.runtime.model.ActorTrustScore.class)
            .setParameter("actorId", actorId)
            .setParameter("scoreType", io.casehub.ledger.runtime.model.ScoreType.BAYESIAN_BETA)
            .setParameter("capabilityKey", "safety-monitoring")
            .setParameter("dimensionKey", ClinicalTrustDimensions.SAFETY_ACCURACY)
            .getResultList();
        return results.isEmpty() ? 0.5 : results.get(0).trustScore;
    } catch (Exception e) {
        return 0.5;
    }
}
```

- [ ] **Step 4: Update handleAeCase to build AeCbrContext**

Use `ide_replace_member` on `handleAeCase` to:
1. After resolving site, call `entityResolver.countEnrollmentsAtSite(site.id)`
2. After building planTraces, extract agent actorId from the first `PlanTrace` with capability `safety-monitoring`
3. Call `entityResolver.findAgentTrustScore(actorId)` (or 0.5 if no agent found)
4. Construct `AeCbrContext` and call `AeCbrFeatureBuilder.buildFeatures(ctx)`

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=ClinicalCaseOutcomeObserverTest --batch-mode`
Expected: PASS

- [ ] **Step 6: Run full test suite to check for regressions**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: PASS (all existing tests still green)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/cbr/ClinicalCaseOutcomeObserver.java runtime/src/test/java/io/casehub/clinical/cbr/ClinicalCaseOutcomeObserverTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#132): wire site enrollment count + agent trust score into CBR observer

EntityResolver gains countEnrollmentsAtSite() and findAgentTrustScore().
handleAeCase() builds AeCbrContext with all 14 features.

Refs #132"
```

---

### Task 3: CbrCompactionJob + MergeKey

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/CbrCompactionJob.java`
- Create: `runtime/src/test/java/io/casehub/clinical/cbr/CbrCompactionJobTest.java`
- Modify: `runtime/src/main/resources/application.properties` (test — add exclude-types)

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.scan()`, `CbrCaseMemoryStore.store()`, `CbrCaseMemoryStore.eraseEntity()`, `CbrCaseMemoryStore.discoverTenants()`, `ClinicalCbrDomains.AE`, `PlanCbrCase`, `FeatureValue`
- Produces: `CbrCompactionJob.compact()` — public method for test-driven invocation

- [ ] **Step 1: Write failing tests**

Create `CbrCompactionJobTest` with mocked `CbrCaseMemoryStore`:

```java
@Test
void compact_threeMatchingCases_mergedIntoOne() {
    // Setup: 3 PlanCbrCases with same merge key (grade=3, eventType=[Neutropenia],
    //   trialPhase=PHASE_III, unexpected=true, suspected=true)
    // but different numeric features (priorAeCount, siteEnrollmentCount, etc.)
    // Mock scan to return 3 CbrCaseSummary entries
    // Mock retrieveSimilar to return full cases
    // Run compact()
    // Verify: store() called once with merged representative
    // Verify: eraseEntity() called 3 times for originals
    // Verify: merged features are weighted averages
    // Verify: mergeCount = 3
}

@Test
void compact_belowThreshold_noCompaction() {
    // Setup: 2 cases with same merge key (below default threshold of 3)
    // Run compact()
    // Verify: store() never called, eraseEntity() never called
}

@Test
void compact_differentMergeKeys_handledIndependently() {
    // Setup: 3 cases key A + 3 cases key B
    // Run compact()
    // Verify: store() called twice (one per group)
}

@Test
void compact_recompaction_weightsbyMergeCount() {
    // Setup: 1 compact representative (mergeCount=5, agentTrustScore=0.8)
    //   + 2 new singles (mergeCount=1 implicit, agentTrustScore=0.6 each)
    // Run compact()
    // Verify: new representative has mergeCount=7
    // Verify: agentTrustScore = (5*0.8 + 1*0.6 + 1*0.6) / 7 = 0.7429...
}

@Test
void compact_entityId_isDeterministic() {
    // Run compact() twice with same input
    // Verify: same entityId produced both times
}

@Test
void compact_categoricalOutcome_majorityVote() {
    // Setup: 3 cases — 2 with dsmbEscalated=true, 1 with false
    // Verify: merged representative has dsmbEscalated=true
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=CbrCompactionJobTest --batch-mode`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement CbrCompactionJob**

Use `ide_create_file` for `runtime/src/main/java/io/casehub/clinical/cbr/CbrCompactionJob.java`:

Key elements:
- `@ApplicationScoped`, `@Scheduled(every = "${casehub.clinical.cbr.compaction.interval:168h}", identity = "cbr-compaction")` on `compactAll()`
- `@ConfigProperty(name = "casehub.clinical.cbr.compaction.enabled", defaultValue = "false")` gates execution
- `@ConfigProperty(name = "casehub.clinical.cbr.compaction.min-group-size", defaultValue = "3")`
- Inner `MergeKey` record: `(int grade, List<String> eventType, String trialPhase, String unexpected, String suspected)` — implements `equals/hashCode` via record semantics
- `compact()` method: scan → group → merge → erase-then-store per group
- `createMergedRepresentative()`: weighted-average numerics by mergeCount, majority-vote categoricals, sum mergeCount
- `computeEntityId(MergeKey)`: `"compact-" + SHA256(key.toString()).substring(0, 12)`
- Erase originals first, then store representative (per failure-mode design)

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=CbrCompactionJobTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Add scheduler exclusion in test application.properties**

Add `io.casehub.clinical.cbr.CbrCompactionJob` to the `quarkus.arc.exclude-types` list in `runtime/src/test/resources/application.properties`.

- [ ] **Step 6: Run full test suite**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/cbr/CbrCompactionJob.java runtime/src/test/java/io/casehub/clinical/cbr/CbrCompactionJobTest.java runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#144): CbrCompactionJob — merge similar AE CBR cases into weighted representatives

Exact categorical merge key (grade, eventType, trialPhase, unexpected,
suspected). Weighted average numerics, majority vote categoricals.
Erase-before-store for failure safety. Disabled by default.

Refs #144"
```

---

### Task 4: Update DemoDataSeeder + production application.properties

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/demo/DemoDataSeeder.java` (if it calls `AeCbrFeatureBuilder` directly — update to use `AeCbrContext`)
- Modify: `runtime/src/main/resources/application.properties` (add compaction config defaults)

**Interfaces:**
- Consumes: `AeCbrContext` (Task 1)
- Produces: production-ready config for compaction (disabled by default)

- [ ] **Step 1: Find and update DemoDataSeeder callers**

Use `ide_find_references` on the old `AeCbrFeatureBuilder.buildFeatures` 6-param overload. Update all callers to use `AeCbrContext`. Once all callers are migrated, remove the 6-param delegation method.

- [ ] **Step 2: Add compaction config to production application.properties**

```properties
casehub.clinical.cbr.compaction.enabled=false
casehub.clinical.cbr.compaction.interval=168h
casehub.clinical.cbr.compaction.min-group-size=3
```

- [ ] **Step 3: Run full test suite**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add -A
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#132,#144): migrate callers to AeCbrContext, add compaction config

Refs #132, Refs #144"
```
