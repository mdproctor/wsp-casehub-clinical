# Guided Steps 1-2 Gap Closure — #98

Steps 1 and 2 were implemented under #105. Three gaps remain between
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

4. **`DemoDataSeeder`** — set per-site targets when creating sites:
   - Site A (dr-chen): 120
   - Site B (dr-martinez): 100
   - Site C (dr-okonkwo): 80
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

**Fix — `step2-agents.ts` only:**

Replace the two columns with a single "Endorsement" column:
```typescript
{ id: "attestationPositive" as ColumnId, label: "Endorsement",
  expression: '(function() { var pos = value || 0; var neg = row.attestationNegative || 0; var total = pos + neg; return total === 0 ? "—" : Math.round(100 * pos / total) + "%"; })()' }
```

Remove the `attestationNegative` column. The ratio is what matters for
trust assessment — raw counts are noise in a summary view.

## Gap 3 — Trust dimensions metric

**Problem:** The "Trust Dimensions Tracked" card is static markdown
(`"3 dimensions: safety-accuracy, eligibility-precision,
protocol-adherence"`). It should be data-driven.

**Fix — `step2-agents.ts` only:**

Replace the markdown card with:
```typescript
[metric({
  title: "Trust Dimensions",
  lookup: lookup("agents", groupBy(null, distinct("trustDimension")))
})]
```

The "Oversight Policy" card stays as markdown — it's a qualitative
policy statement, not a numeric value.

## Files touched

| File | Change |
|------|--------|
| `TrialSite.java` | Add `targetEnrollment` field |
| `V124__add_target_enrollment_to_trial_site.sql` | New migration |
| `TrialDashboardResource.java` | Add field to `SiteRow`, pass through |
| `DemoDataSeeder.java` | Set per-site targets |
| `step1-overview.ts` | Grouped bar chart |
| `step2-agents.ts` | Endorsement ratio, trust dimensions metric |

## Not changed

- Trust routing policy table — stays as static markdown (qualitative
  "below-threshold behavior" descriptions have no numeric data source)
- Oversight policy card — stays as markdown (qualitative statement)
- Narrative text — already accurate
- Sites table — already has all required columns
