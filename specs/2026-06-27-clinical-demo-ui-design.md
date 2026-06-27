# Clinical Trial Demo UI — Design Spec

**Date:** 2026-06-27
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

## Navigation Structure

Two modes accessible via sidebar toggle:

### Guided Mode (default)

Six narrative steps, each combining a markdown panel (context) with live dashboard components (data):

1. **Trial Overview** — introduce the trial, show enrollment across 3 sites
2. **Meet the AI Agents** — trust scores, governance model, oversight policy
3. **Event: Grade 4 AE Reported** — trigger live action, watch engine react
4. **AI Decision & Governance** — hero layout: "what the AI decided" vs "how the platform governed it"
5. **Resolution & Trust Update** — investigator approval, attestation, Bayesian trust update
6. **The Proof** — Merkle verification, ledger entries, compliance supplements, PROV-O export

### Explore Mode

Six operational dashboard pages for free-form exploration:

- Trial Dashboard — metrics, enrollment chart, recent activity
- Site Detail — per-site enrollment, patients, events
- Adverse Events — all AEs with SLA status, grade, escalation
- Protocol Deviations — all deviations with PI approval status
- Audit Trail — ledger entries with type filter, Merkle verification
- Trust Network — agent trust scores, capabilities, decision history

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

### Step 3: Event — Grade 4 AE Reported

**Narrative:** "A Grade 4 hepatotoxicity event is reported at Site B. Watch what happens: the engine creates a 24h SLA work item, triggers SUSAR evaluation, and routes to a trust-weighted safety agent — all within seconds."

**Components:**
- Action button: "Report Adverse Event" → POST Grade 4 hepatotoxicity on Site B patient
- After trigger (polling at 3s):
  - AE detail card: grade, type, reported time, SLA deadline
  - Status transition: REPORTED → ESCALATED
  - Alert (html workaround): "24h SLA activated"
  - Event sequence as timestamps appear

### Step 4: AI Decision & Governance

**Narrative:** "The SUSAR evaluator assessed this event: Grade 4 + unexpected + suspected = SUSAR criteria met. But the agent can't act alone — CaseHub's ActionRiskClassifier unconditionally gates all safety decisions. A qualified investigator must approve."

**Components (hero layout — side-by-side columns):**
- Left: "What the AI decided" — decision breakdown (input criteria → evaluator output)
- Right: "How the platform governed it" — trust context (selected agent, trust score, selection rationale), gate status (PENDING, regulatory citation)

### Step 5: Resolution & Trust Update

**Narrative:** "The investigator approves the SUSAR determination. CaseHub records the attestation — ENDORSED — which feeds into the agent's Bayesian trust score. Good decisions build trust; bad decisions erode it. The regulatory submission work item is created automatically."

**Components:**
- Action button: "Approve SUSAR Determination" (simulates investigator approval)
- Gate decision: APPROVED with investigator ID and timestamp
- Attestation card: ENDORSED → trust signal
- Trust score before/after (two metric cards)
- Regulatory submission status: IND report created with deadline
- Sponsor notification confirmation

### Step 6: The Proof

**Narrative:** "Every decision is recorded in a tamper-evident Merkle audit trail. Each ledger entry is independently verifiable — no trust required in the platform itself. This is what FDA auditors and EU AI Act compliance officers need."

**Components:**
- Ledger entries table: timestamp, type, actor, summary
- Merkle verification button → calls GET /ledger/verify → shows VERIFIED result
- Inclusion proof detail on row click (hash chain display)
- EU AI Act Art.12 compliance supplement display
- PROV-O export link

---

## Page Designs — Explore Mode

### Trial Dashboard

Metric cards (phase, enrollment, AE count, deviation count), enrollment bar chart by site, recent activity table (last 10 events across all types).

### Site Detail

Per-site view. Enrollment table with patient status, site-level AE count, deviation count, investigator details. Cross-filter: click patient row → filters AE and deviation data below.

### Adverse Events

All AEs across the trial. Table with grade, type, site, patient, reported time, SLA deadline, escalation status, regulatory submission status. Cell-level styling for overdue SLAs (workaround for row-level styling). Sortable by deadline urgency.

### Protocol Deviations

All deviations. Table with type, severity, site, PI approval status, commitment lifecycle state. Status formatted via column expressions.

### Audit Trail

Ledger entries table across the full trial. Filter by entry type dropdown. Each row: timestamp, actor, type, summary. Click → Merkle proof detail. "Verify Chain Integrity" button at top.

### Trust Network

Agent trust overview. Table of agents with capability, trust score, decision count, endorsed/challenged ratio. Metric cards for aggregate trust health. Upgrades to graph visualisation when casehub-pages#41 ships.

---

## Data Architecture

### New REST Endpoints (TrialDashboardResource)

| Endpoint | Returns |
|----------|---------|
| `GET /trials/{trialId}/summary` | Enrollment counts per site, AE count by grade, deviation/amendment counts, agent stats |
| `GET /trials/{trialId}/agents` | Agent list: capability, trust score, dimension, phase, decisions, endorsement ratio |
| `GET /trials/{trialId}/patients` | All patients across all sites (flattened with site context) |
| `GET /trials/{trialId}/adverse-events` | All AEs (flattened, includes computed slaTimeRemaining) |
| `GET /trials/{trialId}/deviations` | All deviations (flattened with site context) |
| `GET /trials/{trialId}/ledger-entries` | Paginated ledger entries with `?type=` filter |

All read-only. Response DTOs in `api` module. `@RolesAllowed` consistent with existing resources.

### Datasets

Each endpoint mapped as a casehub-pages dataset with polling refresh:
- Active scenario components: `refresh: { interval: 3000 }` (3s polling)
- Idle dashboard components: `refresh: { interval: 30000 }` (30s polling)
- Static reference data (agents, policies): no refresh

