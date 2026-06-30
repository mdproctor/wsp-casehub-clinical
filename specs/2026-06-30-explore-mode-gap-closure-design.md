# Explore Mode Gap Closure — Design Spec

**Date:** 2026-06-30
**Issues:** #101 (explore mode pages), #113 (AddSiteRequest targetEnrollment), #114 (trust-network.ts field mismatches)
**Branch:** issue-101-explore-mode-dashboards

## Problem

All 6 explore pages exist and are wired into navigation, but render broken data. 11 field name/type mismatches across 4 of 6 pages prevent correct rendering. The TS pages were written against an idealized schema; the Java API records diverge.

## Scope

| Category | Count | Fix side |
|----------|-------|----------|
| TS used wrong field name | 1 | TS only |
| Java returns wrong type | 2 | Java + TS |
| Java missing data UI needs | 7 | Java (enrich records) |
| TS column has no backing data | 1 | TS (remove column) |

Plus: trial dashboard missing bar chart, AddSiteRequest missing targetEnrollment, AdverseEvent entity missing eventType.

## Design

### 1. AdverseEvent Entity

Add `eventType` (String, nullable) to `AdverseEvent`. Modify `V103__adverse_event.sql` to include `event_type VARCHAR(100)`. No new migration — no installs exist.

### 2. Java API Record Changes (TrialDashboardResource)

**AgentRow** — type fixes:
```java
// Before
int maturityPhase, ..., String endorsementRatio
// After
String maturityPhase, ..., Double endorsementRatio
```
- `maturityPhase`: three-phase string ("bootstrap" <10, "emerging" 10–49, "established" ≥50)
- `endorsementRatio`: raw ratio 0.0–1.0, null when no attestations

**AdverseEventRow** — enrichments:
```java
// Add: String siteName, String patientId
// Rename: type → eventType (populated from entity)
```

**DeviationRow** — enrichments:
```java
// Add: String siteName, Instant reportedAt (from commandedAt), String irbDecision
```
- `irbDecision`: join through `IrbApproval.find("deviationId", id)` → `decision.name()`
- `reportedAt`: projected from entity's `commandedAt` (same instant, different semantic)

**LedgerEntryRow** — enrichment:
```java
// Add: String digest
```

### 3. TypeScript Explore Page Fixes

**trust-network.ts:**
- `totalDecisions` → `decisionCount`
- `endorsementRatio` expression: `(value * 100).toFixed(1) + "%"` with null guard (now Double)
- `maturityPhase` expression: already correct (now gets strings from Java)

**adverse-events.ts:**
- `eventType` now populated (was always null)
- `siteName` now a name string (was UUID)
- `patientId` now a readable ID (was missing)

**deviations.ts:**
- `siteName` now populated
- `reportedAt` now populated
- Remove `commitmentState` column — `piApprovalStatus` covers PI decision lifecycle
- `irbDecision` now populated from IrbApproval

**audit-trail.ts:**
- `digest` now populated from LedgerEntry

**trial-dashboard.ts:**
- Add enrollment bar chart (pattern from guided step1-overview.ts)
- Add `sitesDs` to datasets

### 4. SiteResource AddSiteRequest (#113)

Add `@PositiveOrZero int targetEnrollment` to `AddSiteRequest`. Wire in `SiteResource.add()`.

### 5. DemoDataSeeder

Populate `eventType` on seeded adverse events:
- Grade 1–2: NAUSEA, FATIGUE, RASH
- Grade 3–4: THROMBOCYTOPENIA, FEBRILE_NEUTROPENIA
- Grade 5: CARDIAC_ARREST

### 6. Tests

- `TrialDashboardResourceTest`: verify new fields (siteName, patientId, eventType, endorsementRatio type, maturityPhase string, irbDecision, digest)
- `SiteResourceTest`: verify targetEnrollment round-trips through AddSiteRequest

## Decisions

| Decision | Rationale |
|----------|-----------|
| `endorsementRatio` as Double not String | API returns structured data; UI formats for display |
| `maturityPhase` as String not int | Opaque int codes are bad API design; three named phases |
| Remove `commitmentState` from deviations TS | Cross-datasource qhorus query; `piApprovalStatus` is the right clinical abstraction |
| Project `commandedAt` as `reportedAt` | UI-facing concept; same instant in current system |
| Modify original migration not add new | No installs — clean slate |
| Add `eventType` to entity | CTCAE has both Grade and Preferred Term; Grade alone is incomplete domain modeling |

## Out of Scope

- Graph visualization for trust network (blocked on casehub-pages#41)
- Full CTCAE System Organ Class hierarchy (eventType as flat string is sufficient for demo)
- Commitment state from qhorus (file issue if detailed tracking needed)
