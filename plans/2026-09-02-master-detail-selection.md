# Master-Detail Selection Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #148 — master-detail integration — selection-driven AE and deviation panels
**Issue group:** #148

**Goal:** Replace #122's imperative DOM selection handler with native pages parameterised datasets and selection-change listeners in components.

**Architecture:** Declare detail datasets with `#{selection.<dataset>.id}` URL templates in `datasets.ts`. Pages runtime auto-defers fetch until selection, auto-fetches on selection change, auto-aborts in-flight requests. Data-display tabs use DSL `dataTable()` + `lookup()` bound to parameterised datasets. Action components (`cbr-precedents-panel`, `commitment-lifecycle`, `ae-grade-history`, `ae-regrade`) add `selection-change` listeners in `connectedCallback()` to update their `endpoint` property. The central imperative handler in `index.ts` is deleted.

**Tech Stack:** TypeScript, Lit 3, casehub-pages runtime (parameterised URLs via `#{selection.X.Y}` in `RuntimeContext`), `@casehubio/pages-data`

## Global Constraints

- Frontend only — no backend changes
- `#{selection.<datasetId>.<field>}` is the template syntax (verified in `packages/pages-runtime/src/context-wiring.test.ts:834`)
- Parameterised datasets defer fetch until template resolves, abort in-flight on change (verified in `packages/pages-runtime/src/parameterised-urls.test.ts`)
- `selection-change` event bubbles with `composed: true` from `pages-data-table` to `document`
- `SelectionChangeDetail = { selectedKeys: string[], selectedRows: TypedRow[] }` (from `packages/pages-table/src/types.ts:38`)
- The runtime calls `contextManager.updateSelection(datasetId, typedRowToRecord(row, columns))` on every selection (from `packages/pages-runtime/src/site.ts:720`)
- Trial ID comes from URL params or defaults to `316e3846-4ea7-3b18-a6f7-e01ce6582a69`

---

## Batch 1: Parameterised datasets + data-display tabs

### Task 1: Add parameterised detail datasets and wire data-display tabs

**Files:**
- Modify: `runtime/src/main/webui/src/datasets.ts` — add `detailDatasets()` function
- Modify: `runtime/src/main/webui/src/app.ts` — include detail datasets in page definition
- Modify: `runtime/src/main/webui/src/views/safety-workbench.ts` — replace audit trail + precedents with DSL tables
- Modify: `runtime/src/main/webui/src/views/protocol-workbench.ts` — same
- Delete: `runtime/src/main/webui/src/components/clinical-audit-trail.ts` (use `ide_refactor_safe_delete`)
- Modify: `runtime/src/main/webui/src/index.ts` — remove ClinicalAuditTrail import + registration

**Interfaces:**
- Produces: `detailDatasets(trialId: string): DataSourceBinding[]` — parameterised datasets deferred until selection
- Produces: DSL `dataTable()` components in workbenches bound via `lookup("ae-precedents")` etc.

- [ ] **Step 1: Add `detailDatasets()` to `datasets.ts`**

Add below `trialDatasets()`:

```typescript
export function detailDatasets(trialId: string): DataSourceBinding[] {
  return [
    restDataset("ae-precedents", `/api/trials/${trialId}/adverse-events/#{selection.adverse-events.id}/precedents`),
    restDataset("ae-grade-history", `/api/trials/${trialId}/adverse-events/#{selection.adverse-events.id}/grade-history`),
    restDataset("ae-governance", `/api/trials/${trialId}/adverse-events/#{selection.adverse-events.id}/governance`),
    restDataset("dev-precedents", `/api/trials/${trialId}/deviations/#{selection.deviations.id}/precedents`),
    restDataset("dev-responses", `/api/trials/${trialId}/deviations/#{selection.deviations.id}/responses`),
  ];
}
```

- [ ] **Step 2: Include detail datasets in `app.ts`**

Import `detailDatasets` and add to the datasets array:

```typescript
import { trialsDs, trialDatasets, patientDatasets, detailDatasets } from "./datasets.js";

