# Session Handover — 2026-06-17

## Last Session

Three S/XS issues closed on branch `issue-82-grade4-ind-s-fixes` (covers #82, #84, #85). #82: Grade 4 + unexpected AE now triggers IND 7-day expedited reporting — three-location atomic update per PP-20260617-3167e3. #84: SusarOversightLifecycleTest drives gate/lifecycle listeners directly; asserts COMPLETED not REQUESTED-or-COMPLETED. #85: `mvn install` CDI ambiguity fixed — `TenantScopedPrincipal @RequestScoped` (casehub-work) excluded from production CDI; `QhorusInboundCurrentPrincipal` remains sole non-default principal. 382 tests green; `mvn install` BUILD SUCCESS. Branch merged to casehubio/clinical main.

## Immediate Next Step

Pick up #10 (3-site showcase + ClinicalAgent comparison) — now the only remaining dependency before the showcase.

## What's Left

- `casehubio/work#268` — upstream fix: make `TenantScopedPrincipal @Alternative` so clinical's `exclude-types` workaround can come out
- `casehubio/parent#269` — sync `casehub-clinical.md` Layer 7 description (Grade 5 → Grade 3/4/5 IND path)
- `SusarOversightLifecycleTest` tenantId inconsistency fixed (was "test-tenant" vs TEST_TENANCY_ID UUID) · S · Low ✅ done this session
- `mvn install` CDI ambiguity fixed · S · Med ✅ done this session

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #10 | 3-site showcase + ClinicalAgent comparison | XL | High | All S/XS cleared; Grade 4 path now complete |
| #83 | IND reporting deadline enforced as WorkItem claimDeadline | M | Med | Needs regulatory-submission.yaml humanTask binding change |
| #82 | IND Grade 4 7-day path | XS | Low | ✅ closed this session |
| #84 | SusarOversightLifecycleTest direct listener invocation | S | Low | ✅ closed this session |
| #85 | mvn install CDI ambiguity | S | Med | ✅ closed this session |

## References

- Blog: `blog/2026-06-17-mdp01-grade4-cdi-and-a-jta-surprise.md`
- Protocol: PP-20260617-318449 (clinical-test-entity-tenant-stamp — @BeforeEach must stamp tenantId)
- Garden: GE-20260609-77a6f9 revised (work#268 upstream tracker added)
- Upstream issues: casehubio/work#268, casehubio/parent#269
