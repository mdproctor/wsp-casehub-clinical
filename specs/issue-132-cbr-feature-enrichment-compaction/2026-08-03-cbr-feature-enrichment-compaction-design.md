# CBR Feature Enrichment + Case Compaction

**Issues:** casehubio/clinical#132 (feature enrichment), casehubio/clinical#144 (compaction)
**Epic:** casehubio/clinical#115 (CBR roadmap)
**Date:** 2026-08-03
**Status:** Approved

## Summary

Two additions to clinical's CBR layer. #132 (enrichment) should land first — compaction's weighted averaging operates over the enriched feature set including the 3 new fields.

1. **#132** — Enrich `AeCbrFeatureBuilder` with 3 new features: site enrollment count, site target enrollment, and the safety-monitoring agent's trust score at case outcome time.
2. **#144** — Add `CbrCompactionJob` that merges similar CBR cases into weighted representatives, reducing storage and retrieval cost for the `clinical-ae` domain.

## #132 — Feature Enrichment

### New Features

| Feature | Field type | Source | Query feature? |
|---------|-----------|--------|---------------|
| `siteEnrollmentCount` | `numeric(0, 10000)` | `PatientEnrollment.count("siteId = ?1", siteId)` | Yes |
| `siteTargetEnrollment` | `numeric(0, 10000)` | `TrialSite.targetEnrollment` | Yes |
| `agentTrustScore` | `numeric(0, 1)` | `ActorTrustScore` for the safety-monitoring agent | No (outcome feature) |

### Builder Changes

`AeCbrFeatureBuilder` stays a static utility — no CDI dependencies. To avoid a 9-parameter method, introduce `AeCbrContext` record in the same package:

```java
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

The builder methods become:

```java
public static Map<String, Object> buildFeatures(AeCbrContext ctx)
public static Map<String, Object> buildQueryFeatures(AeCbrContext ctx)
public static String buildProblemSummary(AeCbrContext ctx)
public static String buildSolutionSummary(AeCbrContext ctx)
```

`buildQueryFeatures()` gains `siteEnrollmentCount` and `siteTargetEnrollment` but NOT `agentTrustScore` — trust is an outcome-time observation, not a query-time filter.

### Schema Changes

`ClinicalCbrSchemaInitializer.aeSchema()` gains 3 fields:

```java
FeatureField.numeric("siteEnrollmentCount", 0, 10000),
FeatureField.numeric("siteTargetEnrollment", 0, 10000),
FeatureField.numeric("agentTrustScore", 0, 1)
```

Total: 14 features (was 11). The `mergeCount` field (used by compaction) is also registered here: `FeatureField.numeric("mergeCount", 1, 100000)` — 15 fields total in the schema. Non-compacted cases don't set it (the field is optional in the store).

### Caller Changes — ClinicalCaseOutcomeObserver

`handleAeCase()` already navigates `ae → enrollment → site → trial`. Two additions:

1. **Site enrollment count:** `PatientEnrollment.count("siteId = ?1", site.id)` — added to `EntityResolver` interface as `countEnrollmentsAtSite(UUID siteId)`.

2. **Agent trust score:** Extract the safety-monitoring agent's actorId from `PlanTrace` (already built in `buildPlanTraces()`). Query `ActorTrustScore` for `(actorId, scoreType=BAYESIAN_BETA, capabilityKey="safety-monitoring", dimensionKey="safety-accuracy")`. If no score exists (bootstrap phase), use `0.5` (uninformative prior).

`ActorTrustScore` is a ledger entity in the qhorus persistence unit. `ClinicalCaseOutcomeObserver` runs `@Transactional` on the default datasource. The trust score query is read-only and crosses datasource boundaries — this requires XA on both datasources (already configured in clinical's `application.properties`). The `EntityResolver` interface gains `findAgentTrustScore(String actorId)` returning `double`. The `PanacheEntityResolver` implementation uses the named query `ActorTrustScore.findCapabilityDimensionByKeys` with `(actorId, BAYESIAN_BETA, "safety-monitoring", "safety-accuracy")`.

### Data Flow

```
CaseOutcomeEvent
  → ClinicalCaseOutcomeObserver.handleAeCase()
    → resolve ae, enrollment, site, trial (existing)
    → count enrollments at site (NEW)
    → extract agent actorId from PlanTrace (NEW)
    → query ActorTrustScore for safety-accuracy (NEW)
    → AeCbrFeatureBuilder.buildFeatures(..., siteEnrollmentCount, siteTargetEnrollment, agentTrustScore)
    → store PlanCbrCase
