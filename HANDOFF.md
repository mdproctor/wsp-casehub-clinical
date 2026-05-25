# Handoff — casehub-clinical
2026-05-25

## What happened this session

Layer 6 complete: trial-level blackboard aggregation + DSMB rollup (Epic 3, casehubio/clinical#3 closed). Key design decision: no site sub-cases — sites are domain entities, sub-case model is for bounded delegated work. Two production-masking bugs caught: JQ `to_entries | select()` silently fails (needs `to_entries[]`), and `@Transactional + startCase().join()` deadlocks the Agroal pool (three-phase activation is the fix — ADR 0004).

## Current state

- **Project repo:** `main` — 135 tests passing (eb72e81)
- **Workspace:** `main`
- **PR:** casehubio/clinical#35 — pending upstream merge to casehubio/clinical
- Blog: `2026-05-25-mdp01-what-a-sub-case-is-for.md`
- Garden: GE-20260525-d5b5eb (JQ `to_entries[]`), GE-20260525-6f8b88 (`@Transactional+join` deadlock)

## Outstanding

- **casehubio/parent#71** — sync `docs/repos/casehub-clinical.md` (Layer 6 complete, Layer 6→7 ordering corrected, Epic 3 status, new services). Run from parent session.
- **casehubio/clinical#34** — minor review findings (M1–M4): YAML test string assertion, Javadoc layer ref, spec naming drift, error response style.
- **casehubio/clinical PR#35** — pending upstream merge; check before starting Layer 7.

## Stale workspace branches

*Unchanged — `git show HEAD~1:HANDOFF.md`* (epic-deviation-resolution-ledger, epic-multi-site-sub-case, issue-18-deviation-expiration-requires-new, issue-24-minor-cleanups, issue-6-irb-gate due deletion 2026-06-05/06). Backup branches: `backup/pre-squash-main-20260519`, `backup/pre-squash-main-20260523` (local only).

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| Layer 7 | Trust routing + ClinicalAgent comparison | XL | High | Start after PR#35 merges |
| #11 | AE safety officer notification via connectors | M | Med | Can start any time |
| #34 | Minor review findings (M1–M4) | XS | Low | Quick cleanup |

**Immediate next step:** Check casehubio/clinical PR#35 status — if merged, start Layer 7. If still open, work on #11 or #34.
