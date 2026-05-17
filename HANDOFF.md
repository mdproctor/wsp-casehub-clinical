# Handoff — casehub-clinical
2026-05-17

## What happened this session

**LAYER-LOG.md retroactive work:** Layer 3 `🔲` placeholders filled with actual Epic 5 wiring, gotchas, and pattern steps. Layer table in `casehub-clinical.md` corrected (all 7 layers were showing "pending"; Layers 1/2/4 are complete, Layer 3 now complete). Layer table update tracked as parent#22.

**Epic 5 PI authorisation — implemented and merged to main.** Full qhorus COMMAND lifecycle: per-deviation channel (`clinical/deviation/{id}/pi-oversight`), COMMAND to named PI, Commitment auto-opened via MessageService, 6-state PiApprovalStatus, ProtocolDeviationResolvedEvent for downstream epics (6 and 13). Key classes: `ProtocolDeviationService`, `ClinicalInboundNormaliser`, `PiResponseListener`, `DeviationExpirationJob`, `DeviationResource`. Flyway migration subdirectory restructure (see CLAUDE.md). Merged to main, branch deleted.

**qhorus#154 shipped mid-session (concurrent session, commit 448a631):** `InboundHumanMessage.correlationId` added to qhorus. `ClinicalInboundNormaliser` now threads it through. The explicit `commitmentService.fulfill()/decline()` calls in `PiResponseListener` are now redundant — tracked as clinical#16. Blog entry `2026-05-17-mdp01` describes state before this change.

## Current state

- **Project repo (casehub/clinical):** `main`, 56 tests pass, 1 skipped (`PiResponseListenerIntegrationTest @Disabled` pending qhorus#153)
- **Workspace:** `main`
- Untracked `application.properties` in workspace root — investigate origin, likely noise

## Key open items

| Issue | What | Priority |
|---|---|---|
| casehubio/qhorus#153 | MessageReceivedEvent CDI hook — enables PiResponseListenerIntegrationTest | Unblocks full chain |
| casehubio/clinical#16 | Remove redundant commitmentService calls in PiResponseListener (qhorus#154 now auto-fulfills) | After qhorus#153 |
| casehubio/clinical#13 | Sponsor notification — MAJOR deviation escalation consumer | Epic 13 |
| casehubio/clinical#6 | IRB gate — CRITICAL deviation escalation consumer (blocked on work#136) | Epic 6 |
| casehubio/clinical#14 | Ledger entries on PI response and expiration | Follow-up |
| casehubio/parent#22 | casehub-clinical.md layer table update | Next parent session |
| casehubio/parent#28,#29 | New harness protocols (per-entity channels, InboundNormaliser scope) | Next parent session |

## What's next

If **continuing Epic 5 follow-up**: check if qhorus#153 shipped. If yes: un-comment `@ObservesAsync` in `PiResponseListener`, remove `@Disabled` from `PiResponseListenerIntegrationTest`, bump casehub-qhorus version, then close clinical#16.

If **starting Epic 6** (IRB gate): `ProtocolDeviationResolvedEvent` with `IRB_REVIEW` already fires. Need `@ObservesAsync ProtocolDeviationResolvedEvent` listener → creates IrbApproval WorkItem. Blocked on casehubio/work#136.

## References

- Design spec: `specs/2026-05-15-epic5-pi-authorisation-design.md`
- Design journal: `design/JOURNAL.md`
- Blog: `blog/2026-05-17-mdp01-pi-authorisation-ships.md`
- LAYER-LOG.md: Layer 3 entry complete
- Previous handover: `git show HEAD~1:HANDOFF.md`
