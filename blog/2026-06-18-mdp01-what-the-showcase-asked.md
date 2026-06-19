---
layout: post
title: "What the showcase asked"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [quarkus, cdi, panache, casehub-engine, compliance]
series: issue-10-showcase-clinicalagent
---

Epic 10 started from a simple spec: demonstrate the full compliance story across three independent sites, and publish a comparison table showing what ClinicalAgent doesn't do. I expected the implementation to be mostly writing — the layers were done, the hard wiring was behind us. What I didn't expect was how many design questions the showcase would surface that the layers alone never asked.

The first one came up immediately: Site A's scenario required eligibility screening to trigger an IRB consultation when a criterion is marginal. There was no eligibility screening code. The question was whether to build it properly (a full `EligibilityScreeningCaseHub` with its own YAML, ledger entry, SPI) or to fake it with the existing IRB gate. I chose the full feature, which turned out to be the right call — not because the showcase needed it, but because the compliance story needed it. An eligibility agent making decisions with no tamper-evident record is exactly the gap the comparison table was supposed to close.

Protocol amendment was similar. The §7.4 scenario called for an LLM supervisor reading accumulated multi-site context. Engine#101 hasn't shipped, so I built the slot instead: a `ProtocolAmendmentAdvisor` SPI, a `@DefaultBean` stub that always returns PROCEED, and a clean YAML advisory case. When the engine ships the planning strategy, displacing the stub is one CDI bean. The slot is already wired.

What I didn't expect was the implementation revealing bugs that the spec's extended review process hadn't caught.

The most disorienting was `private @Transactional`. I split `ProtocolAmendmentCaseService.prepareAndMark()` into two overloads: one package-private (for unit tests, takes the already-loaded entity), one private (for production, loads from Panache inside the transaction). The private one carried `@Transactional`. `mvn compile` passed. The first `@QuarkusTest` startup failed with:

```
DeploymentException: @Transactional will have no effect on method
ProtocolAmendmentCaseService.prepareAndMark() because the method is private.
```

CDI interceptors proxy via subclassing — private methods can't be overridden. Quarkus ArC validates this at augmentation time, not compile time. The fix was one character: drop `private`. The class-level visibility model doesn't change; the interception does.

The other one was older. `EligibilityScreeningCaseService` initially loaded the enrollment before calling `prepareAndMark()`, then passed the entity in. This is the same trap `AeEscalationCaseService` fell into before we corrected it — we'd even documented the fix in the session notes. The entity loaded outside a transaction is detached; field mutations inside the service's `@Transactional` boundary don't flush. We'd written the exact pattern that a previous session had to unwind. The fix is always the same: load inside the `@Transactional` method.

The third was `outputSchema(".")` on the `Capability.builder()`. Without it, the Java-function worker completes successfully, returns `{ advisorRecommendation: "PROCEED" }`, and the case context is never updated. The goal condition `.advisorRecommendation != null` never fires. The case hangs silently. Nothing in the stack trace, no warning — the worker ran, the case didn't advance. Adding `.outputSchema(".")` to the capability definition is the bridge between the worker's return map and the live case context.

The comparison table ended up being one of the more useful outputs. ClinicalAgent is peer-reviewed, open source, and recent enough to be a fair baseline. Writing the ten-row table forced precision: not "we have an audit trail" but "the FDA auditor calls `GET .../ledger/verify` and gets a Merkle inclusion proof." That level of specificity is what the comparison needed to be defensible.

One design choice worth naming: the idempotency guard in `ProtocolAmendmentListener` uses `supervisorRecommendation != null` rather than checking the business status. The status-based check would be the obvious move — but `REFER_TO_DSMB` keeps status as `SUPERVISED`, so a second `GoalReached` delivery would pass the guard and write a duplicate ledger entry. The `supervisorRecommendation` field is null before the first run and non-null after any branch, including referral. It's a small thing that's easy to get wrong and hard to notice until you add the `REFER_TO_DSMB` test case.
