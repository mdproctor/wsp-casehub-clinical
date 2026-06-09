---
layout: post
title: "The workaround that wasn't"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [casehub-platform, memory, snapshot, ledger]
---

The last entry ended with `ClinicalTestLedgerRepository` in place — a bridge class written because `casehub-ledger-memory`'s `InMemoryLedgerEntryRepository` was still on the old one-arg interface while production code had moved to the two-arg version with tenantId. The CLAUDE.md note was clear: remove when the SNAPSHOT updates.

Checking whether that time had come was easy enough. `ide_find_definition` on `InMemoryLedgerEntryRepository` with `fullElementPreview: true` returns the full decompiled class from the JAR in the local Maven repo — every method signature at once. Every method had the `String tenancyId` parameter. The update had shipped without any signal reaching the consuming project. The workaround had been protecting against a problem that no longer existed.

That pattern is worth keeping in mind. SNAPSHOT updates don't surface in consuming projects unless those projects run against the new API or explicitly check the artifact. Documentation can describe assumptions that are already wrong. The artifact is faster and more reliable than waiting for a sign.

The second item was trickier by design. `toContextMap()` in `ClinicalPatientContext` was producing fact maps without a `grade` field, despite the spec contract showing `{ grade: "GRADE_3", outcome: "RESOLVED", createdAt: "..." }`. The symptom: the key was simply absent. The cause: `storeAeReport` stored `grade.name()` as `MemoryAttributeKeys.OUTCOME`, not as a separate attribute. `hasPriorGrade3OrAbove()` then read `OUTCOME` and tried parsing it as a `CtcaeGrade` — which worked for grade-bearing entries, because `CtcaeGrade.valueOf("GRADE_3")` succeeds, and failed silently for escalation entries, because `CtcaeGrade.valueOf("ESCALATED")` throws `IllegalArgumentException` that is caught and returns false. The conflation went unnoticed because the method gave the right answers. The facts shape was just wrong.

The fix is a new `ClinicalMemoryAttributes.GRADE` key that carries the CTCAE grade as a dedicated attribute, separate from the lifecycle status in `OUTCOME`. Both write paths now stamp the grade explicitly, and the facts map includes both fields. The spec contract is now implemented, not approximated.

The remaining two tasks were the DRUG and IRB memory domains — both deferred when memory integration shipped. The DRUG domain had one design question worth thinking through: what identifies the entity being tracked? The goal is cross-site AE signal aggregation per protocol — all adverse events for a given drug and trial visible in one query, regardless of site. The options were `protocol:{protocolId}` (a string requiring an extra JOIN to retrieve), something drug-name-based (drugs aren't first-class entities in this model), or `trial:{trialId}`. The trial UUID is the right choice: it's the correct aggregate scope for a single protocol run, reachable in one extra query from `TrialSite`, and unambiguous within a tenant.

The IRB domain had a simpler structural gap. `IrbApproval` had `deviationId` but not `deviationType` — and the IRB domain needs to aggregate by type, since the whole point is precedent per deviation category. The options were an extra DB query on every write, or adding `deviation_type` to `IrbApproval` directly. Adding it is the right schema: an IRB review is for a deviation type, and carrying that field on the entity removes a dependency on loading the related `ProtocolDeviation` every time a fact is written. V117 adds the column; `IrbDeviationCaseService` populates it from the event that triggered the case.