```

### AE Trajectory Cases

The trajectory CBR case inherits the same base features. `ClinicalCaseOutcomeObserver.handleAeCase()` builds trajectory features from the same `features` map — the 3 new features propagate automatically.

## #144 — Case Compaction

### CbrCompactionJob

New `@ApplicationScoped` class in `io.casehub.clinical.cbr`.

**Schedule:** Configurable interval, disabled by default. Must run AFTER `CbrRetentionPurgeJob` — purge removes TTL-expired and over-limit cases first, then compaction merges what remains. Both jobs are scheduled independently (Quartz), so ordering is by convention: compaction's default interval (168h) offsets from purge by checking that purge ran recently (log-based, not enforced).

```properties
casehub.clinical.cbr.compaction.enabled=false
casehub.clinical.cbr.compaction.interval=168h
casehub.clinical.cbr.compaction.min-group-size=3
```

**Scope:** `clinical-ae` domain only. Other domains (trajectory, deviation, amendment, trial-safety, site-enrollment) have different feature shapes — extend later if needed.

### Merge Key

Exact match on 5 categorical problem features:

| Feature | Type |
|---------|------|
| `grade` | numeric (bucketed to integer) |
| `eventType` | categoricalList |
| `trialPhase` | categorical |
| `unexpected` | categorical |
| `suspected` | categorical |

Cases with identical values for all 5 features are merge candidates.

### Algorithm

```
for each tenant in store.discoverTenants(AE):
    cursor = null
    groups = HashMap<MergeKey, List<CbrCaseSummary>>
    do:
        result = store.scan(new CbrScanRequest(tenant, AE, "clinical-ae", 100, cursor))
        for each summary in result.items():
            // Retrieve full case by entityId — CbrQuery with exact entityId match
            fullCase = store.retrieveSimilar(exactQuery(summary.entityId()), PlanCbrCase.class)
            if empty, skip (case may have been erased between scan and retrieve)
            compute merge key from the 5 categorical features in fullCase.features()
            add (summary, fullCase) to group
        cursor = result.nextCursor()
    while result.hasMore()

    // Note: scan + per-case retrieve is O(N) queries. Acceptable for a weekly batch
    // job on a bounded case set (post-purge). If case counts grow large, consider
    // adding a bulk retrieve to CbrCaseMemoryStore API (neocortex change).

    for each group where size >= minGroupSize:
        merged = createMergedRepresentative(group)
        store.store(merged, "clinical-ae", "compact-<mergeKeyHash>", AE, tenant, null, Path.root())
        for each original in group:
            store.erase(new EraseRequest(original.caseId(), tenant))
        log("Compacted %d cases into 1 representative", group.size())
```

### Merged Representative

- **Numeric features** (`priorAeCount`, `siteEnrollmentCount`, `siteTargetEnrollment`, `agentTrustScore`): weighted average across group members, weighted by each case's `mergeCount` (1 for non-compacted cases). `grade` is part of the merge key so it's constant within a group — NOT averaged.
- **Categorical outcome features** (`safetyReviewOutcome`, `dsmbEscalated`, `indReportFiled`, `susarOversight`): majority vote (most frequent value), weighted by `mergeCount`
- **confidence:** weighted average of individual confidences, weighted by `mergeCount`
- **mergeCount:** sum of all group members' `mergeCount` values (non-compacted cases have implicit `mergeCount=1`). Stored as `FeatureField.numeric("mergeCount", 1, 100000)` — retrievers see this represents N observations.
- **problem/solution text:** from the most recently stored case in the group
- **entityId:** `compact-<SHA256(mergeKey).substring(0,12)>` — deterministic, idempotent across runs
- **planTraces:** empty list (individual traces lose meaning after merge)

### Failure Handling

Compaction erases originals BEFORE storing the representative. This means a crash mid-compaction loses the group's data rather than creating duplicates. The rationale: lost compaction candidates can be re-derived from future case ingestion, but duplicate cases pollute retrieval results permanently. Each group's erase-then-store is wrapped in try-catch — a failure on one group does not block other groups.

### Concurrency

A new case ingested during the scan window may be missed by compaction. This is acceptable — the case will be picked up on the next compaction run. No locking is needed.

### Idempotency

Compaction is idempotent. On subsequent runs:
- Already-compacted representatives have entityId `compact-<hash>` — the scan includes them
- If the group containing a compact representative has size < `minGroupSize`, nothing happens
- If new cases joined the same merge key, the old representative is erased and a new one is stored with the updated weighted average

### Scheduler Exclusion in Tests

Same pattern as `SponsorNotificationRetryJob` and `SiteEnrollmentTrajectoryJob`:

```properties
quarkus.arc.exclude-types=...io.casehub.clinical.cbr.CbrCompactionJob
```

Tests drive compaction directly via a `compact()` method.

## What This Does NOT Do

- No similarity-based clustering (exact categorical match only)
- No cross-domain compaction (clinical-ae only)
- No LLM-generated merged summaries (uses most recent original's text)
- No schema migration — CBR store is schemaless; new features are additive
- No changes to retrieval — existing `retrieveSimilar()` works unchanged on merged cases

## Testing Strategy

### #132 — Feature Enrichment

**Unit tests (AeCbrFeatureBuilderTest):**
- `buildFeatures()` with the 3 new parameters produces correct map entries
- `buildQueryFeatures()` includes site features but excludes agentTrustScore
- Edge cases: site with 0 enrollments, target enrollment 0, trust score at bounds (0.0, 1.0)
- Default trust score 0.5 when no ActorTrustScore found

**Unit tests (ClinicalCaseOutcomeObserverTest):**
- EntityResolver mock returns enrollment count and trust score
- PlanTrace extraction yields correct agentId for trust lookup
- Missing PlanTrace (no safety-monitoring binding) → trust score defaults to 0.5

**Integration test (ClinicalCaseOutcomeObserverIntegrationTest):**
- Full case outcome flow stores CBR case with all 14 features
- Verify stored features include siteEnrollmentCount, siteTargetEnrollment, agentTrustScore

### #144 — Case Compaction

**Unit tests (CbrCompactionJobTest):**
- 3+ cases with same merge key → compacted into 1 representative
- 2 cases with same merge key (below threshold) → no compaction
- Cases with different merge keys → each group handled independently
- Weighted average of numeric features is correct
- Majority vote of categorical outcome features is correct
- `mergeCount` feature set to group size
- EntityId is deterministic for same merge key
- Idempotency: running twice produces same result
- Re-compaction weighting: compact representative (mergeCount=5) + 2 new singles → new representative has mergeCount=7 and weighted averages respect the 5:1:1 ratio

**Integration test:**
- Store 5 AE CBR cases with same categoricals, different numerics
- Run `compact()`
- Verify: 1 representative stored, 5 originals erased
- Verify: representative features are weighted averages
- Retrieve similar → merged representative appears with correct mergeCount
