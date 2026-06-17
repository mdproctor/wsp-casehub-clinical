# Session Handover — 2026-06-17

## Last Session

Two issues closed on a single branch. #80: Clock injection into `SusarAgentAttestationWriter` — three `Instant.now()` calls replaced with `@Inject Clock clock`; `writeAttestation()` private param removed. #81: Grade 3 IND 15-day expedited path — `isIndReportable()` predicate, `indReportingWindow()` switch, `ClinicalComplianceSupplement.regulatorySubmission(CtcaeGrade)` grade-aware factory. 379 tests green. Squashed 10→3 commits, pushed to fork and upstream. Both issues closed.

## Immediate Next Step

Pick up #10 (3-site showcase + ClinicalAgent comparison) — now the last remaining dependency before the showcase.

*Updated: casehubio/parent#254 closed — removed from backlog.*

## What's Left

- `casehubio/clinical#82` — Grade 4 (life-threatening) + unexpected → IND 7-day (c)(1)(i); extends `isIndReportable()` and `indReportingWindow()` · XS · Low
- `casehubio/clinical#83` — IND reporting deadline enforced as WorkItem `claimDeadline` with auto-escalation · M · Med
- `SusarOversightLifecycleTest` accepts REQUESTED or COMPLETED — gate creation path (via Quartz) never fires in `@QuarkusTest`; documented in CLAUDE.md and GE-20260614-b97659 · S · Low
- `mvn install` production augmentation CDI ambiguity (`MockCurrentPrincipal` + `FixedCurrentPrincipal` under `package` goal) — pre-existing, not introduced this branch; `mvn test -pl runtime` unaffected (379/0/0) · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #10 | 3-site showcase + ClinicalAgent comparison | XL | High | Grade 3 path now complete |
| #82 | Grade 4 IND 7-day path | XS | Low | One-liner in isIndReportable + indReportingWindow + regulatorySubmission |
| #83 | IND deadline enforcement via WorkItem claimDeadline | M | Med | Requires regulatory-submission.yaml humanTask binding change |

## References

- Blog: `blog/2026-06-17-mdp01-grade3-ind-assertion-trap.md`
- Spec: `docs/specs/2026-06-16-grade3-ind-expedited-design.md` (promoted to project)
- Protocol: PP-20260617-3167e3 (clinical-ind-reporting-grade-extension — three-location atomic update rule)
- Garden: GE-20260616-848099 (CFR substring assertion trap), GE-20260616-ba2c72 (LedgerEntry.compliance() in unit tests), GE-20260616-0b20d0 (engine context vs inputSchema blackboard)
