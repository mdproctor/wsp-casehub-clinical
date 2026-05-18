---
layout: post
title: "Closing the Ledger Loop"
date: 2026-05-18
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [ledger, fda-audit-trail, normative-layer]
---

After the PI authorisation work shipped, the Merkle chain was only half-written.
The COMMAND entry existed — the formal obligation to the PI was recorded.
But not how it resolved: approved, rejected, escalated, expired.
An FDA inspector reading only the ledger entries could see the PI was commanded and nothing else.

That's structurally incomplete for a tamper-evident audit trail.

## The question I pushed back on

Before touching code, I asked whether resolution entries should link to the qhorus normative record.
The PI's DONE or DECLINE message is a genuine speech act — it exists in the qhorus Merkle chain.
Should the clinical ledger entry carry a `commitmentId` field to connect the two chains?

My gut said yes. If you're building a system to demonstrate that the normative layer adds value, you want the audit trail to link into it.

Claude came back with a direct answer: adding `commitmentId` to the ledger entry doesn't test the hypothesis.
It's a foreign key, not a cryptographic link.
The chain from clinical domain record to qhorus normative record already exists through `ProtocolDeviation.commitmentId` — it doesn't need to be duplicated in the ledger entry.

That held up. The normative layer's structural value is demonstrated by the architecture — the COMMAND/Commitment lifecycle, the state machine, the named obligation with a deadline — not by a UUID field in a join table.

## Can you prove it empirically?

I asked directly whether the hypothesis is testable. Claude's answer was honest.

Yes, partially. The most concrete test: can a PI receive a COMMAND, respond through their normal channel (Slack, email), and have the clinical domain state update and the Commitment close — with zero clinical-specific response-handling code? If yes, that's measurable evidence. The normative layer did the work.

State machine completeness is also testable: every COMMAND reaches a terminal state — FULFILLED, DECLINED, or FAILED — without clinical code tracking the obligation's lifecycle. That's a structural guarantee a log can't make without discipline.

But the stronger claim — that this is irreducibly better than a sophisticated log — is harder to prove. That's a design argument, not a measurement. The full empirical test isn't runnable yet; it needs the CDI hook from qhorus and a real `HumanParticipatingChannelBackend`.

## DeviationLedgerWriter

Three services write protocol deviation ledger entries: `ProtocolDeviationService` (the COMMAND), `PiResponseListener` (the PI response), and `DeviationExpirationJob` (the expiration). Each writes to the same chain, so each needs to compute `sequenceNumber`. Without a shared owner there's no place to test the invariant that numbers increment correctly across all three.

We extracted `DeviationLedgerWriter` — an `@ApplicationScoped` bean that owns sequence computation via `findLatestBySubjectId` and provides two named methods: `writeCommandEntry` and `writeResolutionEntry`. Each service delegates. The invariant is testable in isolation.

This pattern — dedicated writer bean for a ledger subclass written from multiple services — is worth formalising across the platform. The issue is open on the parent repo.
