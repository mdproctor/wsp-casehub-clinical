# Real-Time Event Cascade — Design Spec

**Issue:** casehubio/clinical#161
**Date:** 2026-09-14
**Branch:** issue-161-sse-event-cascade

## Problem

When an adverse event is reported, the platform fires a cascade: SLA timer → SUSAR gate → trust-weighted agent routing → oversight decision → Merkle seal. Today the UI shows the end-state as static data. The orchestration — the platform's core value — is invisible.

## Goal

Users watch the cascade unfold in real time: event reported → SLA assigned → agent selected (with trust score) → agent reasoning → gate decision → Merkle entry sealed. Each step shows timestamp, what happened, who/what decided, and duration.

## Architecture Overview

```
CDI + Vert.x Events (server)           WebSocket (push)                      UI (client)
┌─────────────────────┐               ┌──────────────────┐                 ┌──────────────────┐
│ AdverseEventService │               │ EventBroadcaster │                 │ EventConnection  │
│ AeEscalationListener│──broadcast()─→│ (pages-push)     │──WebSocket──→  │ (pages-data)     │
│ SusarGateDecision   │               │                  │                 │                  │
│ SafetyOfficerNotif  │               │ topic:           │                 │ listens:         │
│ LedgerWriters       │               │ clinical:ae:     │                 │ clinical:ae:     │
│                     │               │   {aeId}:cascade │                 │   {aeId}:cascade │
└─────────────────────┘               └──────────────────┘                 └──────────────────┘
                                                                                    │
                                                                           ┌────────▼─────────┐
                                                                           │ CascadeTimeline  │
                                                                           │ (pages-viz)      │
                                                                           │                  │
                                                                           │ EventTimelineNode│
                                                                           │ pending→active→  │
                                                                           │   completed      │
                                                                           └──────────────────┘
```

Three layers, all reusing existing platform infrastructure:

1. **Server bridge** — CDI event observers and Vert.x event bus consumers call `EventBroadcaster.broadcast()` on a per-AE topic
2. **Push transport** — `casehub-pages-push` handles WebSocket delivery, topic subscriptions, sequence tracking, gap detection, reconnection
3. **UI timeline** — `EventTimelineNode` with `cascadeTimelineStrategy` renders the cascade with status transitions

## Section 1: Dependencies

