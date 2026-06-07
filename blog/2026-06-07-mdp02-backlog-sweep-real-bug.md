---
layout: post
title: "The backlog sweep that found a real bug"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [sealed-interfaces, flyway, exponential-backoff, code-review]
---

Sweeping the small issues backlog — eight entries, ranging from a compile error to a new feature — is the kind of work that's easy to defer. Nothing urgent, nothing architectural. But deferred small issues accumulate into a codebase that nobody quite trusts.

Most of the eight were exactly what they looked like. The `IrbDecisionListenerTest` compile error: `CallerRef` became a sealed interface with `PlanItemCallerRef` and `GateCallerRef` as its two permitted subtypes. The old `CallerRef.encode(UUID, String)` moved to `PlanItemCallerRef.encode()`. One line changed; the compiler flagged it. Sealed interfaces make API evolution visible immediately — breaking at compile time rather than silently passing at runtime is the value.

The Flyway migration renumbering was a protocol violation from the project's early weeks. Ledger subclass join tables belong in the V2000+ range; the early entries used V1005–V1014 instead, before the rule was written. One thing I noticed: the V2020 entry for sponsor notification ledger — added last session — was correct from day one. Whoever wrote the protocol applied it to new work immediately but didn't revisit what was already there.

The exponential backoff extension for `SponsorNotificationRetryPolicy` is where things got interesting. The policy is a `SingleValuePreference` with a two-field config string: `"3,30"` for three attempts at 30-minute intervals. I extended it to optionally accept `"3,15,2.0"` (with a backoff multiplier) or `"3,15,2.0,120"` (with a multiplier and a cap). Backward compatible: two-field strings parse the same as before.

The `computeDelay()` implementation had a fast path for multiplier=1.0:

```java
if (policy.backoffMultiplier() == 1.0) {
    return policy.retryInterval();  // cap never applied
}
// cap check only reaches here when multiplier > 1.0
```

Claude caught it in review. If someone configures `"3,60,1.0,30"` — a 60-minute interval with a 30-minute cap and no backoff — the cap is silently ignored. Nothing in the Javadoc on `maxInterval` says the cap is conditional on using backoff; a reader has no reason to expect otherwise. The fix moves the cap check outside the branch so it applies unconditionally.

It passes all unit tests easily. Testing multiplier=1.0 and testing maxInterval are both present; testing their cross-product with a cap below the fixed interval is not. The review caught it by reading the contract, not by running tests.

The WorkItem escalation on sponsor notification exhaustion was the other S-complexity item. When all delivery retries are consumed, a new listener creates a WorkItem for the `site-coordinators` group — 24 hours for CRITICAL severity deviations, 72 hours for others. The exhaustion itself already has a ledger entry; the WorkItem is the manual escalation path, not a second audit record.

Nothing about the session was architecturally notable. The bug in `computeDelay()` was real but small. What it does illustrate: a code reviewer reading the contract rather than the tests is sometimes the difference between shipping a correctness bug and not.
