# Handoff — casehub-clinical
2026-05-29

## What happened this session

Branch `issue-28-engine-spi-cleanup` designed, implemented, reviewed, and closed. Five issues delivered: #28 (CaseLifecycleEvent import fix), #39 (LAYER-LOG build note), #27 (AeEscalationStatus + COMPLETED write-back), #26 (engine_case_id on AdverseEvent + ProtocolDeviation), #29 (IrbCommitteeAssignmentPolicy SPI). Also fixed #41 (StubWorkloadProvider — engine#378 deleted CasehubWorkloadProvider, causing @QuarkusTest CDI failure). 11 commits on upstream, 133 tests passing.

Key discovery: in-memory engine doesn't fire `CaseCompleted` CDI reliably; `GoalReached` is the reliable proxy. `AeStatusUpdater @ApplicationScoped` with `REQUIRES_NEW` handles idempotency.

## Current state

- **Project repo:** `main` — 133 tests passing, pushed to `casehubio/clinical` upstream
- **Workspace:** `main`
- **PR:** none
- **Blog:** `2026-05-29-mdp01-the-case-that-completed-silently.md`
- **Garden:** GE-20260529-fef800 (GoalReached/CaseCompleted CDI gap); GE-20260427-5d7c67 revised (StubWorkloadProvider solution added)
- **Protocols:** PP-20260529-3ffe28 (three-phase pattern), PP-20260529-f67675 (WorkloadProvider stub), PP-20260529-fa9cbf (@BeforeEach entity creation)

## Outstanding (filed this session, not yet done)

- **clinical#42** — signaling coverage gap (grade4/5 runtime.signal) · S · Med
- **clinical#43** — @Alternative policy injection test for IrbCommitteeAssignmentPolicy · S · Med
- **clinical#44** — grade4 lifecycle assertions (escalationStatus + engineCaseId) · XS · Low
- **clinical#28** — AeEscalationListener internal event — now CLOSED (fixed in this branch)
- **engine#387** — dynamic `candidateGroups` in YAML humanTask binding (blocks full SPI routing) · M · Med
- **engine#393** — CaseCompleted CDI event unreliable in @QuarkusTest (root cause of GoalReached workaround) · M · Med
- **casehubio/parent#94** — docs/repos/casehub-clinical.md needs new SPI/enum/field entries

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| Layer 7 | Trust routing + ClinicalAgent comparison | XL | High | `IrbCommitteeAssignmentPolicy` wired; `ClinicalTrustRoutingPolicyProvider` is the next addition. engine#387 still blocks dynamic candidateGroups in YAML |
| #11 | AE safety officer notification via connectors | M | Med | — |
| #42 | Grade4/5 signaling test coverage | S | Med | Add @QuarkusTest with trial setup or quarkus-panache-mock |
| #43 | @Alternative IrbCommitteeAssignmentPolicy test | S | Med | @InjectSpy or @QuarkusTestProfile |
| #44 | Grade4 lifecycle assertions | XS | Low | 3 assertThat lines after the await |
