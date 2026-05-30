# Handoff — casehub-clinical
2026-05-30

## What happened this session

Branch `issue-11-ae-safety-officer-notification` designed, implemented, reviewed, and closed. Safety officer AE notification (ICH E6(R3) §5.17 / 21 CFR 312.32) shipped to casehubio/clinical main as one squashed commit. Key design decision: `AdverseEventReportedEvent` as trigger (not `WorkItemLifecycleEvent` — Grade 4/5 engine cases create two WorkItems, so the lifecycle event fires twice). `DefaultSponsorNotifier` aligned to `@All List<Connector>` in the same commit.

## Current state

- **Project repo:** `main` — 152 tests passing, pushed to both fork (mdproctor/clinical) and blessed repo (casehubio/clinical)
- **Workspace:** `main`
- **Blog:** `2026-05-30-mdp01-a-domain-event-fires-once.md`
- **Garden:** GE-20260529-af0f2e (Grade 4/5 multi-WorkItem duplicate notification trap)
- **Protocol:** PP-20260530-2ad9a4 (domain CDI events not WorkItemLifecycleEvent for triggers)

## Outstanding (filed, not yet done)

- **casehubio/clinical#45** — observer exception fallback + integration test for safety officer notification path · M · Med
- **casehubio/clinical#46** — actorId alignment across ledger writers ("system" vs "clinical-service") · S · Low
- **casehubio/parent#104** — sync casehub-clinical.md + PLATFORM.md for #11 completion · XS · Low

## Hygiene

- **`epic-multi-site-sub-case`** workspace branch — 11 days with no commits; open, blocked on engine#387

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | AE safety officer observer fallback + integration test | M | Med | Follow-on from #11 |
| #46 | actorId alignment across ledger writers | S | Low | Cross-cutting fix |
| Layer 7 | Trust routing + ClinicalAgent comparison | XL | High | Blocked by engine#387 |
