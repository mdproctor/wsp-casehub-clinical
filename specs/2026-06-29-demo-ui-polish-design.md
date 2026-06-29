# Demo UI Polish — Sites Endpoint, Web Components, Playwright Smoke Tests

**Date:** 2026-06-29
**Issues:** #102, #108, #109
**Branch:** issue-102-playwright-smoke-tests

## Overview

Three items on one branch, in dependency order:

1. **#108** — `GET /trials/{trialId}/sites` list endpoint (step 1 needs it)
2. **#109** — Replace `html()` inline scripts with custom web components (steps 4, 7, 8, audit-trail)
3. **#102** — Playwright smoke tests for the demo UI

## Item 1: Sites List Endpoint (#108)

### Problem

Step 1 (Trial Overview) has TODO'd bar chart and sites table that need a "sites" dataset. No list endpoint exists — `SiteResource` has POST and GET-by-id only.

### Design

Add `GET /sites` to `TrialDashboardResource` returning enriched `SiteRow` records. This follows the established pattern: every enriched dashboard endpoint (`/summary`, `/patients`, `/adverse-events`, `/deviations`, `/agents`, `/ledger-entries`) lives on `TrialDashboardResource`. The sites list is a dashboard aggregation concern (counts across related entities), not a CRUD concern.

No JAX-RS routing conflict: `SiteResource` at `@Path("/trials/{trialId}/sites")` only has `POST /` and `GET /{siteId}` — the bare `GET /trials/{trialId}/sites` path is unoccupied and resolved to `TrialDashboardResource`'s `@Path("/sites")`.

```java
@GET
@Path("/sites")
@RolesAllowed({ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR,
               ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
public Response sites(@PathParam("trialId") UUID trialId) { ... }
```

#### SiteRow record

```java
public record SiteRow(UUID id, String investigatorId, String status,
                      long enrolledCount, long adverseEventCount,
                      long deviationCount) {}
```

#### Query strategy — two-hop join for adverseEventCount

`enrolledCount` and `deviationCount` are single-hop joins (sites → enrollments, sites → deviations). `adverseEventCount` requires a two-hop join because `AdverseEvent` links through `enrollmentId` → `PatientEnrollment.siteId`:

1. Query `TrialSite` by `trialId` + tenant → site IDs
2. Query `PatientEnrollment` by `siteId in (...)` + tenant → enrollment IDs + `enrollmentId→siteId` map
3. Query `AdverseEvent` by `enrollmentId in (...)` + tenant → group by `enrollment.siteId` for per-site counts
4. Query `ProtocolDeviation` by `siteId in (...)` + tenant → per-site counts

This parallels the existing `adverseEvents()` method (TrialDashboardResource lines 168–213) which builds the same `enrollmentToSite` map, but differs in that results are grouped by site rather than returned as a flat list.

### UI Changes

- Add `sitesDs = dataset("sites", "/trials/${TRIAL_ID}/sites")` to `datasets.ts`
- Uncomment and wire up the bar chart and sites table in `step1-overview.ts`
- The bar chart groups by `investigatorId` (the site label — `TrialSite` has no separate name field) with `enrolledCount` as the value axis
- The sites table shows `investigatorId`, `status`, `enrolledCount`, `adverseEventCount`
- The step 1 TODO referenced `siteName` — this field does not exist on `TrialSite`; use `investigatorId` instead

## Item 2: Custom Web Components (#109)

### Problem

Steps 4, 7, 8, and audit-trail use `html()` with inline `<script>` tags. Browsers don't execute scripts added via `innerHTML`. A MutationObserver hack in `index.ts` re-creates script elements so they run. This is fragile and architecturally wrong — interactive behavior should live in components, not injected scripts.

### Why actionButton() is insufficient

Issue #109 suggests using the native `actionButton()` and `alert()` helpers from `webui/src/helpers.ts`. These helpers produce `action-button` component descriptors that are rendered by pages-runtime's `ActionExecutor`. However, `ActionExecutor.execute()` returns only `{ success: true, status }` — the response body is discarded. This makes `actionButton()` unsuitable for the four pages being converted:

- **Step 4 (`<clinical-pi-approval>`):** Must pre-fetch the deviations list, find a COMMANDED deviation dynamically, and conditionally enable the button. `actionButton()` does simple POSTs to static URLs with no pre-fetch or conditional rendering.
- **Step 7 (`<clinical-susar-gate>`):** Must pre-fetch the AE list to find a REQUESTED AE, perform an idempotency check before enabling, and display structured response data (trust score delta, attestation, investigator ID). `actionButton()` cannot display response bodies.
- **Step 8 / audit-trail (`<clinical-merkle-verify>`):** Performs a GET request (not POST) and displays VERIFIED/FAILED state with Merkle root hash. `actionButton()` only supports POST.

A future enhancement to pages-runtime's `ActionExecutor` to surface response bodies would make `actionButton()` viable for simple POST-and-display cases. This is tracked in the Deferred section.

### Design

Three custom web components replace all four pages' inline scripts:

