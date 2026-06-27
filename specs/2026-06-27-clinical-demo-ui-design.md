# Clinical Trial Demo UI — Design Spec

**Date:** 2026-06-27 (revision 6 — final)
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

Each guided step and explore page is a `page()` call returning a DSL component tree.

**Function signatures** (from `@casehubio/pages-ui`):
- `page(name: string, ...args: (Component | PageOptions)[])` — root container
- `columns(distribution: number[], ...slotContents: Component[][])` — grid layout; distribution array length must match slot count
- `sidebar(...entries: [string, ...Component[]][])` — nav sidebar; same pattern for `tabs`, `pills`, `menu`, `tree`
- `lookup(dataSetId: string, ...ops: DataSetOp[])` — dataset query with chained operations
- `groupBy(source: string | null, ...resultColumns: ResultColumn[])` — aggregation; `null` source for ungrouped aggregates
- `count(source)`, `sum(source)`, `avg(source)`, `col(source)` — single-parameter aggregation functions
- `filterBy(columnId, operator, value)` — filter operation
- `sortBy(columnId, direction)` — sort operation

```typescript
// Example: Step 1 — Trial Overview
import { page, columns, metric, barChart, table, markdown,
         dataset, lookup, groupBy, col, count, sum } from "@casehubio/pages-ui";

export const step1Overview = page("1. Trial Overview",
  markdown(`## ONCO-2024-001 — Phase III Oncology Trial
CaseHub coordinates AI agents for eligibility screening, safety monitoring,
and protocol review — each governed by trust scores, oversight gates,
and a tamper-evident audit trail.`),

  // 4-column metrics row: distribution [3,3,3,3], one component array per slot
  columns([3, 3, 3, 3],
    [metric({ title: "Trial Phase", lookup: lookup("trial-summary") })],
    [metric({ title: "Total Enrolled", lookup: lookup("trial-summary", groupBy(null, sum("enrolled"))) })],
    [metric({ title: "Active AEs", lookup: lookup("trial-summary", groupBy(null, sum("activeAeCount"))) })],
    [metric({ title: "AI Agents Active", lookup: lookup("agents", groupBy(null, count("id"))) })]
  ),

  barChart({
    title: "Enrollment by Site",
    lookup: lookup("sites", groupBy("siteName", col("siteName"), sum("enrolled")))
  }),

  table({
    sortable: true,
    columns: [
      { id: "siteName" },
      { id: "investigator" },
      { id: "enrolled" },
      { id: "status", expression: 'value === "ACTIVE" ? "✅ ACTIVE" : value' }
    ],
    lookup: lookup("sites")
  }),

  {
    datasets: [
      dataset("trial-summary", `/api/trials/${TRIAL_ID}/summary`),
      dataset("sites", `/api/trials/${TRIAL_ID}/sites`),
      dataset("agents", `/api/trials/${TRIAL_ID}/agents`)
    ]
  }
);
```

**Trial ID constant:** The seeder uses a deterministic UUID: `UUID.nameUUIDFromBytes("ONCO-2024-001".getBytes(StandardCharsets.UTF_8))`. The same value is hardcoded in `datasets.ts`:

```typescript
// Deterministic UUID matching DemoDataSeeder.TRIAL_ID
export const TRIAL_ID = "316e3846-4ea7-3b18-a6f7-e01ce6582a69";
```

### Dataset binding

Each REST endpoint maps to a `dataset()` call using the `TRIAL_ID` constant from `datasets.ts`. Datasets with active scenario data use polling:

```typescript
import { TRIAL_ID } from "./datasets";

dataset("adverse-events", `/api/trials/${TRIAL_ID}/adverse-events`,
  { refreshTime: "3s" })  // 3s polling during active scenario
```

Static reference data (agents, policies) omits refresh.

### Navigation structure

The root page uses `tree()` for navigation. `tree()` supports "/" path separators for nested groups — `sidebar()` does not (it renders flat buttons with literal text). All nav components share the same DSL signature: `(...entries: [string, ...Component[]][])`.

