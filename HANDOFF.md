# Handoff — casehub-clinical
2026-05-30

## What happened this session

Docs-only session. Stripped tutorial/showcase framing from `CLAUDE.md` and `LAYER-LOG.md` in the project repo, following the arc42stories direction and using casehub-life as the reference implementation.

Key changes: Agentic Harness Goals rewritten to a single production-first statement. "Tutorial Structure" → "Foundation Layers". Showcase Scenario section removed. "What it shows" → "What it adds" across all six complete LAYER-LOG layers. ClinicalAgent comparison language removed from layer bodies and gap table column headers. tutorial-strategy.md references removed. Arc42Stories migration note added to LAYER-LOG.md header. One commit: `6918cb4 docs: strip tutorial framing from CLAUDE.md and LAYER-LOG.md`.

## Current state

- **Project repo:** `main` — pushed to casehubio/clinical
- **Workspace:** `main`
- **Blog:** `2026-05-30-mdp01-a-domain-event-fires-once.md` (unchanged from previous session)
- **Garden:** GE-20260529-af0f2e (Grade 4/5 multi-WorkItem duplicate notification trap)
- **Protocol:** PP-20260530-2ad9a4 (domain CDI events not WorkItemLifecycleEvent for triggers)

## Outstanding (filed, not yet done)

- **casehubio/clinical#45** — observer exception fallback + integration test for safety officer notification path · M · Med
- **casehubio/clinical#46** — actorId alignment across ledger writers ("system" vs "clinical-service") · S · Low
- **casehubio/parent#104** — sync casehub-clinical.md + PLATFORM.md for #11 completion · XS · Low

## Hygiene

- **`epic-multi-site-sub-case`** workspace branch — 12 days with no commits; open, blocked on engine#387
- **`epic-3-multi-site-sub-case`** workspace + project — 5 days old, no closure stamp; check if merged work needs stamping

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | AE safety officer observer fallback + integration test | M | Med | Follow-on from #11 |
| #46 | actorId alignment across ledger writers | S | Low | Cross-cutting fix |
| parent#104 | Sync casehub-clinical.md + PLATFORM.md for #11 completion | XS | Low | Cross-repo doc |
| Layer 7 | Trust routing | XL | High | Blocked by engine#387 |
