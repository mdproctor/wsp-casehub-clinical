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

`SiteResource` at `@Path("/trials/{trialId}/sites")` has POST and GET-by-id, but no list endpoint. Step 1 (Trial Overview) has TODO'd bar chart and sites table that need a "sites" dataset.

### Design

Add `GET /` to `SiteResource` returning enriched `SiteRow` records:

```java
public record SiteRow(UUID id, String investigatorId, String status,
                      long enrolledCount, long adverseEventCount,
                      long deviationCount) {}
```

The method queries `PatientEnrollment`, `AdverseEvent`, and `ProtocolDeviation` counts per site, following the same tenant-scoped pattern as `TrialDashboardResource`.

### UI Changes

- Add `sitesDs = dataset("sites", "/trials/${TRIAL_ID}/sites")` to `datasets.ts`
- Uncomment and wire up the bar chart and sites table in `step1-overview.ts`
- The bar chart groups by `investigatorId` (the site label — `TrialSite` has no separate name field) with `enrolledCount` as the value axis
- The sites table shows `investigatorId`, `status`, `enrolledCount`, `adverseEventCount`
- The step 1 TODO referenced `siteName` — this field does not exist on `TrialSite`; use `investigatorId` instead

## Item 2: Custom Web Components (#109)

### Problem

Steps 4, 7, 8, and audit-trail use `html()` with inline `<script>` tags. Browsers don't execute scripts added via `innerHTML`. A MutationObserver hack in `index.ts` re-creates script elements so they run. This is fragile and architecturally wrong — interactive behavior should live in components, not injected scripts.

### Design

Three custom web components replace all four pages' inline scripts:

| Component | Element name | Pages | Behavior |
|-----------|-------------|-------|----------|
| `ClinicalPiApproval` | `<clinical-pi-approval>` | Step 4 | Fetch deviations → find COMMANDED → show button → POST approve → display result |
| `ClinicalSusarGate` | `<clinical-susar-gate>` | Step 7 | Read AE ID from sessionStorage → idempotency check → POST gate approve → display trust score delta, attestation |
| `ClinicalMerkleVerify` | `<clinical-merkle-verify>` | Step 8, Audit-trail | Show verify button → GET Merkle verify → display VERIFIED/FAILED + Merkle root |

### Component Conventions

- **Light DOM** — no `attachShadow()`. Inherits page styles, normal event bubbling, no accessibility complications.
- **Attribute-driven** — configuration via HTML attributes (`trial-id`, `site-id`, `patient-id`). Mapped via static `observedAttributes`.
- **One-time init guard** — `connectedCallback` checks `this._initialized` to prevent duplicate setup if the element is moved in the DOM.
- **Cleanup** — `disconnectedCallback` aborts in-flight fetches via `AbortController`.
- **No framework** — vanilla `HTMLElement` subclasses, bundled by existing esbuild.

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

// audit-trail.ts — replaces 72-line html() script block (reuses same component)
html(`<clinical-merkle-verify trial-id="${TRIAL_ID}" site-id="${SITE_B_ID}" patient-id="${PATIENT_B1_ID}"></clinical-merkle-verify>`)
```

### MutationObserver Removal

Once all four pages are converted, the MutationObserver in `index.ts` (lines 14-27) is deleted. No other pages use `html()` with scripts.

### Styling

Each component renders markup with inline styles matching the current approach. Light DOM means the page's theme CSS also applies.

## Item 3: Playwright Smoke Tests (#102)

### Problem

No automated UI tests exist. The demo UI has 14 pages with data binding, interactive actions, and cross-page navigation. Manual verification is slow and error-prone.

### Design

Playwright tests in `runtime/src/main/webui/`, running against `mvn quarkus:dev` with seeded data.

#### Setup

- Add `@playwright/test` as devDependency in `webui/package.json`
- `playwright.config.ts` with `baseURL: "http://localhost:8080"`
- npm scripts: `test:e2e` and `test:e2e:headed`

#### Test Structure

```
webui/
  playwright.config.ts
  tests/
    smoke.spec.ts        — page reachability + data binding
    navigation.spec.ts   — guided/explore switching, sidebar links
    actions.spec.ts      — action button flows
    clipping.spec.ts     — viewport overflow checks
```

#### Test Categories

**1. Page reachability** — navigate to each of 14 pages (8 guided + 6 explore), assert no JS console errors, key content elements visible.

**2. Data binding** — tables have rows, metric cards show values (not NaN/blank), charts render with non-zero dimensions.

**3. No clipping** — key container elements: `scrollWidth <= clientWidth` and `scrollHeight <= clientHeight` at 1440x900 and 1920x1080.

**4. Live action flow** — click "Report CRITICAL Protocol Deviation" (step 3), verify deviations table shows COMMANDED row. Navigate to step 4, click PI approval via the new `<clinical-pi-approval>` component, verify status transitions.

**5. Navigation integrity** — switch between Guided and Explore modes, walk all sidebar links, no dead pages.

#### Not In Scope

Visual regression screenshots, performance benchmarks, mobile viewports.

## Implementation Order

1. #108 — sites endpoint + step 1 UI (unblocks data for smoke tests)
2. #109 — web components (unblocks action flow tests)
3. #102 — Playwright tests (tests the above)

## Deferred (tracked as issues)

- Trial-level Merkle verification endpoint (`GET /trials/{trialId}/ledger/verify`) — audit-trail page currently uses patient-level endpoint as workaround
- `actionButton()` response body surfacing in casehub-pages — would make custom components unnecessary for simple cases (file on casehub-pages repo)
