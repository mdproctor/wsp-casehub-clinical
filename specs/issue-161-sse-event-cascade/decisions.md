## D1: Push mechanism

**Choice:** Reuse existing `EventBroadcaster` + `EventConnection` WebSocket infrastructure
**Alternatives:**
- JAX-RS SSE endpoint — simpler but creates a parallel push channel, no topic filtering, no sequence tracking
- Polling endpoint — simplest but defeats the "watch it happen" goal
**Rationale:** Platform already has production-ready WebSocket push with topic subscriptions, sequence tracking, gap detection, and reconnection. Building a separate SSE endpoint duplicates infrastructure.
**Trade-offs:** None meaningful — the WebSocket path is strictly superior for this use case
**Sources:** `casehub-pages-push` EventBroadcaster, `pages-data` EventConnection, `pages-data` SSEManager
**Exploration:** quick
**Status:** captured

## D2: UI placement

**Choice:** New "Live Cascade" tab in the safety workbench, activated on AE selection
**Alternatives:**
- Separate top-level view under Review — more screen real estate but disconnects cascade from AE context
- Replace/enhance Audit Trail tab — conceptually related but audit trail is retrospective (ledger entries) while cascade is live orchestration
**Rationale:** Users are already in the safety workbench looking at AE details. Watching the cascade unfold for the selected AE is a natural extension of the existing flow.
**Trade-offs:** Tab count grows (now 9) — manageable but worth watching
**Sources:** `safety-workbench.ts` existing tab structure, issue #161 UI integration requirements
**Exploration:** quick
**Depends on:** D1 (push mechanism determines what data feeds the tab)
**Status:** captured