Add to `runtime/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-push</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-push-runtime</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

`casehub-pages-push-runtime` CDI-produces `EventBroadcaster`, `TopicRegistry`, `EventStore` (in-memory), and `JsonWriter`. The `SessionSender` implementation comes from the WebSocket endpoint (Section 3).

Add Jandex indexing in `application.properties`:

```properties
quarkus.index-dependency.pages-push.group-id=io.casehub
quarkus.index-dependency.pages-push.artifact-id=casehub-pages-push
quarkus.index-dependency.pages-push-runtime.group-id=io.casehub
quarkus.index-dependency.pages-push-runtime.artifact-id=casehub-pages-push-runtime
```

## Section 2: Cascade Event Model

### Topic structure

```
clinical:ae:{aeId}:cascade
```

One topic per AE. Uses `:` segment separator — the platform convention enforced by `TopicRegistry` (trie-based matching) and `topic-matching.ts` (client-side pattern matching). Clients subscribe when an AE is selected in the safety workbench, unsubscribe on deselection.

### CascadeStepType enum (api module)

```java
public enum CascadeStepType {
    AE_REPORTED,
    SLA_ASSIGNED,
    ESCALATION_CASE_STARTED,
    AGENT_SELECTED,
    AGENT_REASONING,
    AGENT_RESULT,
    GATE_OPENED,
    GATE_RESOLVED,
    SAFETY_OFFICER_NOTIFIED,
    REGULATORY_SUBMISSION_STARTED,
    LEDGER_SEALED
}
```

TRUST_UPDATED is excluded — trust score computation is a 24h batch job (`TrustScoreJob`, `@Scheduled(every = "24h")`), not a real-time cascade step. Trust scores are available via the REST endpoint's trust service query but are not part of the live cascade.

### CascadeStepStatus enum (api module)

```java
public enum CascadeStepStatus {
    PENDING,
    ACTIVE,
    COMPLETED,
    FAILED,
    SKIPPED
}
```

### CascadeEvent record (api module)

```java
public record CascadeEvent(
    CascadeStepType step,
    CascadeStepStatus status,
    Instant timestamp,
    String actor,           // who/what — "system", agent name, "PI: demo-pi", etc.
    String detail,          // human-readable summary
    Map<String, Object> data // structured payload (grade, gate decision, etc.)
) {}
```

The `data` payload is capped at 4 KB when serialized. For AGENT_RESULT, include only the agent's structured JSON output (the parsed result), not the raw LLM text response. `ClinicalAgentSupport.invoke()` returns `ClinicalAgentResult<T>` — the `data` field carries the parsed `T` value, not `rawText`.

### Cascade template (grade-aware)

The full cascade path depends on the AE grade and flags:

| Condition | Steps |
|-----------|-------|
| Grade 1-2 | AE_REPORTED → SLA_ASSIGNED → LEDGER_SEALED |
| Grade 3+ (baseline) | AE_REPORTED → SLA_ASSIGNED → ∥ ESCALATION_CASE_STARTED → AGENT_SELECTED → AGENT_REASONING → AGENT_RESULT → LEDGER_SEALED · ∥ SAFETY_OFFICER_NOTIFIED |
| + unexpected (IND reportable) | Adds ∥ REGULATORY_SUBMISSION_STARTED (concurrent at report time) |
| + unexpected + suspected (SUSAR) | Adds GATE_OPENED → GATE_RESOLVED (after AGENT_RESULT, before LEDGER_SEALED) |

**Step concurrency:** Steps marked ∥ fire from independent `@ObservesAsync AdverseEventReportedEvent` observers and execute concurrently. The cascade is a DAG: ESCALATION_CASE_STARTED, SAFETY_OFFICER_NOTIFIED, and REGULATORY_SUBMISSION_STARTED are parallel branches rooted at `AdverseEventReportedEvent`. The template lists them for completeness; the timeline renders concurrent branches visually.

**GATE_OPENED/GATE_RESOLVED scope:** These steps appear ONLY for SUSAR cases (unexpected + suspected). The SUSAR oversight gate is opened by `SusarOversightCaseService` as part of the escalation case's susar-oversight subcase. Non-SUSAR Grade 3+ cases skip these steps entirely.

**GATE_RESOLVED outcomes:** The gate carries one of three outcomes in `CascadeEvent.data`: `approved` (PI confirmed), `rejected` (PI rejected), or `expired` (gate timed out without action via `ActionGateExpiredEvent`). Gate expiry is a legitimate outcome handled by `SusarGateDecisionListener.onExpired()`.

**Gate active phase limitation:** The cascade emits both GATE_OPENED and GATE_RESOLVED simultaneously when the gate decision event fires. The gate's "active" phase — the period between gate creation (WorkItem scheduled by `ActionGateWorkItemHandler`) and PI decision — is not visible in the cascade. Showing the gate as active requires engine-level event plumbing for gate creation, which is out of scope. The REST endpoint partially compensates: if the engine case has an active gate (gate exists but no decision ledger entry), GATE_OPENED shows as completed and GATE_RESOLVED shows as pending — indicating the gate is awaiting decision.

**IND reporting condition:** `REGULATORY_SUBMISSION_STARTED` fires for Grade 3+ unexpected AEs regardless of the `suspected` flag — it represents IND expedited safety reporting (21 CFR 312.32), which is independent of the SUSAR determination. `RegulatorySubmissionCaseService` checks `isIndReportable(grade) && unexpected`, not SUSAR criteria.

**Grade 1-2 rationale:** Grade 1-2 AEs do not fire `AdverseEventReportedEvent` (the CDI event only fires when `engineCaseRequired()` is true — Grade 3+). Grade 1-2 AEs are handled synchronously via WorkItem creation in `AdverseEventService.reportAdverseEvent()`. Safety officer notification is not triggered for Grade 1-2 AEs via the CDI observer path, so SAFETY_OFFICER_NOTIFIED is excluded from the Grade 1-2 template.

`CascadeTemplateResolver` provides the expected step list for a given AE. The timeline renders all expected steps as "pending" initially, then transitions them as events arrive.

**Ordering guarantee:** Events may arrive out of template order due to concurrent `@ObservesAsync` CDI observers and Vert.x event bus consumers. The client renders steps in template position order, matching each event to its template slot by `CascadeStepType`. Arrival order is irrelevant to rendering.

## Section 3: Server-Side Push Endpoint

### WebSocket endpoint

`ClinicalPushEndpoint` — a Quarkus WebSockets Next `@WebSocket(path = "/ws/push")` endpoint, following the reference implementation pattern (`PushWebSocket` in `pages/examples/server/`). Uses `WebSocketConnection` (Vert.x-native, CDI-injectable) rather than Jakarta `@ServerEndpoint` / `Session`:

1. `@OnOpen` — registers the `WebSocketConnection` in a `ConnectionRegistry` (concurrent map keyed by `connection.id()`)
2. `@OnTextMessage` — parses JSON via `PushRequest.parse(message)`, pattern-matches on `Listen`, `Unlisten`, `Subscribe`, `Unsubscribe` variants
3. `@OnClose` — calls `TopicRegistry.removeConnection(connection.id())` and removes from `ConnectionRegistry`

`ClinicalSessionSender` — a separate `@ApplicationScoped` bean implementing `SessionSender` that wraps `ConnectionRegistry`. On `send(connectionId, json)`, looks up the `WebSocketConnection` and calls `connection.sendTextAndAwait(json)`. This satisfies `PushProducers`' `SessionSender` injection point — the endpoint and sender are separate CDI beans.

This endpoint is generic — it handles all push topics (cascade events, future data push, etc.) and all `PushRequest` variants (listen/unlisten for event topics, subscribe/unsubscribe for dataset subscriptions).

### ClinicalCascadeBroadcaster

`@ApplicationScoped` service that:

1. Injects `EventBroadcaster`
2. Observes clinical domain events via two mechanisms:
   - **CDI `@ObservesAsync`** — for events fired via `Event.fireAsync()` (e.g., `AdverseEventReportedEvent`)
   - **Vert.x `@ConsumeEvent`** — for events published via `EventBus.publish()` (e.g., gate decision events)

Gate events (`ActionGateApprovedEvent`, `ActionGateRejectedEvent`, `ActionGateExpiredEvent`) are Vert.x event bus messages, not CDI events. `ActionGateCompletionApplier` fires them via `eventBus.publish()`, delivering to ALL registered consumers. Multiple consumers already coexist on these addresses (`SusarGateDecisionListener` + `SusarAgentAttestationWriter`); the broadcaster adds a third.

**caseId → aeId resolution:** Gate and engine events carry `caseId`, not `aeId`. The broadcaster calls `AdverseEvent.findBySusarOversightCaseId(caseId)` to resolve the AE ID for topic construction — the same pattern used by `SusarGateDecisionListener` and `SusarAgentAttestationWriter`.

**Ownership principle:** `ClinicalCascadeBroadcaster` is the single class responsible for constructing and sending ALL cascade events. No domain service touches `EventBroadcaster` directly for cascade topics. The broadcaster receives triggers via two mechanisms:

1. **Event observation** (CDI `@ObservesAsync` / Vert.x `@ConsumeEvent`) — for domain events that already exist or have standalone architectural value
2. **Typed method calls** — domain services call `broadcaster.notifySafetyOfficer(aeId, ...)` etc. when no domain event exists. The broadcast construction logic lives in the broadcaster; the call site is a one-liner.

This gives centralized completeness (one class owns all 11 step types) and centralized testing (mock `EventBroadcaster`, test the broadcaster class alone for all paths).

| Event / Trigger | Mechanism | Cascade Steps |
|-----------------|-----------|---------------|
| `AdverseEventReportedEvent` | CDI `@ObservesAsync` | AE_REPORTED (completed) + SLA_ASSIGNED (completed) |
| `AeEscalationStartedEvent` (new) | CDI `@ObservesAsync` | ESCALATION_CASE_STARTED (completed) |
| `AeEscalationFailedEvent` (new) | CDI `@ObservesAsync` | ESCALATION_CASE_STARTED (failed) |
| Trust routing decision | `broadcaster.agentSelected(aeId, agentId, trustScore)` called from engine worker resolution (policy from `ClinicalTrustRoutingPolicyProvider`) | AGENT_SELECTED (completed, with trust score in data) |
| Agent invocation start | `broadcaster.agentReasoning(aeId, configKey)` called from `ClinicalAgentSupport.invoke()` when `cascadeAeId` non-null | AGENT_REASONING (active) |
| Agent invocation complete | `broadcaster.agentResult(aeId, configKey, success)` called from `ClinicalAgentSupport.invoke()` when `cascadeAeId` non-null | AGENT_RESULT (completed, with parsed result — not raw LLM text) |
| `ActionGateApprovedEvent` | Vert.x `@ConsumeEvent("casehub.action.gate.approved")` | GATE_OPENED (completed) + GATE_RESOLVED (completed, data: `{decision: "approved", approvedBy: ...}`) |
| `ActionGateRejectedEvent` | Vert.x `@ConsumeEvent("casehub.action.gate.rejected")` | GATE_OPENED (completed) + GATE_RESOLVED (completed, data: `{decision: "rejected"}`) |
| `ActionGateExpiredEvent` | Vert.x `@ConsumeEvent("casehub.action.gate.expired")` | GATE_OPENED (completed) + GATE_RESOLVED (failed, data: `{decision: "expired"}`) |
| Safety officer notified | `broadcaster.safetyOfficerNotified(aeId)` called from `SafetyOfficerNotificationListener` after successful notification | SAFETY_OFFICER_NOTIFIED (completed) |
| Regulatory submission started | `broadcaster.regulatorySubmissionStarted(aeId, caseId)` called from `RegulatorySubmissionCaseService` | REGULATORY_SUBMISSION_STARTED (completed) |
| `CaseLifecycleEvent` (GoalReached / CaseCompleted) | CDI `@ObservesAsync` in broadcaster (parallel to `AeEscalationListener`) | LEDGER_SEALED (completed, with digest from completion ledger entry) |

### Grade 1-2 AE cascade broadcast

Grade 1-2 AEs do not fire `AdverseEventReportedEvent`. All cascade steps complete synchronously within `AdverseEventService.reportAdverseEvent()`. The broadcaster is triggered via a `TransactionSynchronization.afterCompletion()` hook in `AdverseEventService` — consistent with the existing after-commit CDI event pattern for Grade 3+ AEs.

After the transaction commits, broadcast `AE_REPORTED (completed)`, `SLA_ASSIGNED (completed)`, and `LEDGER_SEALED (completed)` on the per-AE cascade topic.

### Failure broadcasting

When `AeEscalationCaseService.onAdverseEventReported()` catches an exception from `startCase()`, it calls `markFailed()`. A new `AeEscalationFailedEvent` CDI event is fired in `markFailed()`, which the broadcaster observes to emit `ESCALATION_CASE_STARTED (failed)`.

**Escalation-dependent steps — skipped on failure:**

| Step | Reason skipped |
|------|---------------|
| AGENT_SELECTED | No engine case → no trust-weighted agent routing |
| AGENT_REASONING | No engine case → no agent invocation |
| AGENT_RESULT | No engine case → no agent invocation |
| GATE_OPENED | No engine case → no SUSAR oversight gate (SUSAR template only) |
| GATE_RESOLVED | No engine case → no gate to resolve (SUSAR template only) |
| LEDGER_SEALED | No `CaseLifecycleEvent` fires → no completion ledger entry |

**Independent steps — NOT skipped (fire from concurrent `@ObservesAsync` observers):**

| Step | Independent observer |
|------|---------------------|
| SAFETY_OFFICER_NOTIFIED | `SafetyOfficerNotificationListener.onAeReported()` — fires from `AdverseEventReportedEvent`, independent of escalation |
| REGULATORY_SUBMISSION_STARTED | `RegulatorySubmissionCaseService.onAdverseEventReported()` — fires from `AdverseEventReportedEvent`, independent of escalation |

The broadcaster emits `skipped` only for the escalation-dependent steps listed above. It does NOT emit `skipped` for SAFETY_OFFICER_NOTIFIED, REGULATORY_SUBMISSION_STARTED, or LEDGER_SEALED from independent branches — those steps transition via their own event sources or remain pending if their conditions aren't met.

### Broadcast pattern

```java
@Inject EventBroadcaster broadcaster;

