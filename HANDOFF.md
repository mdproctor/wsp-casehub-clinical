# Session Handover — 2026-06-19

## Last Session

Epic 10 closed: 3-site showcase + ClinicalAgent comparison. Implemented eligibility screening (Site A — eligibility-screening.yaml, EligibilityScreeningCaseService, EligibilityScreeningLedgerWriter, V2024) and protocol amendment (Site C — ProtocolAmendmentAdvisor SPI stub, protocol-amendment.yaml, ProtocolAmendmentListener, V2025). ThreeSiteShowcaseTest (§7.4 narrative) and docs/comparison/clinicalagent.md delivered. Code review surfaced three significant fixes: trial-existence check on POST /amendments, re-screen guard (409) on POST /screen, and entity-outside-@Transactional in EligibilityScreeningCaseService. Full build passes with ThreeSiteShowcaseTest excluded (engine race condition: clinical#87).

## Immediate Next Step

Choose next issue to work on. Candidates: #83 (IND reporting deadline enforced as WorkItem claimDeadline), #86 (wire ProtocolAmendmentAdvisor to LlmPlanningStrategy when engine#101 lands).

*Updated: work#268, parent#287 closed — removed from backlog.*

## What's Left

- `casehubio/parent#269` — sync `casehub-clinical.md` Layer 7+8+9 description (was tracked via parent#287, now closed) · S · Low
- `casehubio/clinical#86` — wire ProtocolAmendmentAdvisor to LlmPlanningStrategy when engine#101 lands · M · High (blocked: engine#101)
- `casehubio/clinical#87` — engine TestCaseInstanceRepository.clear() API for ThreeSiteShowcaseTest isolation · S · Med (blocked: engine team)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — out of scope for #10, deferred

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #83 | IND reporting deadline enforced as WorkItem claimDeadline | M | Med | Needs regulatory-submission.yaml humanTask binding change |
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 (open) |

## References

- Blog: `blog/2026-06-18-mdp01-what-the-showcase-asked.md`
- Spec: `specs/2026-06-18-showcase-clinicalagent-design.md`
- Comparison: `docs/comparison/clinicalagent.md` (project repo)
- LAYER-LOG: Layer 9 (Showcase) entry added
- ARC42STORIES.MD: Status updated to "Layers 1–9 complete"
- Upstream issues: work#268, parent#287, clinical#86, clinical#87
