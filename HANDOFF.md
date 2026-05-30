# Handoff — casehub-clinical
2026-05-30

## What happened this session

Closed #45 (observer exception fallback) and #46 (actorId alignment). Both issues
are in casehubio/clinical main. CLAUDE.md updated with integration test pattern for
`@ObservesAsync` listeners (call directly, no Awaitility, REQUIRES_NEW is synchronous).

Key deliverables: `ClinicalActors.CLINICAL_SERVICE` constant + five writers aligned;
double try/catch fallback with `REQUIRES_NEW writeObserverFailureEntry` on
`SafetyOfficerNotificationListener` and `SponsorNotificationListener`;
`SafetyOfficerNotificationIntegrationTest` (happy path + connector failure).
Code review caught `notifiedAt = null` violating NOT NULL schema constraint —
fix: use `connectorId = null` as failure-mode discriminator instead.

- **Garden:** GE-20260530-01bcfe (NOT NULL violation at REQUIRES_NEW commit, not at save()), GE-20260530-7426b7 (nullable column as failure-mode discriminator)
- **Protocols:** PP-20260530-d6775a (ClinicalActors.CLINICAL_SERVICE), PP-20260530-49856c (observer double try/catch)
- **Blog:** `2026-05-30-mdp02-silence-is-not-an-audit-trail.md`

## Current state

- **Project repo:** `main` — pushed to casehubio/clinical and mdproctor/clinical
- **Workspace:** `main`
- **Blog:** `2026-05-30-mdp02-silence-is-not-an-audit-trail.md`

## Outstanding (filed, not yet done)

- **casehubio/clinical#48** — observer fallback for `AeEscalationListener` + `IrbDecisionListener` · M · Med
- **casehubio/clinical#49** — early-return audit gap (missing-config paths in notification listeners) · S · Med

## Hygiene

- **`epic-multi-site-sub-case`** workspace branch — stale, blocked on engine#387
- **`epic-3-multi-site-sub-case`** workspace + project — check if needs closure stamp

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #48 | AeEscalationListener + IrbDecisionListener observer fallback | M | Med | Constrained data availability — see issue for design notes |
| #49 | Early-return audit gap in notification listeners | S | Med | Missing-config paths leave no trace |
| Layer 7 | Trust routing | XL | High | Blocked on engine#387 |
