---
layout: post
title: "Services Don't Know HTTP"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [quarkus, jax-rs, layering, code-review, regulatory]
---

`TrialActivationService` was throwing `ClientErrorException`. Not via an exception
mapper, not through any abstraction — a raw JAX-RS exception thrown from an
`@ApplicationScoped` service bean that imported `jakarta.ws.rs.core.Response`.

The obvious fix is to move the validation to the resource layer. Check if the trial
exists, check its status, return the appropriate response. But that breaks the
three-phase activation pattern: the status check and the status update need to be
atomic in the same `@Transactional` boundary. Move the check to the resource and
another request can slip in between. We've documented this pattern three times now
in LAYER-LOG and ADR 0004 — and then violated it in the same file.

The actual fix is inner static exception classes on the service:

```java
public static class TrialNotFoundException extends RuntimeException {
    private static final long serialVersionUID = 1L;
    public TrialNotFoundException(UUID id) { super("Trial not found: " + id); }
}
```

`RuntimeException` subclass means `@Transactional` auto-rolls back. Inner static
means the exception is co-located with the code that throws it. The resource catches
and maps to HTTP — which is where that concern belongs.

The coupling was flagged as a code review finding from an earlier session and deferred.
Small issues have a way of waiting until they accumulate enough company to be worth
fixing in batch. This session cleared four of them.

The more interesting catch came from code review on this session's own changes. I
dispatched a reviewer Claude and it came back with two things I hadn't caught.

The first: the debug logging I'd added to `IrbDecisionListener`'s early-return paths
used `LOG.debugf`. The listener observes every `WorkItemLifecycleEvent` in the system
— AE escalations, PI authorisations, DSMB reviews. The non-IRB early-return fires for
all of them. In a production debug session, enabling DEBUG for that class would produce
one skip line per non-IRB WorkItem lifecycle transition. The IRB events — the ones you
actually want — would be invisible. `tracef`, not `debugf`.

The second: the Layer 7 teaching note I'd written for LAYER-LOG claimed that
`AeEscalationCompletedEvent` already carries enough information to trigger FDA IND
expedited reporting for Grade 5 adverse events. It doesn't. Under 21 CFR 312.32(c)(1)(i),
the expedited reporting obligation applies to *unexpected* fatal reactions — the
`unexpected` flag is required alongside grade. The event doesn't have that field. The
note now records this as an API gap for Layer 7, not a solved problem.

Writing regulatory documentation for a teaching project that targets regulated healthcare
is an interesting discipline. The claim has to be true.