void broadcastStep(UUID aeId, CascadeStepType step, CascadeStepStatus status,
                   String actor, String detail, Map<String, Object> data) {
    String topic = "clinical:ae:" + aeId + ":cascade";
    broadcaster.broadcast(topic, new CascadeEvent(step, status, Instant.now(), actor, detail, data));
}
```

### New CDI events: AeEscalationStartedEvent, AeEscalationFailedEvent

Two new CDI event records in the api module, both fired by `AeEscalationCaseService`:

```java
public record AeEscalationStartedEvent(
    UUID aeId,
    UUID caseId,
    CtcaeGrade grade,
    String tenantId) {}

public record AeEscalationFailedEvent(
    UUID aeId,
    CtcaeGrade grade,
    String tenantId,
    String errorMessage) {}
```

**AeEscalationStartedEvent** — fired after `startCase()` returns successfully. The existing `AdverseEventReportedEvent` fires before the escalation case starts; this event tells the cascade broadcaster when the case is actually created.

**AeEscalationFailedEvent** — fired inside `AeEscalationCaseService.markFailed()` when `startCase()` throws. The broadcaster observes this to emit `ESCALATION_CASE_STARTED (failed)` and `skipped` for all escalation-dependent steps. Carries `aeId` (for topic construction), `grade` and `tenantId` (for event payload), and `errorMessage` (for the cascade event detail field).

### Agent execution hooks

`ClinicalAgentSupport.invoke()` (from #160) is the shared utility that calls `agentProvider.invoke()`. Add before/after hooks that broadcast `AGENT_REASONING` (active) before invocation and `AGENT_RESULT` (completed) after. The AE ID is threaded through via the `ClinicalAgentRequest` record — add a `UUID cascadeAeId` field:

```java
public record ClinicalAgentRequest<T>(
    String systemPrompt,
    String userPrompt,
    Class<T> responseClass,
    T fallbackValue,
    String configKey,
    String correlationId,
    UUID cascadeAeId   // null for non-cascade invocations
) {}
```

When `cascadeAeId` is non-null, `ClinicalAgentSupport.invoke()` broadcasts AGENT_REASONING before and AGENT_RESULT after invocation. Existing callers pass `null` — records support null fields cleanly and the broadcast hook is a no-op when `cascadeAeId` is null.

## Section 4: Cascade REST Endpoint

### GET /api/adverse-events/{aeId}/cascade

Returns the current cascade state — the template with completed steps filled in. This serves two purposes:

1. **Initial load** — when a user navigates to the Live Cascade tab, the UI fetches the current state before subscribing to the WebSocket topic
2. **Reconnection gap fill** — if the WebSocket reconnects after a gap, the UI can re-fetch full state instead of replaying missed events

Response: `List<CascadeEvent>` — all expected steps with current status.

### State reconstruction rules

The endpoint queries persistent domain state and infers cascade step status. Each step type has a defined reconstruction rule:

| Step | Source | Reconstruction Rule |
|------|--------|-------------------|
| AE_REPORTED | `AdverseEvent.reportedAt` | Non-null → completed |
| SLA_ASSIGNED | `AdverseEvent.slaDeadline` | Non-null → completed |
| ESCALATION_CASE_STARTED | `AdverseEvent.engineCaseId` + `escalationStatus` | `engineCaseId` non-null → completed; `escalationStatus = FAILED` → failed; `escalationStatus = REQUESTED` → active; otherwise → pending |
| AGENT_SELECTED | Engine case progression | If case has passed agent selection phase (gate is open or resolved) → completed; if case is active and in agent phase → active; otherwise → pending |
| AGENT_REASONING | Engine case progression | Same heuristic — if case progressed past agent phase → completed; cannot reconstruct "active" after restart (transient state) |
| AGENT_RESULT | Engine case progression | Same as AGENT_REASONING — inferred from case progression |
| GATE_OPENED | Engine case gate state | If gate exists in case → completed; otherwise → pending |
| GATE_RESOLVED | Engine case gate state + `SusarDecisionLedgerWriter` entries | If gate decision ledger entry exists → completed (data carries decision); otherwise → pending |
| SAFETY_OFFICER_NOTIFIED | `SafetyOfficerNotificationLedgerWriter` entries | Ledger entry exists → completed; skipped entry exists → skipped; otherwise → pending |
| REGULATORY_SUBMISSION_STARTED | `AdverseEvent.regulatorySubmissionCaseId` | Non-null → completed; `regulatorySubmissionStatus = PENDING` → active; otherwise → pending |
| LEDGER_SEALED | `AeEscalationLedgerEntry` with actorRole `AeEscalationCase` (Grade 3+); initial `AdverseEventLedgerEntry` (Grade 1-2) | Grade 3+: completion entry exists → completed. Grade 1-2: initial report entry exists → completed (Grade 1-2 cascade is synchronous — LEDGER_SEALED represents the initial report ledger write). |

**Transient steps after restart:** Agent lifecycle steps (AGENT_SELECTED, AGENT_REASONING, AGENT_RESULT) are transient — they cannot be precisely reconstructed after JVM restart. The endpoint infers completion from engine case progression: if the case has advanced past the agent phase, those steps are marked as completed. If the case is mid-execution, they show as pending (the case will re-emit events when it resumes).

This is a read-only projection — no new persistence needed.

## Section 5: Client-Side Integration

### EventConnection subscription

When the user selects an AE in the safety workbench and switches to the "Live Cascade" tab:

1. Fetch `GET /api/adverse-events/{aeId}/cascade` for the current state
2. Create an `EventConnection` to `ws://${host}/ws/push`
3. Call `connection.listen(["clinical:ae:{aeId}:cascade"])`
4. On each `pages-event` CustomEvent, update the corresponding `EventTimelineNode` status

