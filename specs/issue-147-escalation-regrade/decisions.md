## D1: Always start a fresh engine case on upgrade regrade

**Choice:** Remove the `engineCaseId != null` guard in `prepareAndMarkForRegrade()`. On any grade upgrade where the policy requires engine-managed escalation, start a fresh case at the new grade — regardless of whether an old case exists or its state (active/completed). The policy SPI is the sole configuration point for grade-based escalation requirements; the regrade mechanism is not separately configurable.

**Alternatives:**
- Re-open/mutate existing engine case context — requires engine API for context mutation on active/completed cases (not supported); re-opening a completed case muddies the audit trail by injecting new requirements into a finished review
- Accept current behavior (signal only, no new case) — creates a gap: Grade 4 DSMB escalation is mandated by `DefaultAdverseEventEscalationPolicy` but never happens when upgrading from Grade 3
- Runtime configuration toggle — the `AdverseEventEscalationPolicy` SPI already controls what each grade requires; a separate "skip regrade escalation" toggle would allow contradicting the policy's own requirements, which is a compliance contradiction

**Rationale:** `DefaultAdverseEventEscalationPolicy` returns different requirements per grade: Grade 3 → senior monitor only (`engineManaged(true, false)`); Grade 4 → senior monitor + DSMB (`engineManaged(true, true)`). A case created with Grade 3 requirements cannot satisfy Grade 4 obligations without context mutation, which the engine doesn't support. Starting a fresh case gives a clean audit trail (each grade has its own case, review, and ledger entries) with no engine API extensions needed. The old case continues to completion if active — its review is recorded in the ledger as a superseded completion. This overrides the previous design decision in `2026-07-21-ae-grade-regrading-design.md:209-211,480` which explicitly treated the existing case as sufficient and tested for "no duplicate."

**Trade-offs:** A human reviewer may complete the old (superseded) Grade 3 safety review while the new Grade 4 case is active. This is minimal overhead — severity upgrades are rare, and the old review still produces valid clinical documentation.

**Implementation consequence — case-ID guarded completion:** `AeStatusUpdater.markCompleted()` returns a `CompletionResult` enum (COMPLETED, SUPERSEDED, ALREADY_COMPLETED, NOT_FOUND) with an `expectedCaseId` parameter for atomic discrimination inside its `REQUIRES_NEW` transaction. Superseded case completions write a ledger entry (ALCOA+ audit trail for the completed review) but skip memory store and `AeEscalationCompletedEvent`. `prepareAndMarkForRegrade()` nulls `engineCaseId` before returning context — closing the TOCTOU window between phase 1 and phase 3 of the three-phase pattern.

**Implementation consequence — grade threshold gate:** `AeGradeChangeEscalationListener` gates on `newGrade >= GRADE_3`. Grade 1→2 upgrades never enter the engine case path (both grades are WorkItem-managed). Without this gate, the regrade path would start a hanging engine case for low-grade upgrades because the policy returns `direct()` with `engineCaseRequired=false` but `prepareAndMarkForRegrade()` ignores `engineCaseRequired()`.

**Sources:**
- `AeEscalationCaseService.java:92-96` — the guard being removed
- `AeStatusUpdater.java:35-38` — completion idempotency (needs case-ID matching)
- `AeEscalationListener.java:53` — completion handler (needs case-ID discrimination)
- `DefaultAdverseEventEscalationPolicy.java:24-28` — grade→requirements mapping
- `ae-escalation.yaml` — case definition with `requiresDsmbEscalation` filter
- `2026-07-21-ae-grade-regrading-design.md:209-211,480` — previous design decision (explicitly "no duplicate"; this D1 overrides it)

**Exploration:** quick
**Status:** captured
