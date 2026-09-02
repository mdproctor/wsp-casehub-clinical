# Workbench Row Selection & Detail-Tab Data Binding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #122 — Workbench row selection and detail-tab data binding
**Issue group:** #122

**Goal:** Wire row selection in Safety and Protocol workbenches so that clicking a row populates all detail tabs with entity-scoped data.

**Architecture:** Enable `selection: "single"` on both `dataTable()` DSL components. Listen for the native `selection-change` DOM event in `index.ts` after `loadSite()`. Discriminate AE vs deviation rows by field presence. Update component properties imperatively via DOM queries (same pattern as existing `configureWorkItemInbox()`). Create a new `<clinical-audit-trail>` Lit component for entity-scoped ledger filtering.

**Tech Stack:** TypeScript, Lit 3, casehub-pages DSL (`@casehubio/pages-ui`), `@casehubio/pages-data` events

## Global Constraints

- Frontend only — no backend changes, no Flyway migrations
- Components use `pages-ui-tokens` CSS custom properties for styling
- Data table selection mode: `"single"` (from `DataTableProps.selection`)
- `selection-change` is the native DOM event emitted by `pages-data-table` (see `packages/pages-runtime/src/site.ts:706`)
- `blocks-approval-gate` is the registered tag name for the approval gate component (`@customElement('blocks-approval-gate')`)
- Trial ID comes from URL params or defaults to `316e3846-4ea7-3b18-a6f7-e01ce6582a69`
- Self-fetching components (`cbr-precedents-panel`, `commitment-lifecycle`, `ae-grade-history`, `ae-regrade`) re-fetch automatically when their `endpoint` property changes

---

## Batch 1: Selection infrastructure + audit trail component

### Task 1: Create `<clinical-audit-trail>` component, enable table selection, wire selection handler

**Files:**
- Create: `runtime/src/main/webui/src/components/clinical-audit-trail.ts`
- Modify: `runtime/src/main/webui/src/views/safety-workbench.ts`
- Modify: `runtime/src/main/webui/src/views/protocol-workbench.ts`
- Modify: `runtime/src/main/webui/src/index.ts`

**Interfaces:**
- Produces: `ClinicalAuditTrail` — Lit component with `trialId: string`, `subjectId: string` properties; fetches and renders filtered ledger entries
- Produces: `configureWorkbenchSelection(trialId: string): void` — called after `loadSite()`, registers `selection-change` listener
- Produces: `SelectionChangeDetail` type usage — `{ selectedKeys: string[], selectedRows: TypedRow[] }`

- [ ] **Step 1: Create `ClinicalAuditTrail` component**