| Component | Element name | Pages | Behavior |
|-----------|-------------|-------|----------|
| `ClinicalPiApproval` | `<clinical-pi-approval>` | Step 4 | Fetch deviations → find COMMANDED → show button → POST approve → display result |
| `ClinicalSusarGate` | `<clinical-susar-gate>` | Step 7 | Fetch AE list → find REQUESTED AE → idempotency check → POST gate approve → display trust score delta, attestation |
| `ClinicalMerkleVerify` | `<clinical-merkle-verify>` | Step 8, Audit-trail | Show verify button → GET Merkle verify → display VERIFIED/FAILED + Merkle root |

### Component Conventions

- **Light DOM** — no `attachShadow()`. Inherits page styles, normal event bubbling, no accessibility complications.
- **Attribute-driven** — configuration via HTML attributes (`trial-id`, `site-id`, `patient-id`). Mapped via static `observedAttributes`.
- **One-time init guard** — `connectedCallback` checks `this._initialized` to prevent duplicate setup if the element is moved in the DOM.
- **Cleanup** — `disconnectedCallback` aborts in-flight fetches via `AbortController`.
- **No framework** — vanilla `HTMLElement` subclasses, bundled by existing esbuild.

### ClinicalSusarGate — self-contained AE discovery

`<clinical-susar-gate>` does NOT use `sessionStorage`. It fetches `GET /trials/{trialId}/adverse-events` on `connectedCallback` and finds the first AE with `escalationStatus === 'REQUESTED'`. This mirrors `<clinical-pi-approval>`'s pattern of fetching deviations and finding COMMANDED ones.

This eliminates the fragile cross-page `sessionStorage` dependency. The current `step7-resolution.ts` reads `sessionStorage.getItem('demo-ae-id')` set by step 5, but `actionButton()` in step 5 discards the response body — the AE ID was never stored. The self-contained fetch approach works regardless of navigation order.

### ClinicalMerkleVerify — dual-mode verification

`<clinical-merkle-verify>` supports both patient-level and trial-level verification:

- **Patient-level** (current): all three attributes present (`trial-id`, `site-id`, `patient-id`) → calls `GET /trials/{trialId}/sites/{siteId}/patients/{patientId}/ledger/verify`
- **Trial-level** (future): only `trial-id` present, `site-id` and `patient-id` omitted → calls `GET /trials/{trialId}/ledger/verify`

This avoids redesigning the component when the trial-level endpoint arrives. Step 8 and audit-trail currently use patient-level mode with hardcoded Patient B1 IDs. The audit-trail page's "Verify All Entries" label is acknowledged as a scope mismatch — it verifies Patient B1's chain, not the full trial. The label will be updated to "Verify Patient B1 Chain" until the trial-level endpoint exists.

### File Structure

```
webui/src/components/
  clinical-pi-approval.ts
  clinical-susar-gate.ts
  clinical-merkle-verify.ts
```

### Registration

In `index.ts`, before `loadSite()`:

```typescript
import { ClinicalPiApproval } from "./components/clinical-pi-approval";
import { ClinicalSusarGate } from "./components/clinical-susar-gate";
import { ClinicalMerkleVerify } from "./components/clinical-merkle-verify";

customElements.define("clinical-pi-approval", ClinicalPiApproval);
customElements.define("clinical-susar-gate", ClinicalSusarGate);
customElements.define("clinical-merkle-verify", ClinicalMerkleVerify);
```

### Usage in Step Files

```typescript
// step4-pi-auth.ts — replaces 70-line html() script block
html(`<clinical-pi-approval trial-id="${TRIAL_ID}"></clinical-pi-approval>`)

// step7-resolution.ts — replaces 95-line html() script block
html(`<clinical-susar-gate trial-id="${TRIAL_ID}"></clinical-susar-gate>`)

// step8-proof.ts — replaces 67-line html() script block
html(`<clinical-merkle-verify trial-id="${TRIAL_ID}" site-id="${SITE_B_ID}" patient-id="${PATIENT_B1_ID}"></clinical-merkle-verify>`)

// audit-trail.ts — patient-level verification (relabeled until trial-level endpoint exists)
html(`<clinical-merkle-verify trial-id="${TRIAL_ID}" site-id="${SITE_B_ID}" patient-id="${PATIENT_B1_ID}"></clinical-merkle-verify>`)
```

### Ghost column cleanup

Steps 3 and 4 reference `respondedAt` and `escalatedAt` columns in their deviation tables. `DeviationRow` only has `commandedAt`; `ProtocolDeviation` entity has no `respondedAt` or `escalatedAt` fields. These columns silently render as "—".

Remove the ghost columns from both step 3 and step 4 tables. Adding lifecycle timestamp fields to `ProtocolDeviation` is tracked as a separate issue.

### MutationObserver Removal

Once all four pages are converted, the MutationObserver in `index.ts` (lines 14-27) is deleted. No other pages use `html()` with scripts.

### Styling

Each component renders markup with inline styles matching the current approach. Light DOM means the page's theme CSS also applies.

