# Commitment Lifecycle Endpoint — qhorus Integration

**Issue:** casehubio/clinical#124
**Branch:** issue-124-commitment-store-stages

## Summary

Update the existing `GET /api/trials/{trialId}/deviations/{devId}/commitment`
endpoint to query qhorus's `CommitmentReader` for commitment state and
`MessageService` for channel messages. Return a response matching the
`CommitmentState` interface expected by the `<commitment-lifecycle>` web component.

## Current State

The endpoint returns a flat `CommitmentLifecycleResponse` built entirely from
`ProtocolDeviation` entity fields:

```java
record CommitmentLifecycleResponse(
    UUID deviationId, String deviationType, String severity,
    String piApprovalStatus, String channelName,
    Instant commandedAt, Instant resolvedAt
)
```

The `<commitment-lifecycle>` component expects:

```typescript
interface CommitmentState {
  id: string;
  currentStage: string;
  stages: Array<{
    key: string;
    actor?: string;
    timestamp?: string;
    status: "completed" | "active" | "pending" | "failed";
  }>;
  messages?: Array<{
    sender: string;
    content: string;
    timestamp: string;
  }>;
}
```

The component fetches from its `endpoint` property, parses the JSON as
`CommitmentState`, and renders a timeline of stages with optional channel
messages below.

## Design

### New service: `CommitmentLifecycleService`

`@ApplicationScoped` service in `io.casehub.clinical.service` that:

1. Takes a `ProtocolDeviation` and `CurrentPrincipal`
2. Queries `CommitmentReader.findByCorrelationId(deviation.id.toString())`
3. Verifies `commitment.tenancyId().equals(principal.tenancyId())` — returns
   empty if mismatch (multi-tenant security)
4. Queries `MessageService.history(channelId, 0L, 100)` via channel lookup
5. Builds and returns a `CommitmentLifecycleResponse` DTO

Inject `CommitmentReader` (the read-only SPI), not `CommitmentStore` — the
endpoint only reads commitment state. Least-privilege principle.

### Stage derivation

The stages array is derived from the commitment snapshot plus the deviation
entity. Two records are required: the `Commitment` (for lifecycle state and
timestamps) and the `ProtocolDeviation` (for `commandedAt` and clinical context).

**Clinical stage model** (4 stages displayed in the timeline):

| Clinical stage | Component key | Meaning |
|---------------|---------------|---------|
| Commanded | `COMMANDED` | PI received formal obligation |
| Acknowledged | `ACKNOWLEDGED` | PI acknowledged receipt |
| Done | `DONE` | PI approved the deviation |
| Declined | `DECLINED` | PI rejected the deviation |

DONE and DECLINED are mutually exclusive terminal stages. The timeline always
shows all 4 nodes — DONE and DECLINED branch from ACKNOWLEDGED.

**Qhorus → Clinical state mapping:**

| Qhorus `CommitmentState` | Clinical stage status | Notes |
|--------------------------|----------------------|-------|
| `OPEN` | COMMANDED=completed, rest=pending | PI obligation issued, awaiting response |
| `ACKNOWLEDGED` | COMMANDED=completed, ACKNOWLEDGED=active, rest=pending | PI confirmed receipt |
| `FULFILLED` | COMMANDED=completed, ACKNOWLEDGED=completed (if `acknowledgedAt` set) or skipped, DONE=completed | PI approved |
| `DECLINED` | COMMANDED=completed, ACKNOWLEDGED=completed (if `acknowledgedAt` set) or skipped, DECLINED=completed | PI rejected |
| `FAILED` | COMMANDED=completed, last-reached stage gets `status: "failed"` | System failure (e.g. DeviationExpirer marks commitment failed) |
| `EXPIRED` | COMMANDED=completed, last-reached stage gets `status: "failed"` | PI missed response deadline |
| `DELEGATED` | COMMANDED=completed, ACKNOWLEDGED=completed (if set), DONE=pending, DECLINED=pending | PI delegated to colleague — child commitment created |

