---
layout: post
title: "PI accountability is shipped — and two qhorus gaps are now documented"
date: 2026-05-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
excerpt: "The protocol deviation PI authorisation implementation is in main — pure qhorus COMMAND lifecycle, per-deviation channels, and two structural gaps in the inbound path that are now documented upstream."
---

The previous entry ended on a note of concern: getting the classification right before writing code matters more than moving fast. I got it right. The implementation is in main.

The core decision was whether to take the PI's response through a dedicated REST endpoint or keep the entire interaction inside qhorus channels. The REST endpoint is tempting — testable, explicit, easy to reason about. It's also wrong. When a real `HumanParticipatingChannelBackend` is wired — Slack, email, an IRB portal — that endpoint becomes dead code or creates a dual-entry problem. Worse, the PI's response would be recorded in clinical's database rather than as a RESPONSE message in the qhorus channel. The COMMAND exists, the Commitment exists, the resolution happens outside the normative record. That's not a code smell; it's a structural audit gap.

I brought Claude in for the implementation. We named the channel after the deviation, not the PI: `clinical/deviation/{deviationId}/pi-oversight`. The reason surfaced from reading qhorus source code. `ChannelGateway.receiveHumanMessage()` passes `correlationId=null` to MessageService — the inbound human path structurally cannot carry a correlationId. A per-PI channel (one inbox per investigator) has no way to identify which deviation the response belongs to. Per-deviation channels make the entity identity structural: it's in the URL, not the message. This is now documented as a gap in qhorus.

The other surprise was a Flyway version collision. casehub-qhorus and casehub-work both ship migrations at `classpath:db/migration` — qhorus V1–V9, work V1–V21+. Quarkus Flyway scans all JARs on the classpath for that path; there's no per-JAR filter. The fix is to move application migrations into datasource-scoped subdirectories: `db/migration/default/` for clinical domain tables, `db/migration/qhorus/` for ledger join tables. Each Flyway datasource points at its own location without touching the other's JARs. Tests use drop-and-create with Flyway disabled — the classpath scan can't be filtered at that granularity.

Claude reviewed the implementation before merging. It caught one correctness issue: `PiResponseListener` injected `CommitmentService` but never used it. The auto-open in `MessageService.send()` works — send a COMMAND with a non-null correlationId and the Commitment opens automatically. The inverse doesn't hold. When the PI responds via `HumanParticipatingChannelBackend`, `receiveHumanMessage()` passes the same `correlationId=null`, so the auto-fulfill never fires. The listener has to call `commitmentService.fulfill()` or `commitmentService.decline()` explicitly — without those two calls the Commitment stays OPEN indefinitely after the PI responds.

One gate remains. `PiResponseListener` has the `@ObservesAsync` method written but commented out. For the CDI event chain to fire — `receiveHumanMessage()` → event → listener — qhorus needs to emit a `MessageReceivedEvent` after storing the inbound message. The unit tests call `listener.process()` directly and cover the state machine fully. The integration test is written and disabled pending the upstream change. The accountability model is complete; the final wiring waits on one qhorus PR.
