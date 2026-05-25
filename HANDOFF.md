# Handoff — casehub-clinical
2026-05-25

## What happened this session

S/XS cleanup: closed #34 (M1–M4 review findings), #36 (POM runtime scope), #31 (logging + OUTCOME_KEY constant), #30 (LAYER-LOG Layer 7 stub). Also corrected a regulatory accuracy error in the Layer 7 stub — 21 CFR 312.32(c)(1)(i) requires an `unexpected` flag alongside grade; `AeEscalationCompletedEvent` doesn't carry it, now documented as an API gap. PR#35 (Layer 6) was already merged when the session started; PR#37 (this session's cleanup) rebase-merged.

## Current state

- **Project repo:** `main` — 136 tests passing (b5f636f)
- **Workspace:** `main`
- **PR:** none pending
- Blog: `2026-05-25-mdp01-services-dont-know-http.md`
- Garden: GE-20260525-65a5c1 (debugf/tracef in observers), GE-20260525-00cbde (JAX-RS coupling in services)

## Outstanding

- **casehubio/clinical#11** — AE safety officer notification via connectors · M · Med · Can start any time

## Stale workspace branches

*Unchanged — `git show HEAD~1:HANDOFF.md`* (epic-multi-site-sub-case confirmed closed — EPIC-CLOSED.md in root not design/, deletion due 2026-06-01; all others have design/EPIC-CLOSED.md)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| Layer 7 | Trust routing + ClinicalAgent comparison | XL | High | Requires `unexpected` field on AeEscalationCompletedEvent (see LAYER-LOG Layer 7 stub) |
| #11 | AE safety officer notification via connectors | M | Med | — |
