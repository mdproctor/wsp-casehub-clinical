# Session Handover — 2026-07-07

## Last Session

Closed #116 — CBR Phase 1. Wired CbrCaseMemoryStore for AE (FeatureVectorCbrCase), deviation (PlanCbrCase), and amendment (TextualCbrCase) precedent storage and retrieval. Three REST endpoints, explore page panels, 30 new tests. Also fixed two pre-existing flaky tests: AdverseEvent @DynamicUpdate for concurrent observer lost-update (GE-20260707-76a6f4), ClinicalCaseDefinitionEquivalenceTest semantic CandidateSetSpec comparison. Filed CBR roadmap epic #115 (6 children) in prior session.

## Immediate Next Step

Pick from What's Next — #99 or #104 are ready. Run `/work` to start.

## What's Left

- `#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10
- Pre-existing flaky test: `ThreeSiteShowcaseTest` in full suite (engine RECIPIENT_FAILURE — clinical#87)
- DemoDataSeeder SUSAR oversight case start times out — partial demo data
- Governance endpoint degraded — WorkerDecisionEntry removed in ledger SNAPSHOT, stubbed with TODO
- CBR explore panels use static demo entity IDs — dynamic row selection deferred (casehub-pages DSL limitation)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #99 | Guided Steps 3-4: AE Event + Governance | M | High | Ready |
| #104 | Guided Steps 3-4: Deviation + PI Auth | M | High | Ready |
| #100 | Guided Steps 5-6: Resolution + Proof | M | Med | Depends on #99 |
| #86 | Wire ProtocolAmendmentAdvisor | M | High | Blocked: engine#101 |
| #78 | CBR over AE history | L | High | Blocked: neocortex#68 |

## References

- `specs/2026-07-07-cbr-phase1-design.md` — design spec (promoted to project)
- `blog/2026-07-07-mdp01-cbr-phase1-three-case-types.md` — session diary
- Garden: GE-20260707-76a6f4 — @ObservesAsync concurrent entity lost-update gotcha
