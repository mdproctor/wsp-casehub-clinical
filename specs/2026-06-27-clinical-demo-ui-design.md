# Clinical Trial Demo UI — Design Spec

**Date:** 2026-06-27 (revised)
**Epic:** casehubio/clinical#93
**casehub-pages Epic:** casehubio/casehub-pages#50

## Purpose

A self-narrating demo UI for casehub-clinical that tells the AI Fusion story — how CaseHub governs AI agent decisions with trust routing, oversight gates, and tamper-evident audit trails. The demo must work without a presenter: someone clones the repo, runs `mvn quarkus:dev`, opens the browser, and understands what CaseHub does and why it matters.

## Audience (priority order)

1. **Investors / board** — "This platform makes regulated AI safe and auditable — no one else does this"
2. **Technical evaluators / CTOs** — "The architecture is sound — trust routing, Merkle audit, and compliance gates are production-grade"
3. **Clinical operations / pharma** — "This system can run Phase III trials with AI agents we can trust"

Target context: open source community adoption and buy-in from peers at work.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Data strategy | Hybrid — pre-seeded baseline + live actions | Immediate visual impact on load; live actions prove it's real |
| Navigation | Narrative flow with dashboard backing | Guides investors through the story; CTOs can break out and explore |
| Visual identity | Balanced professional, light/dark toggle | Neutral for all audiences, rethemeable later |
| Async feedback | Polling with visual indicators | Uses existing casehub-pages refresh; avoids custom event streaming |
| Architecture | Quinoa embedded in casehub-clinical | Single artifact, single `mvn quarkus:dev`, authentic for open source |

---

## casehub-pages DSL Integration

Every page in the demo is a **casehub-pages DSL composition** — not a custom SPA. Pages are built using the TypeScript DSL from `@casehubio/pages-ui`, datasets bind to REST URLs via `dataset()`, and the runtime is mounted via `loadSite()` from `@casehubio/pages-runtime`.

### Package dependencies

```json
{
  "dependencies": {
    "@casehubio/pages-runtime": "0.2.0",
    "@casehubio/pages-ui": "0.2.0"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

`pages-runtime` transitively includes `pages-viz` (charts, tables), `pages-component` (layout), and `pages-data` (datasets). `pages-ui` provides the DSL builders.

### Entry point (`src/index.ts`)

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { dashboard } from "./dashboard";

const container = document.getElementById("app");
if (container) {
  loadSite(container, dashboard).catch(console.error);
}
```

### Build config (`esbuild.config.mjs`)

```javascript
import { build, context } from "esbuild";

const isWatch = process.argv.includes("--watch");
const options = {
  entryPoints: ["src/index.ts"],
  bundle: true,
  outfile: "dist/app.js",
  format: "esm",
  target: "es2020",
  minify: !isWatch,
  sourcemap: isWatch,
};

if (isWatch) {
  const ctx = await context(options);
  await ctx.watch();
} else {
  await build(options);
}
```

### Page composition pattern

Each guided step and explore page is a `page()` call returning a DSL component tree:

```typescript
// Example: Step 1 — Trial Overview
import { page, columns, metric, barChart, table, markdown,
         dataset, lookup, groupBy, col, count, sum } from "@casehubio/pages-ui";
import type { DataSetId, ColumnId } from "@casehubio/data";

const trialSummary = "trial-summary" as DataSetId;
const sites = "sites" as DataSetId;

export const step1Overview = page("1. Trial Overview",
  markdown(`## ONCO-2024-001 — Phase III Oncology Trial
