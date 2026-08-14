## D1: How to construct the stages array

**Choice:** Derive from commitment snapshot + deviation state
**Alternatives:**
- Query commitment transition history — more accurate but requires new qhorus API or fragile message-as-proxy heuristics
**Rationale:** Commitment record already has createdAt, acknowledgedAt, resolvedAt timestamps. Derivation is deterministic from a single record — no new qhorus infrastructure needed.
**Trade-offs:** ACKNOWLEDGED may be skipped (PI goes straight to FULFILLED); derivation must handle missing intermediate states gracefully.
**Exploration:** quick
**Status:** captured

## D2: Where does the qhorus query + translation logic live?

**Choice:** Dedicated service class (`CommitmentLifecycleService`)
**Alternatives:**
- Inline in TrialDashboardResource — simpler but resource already 900 lines with many inner records; adds two new injected dependencies to an already busy class
**Rationale:** Keeps the resource thin, translation logic unit-testable with mocked qhorus dependencies. Clean separation of REST concerns from qhorus integration.
**Trade-offs:** One more class, but small and focused.
**Exploration:** quick
**Status:** captured

## D3: How to fetch channel messages

**Choice:** `MessageService.history(channelId, 0, limit)` — channel-scoped history
**Alternatives:**
- `MessageService.findAllByCorrelationId(deviationId)` — simpler query but mixes messages across channels and loses conversation ordering
**Rationale:** The component displays a conversation in the PI oversight channel. Channel-scoped history preserves ordering and scope. Filter out EVENT-type messages (qhorus infrastructure, not user-visible).
**Trade-offs:** Requires channel lookup first (`channelService.findByName`), but the channel name is already on the deviation entity.
**Exploration:** quick
**Status:** captured
