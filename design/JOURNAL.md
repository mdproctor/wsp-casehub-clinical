# Design Journal — issue-42-43-44-test-coverage

### 2026-05-29 · §Key Abstractions

`TrialSafetySignalService` now owns both directions of the grade4 blackboard flag: `signalGrade4Active(siteId)` sets it when a Grade 4+ AE case starts, and `onAeEscalationCompleted` clears it when the case closes. Previously the set path was a private method in `AeEscalationCaseService` calling `runtime.signal()` directly. The refactor removes `CaseHubRuntime` and `TrialCaseLookup` from `AeEscalationCaseService`'s injection surface — it now delegates all trial blackboard signaling to `TrialSafetySignalService`. This consolidation makes the signaling responsibility explicit and testable (pure Mockito unit tests cover both set and clear paths in `TrialSafetySignalServiceTest`).

Update the `TrialSafetySignalService` bullet to: "owns all grade4 blackboard flag operations: `signalGrade4Active(siteId)` sets the flag when a Grade 4+ AE starts; `onAeEscalationCompleted` clears it on AE case completion. All direct `runtime.signal()` calls for grade4 flags route through this service."

Update the `AeEscalationCaseService` bullet to: "starts AE cases; delegates Grade 4+ trial flag to `TrialSafetySignalService.signalGrade4Active()`."

### 2026-05-29 · §SPI Contracts

`IrbCommitteeAssignmentPolicy` SPI (delivered in branch issue-28-engine-spi-cleanup, #29) is missing from the DESIGN.md SPI table — add it. Maps a protocol deviation context (`IrbCommitteeContext`: deviationId, siteId, trialId, severity) to an `IrbCommitteeAssignment` (committeeId, candidateGroups). `DefaultIrbCommitteeAssignmentPolicy` returns `irb-committee` for all cases; override with `@Alternative @ApplicationScoped` for site-specific routing. Integration-tested in `IrbCommitteePolicySpiTest` via `@QuarkusTestProfile.getEnabledAlternatives()` (note: `@Priority` must NOT be on the test alternative — it globally enables it across all tests, overriding `@DefaultBean` in the entire test run).
