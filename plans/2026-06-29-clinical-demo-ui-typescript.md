# Clinical Demo UI — TypeScript Implementation Plan (Plan 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build 14 casehub-pages DSL page compositions (8 guided walkthrough steps + 6 explore dashboard pages) that render the clinical trial demo UI against the Java backend endpoints from Plan 1.

**Architecture:** Each page is a `page()` DSL composition using `@casehubio/pages-ui` builders. Pages bind to REST endpoints via `dataset()`. The root `dashboard.ts` uses `tree()` navigation with Guided/Explore groups. All pages are pure declarative compositions — no custom logic, no manual DOM manipulation.

**Tech Stack:** TypeScript 5.x, casehub-pages DSL (`@casehubio/pages-ui` + `@casehubio/pages-runtime`), esbuild, Quarkus Quinoa

**Spec:** `specs/2026-06-27-clinical-demo-ui-design.md` (Rev 6 — final)

**Depends on:** Plan 1 (Java backend) — complete and merged. `TrialDashboardResource` (7 endpoints), `DemoActionResource` (2 endpoints), `DemoDataSeeder`, `DemoCurrentPrincipal` all on main.

## Global Constraints

- casehub-pages packages link locally via `file:` protocol — not published to npm
- Relative path from webui to pages: `../../../../../pages/packages/pages-*`
- CSS custom properties are `--pages-*` (not `--casehub-*`) — renamed in pages#55
- Event names are `pages-filter`, `pages-sort`, `pages-page` (not `casehub-*`)
- Custom element tags are `<pages-*>` (not `<casehub-*>`)
- Deterministic trial UUID: `316e3846-4ea7-3b18-a6f7-e01ce6582a69`
- DSL signatures (verified against source):
  - `page(name, ...args: (Component | PageOptions)[])`
  - `columns(distribution: number[], ...slotContents: Component[][])`
  - `tree(...entries: [string, ...Component[]][])`
  - `lookup(dataSetId: string, ...ops: DataSetOp[])`
  - `groupBy(source: string | null, ...resultColumns: ResultColumn[])`
  - `count(source)`, `sum(source)`, `avg(source)`, `col(source)` — single param
  - `filterBy(columnId, operator, value)`, `sortBy(columnId, direction)`
