# 0001 — Layer 5 Engine Case Granularity: Per-Event Cases Over Per-Site Case

Date: 2026-05-23
Status: Accepted

## Context and Problem Statement

Layer 5 introduces casehub-engine to casehub-clinical. The engine needs CasePlanModel
definitions for the two new compliance gates: the IRB consultation gate (CRITICAL deviation
path) and the adverse event escalation routing. The architectural question is what entity
one engine case should represent: one per clinical event (deviation, AE) or one per trial
site accumulating all events.

## Decision Drivers

* Engine cases are bounded processes with clear start, execution, and completion — not
  containers for unbounded streams of events
* Layer 5 tutorial goal: demonstrate adaptive routing clearly without incidental complexity
* Layer 6 (engine#112) adds multi-site sub-case orchestration — design must not foreclose it
* Per-site long-running cases without a parent trial case would be architecturally incomplete
  (orphaned sub-cases with no parent defining their completion goals)

## Considered Options

* **Option A — Per-event cases** — one `ClinicalDeviationCaseHub` per CRITICAL deviation;
  one `ClinicalAdverseEventCaseHub` per Grade 3+ AE. Each case has clear start, bindings,
  completion goals, and terminal state.
* **Option B — Per-site long-running case** — one case per trial site, accumulating all
  AEs and deviations as bindings. Multiple events tracked via event-keyed context. Never
  completes during the trial.
* **Option C — Single unified clinical case** — one case type handling both IRB deviations
  and AE routing; initial context determines which bindings fire.

## Decision Outcome

Chosen option: **Option A (per-event cases)**, because engine cases are bounded processes,
not containers. A per-site case without the trial-level parent case (Layer 6) is
architecturally incomplete. Per-event cases demonstrate the engine's conditional binding
evaluation clearly without the accidental complexity of event-keyed context tracking.

### Positive Consequences

* Bindings have trivially simple conditions (one case = one event, no historical tracking)
* Cases complete when the event is resolved (days to weeks) — no unbounded runtime
* Migration path to Layer 6 is structural promotion: per-event cases become bindings within
  site sub-cases, which become sub-cases of the trial case
* Two distinct case types with clear names make the tutorial teaching objective obvious

### Negative Consequences / Tradeoffs

* Two case types instead of one; small duplication in wiring (two YamlCaseHub subclasses,
  two CDI observers, two YAML files)
* Accumulated context across events (cross-AE patterns) is not available within a single
  case — that requires Layer 6 site-level sub-cases

## Pros and Cons of the Options

### Option A — Per-event cases

* ✅ Engine cases have clear completion goals and bounded runtime
* ✅ Binding conditions are simple — no event-keyed context
* ✅ Compatible with Layer 6 sub-case promotion without redesign
* ❌ Two case types; separate observers for each event type

### Option B — Per-site long-running case

* ✅ Architecturally correct shape for Layer 6 (site-level orchestration)
* ❌ Orphaned without the trial-level parent case — structurally incomplete in Layer 5
* ❌ Event-keyed context (tracking which AEs have been processed) is accidental complexity
* ❌ No completion goals — cases run indefinitely during the trial

### Option C — Single unified clinical case

* ✅ One YAML definition for both event types
* ❌ Mixed-trigger case: starts from either deviation resolution or AE reporting — confused identity
* ❌ Bindings must account for "was this triggered by an AE or a deviation?" — implicit type routing
* ❌ Merges two genuinely distinct clinical processes into one definition

## Links

* casehubio/clinical#6 — Layer 5 implementation (Epic 6)
* engine#112 — sub-case execution wiring (blocked; unblocks Layer 6 per-site cases)
* Spec: `specs/issue-6-irb-gate/2026-05-22-irb-gate-ae-escalation-design.md`