CaseHub coordinates AI agents for eligibility screening, safety monitoring,
and protocol review — each governed by trust scores, oversight gates,
and a tamper-evident audit trail.`),

  columns(
    { span: 3 }, metric({ title: "Trial Phase", lookup: lookup(trialSummary) }),
    { span: 3 }, metric({ title: "Total Enrolled",
      lookup: lookup(trialSummary, [], [groupBy([], [sum("enrolled" as ColumnId)])]) }),
    { span: 3 }, metric({ title: "Active AEs",
      lookup: lookup(trialSummary, [], [groupBy([], [sum("activeAeCount" as ColumnId)])]) }),
    { span: 3 }, metric({ title: "AI Agents Active",
      lookup: lookup("agents" as DataSetId, [], [groupBy([], [count("id" as ColumnId)])]) })
  ),

  barChart({
    title: "Enrollment by Site",
    lookup: lookup(sites, [], [
      groupBy(["siteName" as ColumnId], [col("siteName" as ColumnId), sum("enrolled" as ColumnId)])
    ])
  }),

  table({
    sortable: true,
    columns: [
      { id: "siteName" as ColumnId },
      { id: "investigator" as ColumnId },
      { id: "enrolled" as ColumnId },
      { id: "status" as ColumnId,
        expression: 'value === "ACTIVE" ? "✅ ACTIVE" : value' }
    ],
    lookup: lookup(sites)
  }),

  {
    datasets: [
      dataset(trialSummary, "/api/trials/{trialId}/summary"),
      dataset(sites, "/api/trials/{trialId}/sites"),
      dataset("agents" as DataSetId, "/api/trials/{trialId}/agents")
    ]
  }
);
```

### Dataset binding

Each REST endpoint maps to a `dataset("id", "/api/url")` call. Datasets with active scenario data use polling:

```typescript
dataset("adverse-events" as DataSetId, "/api/trials/{trialId}/adverse-events",
  { refreshTime: "3s" })  // 3s polling during active scenario
```

Static reference data (agents, policies) omits refresh.

### Navigation structure

The root page uses `sidebar()` for guided/explore mode:

```typescript
import { page, sidebar } from "@casehubio/pages-ui";

export const dashboard = page(
  { settings: { mode: "light" } },  // default theme
  [],
  [
    sidebar(
      { navGroupId: "main-nav", width: "280px" },
      // Guided mode pages
      step1Overview,
      step2Agents,
      step3Deviation,
      step4PiAuthorisation,
      step5AeEvent,
      step6Governance,
      step7Resolution,
      step8Proof,
      // Explore mode pages (tree navigation for hierarchy)
      trialDashboard,
      adverseEvents,
      deviations,
      auditTrail,
      trustNetwork,
      siteDetail
    )
  ]
);
```

---

## Navigation Structure

Two modes accessible via sidebar navigation:

### Guided Mode (default)

Eight narrative steps. Each combines a markdown panel (context) with live dashboard components (data). The narrative is in two acts:

**Act I — Accountability (Steps 3-4):** PI authorisation and commitment lifecycle — the qhorus story.
**Act II — AI Governance (Steps 5-7):** Trust-weighted routing, oversight gates, attestation — the engine + ledger story.

1. **Trial Overview** — introduce the trial, show enrollment across 3 sites
2. **Meet the AI Agents** — trust scores, governance model, oversight policy
3. **Event: Protocol Deviation** — report a CRITICAL deviation, PI receives formal COMMAND with deadline
4. **PI Authorisation & Commitment** — PI responds, commitment lifecycle completes, escalation path
5. **Event: Grade 4 AE Reported** — trigger live action, watch SLA, SUSAR gate, trust routing fire
6. **AI Decision & Governance** — hero layout: "what the AI decided" vs "how the platform governed it"
7. **Resolution & Trust Update** — investigator approval, attestation, Bayesian trust score recomputation
8. **The Proof** — Merkle verification, ledger entries, compliance supplements, PROV-O export

### Explore Mode

Six operational dashboard pages, priority-ordered:

**Must have:**
- Trial Dashboard — metrics, enrollment chart, recent activity
- Adverse Events — all AEs with SLA status, grade, escalation
- Audit Trail — ledger entries with type filter, Merkle verification

**Should have:**
- Protocol Deviations — all deviations with PI approval status, commitment state
- Trust Network — agent trust scores, capabilities, decision history

**Could defer:**
- Site Detail — per-site enrollment, patients, events (most operational, least demo-relevant)

---

## Page Designs — Guided Mode

### Step 1: Trial Overview

**Narrative:** "This is ONCO-2024-001, a Phase III oncology trial across 3 sites. CaseHub coordinates AI agents for eligibility screening, safety monitoring, and protocol review — each governed by trust scores, oversight gates, and a tamper-evident audit trail."

**Components:**
- Metric cards row: trial phase, total enrolled, active AEs, protocol deviations
- Bar chart: enrollment by site (target vs actual)
- Table: sites with status, investigator, enrolled count, open AE count
- Metric card: "AI Agents Active" with count + capability breakdown

