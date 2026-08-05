---
layout: post
title: "When the Grade Changes, the Record Has to Follow"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, regrade, compliance, adverse-events]
---

Clinical trials run on a basic assumption: the CTCAE grade assigned to an adverse event at initial report is correct. But clinicians reassess. A Grade 1 fatigue that seemed unremarkable at day two looks very different at day five when the patient can't get out of bed. The grade changes from 1 to 3, and suddenly the compliance obligations change with it — tighter SLA deadlines, safety officer notification, potential SUSAR evaluation.

casehub-clinical already handled most of this. The regrade service, the REST API, the ledger entry, the five async listeners that fan out on grade change — all built during earlier CBR and Layer 7 work. What was missing was the CBR supersession hook: when the grade changes, the case-based reasoning record still carries the old grade's features, and similarity matching returns the wrong precedents.

## The Feature Degradation Trade-Off

The interesting design question wasn't whether to update the CBR case — obviously you should. It was *what to put in it*.

A CBR case stored at escalation completion carries features from two sources. Problem features — grade, event type, trial phase — come from the entity and can be re-read at any time. But outcome features — what the safety review concluded, whether the DSMB was involved — came from the engine case file snapshot at completion time. That data isn't persisted on the adverse event entity. It was ephemeral context, consumed once, then gone.

I had three options. Supersede the old case and accept the loss. Persist the outcome features on the entity so they could be re-derived. Or update the grade and accept that two outcome features would degrade to null.

I chose the third. The grade is the primary retrieval key — it determines which past cases a new AE matches against. Getting that right matters more than preserving `safetyReviewOutcome` and `dsmbEscalated`. A re-stored case tagged with `regradeSource=regrade` tells retrieval consumers exactly what happened: this case was rebuilt after a grade reassessment, and two outcome features are unavailable.

## The Escalation Gap Nobody Asked About

The gap audit surfaced a second finding that's more consequential. When an AE is regraded upward — say Grade 3 to Grade 4 — and the original escalation already completed, `AeEscalationCaseService` silently skips creating a new engine case. The higher-grade AE gets no fresh safety review at the escalated severity. The CBR hook corrects the stale record, but the underlying clinical workflow gap remains. I filed it as a separate issue because it's a design question, not a bug: should a 3→4 upgrade re-open the existing case or start a fresh one?

## The Smaller Fixes That Matter

`RegradeRequest` had no validation on the `reason` field — a clinician could regrade an AE with an empty string, which weakens the FDA audit trail. `@NotBlank @Size(max = 500)` closes that. Two YAML case definitions lacked `completion` expressions that a new engine SNAPSHOT now requires — unreferenced goals went from silently ignored to startup failure.

The UI work — grade history timeline and regrade action form — follows the custom web component pattern established by `cbr-precedents-panel`. Both components use hardcoded demo entity IDs because casehub-pages doesn't support parameterised drill-down datasets yet. I filed that gap with the API I'd want: table selection events flowing into parameterised dataset URLs. Five detail panels in clinical are blocked by the same limitation.

## Observer Fan-Out Is Getting Complex

casehub-clinical is accumulating listeners at a rate that makes the event fan-out on a single domain action genuinely complex. An AE grade change now fires to six independent observers: escalation, SUSAR, regulatory, safety officer, trajectory alerts, and CBR supersession. Each is independently tested, each handles its own errors, each operates on the same event. But the interaction surface grows combinatorially, and the "what if two observers race on the same entity" question gets harder to answer with confidence. The in-memory CBR store's last-writer-wins semantics paper over the race today. Whether that holds when cases carry real retrieval weight is an open question.
