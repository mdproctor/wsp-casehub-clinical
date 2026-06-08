# DSL Companions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add fluent Java DSL companion classes for the three clinical YAML case definitions, each producing the same canonical `CaseDefinition` as the YAML mapper, verified by a pure-Java equivalence test.

**Architecture:** Three `public final class` companions in `runtime/src/main/java/io/casehub/clinical/casedefinition/`, each with a single `static CaseDefinition build()` factory. JQ strings are copied verbatim from the YAML so `JQExpressionEvaluator` record equality makes the equivalence test structurally meaningful. Goals are registered in two independent places (builder `.goals()` and `GoalExpression.allOf()` passed to `.completion()`) because `CaseDefinition.Builder.build()` populates `def.getGoals()` and `def.getCompletion()` via completely separate code paths. One equivalence test class in test scope asserts structural parity field-by-field per companion.

**Tech Stack:** Java 21, casehub-engine-api 0.2-SNAPSHOT (`CaseDefinition`, `Goal`, `Binding`, `ContextChangeTrigger`, `HumanTaskTarget`, `GoalExpression`, `CaseDefinitionYamlMapper`), JUnit 5, AssertJ.

**Spec:** `docs/specs/2026-06-08-dsl-companions-design.md`  
**Issue:** casehubio/clinical#50  
**Branch:** `issue-50-dsl-companions`

---

## File Map

| Action | Path | Responsibility |
|---|---|---|
| Create | `runtime/src/main/java/io/casehub/clinical/casedefinition/DeviationReviewCaseDefinition.java` | DSL mirror of `deviation-review.yaml` |
| Create | `runtime/src/main/java/io/casehub/clinical/casedefinition/AeEscalationCaseDefinition.java` | DSL mirror of `ae-escalation.yaml` |
| Create | `runtime/src/main/java/io/casehub/clinical/casedefinition/TrialCoordinationCaseDefinition.java` | DSL mirror of `trial-coordination.yaml` |
| Create | `runtime/src/test/java/io/casehub/clinical/casedefinition/ClinicalCaseDefinitionEquivalenceTest.java` | Structural equivalence verification, plain JUnit 5 |

---

## Task 1: Write failing test — DeviationReviewCaseDefinition

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/casedefinition/ClinicalCaseDefinitionEquivalenceTest.java`

- [ ] **Step 1.1: Create the test class with the deviation-review test method**

```java
// runtime/src/test/java/io/casehub/clinical/casedefinition/ClinicalCaseDefinitionEquivalenceTest.java
package io.casehub.clinical.casedefinition;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.AllOfGoalExpression;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.GoalBasedCompletion;
import io.casehub.api.model.HumanTaskTarget;
import io.casehub.api.model.converter.CaseDefinitionYamlMapper;
import java.io.IOException;
import org.junit.jupiter.api.Test;

class ClinicalCaseDefinitionEquivalenceTest {

