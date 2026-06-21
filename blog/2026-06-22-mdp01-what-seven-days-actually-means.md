---
layout: post
title: "What Seven Days Actually Means"
date: 2026-06-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [engine, sla, regulatory, ind-reporting, spi]
series: issue-83-ind-deadline-workitem
---

The FDA says seven days. For Grade 4 and 5 unexpected adverse events, that's the window from AE report to IND expedited safety report filing. Grade 3 gets fifteen. `RegulatorySubmissionCaseService` had been computing that deadline correctly since Layer 7 — `ae.reportedAt.plus(indReportingWindow(grade))` — and putting it in the engine case context as `indReportingDeadline`. What it hadn't done was make anything enforce it.

The `regulatory-submission.yaml` used a capability binding. The filing agent received the deadline as input context and proceeded. If it missed the window, nothing downstream cared. The audit trail showed the obligation was identified. It didn't show whether it was met.

That's the gap clinical#83 closes.

## The PropagationContext argument

The first design proposal was `PropagationContext`. Compute `budget = deadline - now()` at case start, pass it as a `PropagationContext`, let the engine propagate `caseBudgetDeadline` to cap the WorkItem's `expiresAt` via `earliestOf`. It almost works.

Two problems. First, `now()` runs twice — once when the budget is computed, once when the deadline is reconstructed — and those instants are not the same. For a seven-day window the drift is academic, but for an FDA audit that asks whether the deadline was enforced precisely, being wrong by construction is the wrong answer. Second, `caseBudgetDeadline` caps every humanTask binding in the case. That's correct here only because there's one binding. The mechanism encodes an absolute regulatory deadline as a resource budget, which is a semantic mismatch regardless of whether it produces the right number today.

I wanted something that stayed in the YAML and evaluated the stored deadline string at scheduling time — not a duration from whenever the case started.

## Building `extractString`

The solution was a new field on `HumanTaskTarget`: `expiresAtExpression`, a JQ (or pluggable-expression-language) value evaluated against the case context WORKING panel when the humanTask binding fires. I brought Claude in for the engine work. We extended `ExpressionEngine` with a `default extractString()` SPI method that throws `UnsupportedOperationException` by default — engines that support value extraction override it. The registry catches the exception from engines that don't, returning `Optional.empty()` with a WARN. JQ overrides it: eval against WORKING panel, guard for empty output, `isTextual()` check (because `NullNode.asText()` returns the string `"null"`, not null, and would reach `Instant.parse()` without it).

The expression is validated at YAML load time using `JsonQuery.compile()`. An invalid expression throws `IllegalArgumentException` at startup. A silent null deadline at runtime is a regulatory SLA failure with no error signal — the load-time check makes it a deployment failure instead.

`HumanTaskScheduleEvent` is a Java record. Adding `expiresAtDeadline: Instant` is a breaking change; all constructor callsites needed updating. There's no incremental path for record fields. That's fine — the breakage is the migration, and the migration is mechanical.

## Six rounds before a line of code

Before implementation started, we went through six rounds of spec review. Each round found real bugs.

Round 2 caught the two-tier breach policy logic. After `EscalateTo` fires, `ExpiryLifecycleService.executeEscalateTo()` replaces `item.candidateGroups` with only the escalation group — the original groups are gone. A policy that only tests `contains("regulatory-affairs")` on the second breach will return `Fail` for its own escalated WorkItem, making the `Exhausted` branch unreachable. The `isRegulatory` guard has to test both groups.

Round 3 found that `WorkItemCallerRef.parseCaseId()` — which I'd proposed as a new engine addition for the breach listener — doesn't exist. `CallerRef.parse(String)` does, in `io.casehub.workadapter`, as a sealed interface handling both `PlanItemCallerRef` and `GateCallerRef` formats and returning null for non-engine callerRefs. `IrbDecisionListener` already uses it. No engine change needed.

Round 3 also surfaced that `WorkItemLifecycleEvent.detail()` is always null for ESCALATED events. `ExpiryLifecycleService.executeExhausted()` writes the `Exhausted(reason)` string to the audit log via `writeAudit()` but calls `fireLifecycleEvent("ESCALATED", item)` with a hardcoded null detail. The breach reason is not recoverable from the event. `IndReportBreachLedgerEntry.breachReason` is therefore a fixed string.

The most important catch was in the final code review: the compliance supplement on `writeFiledEntry()` was wrong. It was attaching `regulatorySubmission(ae.grade)` — the factory that describes the initial obligation identification, with `algorithmRef = "RegulatorySubmissionCaseService"`. The filing event needs its own supplement, `regulatorySubmissionFiled(ae.grade)`, with `algorithmRef = "RegulatorySubmissionCompletedListener"`. An FDA auditor tracing the record would otherwise see a description of grade screening where the record should document submission confirmation. That's an inaccurate FDA audit trail, which is the thing this entire layer exists to prevent.

## The test that confirms the chain

The end-to-end invariant: `WorkItem.expiresAt == ae.reportedAt + indReportingWindow(grade)`. Engine tests verify the scheduler handles `expiresAtDeadline` correctly. JQ tests verify `.indReportingDeadline` is extracted from context. Neither verifies the full chain with actual domain data.

The first attempt at the lifecycle test had a cross-contamination problem: both grade tests were sharing the in-memory WorkItemStore, and filtering by `candidateGroups.contains("regulatory-affairs")` matched WorkItems from the other grade's test. We fixed it by filtering on `callerRef.contains("case:" + caseId)`, scoping each assertion to its own engine case. Grade 3 now confirms `reportedAt + 15 days`; Grade 4 confirms `reportedAt + 7 days`. Both exact.

## What Layer 10 is

This is the first layer that closes a gap that wasn't architecturally impossible before — it was just unenforced. The `expiresAtExpression` field is generic: any humanTask binding in any harness can now express absolute deadline enforcement declaratively with a JQ expression against the case context. The two-tier breach policy pattern is documented and tested. The ledger subclasses give FDA auditors two distinct tamper-evident records: `IndReportFiledLedgerEntry` when the obligation is met; `IndReportBreachLedgerEntry` when it isn't.

The gap was never hidden. Layer 7's spec documented it as "the platform currently does not enforce it as a WorkItem claimDeadline with automatic escalation." Layer 10 is the answer to that sentence.
