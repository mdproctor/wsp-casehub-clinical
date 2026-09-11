# Decisions — #160 Live LLM Agent Execution

## D1: Branch scope

**Choice:** Full epic — bootstrap + all 4 agents (eligibility, safety, DSMB, trial supervision)
**Alternatives:**
- Bootstrap + first 2 agents — proves pattern but defers DSMB and trial supervision
- Bootstrap only — validates Vertex integration but no new agents
**Rationale:** The user wants a real live system working. All 4 agents follow the same pattern; the design work is front-loaded but implementation is repetitive.
**Trade-offs:** Larger branch, longer implementation cycle
**Sources:** Epic #160 issue body, issue-86 spec (reference pattern)
**Exploration:** quick
**Status:** captured

## D2: Model and backend configurability

**Choice:** Configurable per capability via application.properties, defaulting to Sonnet
**Alternatives:**
- Hardcoded Sonnet — simpler but inflexible
- Opus for safety agents, Sonnet for others — premature optimization
**Rationale:** User requirement. Each agent reads `casehub.clinical.agent.<capability>.model`. Backend already configurable via `casehub.platform.agent.default-backend`.
**Trade-offs:** Slightly more config surface area
**Sources:** AgentSessionConfig.model field, RoutingAgentProvider backend discovery
**Exploration:** quick
**Status:** captured

## D3: DSMB and trial supervision integration

**Choice:** Augment existing flows — LLM analysis as a step inside existing scheduler/binding
**Alternatives:**
- New capability bindings — cleaner engine integration but requires YAML changes
- Standalone services — simpler but bypasses trust routing and oversight gates
**Rationale:** Backward compatible. Rule-based detection runs first, LLM augments. Trial supervision gets a new capability binding alongside existing humanTask.
**Trade-offs:** LLM analysis not engine-managed for DSMB; coupled to scheduler lifecycle
**Sources:** TrialSafetyAggregationJob, trial-coordination.yaml
**Exploration:** quick
**Status:** captured

## D4: Structured output strategy

**Choice:** Shared ClinicalAgentSupport utility for prompt construction and JSON parsing
**Alternatives:**
- Per-agent manual parsing — no shared code but repetitive across 5 agents
- Schema-driven tool use — most reliable but requires AgentProvider tool_use support
**Rationale:** Avoids duplicating parse/fallback logic. Each agent defines its response record. Utility handles JSON schema instruction injection, Jackson parsing, fallback.
**Trade-offs:** Adds one shared class; agents coupled to its conventions
**Sources:** LlmProtocolAmendmentAdvisor manual JSON parsing pattern
**Exploration:** quick
**Status:** captured

## D5: Audit trail integration

**Choice:** Same ledger writers, agent output stored in case context
**Alternatives:**
- New AgentDecisionLedgerEntry subclass — richer AI transparency but new migration + writer
**Rationale:** Agent responses (reasoning + recommendation) stored in case context, captured in existing ledger entries' domain content. ComplianceSupplement attached as before. No new ledger infrastructure needed.
**Trade-offs:** No dedicated per-invocation token/latency tracking in the ledger
**Sources:** SusarAgentAttestationWriter, ClinicalComplianceSupplement, LedgerEntry.attach()
**Exploration:** quick
**Status:** captured
