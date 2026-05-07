# Handoff — casehub-clinical Bootstrap
2026-05-07

## What this project is

`casehub-clinical` is the clinical trial coordination application — CaseHub's market entry demonstration for regulated healthcare. It shows that GCP, FDA IND, EMA CTR, and GDPR requirements cannot be met by workflow-based LLM coordination and are structurally satisfied by CaseHub's foundation.

Clinical trials scored 24/25 on market fit — highest of all evaluated use cases. The compliance requirements (GCP, FDA, GDPR for patient data) create hard constraints that ClinicalAgent (arXiv 2404.14777, ACM BCB '24) structurally cannot meet. Positioned as **market entry showcase** (not primary tutorial — requires GCP domain knowledge to follow).

Full analysis: `https://raw.githubusercontent.com/casehubio/parent/main/docs/use-case-analysis.md` §8.1
Showcase scenario: `https://raw.githubusercontent.com/casehubio/parent/main/docs/tutorial-strategy.md` §7

## Current state

Greenfield repo — no code yet. CLAUDE.md written. GitHub repo at `casehubio/clinical`.

## Foundation status relevant to clinical trials

| Foundation capability | Status |
|----------------------|--------|
| Sub-case orchestration (per-site sub-cases) | ✅ DONE (engine#195) |
| CasePlanModel, stage gating, goals | ✅ Production |
| WorkItem with SLA + claimDeadline (adverse event 24h/7d) | ✅ Production |
| EscalationPolicy — auto-escalate on SLA miss | ✅ Production |
| qhorus COMMAND/RESPONSE (PI authorisation commitment) | ✅ Production |
| Commitment lifecycle — PI formally named as obligor | ✅ Production |
| LedgerErasureService — GDPR Art.17 patient data | ✅ Production |
| CaseLedgerEntry — FDA Merkle audit trail | ✅ DONE (2026-04-26) |
| EU AI Act Art.12 ComplianceSupplement | ✅ Production |
| W3C PROV-DM — regulatory decision lineage | ✅ Production |
| HITL WorkItem COMPLETED → case plan signal | ⚠️ Pending (casehub-work-adapter) — critical gap |
| TrustWeightedSelectionStrategy wired | ⚠️ Pending (P1.3) |
| LlmPlanningStrategy (protocol amendment supervisor) | ⚠️ Pending (engine#102) |

## Most important missing foundation piece

**HITL wiring** — `casehub-work-adapter`: WorkItem `COMPLETED` must signal the case plan item transition from WAITING → active. Without this, the IRB approval gate (the most important human-in-the-loop scenario) cannot complete. This is a casehub-work fix — raise an issue there, don't work around it in casehub-clinical.

## What to build first

### Multi-site sub-case structure (start here — foundation ready)

Sub-case orchestration is already production-ready (engine#195) and is the clearest demonstration of what ClinicalAgent cannot do:

```
Trial case (parent)
├── Site A sub-case  → enrollment + safety monitoring + deviation bindings
├── Site B sub-case  → ...
└── Trial-level DSMB rollup binding
    fires when: safety signal from ≥ 2 sites simultaneously
```

### Adverse event escalation (most compliance-critical)

Grade ≥ 3 adverse event → WorkItem with 24h `claimDeadline` (GCP SLA). Auto-escalation to DSMB on miss. This single feature demonstrates more regulatory value than ClinicalAgent's entire codebase.

### PI authorisation commitment (normative layer showcase)

Protocol deviation → COMMAND to PI → formal Commitment (OPEN → ACKNOWLEDGED → FULFILLED). The PI's commitment is what the FDA reads — not an audit log, a formal accountable record.

## 3-Site Showcase Scenario

- **Site A:** Eligibility screening. Marginal criterion → IRB consultation WorkItem (72h SLA).
- **Site B:** Grade 3 adverse event → automatic 24h safety escalation, DSMB notified.
- **Site C:** Protocol amendment → LLM supervisor reads context from all three sites, recommends proceed/halt.

FDA can verify the complete decision chain for every patient at every site via Merkle proof — without server access.

## Key design decisions before writing code

1. **Protocol format:** Simplified YAML eligibility criteria and arms — enough to drive binding conditions without full CDISC compliance.
2. **Patient identity:** Use pseudonymisation SPI from day one — don't retrofit GDPR Art.17.
3. **CTCAE grading:** Grade ≥ 3 triggers 24h SLA. Grade 5 (death) triggers 1h SLA. Model explicitly.
4. **Site scoping:** Each site is a sub-case. Start with 2 sites (minimum for cross-site rollup demo).

## References

- Use case analysis (§8.1): `https://raw.githubusercontent.com/casehubio/parent/main/docs/use-case-analysis.md`
- Showcase scenario (§7): `https://raw.githubusercontent.com/casehubio/parent/main/docs/tutorial-strategy.md`
- ClinicalAgent baseline: https://github.com/LeoYML/clinical-agent (arXiv 2404.14777)
- ICH E6(R3) — Good Clinical Practice guideline
- GitHub epics: casehubio/clinical issues labelled `epic`
