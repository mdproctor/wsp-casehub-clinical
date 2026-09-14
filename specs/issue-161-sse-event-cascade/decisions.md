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