When `acknowledgedAt` is null but the commitment has reached a terminal state
(FULFILLED, DECLINED), ACKNOWLEDGED is skipped — its status is set to
`completed` with no timestamp (the PI went straight to a terminal response).

**Actor attribution:**

- COMMANDED stage: `actor = commitment.requester()` (clinical-service)
- ACKNOWLEDGED stage: `actor = commitment.obligor()` (PI identity)
- DONE/DECLINED stage: `actor = commitment.obligor()` (PI identity)

These are derived from the commitment record's `requester` and `obligor`
fields. Per-transition actor granularity is not available from the commitment
snapshot, but `requester`/`obligor` correctly identify the parties for each
stage in the clinical PI oversight flow.

**Timestamps:**

- COMMANDED: `deviation.commandedAt` (clinical domain timestamp, not `commitment.createdAt`)
- ACKNOWLEDGED: `commitment.acknowledgedAt()` (null if skipped)
- DONE/DECLINED: `commitment.resolvedAt()` (set on all terminal states)

### Channel messages

Fetch via `MessageService.history(channelId, 0L, 100)`. The channel is looked
up via `channelService.findByName(deviation.piCommandChannelName)`.

`history()` excludes EVENT-type messages by default — no explicit filtering
needed.

Each message maps to:

```java
record ChannelMessageResponse(String sender, String content, String timestamp)
```

**Limit:** 100 messages. Clinical PI oversight channels are purpose-built
per-deviation and carry a small number of messages (COMMAND, optional STATUS,
DONE/DECLINE). 100 is well above any realistic conversation length. The
component does not paginate — a fixed limit is acceptable.

### Error handling

| Condition | Behaviour |
|-----------|-----------|
| Deviation not found / wrong trial | HTTP 404 (unchanged) |
| No commitment exists (deviation not yet COMMANDED) | Return response with `currentStage: "PENDING"`, empty stages, no messages |
| Channel not found (GDPR-erased or not yet created) | Return commitment stages but empty messages array |
| Commitment tenancyId mismatch | HTTP 404 (treat as not found — do not leak cross-tenant data) |

### Response DTO

Replace the inner `CommitmentLifecycleResponse` record on `TrialDashboardResource`
with a new top-level record in `io.casehub.clinical.resource` (or nested in the
service — implementation detail):

```java
record CommitmentLifecycleResponse(
    String id,
    String currentStage,
    List<StageResponse> stages,
    List<ChannelMessageResponse> messages
) {
    record StageResponse(String key, String actor, String timestamp, String status) {}
    record ChannelMessageResponse(String sender, String content, String timestamp) {}
}
```

### Endpoint changes

The `getCommitmentLifecycle` method on `TrialDashboardResource` changes from
inline deviation-field reads to a single delegation:

```java
@Inject CommitmentLifecycleService commitmentLifecycleService;

// ... existing validation (deviation lookup, trial ownership check) unchanged ...

return Response.ok(commitmentLifecycleService.buildResponse(deviation, principal)).build();
```

The old `CommitmentLifecycleResponse` inner record is removed.

## Testing

- **Unit test:** `CommitmentLifecycleServiceTest` — mock `CommitmentReader`,
  `MessageService`, `ChannelService`. Test all 7 qhorus state mappings, the
  ACKNOWLEDGED-skip case, tenancy mismatch, missing commitment, missing channel.
- **Integration test:** `CommitmentLifecycleResourceTest` — `@QuarkusTest` hitting
  the endpoint with a seeded deviation + commitment, verifying the JSON shape
  matches `CommitmentState`.

## Out of scope

- Refactoring `TrialDashboardResource` (acknowledged as a God class, but not this issue's concern)
- Moving the endpoint to `DeviationResource` (architectural improvement, separate issue)
- Commitment transition history / event sourcing (qhorus doesn't store this)
- Component changes (the component interface is already correct)