### Step 2: Meet the AI Agents

**Narrative:** "CaseHub doesn't just run AI agents — it governs them. Each agent has a trust score built from its track record. High-stakes decisions are gated: no autonomous action on safety events. The platform selects agents by trust, gates their decisions, and records attestations that feed back into trust scores."

**Components:**
- Agent table: capability, trust score (0–1.0), trust dimension, maturity phase, total decisions, endorsement ratio
- Trust routing policy table: capability → minimum threshold → below-threshold behaviour
- Metric cards: "Gated Action Types: 5/5", "Trust Dimensions: 3", "Oversight Policy: No autonomous safety decisions"

### Step 3: Event — Protocol Deviation Reported

**Narrative:** "A CRITICAL protocol deviation is reported at Site B. Watch what happens: the platform sends a formal COMMAND to the named Principal Investigator — not a notification, an obligation. A Commitment is created with a deadline. If the PI doesn't respond, the platform escalates automatically. This is qhorus — formal accountability that no LLM pipeline can provide."

**Components:**
- Action button (html/onclick workaround): "Report Protocol Deviation" → POST `/trials/{trialId}/sites/{siteId}/deviations` with CRITICAL severity payload
- After trigger (3s polling):
  - Deviation detail card: type, severity, PI commanded, response deadline
  - Status: COMMANDED with PI identity and deadline timestamp
  - Channel card: shows the per-deviation `pi-oversight` channel created
  - Ledger entry: COMMAND entry written with Merkle hash

### Step 4: PI Authorisation & Commitment

**Narrative:** "The PI receives a formal COMMAND with a 24-hour deadline. When they approve, the Commitment lifecycle closes. But this is a CRITICAL deviation — the policy requires IRB committee review. The case suspends in WAITING until the ethics committee decides within 72 hours. Every step — COMMAND, response, escalation — is recorded in the Merkle audit trail."

**Components:**
- Action button: "Approve as PI" → POST `/demo/deviations/{deviationId}/approve-pi` (demo endpoint, see below)
- After trigger (3s polling):
  - Commitment lifecycle display: COMMANDED → APPROVED → ESCALATED (to IRB)
  - IRB gate card: "IRB consultation required — 72h deadline" with committee ID
  - Deviation Merkle chain: COMMAND entry → resolution entry → IRB entry (3 linked ledger entries)
- Narrative callout: "The PI's obligation was created, tracked, and discharged — all with named actors, deadlines, and a tamper-evident record."

### Step 5: Event — Grade 4 AE Reported

**Narrative:** "A Grade 4 hepatotoxicity event is reported at Site B. Watch what happens: the engine creates a 24-hour SLA work item, triggers SUSAR evaluation, and routes to a trust-weighted safety agent — all within seconds."

**Components:**
- Action button: "Report Adverse Event" → POST `/trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events` with Grade 4 hepatotoxicity payload
- After trigger (3s polling):
  - AE detail card: grade, type, reported time, SLA deadline
  - Status transition: REPORTED → ESCALATED
  - Alert (html workaround): "24h SLA activated"
  - Processing indicator (html workaround): shows while engine case is in-flight, clears when expected state detected. Timeout warning after 15s.

### Step 6: AI Decision & Governance

**Narrative:** "The SUSAR evaluator assessed this event: Grade 4 + unexpected + suspected = SUSAR criteria met. But the agent can't act alone — CaseHub's ActionRiskClassifier unconditionally gates all safety decisions. A qualified investigator must approve."

**Components (hero layout — side-by-side columns):**
- Left column: "What the AI decided" — decision breakdown (input criteria → evaluator output → SUSAR criteria met)
- Right column: "How the platform governed it" — trust context (selected agent, trust score, safety-accuracy dimension), gate status (PENDING, regulatory citation: ICH E2A §I.A.1)

### Step 7: Resolution & Trust Update

**Narrative:** "The investigator approves the SUSAR determination. CaseHub records the attestation — ENDORSED — which feeds into the agent's Bayesian trust score. Good decisions build trust; bad decisions erode it. The regulatory submission work item is created automatically."