On AE deselection or tab switch: `connection.unlisten(["clinical:ae:{aeId}:cascade"])` then `connection.close()`.

### CascadeTimelineStrategy

A new `EventTimelineStrategy` implementation in the clinical webui that:

1. Takes the initial cascade state (from REST) as the base node list
2. Merges incoming WebSocket events by matching `CascadeStepType` to node keys
3. Updates node status (pending → active → completed/failed)
4. Enriches node detail with actor, duration, structured data

```typescript
export function cascadeTimelineStrategy(): EventTimelineStrategy<CascadeEvent[]> {
  return {
    toNodes(data: CascadeEvent[]): EventTimelineNode[] {
      return data.map(event => ({
        key: event.step,
        label: STEP_LABELS[event.step],
        status: event.status.toLowerCase() as EventNodeStatus,
        timestamp: event.timestamp,
        actor: event.actor,
        detail: event,
        category: STEP_CATEGORIES[event.step],
      }));
    },
    defaultLayout: 'vertical',
    filterCategories: ['lifecycle', 'agent', 'gate', 'audit'],
  };
}
```

**Merge semantics:** When a WebSocket event arrives, the strategy matches by `CascadeStepType` (the `key`), not by arrival position. The node list preserves template order regardless of event arrival order.

**Status monotonicity:** `COMPLETED`, `FAILED`, and `SKIPPED` are terminal states. Once a step reaches a terminal state, subsequent events for the same `CascadeStepType` are dropped. Valid transitions: `PENDING → ACTIVE → COMPLETED/FAILED/SKIPPED`, `PENDING → COMPLETED/FAILED/SKIPPED`. This prevents nondeterministic arrival order from producing conflicting state — if two concurrent observers both emit events for the same step, the first terminal status wins.

