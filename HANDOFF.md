# Handoff — casehub-clinical
2026-05-22

## What happened this session

**issue-24-minor-cleanups closed** — three issues in one branch:
- #24: `TestSlackConnector` static → `@Singleton` instance (`reset()`/`sent()`/`setShouldThrow()`). Key finding: `@ApplicationScoped` CDI proxies swallow field access — `@Singleton` bypasses the proxy. `CopyOnWriteArrayList` retained for `@ObservesAsync` cross-thread safety.
- #25: `SponsorNotificationListener` log split (trial-not-found vs incomplete-config), `buildTitle()` uses `req.severity().name()` not hardcoded MAJOR, two partial-config test cases added.
- #15: `AdverseEventLedgerWriter` extracted from `AdverseEventService` — owns `nextSequenceNumber()` via `findLatestBySubjectId()`, follows `DeviationLedgerWriter` pattern. Ready for Epic 4 resolution entries.
- Unplanned: `ActorType` moved to `io.casehub.platform.api.identity` in SNAPSHOT — fixed across 9 files; casehubio/parent#40 filed for sibling repos.

**97 tests, 0 failures.** Both repos on `main`. Fork pushed.

## Current state

- **Project repo:** `main`, 97 tests pass
- **Workspace:** `main`
- **PR to upstream:** not opened — offer at start of next session if desired
- Blog: `2026-05-22-mdp01-proxy-singleton-writer.md` published to mdproctor.github.io
- Garden: GE-20260522-99b6a0 (@ApplicationScoped proxy), GE-20260522-bc642c (@ObservesAsync ArrayList)

## Key open items

| Issue | What | Priority |
|-------|------|----------|
| casehubio/clinical#6 | IRB gate — CRITICAL deviation consumer | Epic 6 — unblocked (work#136 closed) |
| casehubio/clinical#11 | Adverse event safety officer notification | After connectors pattern |
| casehubio/parent#40 | Audit ActorType import in sibling repos | Next parent session |

## What's next

**Epic 6 (IRB gate) is unblocked** — work#136 closed this session. `ProtocolDeviationResolvedEvent` with `IRB_REVIEW` → `IrbApproval` WorkItem. Need to verify the engine-side `WorkItemLifecycleEvent` → `CONTEXT_CHANGED` adapter is implemented in casehub-engine before starting.

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #6 | IRB gate — full CRITICAL deviation path | L | High | Verify engine adapter first |
| #11 | AE safety officer notification via connectors | M | Med | After connectors pattern |

## References

- Design: `DESIGN.md` (workspace main — journal merged this session)
- Blog: `blog/2026-05-22-mdp01-proxy-singleton-writer.md`
- Previous handover: `git show HEAD~1:HANDOFF.md`
