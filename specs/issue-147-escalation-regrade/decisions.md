## D1: Always start a fresh engine case on upgrade regrade

**Choice:** Remove the `engineCaseId != null` guard in `prepareAndMarkForRegrade()`. On any grade upgrade where the policy requires engine-managed escalation, start a fresh case at the new grade — regardless of whether an old case exists or its state (active/completed). The policy SPI is the sole configuration point for grade-based escalation requirements; the regrade mechanism is not separately configurable.

**Alternatives:**
- Re-open/mutate existing engine case context — requires engine API for context mutation on active/completed cases (not supported); re-opening a completed case muddies the audit trail by injecting new requirements into a finished review
- Accept current behavior (signal only, no new case) — creates a compliance gap: Grade 4 DSMB escalation is mandated by `DefaultAdverseEventEscalationPolicy` but never happens when upgrading from Grade 3; violates ICH E6(R3) §5.17 proportional severity response
- Runtime configuration toggle — the `AdverseEventEscalationPolicy` SPI already controls what each grade requires; a separate "skip regrade escalation" toggle would allow contradicting the policy's own requirements, which is a compliance contradiction

**Rationale:** ICH E6(R3) §5.17 and CTCAE v5.0 require escalation proportional to current severity. A Grade 3 safety review does not satisfy the Grade 4 DSMB requirement — they are independent regulatory obligations. Starting a fresh case gives a clean audit trail (each grade has its own case, review, and ledger entries) with no engine API extensions needed. The old case continues to completion if active — harmless documentation, not wasted work.

**Trade-offs:** A human reviewer may complete the old (superseded) Grade 3 safety review while the new Grade 4 case is active. This is minimal overhead — severity upgrades are rare, and the old review still produces valid clinical documentation.

**Implementation consequence — case-ID guarded completion:** `AeEscalationListener` must compare the completing case's ID against the AE's current `engineCaseId`. Superseded case completions are silently ignored (no status update, no downstream event). `AeStatusUpdater.markCompleted()` gains an `expectedCaseId` parameter for atomic TOCTOU-safe discrimination inside its `REQUIRES_NEW` transaction.

**Sources:**
- `AeEscalationCaseService.java:92-96` — the guard being removed
- `AeStatusUpdater.java:35-38` — completion idempotency (needs case-ID matching)
- `AeEscalationListener.java:53` — completion handler (needs case-ID discrimination)
- `DefaultAdverseEventEscalationPolicy.java:24-28` — grade→requirements mapping
- `ae-escalation.yaml` — case definition with `requiresDsmbEscalation` filter
- `2026-07-21-ae-grade-regrading-design.md:209-211` — spec acknowledged the gap but didn't resolve it

**Exploration:** quick
**Status:** captured