```typescript
import { page, tree } from "@casehubio/pages-ui";

export const dashboard = page("CaseHub Clinical",
  tree(
    // Act I — Accountability
    ["Guided/1. Trial Overview", step1Overview],
    ["Guided/2. Meet the AI Agents", step2Agents],
    ["Guided/3. Protocol Deviation", step3Deviation],
    ["Guided/4. PI Authorisation", step4PiAuthorisation],
    // Act II — AI Governance
    ["Guided/5. Grade 4 AE Reported", step5AeEvent],
    ["Guided/6. AI Decision & Governance", step6Governance],
    ["Guided/7. Resolution & Trust", step7Resolution],
    ["Guided/8. The Proof", step8Proof],
    // Explore mode
    ["Explore/Trial Dashboard", trialDashboard],
    ["Explore/Adverse Events", adverseEvents],
    ["Explore/Protocol Deviations", deviations],
    ["Explore/Audit Trail", auditTrail],
    ["Explore/Trust Network", trustNetwork],
    ["Explore/Site Detail", siteDetail]
  ),
  { settings: { mode: "light" } }
);
```

`tree()` parses "/" separators via `buildTreeStructure()` in pages-component, rendering a hierarchical sidebar with collapsible groups. The "Guided" group auto-expands on first load; "Explore" collapses to keep focus on the narrative.

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
  - DSMB rollup callout: the seeded Grade 4 AEs at Site A left `grade4Active` flags set on the trial case blackboard. When this new Grade 4 AE fires at Site B, the trial case now sees ≥2 sites with simultaneous Grade 4+ signals — the DSMB rollup binding fires automatically. The narrative highlights this: "Notice: the platform detected a cross-site safety pattern. Two sites now have active Grade 4+ events — a DSMB review has been triggered automatically, with no site-level agent having global visibility."

**DSMB rollup is a feature, not noise.** The seeder intentionally does NOT complete the AE escalation WorkItems for the seeded Grade 4 AEs — it completes only the SUSAR oversight gates (for attestation/trust score seeding). The `grade4Active` flags remain set, priming the trial case for the DSMB rollup when Step 5 adds a second site. This turns a Layer 6 capability into a live demo moment.

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

**Purpose:** Triggers PI approval via qhorus `ChannelGateway`, firing the real `MessageReceivedEvent` CDI chain.

**Implementation:**
1. Load `ProtocolDeviation` by `deviationId`
2. Validate `piApprovalStatus == COMMANDED` (400 if no pending command)
3. Look up channel: `channelService.findByName(deviation.piCommandChannelName)` → get channel UUID
4. Call `channelGateway.receiveHumanMessage()` with the correct record types:
   ```java
   channelGateway.receiveHumanMessage(
       new ChannelRef(channel.id, channel.name),
       new InboundHumanMessage(
           "demo-pi",                              // externalSenderId
           "{\"decision\":\"APPROVED\"}",           // content (JSON)
           Instant.now(),                           // receivedAt
           Map.of(),                                // metadata
           deviationId.toString(),                  // correlationId
           null                                     // inReplyTo
       )
   );
   ```
5. This fires the real chain: `ClinicalInboundNormaliser` maps JSON to DONE → `MessageReceivedEvent` CDI async → `PiResponseListener.onMessage()` → status update → `ProtocolDeviationResolvedEvent` → IRB escalation if CRITICAL
6. Return deviation status after processing

**Note:** `ChannelGateway` (not `ChannelService`) owns `receiveHumanMessage()`. `ChannelRef` and `InboundHumanMessage` are records from `io.casehub.qhorus.api.gateway`. This matches the pattern in `PiResponseListenerIntegrationTest`.

**Error handling:** 404 if deviation not found, 409 if already resolved.

### `POST /demo/adverse-events/{aeId}/approve-susar-gate`

