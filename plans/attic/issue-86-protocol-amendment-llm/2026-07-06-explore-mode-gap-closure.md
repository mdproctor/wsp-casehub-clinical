# Explore Mode Gap Closure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #101 — Explore mode — 6 dashboard pages
**Issue group:** #101, #113, #114

**Goal:** Fix 11 field name/type mismatches across 4 explore pages, enrich Java API records to serve the UI, add `eventType` to the domain model, and add `targetEnrollment` to `AddSiteRequest`.

**Architecture:** Enrich Java API response records in `TrialDashboardResource` to provide human-readable data (site names, patient IDs, event types, IRB decisions) that the TS pages expect. Fix TS column IDs and expressions to match the enriched Java field names. Add the missing `eventType` field to the `AdverseEvent` entity (no new migration — modify original). Add enrollment bar chart to trial dashboard.

**Tech Stack:** Java 21 / Quarkus 3.32.2, TypeScript / casehub-pages DSL

## Global Constraints

- No Flyway migration files — modify original V103 in place (no installs exist)
- Tests use `drop-and-create` + Flyway disabled — entity field additions work without migration changes in tests
- `@QuarkusTest` + `@TestSecurity` pattern for REST endpoint tests
- Pages DSL: `barChart`, `table`, `metric`, `selector`, `lookup`, `groupBy`, `col`, `sum`, `filterBy`, `sortBy`
- Build with `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`

---

### Task 1: Add `eventType` to AdverseEvent entity and migration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Modify: `runtime/src/main/resources/db/migration/default/V103__adverse_event.sql`

**Interfaces:**
- Consumes: nothing
- Produces: `AdverseEvent.eventType` (String, nullable) — used by Task 3 and Task 4

- [ ] **Step 1: Add `eventType` field to `AdverseEvent` entity**

Add after line 33 (`public CtcaeGrade grade;`):

```java
@Column(name = "event_type")
public String eventType;
```

- [ ] **Step 2: Add `event_type` column to V103 migration**

Add `event_type VARCHAR(100),` after the `outcome` line in `V103__adverse_event.sql`:

```sql
CREATE TABLE adverse_event (
    id            UUID        NOT NULL,
    enrollment_id UUID        NOT NULL,
    grade         VARCHAR(50) NOT NULL,
    actuality     VARCHAR(50) NOT NULL DEFAULT 'ACTUAL',
    outcome       VARCHAR(50) NOT NULL DEFAULT 'ONGOING',
    event_type    VARCHAR(100),
    occurred_at   TIMESTAMP   NOT NULL,
    reported_at   TIMESTAMP   NOT NULL,
    sla_deadline  TIMESTAMP,
    CONSTRAINT pk_adverse_event PRIMARY KEY (id),
    CONSTRAINT fk_ae_enrollment FOREIGN KEY (enrollment_id) REFERENCES patient_enrollment(id)
);
```

- [ ] **Step 3: Compile to verify**

Run: `mvn compile -pl api,runtime --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
feat: add eventType field to AdverseEvent entity

CTCAE v5.0 has both Grade (severity) and Preferred Term (event type).
Grade alone is incomplete domain modeling.

Refs casehubio/clinical#101
```

---

### Task 2: Add `targetEnrollment` to `AddSiteRequest` (#113)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/SiteResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/SiteResourceTest.java`

**Interfaces:**
- Consumes: `TrialSite.targetEnrollment` (already exists on entity)
- Produces: `AddSiteRequest(String investigatorId, int targetEnrollment)` — REST API contract

- [ ] **Step 1: Write the failing test**

Add to `SiteResourceTest.java`:

```java
@Test
void post_site_with_target_enrollment_persists_value() {
    String trialLocation = createTrial();
    UUID trialId = UUID.fromString(trialLocation.substring(trialLocation.lastIndexOf('/') + 1));

    String siteLocation =
        given()
            .contentType("application/json")
            .body("""
                {"investigatorId": "pi-target-001", "targetEnrollment": 150}
                """)
        .when()
            .post("/trials/{id}/sites", trialId)
        .then()
            .statusCode(201)
            .extract().header("Location");

    given()
    .when()
        .get(siteLocation)
    .then()
        .statusCode(200)
        .body("investigatorId", equalTo("pi-target-001"))
        .body("targetEnrollment", equalTo(150));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=SiteResourceTest#post_site_with_target_enrollment_persists_value --batch-mode`
