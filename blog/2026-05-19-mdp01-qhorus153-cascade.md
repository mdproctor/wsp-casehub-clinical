---
layout: post
title: "qhorus#153 and the Cascade"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: log
projects: [clinical]
tags: [qhorus, transaction-isolation, cdi]
---

We started today with a handful of small tasks to work on while waiting. The
first check — is qhorus#153 shipped? — turned out not to be small at all.

The issue (`MessageReceivedEvent` CDI hook) closed overnight. Which meant the
commented-out `@ObservesAsync MessageReceivedEvent` handler in `PiResponseListener`
could finally be uncommented. And the `PiResponseListenerIntegrationTest` marked
`@Disabled` since May could be enabled. And clinical#16 (the redundant
`commitmentService.fulfill()` calls that qhorus#154 had made obsolete) could be
removed. One check triggered four changes.

The integration test didn't just pass — we had to implement it. The stub had sat
there for weeks with a comment listing seven steps. We worked through them: report
a deviation, simulate the PI's JSON response via `ChannelGateway.receiveHumanMessage()`,
assert the status changed to APPROVED or ESCALATED.

Two wrong assumptions surfaced immediately.

First: we created the PI oversight channel with `allowedTypes = QUERY,COMMAND`.
That blocks the PI from sending DONE via `receiveHumanMessage()`. The fix is obvious
once you know it — add DONE and DECLINE — but the mental model of the channel as
a one-directional command channel hides it. Claude caught this as
`MessageTypeViolationException` with the allowed types listed in the message.

Second: the `@ObservesAsync` observer called `this.process()`, which has `@Transactional`.
But `this.process()` bypasses the CDI proxy. `@Transactional` on the called method has no
effect. Claude hit `ContextNotActiveException` inside `process()` and traced it to the
missing transaction boundary. The fix: add `@Transactional` directly to `onMessage()`.
The observer is called through the proxy; `process()` then joins that transaction.

The Awaitility polling for the async assertion had the same issue — polling lambdas have
no JTA context. Wrapped in `QuarkusTransaction.requiringNew().call()`. Three separate
CDI transaction surprises, all in one test.

## The transaction isolation work

Before qhorus#153 shipped, we'd already implemented clinical#18: the structural fix
for `DeviationExpirationJob`. The batch loop with a try/catch was aspirationally correct
— "one failure shouldn't roll back other deviations" — but the try/catch cannot recover
from a JPA exception that marks the whole transaction rollback-only. I'd known this for
days and wanted it fixed cleanly.

The fix: extract `DeviationExpirer @ApplicationScoped` with `@Transactional(REQUIRES_NEW)`
per deviation. Each deviation commits independently. The implementation was straightforward
once the design was settled. The test (clinical#20) was trickier — needed `@InjectMock
CommitmentService` to fail on the second call, verifying the first deviation remained committed.

`CommitmentService.fail()` returns `Optional<Commitment>`, not void. That cost us a failed
stub attempt (`doNothing()` doesn't apply to non-void methods). Three lines after the fix:
77 tests passing.

clinical#19 (inject `Clock` into `DeviationLedgerWriter`) was genuinely small. Ten minutes.
The timestamp assertions in the unit test went from `isNotNull()` to `isEqualTo(FIXED_INSTANT)`.
That's the difference the change makes — from "something was written" to "the exact
timestamp the system recorded."
