# Handoff — casehub-clinical
2026-06-02

## What happened this session

Bootstrapped `ARC42STORIES.MD` — 1466 lines, Layers 1–6 complete, Layer 7 stub, 5 chapters, 8 anti-patterns (§8), 5 ADRs (§10). Generator pre-conditions gate added to `arc42stories-spec.md` before generating, ensuring write-content modes were enforced per-section. Three post-generation quality checks run; issue #30 was closed (removed from §12 Active Risks). Closed #54.

Fixed `GroupMembershipProvider` production CDI ambiguity — `MockGroupMembershipProvider` (casehub-platform) and `NoOpGroupMembershipProvider` (casehub-work) both `@DefaultBean`; production augmentation was failing. Added `ClinicalGroupMembershipProvider @ApplicationScoped` to `runtime/src/main/java/`. Closed #55. Build passing.

Filed casehubio/parent#144 — duplicate `## Writing Style` section in arc42stories-spec.md from PR#138 merge; fix needed in parent session.

- **Blog:** `2026-06-02-mdp01-arc42stories-the-gate.md`
- **Garden:** GE-20260529-182916 (revised — workspace false negatives), GE-20260602-7e604f (new — advisory vs gate in LLM specs)
- **Protocol:** PP-20260602-1a0c25 (ARC42STORIES.MD primary record declaration at bootstrap)

## Current state

- **Project repo:** `main` — pushed to fork + upstream
- **Workspace:** `main`

## Outstanding

- **casehubio/clinical#53** — engine API incompatibility: `startCase(CaseDefinition, Object)` missing from `CaseHubRuntimeImpl`; 12 lifecycle tests failing · M · Med
- **casehubio/clinical#49** — early-return audit gap (missing-config paths leave no ledger trace) · S · Med
- **casehubio/clinical#52** — test coverage gaps: enrollmentId null path; IrbDecisionListenerTest approvalId assertion · XS · Low
- **casehubio/parent#144** — duplicate `## Writing Style` section in arc42stories-spec.md · XS · Low (fix in parent session)
- Workspace branch `epic-multi-site-sub-case` overdue for deletion (was due 2026-05-24, scaffold-only)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #53 | Check engine snapshot; re-run build to confirm green | XS | Low | Engine-side fix; nothing to do in clinical until engine publishes |
| #49 | Early-return audit gap in notification listeners | S | Med | Early-return paths leave no ledger trace |
| #52 | Test coverage gaps — two small additions | XS | Low | Batch into #49 session if convenient |
| ARC42STORIES small | Add YAML snippet for L5 binding structure; SafetyOfficerNotificationLedgerEntry key file in L3 | XS | Low | Self-assessment §5 gaps |
| AML migration | Bootstrap ARC42STORIES.MD for AML following clinical pattern | L | Med | arc42stories-spec.md gate now in place |
| Layer 7 | Trust routing | XL | High | Blocked on engine#387 |