### Step labels and categories

```typescript
const STEP_LABELS: Record<string, string> = {
  AE_REPORTED: 'Adverse Event Reported',
  SLA_ASSIGNED: 'SLA Deadline Assigned',
  ESCALATION_CASE_STARTED: 'Escalation Case Started',
  AGENT_SELECTED: 'Agent Selected',
  AGENT_REASONING: 'Agent Reasoning',
  AGENT_RESULT: 'Agent Result',
  GATE_OPENED: 'Oversight Gate Opened',
  GATE_RESOLVED: 'Oversight Gate Resolved',
  SAFETY_OFFICER_NOTIFIED: 'Safety Officer Notified',
  REGULATORY_SUBMISSION_STARTED: 'IND Submission Started',
  LEDGER_SEALED: 'Merkle Entry Sealed',
};

const STEP_CATEGORIES: Record<string, string> = {
  AE_REPORTED: 'lifecycle',
  SLA_ASSIGNED: 'lifecycle',
  ESCALATION_CASE_STARTED: 'lifecycle',
  AGENT_SELECTED: 'agent',
  AGENT_REASONING: 'agent',
  AGENT_RESULT: 'agent',
  GATE_OPENED: 'gate',
  GATE_RESOLVED: 'gate',
  SAFETY_OFFICER_NOTIFIED: 'lifecycle',
  REGULATORY_SUBMISSION_STARTED: 'lifecycle',
  LEDGER_SEALED: 'audit',
};
```

