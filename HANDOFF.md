# Handoff — casehub-clinical
2026-05-31

## What happened this session

Closed #48 (observer fallback for `AeEscalationListener` + `IrbDecisionListener`). Both listeners
now have the double try/catch + REQUIRES_NEW pattern. Key design refinement from the previous
session's pattern: a `ledgerWritten`/`ledgerDecisionWritten` boolean flag set after the critical
ledger write and before `fireAsync` — without it, a caught exception after the ledger write
commits the outer TX normally, and the REQUIRES_NEW failure entry also commits, double-recording
the event. Context resolution moved outside the try block in `AeEscalationListener` (matching
the IRB structure) so `enrollmentId` is always non-null in the catch. Code review caught double
`clock.instant()` in `AeEscalationLedgerWriter.writeObserverFailureEntry` (occurredAt ≠ completedAt).

Build verification surfaced a `CaseLifecycleEvent` API change in the engine snapshot — `tenancyId`
added as 2nd parameter. Fixed in `AeEscalationListenerTest`. Three lifecycle tests now fail due to
a separate `PlanExecutionContext` constructor change — tracked as #53, not caused by this branch.

- **Garden:** GE-20260531-ed2f7a (caught @ObservesAsync try/catch enables outer TX commit → REQUIRES_NEW also commits → double-recording)
- **Protocols:** PP-20260530-49856c revised (ledgerWritten flag requirement added), PP-20260531-11724b new (actorRole = successRole + "-observer-failed")
- **Blog:** `2026-05-31-mdp01-when-caught-exceptions-commit.md`

## Current state

- **Project repo:** `main` — pushed to casehubio/clinical and mdproctor/clinical
- **Workspace:** `main`
- **Backup branch:** `backup/pre-squash-main-20260531` in project repo (safe to delete after 14 days)

## Outstanding (filed, not yet done)

- **casehubio/clinical#49** — early-return audit gap (missing-config paths leave no ledger trace) · S · Med
- **casehubio/clinical#52** — test coverage gaps: enrollmentId null path after markCompleted; IrbDecisionListenerTest missing approvalId assertion · XS · Low
- **casehubio/clinical#53** — engine API incompatibility: PlanExecutionContext constructor changed; AeEscalationLifecycleTest + DsmbRollupTest + IrbGateLifecycleTest failing · S · Med

## Hygiene

*Unchanged — `git show HEAD~1:HANDOFF.md` (Hygiene section)*

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #53 | Fix PlanExecutionContext incompatibility — lifecycle tests failing | S | Med | Needs engine#? — check what changed |
| #49 | Early-return audit gap in notification listeners | S | Med | Missing-config paths leave no trace |
| #52 | Test coverage gaps — two small additions | XS | Low | Batch into #49 session if convenient |
| Layer 7 | Trust routing | XL | High | Blocked on engine#387 |
