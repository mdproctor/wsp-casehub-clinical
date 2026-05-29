# Handoff — casehub-clinical
2026-05-29

## What happened this session

Branch `issue-42-43-44-test-coverage` designed, implemented, reviewed, and closed. Three issues delivered: #44 (grade4 lifecycle assertions), #43 (@Alternative IrbCommitteeAssignmentPolicy SPI test + rename), #42 (signalGrade4Active extracted to TrialSafetySignalService, @InjectSpy routing verification). 136 tests passing. Key CDI 2.0 gotcha: @Alternative @Priority globally enables a bean across all tests — documented in CLAUDE.md and garden (REVISE on GE-20260415-884e48).

## Current state

- **Project repo:** `main` — 136 tests passing, pushed to casehubio/clinical upstream
- **Workspace:** `main`
- **Blog:** `2026-05-29-mdp02-the-global-alternative.md`
- **Garden:** GE-20260415-884e48 revised (getEnabledAlternatives() + @Priority trap)
- **Parent issue:** casehubio/parent#99 filed (TrialSafetySignalService description drift)

## Outstanding (filed, not yet done)

- **casehubio/parent#99** — TrialSafetySignalService description drift · S · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| Layer 7 | Trust routing + ClinicalAgent comparison | XL | High | Blocked by engine#387 (dynamic candidateGroups in YAML) |
| #11 | AE safety officer notification via connectors | M | Med | — |