### Safety workbench integration

Add a "Live Cascade" tab to the safety workbench (`safety-workbench.ts`). The tab renders a `<clinical-cascade-timeline>` custom web component that:

1. Receives the selected AE ID via attribute (`data-ae-id`)
2. On attribute change: fetches initial state, subscribes to push topic
3. Renders using the existing `event-timeline` pages-viz component with `cascadeTimelineStrategy`
4. Shows connection status indicator (connected/reconnecting/disconnected)

The component follows the existing clinical web component pattern (`ClinicalAeGradeHistory`, `ClinicalAeRegrade`, `ClinicalTrustFeedbackDisplay`) — light DOM, attribute-driven, registered in `index.ts` via the `components` array with `customElements.define()`.

## Section 6: Testing

### Unit tests

- `CascadeTemplateResolverTest` — verify correct step list for each grade/flag combination
- `ClinicalCascadeBroadcasterTest` — mock `EventBroadcaster`, fire each CDI event, verify correct topic and payload
- `CascadeEventTest` — serialization/deserialization round-trip

### Integration tests

- `CascadeEndpointTest` (`@QuarkusTest`) — report an AE via `AdverseEventService`, then GET `/api/adverse-events/{aeId}/cascade` and verify the initial steps are present with correct statuses
- `CascadeBroadcastIntegrationTest` — report an AE, verify `EventBroadcaster.broadcast()` is called with the correct topic for each cascade step (use `@InjectMock EventBroadcaster`)