**Components:**
- Action button: "Approve SUSAR Determination" → POST `/demo/adverse-events/{aeId}/approve-susar-gate` (demo endpoint, see below)
- After trigger (3s polling):
  - Gate decision: APPROVED with investigator ID and timestamp
  - Attestation card: ENDORSED → safety-accuracy trust dimension
  - Trust score before/after (two metric cards) — the demo endpoint triggers `TrustScoreJob.runComputation()` after writing the attestation, so the delta is real
  - Regulatory submission status: IND report created with deadline
  - Sponsor notification confirmation

### Step 8: The Proof

**Narrative:** "Every decision is recorded in a tamper-evident Merkle audit trail. Each ledger entry is independently verifiable — no trust required in the platform itself. This is what FDA auditors and EU AI Act compliance officers need."

**Components:**
- Ledger entries table: timestamp, type, actor, summary — spanning both the AE chain and the deviation chain
- Merkle verification button → calls GET `/ledger/verify` → shows VERIFIED result (works for both pre-seeded and live-action entries because the seeder uses real service calls)
- Inclusion proof detail on row click (hash chain display)
- EU AI Act Art.12 compliance supplement display
- PROV-O export link

---

## Page Designs — Explore Mode

### Trial Dashboard (must have)

Metric cards (phase, enrollment, AE count, deviation count), enrollment bar chart by site, recent activity table (last 10 events across all types).

### Adverse Events (must have)