## Item 3: Playwright Smoke Tests (#102)

### Problem

No automated UI tests exist. The demo UI has 14 pages with data binding, interactive actions, and cross-page navigation. Manual verification is slow and error-prone.

### Design

Playwright tests in `runtime/src/test/playwright/`, with a dedicated `package.json` to avoid bloating the production webui with Playwright's browser binaries (~200-400MB). This follows Maven conventions for test location and keeps test infrastructure separate from production source.

#### Setup

Test directory structure:

```
runtime/src/test/playwright/
  package.json              — @playwright/test dependency only
  playwright.config.ts      — baseURL, webServer, viewport
  tests/
    01-smoke.spec.ts        — page reachability + data binding
    02-navigation.spec.ts   — guided/explore switching, sidebar links
    03-clipping.spec.ts     — viewport overflow checks
    04-actions.spec.ts      — action button flows (step 3→4, step 5→7)
```

Test `package.json`:

```json
{
  "name": "casehub-clinical-e2e",
  "private": true,
  "scripts": {
    "test:e2e": "npx playwright test",
    "test:e2e:headed": "npx playwright test --headed"
  },
  "devDependencies": {
    "@playwright/test": "^1.48.0"
  }
}
```

#### Server Lifecycle and CI Integration

Playwright's `webServer` config handles server lifecycle automatically:

```typescript
// playwright.config.ts
export default defineConfig({
  webServer: {
    command: 'mvn -f ../../../pom.xml quarkus:dev -Dquarkus.http.host=0.0.0.0',
    url: 'http://localhost:8080/q/health',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
  use: {
    baseURL: 'http://localhost:8080',
  },
});
```

- **CI:** `reuseExistingServer: false` → Playwright starts `quarkus:dev`, waits for `/q/health` readiness, runs tests, tears down the server.
- **Developer workflow:** `reuseExistingServer: true` → Playwright reuses a running `quarkus:dev` instance. Start it manually in one terminal, run tests in another.
- **Readiness detection:** `/q/health` (Quarkus SmallRye Health) responds only after the application is fully started and DemoDataSeeder has completed.
- **Timeout:** 120s accommodates Maven dependency resolution + Quarkus startup + data seeding.

#### Test Data Isolation Strategy

Each test run starts with fresh seeded data. The dev profile uses H2 in-memory (`jdbc:h2:mem:clinical`) with `hibernate.database.generation=drop-and-create`. Every Quarkus restart creates a fresh schema, and `DemoDataSeeder` populates deterministic baseline data. This gives test isolation between full test runs automatically.

Within a single test run, tests are ordered to handle state accumulation:

1. **Read-only tests first** (smoke, navigation, clipping) — these assert on seeded data and make no mutations
2. **Action tests last** (actions.spec.ts) — these mutate state (report deviation, approve PI, report AE, approve SUSAR gate) in a defined sequence

#### Test Categories

**1. Page reachability** — navigate to each of 14 pages (8 guided + 6 explore), assert no JS console errors, key content elements visible.

**2. Data binding** — tables have rows, metric cards show values (not NaN/blank), charts render with non-zero dimensions.

**3. No clipping** — key container elements: `scrollWidth <= clientWidth` and `scrollHeight <= clientHeight` at 1440x900 and 1920x1080.

**4. Live action flow — step 3→4** — click "Report CRITICAL Protocol Deviation" (step 3), verify deviations table shows COMMANDED row. Navigate to step 4, click PI approval via the new `<clinical-pi-approval>` component, verify status transitions.

**5. Live action flow — step 5→7** — click "Report Grade 4 Adverse Event" (step 5), verify AE table shows new entry with GRADE_4. Wait for the AE table on step 5 to show `escalationStatus = REQUESTED` (`await expect(row).toContainText('REQUESTED', { timeout: 15_000 })`) — this confirms the engine's async processing (AE escalation → SUSAR oversight case) has completed. Then navigate to step 7, verify `<clinical-susar-gate>` auto-discovers the REQUESTED AE (no sessionStorage dependency), click "Approve SUSAR Determination", verify trust score display and gate approval.

**6. Navigation integrity** — switch between Guided and Explore modes, walk all sidebar links, no dead pages.

#### Not In Scope

Visual regression screenshots, performance benchmarks, mobile viewports.

## Implementation Order

1. #108 — sites endpoint + step 1 UI (unblocks data for smoke tests)
2. #109 — web components (unblocks action flow tests)
3. #102 — Playwright tests (tests the above)

## Deferred (tracked as issues)

- Trial-level Merkle verification endpoint (`GET /trials/{trialId}/ledger/verify`) — audit-trail page currently uses patient-level endpoint; `<clinical-merkle-verify>` is designed for both modes
- `actionButton()` response body surfacing in casehub-pages — would make custom components unnecessary for simple POST-and-display cases (file on casehub-pages repo)
- `respondedAt` / `escalatedAt` timestamp fields on `ProtocolDeviation` entity — enables lifecycle timeline display in steps 3 and 4
