# Handoff — casehub-clinical
2026-05-18

## What happened this session

**Epic 5 close (orphaned):** `epic-protocol-deviation-pi-auth` `.meta` was orphaned on workspace main. Closed cleanly: DESIGN.md created from journal, specs promoted to project, issue #5 closed. Root cause noted: workspace epic branch scaffold commits land on main rather than the epic branch — recurring issue with `/epic` workflow.

**clinical#14 + clinical#15 shipped:** Resolution ledger entries for protocol deviation PI authorisation. Full Merkle chain now records COMMAND → APPROVED/REJECTED/ESCALATED/EXPIRED. Key design decision: `commitmentId` NOT added to ledger entries — the normative link already exists through `ProtocolDeviation.commitmentId`; a foreign key annotation doesn't test the normative hypothesis.

**`DeviationLedgerWriter @ApplicationScoped`:** New component centralising all ledger writes for protocol deviations. Three services write to the same ledger chain; the writer owns `sequenceNumber` computation via `findLatestBySubjectId`. Protocol filed on parent: casehubio/parent#30.

**Pre-existing API drift fixed:** casehub-work `WorkItemCreateRequest` gained new fields (24-param constructor); casehub-qhorus `NormalisedMessage` reverted to 3-param form (qhorus#154 artifact not yet published). Both fixed in `AdverseEventService` and `ClinicalInboundNormaliser`.

**`quarkus.arc.exclude-types` pattern:** ledger SNAPSHOT ships reactive services that veto in JDBC-only test env. Documented in CLAUDE.md and DESIGN.md. Tracked casehubio/clinical#17.

## Current state

- **Project repo:** `main`, 68 tests pass, 1 skipped (`PiResponseListenerIntegrationTest @Disabled` pending qhorus#153)
- **Workspace:** `main`
- Workspace epic branch `epic-deviation-resolution-ledger` retained for 14-day verification (deletion due 2026-06-01)
- Workspace epic branch `epic-multi-site-sub-case` exists with no commits — scaffold only, never used

## Key open items

| Issue | What | Priority |
|-------|------|----------|
| casehubio/qhorus#153 | `MessageReceivedEvent` CDI hook — unblocks `PiResponseListenerIntegrationTest` | Unblocks full chain |
| casehubio/clinical#16 | Remove redundant `commitmentService` calls in `PiResponseListener` | After qhorus#153 |
| casehubio/clinical#18 | `DeviationExpirationJob` REQUIRES_NEW — per-deviation transaction isolation | Pre-existing structural issue |
| casehubio/clinical#6 | IRB gate — CRITICAL deviation escalation consumer (blocked on work#136) | Epic 6 |
| casehubio/parent#30 | Protocol: multi-service LedgerEntry writer pattern | Next parent session |

## What's next

**If qhorus#153 ships:** un-comment `@ObservesAsync` in `PiResponseListener`, remove `@Disabled` from `PiResponseListenerIntegrationTest`, close clinical#16.

**If starting Epic 6 (IRB gate):** `ProtocolDeviationResolvedEvent` with `IRB_REVIEW` already fires. Need `@ObservesAsync ProtocolDeviationResolvedEvent` listener → creates `IrbApproval` WorkItem. Blocked on casehubio/work#136.

## References

- Design: `DESIGN.md` (workspace main)
- Blog: `blog/2026-05-18-mdp01-closing-the-ledger-loop.md`
- Previous handover: `git show HEAD~1:HANDOFF.md`
