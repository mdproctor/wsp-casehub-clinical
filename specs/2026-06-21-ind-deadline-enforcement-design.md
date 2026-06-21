# IND Reporting Deadline Enforcement — Design Spec
**Issue:** casehubio/clinical#83  
**Date:** 2026-06-21  
**Scope:** casehub-engine (SPI + scheduler) + casehub-clinical (YAML + compliance layer)

---

## Problem

`RegulatorySubmissionCaseService` correctly computes `indReportingDeadline` (`ae.reportedAt + window`) and places it in the case context. The `regulatory-submission.yaml` uses a `capability` binding with no deadline enforcement. If the filing agent misses the window, nothing in the platform detects it.

The FDA deadline is measured from AE report time — not from case start or WorkItem creation. A relative `expiresIn: P7D` from WorkItem creation is structurally wrong: it measures from the wrong origin and diverges when there is any processing delay.

The correct enforcement invariant: `WorkItem.expiresAt = ae.reportedAt + indReportingWindow(grade)` — exact, absolute, from the domain clock.

---

## Engine design — `expiresAtExpression` via pluggable `ExpressionEngine.extractString()`

### Why not `PropagationContext`

`PropagationContext.createRoot(attrs, budget)` computes `deadline = now() + budget`. To use it for absolute deadline enforcement, the service would compute `budget = indDeadline - now()`, introducing clock drift across two `Instant.now()` calls. More fundamentally, it encodes an absolute regulatory deadline as a resource budget — a semantic mismatch — and requires Java service changes for information that is purely YAML-declarative. The deadline is a property of the humanTask binding, not the case's execution budget.

### New SPI: `ExpressionEngine.extractString()`

The existing `ExpressionEngine` SPI handles only boolean evaluation (`evaluate(ExpressionEvaluator, CaseContext): boolean`). Value-typed extraction (returning a `String`) is a distinct capability. Adding it as a default `default` method lets existing engine implementations compile unmodified:

```java
// ExpressionEngine.java
default Optional<String> extractString(ExpressionEvaluator evaluator, CaseContext context) {
    throw new UnsupportedOperationException(
        "ExpressionEngine '" + type() + "' does not support string extraction. " +
        "Override extractString() to enable this capability.");
}
```

