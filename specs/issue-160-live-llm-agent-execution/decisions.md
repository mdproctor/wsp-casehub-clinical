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

**Choice:** Shared ClinicalAgentSupport utility with Jackson ObjectMapper deserialization
**Alternatives:**
- Per-agent manual parsing — no shared code but repetitive across 5 agents
- Schema-driven tool use — platform has ToolCallComplete events but AgentSessionConfig has no `tools` field; only `mcpServers`. Using MCP for structured output is over-engineered.
- Manual extractJsonValue() string manipulation — fragile, fails on escaped quotes, nested objects, markdown-wrapped JSON
**Rationale:** Jackson `ObjectMapper.readValue(rawJson, responseType)` is robust, handles all JSON variants, and the codebase already uses Jackson extensively. Utility handles: JSON block extraction from markdown fences, Jackson deserialization to typed response records, per-agent fallback on parse failure. Refactor existing LlmProtocolAmendmentAdvisor to use the utility (replacing manual extractJsonValue).
**Trade-offs:** Adds one shared class; agents coupled to its conventions
**Sources:** LlmProtocolAmendmentAdvisor.extractJsonValue() (fragile pattern to replace), Jackson ObjectMapper usage in TrialSafetyAggregationJob
**Exploration:** quick
**Status:** revised (R1-02: replaced manual string parsing with Jackson deserialization)

## D5: Audit trail integration

**Choice:** Same ledger writers + InvocationComplete metrics captured in ComplianceSupplement
**Alternatives:**
- New AgentDecisionLedgerEntry subclass — richer but requires new migration, writer, and persistence unit wiring
- Same ledger writers without invocation metrics — original choice; leaves EU AI Act Art.12 compliance gap
**Rationale:** AgentEvent.InvocationComplete delivers per-invocation: model, inputTokens, outputTokens, thinkingTokens, totalCostUsd, durationMs, sessionId. ClinicalComplianceSupplement gains fields for these metrics. Existing ledger writers attach the enriched supplement. No new LedgerEntry subclass or migration needed — the supplement is a JSON blob inside the existing entry.
**Trade-offs:** Supplement grows larger; no dedicated queryable table for invocation metrics (acceptable for audit, not for analytics)
**Sources:** AgentEvent.InvocationComplete, ClinicalComplianceSupplement, EU AI Act Art.12 record-keeping requirements
**Exploration:** quick
**Status:** revised (R1-03: capture InvocationComplete metrics for Art.12 compliance)

## D6: Agent architecture pattern

**Choice:** Per-agent SPI implementations following the existing ProtocolAmendmentAdvisor pattern
**Alternatives:**
- Generic ClinicalAgent<I,O> base class — DRY but premature abstraction over 4 different domain problems
**Rationale:** Proven pattern. Each agent gets its own SPI interface + @DefaultBean stub + @ApplicationScoped LLM implementation. ClinicalAgentSupport utility handles common concerns (prompt building, JSON parsing, config lookup, fallback). No base class hierarchy. Each agent's domain contract stays explicit and testable independently.
**Trade-offs:** Some boilerplate across 4 implementations (mitigated by shared utility)
**Depends on:** D4 (shared utility handles mechanical commonality)
**Sources:** LlmProtocolAmendmentAdvisor, DefaultProtocolAmendmentAdvisor, SusarCriteriaEvaluator @DefaultBean displacement pattern
**Exploration:** quick
**Status:** captured

## D7: Per-agent fallback policy on LLM failure

**Choice:** Each agent defines a domain-appropriate conservative fallback, not a universal PROCEED
**Alternatives:**
- Universal PROCEED default — inherited from protocol amendment advisor; wrong for safety-critical agents
**Rationale:** Defaulting to "everything is fine" when a safety-critical LLM fails is a patient safety risk. Each agent's fallback must escalate to human review, not suppress the concern.
**Fallback table:**

| Agent | Fallback on failure | Effect |
|-------|-------------------|--------|
| Protocol amendment | PROCEED | Correct — failed advisor should not block amendments |
| Eligibility screening | MARGINAL | Triggers IRB consultation gate, not silent enrollment |
| Safety monitoring | SUSAR_REQUIRED=true | Escalates to human safety officer review |
| DSMB analysis | FLAG_FOR_REVIEW | Adds signal to DSMB WorkItem for human review |
| Trial supervision | REVIEW_REQUIRED | Creates WorkItem for PI/operations review |

**Trade-offs:** Conservative fallbacks may create false escalations, increasing human review burden when the LLM is unavailable
**Depends on:** D6 (each SPI implementation defines its own fallback)
**Sources:** ICH E6(R3) §5.17 (safety reporting obligations), R1-09 decision review finding
**Exploration:** quick
**Status:** captured