### Pre-seeded Data (DemoDataSeeder)

`DemoDataSeeder` CDI bean, `@Observes StartupEvent`, guarded by `casehub.clinical.demo.seed-data` config property (true in dev profile only).

**Trial:** ONCO-2024-001, Phase III oncology, sponsor "Meridian Therapeutics"

| Site | Name | Patients | Pre-seeded events |
|------|------|----------|-------------------|
| Site A | Memorial Cancer Center | 4 | 1 completed eligibility screening (CRITERIA_MET), 1 resolved Grade 2 AE with full ledger trail |
| Site B | Johns Hopkins Oncology | 3 | 1 resolved protocol deviation (PI approved, commitment lifecycle complete) |
| Site C | Mayo Clinic Research | 3 | 1 completed protocol amendment (PROCEED) |

Pre-populated trust score history: safety-monitoring agent has several attestations recorded.

### Live Actions

| Trigger | API call | Engine reaction |
|---------|----------|----------------|
| "Report Adverse Event" (Step 3) | POST Grade 4 hepatotoxicity, Site B | 24h SLA WorkItem, SUSAR oversight case, trust-weighted agent selection |
| "Approve SUSAR Determination" (Step 5) | POST to a new demo-specific endpoint that completes the SUSAR oversight gate (calls the engine's gate approval API internally). Not a simulation — the real gate lifecycle fires, attestation is written, trust score updates. The endpoint abstracts the qhorus channel interaction so the UI doesn't need to know channel IDs. | Gate APPROVED, attestation written, trust score updated, IND report created |

---

## Project Structure

### UI (new)

```
runtime/src/main/webui/
├── package.json
├── tsconfig.json
├── index.html
├── src/
│   ├── index.ts
│   ├── dashboard.ts
│   ├── guided/
│   │   ├── step1-overview.ts
│   │   ├── step2-agents.ts
│   │   ├── step3-event.ts
│   │   ├── step4-governance.ts
│   │   ├── step5-resolution.ts
│   │   └── step6-proof.ts
│   ├── explore/
│   │   ├── trial-dashboard.ts
│   │   ├── site-detail.ts
│   │   ├── adverse-events.ts
│   │   ├── deviations.ts
│   │   ├── audit-trail.ts
│   │   └── trust-network.ts
│   ├── datasets.ts
│   ├── lookups.ts
│   ├── theme.ts
│   └── narrative.ts
```

### Java (new)

```
runtime/src/main/java/io/casehub/clinical/
├── resource/TrialDashboardResource.java
├── dto/
│   ├── TrialSummaryDto.java
│   ├── AgentSummaryDto.java
│   ├── PatientListDto.java
│   ├── AdverseEventListDto.java
│   ├── DeviationListDto.java
│   └── LedgerEntryListDto.java
└── service/DemoDataSeeder.java
```

### Maven

- Add `quarkus-quinoa` extension to `runtime/pom.xml`

### Configuration

```properties
quarkus.quinoa.dev-server.port=3000
quarkus.quinoa.build-dir=dist
%dev.casehub.clinical.demo.seed-data=true
casehub.clinical.demo.seed-data=false
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
| #47 | Conditional component visibility | Stack switching |
| #48 | Dynamic content in markdown | html() with JS |
| #49 | Parameterised dataset URLs | Static URLs (single pre-seeded trial) |

**Priority for demo polish:** #39 (badges), #43 (countdown), #46 (action button), #38 (alerts)

---

## Testing Strategy

### Java tests

- **TrialDashboardResource** — `@QuarkusTest` integration tests for all 6 endpoints
- **DemoDataSeeder** — unit test (entity counts), integration test (config guard)

### Automated smoke tests (Playwright)

Location: `runtime/src/test/playwright/`

1. **Page reachability** — all guided steps + explore pages render without JS errors
2. **Data binding** — tables have rows, metrics show values, charts render
3. **No clipping** — scrollWidth <= clientWidth at 1440x900 and 1920x1080
4. **Live action flow** — "Report Adverse Event" → new AE row appears, status transitions
5. **Navigation integrity** — guided/explore mode switching, all sidebar links work

### Manual verification

- Theme: light and dark modes
- Polling: trigger live action, watch updates at 3s intervals
- Full guided walkthrough end-to-end

---

## Issue Tracking

### casehubio/clinical#93 — Demo UI epic

| # | Description | Scale | Complexity | Depends on |
|---|-------------|-------|------------|------------|
| #94 | Quinoa setup + webui scaffold | S | Low | — |
| #95 | DemoDataSeeder | M | Med | — |
| #96 | TrialDashboardResource (6 endpoints) | M | Med | — |
| #97 | Datasets + lookups module | S | Low | #96 |
| #98 | Guided Steps 1-2 | M | Med | #95, #97 |
| #99 | Guided Steps 3-4 | M | High | #98 |
| #100 | Guided Steps 5-6 | M | Med | #99 |
| #101 | Explore mode (6 pages) | M | Med | #97 |
| #102 | Playwright smoke tests | S | Med | #98-#101 |

### Dependency graph

```
#94 (Quinoa setup) ──────────────────────────┐
#95 (DemoDataSeeder) ───────────────────────┐ │
#96 (REST endpoints) ──→ #97 (Datasets) ──┐ │ │
                                           ↓ ↓ ↓
                                    #98 (Steps 1-2)
                                           ↓
                                    #99 (Steps 3-4)
                                           ↓
                                    #100 (Steps 5-6)
                                           ↓
                         #101 (Explore) ──→ #102 (Smoke tests)
```

### casehubio/casehub-pages#50 — Pages capabilities epic

11 child issues (#37-#43, #46-#49). Non-blocking — each improves the demo incrementally as it ships.
