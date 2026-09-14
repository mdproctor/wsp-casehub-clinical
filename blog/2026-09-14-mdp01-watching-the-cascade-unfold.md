---
title: "Watching the Cascade Unfold"
date: 2026-09-14
author: mdp
entry_type: note
subtype: diary
series: issue-161-sse-event-cascade
projects:
  - casehubio/clinical
tags:
  - websocket
  - event-cascade
  - real-time
  - casehub-pages-push
---

# Watching the Cascade Unfold

When an adverse event is reported in casehub-clinical, the platform fires a cascade: SLA timer, SUSAR gate, trust-weighted agent routing, oversight decision, Merkle seal. Until today, that cascade was invisible. The UI showed end-state data — a grade, a status, a ledger digest — but the orchestration itself, the thing that makes the platform worth building, happened behind the curtain.

Issue #161 changes that. Users now watch each step light up in real time.

## The infrastructure was already there

The interesting discovery was that casehub-pages already had most of what I needed. `EventBroadcaster` in `casehub-pages-push` handles server→client delivery with topic subscriptions, sequence tracking, and gap detection. `EventConnection` on the client side manages reconnection with exponential backoff and cursor persistence. `EventTimelineNode` in pages-viz renders timelines with status transitions and category-based colour coding.

What was missing was the bridge: something to catch CDI events and Vert.x event bus messages as they fire and push them through the WebSocket to whoever is watching.

## The dual-mechanism problem

The first design review caught something I'd glossed over: gate events (`ActionGateApprovedEvent`, `ActionGateRejectedEvent`, `ActionGateExpiredEvent`) don't use CDI `@ObservesAsync`. They fire via Vert.x `eventBus.publish()`. The cascade broadcaster needed to consume both — CDI observers for domain events like `AdverseEventReportedEvent`, and Vert.x `@ConsumeEvent` for gate decisions. Same class, two mechanisms, coexisting cleanly because the engine already supports multiple consumers on the same Vert.x address.

The `caseId → aeId` resolution for gate events follows the same pattern `SusarGateDecisionListener` already uses: `AdverseEvent.findBySusarOversightCaseId(caseId)`. No new query, no new index — just reusing the existing DB lookup.

## Grade-aware templates

Not every AE triggers the full cascade. Grade 1-2 events don't fire `AdverseEventReportedEvent` at all — they're handled synchronously via WorkItem creation. Grade 3+ get the escalation path. Grade 3+ unexpected gets IND regulatory submission. Grade 3+ unexpected + suspected gets the SUSAR oversight gate.

`CascadeTemplateResolver` encodes this: given a grade and two boolean flags (`unexpected`, `suspected`), it returns the exact list of cascade steps the timeline should show. The UI renders all steps as "pending" upfront, then transitions each to "active" → "completed" as events arrive. The user sees where they are in the process and what's coming next.

## The topic separator trap

All three design review dimensions — coherence, structure, robustness — independently flagged the same issue: the spec used `/` as the topic separator (`clinical/ae/{aeId}/cascade`), but `TopicRegistry` uses `:` for its trie-based matching. The client-side `topic-matching.ts` also splits on `:`. Three reviewers, same finding, different angles. The fix was mechanical — `clinical:ae:{aeId}:cascade` — but missing it would have meant silent delivery failures with no error.

## What's next

The cascade timeline shows the orchestration, but it's still observational — the user watches, they don't interact. The gate steps show up as "completed" or "failed" after the fact because there's no engine-level event for gate creation (only gate resolution). Showing the gate as "active" while awaiting PI decision would require plumbing engine events for gate lifecycle, which is a separate piece of work.

Protocol deviation cascades are the obvious next extension — the same broadcaster pattern, different domain events, different template.
