# Guided Steps 1-2 Gap Closure — #98

Steps 1 and 2 were implemented under #105. Four gaps remain between
the #98 spec and the delivered code.

## Gap 1 — Per-site target enrollment (bar chart: target vs actual)

**Problem:** The issue asks for "enrollment by site (target vs actual)."
The current bar chart shows only actual enrollment. `TrialSite` has no
`targetEnrollment` field — the target exists only at the trial level
(`ClinicalTrial.targetEnrollment`).

**Fix — domain model through UI:**

1. **`TrialSite.java`** — add `public int targetEnrollment;` with
   `@Column(name = "target_enrollment", nullable = false)` and default 0.

2. **`V124__add_target_enrollment_to_trial_site.sql`** in
   `db/migration/default/` — `ALTER TABLE trial_site ADD COLUMN
   target_enrollment INTEGER NOT NULL DEFAULT 0;`

3. **`TrialDashboardResource.SiteRow`** — add `int targetEnrollment` field.
   Pass `s.targetEnrollment` in the `/sites` endpoint mapping.

4. **`DemoDataSeeder`** — extend `addSite` signature to
   `addSite(UUID siteId, String investigatorId, int targetEnrollment)`.
   Set `site.targetEnrollment = targetEnrollment` inside the method.
   Update call sites:
   - `addSite(SITE_A_ID, "dr-chen", 120)`
   - `addSite(SITE_B_ID, "dr-martinez", 100)`
   - `addSite(SITE_C_ID, "dr-okonkwo", 80)`
   - Total: 300 = trial target

5. **`step1-overview.ts`** — change bar chart to grouped columns:
   ```typescript
   barChart({
     title: "Enrollment by Site: Target vs Actual",
     subtype: "column",
     lookup: lookup("sites", groupBy("investigatorId",
       col("investigatorId"),
       sum("targetEnrollment"),
       sum("enrolledCount")
     ))
   })
   ```

**Rationale:** FHIR R5 supports per-site enrollment targets. ICH E6(R3)
requires monitoring enrollment against targets. This is a legitimate
domain field, not a UI convenience.

## Gap 2 — Endorsement ratio in agent table

**Problem:** #98 asks for "endorsement ratio." The current table shows
two separate columns: `attestationPositive` and `attestationNegative`.

**Fix — server-side computation + UI change:**

1. **`TrialDashboardResource.AgentRow`** — add `String endorsementRatio`
   field. Compute in the `/agents` endpoint:
   ```java
   String endorsementRatio = (totalPositive + totalNegative) == 0
       ? null
       : Math.round(100.0 * totalPositive / (totalPositive + totalNegative)) + "%";
   ```

2. **`step2-agents.ts`** — replace the two attestation columns with:
   ```typescript
   { id: "endorsementRatio" as ColumnId, label: "Endorsement",
     expression: 'value ?? "—"' }
   ```
   Remove both `attestationPositive` and `attestationNegative` columns
   from the table display. The `value ?? "—"` expression handles the
   bootstrap case (no attestation data) consistently with the trust
   score and threshold columns.

The ratio is what matters for trust assessment — raw counts are noise
in a summary view. Computing server-side keeps the logic in typed,
testable Java code rather than an untyped expression string.

## Gap 3 — Trust dimensions metric

**Problem:** The "Trust Dimensions Tracked" card is static markdown
(`"3 dimensions: safety-accuracy, eligibility-precision,
protocol-adherence"`). It should be data-driven.

**Fix — `step2-agents.ts` only:**

Replace the markdown card with:
```typescript
[metric({
  title: "Trust Dimensions",
  lookup: lookup("agents", groupBy(null, join("trustDimension", ", ")))
})]
```

Uses `join` (not `distinct`) to produce the dimension names as a
comma-separated string rather than just a count. Add `join` to the
import from `@casehubio/pages-ui`.

The "Oversight Policy" card stays as markdown — it's a qualitative
policy statement, not a numeric value.

## Gap 4 — Trust routing policy table shows incorrect thresholds

**Problem:** The static markdown table in `step2-agents.ts` shows
threshold values that do not match `ClinicalTrustRoutingPolicyProvider`.
5 of 6 rows have wrong thresholds. 3 capabilities (`irb-consultation`,
`data-safety-monitoring`, `regulatory-submission`) fall through to
`TrustRoutingPolicy.DEFAULT` (threshold 0.70) but the table shows
specific values (0.75, 0.75, 0.80). Two capabilities
(`pi-authorisation`, `trial-supervisor`) are absent from the table
entirely despite being in `CAPABILITY_DIMENSIONS`.

**Fix — remove static table, add threshold column to agents table:**

1. **`step2-agents.ts`** — remove the static markdown policy table.
   Add `threshold` as a column in the agents table:
   ```typescript
   { id: "threshold" as ColumnId, label: "Threshold",
     expression: 'value != null ? value.toFixed(2) : "—"' }
   ```
   The `/agents` endpoint already returns `threshold` per capability
   from `policy.threshold()` — this is purely a UI change.

2. The "Below Threshold Behavior" text descriptions in the removed
   static table had no data source and were not backed by any code
   or configuration — they were aspirational. Removing them eliminates
   a permanent source of drift.

## Files touched

| File | Change |
|------|--------|
| `TrialSite.java` | Add `targetEnrollment` field |
| `V124__add_target_enrollment_to_trial_site.sql` | New migration |
| `TrialDashboardResource.java` | Add `targetEnrollment` to `SiteRow`, add `endorsementRatio` to `AgentRow` |
| `DemoDataSeeder.java` | Extend `addSite` with `targetEnrollment` param, set per-site targets |
| `step1-overview.ts` | Grouped bar chart |
| `step2-agents.ts` | Endorsement ratio column, trust dimensions `join`, threshold column, remove static policy table |

## Not changed

- Oversight policy card — stays as markdown (qualitative statement)
- Narrative text — already accurate
- Sites table — already has all required columns

## Test plan

1. **`TrialDashboardResourceTest`** — add assertions:
   - `/sites` response includes `targetEnrollment` per site
   - `/agents` response includes `threshold` per capability (non-null)
   - `/agents` response includes `endorsementRatio` (null when no
     attestations, formatted percentage string otherwise)

2. **V124 migration** — exercised automatically by Flyway startup in
   `@QuarkusTest`. No explicit migration test needed.

3. **`DemoDataSeederTest`** — the seeder is `@IfBuildProfile("dev")`
   so CDI does not register it in test profile. Full lifecycle
   integration is covered by `ThreeSiteShowcaseTest` which exercises
   the same service calls. Per-site target values are verified
   indirectly through the `/sites` endpoint in dashboard tests.
