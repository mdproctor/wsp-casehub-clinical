# Real-Time Event Cascade — Design Spec

**Issue:** casehubio/clinical#161
**Date:** 2026-09-14
**Branch:** issue-161-sse-event-cascade

## Problem

When an adverse event is reported, the platform fires a cascade: SLA timer → SUSAR gate → trust-weighted agent routing → oversight decision → Merkle seal. Today the UI shows the end-state as static data. The orchestration — the platform's core value — is invisible.

## Goal

Users watch the cascade unfold in real time: event reported → SLA assigned → agent selected (with trust score) → agent reasoning → gate decision → trust update → Merkle entry sealed. Each step shows timestamp, what happened, who/what decided, and duration.

## Architecture Overview

```
CDI Events (server)                    WebSocket (push)                      UI (client)
┌─────────────────────┐               ┌──────────────────┐                 ┌──────────────────┐
│ AdverseEventService │               │ EventBroadcaster │                 │ EventConnection  │
│ AeEscalationListener│──broadcast()─→│ (pages-push)     │──WebSocket──→  │ (pages-data)     │
│ SusarGateDecision   │               │                  │                 │                  │
│ SafetyOfficerNotif  │               │ topic:           │                 │ listens:         │
│ LedgerWriters       │               │ clinical/ae/     │                 │ clinical/ae/     │
│ TrustScoreJob       │               │   {aeId}/cascade │                 │   {aeId}/cascade │
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

1. **Server bridge** — CDI event observers call `EventBroadcaster.broadcast()` on a per-AE topic
2. **Push transport** — `casehub-pages-push` handles WebSocket delivery, topic subscriptions, sequence tracking, gap detection, reconnection
3. **UI timeline** — `EventTimelineNode` with `eventChronologyStrategy` renders the cascade with status transitions

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
clinical/ae/{aeId}/cascade
```

One topic per AE. Clients subscribe when an AE is selected in the safety workbench, unsubscribe on deselection.

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
    TRUST_UPDATED,
    LEDGER_SEALED
}
```

### CascadeEvent record (api module)

```java
public record CascadeEvent(
    CascadeStepType step,
    String status,          // "pending", "active", "completed", "failed", "skipped"
    Instant timestamp,
    String actor,           // who/what — "system", agent name, "PI: demo-pi", etc.
    String detail,          // human-readable summary
    Map<String, Object> data // structured payload (trust score, grade, gate decision, etc.)
) {}
```

### Cascade template (grade-aware)

The full cascade path depends on the AE grade and flags:

| Grade | Steps |
|-------|-------|
| Grade 1-2 | AE_REPORTED → SLA_ASSIGNED → SAFETY_OFFICER_NOTIFIED → LEDGER_SEALED |
| Grade 3+ | AE_REPORTED → SLA_ASSIGNED → ESCALATION_CASE_STARTED → AGENT_SELECTED → AGENT_REASONING → AGENT_RESULT → GATE_OPENED → GATE_RESOLVED → SAFETY_OFFICER_NOTIFIED → TRUST_UPDATED → LEDGER_SEALED |
| Grade 3+ unexpected+suspected (SUSAR) | Same as Grade 3+ plus REGULATORY_SUBMISSION_STARTED after GATE_RESOLVED |

`CascadeTemplateResolver` provides the expected step list for a given AE. The timeline renders all expected steps as "pending" initially, then transitions them as events arrive.

## Section 3: Server-Side Push Endpoint

### WebSocket endpoint

`ClinicalPushEndpoint` — a Quarkus `@ServerEndpoint("/ws/push")` that:

1. Implements `SessionSender` (bridges to WebSocket sessions)
2. On `@OnMessage`, parses JSON into `PushRequest` variants, routes to `PushRequestHandler` implementations
3. On `@OnOpen`/`@OnClose`, manages the `TopicRegistry` connection lifecycle

This endpoint is generic — it handles all push topics (cascade events, future data push, etc.). The scenario framework's existing `ScenarioPushHandler` plugs in as a `PushRequestHandler` alongside the new cascade handler.

### ClinicalCascadeBroadcaster

`@ApplicationScoped` service that:

1. Injects `EventBroadcaster`
2. Observes clinical CDI events and broadcasts cascade steps

Each CDI event observer maps to one or more cascade step broadcasts:

| CDI Event | Observer | Cascade Steps |
|-----------|----------|---------------|
| `AdverseEventReportedEvent` | `@ObservesAsync` | AE_REPORTED (completed) + SLA_ASSIGNED (completed) |
| `AeEscalationStartedEvent` (new) | `@ObservesAsync` | ESCALATION_CASE_STARTED (completed) |
| Engine trust routing | Hook in `ClinicalSusarOversightCaseHub` | AGENT_SELECTED (completed, with trust score in data) |
| Agent invocation start | Hook in `ClinicalAgentSupport` | AGENT_REASONING (active) |
| Agent invocation complete | Hook in `ClinicalAgentSupport` | AGENT_RESULT (completed, with agent output) |
| `ActionGateApprovedEvent` / `ActionGateRejectedEvent` | Existing listeners + broadcast | GATE_OPENED → GATE_RESOLVED (completed) |
| Safety officer notification sent | Hook in `SafetyOfficerNotificationListener` | SAFETY_OFFICER_NOTIFIED (completed) |
| Regulatory submission case started | Hook in `RegulatorySubmissionCaseService` | REGULATORY_SUBMISSION_STARTED (completed) |
| Trust score recomputed | Hook in `TrustScoreJob` or `SusarGateDecisionListener` | TRUST_UPDATED (completed, with new score) |
| Ledger entry saved | Hook in `AdverseEventLedgerWriter` | LEDGER_SEALED (completed, with digest) |

### Broadcast pattern

```java
@Inject EventBroadcaster broadcaster;

