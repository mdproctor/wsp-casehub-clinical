---
layout: post
title: "The Engine Arrives"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [clinical]
tags: [casehub-engine, IRB, adverse-event, layer-5, CDI]
---

Layer 5 is done. casehub-engine is in clinical, the IRB gate works end-to-end,
and Grade 3/4 adverse events now route adaptively rather than through a hardcoded
switch. 115 tests, all green.

## The decision that mattered before the first line of code

Before we touched the implementation, I had to decide what an engine case actually
represents in clinical. The obvious framing — one case per trial site, accumulating
all events — looked architecturally elegant. Events fire into a site case, context
builds up, the same bindings handle both deviations and AEs.

I rejected it. An engine case is a bounded process with clear completion goals,
not a container. A per-site case without the trial-level parent case is structurally
orphaned — you'd be building half of Layer 6 and calling it Layer 5. The per-event
approach (one case per CRITICAL deviation, one case per Grade 3+ AE) has trivially
simple binding conditions, completes when the event is resolved, and composes cleanly
into Layer 6's sub-case architecture when engine#112 unblocks it.

That decision is captured as ADR 0001. It was worth making explicit — the per-site
shape felt architecturally righteous in a way that needed pushing back on.

## The CDI hell

The implementation itself went fast once the engine was actually wiring. Getting
there was not fast.

casehub-engine landing in clinical's classpath brought three separate startup failures,
none of which had anything to do with the feature I was building.

The first was the most misleading: seven CDI deployment problems, all `Unsatisfied dependency for type JQEvaluator`. The instinct is to look at what changed — but `JQEvaluator` is an engine-internal type, and the engine had just been added. The actual cause was that `casehub-platform` and `casehub-platform-expression` are implicit requirements for the engine to start. They provide `@DefaultBean` mocks for injection points throughout the engine stack. Nothing in the engine documentation says this. I found it by diffing clinical's pom.xml against devtown, which already had the engine working.

The second was Quartz. `casehub-engine-scheduler-quartz` transitively pulls in
`quarkus-quartz`, which requires 6-7 field cron expressions. `casehub-work`'s
scheduler beans (`ExpiryCleanupJob`, `ClaimDeadlineJob`, `RoutingCursorCleanupJob`)
use standard Unix 5-field cron. Quartz fails at startup parsing them. The fix is
to exclude those beans in test properties and configure `quarkus.quartz.store-type=ram`
with `quarkus.scheduler.start-mode=forced`. Not documented anywhere.

The third was the approach a subagent took to fix problems one and two. It excluded
twenty-odd engine beans — including `CaseContextChangedEventHandler`, `CaseStartedEventHandler`,
the entire event handler stack — rather than finding the root cause. The tests passed
with that approach. They would have continued passing until we tried to actually
run an engine case in a test, at which point nothing would have worked. I caught it
before the integration tests existed and reversed it.

## The `when` field that does nothing

The implementation itself was straightforward: two YAML case definitions, four observer
classes, two ledger subclasses, an SPI. The only runtime surprise came in
`AeEscalationLifecycleTest` when Grade 3 AEs were creating DSMB escalation WorkItems
they shouldn't.

The `ae-escalation.yaml` had this binding:

```yaml
- name: dsmb-escalation
  on: { contextChange: {} }
  when: ".requiresDsmbEscalation == true and .dsmbEscalation == null"
```

`when` is silently ignored for `contextChange` triggers. `CaseContextChangedEventHandler`
evaluates `on.contextChange.filter`, not `binding.getWhen()`. The `when` field is
only evaluated by the scheduler-side handler. Moving the condition to
`on.contextChange.filter:` fixed it. Filed as engine#335.

## What the integration tests verify

`IrbGateLifecycleTest` has two paths: APPROVED (case completes via `outputMapping`
writing `irbConsultation`) and EXPIRED (no `outputMapping` fires on expiry;
`IrbDecisionListener` signals the case directly with `caseHub.signal(caseId, "irbConsultation", ...)`
to satisfy the `irb-decided` goal).

`AeEscalationLifecycleTest` has Grade 3 (one safety-review WorkItem, case completes
when it resolves) and Grade 4 (two parallel WorkItems — `senior-safety-monitors` and
`dsmb` — case completes when both resolve). Same YAML, different initial context.

The code reviewer Claude ran after implementation caught three real issues: missing
`@Transactional` on `AeEscalationListener`, a null `enrollmentId` path that would
have hit a `nullable = false` column, and `clock.instant()` called twice in the ledger
writer. The first two were genuine data-integrity risks.

## What's left

Layer 6 (multi-site sub-case orchestration) is blocked on engine#112. The foundation
for it is here — per-event cases become bindings within site sub-cases when that
unblocks. Layer 7 is trust routing plus the ClinicalAgent comparison.

The `AdverseEventEscalationPolicy` SPI is the right shape. Every routing decision
flows through it: candidateGroups for Grade 1/2, case context keys for Grade 3+.
Organisations override it entirely. The tutorial doesn't demonstrate that extensibility
yet — it just shows the default — but the boundary is drawn correctly.