    @Test
    void deviationReview_dslMatchesYaml() throws IOException {
        var fromYaml = CaseDefinitionYamlMapper.load(
            getClass().getClassLoader().getResourceAsStream("clinical/deviation-review.yaml"));
        var fromDsl = DeviationReviewCaseDefinition.build();

        assertThat(fromDsl.getNamespace()).isEqualTo(fromYaml.getNamespace());
        assertThat(fromDsl.getName()).isEqualTo(fromYaml.getName());
        assertThat(fromDsl.getVersion()).isEqualTo(fromYaml.getVersion());
        assertThat(fromDsl.getTitle()).isEqualTo(fromYaml.getTitle());

        // Goals
        assertThat(fromDsl.getGoals()).hasSameSizeAs(fromYaml.getGoals());
        for (int i = 0; i < fromYaml.getGoals().size(); i++) {
            var yamlGoal = fromYaml.getGoals().get(i);
            var dslGoal = fromDsl.getGoals().get(i);
            assertThat(dslGoal.getName()).isEqualTo(yamlGoal.getName());
            assertThat(dslGoal.getKind()).isEqualTo(yamlGoal.getKind());
            assertThat(dslGoal.getCondition()).isEqualTo(yamlGoal.getCondition());
        }

        // Bindings
        assertThat(fromDsl.getBindings()).hasSameSizeAs(fromYaml.getBindings());
        for (int i = 0; i < fromYaml.getBindings().size(); i++) {
            var yamlBinding = fromYaml.getBindings().get(i);
            var dslBinding = fromDsl.getBindings().get(i);
            assertThat(dslBinding.getName()).isEqualTo(yamlBinding.getName());
            assertThat(dslBinding.target().getClass())
                .as("binding '%s' target type", dslBinding.getName())
                .isEqualTo(yamlBinding.target().getClass());
            assertThat(((ContextChangeTrigger) dslBinding.getOn()).getFilter())
                .isEqualTo(((ContextChangeTrigger) yamlBinding.getOn()).getFilter());
            if (yamlBinding.target() instanceof HumanTaskTarget yamlHT
                    && dslBinding.target() instanceof HumanTaskTarget dslHT) {
                assertThat(dslHT.title()).isEqualTo(yamlHT.title());
                assertThat(dslHT.expiresIn()).isEqualTo(yamlHT.expiresIn());
                assertThat(dslHT.candidateGroups()).isEqualTo(yamlHT.candidateGroups());
                assertThat(dslHT.inputMapping()).isEqualTo(yamlHT.inputMapping());
                assertThat(dslHT.outputMapping()).isEqualTo(yamlHT.outputMapping());
            }
        }

        // Completion
        assertThat(fromDsl.getCompletion()).isInstanceOf(GoalBasedCompletion.class);
        var dslCompletion = (GoalBasedCompletion) fromDsl.getCompletion();
        assertThat(dslCompletion.getSuccess()).isInstanceOf(AllOfGoalExpression.class);
        assertThat(((AllOfGoalExpression) dslCompletion.getSuccess()).getGoals())
            .containsExactlyInAnyOrderElementsOf(
                ((AllOfGoalExpression) ((GoalBasedCompletion) fromYaml.getCompletion()).getSuccess()).getGoals());
        assertThat(dslCompletion.getFailure()).isNull();
    }
}
```

- [ ] **Step 1.2: Run to confirm compile failure (companion class missing)**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalCaseDefinitionEquivalenceTest --batch-mode
```

Expected: **COMPILATION FAILURE** — `cannot find symbol: class DeviationReviewCaseDefinition`

---

## Task 2: Implement DeviationReviewCaseDefinition

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/casedefinition/DeviationReviewCaseDefinition.java`

- [ ] **Step 2.1: Create the companion class**

```java
// runtime/src/main/java/io/casehub/clinical/casedefinition/DeviationReviewCaseDefinition.java
package io.casehub.clinical.casedefinition;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.Goal;
import io.casehub.api.model.GoalExpression;
import io.casehub.api.model.GoalKind;
import io.casehub.api.model.HumanTaskTarget;
import java.time.Duration;
import java.util.Set;

/** Fluent DSL companion for {@code clinical/deviation-review.yaml}. */
public final class DeviationReviewCaseDefinition {

    private DeviationReviewCaseDefinition() {}

    public static CaseDefinition build() {
        var irbDecided = Goal.builder()
            .name("irb-decided")
            .kind(GoalKind.SUCCESS)
            .condition(".irbConsultation != null")
            .build();

        return CaseDefinition.builder()
            .namespace("clinical")
            .name("deviation-review")
            .version("1.0.0")
            .title("Protocol Deviation Review — IRB consultation gate")
            .goals(irbDecided)
            .completion(GoalExpression.allOf(irbDecided))
            .bindings(
                Binding.builder()
                    .name("irb-consultation")
                    .on(new ContextChangeTrigger(".irbConsultationRequired == true and .irbConsultation == null"))
                    .humanTask(HumanTaskTarget.inline()
                        .title("IRB consultation required — protocol deviation")
                        .expiresIn(Duration.parse("PT72H"))
                        .candidateGroups(Set.of("irb-committee"))
                        .inputMapping("{ deviationId: .deviationId, severity: .severity }")
                        .outputMapping("{ irbConsultation: . }")
                        .build())
                    .build()
            )
            .build();
    }
}
```

- [ ] **Step 2.2: Run to confirm the first test passes**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalCaseDefinitionEquivalenceTest --batch-mode
```

Expected: **Tests run: 1, Failures: 0, Errors: 0**

---

