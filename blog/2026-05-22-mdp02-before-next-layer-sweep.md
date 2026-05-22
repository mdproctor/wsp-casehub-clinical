---
layout: post
title: "Before the Next Layer, Sweep the Floor"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [clinical]
tags: [documentation, tutorial, layer-log]
---

Before starting Epic 6 — the IRB gate — I wanted to verify the engine-side wiring was
actually there, not just assumed.

The short answer: `WorkItemLifecycleAdapter` in `engine/work-adapter/` is fully implemented.
It listens for `WorkItemLifecycleEvent`, translates terminal statuses to PlanItem transitions,
and fires `CONTEXT_CHANGED` on the event bus. The Javadoc even cites `casehubio/work#136`
directly — the issue that closed last session. Verification took about thirty seconds.

That done, I looked at whether LAYER-LOG.md was actually up to date. It wasn't quite.
`AdverseEventLedgerWriter` — extracted from `AdverseEventService` to own sequenceNumber
computation — had landed in the last session's commits but wasn't in Layer 4's key files list.
Small gap, worth fixing before the next layer starts building on it.

## What AML has that clinical didn't

The deeper work was comparing clinical's CLAUDE.md against AML's. AML is ahead
on tutorial discipline — not because the code is more tutorial-focused, but because
the documentation makes it easier to navigate with that framing in mind.

The most useful thing AML has is a concern-driven lookup table: before designing X,
read Y. "Deciding which layer a feature belongs in → tutorial-strategy.md §6." Clinical
had the same reference documents listed, but as a flat table. The flat table tells you
what exists; the concern table tells you when to reach for it. These are different things.

Clinical also had no Tutorial Structure summary visible in CLAUDE.md — the numbered
layer table with completion status only existed in LAYER-LOG.md. Fine for deep reading,
invisible for orientation. We added both.

The interesting wrinkle is that clinical can't replicate AML's core tutorial mechanism:
the `NaiveXxxService @DefaultBean` CDI displacement pattern. AML teaches layer transitions
by displacement — each new layer implements the same interface, and CDI priority does the
rest. Clinical uses Panache Active Record directly, with no downstream JPA consumers, so
there's no interface to displace through. The layers here are additive — new services bolt
on, compliance chains grow — rather than substitutive. That's documented now in both
CLAUDE.md and LAYER-LOG.md, not as an apology but as an architectural fact that shapes
how the tutorial reads differently from AML.
