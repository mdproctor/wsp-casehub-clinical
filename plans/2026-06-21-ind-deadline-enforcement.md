# IND Deadline Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce the FDA IND expedited safety report deadline as an exact absolute `WorkItem.expiresAt` derived from `ae.reportedAt + window`, via a new `expiresAtExpression` JQ field on `HumanTaskTarget` evaluated at scheduling time.

**Architecture:** Two-repo change. Engine (casehub/engine) adds pluggable `ExpressionEngine.extractString()` SPI + `HumanTaskTarget.expiresAtExpression` field + scheduler wiring. Clinical (casehub/clinical) changes the regulatory-submission YAML from capability to humanTask, adds `ClinicalIndReportingBreachPolicy`, `RegulatorySubmissionCompletedListener`, `RegulatorySubmissionBreachListener`, two new ledger subclasses (V2026/V2027 migrations), and `RegulatorySubmissionStatus.DEADLINE_MISSED`. Engine changes are prerequisite — implement Phase 1 first.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, Mockito, Panache Active Record, Jackson JQ (net.thisptr.jackson.jq), CDI events (`@ObservesAsync`), casehub-work `SlaBreachPolicy` SPI, JPA JOINED inheritance, Flyway.

**Spec:** `specs/2026-06-21-ind-deadline-enforcement-design.md`

---

## Phase 1: Engine (casehub-engine)

Engine repo: `/Users/mdproctor/claude/casehub/engine`  
Engine workspace: `/Users/mdproctor/claude/public/casehub/engine`

---

### Task E1: File engine GitHub issue + create engine branch

**Files:** Engine GitHub repo only

- [ ] **Step 1: File engine issue**

```bash
gh issue create --repo casehubio/engine \
  --title "feat: expiresAtExpression — absolute deadline for humanTask WorkItems via ExpressionEngine.extractString()" \
  --body "$(cat <<'EOF'
## Context

HumanTaskTarget currently supports expiresIn (Duration from creation) and claimDeadlineHours (integer). Neither supports an absolute deadline derived from case context — only a relative duration from WorkItem creation time.

## What's needed

1. Add `ExpressionEngine.extractString(ExpressionEvaluator, CaseContext): Optional<String>` — default throws UnsupportedOperationException; engines override to enable
2. Add matching dispatch to `ExpressionEngineRegistry` + `DefaultExpressionEngineRegistry`
3. Override `extractString()` in `JQExpressionEngine` — evaluates against WORKING panel, isTextual() guard, emptiness guard
4. Add `expiresAtExpression: ExpressionEvaluator` to `HumanTaskTarget` (String + ExpressionEvaluator builder overloads)
5. Add `expiresAtExpression: string` to CaseDefinition.yaml schema + regenerate HumanTask.java
6. Map in `CaseDefinitionYamlMapper.convertHumanTask()` with load-time JQ syntax validation
7. Add `expiresAtDeadline: Instant` to `HumanTaskScheduleEvent` record
8. Evaluate expression in `CaseContextChangedEventHandler.publishHumanTaskSchedule()` → pass as `expiresAtDeadline`
9. Fold `expiresAtDeadline` into `earliestOf` chain in `HumanTaskScheduleHandler.createInline()`; apply to template mode also

## Consumer

casehub-clinical#83 — IND reporting deadline enforcement

Refs casehub-clinical#83
EOF
)"
```

