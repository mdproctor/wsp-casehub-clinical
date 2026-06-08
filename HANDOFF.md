# Handoff — casehub-clinical
2026-06-08

## What happened this session

Shipped #50 — fluent Java DSL companion classes for all three clinical YAML case
definitions (`DeviationReviewCaseDefinition`, `AeEscalationCaseDefinition`,
`TrialCoordinationCaseDefinition`) in production scope
(`io.casehub.clinical.casedefinition`), plus `ClinicalCaseDefinitionEquivalenceTest`
(plain JUnit 5 — no Quarkus). Companions use verbatim JQ strings so
`JQExpressionEvaluator` record equality makes the equivalence test structurally
meaningful. Key finding: `CaseDefinition.Builder` populates `def.getGoals()` and
`setCompletion()` via independent code paths — dual registration required.

TrialCoordinationYamlTest compile failure (pre-existing #68 fix) cherry-picked to
main. Squashed to 4 commits, pushed fork + upstream.

## Current state

- **Project repo:** `main` — pushed to fork + upstream (squashed to 4 commits)
- **Workspace:** `main`
- **Pause stack:** `issue-68-compile-and-tenancy` (#68 compile fix + tenancy #69 work)

## Outstanding

- Workspace branch `epic-multi-site-sub-case` past deletion window — retained per policy
- Backup branches `backup/pre-squash-main-20260519`, `backup/pre-squash-main-20260523` past 14-day retention — confirm deletion when convenient

## What's next

⚡ **engine#387 CLOSED 2026-06-07** — Layer 7 (trust routing, #8) is now unblocked.

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #8 | Epic 8: Trust-weighted safety agent routing (Layer 7) | XL | High | **Unblocked** — engine#387 shipped 2026-06-07 |
| #68/#69 | TrialCoordinationYamlTest fix + multi-tenancy coverage | M | Med | Paused; compile fix cherry-picked to main; tenancy still pending |
| #9 | Epic 9: LLM supervisor mode | XL | High | Blocked on engine#102 |
| #33 | CaseMemoryStore integration | L | High | Blocked on platform#27 |
| #47 | ActionRiskClassifier oversight gate | M | High | Blocked on engine#402 |
| #69 | Multi-tenancy — tenancyId coverage | M | Med | Blocked on work + ledger multi-tenancy |
