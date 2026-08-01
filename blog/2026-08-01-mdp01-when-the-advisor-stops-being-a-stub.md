---
title: "When the advisor stops being a stub"
date: 2026-08-01
author: mdp
tags: [llm, spi, agent-provider, clinical, protocol-amendment]
---

# When the advisor stops being a stub

Layer 9 shipped with a stub. `DefaultProtocolAmendmentAdvisor` always returned PROCEED
regardless of trial context — a placeholder so the showcase could run while the LLM
integration waited on engine#101. Today that stub gets displaced by an implementation
that actually reasons about safety data.

## What changed

`LlmProtocolAmendmentAdvisor` implements the same SPI, injected via CDI displacement
(no config change, no bean selection — `@ApplicationScoped` automatically wins over
`@DefaultBean`). It calls `AgentProvider.invoke()` with a clinical-domain system prompt,
passes the trial's accumulated safety profile, and parses the LLM's recommendation:
PROCEED, REFER_TO_DSMB, or HALT.

The interesting part isn't the LLM call itself — it's the three non-obvious problems
that surfaced getting it to work.

## Problem 1: the dependency that compiles but doesn't

`casehub-platform-agent-api` appeared in `mvn dependency:tree` but its classes were
invisible at compile time. The JAR was in the local repo. The dependency was declared
in the POM. The error — `cannot find symbol: AgentProvider` — made no sense.

The cause: the parent POM's `dependencyManagement` declares `casehub-platform-agent-api`
with `runtime` scope. Adding it as a direct dependency without explicit `<scope>compile</scope>`
silently inherits `runtime`. Maven's default scope is `compile`, so you'd expect an
explicit `<dependency>` to override the managed scope — it doesn't. The managed scope wins.

**The fix is one XML element.** But finding it cost an hour of staring at dependency trees
wondering why a JAR that exists isn't visible.

## Problem 2: the worker that runs in no-man's land

The `ProtocolAmendmentCaseHub` enriches the advisor's context with trial data — AE counts,
grade distribution, prior amendments. This requires Panache queries. The queries worked in
tests. They worked when called from REST endpoints. They failed at runtime with:

> Cannot use the EntityManager/Session because neither a transaction nor a CDI request
> context is active.

Engine worker functions execute on Quartz scheduler threads. These threads have no JTA
transaction and no CDI request context. The worker function is a plain `Function` — not
a CDI bean method — so `@Transactional` can't be applied. Adding `@Transactional` to a
private method called from the lambda doesn't work either (CDI self-invocation bypass).

The fix: `QuarkusTransaction.requiringNew().call(() -> ...)` — programmatically begins a
transaction regardless of thread context. Simple once you know it, invisible until you
hit the error.

## Problem 3: the SPI that shipped somewhere else

engine#101 was supposed to deliver `LlmPlanningStrategy` — a single SPI for LLM-backed
worker selection. What actually shipped was architecturally richer: `RoutingStrategy<T>`
(agent selection) and `DecompositionStrategy<T>` (goal decomposition) in `casehub-blocks`.

Neither fits the clinical use case. Protocol amendment advising isn't routing (we're not
choosing between multiple advisors) and it isn't decomposition (no subtask breakdown).
It's a single-shot classification: given accumulated trial signals, what's the
recommendation?

The right primitive was `AgentProvider` — the platform-level LLM invocation API that both
routing and decomposition build on. Direct call, no orchestration overhead.

## Why this matters

The clinical harness demonstrates that a regulated domain's advisory slot can move from
stub to LLM-backed without changing any caller. The engine case, the YAML binding, the
REST endpoint, the ledger audit trail — all unchanged. The only change is a new CDI bean
that implements an existing SPI.

This is the CDI displacement pattern doing exactly what it was designed for: the system
transitions from deterministic-stub to LLM-backed by dropping a new class on the classpath.
The fallback chain (PROCEED on empty response, parse failure, or invocation error) means
the system degrades to stub behaviour if the LLM is unavailable — the same safety property
the stub provided, without the limitation of always saying PROCEED.

For anyone building regulated AI systems: the SPI boundary between "domain decision logic"
and "how the decision is made" is the entire architecture. Get it right and the LLM is a
pluggable implementation detail. Get it wrong and the LLM is the architecture.
