---
title: "Four Rounds to Get the Attestation Right"
date: 2026-06-16
entry_type: note
projects: [casehub-clinical]
tags: [layer-7, trust-routing, attestation, spec-review]
---

I started Layer 7 confident I understood the trust routing architecture. I was wrong in ways that four rounds of spec review had to correct before we wrote a line of implementation code.

The spec review process is a deliberate gate — write the design, then send a fresh reviewer with no session context to evaluate it. The first round came back with something I hadn't verified: the attestation mechanism I'd designed was writing directly to `ActorTrustScoreRepository`, the score store. What I missed was that `TrustScoreJob` overwrites the entire store on each 24-hour cycle. Any direct writes would be silently discarded on the next run. The correct hook is `LedgerAttestation` — records anchored to `WorkerDecisionEntry` that `TrustScoreJob` reads as its input. I had the mechanism completely backwards.

That sent me back to the bytecode. `casehub-engine-ledger` wasn't a current project dependency, so IntelliJ didn't index it. I decompiled the jar directly — `WorkerDecisionEventCapture`, `TrustScoreCache`, `PerActorTrustComputer` — to understand what the engine actually does with agent decisions. `PerActorTrustComputer.computeForActor()` takes `List<LedgerEntry>` and `Map<UUID, List<LedgerAttestation>>` — the attestations are the quality signal input, not the output.

The second fatal error came in round two. My `AeEscalationAttestationObserver` was supposed to fire on `AeEscalationCompletedEvent` and write attestations for the safety-monitoring agent. The reviewer asked a simple question: where does the safety-monitoring agent actually work? In the AE escalation case (`ae-escalation.yaml`)? I read the YAML. Both bindings are `humanTask`. `WorkerDecisionEventCapture` only fires for agent workers, not humans. There are zero `WorkerDecisionEntry` records in the AE escalation case. The safety-monitoring agent works in the SUSAR oversight case. The correct quality signal is the gate outcome — APPROVED or REJECTED/EXPIRED — not the escalation completion.

By round three, the attestation was using the right trigger and the right case. What remained was a compilation impossibility: I'd put `TrustAttestationStrategy` in `api/spi/` with `AttestationVerdict` from `casehub-ledger-api`, but `casehub-ledger-api` isn't in `api/pom.xml`. Straightforward constraint, easy fix.

Round four was documentation cleanup — the spec had two contradictory sentences about `@DefaultBean` displacement, a wrong file table entry, and a missing sentence about how `RegulatorySubmissionCaseService` calls the ledger writer. All minor, but they would have confused whoever implemented it.

What I take from this is that the spec review process earns its cost. Each round I went in thinking the spec was solid. Each round something genuinely wrong came back. The errors weren't subtle style issues — they were architecture errors that would have produced code that silently does nothing useful at runtime.

The implementation itself was clean once the design was right. Eleven sequential tasks, each with a failing test written before the code, each reviewed before the next began. `ClinicalTrustRoutingPolicyProvider` displaces the engine-ledger's `@DefaultBean` without any registration — just classpath presence. `SusarAgentAttestationWriter` observes the same three gate events as `SusarGateDecisionListener` and uses the same DB-discriminator pattern to identify SUSAR gates. `RegulatorySubmissionCaseService` follows the three-phase pattern from ADR-0004, same as `SusarOversightCaseService` and `TrialActivationService`.

The code review found two more things worth fixing: both new ledger writers were using `ae.tenantId` instead of `"default"`, diverging from eight existing writers; and `RegulatorySubmissionLedgerEntry.domainContentBytes()` was excluding `filedAt` for stated GDPR reasons that don't actually hold — `filedAt` isn't PII and isn't subject to erasure. Small fixes, but the tenantId one would have silently produced wrong behaviour in integration tests.

Layer 7 is the final pre-showcase layer. Layer 8 (action-risk oversight gates) shipped a couple of weeks ago. With trust routing wired, the next step — the 3-site showcase and ClinicalAgent comparison — is unblocked.
