# Design Journal — epic-3-multi-site-sub-case

### 2026-05-25 · §Architecture · §Key Abstractions · §Data Model · §SPI Contracts

Layer 6 introduces trial-level blackboard aggregation. Each trial gets a
`CaseInstance` (via `ClinicalTrialCaseHub`, `trial-coordination.yaml`) started when
the trial transitions to ACTIVE. AE escalation processes at each site signal the
trial case via `runtime.signal(trialCaseId, "grade4Active.<siteId>", Boolean)` — the
designed cross-case communication mechanism. The trial case's DSMB rollup binding
fires when the accumulated `grade4Active` map shows ≥2 sites simultaneously (JQ:
`[to_entries[] | select(.value == true)] | length >= 2`). No site sub-cases — sites
are domain entities and the per-AE escalation cases (Layer 5) remain independent.

Three-phase activation in `TrialActivationService` separates the `@Transactional`
status change from the `startCase().join()` call — holding a DB connection while
blocked on async engine persistence deadlocks the Agroal pool under pool exhaustion.

`AeEscalationCompletedEvent` gains `siteId` so `TrialSafetySignalService` can clear
the `grade4Active` flag when escalation resolves. Grade threshold uses
`Set.of(GRADE_4, GRADE_5)` in both signal services — explicit and ordinal-safe.

`deviation-review.yaml` corrected: `when:` → `on.contextChange.filter:` (GE-20260523-fd8725;
the `when` field is silently ignored for `contextChange` triggers in the engine).