`ExpressionEngineRegistry` gains the same dispatch method. `DefaultExpressionEngineRegistry` implements it: find the matching engine by type, call `engine.extractString()`, catch `UnsupportedOperationException` (engine doesn't override the default), log WARN, return `Optional.empty()`. The distinction between "no engine registered" (throw `IllegalArgumentException` — programming error) and "engine registered but doesn't support string extraction" (graceful `Optional.empty()` + WARN) is preserved.

`JQExpressionEngine` overrides `extractString()`. Evaluates against the **WORKING panel** — same as `evaluate()` and `evaluateInputMapping()`:

```java
@Override
public Optional<String> extractString(ExpressionEvaluator evaluator, CaseContext context) {
    final String expr = ((JQExpressionEvaluator) evaluator).expression();
    if (expr == null || expr.isBlank()) return Optional.empty();
    final ValidationResult vr = jqEvaluator.eval(expr, context.panel(ContextPanel.WORKING).asJsonNode());
    // Guard matches evaluateInputMapping() pattern — empty output means field absent or JQ error
    if (!vr.ok() || vr.output() == null || vr.output().isEmpty()) {
        if (!vr.ok()) LOG.warnf("extractString JQ evaluation failed: %s", vr.error());
        return Optional.empty();
    }
    final JsonNode result = vr.output().get(0);
    // isTextual() guard: NullNode.asText() returns "null" (the string), NumericNode returns digits
    // — both would reach Instant.parse() and throw without this guard
    if (!result.isTextual()) return Optional.empty();
    return Optional.of(result.asText());
}
```

**WORKING panel contract:** `expiresAtExpression` JQ expressions evaluate against the WORKING panel, same as filter conditions. `indReportingDeadline` is in the WORKING panel (placed there by `prepareAndMark()` as part of the initial context map). A deadline in a non-WORKING panel would silently produce `Optional.empty()` — same silent-SLA-gap that load-time JQ validation was added to prevent for syntax errors.

This is fully pluggable: any `ExpressionEngine` CDI bean can support value extraction by overriding `extractString()`. No `instanceof` anywhere.

### `HumanTaskTarget.expiresAtExpression`

Add `expiresAtExpression: ExpressionEvaluator` to `HumanTaskTarget` — same type and builder pattern as `inputMapping`/`outputMapping` (both `String` and `ExpressionEvaluator` builder overloads). The `String` overload wraps in `JQExpressionEvaluator` — same as `inputMapping`:

```java
public Builder expiresAtExpression(String expression) {
    this.expiresAtExpression = new JQExpressionEvaluator(expression);
    return this;
}
public Builder expiresAtExpression(ExpressionEvaluator evaluator) {
    this.expiresAtExpression = evaluator;
    return this;
}
```

### Schema → mapper → event → handler

**`CaseDefinition.yaml` (schema source):** Add `expiresAtExpression: string` to the `HumanTask` definition. `HumanTask.java` is regenerated by Maven from this source — the new `String expiresAtExpression` field follows the same pattern as `String expiresIn`.

**`CaseDefinitionYamlMapper.convertHumanTask()`:** `convertHumanTask()` is private static and has no access to `ExpressionEngineRegistry`. `inputMapping` and `outputMapping` both bypass the registry and call `Builder.inputMapping(String)` → `new JQExpressionEvaluator(expression)` directly. `expiresAtExpression` follows the same pattern:

```java
if (schema.getExpiresAtExpression() != null && !schema.getExpiresAtExpression().isBlank()) {
    // Validate JQ syntax at load time — silent null at runtime is a regulatory SLA failure
    try {
        net.thisptr.jackson.jq.JsonQuery.compile(schema.getExpiresAtExpression(), net.thisptr.jackson.jq.Versions.JQ_1_6);
    } catch (Exception e) {
        throw new IllegalArgumentException(
            "invalid expiresAtExpression '" + schema.getExpiresAtExpression() + "' — " + e.getMessage(), e);
    }
    builder.expiresAtExpression(schema.getExpiresAtExpression());
}
```

Fail-fast at YAML load is the correct invariant. An invalid expression that silently produces `Optional.empty()` at runtime leaves the WorkItem with no absolute deadline — a silent regulatory SLA gap that is only discovered at breach time.

**`HumanTaskScheduleEvent`:** Add `expiresAtDeadline: Instant` field (null when no expression or evaluation fails). The expression is evaluated upstream, in `CaseContextChangedEventHandler`, so `HumanTaskScheduleHandler` remains expression-free.

**`CaseContextChangedEventHandler.publishHumanTaskSchedule()`:** After resolving `caseBudgetDeadline`, add:

```java
final Instant expiresAtDeadline = resolveExpiresAtDeadline(caseInstance, target);
```

New private method:
```java
private Instant resolveExpiresAtDeadline(CaseInstance caseInstance, HumanTaskTarget target) {
    if (target.expiresAtExpression() == null) return null;
    return registry.extractString(target.expiresAtExpression(), caseInstance.getCaseContext())
        .map(s -> {
            try { return Instant.parse(s); }
            catch (Exception e) {
                LOG.warnf("expiresAtExpression result '%s' is not a valid ISO-8601 instant — ignoring", s);
                return null;
            }
        })
        .orElse(null);
}
```

The expression evaluates against the **full case context** (`caseInstance.getCaseContext()`), not against `inputData`. `indReportingDeadline` is a field in the case context placed there by `RegulatorySubmissionCaseService`. Pass `expiresAtDeadline` in the `HumanTaskScheduleEvent` constructor.

**`HumanTaskScheduleHandler.createInline()`:** Add `Instant expiresAtDeadline` as a new parameter, passed from `handleInlineMode()` via `event.expiresAtDeadline()`. Fold into `earliestOf` chain:

```java
private void createInline(
    HumanTaskTarget target,
    Map<String, Object> inputData,
    Set<String> resolvedGroups,
    Set<String> resolvedUsers,
    String callerRef,
    Instant expiresAtDeadline,       // new
    Instant caseBudgetDeadline) {
  Instant taskDeadline = target.expiresIn() != null ? Instant.now().plus(target.expiresIn()) : null;
  Instant effectiveDeadline = earliestOf(earliestOf(taskDeadline, expiresAtDeadline), caseBudgetDeadline);
  ...
}
```

**Template mode (`handleTemplateMode`):** Template-mode WorkItems must also respect `expiresAtDeadline`. After `workItemTemplateService.instantiate()`, apply:

```java
if (event.expiresAtDeadline() != null) {
    workItem.expiresAt = workItem.expiresAt == null
        ? event.expiresAtDeadline()
        : earliestOf(workItem.expiresAt, event.expiresAtDeadline());
}
```

This preserves the template's own `expiresAt` when it is earlier, enforcing that a template's deadline can never be extended by the binding's expression — and that the binding's absolute deadline is honoured even if the template has a later one. The regulatory submission case uses inline mode; template mode support is included for correctness.

---

## Clinical design

### `regulatory-submission.yaml`

Remove the `capabilities` section (no longer used). Change the `file-ind-report` binding from `capability: regulatory-submission` to `humanTask`:

```yaml
bindings:
  - name: file-ind-report
    on:
      contextChange:
        filter: ".grade != null and .submissionFiled == null"
    humanTask:
      title: "IND Expedited Safety Report — 21 CFR 312.32"
      expiresAtExpression: ".indReportingDeadline"
      candidateGroups: [regulatory-affairs]
      inputMapping: "{ aeId: .aeId, grade: .grade, unexpected: .unexpected, indReportingDeadline: .indReportingDeadline }"
      outputMapping: "{ submissionFiled: . }"
```

`expiresAtExpression: ".indReportingDeadline"` evaluates against the case context at scheduling time, returning the ISO-8601 deadline string placed there by `RegulatorySubmissionCaseService`. The engine parses it to `Instant` and sets `WorkItem.expiresAt = indReportingDeadline` exactly. No Java service changes required.

The `goals` and `completion` sections are unchanged: `submission-complete: .submissionFiled != null` fires when the humanTask WorkItem is completed and `outputMapping` writes `submissionFiled` to the case context.

### `RegulatorySubmissionStatus` enum

`FILED` already exists. Add only `DEADLINE_MISSED` (IND deadline exhausted without submission). New value is additive — no Flyway migration required for the enum column (persisted as `VARCHAR`).

### `ClinicalIndReportingBreachPolicy`

**CDI wiring:** `@ApplicationScoped` (no `@DefaultBean`). `ExpiryLifecycleService` injects `@Inject SlaBreachPolicy slaBreachPolicy` as a singular point — exactly one non-default bean resolves, displacing `NoOpSlaBreachPolicy @DefaultBean`.

Implements `SlaBreachPolicy`. The policy is **pure** — makes a decision and returns, no side effects, no CDI service calls, no DB queries.

**Discrimination:** `BreachedTask` exposes only `taskId`, `callerRef`, `title`, `candidateGroups`. There is no payload field; grade is inaccessible. Grade-specific escalation timing therefore cannot be expressed here. The policy uses `candidateGroups` to discriminate regulatory WorkItems:

```java
public BreachDecision onBreach(SlaBreachContext ctx) {
    // After EscalateTo executes, ExpiryLifecycleService replaces candidateGroups with the
    // escalation group — "regulatory-affairs" is gone on the second breach. Both groups
    // must be tested to identify regulatory WorkItems across both breach tiers.
    boolean isRegulatory = ctx.task().candidateGroups().contains("regulatory-affairs")
                        || ctx.task().candidateGroups().contains("regulatory-leadership");
    if (!isRegulatory) {
        return new Fail("no-sla-breach-policy-configured");
    }
    // Second breach: regulatory-leadership was already assigned — exhaustion
    if (ctx.task().candidateGroups().contains("regulatory-leadership")) {
        return new Exhausted("IND reporting deadline exhausted — operator intervention required");
    }
    // First breach: escalate to regulatory leadership with 48h window
    return EscalateTo.to("regulatory-leadership").withDeadline(Duration.ofHours(48));
}
```

**Why the `isRegulatory` guard must test both groups:** `executeEscalateTo()` in `ExpiryLifecycleService` replaces `item.candidateGroups` with `String.join(",", escalate.groups())`. After the first escalation fires, `candidateGroups = {"regulatory-leadership"}` — `"regulatory-affairs"` is gone. Without the combined test, the policy would return `Fail` for its own escalated WorkItem and the `Exhausted` branch would be unreachable.

**Single fixed 48h escalation window for all grades.** Grade-specific logic (e.g. Grade 5 needing faster response) belongs in the `RegulatorySubmissionBreachListener` (see below) where `AdverseEvent.grade` is available via DB, not in the pure policy. The 48h window gives regulatory leadership time to initiate a late filing with explanation regardless of grade.

**For non-regulatory WorkItems:** `Fail("no-sla-breach-policy-configured")` — identical to `NoOpSlaBreachPolicy` default. This makes the policy's scope explicit: it owns regulatory-affairs WorkItems and explicitly defers all others to platform default behaviour. Do NOT use `Chained` — `Chained(primary, fallback)` is a two-armed decision combinator that tries the primary decision; it is not a pass-through.

### `RegulatorySubmissionCompletedListener`

Observes `CaseLifecycleEvent` for `GoalReached` and `CaseCompleted` event types. `CaseLifecycleEvent` does **not** carry a case name or namespace — discrimination is entirely via DB lookup:

```java
public void onCaseLifecycleEvent(@ObservesAsync CaseLifecycleEvent event) {
    if (!"GoalReached".equals(event.eventType()) && !"CaseCompleted".equals(event.eventType())) return;
    markFiled(event.caseId());
}

@Transactional
void markFiled(UUID caseId) {
    AdverseEvent ae = AdverseEvent.find("regulatorySubmissionCaseId", caseId).firstResult();
    if (ae == null) return;  // not a regulatory submission case
    // Guard: only process if still PENDING — protects against DEADLINE_MISSED being overwritten
    // and against CDI at-least-once re-delivery
    if (ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.PENDING) return;
    ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.FILED;
    ledgerWriter.writeFiledEntry(ae);
}
```

### `RegulatorySubmissionBreachListener`

Observes `WorkItemLifecycleEvent` for `ESCALATED` status. Discriminates via `callerRef` → `caseId` → `AdverseEvent.find("regulatorySubmissionCaseId", caseId)`.

**API constraints established from source inspection:**
- `WorkItemLifecycleEvent` has no `callerRef()` method. The embedded `WorkItem` entity is accessible via `event.source()` — null for wire-reconstructed events.
- `WorkItemLifecycleEvent.detail()` is always `null` for `ESCALATED` events: `ExpiryLifecycleService.fireLifecycleEvent()` calls `WorkItemLifecycleEvent.of(event, item, "system", null)` (hardcoded null detail). The `Exhausted(reason)` string goes to the audit log only, not to the lifecycle event.
- `CallerRef.parse(String)` already exists in `io.casehub.workadapter` — a sealed interface with `PlanItemCallerRef` and `GateCallerRef` subtypes. `CallerRef.caseId()` extracts the UUID from either. **No engine addition is needed.** This is the same API used by `IrbDecisionListener` (line 78: `CallerRef ref = CallerRef.parse(workItem.callerRef)`).
- Use `instanceof` pattern matching for `event.source()` — matches `IrbDecisionListener` pattern, null-safe, prevents `ClassCastException` if the contract changes.

```java
public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
    if (event.status() != WorkItemStatus.ESCALATED) return;
    if (!(event.source() instanceof WorkItem workItem)) return;
    CallerRef ref = CallerRef.parse(workItem.callerRef);
    if (ref == null) return;
    markDeadlineMissed(ref.caseId());
}

@Transactional
void markDeadlineMissed(UUID caseId) {
    AdverseEvent ae = AdverseEvent.find("regulatorySubmissionCaseId", caseId).firstResult();
    if (ae == null) return;
    // Guard: only process if still PENDING — protects against FILED being overwritten
    // and against CDI at-least-once re-delivery
    if (ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.PENDING) return;
    ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.DEADLINE_MISSED;
    ledgerWriter.writeBreachEntry(ae);
}
```

`ESCALATED` status fires when `Exhausted` is returned by the policy (all escalation groups exhausted). `ExpiryLifecycleService.executeExhausted()` calls `fireLifecycleEvent("ESCALATED", item)` — the CDI event fires from casehub-work and is observed by clinical directly (`WorkItemLifecycleAdapter` in the engine ignores ESCALATED for PlanItem transitions, but the CDI event itself is still published). Grade is available via `AdverseEvent.grade` for any grade-specific breach handling in the ledger entry.

### Ledger — new subclasses

`LedgerEntryType` has three values: `COMMAND`, `EVENT`, `ATTESTATION`. Both the obligation entry (existing) and the new completion/breach entries are `EVENT` — the enum cannot distinguish them. **New JPA subclasses are required.** The feedback's suggestion of "same subclass, different LedgerEntryType" is not implementable with the current enum.

**`IndReportFiledLedgerEntry`** — `@DiscriminatorValue("IndReportFiled")`:
- `subjectId = ae.enrollmentId` (inherited from `LedgerEntry` — **must be set explicitly**; the Merkle chain groups by subjectId; null breaks enrollment audit chain continuity)
- `aeId UUID NOT NULL`
- `grade VARCHAR(20) NOT NULL`
- `submittedAt TIMESTAMP NOT NULL` — actual filing time (`clock.instant()` at write time)
- `domainContentBytes()`: `aeId + grade + submittedAt.toString()`
- V2026 migration: `ind_report_filed_ledger_entry` join table

**`IndReportBreachLedgerEntry`** — `@DiscriminatorValue("IndReportBreach")`:
- `subjectId = ae.enrollmentId` (same Merkle chain grouping requirement as above)
- `aeId UUID NOT NULL`
- `grade VARCHAR(20) NOT NULL`
- `breachedAt TIMESTAMP NOT NULL` — time of ESCALATED transition (`clock.instant()` at write time)
- `breachReason VARCHAR(255)` — fixed string `"IND reporting deadline exhausted without submission"` (not taken from `WorkItemLifecycleEvent.detail()` which is always null for ESCALATED events; `ExpiryLifecycleService` hardcodes `null` as the detail in `fireLifecycleEvent()`)
- `domainContentBytes()`: `aeId + grade + breachedAt.toString()`
- V2027 migration: `ind_report_breach_ledger_entry` join table

**Note on existing `RegulatorySubmissionLedgerEntry.filedAt`:** This column is named `filed_at` but currently set to `now` at obligation identification time — before any filing occurs. The column name implies the IND was filed at that moment, which is incorrect. This is a pre-existing semantic issue; the existing entry's `filedAt` means "obligation identified at." Not fixed in this issue but noted here to avoid confusion: `IndReportFiledLedgerEntry.submittedAt` is semantically correct (actual filing time), while the original `filed_at` is not.

**`RegulatorySubmissionLedgerWriter` additions:**

```java
@Transactional(TxType.MANDATORY)
public void writeFiledEntry(AdverseEvent ae) { ... }   // IndReportFiledLedgerEntry, submittedAt = now

@Transactional(TxType.MANDATORY)
public void writeBreachEntry(AdverseEvent ae) { ... }  // IndReportBreachLedgerEntry, breachReason = fixed string
```

**ComplianceSupplement factory methods:**
- `writeFiledEntry()` attaches `ClinicalComplianceSupplement.regulatorySubmission(ae.grade)` — same regulatory obligation, same algorithm (the case service that started the obligation)
- `writeBreachEntry()` attaches a new `ClinicalComplianceSupplement.regulatorySubmissionBreach(ae.grade)` factory method with `algorithmRef = "ClinicalIndReportingBreachPolicy — IND deadline exhausted"` and the same grade-specific 21 CFR `planRef` as `regulatorySubmission()`. The actor is different (the breach policy and expiry service, not the submission case service); the supplement must reflect that.

Both set `subjectId = ae.enrollmentId` and call `nextSequenceNumber(ae.enrollmentId, "default")` — same as the existing `writeEntry()`.

### `RegulatorySubmissionCaseService`

**No changes.** The service continues to call `regulatorySubmissionCaseHub.startCase(initialContext)` unchanged. `indReportingDeadline` is already in the initial context; the YAML and engine handle the deadline. The service is not involved in breach or completion handling.

---

## Flyway migrations required

Contrary to the initial spec's claim, **two new Flyway migrations are needed** for the new ledger subclass join tables:

| Migration | Datasource | Table | Version |
|-----------|-----------|-------|---------|
| V2026 | qhorus | `ind_report_filed_ledger_entry` | new |
| V2027 | qhorus | `ind_report_breach_ledger_entry` | new |

V2024 (`eligibility_screening_ledger_entry`) and V2025 (`protocol_amendment_ledger_entry`) are already taken by Layer 9 (clinical#10).

No migration is needed for `RegulatorySubmissionStatus.DEADLINE_MISSED` (VARCHAR enum column) or the YAML/service changes.

---

## Files changed

### casehub-engine (new GitHub issue: engine#N — to be filed before implementation)

| File | Change |
|------|--------|
| `api/src/main/java/io/casehub/api/engine/ExpressionEngine.java` | Add `default Optional<String> extractString(ExpressionEvaluator, CaseContext)` |
| `api/src/main/java/io/casehub/api/engine/ExpressionEngineRegistry.java` | Add `Optional<String> extractString(ExpressionEvaluator, CaseContext)` |
| `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java` | Add `expiresAtExpression: ExpressionEvaluator` field + builder |
| `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` | Map `expiresAtExpression` with load-time JQ validation |
| `schema/src/main/resources/schema/CaseDefinition.yaml` | Add `expiresAtExpression: string` to `HumanTask` |
| `schema/target/.../io/casehub/model/HumanTask.java` | Add `String expiresAtExpression` field (auto-regenerated) |
| `runtime/src/main/java/io/casehub/engine/internal/engine/JQExpressionEngine.java` | Override `extractString()` |
| `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultExpressionEngineRegistry.java` | Implement `extractString()` dispatch |
| `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` | `resolveExpiresAtDeadline()`, pass `expiresAtDeadline` in event |
| `common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java` | Add `expiresAtDeadline: Instant` field |
| `work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java` | `createInline()` gains `expiresAtDeadline` param; template mode caps with `earliestOf` |
| `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java` | Tests for inline + template mode `expiresAtDeadline` behaviour |

### casehub-clinical (issue casehubio/clinical#83)

| File | Change |
|------|--------|
| `runtime/src/main/resources/clinical/regulatory-submission.yaml` | `capability` → `humanTask` with `expiresAtExpression` |
| `api/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java` | Add `DEADLINE_MISSED` only (FILED exists) |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalIndReportingBreachPolicy.java` | New — pure SlaBreachPolicy, stateless two-tier, 48h window |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCompletedListener.java` | New — GoalReached/CaseCompleted observer, DB discriminates |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionBreachListener.java` | New — WorkItemLifecycleEvent(ESCALATED) observer |
| `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java` | Add `writeFiledEntry()`, `writeBreachEntry()` |
| `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java` | Add `regulatorySubmissionBreach(CtcaeGrade)` factory method |
| `runtime/src/main/java/io/casehub/clinical/ledger/IndReportFiledLedgerEntry.java` | New JPA subclass, `@DiscriminatorValue("IndReportFiled")` |
| `runtime/src/main/java/io/casehub/clinical/ledger/IndReportBreachLedgerEntry.java` | New JPA subclass, `@DiscriminatorValue("IndReportBreach")` |
| `runtime/src/main/resources/db/migration/qhorus/V2026__ind_report_filed_ledger_entry.sql` | New join table (V2024/V2025 taken by Layer 9) |
| `runtime/src/main/resources/db/migration/qhorus/V2027__ind_report_breach_ledger_entry.sql` | New join table |
| `runtime/src/test/java/io/casehub/clinical/service/ClinicalIndReportingBreachPolicyTest.java` | New unit tests |
| `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCompletedListenerTest.java` | New integration tests |
| `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionBreachListenerTest.java` | New integration tests |
| `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionDeadlineLifecycleTest.java` | End-to-end invariant: `WorkItem.expiresAt == ae.reportedAt + window` |

---

## Testing strategy

### Engine

**`HumanTaskScheduleHandlerTest`:**
- `expiresAtDeadline` in event → WorkItem `expiresAt` set to that value
- `expiresAtDeadline` earlier than `expiresIn`-derived deadline → `expiresAtDeadline` wins
- `caseBudgetDeadline` earlier than `expiresAtDeadline` → budget wins
- All three set → earliest wins
- `expiresAtDeadline = null` → falls back to existing behaviour
- Template mode: `expiresAtDeadline` caps template's `expiresAt` when later; preserved when template's is earlier

**`JQExpressionEngineTest`:**
- `extractString()` returns ISO-8601 string from WORKING panel context field
- JQ evaluation failure → `Optional.empty()` + WARN (no IndexOutOfBoundsException)
- Field absent in context → JQ evaluates to `null` (one NullNode output, `vr.output().isEmpty()` is false) → `isTextual()` false → `Optional.empty()`, no WARN
- Expression produces no output at all (e.g. `empty`) → `vr.output().isEmpty()` true → `Optional.empty()`, no WARN
- Non-text JQ output (number, boolean) → `Optional.empty()`
- Null/blank evaluator → `Optional.empty()`

### Clinical

**`ClinicalIndReportingBreachPolicyTest` (unit):**
- Regulatory-affairs WorkItem (first breach): `EscalateTo("regulatory-leadership", PT48H)`
- Regulatory-leadership WorkItem (second breach — candidateGroups replaced after first escalation): `Exhausted`
- Non-regulatory WorkItem: `Fail("no-sla-breach-policy-configured")`

**`RegulatorySubmissionCompletedListenerTest` (integration, `@QuarkusTest`):**
- GoalReached for regulatory-submission case → `ae.regulatorySubmissionStatus = FILED`, ledger entry written
- CaseCompleted → same result
- GoalReached for unrelated caseId → AE status unchanged
- Double GoalReached → idempotency guard, second call is no-op

**`RegulatorySubmissionBreachListenerTest` (integration, `@QuarkusTest`):**
- ESCALATED event with regulatory-submission `callerRef` → `ae.regulatorySubmissionStatus = DEADLINE_MISSED`, breach ledger entry written
- ESCALATED event with non-regulatory `callerRef` → no AE state change
- Duplicate ESCALATED event → idempotency guard (`!= PENDING`), second call is no-op
- ESCALATED event when AE is already FILED → idempotency guard protects against overwrite
- Non-ESCALATED status events → ignored
- Wire-reconstructed event (`event.source() == null`) → no AE state change

**`RegulatorySubmissionDeadlineLifecycleTest` (integration, `@QuarkusTest`, engine on classpath, no `@InjectMock` on CaseHub):**

This is the critical invariant test — verifies the full data path: `ae.reportedAt + indReportingWindow(grade) → case context → JQ expression (.indReportingDeadline) → HumanTaskScheduleEvent.expiresAtDeadline → WorkItem.expiresAt`. Without it, the expression could be wired incorrectly (wrong field name, wrong format) and no other test would catch it.

Setup requirements:
- Stamp `ae.tenantId = principal.tenancyId()` on persist (CLAUDE.md required pattern — omission causes `SecurityException` from `MemoryPermissions.assertTenant()` during engine case processing, producing a confusing failure rather than a clear assertion failure)
- All other entity fields per existing lifecycle test patterns (`DsmbRollupTest`, `AeEscalationLifecycleTest`)

Test steps:
- Persist Grade 3 unexpected AE with `reportedAt = Instant.parse("2026-07-01T10:00:00Z")`
- Fire `AdverseEventReportedEvent` via `RegulatorySubmissionCaseService.onAdverseEventReported()`
- `HumanTaskScheduleHandler` fires on a Vert.x worker thread (`@ConsumeEvent(blocking=true)`) after `onAdverseEventReported()` returns — the WorkItem does not exist yet at return. Use Awaitility: `await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(...)` (reference: `RegulatorySubmissionCaseServiceTest`, `AeEscalationLifecycleTest`)
- Find WorkItem by `candidateGroups` containing `"regulatory-affairs"` and `callerRef` starting with `"case:"`
- Assert `workItem.expiresAt == Instant.parse("2026-07-16T10:00:00Z")` (reportedAt + 15 days, Grade 3)
- Repeat for Grade 4: `reportedAt + 7 days`

**Existing `RegulatorySubmissionCaseServiceTest`:** All existing tests pass unchanged — service API is unmodified.