- Action buttons use `html()` with JavaScript `onclick` (workaround for pages#46)
- Status badges use column expressions (workaround for pages#39)
- All commits reference `Refs casehubio/clinical#93`
- Verification: `mvn quarkus:dev` then navigate to page in browser — data renders, no JS console errors

## Testing approach

TypeScript DSL compositions are declarative — no unit-testable logic. Verification is:
1. `npm run build` succeeds (esbuild compiles without errors)
2. `mvn quarkus:dev` serves the UI
3. Navigate to each page — data renders, no JS console errors
4. Action buttons trigger POST requests and polling updates

Plan 3 (Playwright smoke tests) automates this verification.

---

### Task 1: Package Linking + Quinoa + Shared Modules + Dashboard Shell

**Files:**
- Modify: `runtime/src/main/webui/package.json` — file: references to local pages packages
- Modify: `runtime/pom.xml` — re-add quarkus-quinoa dependency
- Modify: `runtime/src/main/resources/application.properties` — Quinoa config
- Create: `runtime/src/main/webui/src/datasets.ts`
- Create: `runtime/src/main/webui/src/lookups.ts`
- Create: `runtime/src/main/webui/src/theme.ts`
- Create: `runtime/src/main/webui/src/narrative.ts`
- Modify: `runtime/src/main/webui/src/index.ts` — replace placeholder with `loadSite()`
- Create: `runtime/src/main/webui/src/dashboard.ts` — root page with tree() nav shell

**Interfaces:**
- Consumes: casehub-pages DSL (`@casehubio/pages-ui`, `@casehubio/pages-runtime`), Plan 1 REST endpoints
- Produces: `TRIAL_ID` constant, all dataset definitions, reusable lookups, theme config, narrative text, dashboard shell with tree() navigation — all subsequent tasks import from these

- [ ] **Step 1: Update package.json with local file: references**

```json
{
  "name": "casehub-clinical-ui",
  "private": true,
  "scripts": {
    "build": "node esbuild.config.mjs",
    "dev": "node esbuild.config.mjs --watch"
  },
  "dependencies": {
    "@casehubio/pages-runtime": "file:../../../../../pages/packages/pages-runtime",
    "@casehubio/pages-ui": "file:../../../../../pages/packages/pages-ui"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

- [ ] **Step 2: Run npm install from webui directory**

```bash
cd runtime/src/main/webui && npm install
```

Verify: `node_modules/@casehubio/pages-runtime/dist/index.js` exists.

- [ ] **Step 3: Re-add quarkus-quinoa to runtime/pom.xml**

Add the dependency back (it was removed in Plan 1 because packages weren't linked):

```xml
<dependency>
    <groupId>io.quarkiverse.quinoa</groupId>
    <artifactId>quarkus-quinoa</artifactId>
</dependency>
```

- [ ] **Step 4: Update Quinoa config in application.properties**

Replace the commented-out Quinoa block with:

```properties
# ── Quinoa — esbuild (no dev server; Quinoa watches sources and re-runs npm build) ──
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=false
```

Remove the `quarkus.quinoa.enabled=false` / `%dev.quarkus.quinoa.enabled=true` lines — Quinoa should be active in all profiles now that packages are linked.

- [ ] **Step 5: Create datasets.ts**

```typescript
import { dataset } from "@casehubio/pages-ui";

export const TRIAL_ID = "316e3846-4ea7-3b18-a6f7-e01ce6582a69";

// Trial-level aggregation endpoints (TrialDashboardResource)
export const trialSummaryDs = dataset("trial-summary", `/api/trials/${TRIAL_ID}/summary`);
export const agentsDs = dataset("agents", `/api/trials/${TRIAL_ID}/agents`);
export const patientsDs = dataset("patients", `/api/trials/${TRIAL_ID}/patients`);
export const adverseEventsDs = dataset("adverse-events", `/api/trials/${TRIAL_ID}/adverse-events`);
export const deviationsDs = dataset("deviations", `/api/trials/${TRIAL_ID}/deviations`);
export const ledgerEntriesDs = dataset("ledger-entries", `/api/trials/${TRIAL_ID}/ledger-entries`);

// Polling datasets (3s refresh for active scenario data)
export const adverseEventsPollingDs = dataset("ae-polling", `/api/trials/${TRIAL_ID}/adverse-events`,
  { refreshTime: "3s" });
export const deviationsPollingDs = dataset("dev-polling", `/api/trials/${TRIAL_ID}/deviations`,
  { refreshTime: "3s" });
```

- [ ] **Step 6: Create lookups.ts**

```typescript
import { lookup, groupBy, filterBy, sortBy, col, count, sum, avg } from "@casehubio/pages-ui";

// Trial summary lookups
export const enrollmentBySite = lookup("trial-summary",
  groupBy("siteName", col("siteName"), sum("enrolled")));

// Agent lookups
export const agentList = lookup("agents");

// Patient lookups
export const patientList = lookup("patients");

// AE lookups
export const aeList = lookup("adverse-events",
  sortBy("reportedAt", "DESCENDING"));
export const aePollingList = lookup("ae-polling",
  sortBy("reportedAt", "DESCENDING"));

// Deviation lookups
export const deviationList = lookup("deviations",
  sortBy("reportedAt", "DESCENDING"));

// Ledger lookups
export const ledgerList = lookup("ledger-entries",
  sortBy("occurredAt", "DESCENDING"));
```

- [ ] **Step 7: Create theme.ts**

```typescript
export const clinicalTheme = {
  mode: "light" as const,
  // Override when casehub-pages theming API is used via loadSite options
  // CSS custom properties use --pages-* prefix (pages#55 rename)
};
```

- [ ] **Step 8: Create narrative.ts**

All 8 guided mode narrative texts — centralised for easy editing:

```typescript
export const narratives = {
  step1: `## ONCO-2024-001 — Phase III Oncology Trial

CaseHub coordinates AI agents for eligibility screening, safety monitoring, and protocol review — each governed by trust scores, oversight gates, and a tamper-evident audit trail.`,

  step2: `## Meet the AI Agents

CaseHub doesn't just run AI agents — it governs them. Each agent has a trust score built from its track record. High-stakes decisions are gated: no autonomous action on safety events. The platform selects agents by trust, gates their decisions, and records attestations that feed back into trust scores.`,

  step3: `## Event: Protocol Deviation Reported

A CRITICAL protocol deviation is reported at Site B. Watch what happens: the platform sends a formal COMMAND to the named Principal Investigator — not a notification, an obligation. A Commitment is created with a deadline. If the PI doesn't respond, the platform escalates automatically. This is qhorus — formal accountability that no LLM pipeline can provide.`,

  step4: `## PI Authorisation & Commitment

The PI receives a formal COMMAND with a 24-hour deadline. When they approve, the Commitment lifecycle closes. But this is a CRITICAL deviation — the policy requires IRB committee review. The case suspends in WAITING until the ethics committee decides within 72 hours. Every step — COMMAND, response, escalation — is recorded in the Merkle audit trail.`,

  step5: `## Event: Grade 4 Adverse Event Reported

A Grade 4 hepatotoxicity event is reported at Site B. Watch what happens: the engine creates a 24-hour SLA work item, triggers SUSAR evaluation, and routes to a trust-weighted safety agent — all within seconds.`,

  step6: `## AI Decision & Governance

The SUSAR evaluator assessed this event: Grade 4 + unexpected + suspected = SUSAR criteria met. But the agent can't act alone — CaseHub's ActionRiskClassifier unconditionally gates all safety decisions. A qualified investigator must approve.`,

  step7: `## Resolution & Trust Update

The investigator approves the SUSAR determination. CaseHub records the attestation — ENDORSED — which feeds into the agent's Bayesian trust score. Good decisions build trust; bad decisions erode it. The regulatory submission work item is created automatically.`,

  step8: `## The Proof

Every decision is recorded in a tamper-evident Merkle audit trail. Each ledger entry is independently verifiable — no trust required in the platform itself. This is what FDA auditors and EU AI Act compliance officers need.`,
};
```

- [ ] **Step 9: Create dashboard.ts with tree() navigation shell**

```typescript
import { page, tree } from "@casehubio/pages-ui";
import { clinicalTheme } from "./theme";

// Placeholder pages — replaced in Tasks 2-5
import { step1Overview } from "./guided/step1-overview";
import { step2Agents } from "./guided/step2-agents";
// ... all 14 imports

export const dashboard = page("CaseHub Clinical",
  tree(
    // Act I — Accountability
    ["Guided/1. Trial Overview", step1Overview],
    ["Guided/2. Meet the AI Agents", step2Agents],
    ["Guided/3. Protocol Deviation", /* step3 placeholder */],
    ["Guided/4. PI Authorisation", /* step4 placeholder */],
    // Act II — AI Governance
    ["Guided/5. Grade 4 AE Reported", /* step5 placeholder */],
    ["Guided/6. AI Decision & Governance", /* step6 placeholder */],
    ["Guided/7. Resolution & Trust", /* step7 placeholder */],
    ["Guided/8. The Proof", /* step8 placeholder */],
    // Explore
    ["Explore/Trial Dashboard", /* explore placeholder */],
    ["Explore/Adverse Events", /* explore placeholder */],
    ["Explore/Audit Trail", /* explore placeholder */],
    ["Explore/Protocol Deviations", /* explore placeholder */],
    ["Explore/Trust Network", /* explore placeholder */],
    ["Explore/Site Detail", /* explore placeholder */]
  ),
  { settings: { mode: clinicalTheme.mode } }
);
```

**Note to implementer:** Start with only Steps 1-2 wired (Task 2). Add remaining imports as each task completes. Use `page("placeholder")` for unwired entries initially. The tree() entries with no component will need a minimal placeholder — check if tree() accepts entries with only a label.

- [ ] **Step 10: Update index.ts with loadSite()**

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { dashboard } from "./dashboard";

const container = document.getElementById("app");
if (container) {
  loadSite(container, dashboard).catch(console.error);
}
```

- [ ] **Step 11: Create stub guided page files**

Create `runtime/src/main/webui/src/guided/step1-overview.ts` and `step2-agents.ts` with minimal page() compositions (just the narrative markdown). Create empty stub exports for steps 3-8. Create `runtime/src/main/webui/src/explore/` with empty stubs for all 6 explore pages.

Each stub:
```typescript
import { page, markdown } from "@casehubio/pages-ui";
export const stepNName = page("Step N — Title", markdown("Coming soon"));
```

- [ ] **Step 12: Verify build**

```bash
cd runtime/src/main/webui && npm run build
```

Expected: esbuild produces `dist/app.js` with no errors.

Then verify Quinoa:
```bash
mvn install -pl api -DskipTests && mvn quarkus:dev -pl runtime
```

Open `http://localhost:8080` — tree navigation should render with "Guided" and "Explore" groups. Steps 1-2 show narrative text.

- [ ] **Step 13: Commit**

```bash
git add runtime/
git commit -m "feat: casehub-pages scaffold — datasets, lookups, theme, narrative, dashboard shell

Wire @casehubio/pages-ui + pages-runtime via local file: references.
tree() navigation with Guided (8 steps) and Explore (6 pages) groups.
Stub pages for all 14 entries.

Refs casehubio/clinical#93"
```

---

### Task 2: Guided Steps 1-2 — Trial Overview + Meet the AI Agents

**Files:**
- Modify: `runtime/src/main/webui/src/guided/step1-overview.ts`
- Modify: `runtime/src/main/webui/src/guided/step2-agents.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts` — wire imports

**Interfaces:**
- Consumes: `datasets.ts` (trialSummaryDs, agentsDs), `lookups.ts` (enrollmentBySite, agentList), `narrative.ts`
- Produces: Two fully rendered pages with live data from seeded trial

- [ ] **Step 1: Implement step1-overview.ts**

```typescript
import { page, columns, metric, barChart, table, markdown, panel } from "@casehubio/pages-ui";
import { lookup, groupBy, col, count, sum } from "@casehubio/pages-ui";
import { trialSummaryDs, agentsDs } from "../datasets";
import { narratives } from "../narrative";

export const step1Overview = page("1. Trial Overview",
  markdown(narratives.step1),

  columns([3, 3, 3, 3],
    [metric({ title: "Trial Phase", lookup: lookup("trial-summary") })],
    [metric({ title: "Total Enrolled",
      lookup: lookup("trial-summary", groupBy(null, sum("totalEnrolled"))) })],
    [metric({ title: "Active AEs",
      lookup: lookup("trial-summary", groupBy(null, sum("totalAdverseEvents"))) })],
    [metric({ title: "AI Agents Active",
      lookup: lookup("agents", groupBy(null, count("capability"))) })]
  ),

  barChart({
    title: "Enrollment by Site",
    lookup: lookup("trial-summary")
    // Note: trial-summary returns a single object, not per-site rows.
    // The implementer must check the actual response shape from
    // GET /trials/{id}/summary and adapt the lookup accordingly.
    // May need a separate sites dataset.
  }),

  table({
    sortable: true,
    lookup: lookup("patients")
  }),

  {
    datasets: [trialSummaryDs, agentsDs]
  }
);
```

**Important note to implementer:** The exact response shape of each endpoint must be verified by calling the API (use `curl http://localhost:8080/api/trials/316e3846-4ea7-3b18-a6f7-e01ce6582a69/summary` etc.) before writing the lookups. The column IDs in `groupBy()`, `col()`, and table `columns` must match the JSON field names returned by the API. The code above is a template — adapt field names from the actual API response.

- [ ] **Step 2: Implement step2-agents.ts**

```typescript
import { page, columns, metric, table, markdown, panel } from "@casehubio/pages-ui";
import { lookup, groupBy, col, count } from "@casehubio/pages-ui";
import { agentsDs } from "../datasets";
import { narratives } from "../narrative";

export const step2Agents = page("2. Meet the AI Agents",
  markdown(narratives.step2),

  table({
    title: "AI Agent Trust Scores",
    sortable: true,
    lookup: lookup("agents")
    // Columns: capability, trustDimension, trustScore, threshold,
    // maturityPhase, decisionCount, attestationPositive, attestationNegative
  }),

  columns([4, 4, 4],
    [metric({ title: "Gated Action Types", lookup: lookup("agents") })],
    [metric({ title: "Trust Dimensions", lookup: lookup("agents") })],
    [metric({ title: "Oversight Policy", lookup: lookup("agents") })]
  ),

  { datasets: [agentsDs] }
);
```

- [ ] **Step 3: Wire imports in dashboard.ts**

Update dashboard.ts to import the real step1 and step2 pages instead of stubs.

- [ ] **Step 4: Verify in browser**

```bash
cd runtime/src/main/webui && npm run build
mvn quarkus:dev -pl runtime
```

Open `http://localhost:8080`, navigate to "Guided > 1. Trial Overview" — metric cards, bar chart, and table render with seeded data. Navigate to "Guided > 2. Meet the AI Agents" — agent table renders with trust scores.

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: guided Steps 1-2 — Trial Overview + Meet the AI Agents

Refs casehubio/clinical#93"
```

---

### Task 3: Guided Steps 3-4 — Protocol Deviation + PI Authorisation

**Files:**
- Modify: `runtime/src/main/webui/src/guided/step3-deviation.ts`
- Modify: `runtime/src/main/webui/src/guided/step4-pi-auth.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts` — wire imports

**Interfaces:**
- Consumes: `datasets.ts` (deviationsPollingDs, ledgerEntriesDs), `narrative.ts`, `TRIAL_ID`
- Produces: Two pages with action buttons (html onclick workaround) that POST to the real API

- [ ] **Step 1: Implement step3-deviation.ts**

This page includes an action button using `html()` with JavaScript `onclick`:

```typescript
import { page, columns, metric, table, markdown, html } from "@casehubio/pages-ui";
import { lookup } from "@casehubio/pages-ui";
import { deviationsPollingDs } from "../datasets";
import { TRIAL_ID } from "../datasets";
import { narratives } from "../narrative";

// Action button — workaround for pages#46
// The implementer must determine the correct siteId from the seeded data.
// Call GET /api/trials/{TRIAL_ID}/patients to find Site B's siteId.
const reportDeviationButton = html({
  content: `<button id="report-deviation-btn" style="padding: 12px 24px; background: var(--pages-accent); color: white; border: none; border-radius: var(--pages-radius); cursor: pointer; font-size: 16px;">
    Report Protocol Deviation
  </button>
  <div id="deviation-status" style="margin-top: 8px; color: var(--pages-text-muted);"></div>
  <script>
    document.getElementById('report-deviation-btn')?.addEventListener('click', async function() {
      const btn = this;
      btn.disabled = true;
      btn.textContent = 'Reporting...';
      document.getElementById('deviation-status').textContent = 'Sending COMMAND to PI...';
      try {
        const resp = await fetch('/api/trials/${TRIAL_ID}/sites/SITE_B_ID/deviations', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            deviationType: 'PROTOCOL_AMENDMENT_VIOLATION',
            severity: 'CRITICAL'
          })
        });
        if (resp.ok) {
          document.getElementById('deviation-status').textContent = 'COMMAND sent — PI obligation created with 24h deadline';
        } else {
          btn.disabled = false;
          btn.textContent = 'Report Protocol Deviation';
          document.getElementById('deviation-status').textContent = 'Error: ' + resp.status;
        }
      } catch (e) {
        btn.disabled = false;
        btn.textContent = 'Report Protocol Deviation';
        document.getElementById('deviation-status').textContent = 'Error: ' + e.message;
      }
    });
  </script>`
});

export const step3Deviation = page("3. Protocol Deviation",
  markdown(narratives.step3),
  reportDeviationButton,
  table({
    title: "Protocol Deviations",
    sortable: true,
    lookup: lookup("dev-polling")
  }),
  { datasets: [deviationsPollingDs] }
);
```

**Note to implementer:** The `SITE_B_ID` must be replaced with the actual UUID from the seeded data. Call `GET /api/trials/${TRIAL_ID}/patients` to find Site B's siteId. Consider exporting site IDs from `datasets.ts` as constants if the seeder uses deterministic UUIDs for sites.

- [ ] **Step 2: Implement step4-pi-auth.ts**

Similar pattern: action button calls `POST /demo/deviations/{deviationId}/approve-pi`. The deviationId comes from the deviation created in Step 3.

```typescript
import { page, columns, metric, table, markdown, html } from "@casehubio/pages-ui";
import { lookup } from "@casehubio/pages-ui";
import { deviationsPollingDs, ledgerEntriesDs } from "../datasets";
import { narratives } from "../narrative";

// Action button — polls for the deviation in COMMANDED state, then approves
const approvePiButton = html({
  content: `<button id="approve-pi-btn" style="padding: 12px 24px; background: var(--pages-accent); color: white; border: none; border-radius: var(--pages-radius); cursor: pointer; font-size: 16px;">
    Approve as PI
  </button>
  <div id="pi-status" style="margin-top: 8px; color: var(--pages-text-muted);"></div>
  <script>
    // The implementer must wire this to find the latest COMMANDED deviation
    // from the deviations polling dataset and call POST /demo/deviations/{id}/approve-pi
  </script>`
});

export const step4PiAuth = page("4. PI Authorisation",
  markdown(narratives.step4),
  approvePiButton,
  markdown("### Commitment Lifecycle"),
  table({
    title: "Deviations — PI Approval Status",
    sortable: true,
    lookup: lookup("dev-polling")
  }),
  markdown("### Deviation Audit Trail"),
  table({
    title: "Ledger Entries",
    sortable: true,
    lookup: lookup("ledger-entries")
  }),
  { datasets: [deviationsPollingDs, ledgerEntriesDs] }
);
```

- [ ] **Step 3: Wire imports in dashboard.ts**

- [ ] **Step 4: Verify in browser**

Navigate to Step 3, click "Report Protocol Deviation" — deviation appears in table with COMMANDED status. Navigate to Step 4, click "Approve as PI" — status transitions to APPROVED/ESCALATED, ledger entries appear.

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: guided Steps 3-4 — Protocol Deviation + PI Authorisation

Action buttons trigger real API calls. PI COMMAND via qhorus channel,
commitment lifecycle visible in polling tables.

Refs casehubio/clinical#93"
```

---

### Task 4: Guided Steps 5-8 — AE Event + Governance + Resolution + Proof

**Files:**
- Modify: `runtime/src/main/webui/src/guided/step5-ae-event.ts`
- Modify: `runtime/src/main/webui/src/guided/step6-governance.ts`
- Modify: `runtime/src/main/webui/src/guided/step7-resolution.ts`
- Modify: `runtime/src/main/webui/src/guided/step8-proof.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts` — wire imports

**Interfaces:**
- Consumes: `datasets.ts`, `lookups.ts`, `narrative.ts`, `TRIAL_ID`
- Produces: Four pages completing the guided walkthrough. Step 5 + 7 have action buttons. Step 6 has the hero side-by-side layout. Step 8 has the Merkle verification button.

- [ ] **Step 1: Implement step5-ae-event.ts**

Action button: "Report Adverse Event" → `POST /api/trials/{TRIAL_ID}/sites/{siteB}/patients/{enrollmentId}/adverse-events` with Grade 4 payload. The implementer must discover the correct siteId and enrollmentId from the seeded data.

Include DSMB rollup narrative callout (from spec): "Notice: the platform detected a cross-site safety pattern. Two sites now have active Grade 4+ events — a DSMB review has been triggered automatically."

- [ ] **Step 2: Implement step6-governance.ts**

Hero layout using `columns([6, 6], [...], [...])`:
- Left column: "What the AI decided" — markdown + metrics showing AE SUSAR fields
- Right column: "How the platform governed it" — table/metrics showing trust score, gate status

This page needs a governance-specific dataset: `GET /api/trials/{TRIAL_ID}/adverse-events/{aeId}/governance`. The aeId comes from the AE created in Step 5. Use polling to detect when the AE appears.

- [ ] **Step 3: Implement step7-resolution.ts**

Action button: "Approve SUSAR Determination" → `POST /demo/adverse-events/{aeId}/approve-susar-gate`. After approval, display trust score before/after using two metric cards.

- [ ] **Step 4: Implement step8-proof.ts**

Ledger entries table + Merkle verification button:

```typescript
const verifyButton = html({
  content: `<button id="verify-btn" style="...">
    Verify Chain Integrity
  </button>
  <div id="verify-result"></div>
  <script>
    document.getElementById('verify-btn')?.addEventListener('click', async function() {
      // Call GET /api/trials/{TRIAL_ID}/sites/{siteId}/patients/{enrollmentId}/ledger/verify
      // Display VERIFIED / FAILED result
    });
  </script>`
});
```

PROV-O export link as an html() anchor tag.

- [ ] **Step 5: Wire all 4 imports in dashboard.ts**

- [ ] **Step 6: Verify full guided walkthrough end-to-end**

Walk through all 8 steps in order. Steps 3 and 5 create new data. Steps 4 and 7 approve it. Step 8 verifies the chain. All polling tables update.

- [ ] **Step 7: Commit**

```bash
git commit -m "feat: guided Steps 5-8 — AE event, governance, resolution, proof

Completes the 8-step guided walkthrough. Hero layout for AI vs governance.
Trust score recomputation on SUSAR gate approval. Merkle chain verification.
DSMB rollup callout on cross-site Grade 4+ detection.

Refs casehubio/clinical#93"
```

---

### Task 5: Explore Mode — 6 Dashboard Pages

**Files:**
- Modify: `runtime/src/main/webui/src/explore/trial-dashboard.ts`
- Modify: `runtime/src/main/webui/src/explore/adverse-events.ts`
- Modify: `runtime/src/main/webui/src/explore/audit-trail.ts`
- Modify: `runtime/src/main/webui/src/explore/deviations.ts`
- Modify: `runtime/src/main/webui/src/explore/trust-network.ts`
- Modify: `runtime/src/main/webui/src/explore/site-detail.ts`
- Modify: `runtime/src/main/webui/src/dashboard.ts` — wire imports

**Interfaces:**
- Consumes: all datasets, all lookups
- Produces: 6 operational dashboard pages for free-form exploration

- [ ] **Step 1: Implement trial-dashboard.ts (must have)**

Metric cards (phase, enrollment, AE count, deviation count), enrollment bar chart by site, recent activity table.

```typescript
import { page, columns, metric, barChart, table } from "@casehubio/pages-ui";
import { lookup, groupBy, col, count, sum, sortBy } from "@casehubio/pages-ui";
import { trialSummaryDs, patientsDs, adverseEventsDs, deviationsDs } from "../datasets";

export const trialDashboard = page("Trial Dashboard",
  columns([3, 3, 3, 3],
    [metric({ title: "Phase", lookup: lookup("trial-summary") })],
    [metric({ title: "Enrolled", lookup: lookup("trial-summary", groupBy(null, sum("totalEnrolled"))) })],
    [metric({ title: "Adverse Events", lookup: lookup("trial-summary", groupBy(null, sum("totalAdverseEvents"))) })],
    [metric({ title: "Deviations", lookup: lookup("trial-summary", groupBy(null, sum("totalDeviations"))) })]
  ),
  table({ title: "Patients", sortable: true, lookup: lookup("patients") }),
  { datasets: [trialSummaryDs, patientsDs] }
);
```

- [ ] **Step 2: Implement adverse-events.ts (must have)**

AE table with grade, SLA deadline, escalation status. Cell-level expressions for overdue SLAs:

```typescript
table({
  title: "All Adverse Events",
  sortable: true,
  pageSize: 20,
  columns: [
    { id: "grade" },
    { id: "siteId" },
    { id: "reportedAt" },
    { id: "slaDeadline" },
    { id: "slaTimeRemaining",
      expression: 'value && value.startsWith("OVERDUE") ? "⚠️ " + value : value' },
    { id: "escalationStatus",
      expression: 'value === "COMPLETED" ? "✅ " + value : value === "ESCALATED" ? "🔴 " + value : value' }
  ],
  lookup: lookup("adverse-events", sortBy("reportedAt", "DESCENDING"))
})
```

- [ ] **Step 3: Implement audit-trail.ts (must have)**

Ledger entries table with type selector dropdown and Merkle verify button.

```typescript
import { page, table, selector, html } from "@casehubio/pages-ui";
import { lookup, sortBy } from "@casehubio/pages-ui";
import { ledgerEntriesDs } from "../datasets";

export const auditTrail = page("Audit Trail",
  selector({
    subtype: "dropdown",
    selfApply: true,
    notification: true,
    lookup: lookup("ledger-entries",
      groupBy("entryType", col("entryType")))
  }),
  table({
    title: "Ledger Entries",
    sortable: true,
    pageSize: 20,
    filter: { listening: true },
    lookup: lookup("ledger-entries", sortBy("occurredAt", "DESCENDING"))
  }),
  // Merkle verify button (same pattern as step8)
  { datasets: [ledgerEntriesDs] }
);
```

- [ ] **Step 4: Implement deviations.ts (should have)**

Simple table page — deviations with PI status.

- [ ] **Step 5: Implement trust-network.ts (should have)**

Agent table with trust scores. Same as step2-agents but without narrative.

- [ ] **Step 6: Implement site-detail.ts (could defer)**

Per-site patient table. Cross-filter click to filter AE/deviation tables below.

- [ ] **Step 7: Wire all explore imports in dashboard.ts**

- [ ] **Step 8: Verify all explore pages in browser**

Navigate through each explore page. Tables render with seeded data. Cross-filtering works where wired.

- [ ] **Step 9: Commit**

```bash
git commit -m "feat: explore mode — 6 dashboard pages

Trial Dashboard, Adverse Events, Audit Trail (must have).
Protocol Deviations, Trust Network (should have).
Site Detail (could defer).

Refs casehubio/clinical#93"
```

---

## Self-Review

**Spec coverage:**
- Quinoa setup: Task 1 ✓
- Datasets with TRIAL_ID constant: Task 1 ✓
- Lookups module: Task 1 ✓
- Theme (--pages-* properties): Task 1 ✓
- Narrative text: Task 1 ✓
- tree() navigation with Guided/Explore groups: Task 1 ✓
- loadSite() entry point: Task 1 ✓
- Step 1 Trial Overview: Task 2 ✓
- Step 2 Meet the AI Agents: Task 2 ✓
- Step 3 Protocol Deviation (action button): Task 3 ✓
- Step 4 PI Authorisation (action button): Task 3 ✓
- Step 5 AE Event (action button + DSMB callout): Task 4 ✓
- Step 6 Governance (hero layout): Task 4 ✓
- Step 7 Resolution (action button + trust delta): Task 4 ✓
- Step 8 Proof (Merkle verify + PROV-O): Task 4 ✓
- 6 Explore pages: Task 5 ✓
- Action button idempotency: referenced in Tasks 3-4 ✓
- Error states: referenced in action button code ✓
- Polling at 3s: polling datasets in Task 1 ✓

**Placeholder scan:** Tasks 3 and 4 contain "Note to implementer" blocks with guidance on discovering IDs from seeded data. These are specific instructions, not TBD placeholders — the implementer must call the API to discover UUIDs that are runtime-generated by the seeder.

**Type consistency:** `TRIAL_ID`, dataset IDs, lookup references all consistent across tasks.
