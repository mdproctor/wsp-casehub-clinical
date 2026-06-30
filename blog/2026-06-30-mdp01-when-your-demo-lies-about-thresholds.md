---
layout: post
title: "When Your Demo Lies About Thresholds"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [design-review, pages-dsl, trust-routing]
---

The guided walkthrough pages for casehub-clinical looked complete — shipped under #105 with all 8 steps and 6 explore pages. Issue #98 asked for specific things: a target-vs-actual enrollment bar chart, an endorsement ratio column, data-driven trust dimension metrics. Three gaps, straightforward.

Then the design review found a fourth.

The static markdown table in Step 2 listed trust routing thresholds that didn't match `ClinicalTrustRoutingPolicyProvider`. Five of six rows were wrong. Three capabilities fell through to `TrustRoutingPolicy.DEFAULT` at 0.70 but the table showed 0.75 and 0.80. Two capabilities were absent entirely. The table had been hand-written when the policy provider was first implemented and never updated.

The fix was to delete it. The `/agents` endpoint already returns `threshold` per capability from `policy.threshold()` — adding a column to the data-driven table replaced a lie with a live value. No static table means no drift.

The review also caught two DSL limitations I'd have shipped. The pages expression evaluator doesn't support `row.` cross-column access — my endorsement ratio expression (`row.attestationNegative`) would have silently evaluated to undefined. And `join("trustDimension", ", ")` in the pages aggregation engine doesn't deduplicate — 8 agent rows with 3 unique dimensions would produce a string with 5 duplicates. Both moved to server-side computation in `AgentRow`.

Per-site `targetEnrollment` is now a proper domain field on `TrialSite`. FHIR R5 supports it, ICH E6(R3) expects it. The DemoDataSeeder distributes 120/100/80 across three sites, and the bar chart renders grouped columns.

The pattern here is worth noting: a data-driven table that renders from the same API the rest of the app uses cannot drift from reality. A static markdown table will always drift — it's a matter of when, not if. The design review caught it because it verified the spec against the actual `ClinicalTrustRoutingPolicyProvider` source. I wouldn't have checked.