Expected: FAIL — `targetEnrollment` is 0 (default) because `AddSiteRequest` doesn't have the field

- [ ] **Step 3: Update `AddSiteRequest` and `add()` method**

In `SiteResource.java`, change the record and wire it:

```java
public record AddSiteRequest(@NotBlank String investigatorId,
                              @PositiveOrZero int targetEnrollment) {}
```

In `SiteResource.add()`, add after `site.investigatorId = req.investigatorId();`:

```java
site.targetEnrollment = req.targetEnrollment();
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=SiteResourceTest#post_site_with_target_enrollment_persists_value --batch-mode`
Expected: PASS

- [ ] **Step 5: Run full SiteResourceTest suite**

Run: `mvn test -pl runtime -Dtest=SiteResourceTest --batch-mode`
Expected: All tests PASS (existing tests send no `targetEnrollment` — defaults to 0)

- [ ] **Step 6: Commit**

```
feat: add targetEnrollment to SiteResource.AddSiteRequest

Closes casehubio/clinical#113
```

---

### Task 3: Enrich Java API records in TrialDashboardResource

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/TrialDashboardResource.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/resource/TrialDashboardResourceTest.java`

**Interfaces:**
- Consumes: `AdverseEvent.eventType` (from Task 1), `IrbApproval.deviationId` / `IrbApproval.decision`
- Produces: enriched JSON fields consumed by TS explore pages (Task 5)

- [ ] **Step 1: Write failing tests for AgentRow type changes**

Add to `TrialDashboardResourceTest.java`:

```java
@Test
void agents_returns_endorsement_ratio_as_number_not_string() {
    given()
        .when().get("/trials/{trialId}/agents", trialId)
        .then()
        .statusCode(200)
        .body("[0].maturityPhase", isA(String.class))
        .body("[0].decisionCount", notNullValue());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest#agents_returns_endorsement_ratio_as_number_not_string --batch-mode`
Expected: FAIL — `maturityPhase` is currently an `int`, not a `String`

- [ ] **Step 3: Fix AgentRow record and agents() endpoint**

Change the `AgentRow` record (line 62):

```java
public record AgentRow(
    String capability, String trustDimension,
    Double trustScore, Double threshold, String maturityPhase,
    int decisionCount, int attestationPositive, int attestationNegative,
    Double endorsementRatio, String distinctTrustDimensions
) {}
```

In the `agents()` method, replace the maturity computation (line 280):

```java
String maturity;
if (totalDecisions < 10) maturity = "bootstrap";
else if (totalDecisions < 50) maturity = "emerging";
else maturity = "established";
```

Replace the `endorsementRatio` computation (lines 282-284):

```java
Double endorsementRatio = totalAttestations == 0
    ? null
    : (double) totalPositive / totalAttestations;
```

Update the empty-scores return (line 272-273) — change `0` to `"bootstrap"` for maturityPhase, and `null` stays null for endorsementRatio (already null):

```java
return new AgentRow(capability, dimension, null,
    policy.threshold(), "bootstrap", 0, 0, 0, null, distinctDimensions);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest#agents_returns_endorsement_ratio_as_number_not_string --batch-mode`
Expected: PASS

- [ ] **Step 5: Write failing tests for AdverseEventRow enrichment**

Add to `TrialDashboardResourceTest.java`:

```java
@Test
void adverse_events_returns_enriched_fields() {
    given()
        .when().get("/trials/{trialId}/adverse-events", trialId)
        .then()
        .statusCode(200)
        .body("[0].siteName", equalTo("dr-chen"))
        .body("[0].patientId", equalTo("P-001"));
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest#adverse_events_returns_enriched_fields --batch-mode`
Expected: FAIL — `siteName` and `patientId` fields don't exist in response

- [ ] **Step 7: Enrich AdverseEventRow record and adverseEvents() endpoint**

Change the `AdverseEventRow` record (line 51):

```java
public record AdverseEventRow(
    UUID id, UUID enrollmentId, UUID siteId, String siteName,
    String patientId, String grade, String eventType,
    Instant reportedAt, Instant slaDeadline, String escalationStatus,
    String regulatorySubmissionStatus, String slaTimeRemaining
) {}
```

In the `adverseEvents()` method, build lookup maps after the existing `enrollmentToSite` map (after line 191):

```java
Map<UUID, String> siteIdToName = sites.stream()
    .collect(Collectors.toMap(s -> s.id, s -> s.investigatorId));
Map<UUID, String> enrollmentToPatientId = enrollments.stream()
    .collect(Collectors.toMap(e -> e.id, e -> e.patientId));
```

Update the `AdverseEventRow` constructor call (replace lines 208-215):

```java
return new AdverseEventRow(
    ae.id, ae.enrollmentId, enrollmentToSite.get(ae.enrollmentId),
    siteIdToName.get(enrollmentToSite.get(ae.enrollmentId)),
    enrollmentToPatientId.get(ae.enrollmentId),
    ae.grade != null ? ae.grade.name() : null,
    ae.eventType,
    ae.reportedAt, ae.slaDeadline,
    ae.escalationStatus != null ? ae.escalationStatus.name() : null,
    ae.regulatorySubmissionStatus != null ? ae.regulatorySubmissionStatus.name() : null,
    slaRemaining
);
```

- [ ] **Step 8: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest#adverse_events_returns_enriched_fields --batch-mode`
Expected: PASS

- [ ] **Step 9: Write failing tests for DeviationRow enrichment**

First, set up a deviation in `@BeforeEach` — add after the AE persist block:

```java
ProtocolDeviation dev = new ProtocolDeviation();
dev.id = UUID.randomUUID();
dev.tenantId = principal.tenancyId();
dev.siteId = siteAId;
dev.deviationType = "CONSENT_VIOLATION";
dev.severity = DeviationSeverity.MAJOR;
dev.piApprovalStatus = PiApprovalStatus.APPROVED;
dev.commandedAt = Instant.now();
dev.persist();
```

Add the import for `DeviationSeverity` and `PiApprovalStatus`.

Then add the test:

```java
@Test
void deviations_returns_enriched_fields() {
    given()
        .when().get("/trials/{trialId}/deviations", trialId)
        .then()
        .statusCode(200)
        .body("size()", greaterThanOrEqualTo(1))
        .body("[0].siteName", equalTo("dr-chen"))
        .body("[0].reportedAt", notNullValue());
}
```

- [ ] **Step 10: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest#deviations_returns_enriched_fields --batch-mode`
Expected: FAIL — `siteName` and `reportedAt` fields don't exist

- [ ] **Step 11: Enrich DeviationRow record and deviations() endpoint**

Change the `DeviationRow` record (line 57):

```java
public record DeviationRow(
    UUID id, UUID siteId, String siteName, String deviationType,
    String severity, String piApprovalStatus,
    Instant reportedAt, String irbDecision
) {}
```

In the `deviations()` method, build the site name lookup after the sites query (after line 231):

```java
Map<UUID, String> siteIdToName = sites.stream()
    .collect(Collectors.toMap(s -> s.id, s -> s.investigatorId));
```

Update the row mapping (replace lines 238-243):

```java
List<DeviationRow> rows = devs.stream().map(d -> {
    String irbDecision = null;
    IrbApproval irb = IrbApproval.find("deviationId", d.id).firstResult();
    if (irb != null) {
        irbDecision = irb.decision.name();
    }
    return new DeviationRow(
        d.id, d.siteId, siteIdToName.get(d.siteId), d.deviationType,
        d.severity != null ? d.severity.name() : null,
        d.piApprovalStatus != null ? d.piApprovalStatus.name() : null,
        d.commandedAt, irbDecision
    );
}).toList();
```

Add the import: `import io.casehub.clinical.entity.IrbApproval;`

- [ ] **Step 12: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest#deviations_returns_enriched_fields --batch-mode`
Expected: PASS

- [ ] **Step 13: Enrich LedgerEntryRow with digest**

Change the `LedgerEntryRow` record (line 78):

```java
public record LedgerEntryRow(
    UUID id, UUID subjectId, int sequenceNumber,
    String entryType, String actorId, String actorRole,
    Instant occurredAt, String digest, String summary
) {}
```

Update the row mapping in `ledgerEntries()` (the `.map(entry -> new LedgerEntryRow(...))` call) — add `entry.digest` between `entry.occurredAt` and `buildLedgerSummary(entry)`:

```java
.map(entry -> new LedgerEntryRow(
    entry.id, entry.subjectId, entry.sequenceNumber,
    entry.entryType != null ? entry.entryType.name() : null,
    entry.actorId, entry.actorRole,
    entry.occurredAt,
    entry.digest,
    buildLedgerSummary(entry)
))
```

- [ ] **Step 14: Update existing test assertions**

The existing `agents_returns_capability_list_with_trust_data` test checks `endorsementRatio` as `nullValue()`. Since the type changed from `String` to `Double`, this still works (null is null regardless of type). Update the test to also verify `maturityPhase` is a string:

```java
.body("[0].maturityPhase", equalTo("bootstrap"))
```

- [ ] **Step 15: Run full TrialDashboardResourceTest suite**

Run: `mvn test -pl runtime -Dtest=TrialDashboardResourceTest --batch-mode`
Expected: All tests PASS

- [ ] **Step 16: Commit**

```
feat: enrich API records — siteName, patientId, eventType, endorsementRatio, maturityPhase, irbDecision, digest

AgentRow: endorsementRatio String→Double, maturityPhase int→String (3-phase)
AdverseEventRow: add siteName, patientId, eventType; remove type
DeviationRow: add siteName, reportedAt (from commandedAt), irbDecision
LedgerEntryRow: add digest

Refs casehubio/clinical#101, casehubio/clinical#114
```

---

### Task 4: Populate `eventType` in DemoDataSeeder

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/demo/DemoDataSeeder.java`

**Interfaces:**
- Consumes: `AdverseEvent.eventType` (from Task 1)
- Produces: seeded data with event types for demo rendering

- [ ] **Step 1: Add eventType to Grade 2 AE (reportGrade2Ae)**

In `reportGrade2Ae()` (line 226), add after `ae.suspected = true;` (line 237):

```java
ae.eventType = "NAUSEA";
```

- [ ] **Step 2: Add eventType to Grade 4 SUSAR AEs (seedSingleSusarLifecycle)**

In `seedSingleSusarLifecycle()` (line 262), add after `ae.suspected = true;` (line 275):

```java
ae.eventType = index == 1 ? "THROMBOCYTOPENIA" : index == 2 ? "FEBRILE_NEUTROPENIA" : "HEPATOTOXICITY";
```

- [ ] **Step 3: Run dev mode to verify seeded data renders**

Run: `mvn quarkus:dev -pl runtime`
Check: `http://localhost:8080/trials/<TRIAL_ID>/adverse-events` returns `eventType` values

- [ ] **Step 4: Commit**

```
feat: populate eventType on seeded adverse events

CTCAE Preferred Terms: NAUSEA (Grade 2), THROMBOCYTOPENIA/
FEBRILE_NEUTROPENIA/HEPATOTOXICITY (Grade 4 SUSAR).

Refs casehubio/clinical#101
```

---

### Task 5: Fix TypeScript explore pages

**Files:**
- Modify: `runtime/src/main/webui/src/explore/trust-network.ts`
- Modify: `runtime/src/main/webui/src/explore/adverse-events.ts`
- Modify: `runtime/src/main/webui/src/explore/deviations.ts`
- Modify: `runtime/src/main/webui/src/explore/audit-trail.ts`
- Modify: `runtime/src/main/webui/src/explore/trial-dashboard.ts`

**Interfaces:**
- Consumes: enriched Java API records (from Task 3)
- Produces: working explore pages that render correctly with seeded data

- [ ] **Step 1: Fix trust-network.ts**

Change `totalDecisions` column (line 70):

```typescript
{ id: "decisionCount" as ColumnId, label: "Total Decisions" },
```

Change `endorsementRatio` expression (lines 71-76) — now a Double, not a pre-formatted string:

```typescript
{ id: "endorsementRatio" as ColumnId, label: "Endorsement Ratio",
  expression: `
    if (value == null) return "—";
    return (value * 100).toFixed(1) + "%";
  ` }
```

- [ ] **Step 2: Fix adverse-events.ts**

Change `eventType` column (line 32) — rename from `type` and keep formatter:

```typescript
{ id: "eventType" as ColumnId, label: "Event Type",
  expression: 'value ? value.replace(/_/g, " ").toLowerCase().replace(/\\b\\w/g, l => l.toUpperCase()) : "—"' },
```

Change `siteName` column (line 33) — now a string, remove UUID truncation:

```typescript
{ id: "siteName" as ColumnId, label: "Site" },
```

Change `patientId` column (lines 35-36) — now a readable ID, simplify:

```typescript
{ id: "patientId" as ColumnId, label: "Patient" },
```

- [ ] **Step 3: Fix deviations.ts**

Change `siteName` column (line 37) — now a string:

```typescript
{ id: "siteName" as ColumnId, label: "Site" },
```

Change `reportedAt` column (lines 38-39) — field now exists (projected from commandedAt):

```typescript
{ id: "reportedAt" as ColumnId, label: "Reported",
  expression: 'value ? new Date(value).toLocaleString() : "—"' },
```

Remove `commitmentState` column (lines 50-58) entirely — `piApprovalStatus` covers it.

Change `irbDecision` column (lines 59-60) — now populated from IrbApproval:

```typescript
{ id: "irbDecision" as ColumnId, label: "IRB Decision",
  expression: `
    if (!value || value === "PENDING") return "—";
    if (value === "APPROVED") return "✅ APPROVED";
    if (value === "REJECTED") return "❌ REJECTED";
    if (value === "DEFERRED") return "⏳ DEFERRED";
    if (value === "EXPIRED") return "⏰ EXPIRED";
    return value;
  ` }
```

- [ ] **Step 4: Fix audit-trail.ts**

Change `digest` column (lines 48-49) — field now populated:

```typescript
{ id: "digest" as ColumnId, label: "Digest (SHA-256)",
  expression: 'value ? value.substring(0, 16) + "..." : "—"' }
```

- [ ] **Step 5: Add enrollment bar chart to trial-dashboard.ts**

Add `barChart` to imports:

```typescript
import { page, columns, metric, barChart, table, markdown, lookup, groupBy, col, sum, sortBy } from "@casehubio/pages-ui";
```

Add `sitesDs` to imports:

```typescript
import { trialSummaryDs, sitesDs, ledgerEntriesDs } from "../datasets";
```

Add the bar chart component after the metrics `columns()` block and before the `table()`:

```typescript
barChart({
  title: "Enrollment by Site: Target vs Actual",
  subtype: "column",
  lookup: lookup("sites", groupBy("investigatorId",
    col("investigatorId"),
    sum("targetEnrollment"),
    sum("enrolledCount")
  ))
}),
```

Update the datasets array at the end to include `sitesDs`:

```typescript
{ datasets: [trialSummaryDs, sitesDs, ledgerEntriesDs] }
```

- [ ] **Step 6: Build webui to verify TypeScript compiles**

Run: `cd runtime/src/main/webui && npm run build`
Expected: Build succeeds with no errors

- [ ] **Step 7: Commit**

```
feat: fix explore page field mismatches + add trial dashboard bar chart

trust-network.ts: totalDecisions→decisionCount, endorsementRatio Double format
adverse-events.ts: eventType, siteName, patientId now populated
deviations.ts: siteName, reportedAt, irbDecision; remove commitmentState
audit-trail.ts: digest now populated
trial-dashboard.ts: enrollment bar chart with sitesDs

Closes casehubio/clinical#114
Refs casehubio/clinical#101
```

---

### Task 6: Full integration verification

**Files:**
- No new files — verification only

**Interfaces:**
- Consumes: all changes from Tasks 1–5

- [ ] **Step 1: Run full test suite**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: All tests PASS

- [ ] **Step 2: Run dev mode and verify explore pages**

Run: `mvn quarkus:dev -pl runtime`
Verify each explore page:
1. Trial Dashboard — metrics + bar chart + recent activity table
2. Site Detail — site selector + patient table with cross-filtering
3. Adverse Events — all columns populated (grade, eventType, siteName, patientId, SLA, escalation)
4. Protocol Deviations — siteName, reportedAt, piApprovalStatus, irbDecision
5. Audit Trail — digest column populated, Merkle verification works
6. Trust Network — decisionCount, endorsementRatio as percentage, maturityPhase as string

- [ ] **Step 3: Commit any final fixes**

If verification reveals minor issues, fix and commit with:

```
fix: explore page rendering corrections from integration verification

Refs casehubio/clinical#101
```
