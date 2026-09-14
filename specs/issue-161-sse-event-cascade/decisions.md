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

## D3: Event granularity

**Choice:** Domain steps only — the 7-8 major cascade steps a clinical user cares about
**Alternatives:**
- Full engine detail (binding activations, worker resolution, case lifecycle) — adds noise for domain users, useful only for platform debugging
- Hybrid with toggle — domain steps by default, engine detail behind an expand/filter; adds UI complexity for marginal benefit
**Rationale:** The issue's cascade list (event reported → SLA assigned → agent selected → agent reasoning → gate decision → trust update → Merkle sealed) maps directly to clinical user intent. Engine internals are plumbing. eventChronologyStrategy already supports category-based filtering if engine detail is needed later.
**Trade-offs:** Platform developers can't debug engine behavior from the cascade view — they'd use engine event logs for that
**Sources:** Issue #161 cascade list, `eventChronologyStrategy` filterCategories support
**Exploration:** quick
**Status:** captured

## D4: Timeline rendering model

**Choice:** Show full expected cascade upfront with pending nodes — transition each node from pending → active → completed as events arrive
**Alternatives:**
- Event-only stream — append nodes as events arrive, no prediction of future steps; simpler but users can't see where in the cascade they are or what's coming
**Rationale:** The orchestration structure is the value — users need to see the full path and their position in it. EventTimelineNode.status already supports completed/active/pending/failed/skipped, so the infrastructure fits directly.
**Trade-offs:** Requires knowing the expected cascade shape per AE grade/type upfront — different grades trigger different paths (Grade 1-2 skip SUSAR, Grade 3+ trigger escalation cases). The cascade template must be grade-aware.
**Sources:** `EventTimelineNode` status enum, issue #161 "visual state (pending → active → resolved) with transitions"
**Depends on:** D3 (domain steps define the node set)
**Exploration:** quick
**Status:** captured