**Purpose:** Approves the pending SUSAR oversight gate by completing its WorkItem, triggering the real gate lifecycle and trust score recomputation.

**Mechanism:** There is no `ActionGateService.approve()` — gates are approved by completing WorkItems. When the engine schedules an action gate, `ActionGateWorkItemHandler` creates a WorkItem with `callerRef = "case:{caseId}/gate:{gateId}"` (via `GateCallerRef.encode()`). Completing that WorkItem triggers `ActionGateCompletionApplier`, which publishes `ActionGateApprovedEvent`. The demo endpoint abstracts this WorkItem lifecycle.

**Implementation:**
1. Load `AdverseEvent` by `aeId`
2. Validate `ae.susarOversightCaseId != null` and `ae.susarOversightStatus == REQUESTED` (400/409 otherwise)
3. Find the gate WorkItem: query active WorkItems whose `callerRef` starts with `"case:" + ae.susarOversightCaseId + "/gate:"`. Since there is at most one pending gate per SUSAR oversight case, this is unambiguous. Use `WorkItemStore` to scan for the matching callerRef prefix.
4. Claim the WorkItem: `workItemService.claim(workItem.id, "demo-investigator")` — moves PENDING → IN_PROGRESS (required before completion)
5. Complete the WorkItem: `workItemService.complete(workItem.id, "demo-investigator", "{\"decision\":\"APPROVED\"}", "APPROVED")` — this triggers the real chain:
   - `ActionGateCompletionApplier` detects the gate callerRef and publishes `ActionGateApprovedEvent`
   - `SusarGateDecisionListener` consumes the event and writes the SUSAR decision ledger entry
   - `SusarAgentAttestationWriter` consumes the event and writes the `LedgerAttestation` (ENDORSED)
6. After attestation is written, call `TrustScoreJob.runComputation()` to recompute Bayesian Beta scores immediately (instead of waiting for 24h cron)
7. Return gate decision + attestation verdict + trust score delta (before/after)

**WorkItem lookup strategy:** `WorkItemService.findActiveByCallerRef()` takes an exact string match. The demo endpoint doesn't know the `gateId` (it's an `EventLog` entry ID internal to the engine). Options:
- Add a `findActiveByCallerRefPrefix(String prefix)` method to `WorkItemStore` — returns the single active WorkItem matching the prefix. This is a clean extension.
- Or scan via `WorkItemStore` directly using the stream API to filter by callerRef prefix.

The first option is cleaner — file an issue on casehub-work if needed.

**Trust score recomputation:** `TrustScoreJob.runComputation()` is a public `@Transactional` method on the CDI bean. The demo endpoint calls it directly after the gate approval events have been processed. This is the same computation the 24h cron job performs — triggered on-demand so the trust score delta is visible immediately. Without this, the "before/after" metric cards would show identical values because trust scores are batch-computed.

**Error handling:** 404 if AE not found, 400 if no SUSAR oversight case exists, 409 if gate already resolved, 409 if no active gate WorkItem found.

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

**No `selected-alternatives` entry needed.** `DemoCurrentPrincipal` lives in clinical's `runtime` module (Jandex-indexed automatically). `@Alternative @Priority(150)` beats `OidcCurrentPrincipal @Alternative @Priority(100)` via standard ArC priority resolution — the same mechanism by which `OidcCurrentPrincipal` itself beats `MockCurrentPrincipal @DefaultBean` in production without being listed in `selected-alternatives`. `@IfBuildProfile("dev")` excludes the bean entirely from non-dev builds, eliminating any ambiguity.

**Do NOT add `%dev.quarkus.arc.selected-alternatives`.** Quarkus profile-specific properties completely replace their non-profiled counterparts. The existing `quarkus.arc.selected-alternatives` lists `JpaLedgerEntryRepository`, `JpaPlanItemStore`, and `JpaSubCaseGroupRepository`. A `%dev.` override would drop all three, causing CDI unsatisfied dependency failures at startup. The `@IfBuildProfile + @Alternative + @Priority` pattern avoids this entirely.

