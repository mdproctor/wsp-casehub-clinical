# Session Handover — 2026-06-10

## Last Session

Implemented tenant query isolation (#71) — all 6 entities get `findByIdForTenant(UUID, CurrentPrincipal)` helpers; 12 REST read call sites updated; child entity write-path stamp derives from parent entity's `tenantId`, not `principal.tenancyId()`. `AdverseEventService` refactored to load enrollment once and drop `CurrentPrincipal` injection. CDI fixes: `FixedCurrentPrincipal` in `selected-alternatives`, renamed casehub-work timer jobs excluded. Branch closed, squashed 11→4 commits on main.

## Immediate Next Step

Run `/work` to start the next issue. Layer 7 (trust routing, #8) is the natural next epic. Check `docs/AGENTIC-HARNESS-GUIDE.md` for current layer status before starting.

## What's Left

- ARC42STORIES.MD §9.4 Layer 7+ entries for memory integration not yet added — comes via journal merge at next branch close
- Paused: `issue-68-compile-and-tenancy` (paused 2026-06-07, stack entry preserved)
- Engine async failures in `AeEscalationLifecycleTest` and `DsmbRollupTest.two_sites_grade4` — ConditionTimeout from casehub-work SNAPSHOT CDI graph change; pre-existing engine#393 pattern, not a regression

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #8 | Trust-weighted safety agent routing — Layer 7 | L | High | Next layer per harness guide |
| #10 | 3-site showcase and ClinicalAgent comparison — Layer 8 | M | Med | After trust routing |

## References

- Blog: `blog/2026-06-10-mdp01-the-filter-that-wasnt-there.md`
- Spec: `docs/specs/2026-06-09-tenant-query-isolation-design.md` (project repo)
- Garden: `GE-20260610-e6929a` (Hibernate @Filter/findById gotcha), `GE-20260610-711f61` (Panache tenant isolation pattern)
- Last commit: `bdbf0d2 docs(claude): FixedCurrentPrincipal CDI ambiguity + direct entity tenantId stamping`
