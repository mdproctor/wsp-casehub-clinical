---
layout: post
title: "When the Agents Stop Being Stubs"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [casehubio/clinical]
tags: [agents, llm, clinical-trials, casehub, compliance, spi]
series: issue-160-live-llm-agent-execution
---

# When the Agents Stop Being Stubs

Every agent in casehub-clinical was a lie. The harness had trust routing, oversight gates, Merkle audit trails — the full compliance apparatus. But the agents making the clinical decisions? Pre-seeded by `DemoDataSeeder`. No LLM ever ran. `NoOpAgentProvider` was the only provider on the classpath.

The gap was structural, not aspirational. `AgentProvider` existed. `ClaudeAgentProvider` existed. The wiring from clinical#86 proved the pattern worked for protocol amendment advice. What didn't exist was activating real agents across all five clinical decision points — eligibility screening, SUSAR assessment, DSMB signal analysis, trial supervision, and the already-wired amendment advisor.

The design decision that shaped everything: per-agent SPI displacement. Each agent gets an interface in `api/`, a `@DefaultBean` stub in `runtime/` that returns a safe default, and an `@ApplicationScoped` LLM implementation that displaces the stub automatically via CDI priority. A shared `ClinicalAgentSupport` utility handles the mechanics — prompt building, Mutiny stream collection, JSON fence extraction, Jackson parsing, and conservative fallback when the LLM is unavailable.

The fallback design is where domain knowledge meets engineering. Each agent's failure mode is different because the clinical consequences are different. Eligibility screening falls back to MARGINAL — borderline patients get IRB consultation rather than silent exclusion. SUSAR assessment falls back to `susarRequired=true` — a false escalation costs a safety officer's review time, but a false suppression risks patient safety. DSMB analysis falls back to FLAG_FOR_REVIEW — rule-based signals still fire, the LLM just can't add narrative enrichment. Trial supervision falls back to REVIEW_REQUIRED — operations teams get a manual review prompt.

The existing `SusarCriteriaEvaluator` was the most interesting displacement. It's a rule-based evaluator — Grade 4/5, unexpected, suspected causal relationship. The LLM replacement adds causality reasoning that rules can't express: temporal relationship to drug administration, dose-response patterns, dechallenge/rechallenge history, known drug class effects. But the Grade 3 scope boundary stays locked — that's deferred to clinical#76, and the LLM doesn't get to expand regulatory scope unilaterally.

`ClinicalTrialCaseHub` gained something it didn't have before: a worker. The trial coordination case definition previously only had a DSMB rollup humanTask binding. Now it has an `augment()` override registering a trial-supervision worker via `Worker.builder().function()`, and a new YAML capability binding that fires when safety metrics update. The LLM assesses operational health — enrollment trajectory anomalies, protocol adherence degradation, site performance outliers — and the output flows through the same engine case context as every other trial-level signal.

What this opens up: the harness now demonstrates that GCP, FDA, and GDPR requirements are structurally satisfied by CaseHub's accountability layer while real LLM agents make real clinical decisions. The compliance supplements carry `InvocationMetrics` — model, token counts, cost, duration — serialized into the existing `detail` field on every ledger entry. EU AI Act Art.12 record-keeping is satisfied without a schema migration, without a new `LedgerEntry` subclass, without touching platform-owned types.

The agents are no longer stubs. The audit trail knows it.