The `DemoDataSeeder` stamps all entities with `tenantId = "demo-tenant"` to match.

---

## Data Architecture

### New REST Endpoints (TrialDashboardResource)

Read-only aggregation/summary endpoints with trial-level auth. These are **not replacements** for the existing ownership-chain-validated entity endpoints — they are a different access pattern for dashboard views. The existing endpoints validate the full trial→site→patient→AE ownership chain per entity. These endpoints validate only trial-level access and return flattened, pre-joined data.

| Endpoint | Returns | Data sources |
|----------|---------|-------------|
| `GET /trials/{trialId}/summary` | Enrollment counts per site, AE count by grade, deviation/amendment counts, agent stats | Panache queries on domain entities (default datasource) |
| `GET /trials/{trialId}/agents` | Agent list: capability, trust score, dimension, phase, decisions, endorsement ratio | **Cross-source aggregation:** static capabilities from `ClinicalCapabilities` + trust scores from `ActorTrustScoreRepository` (qhorus) + decision counts from `WorkerDecisionEntry` (qhorus) + attestation ratios from `LedgerAttestation` queries (qhorus). Not a simple Panache query. |
| `GET /trials/{trialId}/patients` | All patients across all sites (flattened with site context) | Panache queries (default datasource) |
| `GET /trials/{trialId}/adverse-events` | All AEs (flattened, includes computed slaTimeRemaining) | Panache queries (default datasource) |
| `GET /trials/{trialId}/adverse-events/{aeId}/governance` | SUSAR decision context for Step 6 hero layout: AE SUSAR fields + `WorkerDecisionEntry` (workerId, capabilityTag, trustScoreAtRouting, thresholdApplied) + current `ActorTrustScore` + gate status | **Cross-source aggregation:** AE entity (default) + `CaseLedgerEntryRepository.findWorkerDecisionsByCaseId(ae.susarOversightCaseId)` (qhorus) + `ActorTrustScoreRepository` (qhorus). Gate status derived from `ae.susarOversightStatus` (not ephemeral in-memory gate state). |
| `GET /trials/{trialId}/deviations` | All deviations (flattened with site context) | Panache queries (default datasource) |
| `GET /trials/{trialId}/ledger-entries` | Paginated ledger entries with `?type=` filter | **Cross-datasource query:** find all enrollment IDs + deviation IDs for the trial (default datasource), then query `LedgerEntryRepository.findBySubjectId()` for each subject (qhorus datasource). No `trialId` column on `LedgerEntry` — the join is application-level. |

All carry `@RolesAllowed` consistent with existing resources — even though dev mode disables enforcement, the annotations maintain production consistency.

Response types are **nested records inside `TrialDashboardResource`** — matching the existing pattern in clinical where request/response types are nested in resource classes. No standalone DTO package.

**Governance endpoint detail:** The `/governance` endpoint powers Step 6's hero layout. The left panel ("What the AI decided") uses AE fields (`unexpected`, `suspected`, `grade`) and `WorkerDecisionEntry` output. The right panel ("How the platform governed it") uses `WorkerDecisionEntry.trustScoreAtRouting`, `thresholdApplied`, and `ActorTrustScore` current values. The trust routing selection rationale is not persisted as a separate record, but the post-hoc capture on `WorkerDecisionEntry` shows exactly what policy was in effect and what score was observed — sufficient for the demo display.

### Datasets

Each endpoint maps to a casehub-pages `dataset()` call:

```typescript
import { TRIAL_ID } from "./datasets";

// Active scenario data — 3s polling
dataset("adverse-events", `/api/trials/${TRIAL_ID}/adverse-events`, { refreshTime: "3s" })

// Static reference data — no polling
dataset("agents", `/api/trials/${TRIAL_ID}/agents`)

// Idle dashboard — 30s polling
dataset("trial-summary", `/api/trials/${TRIAL_ID}/summary`, { refreshTime: "30s" })
```