### WebSocket endpoint test

- `ClinicalPushEndpointTest` — connect via WebSocket, send a `listen` request for a topic, broadcast an event, verify the client receives it

### Frontend

- `CascadeTimelineStrategy` unit test — verify `toNodes()` mapping, status transitions, category assignment

## Section 7: Configuration

```properties
# Push event store capacity (in-memory, per topic) — matches PushProducers @ConfigProperty
casehub.pages.push.max-events-per-topic=100
# Topic eviction: remove topic buffers with no subscribers and no activity for this duration
casehub.pages.push.topic-eviction-idle=PT1H
# Maximum serialized size of CascadeEvent.data payload (bytes)
casehub.clinical.cascade.max-data-payload-bytes=4096
```

### Topic eviction

`InMemoryEventStore` creates one `TopicBuffer` per unique topic. With per-AE cascade topics (`clinical:ae:{aeId}:cascade`), the buffer map grows proportionally to total AEs ever reported. To prevent unbounded memory growth:

- A periodic sweep (every 5 minutes) checks each topic buffer for idle time (no `append()` or `replay()` calls) and subscriber count (via `TopicRegistry.connections(topic)`).
- Topics with zero subscribers AND idle time exceeding `topic-eviction-idle` (default: 1 hour) are evicted from the buffer map.
- Evicted topics lose their event history — reconnecting clients use the REST endpoint for full state reconstruction.