## Task 3: Add ae-escalation test method

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/casedefinition/ClinicalCaseDefinitionEquivalenceTest.java`

- [ ] **Step 3.1: Add the ae-escalation test method to the existing test class**

Add this method inside `ClinicalCaseDefinitionEquivalenceTest`, after `deviationReview_dslMatchesYaml()`:

```java
    @Test
    void aeEscalation_dslMatchesYaml() throws IOException {
        var fromYaml = CaseDefinitionYamlMapper.load(
            getClass().getClassLoader().getResourceAsStream("clinical/ae-escalation.yaml"));
        var fromDsl = AeEscalationCaseDefinition.build();

        assertThat(fromDsl.getNamespace()).isEqualTo(fromYaml.getNamespace());
        assertThat(fromDsl.getName()).isEqualTo(fromYaml.getName());
        assertThat(fromDsl.getVersion()).isEqualTo(fromYaml.getVersion());
        assertThat(fromDsl.getTitle()).isEqualTo(fromYaml.getTitle());

        // Goals
        assertThat(fromDsl.getGoals()).hasSameSizeAs(fromYaml.getGoals());
        for (int i = 0; i < fromYaml.getGoals().size(); i++) {
            var yamlGoal = fromYaml.getGoals().get(i);
            var dslGoal = fromDsl.getGoals().get(i);
            assertThat(dslGoal.getName()).isEqualTo(yamlGoal.getName());
            assertThat(dslGoal.getKind()).isEqualTo(yamlGoal.getKind());
            assertThat(dslGoal.getCondition()).isEqualTo(yamlGoal.getCondition());
        }

        // Bindings
        assertThat(fromDsl.getBindings()).hasSameSizeAs(fromYaml.getBindings());
        for (int i = 0; i < fromYaml.getBindings().size(); i++) {
            var yamlBinding = fromYaml.getBindings().get(i);
            var dslBinding = fromDsl.getBindings().get(i);
            assertThat(dslBinding.getName()).isEqualTo(yamlBinding.getName());
            assertThat(dslBinding.target().getClass())
                .as("binding '%s' target type", dslBinding.getName())
                .isEqualTo(yamlBinding.target().getClass());
            assertThat(((ContextChangeTrigger) dslBinding.getOn()).getFilter())
                .isEqualTo(((ContextChangeTrigger) yamlBinding.getOn()).getFilter());
            if (yamlBinding.target() instanceof HumanTaskTarget yamlHT
                    && dslBinding.target() instanceof HumanTaskTarget dslHT) {
                assertThat(dslHT.title()).isEqualTo(yamlHT.title());
                assertThat(dslHT.expiresIn()).isEqualTo(yamlHT.expiresIn());
                assertThat(dslHT.candidateGroups()).isEqualTo(yamlHT.candidateGroups());
                assertThat(dslHT.inputMapping()).isEqualTo(yamlHT.inputMapping());
                assertThat(dslHT.outputMapping()).isEqualTo(yamlHT.outputMapping());
            }
        }

        // Completion
        assertThat(fromDsl.getCompletion()).isInstanceOf(GoalBasedCompletion.class);
        var dslCompletion = (GoalBasedCompletion) fromDsl.getCompletion();
        assertThat(dslCompletion.getSuccess()).isInstanceOf(AllOfGoalExpression.class);
        assertThat(((AllOfGoalExpression) dslCompletion.getSuccess()).getGoals())
            .containsExactlyInAnyOrderElementsOf(
                ((AllOfGoalExpression) ((GoalBasedCompletion) fromYaml.getCompletion()).getSuccess()).getGoals());
        assertThat(dslCompletion.getFailure()).isNull();
    }
```

- [ ] **Step 3.2: Run to confirm compile failure (companion class missing)**

```bash
mvn test -pl runtime -Dtest=ClinicalCaseDefinitionEquivalenceTest --batch-mode
```

Expected: **COMPILATION FAILURE** — `cannot find symbol: class AeEscalationCaseDefinition`

---

## Task 4: Implement AeEscalationCaseDefinition

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/casedefinition/AeEscalationCaseDefinition.java`

- [ ] **Step 4.1: Create the companion class**

