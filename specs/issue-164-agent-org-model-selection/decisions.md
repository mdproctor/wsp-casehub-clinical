# Design Decisions — #164 Agent Organization + Model Selection

## D1: UI layout approach

**Choice:** Convert Safety Workbench and other views to dockWorkbench
**Alternatives:**
- Keep tabs, add new components as additional tabs — minimal change but doesn't unlock the panel ecosystem (registerPanel/hostPanel pattern)
- Hybrid — dockWorkbench for new Ops view only, keep existing layouts — incremental adoption but inconsistent UX across views
**Rationale:** Aligns with fsitrading's pattern, unlocks the full blocks-ui panel ecosystem (registerPanel, hostPanel, dock zones), makes future blocks-ui components drop-in. One-time upfront cost.
**Trade-offs:** Larger initial change to existing UI code; all existing page layouts need rewriting from tree/tabs/columns to dockWorkbench zones.
**Sources:** fsitrading `site.ts` (ops-centre, trading-desk layouts), clinical `app.ts`/`safety-workbench.ts`/`operations.ts` (current layout)
**Exploration:** quick
**Status:** captured

## D2: Org structure hierarchy model

**Choice:** Functional teams — Safety Team, Site Teams, Regulatory Team, Trial Supervisor at top
**Alternatives:**
- Per-case-type teams — group by YAML case definition (trial-coordination team, ae-escalation team, etc.). Mirrors code structure but doesn't reflect real clinical organisation
- Flat with supervision edges only — no teams, just supervision relationships between individual agents. Maximum flexibility but loses org-diagram visualisation value
**Rationale:** Maps to how clinical trials are actually organised. Escalation chains (site coordinator → PI → safety officer → sponsor) are natural supervision relationships. The functional grouping gives org-diagram clear visual hierarchy.
**Trade-offs:** Team boundaries are a design decision that may need adjustment as more agents/capabilities are added.
**Sources:** ICH E6(R3) GCP roles (PI, sponsor, safety officer), clinical domain model (ClinicalTrial, TrialSite, PatientEnrollment)
**Exploration:** quick
**Status:** captured

## D3: Model tier mapping

**Choice:** All FLAGSHIP except eligibility-screening (STANDARD)
**Alternatives:**
- Split FLAGSHIP / STANDARD / FAST — safety-monitoring and susar-criteria FLAGSHIP, protocol-amendment and trial-supervision STANDARD, eligibility-screening FAST. More cost-optimised but may under-power regulatory reasoning.
- All FLAGSHIP — every clinical agent is a regulated decision, no cost savings worth the risk. Simplest but highest cost.
**Rationale:** Safety-monitoring, susar-criteria, protocol-amendment-advisor, and trial-supervision all involve life-safety or regulatory reasoning where model capability directly affects compliance. Eligibility-screening is structured criteria matching against protocol I/E criteria — less interpretive, STANDARD is sufficient.
**Trade-offs:** Slightly higher cost than a split mapping, but clinical trials are the domain where under-powering an AI decision has the highest regulatory consequence.
**Sources:** Platform `ModelTier` enum (FLAGSHIP, STANDARD, FAST, EMBEDDING), eidos `ModelTierTerm` vocabulary
**Exploration:** quick
**Status:** captured

## D4: Narrative signal scope

**Choice:** All 7 case types — emit narrative signals for every case definition
**Alternatives:**
- Only capability-bound cases (3: trial-coordination, susar-oversight, protocol-amendment) — simpler, but loses visibility into human-in-the-loop gates (IRB, PI approval, IND filing)
**Rationale:** The narrative value is seeing the full orchestration cascade including human gates. IRB consultations, PI authorisations, IND filings, and AE escalation reviews are all part of the governed decision trail. Excluding them loses the "platform governs everything" story.
**Trade-offs:** More signal volume — humanTask completions generate step outcomes even though no LLM agent was involved. The narrative pipeline handles this gracefully (step outcome with workerId = human actor).
**Sources:** Clinical YAML case definitions (7 files), fsitrading `FsiNarrativeSignalStrategy` (filters for 1 case type)
**Exploration:** quick
**Status:** captured

## D5: Narrative data flow

**Choice:** REST endpoint for historical state + WebSocket push for real-time signals
**Alternatives:**
- WebSocket only — no REST endpoint, component builds state client-side from signal events. Simpler server but requires client-side state accumulation and loses state on reconnect.
- REST only, poll-based — just GET endpoint, component polls on interval. Simplest but loses real-time cascade visibility.
**Rationale:** Clinical already has WebSocket push infrastructure (EventBroadcaster → TopicRegistry → `/ws/push`) used by ClinicalCascadeBroadcaster for AE cascade events. Narrative signals push over the same infrastructure on topic `clinical:narrative:{caseId}`. REST provides initial/historical NarrativeState for the narrative-timeline component's DataSourceMixin. Same pattern as existing cascade events.
**Trade-offs:** Two data paths (REST + WS) to maintain, but both are thin wrappers around the same DecisionNarrativePipeline state.
**Sources:** `ClinicalCascadeBroadcaster.java` (existing WS push pattern), `ClinicalPushEndpoint.java` (`/ws/push`), blocks-ui `narrative-timeline` component (DataSourceMixin expects REST endpoint)
**Exploration:** quick
**Status:** captured
