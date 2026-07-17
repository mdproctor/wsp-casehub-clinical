---
layout: post
title: "casehub-clinical — Closing the CBR Audit Loop"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, ledger, audit, fda, explanation-renderer]
series: issue-117-cbr-phase4-outcome-audit
---

# casehub-clinical — Closing the CBR Audit Loop

**Date:** 2026-07-17
**Type:** phase-update

---

## What I was trying to achieve: make every precedent consultation auditable

Issue #117 is CBR Phase 4 — the final phase of the CBR roadmap. When I looked at what was left, outcome recording was already done. #78 had landed `ClinicalCaseOutcomeObserver` and it was solid — stores `PlanCbrCase` with plan traces, records `CbrOutcome` with success/failure mapping, handles AE/deviation/amendment paths. Twelve tests covering all the edges.

What remained was the other half of the loop: when a clinician consults precedents to inform a decision, that consultation needs a tamper-evident record. FDA wants to trace from decision to precedents to their outcomes. And the clinician needs a structured explanation — not "4 cases found" but "4 prior cases with similarity above 0.85, top match Grade 3 Neutropenia in Phase III, 3 of 4 resolved without escalation."

## The root question: where does the audit happen?

The existing `ClinicalCbrService` is a thin data wrapper. Two methods, zero side effects. The temptation was to pile everything into it — trace building, ledger writing, explanation rendering, principal injection. That would have turned a clean data wrapper into an orchestrator holding unrelated concerns.

I separated the responsibilities instead. Retrieval stays pure. Explanation rendering is a CDI-displaced SPI implementation (`ClinicalExplanationRenderer` displaces neocortex's `DefaultExplanationRenderer`). Ledger writing follows the existing writer pattern — `CbrRetrievalLedgerWriter` with `REQUIRES_NEW`, same as the ten other writers in the project. The service gets one new method, `retrieveWithAudit()`, that composes the pipeline: retrieve, trace, explain, audit, return.

The design review pushed hard on the failure semantics. If the renderer throws, the retrieval still happened and still needs auditing — catch the exception, set explanation to null, write the ledger entry. If the writer throws, propagate — returning unaudited AI decision-support results is a compliance violation. We ended up with a clear invariant: every retrieval that progresses past trace construction produces exactly one ledger entry.

## The SNAPSHOT that broke everything

Rebuilding against a new neocortex SNAPSHOT surfaced a breaking API change. `CbrQuery.of()` gained a mandatory `Path scope` parameter — the old 5-arg form was removed outright. Ten call sites across production and test code broke simultaneously. The fix was straightforward (`Path.root()` as the scope), but REST resource classes that import `jakarta.ws.rs.Path` resolved unqualified `Path.root()` to the JAX-RS annotation. The compiler error said "cannot find symbol: method root()" — no hint that a different `Path` class was needed. Two separate issues masquerading as one.

## What it is now

Three endpoints — AE precedents, deviation precedents, amendment precedents — now return `{ traceId, explanation, precedents: [...] }` instead of a bare list. Every consultation writes a `CbrRetrievalLedgerEntry` with the query features, retrieved case count, top similarity score, and the full explanation text. The explanation is included in the Merkle hash — any post-hoc modification breaks the chain.

The `ClinicalExplanationRenderer` produces domain-aware text: "Adverse event precedent consultation" for AE queries, "Protocol deviation precedent consultation" for deviations. It reports confidence bands across the result set and handles the edge cases — empty results, null confidence values, mixed domains.

The most valuable finding from the design review was `retrievalTraceId` — the spec originally used `traceId`, which shadows `LedgerEntry.traceId` (the OTel trace correlation field on the base table). A JPA mapping collision that would have surfaced only at runtime.
