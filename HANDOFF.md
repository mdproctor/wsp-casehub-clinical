*Updated: parent#296, parent#297, platform#111 closed — removed from backlog.*

# Session Handover — 2026-06-24

## Last Session

clinical#88 closed: wire casehub-platform-oidc. `OidcCurrentPrincipal` is now the sole active `CurrentPrincipal` — tenant identity from JWT, not X-Tenancy-ID header. `ClinicalGroups` (SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR) in `api/`. `@RolesAllowed` on all 19 REST endpoints per GCP/FDA regulatory topology. `RbacBoundaryTest` with 27 boundary tests. `quarkus.security.deny-unannotated-members=true`. Four-round spec review caught three critical security bugs before any code was written. Filed casehubio/platform#111 (OidcCurrentPrincipal needs @Alternative @Priority(100)) and casehubio/clinical#89 (MissingTenancyClaimExceptionMapper — blocked on platform#111). Two protocols captured (PP-20260623-491bb3 RBAC topology, PP-20260623-b105bb risk gates severity-driven). Four garden entries submitted (GE-20260623-4613f4, GE-20260623-941ade, GE-20260623-5b192f, GE-20260623-18f8c0).

## Immediate Next Step

Pick next issue. Candidates from backlog: #86 (wire ProtocolAmendmentAdvisor to LlmPlanningStrategy — blocked: engine#101), #87 (engine TestCaseInstanceRepository.clear() for ThreeSiteShowcaseTest isolation — blocked: engine team), #79 (GDPR Art.17 erasure endpoint — unblocked, ~1 day).

## What's Left

- `casehubio/clinical#89` — MissingTenancyClaimExceptionMapper · S · Low (unblocked — platform#111 closed)
- `casehubio/clinical#86` — ProtocolAmendmentAdvisor → LlmPlanningStrategy · M · High (blocked: engine#101)
- `casehubio/clinical#87` — CaseLifecycleEvent listeners hold JTA TX while .await()-ing reactive CaseInstanceRepository · S · Med (unblocked)
- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` + IRB completion listener — deferred from #10

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #79 | GDPR Art.17 erasure endpoint | M | Med | Unblocked — ConsentWithdrawalService exists, needs REST endpoint |
| #86 | Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy | M | High | Blocked: engine#101 |
| #87 | CaseLifecycleEvent listeners hold JTA TX while .await()-ing reactive repo | S | Med | Unblocked — perf optimisation |

## References

- Blog: `blog/2026-06-23-mdp01-the-spec-that-caught-three-bugs.md`
- Spec: `docs/specs/2026-06-23-oidc-platform-wiring-design.md` (project)
- Platform issue: casehubio/platform#111
- Follow-up: casehubio/clinical#89
- ARC42STORIES.MD: Layers 1–10 complete; auth wiring not a layer (cross-cutting)
