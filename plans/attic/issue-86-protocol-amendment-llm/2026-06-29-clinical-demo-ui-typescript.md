# Clinical Demo UI — TypeScript Implementation Plan (Plan 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build 14 casehub-pages DSL page compositions (8 guided walkthrough + 6 explore dashboard) wired to the backend endpoints from Plan 1, rendered via Quinoa.

**Architecture:** Each page is a `page()` call from `@casehubio/pages-ui` returning a declarative component tree. The root `dashboard.ts` uses `tree()` navigation with "Guided/" and "Explore/" path groups. All data comes from REST endpoints via `dataset()` bindings. The `loadSite()` runtime mounts the tree into the DOM. No custom React/Vue — pure casehub-pages DSL.

**Tech Stack:** TypeScript 5.x, casehub-pages (`@casehubio/pages-ui`, `@casehubio/pages-runtime`), esbuild, Quarkus Quinoa

**Spec:** `specs/2026-06-27-clinical-demo-ui-design.md` (Rev 6 — final)

**Plan 1 (prerequisite):** `plans/2026-06-27-clinical-demo-ui-backend.md` — all REST endpoints exist

## Global Constraints

- casehub-pages packages linked locally via `file:` protocol — not published to npm
- Packages at `/Users/mdproctor/claude/casehub/pages/packages/pages-runtime` and `.../pages-ui`
- Post-rename (#55): CSS vars are `--pages-*`, events are `pages-*`, elements are `<pages-*>`
- DSL builders (`page`, `table`, `metric`, `barChart`, `tree`, `columns`, `lookup`, `groupBy`, `col`, `count`, `sum`, etc.) — stable, unaffected by rename
- Imports from `@casehubio/pages-ui` (DSL) and `@casehubio/pages-runtime` (loadSite)
- Deterministic trial UUID: `316e3846-4ea7-3b18-a6f7-e01ce6582a69`
- **The Patient Tracker example** (`casehub-pages/examples/dashboards/Clinical/Patient Tracker.ts`) is the canonical DSL reference — follow its calling conventions for `columns()`, `metric()`, `lookup()`, `groupBy()`, `count()` etc.
- **The Quinoa host template** (`casehub-pages/templates/quinoa-host/`) is the integration reference for `loadSite()` and URL-bound `dataset()`
- Action buttons use `html()` with JavaScript `fetch()` onclick (workaround until casehub-pages#46)
- Testing is visual: `mvn quarkus:dev` + browser. Playwright (Plan 3) adds automation later.
- All commits reference `Refs casehubio/clinical#93`
- Verify DSL signatures against casehub-pages source if compilation errors occur

---

### Task 1: Package Linking + Quinoa Re-add + Infrastructure Files

**Files:**
- Modify: `runtime/src/main/webui/package.json` — switch to `file:` deps
- Modify: `runtime/pom.xml` — re-add `quarkus-quinoa` dependency
- Modify: `runtime/src/main/resources/application.properties` — re-enable Quinoa
- Create: `runtime/src/main/webui/src/datasets.ts`
- Create: `runtime/src/main/webui/src/theme.ts`
- Create: `runtime/src/main/webui/src/narrative.ts`
- Create: `runtime/src/main/webui/src/dashboard.ts`
- Modify: `runtime/src/main/webui/src/index.ts` — wire loadSite()

**Interfaces:**
- Consumes: `@casehubio/pages-ui` (page, tree, dataset, markdown), `@casehubio/pages-runtime` (loadSite)
- Produces: `TRIAL_ID` constant, dataset definitions, theme, narrative text, root dashboard with tree navigation (placeholder pages until Tasks 2-6)

- [ ] **Step 1: Update package.json with file: deps**

Replace version strings with relative file paths. Verify the path resolves: `ls runtime/src/main/webui/../../../../../pages/packages/pages-runtime/package.json`

```json
{
  "dependencies": {
    "@casehubio/pages-runtime": "file:../../../../../pages/packages/pages-runtime",
    "@casehubio/pages-ui": "file:../../../../../pages/packages/pages-ui"
  }
}
```

- [ ] **Step 2: Run npm install**

```bash
cd runtime/src/main/webui && npm install
```

Verify: `ls node_modules/@casehubio/pages-runtime/dist/index.js`

- [ ] **Step 3: Re-add Quinoa to pom.xml**

Add `quarkus-quinoa` dependency (was removed in Plan 1 because npm install failed without linked packages).

- [ ] **Step 4: Update application.properties**

Replace the commented-out Quinoa section with:
```properties
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=false
```

- [ ] **Step 5: Create datasets.ts**

```typescript
import { dataset } from "@casehubio/pages-ui";

export const TRIAL_ID = "316e3846-4ea7-3b18-a6f7-e01ce6582a69";

export const trialSummaryDs = dataset("trial-summary", `/api/trials/${TRIAL_ID}/summary`);
export const agentsDs = dataset("agents", `/api/trials/${TRIAL_ID}/agents`);
export const patientsDs = dataset("patients", `/api/trials/${TRIAL_ID}/patients`);
export const adverseEventsDs = dataset("adverse-events", `/api/trials/${TRIAL_ID}/adverse-events`);
export const deviationsDs = dataset("deviations", `/api/trials/${TRIAL_ID}/deviations`);
export const ledgerEntriesDs = dataset("ledger-entries", `/api/trials/${TRIAL_ID}/ledger-entries`);
```

Verify the `dataset()` function signature against the Quinoa host template. If it accepts refresh config, add `{ refreshTime: "3s" }` to `adverseEventsDs` and `deviationsDs`.

- [ ] **Step 6: Create theme.ts**

CSS custom properties using `--pages-*` (post-rename). Inject via a `<style>` tag in `index.html` or via the `loadSite()` options if a theme API is available.

- [ ] **Step 7: Create narrative.ts**

All 8 guided step narratives as exported string constants (STEP1_NARRATIVE through STEP8_NARRATIVE). Content from the spec's page designs section.

- [ ] **Step 8: Create dashboard.ts with tree() navigation**

```typescript
import { page, tree, markdown } from "@casehubio/pages-ui";

const placeholder = (name: string) => page(name, markdown(`*${name} — coming soon*`));

export const dashboard = page("CaseHub Clinical",
  tree(
    ["Guided/1. Trial Overview", placeholder("Trial Overview")],
    ["Guided/2. Meet the AI Agents", placeholder("AI Agents")],
    ["Guided/3. Protocol Deviation", placeholder("Deviation")],
    ["Guided/4. PI Authorisation", placeholder("PI Auth")],
    ["Guided/5. Grade 4 AE Reported", placeholder("AE Event")],
    ["Guided/6. AI Decision & Governance", placeholder("Governance")],
    ["Guided/7. Resolution & Trust", placeholder("Resolution")],
    ["Guided/8. The Proof", placeholder("The Proof")],
    ["Explore/Trial Dashboard", placeholder("Trial Dashboard")],
    ["Explore/Adverse Events", placeholder("Adverse Events")],
    ["Explore/Audit Trail", placeholder("Audit Trail")],
    ["Explore/Protocol Deviations", placeholder("Deviations")],
    ["Explore/Trust Network", placeholder("Trust Network")],
    ["Explore/Site Detail", placeholder("Site Detail")]
  )
);
```

- [ ] **Step 9: Wire index.ts**

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { dashboard } from "./dashboard";

const container = document.getElementById("app");
if (container) {
  loadSite(container, dashboard).catch(console.error);
}
```

- [ ] **Step 10: Build and verify**

`mvn quarkus:dev -pl runtime` — open browser at `http://localhost:8080`. Tree nav renders with Guided/Explore groups. Placeholder pages display. No JS errors.

- [ ] **Step 11: Commit**

---

### Task 2: Guided Steps 1-2 — Trial Overview + AI Agents

**Files:**
- Create: `runtime/src/main/webui/src/guided/step1-overview.ts`
- Create: `runtime/src/main/webui/src/guided/step2-agents.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts`

**Interfaces:**
- Consumes: datasets.ts, narrative.ts, Patient Tracker DSL patterns
- Produces: `step1Overview`, `step2Agents` page exports

- [ ] **Step 1: Create step1-overview.ts**

Components: narrative markdown, 4-column metric row (phase, enrolled, AEs, deviations), enrollment bar chart by site, sites table. Use `trialSummaryDs`. Follow Patient Tracker `columns()`/`metric()`/`lookup()` patterns exactly.

- [ ] **Step 2: Create step2-agents.ts**

Components: narrative markdown, agents table (capability, trust score, dimension, phase, decisions, endorsements), trust policy summary (markdown table — static data from `ClinicalTrustRoutingPolicyProvider`), 3 metric cards (gated types, dimensions, oversight policy).

- [ ] **Step 3: Update dashboard.ts, build, verify, commit**

---

### Task 3: Guided Steps 3-4 — Deviation + PI Authorisation

**Files:**
- Create: `runtime/src/main/webui/src/guided/step3-deviation.ts`
- Create: `runtime/src/main/webui/src/guided/step4-pi-auth.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts`

**Interfaces:**
- Consumes: datasets.ts, narrative.ts, `POST /api/trials/{id}/sites/{siteId}/deviations`, `POST /api/demo/deviations/{id}/approve-pi`
- Produces: `step3Deviation`, `step4PiAuth` page exports

- [ ] **Step 1: Create step3-deviation.ts**

Action button via `html()` onclick → POST CRITICAL deviation. Deviations table with polling. Button checks dataset for existing CRITICAL deviation before enabling (idempotency).

**Entity ID resolution:** The seeder creates entities with random UUIDs (except the trial). To get Site B's ID and a patient enrollment ID, either: (a) fetch from the patients/summary endpoint at page load, or (b) add a `GET /api/demo/config` endpoint that returns all seeded entity IDs. Option (b) is cleaner — file an issue or implement inline.

- [ ] **Step 2: Create step4-pi-auth.ts**

Action button: "Approve as PI" → POST to demo endpoint. Shows commitment lifecycle (COMMANDED → APPROVED → ESCALATED). Deviation ledger chain from ledger-entries dataset filtered by type.

- [ ] **Step 3: Update dashboard.ts, build, verify, commit**

---

### Task 4: Guided Steps 5-6 — AE Event + AI Governance

**Files:**
- Create: `runtime/src/main/webui/src/guided/step5-ae-event.ts`
- Create: `runtime/src/main/webui/src/guided/step6-governance.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts`

**Interfaces:**
- Consumes: datasets.ts, narrative.ts, `POST /adverse-events`, `GET /adverse-events/{aeId}/governance`
- Produces: `step5AeEvent`, `step6Governance` page exports

- [ ] **Step 1: Create step5-ae-event.ts**

Action button: "Report Adverse Event" → POST Grade 4 unexpected AE at Site B. AE detail display + table with polling. DSMB rollup callout in narrative.

- [ ] **Step 2: Create step6-governance.ts — hero layout**

Two-column layout via `columns()`. Left: "What the AI decided" (AE SUSAR fields). Right: "How the platform governed it" (trust score, threshold, gate status). Data from governance endpoint — fetched via `html()` with `fetch()` since it's a single-object response, not a dataset/lookup pattern.

- [ ] **Step 3: Update dashboard.ts, build, verify, commit**

---

### Task 5: Guided Steps 7-8 — Resolution + The Proof

**Files:**
- Create: `runtime/src/main/webui/src/guided/step7-resolution.ts`
- Create: `runtime/src/main/webui/src/guided/step8-proof.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts`

**Interfaces:**
- Consumes: datasets.ts, narrative.ts, `POST /demo/adverse-events/{aeId}/approve-susar-gate`, Merkle verification endpoint
- Produces: `step7Resolution`, `step8Proof` page exports

- [ ] **Step 1: Create step7-resolution.ts**

Action button: "Approve SUSAR Determination" → POST to demo endpoint. Displays gate decision, attestation (ENDORSED), trust score before/after, regulatory submission status.

- [ ] **Step 2: Create step8-proof.ts**

Ledger entries table. Merkle verification button via `html()` → `GET .../ledger/verify`. Displays VERIFIED result. Inclusion proof detail on entry selection.

- [ ] **Step 3: Update dashboard.ts, build, verify, commit**

All 8 guided steps complete.

---

### Task 6: Explore Mode — 6 Dashboard Pages

**Files:**
- Create: `runtime/src/main/webui/src/explore/trial-dashboard.ts`
- Create: `runtime/src/main/webui/src/explore/adverse-events.ts`
- Create: `runtime/src/main/webui/src/explore/audit-trail.ts`
- Create: `runtime/src/main/webui/src/explore/deviations.ts`
- Create: `runtime/src/main/webui/src/explore/trust-network.ts`
- Create: `runtime/src/main/webui/src/explore/site-detail.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts`

**Interfaces:**
- Consumes: datasets.ts, casehub-pages DSL
- Produces: 6 explore page exports

- [ ] **Steps 1-6: Create each explore page**

1. **trial-dashboard.ts** (must have) — metrics, enrollment chart, activity table
2. **adverse-events.ts** (must have) — AE table with SLA styling, sortable
3. **audit-trail.ts** (must have) — ledger table with type filter selector, verify button
4. **deviations.ts** (should have) — deviation table with PI status
5. **trust-network.ts** (should have) — agent table with trust metrics
6. **site-detail.ts** (could defer) — per-site patient table with cross-filter

Each is a standalone `page()` composition using the datasets and DSL patterns established in Tasks 1-5.

- [ ] **Step 7: Update dashboard.ts, build, verify all 14 pages, commit**

---

## Self-Review

**Spec coverage:** All 8 guided steps ✓, all 6 explore pages ✓, tree() navigation ✓, datasets with TRIAL_ID ✓, theme (--pages-*) ✓, narrative text ✓, action buttons (html workaround) ✓, hero layout ✓, Merkle verification ✓, DSMB callout ✓, button idempotency ✓, Quinoa re-add ✓.

**Open implementation decision:** Entity ID resolution for action buttons (Tasks 3-4). The seeder's entity IDs (except trial) are random UUIDs. The UI needs Site B's ID and enrollment IDs to construct POST URLs. Resolution: either fetch from existing endpoints or add a `GET /demo/config` convenience endpoint. The implementer decides.

**Type consistency:** `TRIAL_ID` defined once in datasets.ts, used throughout. Dataset IDs consistent across all files. Narrative exports match between narrative.ts and step files.
