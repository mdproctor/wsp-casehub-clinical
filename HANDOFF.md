# Handoff — casehub-clinical
2026-06-01

## What happened this session

Housekeeping and build triage. Completed the Layer 5 LAYER-LOG.md entry (missing
Completed date + #48 observer fallback extension as a prerequisite for ARC42STORIES.MD
migration, #54). Revised issue #54 with corrected references and a fuller migration plan.

Ran the full build and fixed four things: `findByUuid(UUID, String tenancyId)` API
change in `AeEscalationListener` and tests; EVENT channel type added to `pi-oversight`
(qhorus#153 shipped); `IrbCommitteePolicySpiTest` rewritten with `@InjectMock` (the
`@TestProfile + getEnabledAlternatives()` approach replaces `selected-alternatives`,
deactivating `MemoryPlanItemStore`); `GroupMembershipProvider` CDI ambiguity resolved
via `ClinicalGroupMembershipProvider` in test/support (`quarkus.arc.exclude-types` is
silently ignored for Jandex-indexed JAR beans — GE-20260601-848232).

After fixing CDI, 12 tests failed with `AbstractMethodError` — `CaseHubRuntimeImpl`
doesn't implement `startCase(CaseDefinition, Object)`. The June 1 engine-api SNAPSHOT
added this method; the engine implementation JAR wasn't rebuilt against it. IntelliJ's
background compilation of sibling projects silently installed both JARs into `~/.m2`
mid-session. #53 updated with this finding.

- **Blog:** `2026-06-01-mdp01-what-the-build-cache-was-hiding.md`
- **Garden:** GE-20260601-848232, GE-20260601-cee623, GE-20260601-265cdc, GE-20260601-81be07
- **Protocol:** PP-20260601-aec35f (use @InjectMock for SPI override tests)
- **CLAUDE.md:** updated — @InjectMock rule, qhorus#153 shipped, GroupMembershipProvider ambiguity, findByUuid tenancyId note

## Current state

- **Project repo:** `main` — pushed
- **Workspace:** `main`

## Outstanding (filed, not yet done)

- **casehubio/clinical#53** — engine API incompatibility: `startCase(CaseDefinition, Object)` missing from `CaseHubRuntimeImpl`; `findByUuid` + `PlanExecutionContext` may also be affected; 12 lifecycle tests failing · M · Med
- **casehubio/clinical#54** — bootstrap ARC42STORIES.MD (LAYER-LOG.md Layer 5 now complete — prerequisite done) · L · Med
- **casehubio/clinical#49** — early-return audit gap (missing-config paths leave no ledger trace) · S · Med
- **casehubio/clinical#52** — test coverage gaps: enrollmentId null path; IrbDecisionListenerTest missing approvalId assertion · XS · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #53 | Wait for engine snapshot to be fixed; then re-run build to confirm green | XS | Low | Engine-side fix; nothing to do in clinical until engine publishes |
| #54 | Bootstrap ARC42STORIES.MD — Layer 5 LAYER-LOG entry complete, ready to migrate | L | Med | Follow arc42stories-casehub-profile.md; devtown as reference |
| #49 | Early-return audit gap in notification listeners | S | Med | Missing-config paths leave no trace |
| #52 | Test coverage gaps — two small additions | XS | Low | Batch into #49 session if convenient |
| Layer 7 | Trust routing | XL | High | Blocked on engine#387 |