This bounds memory to active cascade topics only. A completed cascade with no viewers is evicted after 1 hour; reopening the AE triggers a REST fetch + fresh WebSocket subscription.

### Resilience strategy

Event delivery has three layers of defense:

1. **In-flight delivery** — `EventBroadcaster.broadcast()` sends to all subscribed connections. Per-connection send failures are caught and swallowed (correct for broadcast — one broken connection must not block others).
2. **Sequence gap replay** — `EventConnection` (client) tracks sequence numbers. On reconnection, it requests replay from `InMemoryEventStore` for missed sequence numbers.
3. **Full state reconstruction** — if the event store buffer has rolled past the gap (buffer full or topic evicted), the client falls back to `GET /api/adverse-events/{aeId}/cascade` for full state reconstruction from persistent domain entities.

## Scope Boundaries

**In scope:**
- AE cascade only (not protocol deviations — separate future work)
- Domain-level steps (not engine internals)
- Safety workbench tab integration
- WebSocket push via EventBroadcaster
- REST endpoint for initial state + reconnection

**Out of scope:**
- Protocol deviation cascade (future issue)
- Engine-level event detail (BINDING_ACTIVATED, WORKER_RESOLVED, etc.)
- Guided walkthrough integration (the tab works in both modes already)
- Historical cascade replay (EventStore handles this, but UI replay is future work)
- Push notifications (browser Notification API) for cascade events

## References

- `casehub-pages-push` EventBroadcaster API — `broadcast(topic, payload)` returns sequence number
- `casehub-pages-push-runtime` PushProducers — CDI producers for EventBroadcaster, TopicRegistry, EventStore
- `pages-data/src/dataset/external/sources/event-connection.ts` — WebSocket client with topic subscriptions
- `pages-data/src/sse/sse-manager.ts` — SSE client (not used, but available)
- `pages-viz/src/components/event-timeline-types.ts` — EventTimelineNode, EventNodeStatus, EventTimelineStrategy
- `blocks-timeline/src/strategies/event-chronology.ts` — CaseHubEventType, eventChronologyStrategy, category colors
- `blocks-ui-core/src/types/orchestration.ts` — OrchestrationAuditEvent types
- `AdverseEventService.java` — AE reporting with after-commit CDI event firing
- `ClinicalScenarioActions.java` — scenario actions for demo cascade triggering
- Issue casehubio/clinical#160 — live LLM agent execution (prerequisite, closed)
