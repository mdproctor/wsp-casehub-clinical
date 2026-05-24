# Handoff — casehub-clinical
2026-05-24

## What happened this session

Maintenance session post-Epic 6. Squashed 69 commits to 11 (pushed to both remotes). Added GitHub Actions CI (Build and Test + Publish workflows) — green. Fixed two production CDI failures hidden by the test suite: `casehub-engine-persistence-hibernate` missing from production classpath, and `quarkus.datasource.reactive=false` needed in production `application.properties` as well as test. CLAUDE.md synced (engine#112 closed, production CDI wiring conventions). Parent deep-dive (`docs/repos/casehub-clinical.md`) edited locally — **not yet committed in parent session**.

## Current state

- **Project repo:** `main` — CI green (98aa196)
- **Workspace:** `main`
- Blog: `2026-05-24-mdp01-what-the-test-suite-isnt-telling-you.md`
- Both remotes (mdproctor + casehubio) in sync

## Outstanding

- **Parent repo commit needed** — `docs/repos/casehub-clinical.md` was edited at `/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-clinical.md` but not committed. Run from parent session: `git -C /Users/mdproctor/claude/casehub/parent add docs/repos/casehub-clinical.md && git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: update casehub-clinical deep-dive — Layers 1–5 complete, Layer 6 blocked\n\nCloses #49" && git -C /Users/mdproctor/claude/casehub/parent push`

## Stale workspace branches

*Unchanged — `git show HEAD~1:HANDOFF.md`* (epic-deviation-resolution-ledger, epic-multi-site-sub-case, issue-18-deviation-expiration-requires-new, issue-24-minor-cleanups, issue-6-irb-gate due deletion 2026-06-05/06). Note: epic-deviation-resolution-ledger and epic-multi-site-sub-case lack EPIC-CLOSED.md — pre-date work-end discipline.

Backup branches: `backup/pre-squash-main-20260519` and `backup/pre-squash-main-20260523` (local only).

## Key open items

*Unchanged — `git show HEAD~1:HANDOFF.md`* — except engine#112 is now CLOSED (2026-05-15). Layer 6 is structurally unblocked but engine CI is red (PR#334 broke HumanTaskScheduleHandlerTest with new DELEGATED state).

| Issue | What | Priority |
|-------|------|----------|
| casehubio/clinical#11 | AE safety officer notification via connectors | After connectors pattern |
| casehubio/engine PR#334 | DELEGATED state broke work-adapter tests — blocks clean SNAPSHOT | Engine team |
| casehubio/parent#49 | Sync casehub-clinical.md — file edited, needs parent session commit | Next parent session |

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #11 | AE safety officer notification via connectors | M | Med | After connectors pattern established |
| Layer 6 | Multi-site sub-case + DSMB rollup | XL | High | Unblocked when engine PR#334 merges cleanly |
| Layer 7 | Trust routing + ClinicalAgent comparison | XL | High | After Layer 6 |

**Immediate next step:** Check engine PR#334 status — if merged and CI green, Layer 6 is unblocked. Otherwise work on clinical#11.