```java
// runtime/src/main/java/io/casehub/clinical/casedefinition/AeEscalationCaseDefinition.java
package io.casehub.clinical.casedefinition;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.Goal;
import io.casehub.api.model.GoalExpression;
import io.casehub.api.model.GoalKind;
import io.casehub.api.model.HumanTaskTarget;
import java.time.Duration;
import java.util.Set;

/** Fluent DSL companion for {@code clinical/ae-escalation.yaml}. */
public final class AeEscalationCaseDefinition {

    private AeEscalationCaseDefinition() {}

    public static CaseDefinition build() {
        var safetyReviewComplete = Goal.builder()
            .name("safety-review-complete")
            .kind(GoalKind.SUCCESS)
            .condition(".safetyReview != null")
            .build();

        var dsmbComplete = Goal.builder()
            .name("dsmb-complete")
            .kind(GoalKind.SUCCESS)
            .condition(".requiresDsmbEscalation == false or .dsmbEscalation != null")
            .build();

        return CaseDefinition.builder()
            .namespace("clinical")
            .name("ae-escalation")
            .version("1.0.0")
            .title("Adverse Event Safety Escalation — adaptive severity routing")
            .goals(safetyReviewComplete, dsmbComplete)
            .completion(GoalExpression.allOf(safetyReviewComplete, dsmbComplete))
            .bindings(
                Binding.builder()
                    .name("safety-review")
                    .on(new ContextChangeTrigger(".requiresSeniorMonitor == true and .safetyReview == null"))
                    .humanTask(HumanTaskTarget.inline()
                        .title("Senior safety monitor review — adverse event")
                        .expiresIn(Duration.parse("PT24H"))
                        .candidateGroups(Set.of("senior-safety-monitors"))
                        .inputMapping("{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId }")
                        .outputMapping("{ safetyReview: . }")
                        .build())
                    .build(),
                Binding.builder()
                    .name("dsmb-escalation")
                    .on(new ContextChangeTrigger(".requiresDsmbEscalation == true and .dsmbEscalation == null"))
                    .humanTask(HumanTaskTarget.inline()
                        .title("DSMB escalation — Grade 4+ adverse event")
                        .expiresIn(Duration.parse("PT24H"))
                        .candidateGroups(Set.of("dsmb"))
                        .inputMapping("{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId }")
                        .outputMapping("{ dsmbEscalation: . }")
                        .build())
                    .build()
            )
            .build();
    }
}
```

- [ ] **Step 4.2: Run to confirm both tests pass**

```bash
mvn test -pl runtime -Dtest=ClinicalCaseDefinitionEquivalenceTest --batch-mode
```

Expected: **Tests run: 2, Failures: 0, Errors: 0**

---

## Task 5: Add trial-coordination test method

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/casedefinition/ClinicalCaseDefinitionEquivalenceTest.java`

- [ ] **Step 5.1: Add the trial-coordination test method**

Add this method inside `ClinicalCaseDefinitionEquivalenceTest`, after `aeEscalation_dslMatchesYaml()`:

```java
    @Test
    void trialCoordination_dslMatchesYaml() throws IOException {
        var fromYaml = CaseDefinitionYamlMapper.load(
            getClass().getClassLoader().getResourceAsStream("clinical/trial-coordination.yaml"));
        var fromDsl = TrialCoordinationCaseDefinition.build();

        assertThat(fromDsl.getNamespace()).isEqualTo(fromYaml.getNamespace());
        assertThat(fromDsl.getName()).isEqualTo(fromYaml.getName());
        assertThat(fromDsl.getVersion()).isEqualTo(fromYaml.getVersion());
        assertThat(fromDsl.getTitle()).isEqualTo(fromYaml.getTitle());

        // No goals — trial runs for its lifetime with no completion condition
        assertThat(fromDsl.getGoals()).isEmpty();
        assertThat(fromDsl.getCompletion()).isNull();

        // Bindings
        assertThat(fromDsl.getBindings()).hasSameSizeAs(fromYaml.getBindings());
        for (int i = 0; i < fromYaml.getBindings().size(); i++) {
            var yamlBinding = fromYaml.getBindings().get(i);
            var dslBinding = fromDsl.getBindings().get(i);
            assertThat(dslBinding.getName()).isEqualTo(yamlBinding.getName());
            assertThat(dslBinding.target().getClass())
                .as("binding '%s' target type", dslBinding.getName())
                .isEqualTo(yamlBinding.target().getClass());
            assertThat(((ContextChangeTrigger) dslBinding.getOn()).getFilter())
                .isEqualTo(((ContextChangeTrigger) yamlBinding.getOn()).getFilter());
            if (yamlBinding.target() instanceof HumanTaskTarget yamlHT
                    && dslBinding.target() instanceof HumanTaskTarget dslHT) {
                assertThat(dslHT.title()).isEqualTo(yamlHT.title());
                assertThat(dslHT.expiresIn()).isEqualTo(yamlHT.expiresIn());
                assertThat(dslHT.candidateGroups()).isEqualTo(yamlHT.candidateGroups());
                assertThat(dslHT.inputMapping()).isEqualTo(yamlHT.inputMapping());
                assertThat(dslHT.outputMapping()).isEqualTo(yamlHT.outputMapping());
            }
        }
    }
```

- [ ] **Step 5.2: Run to confirm compile failure**

```bash
mvn test -pl runtime -Dtest=ClinicalCaseDefinitionEquivalenceTest --batch-mode
```

Expected: **COMPILATION FAILURE** — `cannot find symbol: class TrialCoordinationCaseDefinition`

---

## Task 6: Implement TrialCoordinationCaseDefinition

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/casedefinition/TrialCoordinationCaseDefinition.java`