```typescript
// runtime/src/main/webui/src/components/clinical-audit-trail.ts
import { LitElement, html, css } from "lit";
import { property, state } from "lit/decorators.js";

interface LedgerRow {
  readonly occurredAt: string;
  readonly entryType: string;
  readonly actorId: string;
  readonly subjectId: string;
  readonly digest: string;
}

export class ClinicalAuditTrail extends LitElement {
  static styles = css`
    :host { display: block; font-family: var(--pages-font-family, sans-serif); }
    .empty { color: var(--pages-neutral-9, #95a5a6); font-style: italic; padding: var(--pages-space-4, 1rem); }
    table { width: 100%; border-collapse: collapse; font-size: 14px; }
    th { background: var(--pages-neutral-3, #ecf0f1); padding: var(--pages-space-3, 0.75rem); text-align: left; font-weight: 600; border-bottom: 2px solid var(--pages-neutral-6, #bdc3c7); }
    td { padding: var(--pages-space-3, 0.75rem); border-bottom: 1px solid var(--pages-neutral-4, #eee); }
    tr:hover { background: var(--pages-neutral-2, #f8f9fa); }
  `;

  @property({ attribute: "trial-id" }) trialId = "";
  @property({ attribute: "subject-id" }) subjectId = "";
  @state() private _entries: LedgerRow[] = [];
  @state() private _loading = false;
  @state() private _error = "";

  updated(changed: Map<string, unknown>) {
    if ((changed.has("trialId") || changed.has("subjectId")) && this.trialId && this.subjectId) {
      this._fetch();
    }
  }

  private async _fetch() {
    this._loading = true;
    this._error = "";
    try {
      const res = await fetch(`/api/trials/${this.trialId}/ledger-entries`);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const all: LedgerRow[] = await res.json();
      this._entries = all.filter(e => e.subjectId === this.subjectId);
    } catch (e) {
      this._error = e instanceof Error ? e.message : "Failed to load";
    } finally {
      this._loading = false;
    }
  }

  render() {
    if (!this.subjectId) return html`<div class="empty">Select an entity to view its audit trail.</div>`;
    if (this._loading) return html`<div class="empty">Loading audit trail...</div>`;
    if (this._error) return html`<div class="empty">Unable to load audit trail.</div>`;
    if (!this._entries.length) return html`<div class="empty">No ledger entries for this entity.</div>`;
    return html`
      <table>
        <thead><tr><th>Timestamp</th><th>Type</th><th>Actor</th><th>Subject</th><th>Digest</th></tr></thead>
        <tbody>
          ${this._entries.map(e => html`
            <tr>
              <td>${e.occurredAt}</td>
              <td>${e.entryType}</td>
              <td>${e.actorId}</td>
              <td>${e.subjectId}</td>
              <td>${e.digest ? e.digest.substring(0, 16) + "..." : ""}</td>
            </tr>
          `)}
        </tbody>
      </table>
    `;
  }
}
```

- [ ] **Step 2: Register `ClinicalAuditTrail` in `index.ts`**

Add to the `components` array in `index.ts`:

```typescript
import { ClinicalAuditTrail } from "./components/clinical-audit-trail.js";

// Add to components array:
["clinical-audit-trail", ClinicalAuditTrail],
```

- [ ] **Step 3: Enable selection on both data tables**

In `safety-workbench.ts`, add `selection: "single"` to the `dataTable()` config:

```typescript
const aeTable = dataTable({
    title: "Adverse Events",
    lookup: lookup("adverse-events"),
    sortable: true,
    pageSize: 25,
    selection: "single",
    // ... columns and rowStyle unchanged
```

In `protocol-workbench.ts`, same change:

```typescript
const deviationTable = dataTable({
    title: "Protocol Deviations",
    lookup: lookup("deviations"),
    sortable: true,
    pageSize: 25,
    selection: "single",
    // ... columns and rowStyle unchanged
```

- [ ] **Step 4: Update Safety Workbench detail tabs with identifiable components**

Replace the static detail tabs in `safety-workbench.ts`. Key changes:
- Overview: replace `markdown(...)` with `html(...)` containing an `#ae-overview` container
- SUSAR: fix tag to `blocks-approval-gate`, add `id="susar-gate"`
- Precedents: add `id="ae-precedents"`
- Audit Trail: replace `auditTrailStub(...)` with `<clinical-audit-trail id="ae-audit-trail">`
- Grade History: add `id="ae-grade-history"`
- Regrade: add `id="ae-regrade"`

```typescript
const detailTabs = tabs(
    ["Overview", panel("AE Overview",
      html('<div id="ae-overview"><p style="color: var(--pages-neutral-9); font-style: italic; padding: 1rem;">Select an adverse event from the list to view details.</p></div>'),
    )],
    ["SUSAR Evaluation", panel("SUSAR Evaluation",
      html(`<blocks-approval-gate id="susar-gate"
        prompt="Review SUSAR determination for this adverse event"
        context-text="Grade 4+ unexpected suspected adverse reaction — SUSAR criteria evaluation"
      ></blocks-approval-gate>`),
    )],
    ["Trust & Attestation", panel("Trust Feedback",
      html('<trust-feedback-display id="ae-trust-feedback"></trust-feedback-display>'),
    )],
    ["Regulatory", panel("Regulatory Status",
      html('<sla-breach-policy-indicator id="ae-sla-breach"></sla-breach-policy-indicator>'),
    )],
    ["Precedents", panel("Similar Past Cases",
      html('<cbr-precedents-panel id="ae-precedents" empty-message="No similar adverse events found in case memory"></cbr-precedents-panel>'),
    )],
    ["Audit Trail", panel("Ledger Entries",
      html('<clinical-audit-trail id="ae-audit-trail"></clinical-audit-trail>'),
    )],
    ["Grade History", panel("Grade History",
      html('<clinical-ae-grade-history id="ae-grade-history"></clinical-ae-grade-history>'),
    )],
    ["Regrade", panel("Regrade Assessment",
      html('<clinical-ae-regrade id="ae-regrade"></clinical-ae-regrade>'),
    )],
  );
```

Remove the `auditTrailStub` import if no longer used.

- [ ] **Step 5: Update Protocol Workbench detail tabs with identifiable components**

Same pattern for `protocol-workbench.ts`:

```typescript
const detailTabs = tabs(
    ["Overview", panel("Deviation Overview",
      html('<div id="dev-overview"><p style="color: var(--pages-neutral-9); font-style: italic; padding: 1rem;">Select a protocol deviation from the list to view details.</p></div>'),
    )],
    ["PI Commitment", panel("PI Commitment Lifecycle",
      html('<commitment-lifecycle id="dev-commitment"></commitment-lifecycle>'),
    )],
    ["IRB Review", panel("IRB Review",
      html(`<blocks-approval-gate id="irb-gate"
        prompt="Review protocol deviation for IRB approval"
        context-text="Protocol deviation requires ethics committee review — 72h deadline"
      ></blocks-approval-gate>`),
    )],
    ["Precedents", panel("Similar Past Deviations",
      html('<cbr-precedents-panel id="dev-precedents" empty-message="No similar deviations found in case memory"></cbr-precedents-panel>'),
    )],
    ["Audit Trail", panel("Ledger Entries",
      html('<clinical-audit-trail id="dev-audit-trail"></clinical-audit-trail>'),
    )],
  );
```

Remove the `auditTrailStub` import.

- [ ] **Step 6: Create the selection handler in `index.ts`**

Add after `configureWorkItemInbox()`:

```typescript
function configureWorkbenchSelection(trialId: string) {
  document.addEventListener("selection-change", (e: Event) => {
    const detail = (e as CustomEvent).detail;
    const rows = detail?.selectedRows ?? [];
    if (!rows.length) return;
    const row = rows[0];

    const cells: Record<string, string> = {};
    try {
      for (const col of ["id", "grade", "eventType", "deviationType", "severity"]) {
        const cell = row.cell(col);
        if (cell && cell.type !== "NULL") cells[col] = String(cell.value);
      }
    } catch { /* column not present */ }

    if (cells.grade) {
      updateSafetyWorkbench(trialId, cells.id ?? "", cells);
    } else if (cells.deviationType) {
      updateProtocolWorkbench(trialId, cells.id ?? "", cells);
    }
  });
}

function updateSafetyWorkbench(trialId: string, aeId: string, data: Record<string, string>) {
  if (!aeId) return;

  const overview = document.getElementById("ae-overview");
  if (overview) {
    overview.innerHTML = `
      <dl style="display:grid; grid-template-columns: max-content 1fr; gap: 0.5rem 1rem; padding: 1rem; margin: 0;">
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Grade</dt><dd>${data.grade ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Event Type</dt><dd>${data.eventType ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Patient</dt><dd>${data.patientId ? data.patientId.substring(0, 8) + "..." : "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Site</dt><dd>${data.siteName ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Escalation</dt><dd>${data.escalationStatus ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">IND Status</dt><dd>${data.regulatorySubmissionStatus ?? "—"}</dd>
      </dl>`;
  }

  const susarGate = document.getElementById("susar-gate") as any;
  if (susarGate) {
    susarGate.endpoint = `/api/trials/${trialId}/adverse-events/${aeId}/governance`;
    susarGate.setAttribute("gate-id", aeId);
  }

  const precedents = document.getElementById("ae-precedents") as any;
  if (precedents) precedents.endpoint = `/api/trials/${trialId}/adverse-events/${aeId}/precedents`;

  const auditTrail = document.getElementById("ae-audit-trail") as any;
  if (auditTrail) {
    auditTrail.setAttribute("trial-id", trialId);
    auditTrail.setAttribute("subject-id", aeId);
  }

  const gradeHistory = document.getElementById("ae-grade-history") as any;
  if (gradeHistory) gradeHistory.endpoint = `/api/trials/${trialId}/adverse-events/${aeId}/grade-history`;

  const regrade = document.getElementById("ae-regrade") as any;
  if (regrade) regrade.endpoint = `/api/trials/${trialId}/adverse-events/${aeId}/regrade`;
}

function updateProtocolWorkbench(trialId: string, devId: string, data: Record<string, string>) {
  if (!devId) return;

  const overview = document.getElementById("dev-overview");
  if (overview) {
    overview.innerHTML = `
      <dl style="display:grid; grid-template-columns: max-content 1fr; gap: 0.5rem 1rem; padding: 1rem; margin: 0;">
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Type</dt><dd>${data.deviationType ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Severity</dt><dd>${data.severity ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Site</dt><dd>${data.siteName ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">PI Approval</dt><dd>${data.piApprovalStatus ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">IRB Decision</dt><dd>${data.irbDecision ?? "—"}</dd>
        <dt style="font-weight:600; color: var(--pages-neutral-11);">Reported</dt><dd>${data.reportedAt ? data.reportedAt.substring(0, 10) : "—"}</dd>
      </dl>`;
  }

  const commitment = document.getElementById("dev-commitment") as any;
  if (commitment) {
    commitment.commitmentId = devId;
    commitment.endpoint = `/api/trials/${trialId}/deviations/${devId}/commitment`;
  }

  const irbGate = document.getElementById("irb-gate") as any;
  if (irbGate) {
    irbGate.endpoint = `/api/trials/${trialId}/deviations/${devId}/irb-gate`;
    irbGate.setAttribute("gate-id", devId);
  }

  const precedents = document.getElementById("dev-precedents") as any;
  if (precedents) precedents.endpoint = `/api/trials/${trialId}/deviations/${devId}/precedents`;

  const auditTrail = document.getElementById("dev-audit-trail") as any;
  if (auditTrail) {
    auditTrail.setAttribute("trial-id", trialId);
    auditTrail.setAttribute("subject-id", devId);
  }
}
```

- [ ] **Step 7: Wire `configureWorkbenchSelection` call in `loadSite` callback**

In `index.ts`, update the `loadSite` callback:

```typescript
loadSite(container, app).then(() => {
  configureWorkItemInbox();
  configureWorkbenchSelection(trialId);
}).catch(...);
```

This requires `trialId` to be accessible. Move the `trialId` resolution from `app.ts` to `index.ts` (or export it from `app.ts`).

- [ ] **Step 8: Extract more cell values in the selection handler**

Update the cell extraction loop to include all fields needed by the overview tabs:

```typescript
for (const col of ["id", "grade", "eventType", "patientId", "siteName",
    "slaTimeRemainingHours", "escalationStatus", "regulatorySubmissionStatus",
    "deviationType", "severity", "piApprovalStatus", "irbDecision", "reportedAt"]) {
```

- [ ] **Step 9: Compile and verify**

Run: `mvn compile -pl runtime --batch-mode` (compiles webui via quinoa)

If quinoa is disabled, build the webui directly:
```bash
cd runtime/src/main/webui && npm run build
```

- [ ] **Step 10: Commit**

```bash
git add runtime/src/main/webui/src/
git commit -m "feat(#122): wire workbench row selection and detail-tab data binding

Enable selection: 'single' on both data tables. Add selection-change
handler that discriminates AE vs deviation rows and updates detail
tab components. Create <clinical-audit-trail> Lit component for
entity-scoped ledger filtering. Replace static markdown overviews
with data-driven display.

Refs #122"
```

---

## References

- [docs/specs/2026-07-07-blocks-ui-migration-design.md] — design spec (§Detail-level datasets, §Safety Workbench, §Protocol Workbench)
- [runtime/src/main/webui/src/views/safety-workbench.ts] — current safety workbench layout
- [runtime/src/main/webui/src/views/protocol-workbench.ts] — current protocol workbench layout
- [runtime/src/main/webui/src/index.ts] — app entry point, existing configureWorkItemInbox pattern
- [runtime/src/main/webui/src/datasets.ts] — dataset wiring, trialId resolution
- [packages/pages-runtime/src/site.ts:706] — selection-change event handling in pages runtime
- [packages/pages-table/src/types.ts:38-42] — SelectionChangeDetail type definition
- [packages/pages-component/src/model/displayer-types.ts:103] — selection property on DataTableProps
- [components/approval-gate/src/approval-gate.ts:51-60] — blocks-approval-gate properties (gate-id, endpoint)
- [GitHub #122] — focal issue
