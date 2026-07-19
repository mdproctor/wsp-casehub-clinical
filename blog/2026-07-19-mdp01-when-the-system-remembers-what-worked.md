---
layout: post
title: "casehub-clinical — When the System Remembers What Worked"
date: 2026-07-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, plan-adaptation, escalation, reuse]
---

# casehub-clinical — When the System Remembers What Worked

**Date:** 2026-07-19

Phase 4 gave clinical the ability to store what happened — every AE escalation case now writes a `PlanCbrCase` with its execution trace and a full audit trail. Phase 5 makes that memory useful: when a new AE triggers escalation, the system retrieves similar past cases and adapts their plans as advisory recommendations.

## The interesting part isn't retrieval

Retrieval is a solved problem — `ClinicalCbrService.retrieveWithAudit()` already handles it. The real design question was adaptation: given a plan that worked for a similar Grade 3 hepatotoxicity event, what do you change when the current case is Grade 4 and meets SUSAR criteria?

We implemented `ClinicalPlanAdapter` — a clinical-specific `PlanAdapter` SPI implementation with four rules. The rules are simple individually but compose in ways that matter:

- Steps that failed in past cases get suppressed — priority drops to zero
- Steps that succeeded get boosted
- When the current AE is higher grade than the precedent, safety-related steps get an additional urgency bump — but only if they weren't already suppressed
- When the current AE meets SUSAR criteria but the precedent didn't, a synthetic `susar-oversight` step gets added

That third rule's interaction with the first is the one that caught my attention during the design review. A step that failed shouldn't get promoted just because the current case is more severe — the failure signal should dominate. Without the explicit exclusion, a Grade 4 event could recommend repeating exactly the step that didn't work for a Grade 3.

## Two consumption paths

The adapted plan surfaces in two places. First, `AeEscalationCaseService` injects it into the case context at escalation start — alongside the existing patient and site context. Human task assignees (safety monitors, DSMB members) see what worked in past similar cases as part of their work item payload.

Second, a REST endpoint at `/api/adverse-events/{aeId}/escalation-plans` makes the same recommendation queryable at any point during the AE lifecycle. An investigator reviewing a case mid-flight can ask "what worked for similar events?" without waiting for a new escalation to trigger the injection.

## What this actually means

Priority values in the adapted plan are advisory ordering hints, not engine execution directives — the case bindings still fire based on their `contextChange` filters. The system isn't overriding clinical judgment. It's presenting precedent: "4 out of 5 times for similar Grade 3 events, this sequence succeeded." The PI or safety monitor decides what to do with that information.

This closes the Reuse step in the CBR cycle for clinical AE escalation. Retain (Phase 4) stores what happened. Retrieve finds similar cases. Reuse adapts them. The system gets better at handling adverse events over time by surfacing what worked — and what didn't.
