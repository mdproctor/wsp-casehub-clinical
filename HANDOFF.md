# Handoff — casehub-clinical
2026-05-19

## What happened this session

**clinical#18 done** — `DeviationExpirer @ApplicationScoped` with `@Transactional(REQUIRES_NEW)` per deviation. The old single-transaction batch loop with try/catch was structurally broken for JPA exceptions. Now each deviation commits independently. XA config added to production `application.properties` (was test-only — latent gap).

**qhorus#153 shipped** — CDI `MessageReceivedEvent` hook live. Unblocked:
- `PiResponseListener.onMessage(@ObservesAsync MessageReceivedEvent)` — enabled
- `PiResponseListenerIntegrationTest` — fully implemented and passing (was `@Disabled` stub)
- clinical#16 closed — redundant `commitmentService.fulfill()/decline()` calls removed
- `ClinicalInboundNormaliser` — bug fixed: was passing `channel.name()` instead of `raw.correlationId()` as NormalisedMessage param 4
- `CHANNEL_ALLOWED_TYPES` expanded to `QUERY,COMMAND,DONE,DECLINE` — PI oversight channels need DONE/DECLINE for `receiveHumanMessage()` to accept responses

**clinical#19 done** — `Clock` injected into `DeviationLedgerWriter` via `ClinicalClockProducer`. Timestamp assertions now exact (`isEqualTo(FIXED_INSTANT)`) rather than `isNotNull()`.

**clinical#20 done** — `DeviationExpirerIsolationTest` verifies REQUIRES_NEW guarantee: failure on deviation N does not roll back deviation N-1. Uses `@InjectMock CommitmentService` (needed `quarkus-junit5-mockito` dep).

**77 tests passing**, 0 failures.

## Current state

- **Project repo:** `main`, 77 tests pass, 0 skipped (integration test now live)
- **Workspace:** `main`
- Workspace branches with 14-day retention: `epic-deviation-resolution-ledger` (2026-06-01), `epic-multi-site-sub-case` (2026-06-01), `issue-18-deviation-expiration-requires-new` (2026-06-02)

## Key open items

| Issue | What | Priority |
|-------|------|----------|
| casehubio/clinical#6 | IRB gate — CRITICAL deviation consumer (blocked on work#136) | Epic 6 |
| casehubio/clinical#13 | Sponsor notification — MAJOR deviation consumer | After connectors pattern |
| casehubio/clinical#17 | Upstream ledger fix: reactive services need conditional activation | casehub-ledger session |
| casehubio/clinical#18 REQUIRES_NEW | `DeviationExpirationJob` isolation guarantee test (clinical#20) — now done | ✅ done |
| casehubio/parent#30 | Protocol: multi-service LedgerEntry writer pattern | Next parent session |

## What's next

**If starting Epic 6** (IRB gate): `ProtocolDeviationResolvedEvent` with `IRB_REVIEW` fires on CRITICAL approval. Listener → `IrbApproval` WorkItem creation. Blocked on casehubio/work#136.

**If checking qhorus integration**: run `PiResponseListenerIntegrationTest` to confirm full channel-flow chain — report deviation → PI responds via channel → APPROVED/ESCALATED.

## References

- Design: `DESIGN.md` (workspace main)
- Blog: `blog/2026-05-19-mdp01-qhorus153-cascade.md`
- Previous handover: `git show HEAD~1:HANDOFF.md`
