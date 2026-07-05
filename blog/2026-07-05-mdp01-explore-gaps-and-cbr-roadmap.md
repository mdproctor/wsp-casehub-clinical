---
layout: post
title: "Explore Mode Gap Closure + CBR Roadmap"
date: 2026-07-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, explore-mode, neocortex, field-alignment]
---

## Explore Mode Gap Closure + CBR Roadmap

Started the session picking up three issues — #114 (trust-network.ts field mismatches), #113 (AddSiteRequest missing targetEnrollment), and #101 (explore mode dashboards). What I expected to be a quick field-rename exercise turned into something more interesting.

**The schema that never was.** All six explore pages were already written and wired into navigation. The TypeScript referenced field names like `siteName`, `patientId`, `eventType`, `commitmentState`, `digest` — a clean, human-readable API. The Java records returned `siteId` (raw UUID), `enrollmentId` (wrong entity), `type` (always null), and no commitment or digest fields at all. The TS was written against the API the UI *wanted*; the Java was written against what the database *had*. Eleven mismatches across four of six pages, not the two that #114 reported.

The fix isn't "rename the TS to match Java" — that would leave the UI showing UUIDs and empty columns. The right answer is to enrich the Java records: add `siteName` from TrialSite.investigatorId via the join that already exists, add `patientId` from PatientEnrollment, project `commandedAt` as `reportedAt` (same instant, different semantic), and pull `irbDecision` through from IrbApproval. Two type fixes on AgentRow: `endorsementRatio` from pre-formatted `"75%"` string back to a raw Double (the API should return data, the UI should format), and `maturityPhase` from an opaque int to a descriptive string.

The `eventType` gap was real — the AdverseEvent entity had grade (severity) but no event type (what happened). CTCAE v5.0 has both Grade and Preferred Term. Since there are no installs, I'm adding the field to the entity and modifying the original migration. Spec written and committed to workspace main.

**CBR: from clinical use cases to platform roadmap.** The second half of the session was different — scoping what neocortex's memory/CBR layer needs to support clinical trial use cases. I started from the domain: what would a PI actually want to see when reviewing a deviation? What would a safety officer want when a Grade 3 AE arrives? What does the DSMB need for cross-site signal detection?

I read the neocortex memory-api surface to understand what already exists. The `CbrCaseMemoryStore` SPI is well-designed — typed cases (`FeatureVectorCbrCase`, `PlanCbrCase`), feature schemas (categorical/numeric/text), query model with topK, temporal filtering, min similarity threshold. The in-memory implementation handles exact-match filtering. But `retrieveSimilar()` returns score=1.0 for everything — no actual similarity computation, no semantic retrieval on the problem text, no outcome feedback loop, no plan adaptation.

Seven phases emerged, each independently useful in clinical:

1. **Clinical CBR baseline** — use existing SPI as-is for structured precedent lookup
2. **Similarity scoring** — weighted distance across feature fields
3. **Semantic retrieval** — bridge CBR with existing RAG (dense+sparse, the infrastructure is already there)
4. **Outcome learning + traceability** — close the feedback loop AND make it auditable
5. **Plan adaptation** — reuse past case plans with worker substitution and priority adjustment
6. **Temporal trajectories** — "this AE progression matches 3 past cases that escalated"
7. **Scoping + lifecycle** — patient/site/trial hierarchy with GDPR-aware erasure and trust-weighted retention

Filed as epics: clinical #115 (6 children), neocortex #81 (6 children). Each neocortex phase unlocks the corresponding clinical phase, but Phase 1 needs zero neocortex changes — clinical can start immediately with exact-match filtering.

The part I keep thinking about is Phase 6 — trajectory matching. Not "find a similar state" but "find a similar trajectory." A Grade 1 AE that escalated to Grade 3 in 48 hours is a fundamentally different signal than a Grade 3 at presentation. DTW over feature snapshots would catch this. It's the most powerful thing CBR could do for clinical safety, and it's not something you get from any amount of prompt engineering.