// In page() options:
datasets: [
  trialsDs,
  ...trialDatasets(trialId),
  ...detailDatasets(trialId),
  ...(siteId && enrollmentId ? patientDatasets(trialId, siteId, enrollmentId) : []),
],
```

- [ ] **Step 3: Replace Safety Workbench data-display tabs**

In `safety-workbench.ts`, replace the Precedents and Audit Trail tabs to use DSL `dataTable()` bound to parameterised datasets. Add `lookup` import (already imported). Keep Grade History and Regrade as Lit components (they have custom rendering / form logic).

Replace the Precedents tab:
```typescript
["Precedents", panel("Similar Past Cases",
  dataTable({
    title: "AE Precedents",
    lookup: lookup("ae-precedents"),
    sortable: true,
    pageSize: 10,
    columns: [
      { id: "similarity" as never, name: "Similarity", expression: '$string($round($number(value))) & "%"' },
      { id: "grade" as never, name: "Grade" },
      { id: "outcome" as never, name: "Outcome" },
      { id: "resolutionTime" as never, name: "Resolution Time" },
      { id: "reportedDate" as never, name: "Reported" },
    ],
  }),
)],
```

Replace the Audit Trail tab:
```typescript
["Audit Trail", panel("Ledger Entries",
  dataTable({
    title: "Audit Trail",
    lookup: lookup("ledger-entries"),
    sortable: true,
    pageSize: 25,
    columns: [
      { id: "occurredAt" as never, name: "Timestamp" },
      { id: "entryType" as never, name: "Type" },
      { id: "actorId" as never, name: "Actor" },
      { id: "subjectId" as never, name: "Subject" },
      { id: "digest" as never, name: "Digest", expression: 'value ? $substring(value, 0, 16) & "..." : ""' },
    ],
    filter: { enabled: true },
  }),
)],
```

Note: the audit trail stays bound to the full `ledger-entries` dataset with filter enabled — users can filter by subject ID using the column filter. This avoids needing a backend endpoint change. The precedents tab uses the parameterised `ae-precedents` dataset which auto-fetches when an AE is selected.

- [ ] **Step 4: Replace Protocol Workbench data-display tabs**

Same pattern for `protocol-workbench.ts`:

Replace Precedents tab:
```typescript
["Precedents", panel("Similar Past Deviations",
  dataTable({
    title: "Deviation Precedents",
    lookup: lookup("dev-precedents"),
    sortable: true,
    pageSize: 10,
    columns: [
      { id: "similarity" as never, name: "Similarity", expression: '$string($round($number(value))) & "%"' },
      { id: "severity" as never, name: "Severity" },
      { id: "outcome" as never, name: "Outcome" },
      { id: "resolutionTime" as never, name: "Resolution Time" },
      { id: "reportedDate" as never, name: "Reported" },
    ],
  }),
)],
```

Replace Audit Trail tab (same as safety — full ledger with filter):
```typescript
["Audit Trail", panel("Ledger Entries",
  dataTable({
    title: "Audit Trail",
    lookup: lookup("ledger-entries"),
    sortable: true,
    pageSize: 25,
    columns: [
      { id: "occurredAt" as never, name: "Timestamp" },
      { id: "entryType" as never, name: "Type" },
      { id: "actorId" as never, name: "Actor" },
      { id: "subjectId" as never, name: "Subject" },
      { id: "digest" as never, name: "Digest", expression: 'value ? $substring(value, 0, 16) & "..." : ""' },
    ],
    filter: { enabled: true },
  }),
)],
```

- [ ] **Step 5: Delete `ClinicalAuditTrail` component**

Use `ide_refactor_safe_delete` on `runtime/src/main/webui/src/components/clinical-audit-trail.ts`.

Remove the import and registration from `index.ts`:
- Delete `import { ClinicalAuditTrail } from "./components/clinical-audit-trail.js";`
- Delete `["clinical-audit-trail", ClinicalAuditTrail],` from the components array

- [ ] **Step 6: Commit**

```
feat(#148): add parameterised detail datasets and wire data-display tabs

Declare ae-precedents, ae-grade-history, ae-governance, dev-precedents,
dev-responses as parameterised datasets with #{selection.<dataset>.id}
URL templates. Pages runtime auto-defers fetch until row selection,
auto-re-fetches on selection change. Replace custom audit trail and
precedents components with DSL dataTable() bound via lookup().
Delete ClinicalAuditTrail component (superseded by parameterised dataset).

Refs #148
```

---

## Batch 2: Selection-aware components + remove imperative handler

### Task 2: Add selection-change listeners to action components, delete imperative handler

**Files:**
- Modify: `runtime/src/main/webui/src/components/cbr-precedents-panel.ts` — add selection listener
- Modify: `runtime/src/main/webui/src/components/commitment-lifecycle.ts` — add selection listener
- Modify: `runtime/src/main/webui/src/components/ae-grade-history.ts` — add selection listener
- Modify: `runtime/src/main/webui/src/components/ae-regrade.ts` — add selection listener
- Modify: `runtime/src/main/webui/src/views/safety-workbench.ts` — add `data-trial-id` + `data-source-dataset` attributes to components
- Modify: `runtime/src/main/webui/src/views/protocol-workbench.ts` — same
- Modify: `runtime/src/main/webui/src/index.ts` — delete imperative handler, keep overview wiring

**Interfaces:**
- Consumes: `selection-change` DOM event with `SelectionChangeDetail` from `pages-data-table`
- Produces: each component self-updates its `endpoint` on selection change

- [ ] **Step 1: Add selection mixin pattern**

Each self-fetching component needs the same listener pattern. Rather than duplicating, create a helper in a shared file:

Create `runtime/src/main/webui/src/selection-bridge.ts`:

```typescript
export interface SelectionBridgeConfig {
  sourceDataset: string;
  buildEndpoint: (trialId: string, entityId: string) => string;
}

export function onTableSelection(
  el: HTMLElement,
  config: SelectionBridgeConfig,
  callback: (entityId: string, trialId: string) => void,
): () => void {
  const handler = (e: Event) => {
    const detail = (e as CustomEvent).detail;
    const rows = detail?.selectedRows ?? [];
    if (!rows.length) return;
    const row = rows[0];
    try {
      const id = row.cell("id");
      if (!id || id.type === "NULL") return;

      const dataset = el.getAttribute("data-source-dataset") ?? "";
      if (!dataset) return;

      // Check if this selection is from the dataset we care about
      // by testing for a field unique to that dataset
      const testCol = dataset === "adverse-events" ? "grade" : "deviationType";
      const test = row.cell(testCol);
      if (!test || test.type === "NULL") return;

      const trialId = el.getAttribute("data-trial-id") ?? "";
      callback(String(id.value), trialId);
    } catch { /* not our dataset */ }
  };
  document.addEventListener("selection-change", handler);
  return () => document.removeEventListener("selection-change", handler);
}
```

- [ ] **Step 2: Update `cbr-precedents-panel` to listen for selection**

In `connectedCallback()`, add:

```typescript
import { onTableSelection } from "../selection-bridge.js";

// In class:
private _unsub?: () => void;

connectedCallback() {
  super.connectedCallback();
  if (this.data) {
    this._precedents = this.data;
  } else if (this.demo) {
    this._precedents = this._getDemoData();
  } else if (this.endpoint) {
    this._fetchPrecedents();
  }

  const sourceDataset = this.getAttribute("data-source-dataset");
  const trialId = this.getAttribute("data-trial-id");
  if (sourceDataset && trialId) {
    this._unsub = onTableSelection(this, {
      sourceDataset,
      buildEndpoint: (tid, eid) =>
        sourceDataset === "adverse-events"
          ? `/api/trials/${tid}/adverse-events/${eid}/precedents`
          : `/api/trials/${tid}/deviations/${eid}/precedents`,
    }, (entityId, tid) => {
      this.endpoint = sourceDataset === "adverse-events"
        ? `/api/trials/${tid}/adverse-events/${entityId}/precedents`
        : `/api/trials/${tid}/deviations/${entityId}/precedents`;
    });
  }
}

disconnectedCallback() {
  super.disconnectedCallback();
  this._unsub?.();
}
```

- [ ] **Step 3: Update `commitment-lifecycle` to listen for selection**

Same pattern — add `onTableSelection` listener that sets `commitmentId` and `endpoint`:

```typescript
// In connectedCallback(), after existing code:
const sourceDataset = this.getAttribute("data-source-dataset");
const trialId = this.getAttribute("data-trial-id");
if (sourceDataset && trialId) {
  this._unsub = onTableSelection(this, {
    sourceDataset,
    buildEndpoint: (tid, eid) => `/api/trials/${tid}/deviations/${eid}/commitment`,
  }, (entityId, tid) => {
    this.commitmentId = entityId;
    this.endpoint = `/api/trials/${tid}/deviations/${entityId}/commitment`;
  });
}
```

- [ ] **Step 4: Update `ae-grade-history` and `ae-regrade`**

Same pattern for both — listen for AE selection, set endpoint:

`ae-grade-history.ts`:
```typescript
// In connectedCallback():
this._unsub = onTableSelection(this, {
  sourceDataset: "adverse-events",
  buildEndpoint: (tid, eid) => `/api/trials/${tid}/adverse-events/${eid}/grade-history`,
}, (entityId, tid) => {
  this.endpoint = `/api/trials/${tid}/adverse-events/${entityId}/grade-history`;
});
```

`ae-regrade.ts`:
```typescript
this._unsub = onTableSelection(this, {
  sourceDataset: "adverse-events",
  buildEndpoint: (tid, eid) => `/api/trials/${tid}/adverse-events/${eid}/regrade`,
}, (entityId, tid) => {
  this.endpoint = `/api/trials/${tid}/adverse-events/${entityId}/regrade`;
});
```

- [ ] **Step 5: Add `data-trial-id` and `data-source-dataset` attributes to component HTML in workbenches**

In `safety-workbench.ts`, update the `html()` blocks:

```typescript
html(`<blocks-approval-gate id="susar-gate"
  data-trial-id="${trialId}"
  data-source-dataset="adverse-events"
  prompt="Review SUSAR determination for this adverse event"
  context-text="Grade 4+ unexpected suspected adverse reaction"
></blocks-approval-gate>`),

html('<cbr-precedents-panel id="ae-precedents" data-trial-id="' + trialId + '" data-source-dataset="adverse-events" empty-message="No similar adverse events found in case memory"></cbr-precedents-panel>'),

html('<clinical-ae-grade-history id="ae-grade-history" data-trial-id="' + trialId + '" data-source-dataset="adverse-events"></clinical-ae-grade-history>'),

html('<clinical-ae-regrade id="ae-regrade" data-trial-id="' + trialId + '" data-source-dataset="adverse-events"></clinical-ae-regrade>'),
```

This requires `trialId` to be passed into `safetyWorkbench()`. Change the function signature:

```typescript
export function safetyWorkbench(trialId: string): Component {
```

In `protocol-workbench.ts`, same pattern with `data-source-dataset="deviations"`:

```typescript
export function protocolWorkbench(trialId: string): Component {
```

Update `app.ts` to pass `trialId` to both workbench functions.

- [ ] **Step 6: Delete the imperative handler from `index.ts`**

Remove these functions entirely:
- `configureWorkbenchSelection()`
- `updateSafetyWorkbench()`
- `updateProtocolWorkbench()`
- `esc()`

Remove the `configureWorkbenchSelection()` call from the `loadSite` callback.

Keep `DEFAULT_TRIAL_ID` and `trialId` — still needed for work-item-inbox configuration.

The overview tabs keep their `html('<div id="ae-overview">...')` placeholder. The overview imperative update was a convenience from #122 — without it, users see the placeholder until they click a row. We can add a selection-driven overview component in a follow-up, or use a DSL table bound to the AE dataset with single-row mode. For now, the overview tab is the least critical since all data is visible in the table columns.

- [ ] **Step 7: Commit**

```
feat(#148): selection-aware components + delete imperative handler

Add selection-bridge.ts helper. Components (cbr-precedents-panel,
commitment-lifecycle, ae-grade-history, ae-regrade) listen for
selection-change on document and self-update their endpoint.
data-trial-id and data-source-dataset attributes carry context.
Delete configureWorkbenchSelection(), updateSafetyWorkbench(),
updateProtocolWorkbench(), esc() from index.ts.

Closes #148
```

---

## References

- [docs/specs/2026-07-07-blocks-ui-migration-design.md] — design spec (§Detail-level datasets, §Safety Workbench, §Protocol Workbench)
- [packages/pages-runtime/src/context-wiring.ts:72-79] — `updateSelection()` writes to `RuntimeContext.selection`
- [packages/pages-runtime/src/context-wiring.test.ts:834] — `#{selection.adverseEvents.id}` template resolution
- [packages/pages-runtime/src/parameterised-urls.test.ts] — auto-deferred fetch, abort-on-change behaviour
- [packages/pages-runtime/src/selection-forwarding.ts] — `pages-selection-changed` dispatch to host-panels
- [packages/pages-component/src/context/types.ts:7] — `RuntimeContext.selection: Record<string, Record<string, unknown>>`
- [packages/pages-table/src/types.ts:38-42] — `SelectionChangeDetail` type
- [packages/pages-runtime/src/site.ts:706-725] — runtime `selection-change` handler
- [GitHub #148] — focal issue
- [GitHub casehubio/casehub-pages#289] — parent epic (closed, APIs delivered)
- [GitHub casehubio/casehub-pages#298, #299, #300] — child issues (all closed)
- [GitHub #122] — prior imperative implementation being replaced