void broadcastStep(UUID aeId, CascadeStepType step, String actor, String detail, Map<String, Object> data) {
    String topic = "clinical/ae/" + aeId + "/cascade";
    broadcaster.broadcast(topic, new CascadeEvent(
        step, "completed", Instant.now(), actor, detail, data));
}
```

### New CDI event: AeEscalationStartedEvent

The existing `AdverseEventReportedEvent` fires before the escalation case starts. The cascade needs to know when the escalation case is actually created. Add a new `AeEscalationStartedEvent` record in the api module, fired by `AeEscalationCaseService` after `startCase().join()` succeeds.

### Agent execution hooks

`ClinicalAgentSupport.invoke()` (from #160) is the shared utility that calls `agentProvider.invoke()`. Add before/after hooks that broadcast `AGENT_REASONING` (active) before invocation and `AGENT_RESULT` (completed) after. The AE ID must be threaded through — the `ClinicalAgentSupport` API will accept an optional `UUID cascadeAeId` parameter. When non-null, it broadcasts.

## Section 4: Cascade REST Endpoint

### GET /api/adverse-events/{aeId}/cascade

Returns the current cascade state — the template with completed steps filled in. This serves two purposes:

1. **Initial load** — when a user navigates to the Live Cascade tab, the UI fetches the current state before subscribing to the WebSocket topic
2. **Reconnection gap fill** — if the WebSocket reconnects after a gap, the UI can re-fetch full state instead of replaying missed events

Response: `List<CascadeEvent>` — all expected steps with current status.

The endpoint queries:
- `AdverseEvent` entity for grade, flags, timestamps
- Engine case state for escalation/gate status
- Ledger entries for seal status
- Trust service for current scores

This is a read-only projection — no new persistence needed.

## Section 5: Client-Side Integration

### EventConnection subscription

When the user selects an AE in the safety workbench and switches to the "Live Cascade" tab:

1. Fetch `GET /api/adverse-events/{aeId}/cascade` for the current state
2. Create an `EventConnection` to `ws://${host}/ws/push`
3. Call `connection.listen(["clinical/ae/{aeId}/cascade"])`
4. On each `pages-event` CustomEvent, update the corresponding `EventTimelineNode` status

On AE deselection or tab switch: `connection.unlisten()` + `connection.close()`.

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
        status: event.status as EventNodeStatus,
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
  TRUST_UPDATED: 'Trust Score Updated',
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
  TRUST_UPDATED: 'audit',
  LEDGER_SEALED: 'audit',
};
```

### Safety workbench integration

Add a "Live Cascade" tab to the safety workbench (`safety-workbench.ts`). The tab renders a `<clinical-cascade-timeline>` custom web component that:

1. Receives the selected AE ID via attribute (`data-ae-id`)
2. On attribute change: fetches initial state, subscribes to push topic
3. Renders using the existing `event-timeline` pages-viz component with `cascadeTimelineStrategy`
4. Shows connection status indicator (connected/reconnecting/disconnected)

The component follows the existing clinical web component pattern (`ClinicalPiApproval`, `ClinicalSusarGate`, `ClinicalMerkleVerify`) — light DOM, attribute-driven, registered in `index.ts`.

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
# Push event store buffer size (in-memory, per topic)
casehub.pages.push.buffer-size=100
```

No other configuration needed. The push infrastructure is self-contained with sensible defaults.

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
