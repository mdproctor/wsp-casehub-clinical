---
layout: post
title: "Two authoring paths, one canonical model"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [casehub-engine, case-definition, dsl, testing]
---

Protocol PP-20260518-case-definition-layers has a simple mandate: every YAML case definition needs a companion fluent Java DSL class that produces the same canonical `CaseDefinition`. Clinical has three YAML definitions and had none. Today we fixed that.

The first question was scope. My instinct was test scope — no production consumers exist, and the YAML path via `YamlCaseHub` is the deployed authoring path. But the devtown reference implementation (`PrReviewCaseDefinition`) is in `src/main/java`, `public final class`. The protocol says "two equal production-grade authoring paths." Test scope contradicts that without any architectural benefit. So production scope it is.

The more interesting question was which evaluator to use. The devtown companion uses `LambdaExpressionEvaluator` throughout — type-safe, no JQ parsing overhead, conditions co-located with the code. That's the right choice for a test fixture. But I wanted the equivalence test to actually prove structural parity, not just verify that the definition runs. Lambda predicates have no value equality — you can't assert that `ctx -> ctx.get("x") != null` matches `.x != null`. The test would be checking behaviour, not structure.

The insight that changed the approach: `JQExpressionEvaluator` is a Java record. Records implement `equals()` by field value. If we copy the JQ strings verbatim from the YAML source, both the YAML mapper and the DSL companion produce `JQExpressionEvaluator` instances wrapping identical strings. The equivalence test can then use a plain `assertThat(dslGoal.getCondition()).isEqualTo(yamlGoal.getCondition())` — no cast, no `.toString()` workaround. The same holds for trigger filters, input mappings, and output mappings. `ListEvaluator.StaticList` for `candidateGroups` is also a record, so group comparison works the same way.

This made the equivalence test genuinely meaningful. Not "do these produce the same results?" but "do these produce the same model?"

The catch we uncovered while reading the decompiled `CaseDefinition.Builder.build()` bytecode: the goals list and the completion expression are completely independent. Lines 252–256:

```java
if (this.goals != null) {
    caseHubDefinition.goals.addAll(this.goals);
}
caseHubDefinition.setCompletion(this.completion);
```

Calling `.completion(GoalExpression.allOf(irbDecided))` stores the goal reference inside the `AllOfGoalExpression` only. It does not add anything to `def.getGoals()`. Forget to call `.goals(irbDecided)` and the equivalence test's goal-list assertions pass cleanly — but the completion goal comparison catches it. Forget the completion expression and the reverse happens. Registration in both places with the same `Goal` instances is the correct pattern.

`TrialCoordinationCaseDefinition` is the edge case: no goals, no completion block. The trial has no logical endpoint — it runs for the trial's lifetime and is managed externally. The companion correctly has neither a `.goals()` call nor a `.completion()` call. The builder's `setCompletion(null)` produces null. The test asserts it.