All AEs across the trial. Table with grade, type, site, patient, reported time, SLA deadline, escalation status, regulatory submission status. Cell-level styling for overdue SLAs (workaround for row-level styling — casehub-pages#40). Sortable by deadline urgency.

### Audit Trail (must have)

Ledger entries table across the full trial. Selector dropdown to filter by entry type (AE, deviation, screening, SUSAR, attestation, compliance). Each row: timestamp, actor, type, summary. Click → Merkle proof detail. "Verify Chain Integrity" button at top.

### Protocol Deviations (should have)

All deviations. Table with type, severity, site, PI approval status, commitment lifecycle state. Status formatted via column expressions.

### Trust Network (should have)

Agent trust overview. Table of agents with capability, trust score, decision count, endorsed/challenged ratio. Metric cards for aggregate trust health. Upgrades to graph visualisation when casehub-pages#41 ships.

### Site Detail (could defer)

Per-site view. Enrollment table with patient status, site-level AE count, deviation count, investigator details. Cross-filter: click patient row → filters AE and deviation data below.

---

## Demo-Specific Endpoints

Two endpoints bridge the UI to the engine's internal mechanics. They abstract plumbing (channel IDs, gate IDs) so the UI doesn't need to know internal identifiers. Both use real platform mechanics — they are not simulations.

### `POST /demo/deviations/{deviationId}/approve-pi`

**Purpose:** Simulates PI approval via qhorus channel, triggering the real `MessageReceivedEvent` CDI chain.

**Implementation:**
1. Load `ProtocolDeviation` by `deviationId`
2. Validate `piApprovalStatus == COMMANDED` (400 if no pending command)
3. Get channel name from `deviation.piCommandChannelName`
4. Call `channelService.receiveHumanMessage(channelId, senderId="demo-pi", type=DONE, content={"decision":"APPROVED"})` — fires real `MessageReceivedEvent` → `PiResponseListener` → `ProtocolDeviationResolvedEvent` → IRB escalation if applicable
5. Return deviation status after processing

**Error handling:** 404 if deviation not found, 409 if already resolved.

### `POST /demo/adverse-events/{aeId}/approve-susar-gate`

**Purpose:** Approves the pending SUSAR oversight gate, triggering the real gate lifecycle and trust score recomputation.

**Implementation:**
1. Load `AdverseEvent` by `aeId`
2. Validate `ae.susarOversightCaseId != null` and `ae.susarOversightStatus == REQUESTED` (400/409 otherwise)
3. Find the pending `ActionGate` via `CaseInstanceRepository.findByUuid(ae.susarOversightCaseId, tenantId)` and inspect the case's pending action gate
4. Call `ActionGateService.approve(gateId, "demo-investigator")` — fires real `ActionGateApprovedEvent` → `SusarGateDecisionListener` writes ledger entry → `SusarAgentAttestationWriter` writes attestation
5. After attestation is written, call `TrustScoreJob.runComputation()` to recompute Bayesian Beta scores immediately (instead of waiting for 24h cron)
6. Return gate decision + attestation verdict + trust score delta (before/after)

**Trust score recomputation:** `TrustScoreJob.runComputation()` is a public `@Transactional` method on the CDI bean. The demo endpoint calls it directly after attestation write. This is the same computation the 24h cron job performs — triggered on-demand so the trust score delta is visible immediately. The spec documents this explicitly because the batch nature of trust score computation would otherwise make the "before/after" metric cards show identical values.

**Error handling:** 404 if AE not found, 400 if no SUSAR oversight case exists, 409 if gate already resolved.

### Profile guard

Both endpoints are annotated `@Path("/demo/...")` and guarded by `%dev` profile activation. They do not exist in production builds:

```java
@Path("/demo")
@io.quarkus.arc.profile.IfBuildProfile("dev")
public class DemoActionResource { ... }
```

---

## Authentication in Dev Mode

### Problem

All entity queries call `principal.tenancyId()` for tenant-scoped access. In dev mode, `auth.enabled-in-dev-mode=false` disables `@RolesAllowed` enforcement, but `OidcCurrentPrincipal @Alternative @Priority(100)` still wins CDI resolution. With OIDC disabled, there is no JWT — `tenancyId()` will return null or throw, breaking every query.

### Solution

A dev-profile `CurrentPrincipal` implementation:

```java
@ApplicationScoped
@Alternative
@Priority(150)  // above OIDC (100), below test (200)
@io.quarkus.arc.profile.IfBuildProfile("dev")
public class DemoCurrentPrincipal implements CurrentPrincipal {
    @Override public String tenancyId() { return "demo-tenant"; }
    @Override public String actorId()   { return "demo-user"; }
    // other methods return sensible defaults
}
```

Activated via:
```properties
%dev.quarkus.arc.selected-alternatives=io.casehub.clinical.demo.DemoCurrentPrincipal
```

The `DemoDataSeeder` stamps all entities with `tenantId = "demo-tenant"` to match.

---

## Data Architecture

### New REST Endpoints (TrialDashboardResource)

Read-only aggregation/summary endpoints with trial-level auth. These are **not replacements** for the existing ownership-chain-validated entity endpoints — they are a different access pattern for dashboard views. The existing endpoints validate the full trial→site→patient→AE ownership chain per entity. These endpoints validate only trial-level access and return flattened, pre-joined data.

| Endpoint | Returns |
|----------|---------|
| `GET /trials/{trialId}/summary` | Enrollment counts per site, AE count by grade, deviation/amendment counts, agent stats |
| `GET /trials/{trialId}/agents` | Agent list: capability, trust score, dimension, phase, decisions, endorsement ratio |
| `GET /trials/{trialId}/patients` | All patients across all sites (flattened with site context) |
| `GET /trials/{trialId}/adverse-events` | All AEs (flattened, includes computed slaTimeRemaining) |
| `GET /trials/{trialId}/deviations` | All deviations (flattened with site context) |
| `GET /trials/{trialId}/ledger-entries` | Paginated ledger entries with `?type=` filter |

All carry `@RolesAllowed` consistent with existing resources — even though dev mode disables enforcement, the annotations maintain production consistency.

Response types are **nested records inside `TrialDashboardResource`** — matching the existing pattern in clinical where request/response types are nested in resource classes. No standalone DTO package.

### Datasets

Each endpoint maps to a casehub-pages `dataset()` call:

```typescript
// Active scenario data — 3s polling
dataset("adverse-events" as DataSetId, "/api/trials/{trialId}/adverse-events", { refreshTime: "3s" })

// Static reference data — no polling
dataset("agents" as DataSetId, "/api/trials/{trialId}/agents")

// Idle dashboard — 30s polling
dataset("trial-summary" as DataSetId, "/api/trials/{trialId}/summary", { refreshTime: "30s" })
```

**Note on URL parameters:** The pre-seeded trial has a fixed ID known at startup. Dataset URLs use the concrete trial UUID, not a template variable. Dynamic URL templating (casehub-pages#49) is not needed for the demo but would be needed for a multi-tenant production dashboard.

### Pre-seeded Data (DemoDataSeeder)

**Approach:** The seeder calls **real service methods** — `AdverseEventService.reportAdverseEvent()`, `ProtocolDeviationService.reportDeviation()`, etc. — not direct entity inserts. This is critical because:

1. Ledger entries must pass Merkle verification. `LedgerEntryRepository.save()` computes digests and updates the Merkle frontier automatically. Direct entity inserts bypass this pipeline — the verification endpoint in Step 8 would fail.
2. Async engine case processing (SUSAR oversight, AE escalation, IRB gate) produces the ledger entries and state transitions that the demo displays. Bypassing the service layer would require replicating this logic.

**Trade-off:** The seeder replays a scenario through the real service layer, which involves async CDI observers and engine case processing. This means:
- Startup takes 5-10 seconds longer than direct inserts (engine cases process asynchronously)
- The seeder uses polling (same pattern as `ThreeSiteShowcaseTest` with `Awaitility`) to wait for async processing to complete before proceeding
- The seeder verifies Merkle chain integrity for each subject after seeding, using `LedgerVerificationService.verify()` — if verification fails, the seeder throws and startup aborts

**`DemoDataSeeder`** is a CDI bean observing `StartupEvent`, guarded by `casehub.clinical.demo.seed-data` config property (true in dev profile only).

**Trial:** ONCO-2024-001, Phase III oncology, sponsor "Meridian Therapeutics"

| Site | Name | Patients | Pre-seeded events |
|------|------|----------|-------------------|
| Site A | Memorial Cancer Center | 4 | 1 completed eligibility screening (CRITERIA_MET), 1 resolved Grade 2 AE with full Merkle-verified ledger trail |
| Site B | Johns Hopkins Oncology | 3 | 1 resolved protocol deviation (PI approved via `receiveHumanMessage`, commitment lifecycle complete, ledger entries verified) |
| Site C | Mayo Clinic Research | 3 | 1 completed protocol amendment (PROCEED, ledger entries verified) |

Pre-populated trust score history: after seeding the resolved AE (Site A), the seeder calls `TrustScoreJob.runComputation()` to materialise initial trust scores from the attestations written during AE processing. This gives the "Meet the AI Agents" page real trust score data on first load.

### Live Actions

| Step | Trigger | API call | Engine reaction |
|------|---------|----------|----------------|
| Step 3 | "Report Protocol Deviation" | `POST /trials/{trialId}/sites/{siteId}/deviations` | PI COMMAND via qhorus channel, Commitment created with deadline, ledger entry |
| Step 4 | "Approve as PI" | `POST /demo/deviations/{deviationId}/approve-pi` | `receiveHumanMessage` → `PiResponseListener` → commitment closed → IRB escalation if CRITICAL |
| Step 5 | "Report Adverse Event" | `POST /trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/adverse-events` | 24h SLA WorkItem, SUSAR oversight case, trust-weighted agent selection |
| Step 7 | "Approve SUSAR Determination" | `POST /demo/adverse-events/{aeId}/approve-susar-gate` | Gate APPROVED, attestation written, `TrustScoreJob.runComputation()` triggered, IND report created |

---

## Polling and Async UX

### Between action and feedback

After a POST action:
1. The action button disables and shows a loading state
2. An html() component renders a "processing" indicator (e.g., "Engine processing — SUSAR evaluation in progress...")
3. Dashboard components poll at 3s intervals
4. When the expected state transition is detected (e.g., `escalationStatus` changes from `REPORTED` to `ESCALATED`), the processing indicator clears
5. If no change after 15s, the indicator changes to a warning: "Processing is taking longer than expected"
6. Once the expected state is reached, polling drops to idle rate (30s) to avoid unnecessary requests

### Implementation

casehub-pages `refresh: { interval: 3000 }` handles the polling. State detection is done via dataset content — the narrative page checks whether the expected value appears in the refreshed dataset. The processing indicator uses an html() component with JavaScript that reads from the same dataset.

---

## Error States

| Scenario | UI behaviour |
|----------|-------------|
| REST endpoint returns 5xx | Metric cards show "—"; tables show "Error loading data" placeholder; no crash |
| Dataset returns empty array | Tables show "No data available" message; charts show empty state |
| Polling timeout (15s) | Processing indicator changes to warning; does not block navigation |
| Action button POST fails | Button re-enables; error message shown inline below button |

These are handled via casehub-pages' existing error handling where available, and html() components with JavaScript for custom error states.

---

## Project Structure

### UI (new)

```
runtime/src/main/webui/
├── package.json              # @casehubio/pages-runtime + pages-ui deps
├── tsconfig.json             # ES2020 target, ESNext modules, bundler resolution
├── esbuild.config.mjs        # Single entry point → dist/app.js
├── index.html                # Shell: <div id="app"></div> + <script src="dist/app.js">
├── src/
│   ├── index.ts              # Entry: loadSite(container, dashboard)
│   ├── dashboard.ts          # Root page: sidebar with guided + explore mode pages
│   ├── guided/
│   │   ├── step1-overview.ts       # page() composition with markdown + metrics + chart + table
│   │   ├── step2-agents.ts         # page() with agent table + trust policy table
│   │   ├── step3-deviation.ts      # page() with action button + deviation detail + channel card
│   │   ├── step4-pi-auth.ts        # page() with action button + commitment lifecycle + IRB gate
│   │   ├── step5-ae-event.ts       # page() with action button + AE detail + SLA
│   │   ├── step6-governance.ts     # page() with columns (hero layout: AI vs governance)
│   │   ├── step7-resolution.ts     # page() with action button + attestation + trust delta
│   │   └── step8-proof.ts          # page() with ledger table + Merkle verification
│   ├── explore/
│   │   ├── trial-dashboard.ts      # page() with metrics + enrollment chart + activity table
│   │   ├── adverse-events.ts       # page() with AE table (sortable, filterable, SLA styling)
│   │   ├── audit-trail.ts          # page() with ledger table + type selector + verify button
│   │   ├── deviations.ts           # page() with deviation table + PI status
│   │   ├── trust-network.ts        # page() with agent table + trust metrics
│   │   └── site-detail.ts          # page() with per-site patient table + cross-filter
│   ├── datasets.ts           # All dataset() definitions with URL bindings + refresh config
│   ├── lookups.ts            # Reusable lookup() definitions (groupBy, filterBy combos)
│   ├── theme.ts              # Wrapper around casehub-pages setTheme() — balanced professional + dark mode config via CSS custom properties
│   └── narrative.ts          # Markdown text content for all 8 guided mode panels — centralised for easy editing
```

### Java (new)

```
runtime/src/main/java/io/casehub/clinical/
├── resource/
│   └── TrialDashboardResource.java    # 6 list/summary endpoints; response records nested inside
├── demo/
│   ├── DemoActionResource.java        # /demo/... endpoints (IfBuildProfile("dev"))
│   ├── DemoCurrentPrincipal.java      # CurrentPrincipal for dev mode (IfBuildProfile("dev"))
│   └── DemoDataSeeder.java            # @Observes StartupEvent, config-guarded, calls real services
```

### Maven

Add `quarkus-quinoa` extension to `runtime/pom.xml`.

### Configuration

```properties
# Quinoa
quarkus.quinoa.dev-server.port=3000
quarkus.quinoa.build-dir=dist

# Demo data (dev profile only)
%dev.casehub.clinical.demo.seed-data=true
casehub.clinical.demo.seed-data=false

# Dev-profile CurrentPrincipal
%dev.quarkus.arc.selected-alternatives=io.casehub.clinical.demo.DemoCurrentPrincipal
```

---

## casehub-pages Improvements

Tracked in casehubio/casehub-pages#50. All have workarounds — demo is not blocked.

### New components

| Issue | Feature | Demo workaround |
|-------|---------|----------------|
| #37 | Timeline / Gantt chart | Timeseries chart + timestamp table |
| #38 | Alert / notification banner | html() component with conditional content |
| #39 | Status badge / tag | Column expressions with emoji/text formatting |
| #41 | Graph / network visualisation | Agent table with trust scores |
| #43 | Countdown / deadline metric | Server-side computed string, polling refresh |
| #46 | Action button (POST + refresh) | html() with JS onclick |

### Table enhancements

| Issue | Feature | Demo workaround |
|-------|---------|----------------|
| #40 | Row-level conditional styling | Cell-level expressions on every column |
| #42 | Expandable / drill-down rows | Cross-filter + detail panel below |

### Data & rendering enhancements

| Issue | Feature | Demo workaround |
|-------|---------|----------------|
| #47 | Conditional component visibility | Stack switching or html() with JS |
| #48 | Dynamic content in markdown | html() with JS |
| #49 | Parameterised dataset URLs | Static URLs (single pre-seeded trial) |

**Priority for demo polish:** #39 (badges), #43 (countdown), #46 (action button), #38 (alerts)

---

## Testing Strategy

### Java tests

- **TrialDashboardResource** — `@QuarkusTest` integration tests for all 6 endpoints. Pre-create entities in `@BeforeEach`, verify response shape, filtering, pagination.
- **DemoActionResource** — `@QuarkusTest` integration tests for both demo endpoints. Verify gate approval fires real lifecycle, trust score recomputation produces delta.
- **DemoDataSeeder** — integration test: verify entity counts, Merkle chain integrity (`ledgerVerificationService.verify()` passes for all seeded subjects), config guard (`seed-data=false` → no seeding).
- **DemoCurrentPrincipal** — unit test verifying fixed tenant/actor values.

### Automated smoke tests (Playwright)

Location: `runtime/src/test/playwright/`

Runs separately from Maven lifecycle — not in `mvn test`. Requires Quarkus dev server running (`mvn quarkus:dev`).

1. **Page reachability** — all 8 guided steps + 6 explore pages render without JS console errors
2. **Data binding** — tables have rows, metrics show values (not NaN/blank), charts render (canvas/SVG with non-zero dimensions)
3. **No clipping** — key components: `scrollWidth <= clientWidth` and `scrollHeight <= clientHeight` at 1440x900 and 1920x1080
4. **Live action flow** — click "Report Adverse Event" → new AE row appears in table, status transitions within 15s timeout
5. **Navigation integrity** — guided/explore mode switching, all sidebar links work, no dead pages

### Manual verification

- Theme: light and dark modes
- Polling: trigger live action, watch updates at 3s intervals
- Full guided walkthrough end-to-end (all 8 steps)

---

## Issue Tracking

### casehubio/clinical#93 — Demo UI epic

Issues to be updated with the revised scope (8 guided steps, 2 demo endpoints, DemoCurrentPrincipal, revised DemoDataSeeder).

| # | Description | Scale | Complexity | Depends on |
|---|-------------|-------|------------|------------|
| #94 | Quinoa setup + webui scaffold | S | Low | — |
| #95 | DemoDataSeeder (service-layer seeding with Merkle verification) | M | High | — |
| #96 | TrialDashboardResource (6 endpoints) | M | Med | — |
| NEW | DemoActionResource (2 demo endpoints) + DemoCurrentPrincipal | M | High | — |
| #97 | Datasets + lookups module | S | Low | #96 |
| #98 | Guided Steps 1-2 | M | Med | #95, #97 |
| NEW | Guided Steps 3-4 (deviation + PI auth) | M | High | #98 |
| #99 | Guided Steps 5-6 (AE + governance) | M | High | Steps 3-4 |
| #100 | Guided Steps 7-8 (resolution + proof) | M | Med | #99 |
| #101 | Explore mode (6 pages, priority-ordered) | M | Med | #97 |
| #102 | Playwright smoke tests | S | Med | all UI issues |

**DemoDataSeeder complexity revised to High:** It replays a scenario through the real service layer with async processing, polling for completion, and Merkle chain verification. This is more complex than the original spec acknowledged.

### Dependency graph

```
#94 (Quinoa setup) ──────────────────────────────────┐
#95 (DemoDataSeeder) ──────────────────────────────┐ │
#96 (REST endpoints) ──→ #97 (Datasets) ─────────┐ │ │
NEW (Demo endpoints + DemoCurrentPrincipal) ─────┐ │ │ │
                                                  ↓ ↓ ↓ ↓
                                           #98 (Steps 1-2)
                                                  ↓
                                           NEW (Steps 3-4)
                                                  ↓
                                           #99 (Steps 5-6)
                                                  ↓
                                           #100 (Steps 7-8)
                                                  ↓
                            #101 (Explore) ──→ #102 (Smoke tests)
```

### casehubio/casehub-pages#50 — Pages capabilities epic

11 child issues (#37-#43, #46-#49). Non-blocking — each improves the demo incrementally as it ships.
