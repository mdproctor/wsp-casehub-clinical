# Design Journal — issue-50-dsl-companions

### 2026-05-22 · §Platform Integration

`AdverseEventLedgerWriter @ApplicationScoped` extracted from `AdverseEventService`, following the `DeviationLedgerWriter` pattern. The AE service was the last write site that inlined ledger construction and hardcoded `sequenceNumber = 1`. This matters because Epic 4 will add resolution and escalation entries for adverse events — without a dedicated writer, each new write site would independently read the latest sequence with no shared place to test the invariant. The extraction happens now rather than at Epic 4 so the pattern is established before the ledger chain grows.

The `DeviationLedgerWriter` pattern note in `§Key Architecture Decisions` ("The pattern should be followed for any future ledger subclass written from multiple services") was prescriptive. `AdverseEventLedgerWriter` is the first case where the prescription applies in practice.

`ActorType` moved from `io.casehub.ledger.api.model` to `io.casehub.platform.api.identity` in a platform SNAPSHOT update. Updated across all 9 affected files. No architectural change — import path only.

### 2026-05-22 · §Key Architecture Decisions

`TestSlackConnector` converted from static fields to `@Singleton` instance with `sent()` / `setShouldThrow()` accessor methods and a `reset()` method. The `@Singleton` scope is required: `@ApplicationScoped` beans in Quarkus ArC are subclass-proxied — direct field access on the proxy reads the proxy's own field (always empty), not the delegate's, because Java field access is not virtualised through the proxy dispatch mechanism. `@Singleton` beans are injected directly with no proxy, so field access and method calls both hit the real instance. `CopyOnWriteArrayList` is retained for the `sent` backing store because `SponsorNotificationIntegrationTest` uses `@ObservesAsync`, dispatching `send()` on a managed executor thread while the test thread reads `sent()` — `ArrayList` has no JMM visibility guarantee across threads.

`SponsorNotificationListener` now distinguishes "trial not found in DB" from "trial has incomplete connector config (one field null)" with different log messages and diagnostic detail. The distinction matters operationally: a missing trial record indicates a data integrity issue; a partially-configured trial is a deployment configuration error. Giving an operator the same log message for both makes triage harder.

`DefaultSponsorNotifier.buildTitle()` now uses `req.severity().name()` rather than hardcoding `"[MAJOR Deviation]"`. The current escalation mapping only routes MAJOR deviations to sponsor notification, so the hardcode was functionally correct. It was changed because the mapping is in `DeviationResponsePolicy` (a deployer-configurable SPI) — if a deployer changes the policy to route CRITICAL deviations to sponsors, the hardcoded title would silently produce wrong notifications.

### 2026-05-22 · §Open / Deferred

Several open items are now resolved and should be removed from the table at branch close:
- casehubio/qhorus#153 — shipped; `PiResponseListenerIntegrationTest` is live.
- casehubio/clinical#13 — closed; sponsor notification shipped in a prior epic.
- casehubio/clinical#16 — closed; redundant commitment calls removed.
- casehubio/clinical#17 — closed; ledger#92 fixed reactive service activation upstream via `Instance<T>` guard. The `quarkus.arc.exclude-types` workaround for ledger SNAPSHOT services (in `§Key Architecture Decisions`) is also stale — it was never applied in clinical (the fix landed before clinical consumed the service) and should be removed.
- casehubio/clinical#19 — closed; `Clock` injected into `DeviationLedgerWriter`.
- casehubio/clinical#20 — closed; `DeviationExpirerIsolationTest` verifies REQUIRES_NEW guarantee.
- casehubio/work#136 — closed; unblocks casehubio/clinical#6 (IRB gate).