Note the issue number from output (e.g., engine#N).

- [ ] **Step 2: Create engine branch**

```bash
git -C /Users/mdproctor/claude/casehub/engine checkout -b issue-N-expires-at-expression
git -C /Users/mdproctor/claude/public/casehub/engine checkout -b issue-N-expires-at-expression 2>/dev/null || true
```

Replace `N` with the actual engine issue number.

---

### Task E2: ExpressionEngine.extractString() default method (api module)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/ExpressionEngine.java`

No direct test at this level — the SPI default is tested via `JQExpressionEngine` (Task E4) and the registry (Task E3).

- [ ] **Step 1: Add the default method**

In `ExpressionEngine.java`, add after `validate()`:

```java
/**
 * Extracts a string value from the given context using this evaluator.
 *
 * <p>Default implementation throws {@link UnsupportedOperationException}. Expression engines
 * that support value extraction (not just boolean evaluation) must override this method.
 *
 * <p>The {@link DefaultExpressionEngineRegistry} catches {@code UnsupportedOperationException}
 * from this method and returns {@code Optional.empty()} + WARN — so callers never see the
 * exception propagate unless they invoke this method directly on an engine that doesn't support it.
 *
 * @param evaluator the expression to evaluate — guaranteed to match {@link #type()}
 * @param context the current case state; implementations evaluate against the WORKING panel
 * @return the string value extracted from context, or empty if absent or evaluation fails
 */
default Optional<String> extractString(ExpressionEvaluator evaluator, CaseContext context) {
    throw new UnsupportedOperationException(
            "ExpressionEngine '" + type() + "' does not support string extraction. "
            + "Override extractString() to enable this capability.");
}
```

Add `import java.util.Optional;` to the imports.

- [ ] **Step 2: Build api module to verify compilation**

```bash
mvn install -pl api --batch-mode -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task E3: ExpressionEngineRegistry.extractString() + DefaultExpressionEngineRegistry dispatch

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/ExpressionEngineRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultExpressionEngineRegistry.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/ExpressionEngineRegistryTest.java`

- [ ] **Step 1: Write failing tests for extractString()**

In `ExpressionEngineRegistryTest.java`, add a new `@Nested` class at the bottom:

```java
@Nested
@DisplayName("extractString()")
class ExtractString {

    @Test
    @DisplayName("JQ — extracts string field from WORKING panel context")
    void jq_extractsStringField() {
        final var context = new CaseContextImpl(Map.of("indReportingDeadline", "2026-07-16T10:00:00Z"));
        final var evaluator = new JQExpressionEvaluator(".indReportingDeadline");
        final Optional<String> result = registry.extractString(evaluator, context);
        assertThat(result).contains("2026-07-16T10:00:00Z");
    }

    @Test
    @DisplayName("JQ — missing field returns empty (JQ null output, isTextual guard)")
    void jq_missingField_returnsEmpty() {
        final var context = new CaseContextImpl(Map.of("otherField", "value"));
        final var evaluator = new JQExpressionEvaluator(".indReportingDeadline");
        final Optional<String> result = registry.extractString(evaluator, context);
        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("JQ — expression produces no output returns empty (isEmpty guard)")
    void jq_emptyOutput_returnsEmpty() {
        final var context = new CaseContextImpl(Map.of("foo", "bar"));
        final var evaluator = new JQExpressionEvaluator("empty");
        final Optional<String> result = registry.extractString(evaluator, context);
        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("JQ — non-text output (number) returns empty")
    void jq_numericOutput_returnsEmpty() {
        final var context = new CaseContextImpl(Map.of("count", 42));
        final var evaluator = new JQExpressionEvaluator(".count");
        final Optional<String> result = registry.extractString(evaluator, context);
        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("unsupported engine type returns empty with no exception")
    void unsupportedEngine_returnsEmpty() {
        // LambdaExpressionEvaluator type is registered but does not override extractString()
        final var context = new CaseContextImpl(Map.of());
        final var evaluator = new LambdaExpressionEvaluator(() -> true);
        // Must not throw — registry catches UnsupportedOperationException
        final Optional<String> result = assertDoesNotThrow(
                () -> registry.extractString(evaluator, context));
        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("null evaluator returns empty")
    void nullEvaluator_returnsEmpty() {
        final var context = new CaseContextImpl(Map.of());
        assertThat(registry.extractString(null, context)).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=ExpressionEngineRegistryTest --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: FAIL — `extractString` not yet on registry interface.

- [ ] **Step 3: Add extractString() to ExpressionEngineRegistry interface**

In `ExpressionEngineRegistry.java`, add after `assertLanguageSupported()`:

```java
/**
 * Extracts a string value from the given context using the expression in {@code evaluator}.
 *
 * <p>Dispatches to the registered {@link ExpressionEngine} whose {@link ExpressionEngine#type()}
 * matches {@link ExpressionEvaluator#type()}. If the matched engine does not override
 * {@link ExpressionEngine#extractString(ExpressionEvaluator, CaseContext)}, this method
 * returns {@link Optional#empty()} and logs a WARN — it does NOT propagate
 * {@link UnsupportedOperationException}.
 *
 * @param evaluator the expression; returns empty if null
 * @param context the current case state
 * @return the extracted string, or empty if unavailable or unsupported
 * @throws IllegalArgumentException if no engine is registered for the evaluator type
 */
Optional<String> extractString(ExpressionEvaluator evaluator, CaseContext context);
```

Add `import java.util.Optional;` to imports.

- [ ] **Step 4: Implement extractString() in DefaultExpressionEngineRegistry**

In `DefaultExpressionEngineRegistry.java`, add after `assertLanguageSupported()`:

```java
@Override
public Optional<String> extractString(final ExpressionEvaluator evaluator, final CaseContext context) {
    if (evaluator == null) {
        return Optional.empty();
    }
    final String type = evaluator.type();
    for (ExpressionEngine engine : engines) {
        if (engine.type().equals(type)) {
            try {
                return engine.extractString(evaluator, context);
            } catch (UnsupportedOperationException e) {
                LOG.warnf("ExpressionEngine '%s' does not support string extraction — returning empty", type);
                return Optional.empty();
            }
        }
    }
    throw new IllegalArgumentException("No ExpressionEngine registered for type '" + type + "'");
}
```

Add `import java.util.Optional;` to imports.

- [ ] **Step 5: Run tests to verify they fail (JQ extractString not yet implemented)**

```bash
mvn test -pl runtime -Dtest=ExpressionEngineRegistryTest --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: The extractString() tests for JQ will fail (UnsupportedOperationException caught, returns empty — so the `jq_extractsStringField` test fails). The `unsupportedEngine_returnsEmpty` test passes.

---

### Task E4: JQExpressionEngine.extractString() (runtime module)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/JQExpressionEngine.java`

- [ ] **Step 1: Override extractString() in JQExpressionEngine**

Add after `validate()` in `JQExpressionEngine.java`:

```java
@Override
public Optional<String> extractString(final ExpressionEvaluator evaluator, final CaseContext context) {
    final String expr = ((JQExpressionEvaluator) evaluator).expression();
    if (expr == null || expr.isBlank()) {
        return Optional.empty();
    }
    final ValidationResult vr =
            jqEvaluator.eval(expr, context.panel(ContextPanel.WORKING).asJsonNode());
    if (!vr.ok() || vr.output() == null || vr.output().isEmpty()) {
        if (!vr.ok()) {
            LOG.warnf("extractString JQ evaluation failed: %s", vr.error());
        }
        return Optional.empty();
    }
    final JsonNode result = vr.output().get(0);
    if (!result.isTextual()) {
        return Optional.empty();
    }
    return Optional.of(result.asText());
}
```

Add needed imports: `com.fasterxml.jackson.databind.JsonNode`, `io.casehub.api.context.ContextPanel`, `java.util.Optional`.

- [ ] **Step 2: Run all extractString registry tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=ExpressionEngineRegistryTest --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: all `extractString()` tests pass.

- [ ] **Step 3: Run full runtime test suite**

```bash
mvn test -pl runtime --batch-mode -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS` — no regressions.

- [ ] **Step 4: Commit SPI extension**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  api/src/main/java/io/casehub/api/engine/ExpressionEngine.java \
  api/src/main/java/io/casehub/api/engine/ExpressionEngineRegistry.java \
  runtime/src/main/java/io/casehub/engine/internal/engine/DefaultExpressionEngineRegistry.java \
  runtime/src/main/java/io/casehub/engine/internal/engine/JQExpressionEngine.java \
  runtime/src/test/java/io/casehub/engine/internal/engine/ExpressionEngineRegistryTest.java

git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
feat(engine#N): ExpressionEngine.extractString() — pluggable string extraction SPI

Adds ExpressionEngine.extractString(ExpressionEvaluator, CaseContext): Optional<String>
as a default method that throws UnsupportedOperationException. ExpressionEngineRegistry
gains the matching dispatch method; DefaultExpressionEngineRegistry catches
UnsupportedOperationException and returns Optional.empty() + WARN. JQExpressionEngine
overrides extractString(): evaluates against WORKING panel, emptiness guard, isTextual()
guard, null-blank guard. No instanceof anywhere.

Enables absolute-deadline WorkItem enforcement via HumanTaskTarget.expiresAtExpression.

Refs casehubio/engine#N
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task E5: HumanTaskTarget.expiresAtExpression field (api module)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java`

- [ ] **Step 1: Add expiresAtExpression field + accessor + builder methods**

In `HumanTaskTarget.java`:

Add to private fields (after `claimDeadlineHours`):
```java
private final ExpressionEvaluator expiresAtExpression;
```

Add to constructor `HumanTaskTarget(Builder builder)`:
```java
this.expiresAtExpression = builder.expiresAtExpression;
```

Add accessor:
```java
/** JQ expression evaluated against case context WORKING panel to produce an absolute deadline Instant. */
public ExpressionEvaluator expiresAtExpression() {
    return expiresAtExpression;
}
```

Add to `Builder` class (after `claimDeadlineHours` field):
```java
private ExpressionEvaluator expiresAtExpression;
```

Add to `Builder`:
```java
/** Absolute deadline from case context — string overload creates JQExpressionEvaluator. */
public Builder expiresAtExpression(String expression) {
    this.expiresAtExpression = new JQExpressionEvaluator(expression);
    return this;
}

public Builder expiresAtExpression(ExpressionEvaluator evaluator) {
    this.expiresAtExpression = evaluator;
    return this;
}
```

- [ ] **Step 2: Build api module**

```bash
mvn install -pl api --batch-mode -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task E6: CaseDefinition.yaml schema + HumanTask.java pojo (schema module)

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml`
- Modify: `schema/target/generated-sources/jsonschema2pojo/io/casehub/model/HumanTask.java`

The `HumanTask.java` in `schema/target/` is auto-generated by jsonschema2pojo during `mvn generate-sources`. We add the field manually now; the next full build will regenerate it correctly.

- [ ] **Step 1: Add expiresAtExpression to CaseDefinition.yaml**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, find the `HumanTask` definition (around the `expiresIn` property). Add after `claimDeadlineHours`:

```yaml
      expiresAtExpression:
        type: string
        description: >
          JQ expression evaluated against the case context WORKING panel at scheduling time.
          Must produce an ISO-8601 Instant string (e.g. ".indReportingDeadline").
          Validated at YAML load time. Used to enforce an absolute deadline derived from
          domain data (e.g. ae.reportedAt + window) rather than a relative duration from
          WorkItem creation.
```

- [ ] **Step 2: Add field to HumanTask.java (generated pojo)**

In `schema/target/generated-sources/jsonschema2pojo/io/casehub/model/HumanTask.java`, add the field alongside the existing `expiresIn` field:

In `@JsonPropertyOrder`:
```java
"expiresIn",
"claimDeadlineHours",
"expiresAtExpression",  // add this
"outcomes"
```

Add field:
```java
@JsonProperty("expiresAtExpression")
@JsonPropertyDescription("JQ expression evaluated against the case context WORKING panel at scheduling time. Must produce an ISO-8601 Instant string.")
private String expiresAtExpression;
```

Add getter and setter following the `expiresIn` pattern:
```java
@JsonProperty("expiresAtExpression")
public String getExpiresAtExpression() {
    return expiresAtExpression;
}

@JsonProperty("expiresAtExpression")
public void setExpiresAtExpression(String expiresAtExpression) {
    this.expiresAtExpression = expiresAtExpression;
}
```

- [ ] **Step 3: Build schema module to verify**

```bash
mvn install -pl schema --batch-mode -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task E7: CaseDefinitionYamlMapper mapping with load-time JQ validation (api module)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`

- [ ] **Step 1: Add expiresAtExpression mapping in convertHumanTask()**

In `convertHumanTask()`, after the `expiresIn` block (around line 544):

```java
if (schema.getExpiresAtExpression() != null && !schema.getExpiresAtExpression().isBlank()) {
    // Validate JQ syntax at load time — a silent runtime null is a regulatory SLA failure
    try {
        net.thisptr.jackson.jq.JsonQuery.compile(
                schema.getExpiresAtExpression(),
                net.thisptr.jackson.jq.Versions.JQ_1_6);
    } catch (Exception e) {
        throw new IllegalArgumentException(
                "invalid expiresAtExpression '" + schema.getExpiresAtExpression()
                + "' — " + e.getMessage(), e);
    }
    builder.expiresAtExpression(schema.getExpiresAtExpression());
}
```

- [ ] **Step 2: Build and test api module**

```bash
mvn install -pl api --batch-mode -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task E8: HumanTaskScheduleEvent — add expiresAtDeadline field (common module)

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java`

This record change **breaks all callers** — every `new HumanTaskScheduleEvent(...)` call needs a new `null` argument. This is intentional per design principles.

- [ ] **Step 1: Add expiresAtDeadline to the record**

Replace the record definition with:

```java
public record HumanTaskScheduleEvent(
    UUID caseId,
    String bindingName,
    HumanTaskTarget target,
    Map<String, Object> inputData,
    Set<String> resolvedCandidateGroups,
    Set<String> resolvedCandidateUsers,
    Instant caseBudgetDeadline,
    Instant expiresAtDeadline,
    String tenancyId) {}
```

- [ ] **Step 2: Fix all callers of HumanTaskScheduleEvent constructor**

Search in engine repo for `new HumanTaskScheduleEvent(` and fix each:

In `CaseContextChangedEventHandler.java` (in the `publishHumanTaskSchedule()` method), the event constructor call currently has 8 args — add `null` as the 8th arg for now (before `caseInstance.tenancyId`):

```java
new HumanTaskScheduleEvent(
    caseInstance.getUuid(),
    binding.getName(),
    target,
    inputData,
    resolvedGroups,
    resolvedUsers,
    caseBudgetDeadline,
    null,              // expiresAtDeadline — wired in Task E9
    caseInstance.tenancyId)
```

In `HumanTaskScheduleHandlerTest.java`, update ALL `new HumanTaskScheduleEvent(...)` calls to add `null` as 8th arg. Example (the smoke test):

```java
new HumanTaskScheduleEvent(
    caseId, "irb-binding", target, Map.of(), null, null, null, null, TENANCY_ID)
```

- [ ] **Step 3: Build common + work-adapter to verify all callers fixed**

```bash
mvn install -pl api,common,schema --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml && \
mvn compile -pl work-adapter --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task E9: CaseContextChangedEventHandler.resolveExpiresAtDeadline() (runtime module)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`

- [ ] **Step 1: Add resolveExpiresAtDeadline() private method**

Add after `evaluateInputMapping()`:

```java
private Instant resolveExpiresAtDeadline(final CaseInstance caseInstance, final HumanTaskTarget target) {
    if (target.expiresAtExpression() == null) {
        return null;
    }
    return registry.extractString(target.expiresAtExpression(), caseInstance.getCaseContext())
            .map(s -> {
                try {
                    return Instant.parse(s);
                } catch (Exception e) {
                    LOG.warnf(
                            "expiresAtExpression result '%s' is not a valid ISO-8601 instant — ignoring",
                            s);
                    return null;
                }
            })
            .orElse(null);
}
```

- [ ] **Step 2: Wire resolveExpiresAtDeadline() into publishHumanTaskSchedule()**

In `publishHumanTaskSchedule()`, after the `caseBudgetDeadline` resolution, add:

```java
final Instant expiresAtDeadline = resolveExpiresAtDeadline(caseInstance, target);
```

Then pass `expiresAtDeadline` in the event constructor (replacing the `null` added in Task E8):

```java
new HumanTaskScheduleEvent(
    caseInstance.getUuid(),
    binding.getName(),
    target,
    inputData,
    resolvedGroups,
    resolvedUsers,
    caseBudgetDeadline,
    expiresAtDeadline,
    caseInstance.tenancyId)
```

- [ ] **Step 3: Build runtime module**

```bash
mvn install -pl api,common,schema --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml && \
mvn compile -pl runtime --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task E10: HumanTaskScheduleHandler — createInline() + handleTemplateMode() (work-adapter module)

**Files:**
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java`

`createInline()` currently takes 6 parameters. We add `Instant expiresAtDeadline` as a new 6th parameter (before `caseBudgetDeadline`).

- [ ] **Step 1: Update createInline() signature and implementation**

Change signature from:
```java
private void createInline(
    HumanTaskTarget target,
    Map<String, Object> inputData,
    Set<String> resolvedGroups,
    Set<String> resolvedUsers,
    String callerRef,
    Instant caseBudgetDeadline)
```

To:
```java
private void createInline(
    HumanTaskTarget target,
    Map<String, Object> inputData,
    Set<String> resolvedGroups,
    Set<String> resolvedUsers,
    String callerRef,
    Instant expiresAtDeadline,
    Instant caseBudgetDeadline)
```

Update the `effectiveDeadline` computation inside `createInline()`:

```java
Instant taskDeadline = target.expiresIn() != null ? Instant.now().plus(target.expiresIn()) : null;
Instant effectiveDeadline = earliestOf(earliestOf(taskDeadline, expiresAtDeadline), caseBudgetDeadline);
```

The existing 2-arg `earliestOf` handles null correctly. The chain `earliestOf(earliestOf(a, b), c)` picks the earliest non-null of all three.

- [ ] **Step 2: Update handleInlineMode() to pass event.expiresAtDeadline()**

In `handleInlineMode()`, update the `createInline()` call:

```java
private void handleInlineMode(PlanItem item, HumanTaskScheduleEvent event) {
    String callerRef = PlanItemCallerRef.encode(event.caseId(), item.getPlanItemId());
    createInline(
        event.target(),
        event.inputData(),
        event.resolvedCandidateGroups(),
        event.resolvedCandidateUsers(),
        callerRef,
        event.expiresAtDeadline(),    // new
        event.caseBudgetDeadline());
    ...
}
```

- [ ] **Step 3: Apply expiresAtDeadline in handleTemplateMode()**

In `handleTemplateMode()`, after `workItem.persist()` is called (no, before — after `instantiate()` and before `persist()`), add:

```java
if (event.expiresAtDeadline() != null) {
    workItem.expiresAt = workItem.expiresAt == null
            ? event.expiresAtDeadline()
            : earliestOf(workItem.expiresAt, event.expiresAtDeadline());
}
```

- [ ] **Step 4: Build work-adapter**

```bash
mvn install -pl api,common,schema --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml && \
mvn compile -pl work-adapter --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task E11: HumanTaskScheduleHandlerTest — new expiresAtDeadline test cases (work-adapter)

**Files:**
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java`

- [ ] **Step 1: Write failing tests for expiresAtDeadline**

Add the following test methods to `HumanTaskScheduleHandlerTest.java` (in the `// ── Inline mode` section):

```java
@Test
void inlineMode_expiresAtDeadline_setsWorkItemExpiresAt() {
    Instant deadline = Instant.now().plusSeconds(3600);
    HumanTaskTarget target = HumanTaskTarget.inline().title("IND Filing").build();

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId, "irb-binding", target, Map.of(), null, null, null, deadline, TENANCY_ID));

    WorkItem created = workItemStore.scanAll().stream()
        .filter(w -> PlanItemCallerRef.encode(caseId, planItem.getPlanItemId()).equals(w.callerRef))
        .findFirst().orElseThrow();
    assertThat(created.expiresAt).isEqualTo(deadline);
}

@Test
void inlineMode_expiresAtDeadline_earlierThanExpiresIn_winsOverExpiresIn() {
    Instant absoluteDeadline = Instant.now().plusSeconds(3600);   // 1h from now
    HumanTaskTarget target = HumanTaskTarget.inline()
        .title("IND Filing")
        .expiresIn(Duration.ofDays(7))                           // 7d from now — later
        .build();

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId, "irb-binding", target, Map.of(), null, null, null, absoluteDeadline, TENANCY_ID));

    WorkItem created = workItemStore.scanAll().stream()
        .filter(w -> PlanItemCallerRef.encode(caseId, planItem.getPlanItemId()).equals(w.callerRef))
        .findFirst().orElseThrow();
    // absolute deadline is earlier — it wins
    assertThat(created.expiresAt).isEqualTo(absoluteDeadline);
}

@Test
void inlineMode_caseBudgetDeadline_earlierThanExpiresAtDeadline_winsOverExpiresAtDeadline() {
    Instant absoluteDeadline = Instant.now().plusSeconds(7200);   // 2h — later
    Instant budgetDeadline = Instant.now().plusSeconds(3600);     // 1h — earlier
    HumanTaskTarget target = HumanTaskTarget.inline().title("IND Filing").build();

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId, "irb-binding", target, Map.of(), null, null, budgetDeadline, absoluteDeadline, TENANCY_ID));

    WorkItem created = workItemStore.scanAll().stream()
        .filter(w -> PlanItemCallerRef.encode(caseId, planItem.getPlanItemId()).equals(w.callerRef))
        .findFirst().orElseThrow();
    assertThat(created.expiresAt).isEqualTo(budgetDeadline);
}

@Test
void inlineMode_nullExpiresAtDeadline_fallsBackToExpiresIn() {
    HumanTaskTarget target = HumanTaskTarget.inline()
        .title("IND Filing")
        .expiresIn(Duration.ofHours(24))
        .build();

    Instant before = Instant.now().plusSeconds(23 * 3600);
    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId, "irb-binding", target, Map.of(), null, null, null, null, TENANCY_ID));
    Instant after = Instant.now().plusSeconds(25 * 3600);

    WorkItem created = workItemStore.scanAll().stream()
        .filter(w -> PlanItemCallerRef.encode(caseId, planItem.getPlanItemId()).equals(w.callerRef))
        .findFirst().orElseThrow();
    assertThat(created.expiresAt).isBetween(before, after);
}
```

Also update all existing test instantiations of `HumanTaskScheduleEvent` to include the new `null` for `expiresAtDeadline` — this was done in Task E8 but verify now.

- [ ] **Step 2: Run tests to verify new ones fail**

```bash
mvn test -pl work-adapter -Dtest=HumanTaskScheduleHandlerTest --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: the four new tests FAIL (expiresAtDeadline not yet wired).

- [ ] **Step 3: Verify tests pass after Task E10 implementation**

The implementation is already done in Task E10. Run tests again:

```bash
mvn test -pl work-adapter -Dtest=HumanTaskScheduleHandlerTest --batch-mode \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: all tests PASS including the four new ones.

- [ ] **Step 4: Run full engine test suite**

```bash
mvn test --batch-mode -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit engine HumanTask deadline wiring**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  api/src/main/java/io/casehub/api/model/HumanTaskTarget.java \
  api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java \
  schema/src/main/resources/schema/CaseDefinition.yaml \
  schema/target/generated-sources/jsonschema2pojo/io/casehub/model/HumanTask.java \
  common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java \
  runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java

git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
feat(engine#N): HumanTaskTarget.expiresAtExpression — absolute deadline in humanTask YAML binding

- HumanTaskTarget: expiresAtExpression: ExpressionEvaluator field + String/Evaluator builder overloads
- CaseDefinition.yaml: expiresAtExpression: string field in HumanTask schema definition
- HumanTask.java (pojo): String expiresAtExpression field (regenerated on next build)
- CaseDefinitionYamlMapper: map expiresAtExpression with JQ syntax validation at load time
- HumanTaskScheduleEvent: add expiresAtDeadline: Instant (BREAKING — all 8-arg callers → 9-arg)
- CaseContextChangedEventHandler: resolveExpiresAtDeadline() evaluates expression against
  case context WORKING panel, passes resolved Instant in HumanTaskScheduleEvent
- HumanTaskScheduleHandler: createInline() folds expiresAtDeadline into earliestOf chain;
  handleTemplateMode() caps template expiresAt with expiresAtDeadline
- Tests: 4 new cases for earliest-wins semantics (expiresAtDeadline vs expiresIn vs budget)

Refs casehubio/engine#N, casehubio/clinical#83
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 6: Update engine workspace HANDOFF.md**

```bash
# Read current engine HANDOFF (if exists) then update
cat > /Users/mdproctor/claude/public/casehub/engine/HANDOFF.md << 'EOF'
# Engine Session Handover — 2026-06-21

## Work Done

Branch `issue-N-expires-at-expression` (engine#N):
- Added `ExpressionEngine.extractString()` default method + `ExpressionEngineRegistry` dispatch
- `JQExpressionEngine` overrides `extractString()` (WORKING panel, emptiness + isTextual guards)
- `DefaultExpressionEngineRegistry` implements dispatch with UnsupportedOperationException catch
- `HumanTaskTarget.expiresAtExpression: ExpressionEvaluator` field + String/Evaluator builder
- `CaseDefinition.yaml` schema updated + `HumanTask.java` pojo field
- `CaseDefinitionYamlMapper` maps expression with JQ load-time validation
- `HumanTaskScheduleEvent` record: new `expiresAtDeadline: Instant` field (BREAKING — 8→9 args)
- `CaseContextChangedEventHandler.resolveExpiresAtDeadline()` wired to publishHumanTaskSchedule()
- `HumanTaskScheduleHandler.createInline()` folds expiresAtDeadline; template mode also applied
- All engine tests passing

## Status

Branch committed, not pushed. Consumer: casehubio/clinical#83.
EOF

git -C /Users/mdproctor/claude/public/casehub/engine add HANDOFF.md
git -C /Users/mdproctor/claude/public/casehub/engine commit -m "docs: engine HANDOFF for issue-N-expires-at-expression" 2>/dev/null || git -C /Users/mdproctor/claude/public/casehub/engine add HANDOFF.md
```

---

## Phase 2: Clinical (casehub-clinical)

Clinical project: `/Users/mdproctor/claude/casehub/clinical`  
Branch: `issue-83-ind-deadline-workitem` (already active)  
Build commands: run from `/Users/mdproctor/claude/casehub/clinical`

```bash
# Install api (required before runtime tests)
mvn install -pl api --batch-mode

# Run single test class
mvn test -pl runtime -Dtest=ClassName --batch-mode

# Full build
mvn test --batch-mode
```

---

### Task C1: YAML change + RegulatorySubmissionStatus.DEADLINE_MISSED

**Files:**
- Modify: `runtime/src/main/resources/clinical/regulatory-submission.yaml`
- Modify: `api/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java`

These are simple changes with no tests needed (the deadline test in Task C8 covers the YAML end-to-end).

- [ ] **Step 1: Update regulatory-submission.yaml**

Replace the entire file content:

```yaml
dsl: "0.1"
version: "1.0.0"
name: regulatory-submission
namespace: clinical
title: IND Expedited Safety Report Filing

spec:

  goals:
    - name: submission-complete
      kind: success
      condition: ".submissionFiled != null"

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

- [ ] **Step 2: Add DEADLINE_MISSED to RegulatorySubmissionStatus**

```java
public enum RegulatorySubmissionStatus {
    /** Default — AE does not meet IND reportable criteria; no submission triggered. */
    NONE,
    /** Grade 3/4/5 + unexpected confirmed; regulatory submission case started. */
    PENDING,
    /** IND expedited safety report filed. */
    FILED,
    /** IND reporting deadline exhausted without submission. */
    DEADLINE_MISSED
}
```

- [ ] **Step 3: Compile to verify**

```bash
mvn compile -pl api,runtime --batch-mode
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/resources/clinical/regulatory-submission.yaml \
  api/src/main/java/io/casehub/clinical/api/model/RegulatorySubmissionStatus.java

git -C /Users/mdproctor/claude/casehub/clinical commit -m "$(cat <<'EOF'
feat(#83): regulatory-submission.yaml: capability → humanTask with expiresAtExpression

- YAML: remove capabilities section; file-ind-report binding uses humanTask with
  expiresAtExpression: ".indReportingDeadline" (absolute FDA deadline from case context),
  candidateGroups: [regulatory-affairs], inputMapping + outputMapping
- RegulatorySubmissionStatus: add DEADLINE_MISSED (FILED already existed)

Refs casehubio/clinical#83
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task C2: IndReportFiledLedgerEntry + V2026 migration

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/IndReportFiledLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2026__ind_report_filed_ledger_entry.sql`

- [ ] **Step 1: Write the Flyway migration**

`V2026__ind_report_filed_ledger_entry.sql`:

```sql
-- Ledger entry for IND expedited safety report filed (WorkItem completed)
-- Joins to casehub-ledger base table on the qhorus datasource.
-- V2024 and V2025 are taken by Layer 9 (eligibility_screening, protocol_amendment).
CREATE TABLE ind_report_filed_ledger_entry (
    id        UUID PRIMARY KEY REFERENCES ledger_entry(id),
    ae_id     UUID         NOT NULL,
    grade     VARCHAR(20)  NOT NULL,
    submitted_at TIMESTAMP WITH TIME ZONE NOT NULL
);
```

- [ ] **Step 2: Create IndReportFiledLedgerEntry**

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident record for IND expedited safety report actually filed (WorkItem completed).
 *
 * <p>Written by RegulatorySubmissionLedgerWriter.writeFiledEntry() when
 * RegulatorySubmissionCompletedListener observes GoalReached for the regulatory-submission case.
 *
 * <p>JOINED inheritance on qhorus datasource. V2026.
 *
 * <p>subjectId = ae.enrollmentId — same Merkle chain as RegulatorySubmissionLedgerEntry.
 */
@Entity
@Table(name = "ind_report_filed_ledger_entry")
@DiscriminatorValue("IndReportFiled")
public class IndReportFiledLedgerEntry extends LedgerEntry {

    @Column(name = "ae_id", nullable = false)
    public UUID aeId;

    @Column(name = "grade", nullable = false, length = 20)
    public String grade;

    @Column(name = "submitted_at", nullable = false)
    public Instant submittedAt;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                aeId     != null ? aeId.toString()          : "",
                grade    != null ? grade                     : "",
                submittedAt != null ? submittedAt.toString() : "")
                .getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 3: Compile to verify**

```bash
mvn compile -pl runtime --batch-mode
```

Expected: `BUILD SUCCESS`

---

### Task C3: IndReportBreachLedgerEntry + V2027 migration

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/IndReportBreachLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2027__ind_report_breach_ledger_entry.sql`

- [ ] **Step 1: Write the Flyway migration**

`V2027__ind_report_breach_ledger_entry.sql`:

```sql
-- Ledger entry for IND reporting deadline exhausted (WorkItem ESCALATED terminal)
CREATE TABLE ind_report_breach_ledger_entry (
    id           UUID PRIMARY KEY REFERENCES ledger_entry(id),
    ae_id        UUID         NOT NULL,
    grade        VARCHAR(20)  NOT NULL,
    breached_at  TIMESTAMP WITH TIME ZONE NOT NULL,
    breach_reason VARCHAR(255)
);
```

- [ ] **Step 2: Create IndReportBreachLedgerEntry**

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident record for IND reporting deadline exhausted without submission.
 *
 * <p>Written by RegulatorySubmissionLedgerWriter.writeBreachEntry() when
 * RegulatorySubmissionBreachListener observes WorkItemLifecycleEvent(ESCALATED)
 * for a regulatory-submission WorkItem.
 *
 * <p>JOINED inheritance on qhorus datasource. V2027.
 *
 * <p>breachReason is a fixed string — WorkItemLifecycleEvent.detail() is always null for
 * ESCALATED events (ExpiryLifecycleService hardcodes null in fireLifecycleEvent()).
 */
@Entity
@Table(name = "ind_report_breach_ledger_entry")
@DiscriminatorValue("IndReportBreach")
public class IndReportBreachLedgerEntry extends LedgerEntry {

    @Column(name = "ae_id", nullable = false)
    public UUID aeId;

    @Column(name = "grade", nullable = false, length = 20)
    public String grade;

    @Column(name = "breached_at", nullable = false)
    public Instant breachedAt;

    @Column(name = "breach_reason", length = 255)
    public String breachReason;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                aeId      != null ? aeId.toString()        : "",
                grade     != null ? grade                   : "",
                breachedAt != null ? breachedAt.toString() : "")
                .getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 3: Compile to verify**

```bash
mvn compile -pl runtime --batch-mode
```

Expected: `BUILD SUCCESS`

---

### Task C4: ClinicalComplianceSupplement.regulatorySubmissionBreach() + RegulatorySubmissionLedgerWriter additions

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java`

- [ ] **Step 1: Write failing tests for writeFiledEntry() and writeBreachEntry()**

Add to `RegulatorySubmissionLedgerWriterTest.java`:

```java
@Test
void writeFiledEntry_writes_ind_report_filed_entry_with_correct_fields() {
    final Instant now = Instant.parse("2026-07-16T10:00:00Z");
    when(clock.instant()).thenReturn(now);
    when(ledgerEntryRepository.findLatestBySubjectId(any(), any())).thenReturn(Optional.empty());

    AdverseEvent ae = new AdverseEvent();
    ae.id = UUID.randomUUID();
    ae.enrollmentId = UUID.randomUUID();
    ae.grade = CtcaeGrade.GRADE_3;
    ae.tenantId = "test-tenant";

    writer.writeFiledEntry(ae);

    verify(ledgerEntryRepository).save(
            argThat(entry -> {
                IndReportFiledLedgerEntry e = (IndReportFiledLedgerEntry) entry;
                return e.aeId.equals(ae.id)
                        && "GRADE_3".equals(e.grade)
                        && e.submittedAt.equals(now)
                        && e.subjectId.equals(ae.enrollmentId)
                        && e.sequenceNumber == 1
                        && e.compliance()
                                .map(c -> c.planRef)
                                .orElse("")
                                .contains("(c)(1)(ii)");
            }), eq("default"));
}

@Test
void writeBreachEntry_writes_ind_report_breach_entry_with_fixed_reason() {
    final Instant now = Instant.parse("2026-07-16T10:00:00Z");
    when(clock.instant()).thenReturn(now);
    when(ledgerEntryRepository.findLatestBySubjectId(any(), any())).thenReturn(Optional.empty());

    AdverseEvent ae = new AdverseEvent();
    ae.id = UUID.randomUUID();
    ae.enrollmentId = UUID.randomUUID();
    ae.grade = CtcaeGrade.GRADE_5;
    ae.tenantId = "test-tenant";

    writer.writeBreachEntry(ae);

    verify(ledgerEntryRepository).save(
            argThat(entry -> {
                IndReportBreachLedgerEntry e = (IndReportBreachLedgerEntry) entry;
                return e.aeId.equals(ae.id)
                        && "GRADE_5".equals(e.grade)
                        && e.breachedAt.equals(now)
                        && e.subjectId.equals(ae.enrollmentId)
                        && e.sequenceNumber == 1
                        && "IND reporting deadline exhausted without submission"
                                .equals(e.breachReason)
                        && e.compliance()
                                .map(c -> c.algorithmRef)
                                .orElse("")
                                .contains("ClinicalIndReportingBreachPolicy");
            }), eq("default"));
}
```

Add import: `import io.casehub.clinical.ledger.IndReportFiledLedgerEntry;`, `import io.casehub.clinical.ledger.IndReportBreachLedgerEntry;`

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionLedgerWriterTest --batch-mode
```

Expected: FAIL — methods don't exist yet.

- [ ] **Step 3: Add regulatorySubmissionBreach() to ClinicalComplianceSupplement**

At the bottom of `ClinicalComplianceSupplement.java`:

```java
public static ComplianceSupplement regulatorySubmissionBreach(CtcaeGrade grade) {
    Objects.requireNonNull(grade, "grade");
    ComplianceSupplement s = new ComplianceSupplement();
    s.planRef = switch (grade) {
        case GRADE_5 -> "21 CFR 312.32(c)(1)(i) — IND 7-day expedited reporting deadline missed, unexpected fatal AE";
        case GRADE_4 -> "21 CFR 312.32(c)(1)(i) — IND 7-day expedited reporting deadline missed, unexpected life-threatening AE";
        case GRADE_3 -> "21 CFR 312.32(c)(1)(ii) — IND 15-day expedited reporting deadline missed, unexpected serious AE";
        default -> throw new IllegalArgumentException("no IND planRef for grade: " + grade);
    };
    s.algorithmRef = "ClinicalIndReportingBreachPolicy — IND deadline exhausted";
    s.humanOverrideAvailable = true;
    return s;
}
```

- [ ] **Step 4: Add writeFiledEntry() and writeBreachEntry() to RegulatorySubmissionLedgerWriter**

```java
@Transactional(Transactional.TxType.MANDATORY)
public void writeFiledEntry(AdverseEvent ae) {
    IndReportFiledLedgerEntry entry = new IndReportFiledLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = ae.enrollmentId;
    entry.sequenceNumber = nextSequenceNumber(ae.enrollmentId, "default");
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = ClinicalActors.CLINICAL_SERVICE;
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = "RegulatorySubmissionFiled";
    entry.occurredAt = clock.instant();
    entry.aeId = ae.id;
    entry.grade = ae.grade.name();
    entry.submittedAt = clock.instant();
    entry.attach(ClinicalComplianceSupplement.regulatorySubmission(ae.grade));
    ledgerEntryRepository.save(entry, "default");
}

@Transactional(Transactional.TxType.MANDATORY)
public void writeBreachEntry(AdverseEvent ae) {
    IndReportBreachLedgerEntry entry = new IndReportBreachLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = ae.enrollmentId;
    entry.sequenceNumber = nextSequenceNumber(ae.enrollmentId, "default");
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = ClinicalActors.CLINICAL_SERVICE;
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = "RegulatorySubmissionBreach";
    entry.occurredAt = clock.instant();
    entry.aeId = ae.id;
    entry.grade = ae.grade.name();
    entry.breachedAt = clock.instant();
    entry.breachReason = "IND reporting deadline exhausted without submission";
    entry.attach(ClinicalComplianceSupplement.regulatorySubmissionBreach(ae.grade));
    ledgerEntryRepository.save(entry, "default");
}
```

Add imports: `import io.casehub.clinical.ledger.IndReportFiledLedgerEntry;`, `import io.casehub.clinical.ledger.IndReportBreachLedgerEntry;`

- [ ] **Step 5: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionLedgerWriterTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 6: Commit ledger infrastructure**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/ledger/IndReportFiledLedgerEntry.java \
  runtime/src/main/java/io/casehub/clinical/ledger/IndReportBreachLedgerEntry.java \
  runtime/src/main/resources/db/migration/qhorus/V2026__ind_report_filed_ledger_entry.sql \
  runtime/src/main/resources/db/migration/qhorus/V2027__ind_report_breach_ledger_entry.sql \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java \
  runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriter.java \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionLedgerWriterTest.java

git -C /Users/mdproctor/claude/casehub/clinical commit -m "$(cat <<'EOF'
feat(#83): IND ledger infrastructure — IndReportFiled/Breach entries + V2026/V2027 migrations

- IndReportFiledLedgerEntry (@DiscriminatorValue("IndReportFiled")): aeId, grade, submittedAt
- IndReportBreachLedgerEntry (@DiscriminatorValue("IndReportBreach")): aeId, grade, breachedAt, breachReason
- Both: subjectId = ae.enrollmentId (Merkle chain continuity), domainContentBytes()
- V2026: ind_report_filed_ledger_entry join table
- V2027: ind_report_breach_ledger_entry join table (V2024/V2025 taken by Layer 9)
- ClinicalComplianceSupplement: regulatorySubmissionBreach(grade) factory method
- RegulatorySubmissionLedgerWriter: writeFiledEntry() + writeBreachEntry()

Refs casehubio/clinical#83
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task C5: ClinicalIndReportingBreachPolicy

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalIndReportingBreachPolicy.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/ClinicalIndReportingBreachPolicyTest.java`

- [ ] **Step 1: Write failing tests**

`ClinicalIndReportingBreachPolicyTest.java` — pure Mockito unit test, no Quarkus needed:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.BreachType;
import io.casehub.work.api.BreachedTask;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.platform.api.path.Path;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ClinicalIndReportingBreachPolicyTest {

    ClinicalIndReportingBreachPolicy policy;

    @BeforeEach
    void setUp() {
        policy = new ClinicalIndReportingBreachPolicy();
    }

    @Test
    void regulatory_affairs_first_breach_escalates_to_regulatory_leadership_48h() {
        SlaBreachContext ctx = ctx(Set.of("regulatory-affairs"));
        BreachDecision decision = policy.onBreach(ctx);
        assertThat(decision).isInstanceOf(BreachDecision.EscalateTo.class);
        BreachDecision.EscalateTo escalate = (BreachDecision.EscalateTo) decision;
        assertThat(escalate.groups()).containsExactly("regulatory-leadership");
        assertThat(escalate.deadline()).hasHours(48);
    }

    @Test
    void regulatory_leadership_second_breach_is_exhausted() {
        // After first escalation, ExpiryLifecycleService replaces candidateGroups
        // with the escalation group — "regulatory-affairs" is gone
        SlaBreachContext ctx = ctx(Set.of("regulatory-leadership"));
        BreachDecision decision = policy.onBreach(ctx);
        assertThat(decision).isInstanceOf(BreachDecision.Exhausted.class);
        BreachDecision.Exhausted exhausted = (BreachDecision.Exhausted) decision;
        assertThat(exhausted.reason()).contains("IND reporting deadline exhausted");
    }

    @Test
    void non_regulatory_workitem_returns_fail_with_no_policy_reason() {
        SlaBreachContext ctx = ctx(Set.of("senior-safety-monitors"));
        BreachDecision decision = policy.onBreach(ctx);
        assertThat(decision).isInstanceOf(BreachDecision.Fail.class);
        BreachDecision.Fail fail = (BreachDecision.Fail) decision;
        assertThat(fail.reason()).isEqualTo("no-sla-breach-policy-configured");
    }

    @Test
    void empty_groups_returns_fail() {
        SlaBreachContext ctx = ctx(Set.of());
        BreachDecision decision = policy.onBreach(ctx);
        assertThat(decision).isInstanceOf(BreachDecision.Fail.class);
    }

    private SlaBreachContext ctx(Set<String> candidateGroups) {
        BreachedTask task = new BreachedTask(UUID.randomUUID(), null, "Test Task", candidateGroups);
        return new SlaBreachContext(BreachType.COMPLETION_EXPIRED, task, Path.root(), null);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=ClinicalIndReportingBreachPolicyTest --batch-mode
```

Expected: FAIL — class doesn't exist yet.

- [ ] **Step 3: Implement ClinicalIndReportingBreachPolicy**

```java
package io.casehub.clinical.service;

import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.work.api.SlaBreachPolicy;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Duration;

/**
 * SlaBreachPolicy for IND expedited safety report filing WorkItems.
 *
 * <p>CDI: @ApplicationScoped (no @DefaultBean) — displaces NoOpSlaBreachPolicy @DefaultBean.
 * ExpiryLifecycleService injects SlaBreachPolicy as a singular @Inject point.
 *
 * <p>Pure: makes a decision and returns. No CDI calls, no DB queries, no side effects.
 *
 * <p>Stateless two-tier escalation via candidateGroups. After EscalateTo executes,
 * ExpiryLifecycleService replaces item.candidateGroups with the escalation group —
 * "regulatory-affairs" is gone on the second breach. Both groups must be tested
 * to identify regulatory WorkItems across both tiers.
 */
@ApplicationScoped
public class ClinicalIndReportingBreachPolicy implements SlaBreachPolicy {

    @Override
    public BreachDecision onBreach(SlaBreachContext ctx) {
        boolean isRegulatory = ctx.task().candidateGroups().contains("regulatory-affairs")
                || ctx.task().candidateGroups().contains("regulatory-leadership");
        if (!isRegulatory) {
            return new BreachDecision.Fail("no-sla-breach-policy-configured");
        }
        if (ctx.task().candidateGroups().contains("regulatory-leadership")) {
            return new BreachDecision.Exhausted(
                    "IND reporting deadline exhausted — operator intervention required");
        }
        return BreachDecision.EscalateTo.to("regulatory-leadership")
                .withDeadline(Duration.ofHours(48));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=ClinicalIndReportingBreachPolicyTest --batch-mode
```

Expected: all tests PASS.

---

### Task C6: RegulatorySubmissionCompletedListener

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCompletedListener.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCompletedListenerTest.java`

- [ ] **Step 1: Write failing integration tests**

`RegulatorySubmissionCompletedListenerTest.java`:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class RegulatorySubmissionCompletedListenerTest {

    @Inject RegulatorySubmissionCompletedListener listener;
    @InjectMock RegulatorySubmissionLedgerWriter ledgerWriter;

    @BeforeEach
    void stubLedgerWriter() {
        // no-op by default — verify is called in tests
    }

    @Test
    void goalReached_for_regulatory_submission_case_sets_status_filed() {
        UUID caseId = UUID.randomUUID();
        UUID aeId = persistAe(caseId, RegulatorySubmissionStatus.PENDING);

        listener.onCaseLifecycleEvent(event("GoalReached", caseId));

        assertThat(findAe(aeId).regulatorySubmissionStatus)
                .isEqualTo(RegulatorySubmissionStatus.FILED);
        verify(ledgerWriter).writeFiledEntry(any());
    }

    @Test
    void caseCompleted_sets_status_filed() {
        UUID caseId = UUID.randomUUID();
        UUID aeId = persistAe(caseId, RegulatorySubmissionStatus.PENDING);

        listener.onCaseLifecycleEvent(event("CaseCompleted", caseId));

        assertThat(findAe(aeId).regulatorySubmissionStatus)
                .isEqualTo(RegulatorySubmissionStatus.FILED);
    }

    @Test
    void unrelated_case_id_is_ignored() {
        listener.onCaseLifecycleEvent(event("GoalReached", UUID.randomUUID()));
        verify(ledgerWriter, never()).writeFiledEntry(any());
    }

    @Test
    void double_goalReached_is_idempotent() {
        UUID caseId = UUID.randomUUID();
        persistAe(caseId, RegulatorySubmissionStatus.PENDING);
        listener.onCaseLifecycleEvent(event("GoalReached", caseId));
        listener.onCaseLifecycleEvent(event("GoalReached", caseId));
        verify(ledgerWriter, org.mockito.Mockito.times(1)).writeFiledEntry(any());
    }

    @Test
    void already_deadline_missed_is_protected_by_guard() {
        UUID caseId = UUID.randomUUID();
        UUID aeId = persistAe(caseId, RegulatorySubmissionStatus.DEADLINE_MISSED);
        listener.onCaseLifecycleEvent(event("GoalReached", caseId));
        assertThat(findAe(aeId).regulatorySubmissionStatus)
                .isEqualTo(RegulatorySubmissionStatus.DEADLINE_MISSED);
        verify(ledgerWriter, never()).writeFiledEntry(any());
    }

    @Test
    void non_goalReached_eventType_is_ignored() {
        UUID caseId = UUID.randomUUID();
        persistAe(caseId, RegulatorySubmissionStatus.PENDING);
        listener.onCaseLifecycleEvent(event("CaseStarted", caseId));
        verify(ledgerWriter, never()).writeFiledEntry(any());
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    @Transactional
    UUID persistAe(UUID caseId, RegulatorySubmissionStatus status) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_3;
        ae.unexpected = true;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "test-tenant";
        ae.regulatorySubmissionCaseId = caseId;
        ae.regulatorySubmissionStatus = status;
        ae.persist();
        return ae.id;
    }

    @Transactional
    AdverseEvent findAe(UUID aeId) {
        return AdverseEvent.findById(aeId);
    }

    private CaseLifecycleEvent event(String eventType, UUID caseId) {
        return new CaseLifecycleEvent(
                caseId, "test-tenant", "SomeCommand", eventType,
                null, null, null, null);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionCompletedListenerTest --batch-mode
```

Expected: FAIL — class doesn't exist.

- [ ] **Step 3: Implement RegulatorySubmissionCompletedListener**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Updates ae.regulatorySubmissionStatus = FILED when the regulatory-submission case
 * reaches GoalReached or CaseCompleted.
 *
 * <p>Discriminates via DB lookup — CaseLifecycleEvent carries no case name or namespace.
 * Idempotency guard: only processes if status == PENDING.
 */
@ApplicationScoped
public class RegulatorySubmissionCompletedListener {

    private static final Logger LOG = Logger.getLogger(RegulatorySubmissionCompletedListener.class);

    @Inject RegulatorySubmissionLedgerWriter ledgerWriter;

    public void onCaseLifecycleEvent(@ObservesAsync CaseLifecycleEvent event) {
        if (!"GoalReached".equals(event.eventType()) && !"CaseCompleted".equals(event.eventType())) {
            return;
        }
        markFiled(event.caseId());
    }

    @Transactional
    void markFiled(UUID caseId) {
        AdverseEvent ae = AdverseEvent.find("regulatorySubmissionCaseId", caseId).firstResult();
        if (ae == null) {
            return;
        }
        if (ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.PENDING) {
            LOG.debugf("RegulatorySubmissionCompletedListener: caseId=%s status=%s — skipping (not PENDING)",
                    caseId, ae.regulatorySubmissionStatus);
            return;
        }
        ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.FILED;
        ledgerWriter.writeFiledEntry(ae);
        LOG.infof("RegulatorySubmissionCompletedListener: aeId=%s set FILED", ae.id);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionCompletedListenerTest --batch-mode
```

Expected: all tests PASS.

---

### Task C7: RegulatorySubmissionBreachListener

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionBreachListener.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionBreachListenerTest.java`

`CallerRef` and related types are in `io.casehub.workadapter` — already on the classpath via `casehub-engine-work-adapter`.

- [ ] **Step 1: Write failing integration tests**

`RegulatorySubmissionBreachListenerTest.java`:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.workadapter.PlanItemCallerRef;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class RegulatorySubmissionBreachListenerTest {

    @Inject RegulatorySubmissionBreachListener listener;
    @InjectMock RegulatorySubmissionLedgerWriter ledgerWriter;

    @BeforeEach
    void stub() {}

    @Test
    void escalated_event_for_regulatory_submission_sets_deadline_missed() {
        UUID caseId = UUID.randomUUID();
        UUID planItemId = UUID.randomUUID();
        UUID aeId = persistAe(caseId, RegulatorySubmissionStatus.PENDING);

        listener.onWorkItemLifecycle(escalatedEvent(
                PlanItemCallerRef.encode(caseId, planItemId.toString())));

        assertThat(findAe(aeId).regulatorySubmissionStatus)
                .isEqualTo(RegulatorySubmissionStatus.DEADLINE_MISSED);
        verify(ledgerWriter).writeBreachEntry(any());
    }

    @Test
    void non_regulatory_callerRef_is_ignored() {
        // "clinical:adverse-event/{id}" format — CallerRef.parse() returns null
        listener.onWorkItemLifecycle(escalatedEvent("clinical:adverse-event/" + UUID.randomUUID()));
        verify(ledgerWriter, never()).writeBreachEntry(any());
    }

    @Test
    void unrelated_case_id_is_ignored() {
        listener.onWorkItemLifecycle(escalatedEvent(
                PlanItemCallerRef.encode(UUID.randomUUID(), UUID.randomUUID().toString())));
        verify(ledgerWriter, never()).writeBreachEntry(any());
    }

    @Test
    void duplicate_escalated_event_is_idempotent() {
        UUID caseId = UUID.randomUUID();
        String callerRef = PlanItemCallerRef.encode(caseId, UUID.randomUUID().toString());
        persistAe(caseId, RegulatorySubmissionStatus.PENDING);
        listener.onWorkItemLifecycle(escalatedEvent(callerRef));
        listener.onWorkItemLifecycle(escalatedEvent(callerRef));
        verify(ledgerWriter, org.mockito.Mockito.times(1)).writeBreachEntry(any());
    }

    @Test
    void already_filed_is_protected_by_guard() {
        UUID caseId = UUID.randomUUID();
        UUID aeId = persistAe(caseId, RegulatorySubmissionStatus.FILED);
        listener.onWorkItemLifecycle(escalatedEvent(
                PlanItemCallerRef.encode(caseId, UUID.randomUUID().toString())));
        assertThat(findAe(aeId).regulatorySubmissionStatus)
                .isEqualTo(RegulatorySubmissionStatus.FILED);
        verify(ledgerWriter, never()).writeBreachEntry(any());
    }

    @Test
    void non_escalated_status_is_ignored() {
        UUID caseId = UUID.randomUUID();
        persistAe(caseId, RegulatorySubmissionStatus.PENDING);
        WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of(
                "COMPLETED", workItem(PlanItemCallerRef.encode(caseId, UUID.randomUUID().toString())),
                "system", null);
        listener.onWorkItemLifecycle(event);
        verify(ledgerWriter, never()).writeBreachEntry(any());
    }

    @Test
    void wire_reconstructed_event_source_null_is_ignored() {
        // fromWire() sets workItem = null
        WorkItemLifecycleEvent event = WorkItemLifecycleEvent.fromWire(
                "io.casehub.work.workitem.escalated", "/workitems/" + UUID.randomUUID(),
                UUID.randomUUID().toString(), UUID.randomUUID(), WorkItemStatus.ESCALATED,
                Instant.now(), "system", null, null, null, null, "test-tenant");
        listener.onWorkItemLifecycle(event);  // should not throw
        verify(ledgerWriter, never()).writeBreachEntry(any());
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    @Transactional
    UUID persistAe(UUID caseId, RegulatorySubmissionStatus status) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_5;
        ae.unexpected = true;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "test-tenant";
        ae.regulatorySubmissionCaseId = caseId;
        ae.regulatorySubmissionStatus = status;
        ae.persist();
        return ae.id;
    }

    @Transactional
    AdverseEvent findAe(UUID aeId) {
        return AdverseEvent.findById(aeId);
    }

    private WorkItemLifecycleEvent escalatedEvent(String callerRef) {
        return WorkItemLifecycleEvent.of("ESCALATED", workItem(callerRef), "system", null);
    }

    private WorkItem workItem(String callerRef) {
        WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.callerRef = callerRef;
        wi.status = WorkItemStatus.ESCALATED;
        wi.tenancyId = "test-tenant";
        return wi;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionBreachListenerTest --batch-mode
```

Expected: FAIL — class doesn't exist.

- [ ] **Step 3: Implement RegulatorySubmissionBreachListener**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.workadapter.CallerRef;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Sets ae.regulatorySubmissionStatus = DEADLINE_MISSED when a regulatory submission WorkItem
 * reaches ESCALATED terminal status (breach exhausted).
 *
 * <p>Discrimination: CallerRef.parse(workItem.callerRef) → caseId →
 * AdverseEvent.find("regulatorySubmissionCaseId", caseId).
 *
 * <p>Pattern matches IrbDecisionListener exactly:
 * - instanceof WorkItem guard (null-safe, type-safe)
 * - CallerRef.parse() for callerRef extraction (existing sealed interface API)
 * - != PENDING idempotency guard
 */
@ApplicationScoped
public class RegulatorySubmissionBreachListener {

    private static final Logger LOG = Logger.getLogger(RegulatorySubmissionBreachListener.class);

    @Inject RegulatorySubmissionLedgerWriter ledgerWriter;

    public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
        if (event.status() != WorkItemStatus.ESCALATED) {
            return;
        }
        if (!(event.source() instanceof WorkItem workItem)) {
            return;
        }
        CallerRef ref = CallerRef.parse(workItem.callerRef);
        if (ref == null) {
            return;
        }
        markDeadlineMissed(ref.caseId());
    }

    @Transactional
    void markDeadlineMissed(UUID caseId) {
        AdverseEvent ae = AdverseEvent.find("regulatorySubmissionCaseId", caseId).firstResult();
        if (ae == null) {
            return;
        }
        if (ae.regulatorySubmissionStatus != RegulatorySubmissionStatus.PENDING) {
            LOG.debugf("RegulatorySubmissionBreachListener: caseId=%s status=%s — skipping (not PENDING)",
                    caseId, ae.regulatorySubmissionStatus);
            return;
        }
        ae.regulatorySubmissionStatus = RegulatorySubmissionStatus.DEADLINE_MISSED;
        ledgerWriter.writeBreachEntry(ae);
        LOG.infof("RegulatorySubmissionBreachListener: aeId=%s set DEADLINE_MISSED", ae.id);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionBreachListenerTest --batch-mode
```

Expected: all tests PASS.

- [ ] **Step 5: Commit compliance layer**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalIndReportingBreachPolicy.java \
  runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionCompletedListener.java \
  runtime/src/main/java/io/casehub/clinical/service/RegulatorySubmissionBreachListener.java \
  runtime/src/test/java/io/casehub/clinical/service/ClinicalIndReportingBreachPolicyTest.java \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionCompletedListenerTest.java \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionBreachListenerTest.java

git -C /Users/mdproctor/claude/casehub/clinical commit -m "$(cat <<'EOF'
feat(#83): IND compliance layer — breach policy, completion/breach listeners

ClinicalIndReportingBreachPolicy (@ApplicationScoped, displaces NoOpSlaBreachPolicy):
  - isRegulatory guard tests both "regulatory-affairs" and "regulatory-leadership"
    (ExpiryLifecycleService replaces candidateGroups after EscalateTo)
  - First breach: EscalateTo("regulatory-leadership", 48h)
  - Second breach: Exhausted
  - Non-regulatory: Fail("no-sla-breach-policy-configured")

RegulatorySubmissionCompletedListener:
  - @ObservesAsync GoalReached/CaseCompleted; DB lookup discriminates; != PENDING guard

RegulatorySubmissionBreachListener:
  - @ObservesAsync WorkItemLifecycleEvent(ESCALATED); instanceof WorkItem guard;
    CallerRef.parse() (IrbDecisionListener pattern); != PENDING guard

Refs casehubio/clinical#83
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task C8: RegulatorySubmissionDeadlineLifecycleTest — end-to-end invariant

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionDeadlineLifecycleTest.java`

This test verifies the entire data path: `ae.reportedAt + window → indReportingDeadline → case context → expiresAtExpression JQ → expiresAtDeadline → WorkItem.expiresAt`.

**Important:** This is a `@QuarkusTest` with the full engine on classpath. Do NOT add `@InjectMock` on `ClinicalRegulatorySubmissionCaseHub` — the real case must run. Stamp `ae.tenantId = principal.tenancyId()` per CLAUDE.md pattern.

- [ ] **Step 1: Write the lifecycle test**

```java
package io.casehub.clinical.service;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.RegulatorySubmissionStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/**
 * End-to-end invariant test: WorkItem.expiresAt == ae.reportedAt + indReportingWindow(grade).
 *
 * Full engine on classpath. No @InjectMock on ClinicalRegulatorySubmissionCaseHub.
 * HumanTaskScheduleHandler fires on a Vert.x worker thread — Awaitility required.
 */
@QuarkusTest
class RegulatorySubmissionDeadlineLifecycleTest {

    @Inject RegulatorySubmissionCaseService service;
    @Inject WorkItemStore workItemStore;
    @Inject FixedCurrentPrincipal principal;

    @Test
    void grade3_workItem_expiresAt_equals_reportedAt_plus_15_days() {
        Instant reportedAt = Instant.parse("2026-07-01T10:00:00Z");
        UUID aeId = persistAe(CtcaeGrade.GRADE_3, reportedAt);
        AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_3);

        service.onAdverseEventReported(event);

        Instant expectedExpiry = Instant.parse("2026-07-16T10:00:00Z"); // +15 days
        await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
            var workItem = workItemStore.scanAll().stream()
                .filter(wi -> wi.candidateGroups != null
                        && wi.candidateGroups.contains("regulatory-affairs")
                        && wi.callerRef != null
                        && wi.callerRef.startsWith("case:"))
                .findFirst()
                .orElseThrow(() -> new AssertionError("No regulatory-affairs WorkItem found yet"));
            assertThat(workItem.expiresAt).isEqualTo(expectedExpiry);
            assertThat(workItem.status).isEqualTo(WorkItemStatus.PENDING);
        });
    }

    @Test
    void grade4_workItem_expiresAt_equals_reportedAt_plus_7_days() {
        Instant reportedAt = Instant.parse("2026-07-01T10:00:00Z");
        UUID aeId = persistAe(CtcaeGrade.GRADE_4, reportedAt);
        AdverseEventReportedEvent event = buildEvent(aeId, CtcaeGrade.GRADE_4);

        service.onAdverseEventReported(event);

        Instant expectedExpiry = Instant.parse("2026-07-08T10:00:00Z"); // +7 days
        await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
            var workItem = workItemStore.scanAll().stream()
                .filter(wi -> wi.candidateGroups != null
                        && wi.candidateGroups.contains("regulatory-affairs")
                        && wi.callerRef != null
                        && wi.callerRef.startsWith("case:"))
                .findFirst()
                .orElseThrow(() -> new AssertionError("No regulatory-affairs WorkItem found yet"));
            assertThat(workItem.expiresAt).isEqualTo(expectedExpiry);
        });
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    @Transactional
    UUID persistAe(CtcaeGrade grade, Instant reportedAt) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = grade;
        ae.unexpected = true;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = reportedAt;
        ae.reportedAt = reportedAt;
        // Required per CLAUDE.md: prevents SecurityException from MemoryPermissions.assertTenant()
        ae.tenantId = principal.tenancyId();
        ae.persist();
        return ae.id;
    }

    private AdverseEventReportedEvent buildEvent(UUID aeId, CtcaeGrade grade) {
        return new AdverseEventReportedEvent(
                aeId, UUID.randomUUID(), UUID.randomUUID(), grade,
                Instant.now(), principal.tenancyId());
    }
}
```

- [ ] **Step 2: Run the test**

```bash
mvn install -pl api --batch-mode && \
mvn test -pl runtime -Dtest=RegulatorySubmissionDeadlineLifecycleTest --batch-mode
```

Expected: PASS for both grade3 and grade4 tests.

If the engine SNAPSHOT has not been published yet (expiresAtExpression not in local repo), you may need to install the engine locally first:
```bash
mvn install --batch-mode -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

---

### Task C9: Full build validation + final commit

- [ ] **Step 1: Run full clinical test suite**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode
```

Expected: `BUILD SUCCESS` — all tests pass.

- [ ] **Step 2: Verify existing service tests unchanged**

```bash
mvn test -pl runtime -Dtest=RegulatorySubmissionCaseServiceTest --batch-mode
```

Expected: all existing tests PASS (service API is unmodified).

- [ ] **Step 3: Commit tests + lifecycle test**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/RegulatorySubmissionDeadlineLifecycleTest.java

git -C /Users/mdproctor/claude/casehub/clinical commit -m "$(cat <<'EOF'
test(#83): RegulatorySubmissionDeadlineLifecycleTest — end-to-end deadline invariant

Verifies the full data path:
  ae.reportedAt + window → case context → .indReportingDeadline JQ → WorkItem.expiresAt

Grade 3 (15d): reportedAt + 15d == WorkItem.expiresAt
Grade 4 (7d):  reportedAt + 7d  == WorkItem.expiresAt

Uses real engine (no CaseHub mock). Awaitility for async HumanTaskScheduleHandler.
ae.tenantId = principal.tenancyId() per CLAUDE.md pattern.

Refs casehubio/clinical#83
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Post-Implementation

- [ ] Run `superpowers:requesting-code-review` on all changes before final sign-off
- [ ] Run `implementation-doc-sync` after final commit
- [ ] Push engine branch + create engine PR
- [ ] Push clinical branch + create clinical PR referencing engine#N
- [ ] Verify both CI pipelines green