- [ ] **Step 6.1: Create the companion class**

```java
// runtime/src/main/java/io/casehub/clinical/casedefinition/TrialCoordinationCaseDefinition.java
package io.casehub.clinical.casedefinition;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.HumanTaskTarget;
import java.time.Duration;
import java.util.Set;

/** Fluent DSL companion for {@code clinical/trial-coordination.yaml}. */
public final class TrialCoordinationCaseDefinition {

    private TrialCoordinationCaseDefinition() {}

    public static CaseDefinition build() {
        return CaseDefinition.builder()
            .namespace("clinical")
            .name("trial-coordination")
            .version("1.0.0")
            .title("Clinical Trial Coordination — cross-site safety monitoring")
            .bindings(
                Binding.builder()
                    .name("dsmb-rollup")
                    .on(new ContextChangeTrigger("[.grade4Active // {} | to_entries[] | select(.value == true)] | length >= 2"))
                    .humanTask(HumanTaskTarget.inline()
                        .title("DSMB review — simultaneous Grade 4+ events at multiple sites")
                        .expiresIn(Duration.parse("PT48H"))
                        .candidateGroups(Set.of("dsmb"))
                        .inputMapping("{ trialId: .trialId, activeSites: [.grade4Active // {} | to_entries[] | select(.value == true) | .key] }")
                        .outputMapping("{ dsmbReview: . }")
                        .build())
                    .build()
            )
            .build();
    }
}
```

- [ ] **Step 6.2: Run to confirm all three tests pass**

```bash
mvn test -pl runtime -Dtest=ClinicalCaseDefinitionEquivalenceTest --batch-mode
```

Expected: **Tests run: 3, Failures: 0, Errors: 0**

---

## Task 7: Full suite verification and commit

- [ ] **Step 7.1: Run the full runtime test suite to confirm no regressions**

```bash
mvn test -pl runtime --batch-mode
```

Expected: All existing tests continue to pass; `ClinicalCaseDefinitionEquivalenceTest` adds 3 passing tests.

- [ ] **Step 7.2: Stage all new files**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/casedefinition/DeviationReviewCaseDefinition.java \
  runtime/src/main/java/io/casehub/clinical/casedefinition/AeEscalationCaseDefinition.java \
  runtime/src/main/java/io/casehub/clinical/casedefinition/TrialCoordinationCaseDefinition.java \
  runtime/src/test/java/io/casehub/clinical/casedefinition/ClinicalCaseDefinitionEquivalenceTest.java
```

- [ ] **Step 7.3: Proceed to code review before committing**

Do not commit yet. Invoke `superpowers:requesting-code-review` on the staged changes.

---

## Self-Review

**Spec coverage check:**
- ✅ Three companion classes in `src/main/java/io/casehub/clinical/casedefinition/` — Tasks 2, 4, 6
- ✅ `public final class` with private constructor and static `build()` — all three companions
- ✅ Dual goal registration (`.goals()` + `GoalExpression.allOf()`) — Tasks 2 and 4
- ✅ JQ strings verbatim from YAML — all field values match spec tables exactly
- ✅ `dsl` field not set — no `setDsl()` call anywhere
- ✅ Equivalence test in `src/test/java/io/casehub/clinical/casedefinition/` — Task 1, 3, 5
- ✅ `CaseDefinitionYamlMapper.load()` (not `parse()`) — Tasks 1, 3, 5
- ✅ Goals: count + name + kind + condition — all three test methods
- ✅ Bindings: count + name + target type + trigger filter + humanTask fields — all three test methods
- ✅ Completion: `isInstanceOf(GoalBasedCompletion)` + `isInstanceOf(AllOfGoalExpression)` + `containsExactlyInAnyOrderElementsOf` + `getFailure().isNull()` — Tasks 1 and 3
- ✅ TrialCoordination: null completion + empty goals — Task 5
- ✅ `TrialCoordinationCaseDefinition` has no `.goals()` or `.completion()` call — Task 6

**Placeholder scan:** None found. All code blocks are complete.

**Type consistency:**
- `DeviationReviewCaseDefinition.build()` returns `CaseDefinition` ✅
- `AeEscalationCaseDefinition.build()` returns `CaseDefinition` ✅
- `TrialCoordinationCaseDefinition.build()` returns `CaseDefinition` ✅
- Test class references all three by their exact class names ✅