**Trial ID resolution:** `TRIAL_ID` is a TypeScript constant in `datasets.ts` — the deterministic UUID matching `DemoDataSeeder.TRIAL_ID` in Java. Both use `UUID.nameUUIDFromBytes("ONCO-2024-001".getBytes(StandardCharsets.UTF_8))`. No dynamic URL templating (casehub-pages#49) needed; the UUID is a build-time constant shared between Java and TypeScript.

### Pre-seeded Data (DemoDataSeeder)

**Approach:** The seeder calls **real service methods** — `AdverseEventService.reportAdverseEvent()`, `ProtocolDeviationService.reportDeviation()`, etc. — not direct entity inserts. This is critical because:

1. Ledger entries must pass Merkle verification. `LedgerEntryRepository.save()` computes digests and updates the Merkle frontier automatically. Direct entity inserts bypass this pipeline — the verification endpoint in Step 8 would fail.
2. Async engine case processing (SUSAR oversight, AE escalation, IRB gate) produces the ledger entries and state transitions that the demo displays. Bypassing the service layer would require replicating this logic.

**Trade-off:** The seeder replays a scenario through the real service layer, which involves async CDI observers and engine case processing. This means:
- Startup takes 5-10 seconds longer than direct inserts (engine cases process asynchronously)
- The seeder uses polling (same pattern as `ThreeSiteShowcaseTest` with `Awaitility`) to wait for async processing to complete before proceeding
- The seeder verifies Merkle chain integrity for each subject after seeding, using `LedgerVerificationService.verify()` — if verification fails, the seeder throws and startup aborts

**`DemoDataSeeder`** is a CDI bean guarded by `casehub.clinical.demo.seed-data` config property (true in dev profile only).

**Idempotency:** Quarkus dev mode uses a persistent H2 database. If the user stops and restarts `mvn quarkus:dev`, the seeder runs again on existing data. The seeder checks `ClinicalTrial.find("protocolId", "ONCO-2024-001").firstResult() != null` at the top — if the trial exists, skip entirely. No partial re-seeding.

**Startup timing:** The seeder calls services that fire `@ObservesAsync` CDI events dispatched on Vert.x worker threads. The seeder's polling loop runs on the calling thread while Vert.x workers process async events concurrently. Quarkus initializes the Vert.x event bus and worker pools before CDI bean creation, so by `StartupEvent` time Vert.x workers are available.

**Risk:** If the Vert.x event bus proves unreliable during `StartupEvent` in practice (async observers not dispatching), switch to a `@Scheduled` one-shot approach:
```java
@Scheduled(every = "1s", delayed = "3s", identity = "demo-seeder")
void seedOnce() {
    if (seeded.compareAndSet(false, true)) { runSeed(); }
}
```
This fires 3 seconds after full startup and self-disables via `AtomicBoolean`.

**Connection pool concern:** The seeder's polling loop must NOT be `@Transactional` — each poll is a separate transaction. Holding a JDBC connection during polling while async handlers need connections from the same pool would cause Agroal pool exhaustion. This is the same constraint as the existing three-phase pattern documented in CLAUDE.md.

**Trial:** ONCO-2024-001, Phase III oncology, sponsor "Meridian Therapeutics"

| Site | Name | Patients | Pre-seeded events |
|------|------|----------|-------------------|
| Site A | Memorial Cancer Center | 4 | 1 eligibility screening (CRITERIA_MET), 1 resolved Grade 2 AE (baseline — no SUSAR gate), 2–3 resolved Grade 4 unexpected AEs with full SUSAR lifecycle (gate WorkItem claimed + completed → attestations written → trust scores materialised) |
| Site B | Johns Hopkins Oncology | 3 | 1 resolved protocol deviation (PI approved via `channelGateway.receiveHumanMessage`, commitment lifecycle complete, ledger entries verified) |
| Site C | Mayo Clinic Research | 3 | 1 completed protocol amendment (PROCEED, ledger entries verified) |

**Trust score materialisation:** Grade 2 AEs do not trigger SUSAR oversight — `SusarCriteriaEvaluator` gates on `GRADE_4`/`GRADE_5`, and `DefaultAdverseEventEscalationPolicy` routes Grade 1-2 directly with no engine case. Zero attestations means `TrustScoreJob.runComputation()` produces nothing. The 2–3 Grade 4 unexpected AEs at Site A are specifically for trust score seeding: each completes the full SUSAR gate lifecycle (report → SUSAR case starts → gate WorkItem created → claimed → completed → `SusarAgentAttestationWriter` writes ENDORSED attestation). After all are resolved, the seeder calls `TrustScoreJob.runComputation()` to materialise Bayesian Beta scores from the accumulated attestations. This gives the "Meet the AI Agents" page real trust score data on first load — agents with 2–3 decisions, measurable endorsement ratios, and non-bootstrap trust scores.

The seeder replays the same flow as live demo Steps 5–7 for each Grade 4 AE: report → wait for SUSAR case → find gate WorkItem → claim → complete → wait for attestation. This adds startup time (~5s per AE) but produces genuine trust data.

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

### Action button idempotency

Action buttons check the dataset for existing state before enabling. This handles restarts, re-runs, and double-clicks:

- "Report Protocol Deviation" (Step 3): button disabled if the deviations dataset already contains a CRITICAL deviation at Site B
- "Report Adverse Event" (Step 5): button disabled if the adverse-events dataset already contains a Grade 4 AE at Site B with the demo patient
- "Approve as PI" (Step 4): button disabled if deviation status is not COMMANDED
- "Approve SUSAR Determination" (Step 7): button disabled if AE susarOversightStatus is not REQUESTED

On navigation away and back, the button re-checks the dataset — if the action was completed during a previous visit, the button stays disabled and shows the result. This makes the demo resilient to restarts without server-side idempotency.

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
│   ├── dashboard.ts          # Root page: tree() nav with Guided + Explore groups
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
│   ├── datasets.ts           # TRIAL_ID constant + all dataset() definitions with URL bindings + refresh config
│   ├── lookups.ts            # Reusable lookup() definitions (groupBy, filterBy combos)
│   ├── theme.ts              # Wrapper around casehub-pages setTheme() — balanced professional + dark mode config via CSS custom properties
│   └── narrative.ts          # Markdown text content for all 8 guided mode panels — centralised for easy editing
```

### Java (new)

```
runtime/src/main/java/io/casehub/clinical/
├── resource/
│   └── TrialDashboardResource.java    # 7 endpoints (6 list/summary + governance); response records nested inside
├── demo/
│   ├── DemoActionResource.java        # /demo/... endpoints (IfBuildProfile("dev"))
│   ├── DemoCurrentPrincipal.java      # CurrentPrincipal for dev mode (IfBuildProfile("dev"))
│   └── DemoDataSeeder.java            # @Observes StartupEvent, config-guarded, calls real services
```

### Maven

Add `quarkus-quinoa` extension to `runtime/pom.xml`.

### Configuration

```properties
# Quinoa — esbuild (no dev server; Quinoa watches sources and re-runs npm build)
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true

# Demo data (dev profile only)
%dev.casehub.clinical.demo.seed-data=true
casehub.clinical.demo.seed-data=false

# DemoCurrentPrincipal is activated via @IfBuildProfile("dev") + @Alternative @Priority(150)
# — no selected-alternatives entry needed (see Authentication in Dev Mode section)
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

- **TrialDashboardResource** — `@QuarkusTest` integration tests for all 7 endpoints. Pre-create entities in `@BeforeEach`, verify response shape, filtering, pagination.
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
| #96 | TrialDashboardResource (7 endpoints incl. governance) | M | Med | — |
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
