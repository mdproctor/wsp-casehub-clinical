# Session Handover — 2026-07-10

## Last Session

Closed #121 — product UI blocks-ui migration. Replaced the guided demo walkthrough with 4 operational views (Work Queue, Safety Workbench, Protocol Workbench, Operations) using blocks-ui shared components. Built 6 promotion candidate Lit web components for eventual extraction to blocks-ui. Added commitment lifecycle Java endpoint. Migrated from pages-ui `dataset()`/`inlineDataset()` (removed) to `bind()`/`csvSource()` API. Filed cross-repo coordination: blocks-ui#38 (clinical child epic), blocks-ui#35 comment (dual-mode datasets convention), aml#101 (dual-mode dataset proposal).

Design review: 5-round spec review (26 issues, all resolved, $21.67), 3-round code review (17 issues, all resolved). SDD execution: 9 tasks, all passed per-task review + VBC.

Post-merge fixes (038eae7): esbuild `define` for `import.meta.env` (not a Vite project), dataset API migration to `bind()`/`csvSource()`, mock CSV column trimming (pages-ui renders all CSV columns), work-queue demo table fallback, SPA routing config.

## Immediate Next Step

The UI renders but needs visual verification and polish. Start the dev server (`mvn quarkus:dev`) or static server (`python3 -m http.server 3000 --directory runtime/src/main/webui/dist`) and walk through all 4 views. The Playwright MCP is now available for automated browser testing.

Priorities:
1. Visual walkthrough — verify all views render correctly with mock data
2. Fix any layout/styling issues found
3. Wire live mode — test with `VITE_DEMO_MODE=false` against running Quarkus

## What's Left

- **UI polish** — visual verification of all 4 views, layout fixes, responsive behaviour
- **Live mode** — production dataset wiring (`restSource` needs runtime context resolution — may need pages-runtime enhancement)
- **Quarkus dev mode startup** — `Mutiny.SessionFactory bean not found` blocks `mvn quarkus:dev` (pre-existing reactive Hibernate issue, not caused by #121)
- **EngineStateCleaner** — `MemoryPlanItemStore` removed in engine SNAPSHOT, workaround in place (commented out injection)
- `#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- Governance endpoint degraded — WorkerDecisionEntry removed in ledger SNAPSHOT, stubbed with TODO
- DemoDataSeeder SUSAR oversight case start times out — partial demo data

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | UI visual verification + polish | S | Low | Ready — use Playwright MCP |
| — | Live mode dataset wiring | M | Med | Needs pages-runtime investigation |
| #99 | Guided Steps 3-4: AE Event + Governance | M | High | Now "guided overlay" concern — deferred |
| #104 | Guided Steps 3-4: Deviation + PI Auth | M | High | Now "guided overlay" concern — deferred |
| #86 | Wire ProtocolAmendmentAdvisor | M | High | Blocked: engine#101 |
| #78 | CBR over AE history | L | High | Blocked: neocortex#68 |

## Key Files Changed (#121)

```
runtime/src/main/webui/src/
├── index.ts                    # Entry: register 6 Lit components, loadSite()
├── app.ts                      # Sidebar: tree() with 4 views
├── datasets.ts                 # bind()/csvSource() dual-mode datasets
├── views/work-queue.ts         # Table from CSV mock (demo), <work-item-inbox> (live)
├── views/safety-workbench.ts   # AE list + 6 detail tabs (split-pane)
├── views/protocol-workbench.ts # Deviation list + 5 detail tabs (split-pane)
├── views/operations.ts         # 5 dashboard tabs
├── components/                 # 6 promotion candidates for blocks-ui
│   ├── commitment-lifecycle.ts
│   ├── cbr-precedents-panel.ts
│   ├── trust-feedback-display.ts
│   ├── regulatory-compliance-summary.ts
│   ├── gdpr-erasure-action.ts
│   └── sla-breach-policy-indicator.ts
├── stubs/audit-trail-viewer.ts # Stub until blocks-ui#9
└── mock/                       # 10 CSV files for demo mode
```

Java: `TrialDashboardResource.java` — new `GET /api/trials/{trialId}/deviations/{devId}/commitment` endpoint + `CommitmentEndpointTest.java` (3 tests).

## Cross-Repo Coordination

- **blocks-ui#35** — parent migration tracking epic (updated with clinical promotions + overlap watch)
- **blocks-ui#38** — clinical child epic (Phase 1: consume, Phase 2: promote, Phase 3: cleanup)
- **blocks-ui#42** — WorkItemResponse `types` field gap (blocks Work Queue navigation)
- **aml#101** — dual-mode dataset convention (proposed from clinical, filed on AML)
- **pages-data** — cross-repo fix: added `exports` field for subpath imports (committed to pages repo)

## References

- Spec: `specs/2026-07-07-blocks-ui-migration-design.md`
- Plan: `plans/2026-07-07-blocks-ui-migration.md`
- Design review workspace: `~/adr/casehub-clinical/blocks-ui-migration-20260707-163123/`
- Code review workspace: `~/adr/casehub-clinical/blocks-ui-impl-review-20260707-205154/`
- Progress ledger: `.hortora/sdd/progress.md`
