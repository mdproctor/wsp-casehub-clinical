---
layout: post
title: "What a sub-case is for"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [casehub-engine, quarkus, jq, agroal, design]
---

The original description for Epic 3 was "multi-site sub-case structure." It sat in the backlog for three months, blocked on an engine dependency. When that dependency shipped last week and we started the brainstorm, I asked a question I probably should have asked in March.

What does "a site is a sub-case" actually mean?

## What a site sub-case would have done

In the engine model, a sub-case is bounded delegated work. A parent starts a child, the child runs, the child completes, the parent is notified. That lifecycle fits a patient screening process — start it, get an answer, continue. It doesn't fit an investigational site that may be running for two years.

We asked what bindings the site sub-case would have. AE escalation is handled by per-AE engine cases from Layer 5. PI authorisation is a Qhorus commitment. IRB consultation is its own case. A site sub-case would be a structural container with no bindings of its own.

The actual gap — what ClinicalAgent can't do — is detecting cross-site safety patterns. Site A has a Grade 4 AE. Site B has a Grade 4 AE. No individual site agent sees both; a trial coordinator does. That's the thing worth building.

So no site sub-cases. Sites are domain entities. One trial `CaseInstance`, per trial. Per-site safety signals go in via `runtime.signal(trialCaseId, "grade4Active.<siteId>", Boolean)`. When two sites are flagged simultaneously, a DSMB rollup binding fires. The engine detects the cross-site pattern from accumulated blackboard state, without any site-level agent knowing about any other site.

The sub-case model turns out to be right for a later layer — patient batch screening, where you actually want to delegate bounded work, wait for results, and reaggregate. Not here.

## Two failures the tests can't see

Then implementation produced two problems the test suite can't tell you about.

The first was in the JQ binding filter. The DSMB binding never fired. No error. We added logging, checked the context — the signals were arriving, the values were there. We read the engine source.

```
# silently evaluates to false
[.grade4Active // {} | to_entries | select(.value == true) | length] >= 2

# correct
[.grade4Active // {} | to_entries[] | select(.value == true)] | length >= 2
```

`to_entries` returns an array. `select` applied to an array tests the array as a whole — `.value == true` with `.` being the array is `null == true`, which is false. The entire array discarded. Nothing to count. One character difference: `[]` after `to_entries` to iterate elements rather than test the collection.

The second was in `TrialActivationService`. The original design called `startCase().toCompletableFuture().join()` inside a `@Transactional` method. Tests were green.

`@Transactional` holds an Agroal connection from the first Panache call. `startCase()` is async and uses Vert.x event bus `request()` — which waits for `CaseStartedEventHandler` to reply. That handler writes to JPA repositories. Same pool. Under pool exhaustion, the calling thread holds the only available connection while the handler waits for one that can't be released until the handler completes. Deadlock.

Tests use memory stores. No JDBC calls, no pool contention. Green.

The fix is three-phase activation: commit the status change, call `join()` outside any transaction, commit the returned caseId. The `activate()` method itself is not `@Transactional`. It's the same pattern as last week — something the production build discovers that the test suite never will.