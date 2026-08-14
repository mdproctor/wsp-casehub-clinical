---
layout: post
title: "Wiring the commitment lifecycle through qhorus"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [qhorus, commitment, pi-authorisation, multi-tenant]
---

The commitment endpoint has been a placeholder since the demo UI was scaffolded — returning a flat `CommitmentLifecycleResponse` built entirely from `ProtocolDeviation` fields. DevId, status string, commanded-at timestamp. Enough to render something, not enough to show the actual lifecycle.

The `<commitment-lifecycle>` component expects richer data: a stages array with per-stage actor attribution and timestamps, plus the channel messages from the PI oversight conversation. That data lives in qhorus — in the `Commitment` record and the channel's message history. The endpoint needed to query both and translate the result.

The translation is the interesting part. Qhorus models commitments with seven states: OPEN, ACKNOWLEDGED, FULFILLED, DECLINED, FAILED, DELEGATED, EXPIRED. The clinical domain uses different vocabulary — a PI receives a COMMAND (not an "open" obligation), approves or rejects it (not "fulfills" or "declines"). So the endpoint maps OPEN to COMMANDED, FULFILLED to DONE, and so on. FAILED and EXPIRED both produce a `failed` status on the last reached stage node, because the component renders failure as a visual state on the timeline — red node instead of green.

One edge case worth noting: ACKNOWLEDGED can be skipped entirely. A PI can go straight from receiving the COMMAND to approving the deviation — no explicit acknowledgment step. The `Commitment` record tells you this happened because `acknowledgedAt` is null but the state is terminal. The stage derivation handles this by marking ACKNOWLEDGED as `completed` with no timestamp — the timeline shows progression without a gap, but doesn't fabricate an event that never occurred.

The design review surfaced a tenancy concern I hadn't considered. `CommitmentReader.findByCorrelationId()` doesn't accept a tenant parameter. In a multi-tenant deployment, two tenants could theoretically share a correlation ID format — both use deviation UUIDs, which are globally unique per tenant but the reader doesn't know that. The fix is a post-query check: verify `commitment.tenancyId()` matches `principal.tenancyId()` before returning anything. Return 404 on mismatch — don't leak that a commitment exists in another tenant.

A related decision: inject `CommitmentReader` (the read-only SPI), not `CommitmentStore`. The endpoint only reads. In a clinical compliance context, giving a display endpoint write access to commitment state would be a legitimate audit finding. Least privilege matters when FDA inspectors are reviewing your access patterns.

Channel messages come from `MessageService.history()`, which already excludes EVENT-type messages by default. These are qhorus infrastructure signals — the PI oversight channel carries the COMMAND, any STATUS updates, and the terminal DONE or DECLINE. EVENTs are internal bookkeeping and would just confuse the UI. One query, 100-message limit, and the conversation is complete — PI oversight channels are purpose-built per-deviation and rarely exceed a handful of messages.

The whole thing landed as a single service class — `CommitmentLifecycleService` — that the endpoint delegates to. The old flat DTO is gone. The component gets what it was always designed to consume.
