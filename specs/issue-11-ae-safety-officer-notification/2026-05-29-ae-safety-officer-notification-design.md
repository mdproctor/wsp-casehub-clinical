# Design: AE Safety Officer Notification — Issue #11

**Date:** 2026-05-29
**Branch:** `issue-11-ae-safety-officer-notification`
**Issue:** casehubio/clinical#11

---

## Summary

When a serious adverse event (Grade ≥ 3) is reported, deliver a notification to the safety officer
via casehub-connectors. Grade 5 (death) notifications carry a `[CRITICAL]` prefix. Routing
(connectorId + destination) is SPI-configurable; the default reads from `ClinicalTrial` entity fields.

---

## Architecture and Trigger

```
AdverseEventService.reportAdverseEvent()
  → fires AdverseEventReportedEvent (Grade 3+ only — existing event)
  → SafetyOfficerNotificationListener (@ObservesAsync)
      → SafetyOfficerRoutingPolicy.resolve(aeId, siteId, grade) → SafetyOfficerRoute
      → SafetyOfficerNotifier.notify(SafetyOfficerNotificationRequest)
          → Connector.send(ConnectorMessage)
```

**Trigger rationale:** The issue spec suggests `WorkItemLifecycleEvent`, but Grade 4/5 AEs create
two engine WorkItems (senior-safety-monitor + DSMB). Observing WorkItem creation would fire the
notification twice. `AdverseEventReportedEvent` fires exactly once per AE, already carries
`aeId`, `grade`, `siteId`, and only fires for Grade ≥ 3 (engine-managed cases). No grade filter
needed in the listener — the event already implies Grade ≥ 3.

---

## API Types (`api/` module)

### `SafetyOfficerRoutingPolicy` — `api/spi/`
Policy SPI that deployers implement to control where notifications are routed.

```java
public interface SafetyOfficerRoutingPolicy {
    SafetyOfficerRoute resolve(UUID aeId, UUID siteId, CtcaeGrade grade);
}
```

Returns `null` to decline routing (listener skips notification silently).

### `SafetyOfficerRoute` — `api/spi/`
Routing decision returned by the policy.

```java
public record SafetyOfficerRoute(String connectorId, String destination) {}
```

### `SafetyOfficerNotifier` — `api/`
Dispatch SPI (parallel to `SponsorNotifier`). `@DefaultBean` implementation lives in `runtime/`.

```java
public interface SafetyOfficerNotifier {
    void notify(SafetyOfficerNotificationRequest request);
}
```

### `SafetyOfficerNotificationRequest` — `api/`
Carries everything the notifier needs to compose and dispatch the message.

```java
public record SafetyOfficerNotificationRequest(
    UUID aeId,
    UUID enrollmentId,
    UUID siteId,
    CtcaeGrade grade,
    String connectorId,
    String destination
) {}
```

---

## Runtime Components (`runtime/` module)

### `SafetyOfficerNotificationListener` — `service/`
Observes `AdverseEventReportedEvent`, resolves route via policy, delegates to notifier.

```java
@ApplicationScoped
public class SafetyOfficerNotificationListener {
    @Inject SafetyOfficerRoutingPolicy routingPolicy;
    @Inject SafetyOfficerNotifier notifier;

    @Transactional
    public void onAeReported(@ObservesAsync AdverseEventReportedEvent event) {
        SafetyOfficerRoute route = routingPolicy.resolve(
            event.aeId(), event.siteId(), event.grade());
        if (route == null) return;
        notifier.notify(new SafetyOfficerNotificationRequest(
            event.aeId(), event.enrollmentId(), event.siteId(), event.grade(),
            route.connectorId(), route.destination()));
    }
}
```

### `DefaultSafetyOfficerRoutingPolicy` — `service/`
`@ApplicationScoped @DefaultBean`. Reads from `ClinicalTrial` entity.

- Lookup: `siteId` → `TrialSite.trialId` → `ClinicalTrial` → `safetyOfficerConnectorId` + `safetyOfficerDestination`
- Returns `null` if either field is null or the trial/site is not found — policy declined, listener skips

### `DefaultSafetyOfficerNotifier` — `service/`
`@ApplicationScoped @DefaultBean`. Dispatches via `Connector`.

- Injects `@All List<Connector>` (more testable than `Instance<>` — GE-20260526-1653dc)
- Resolves connector by `connectorId`; logs warning and returns if not found (no exception — notification is best-effort)
- Message title: `[CRITICAL] Grade 5 Adverse Event — ...` for Grade 5; `[Grade X AE] ...` for Grade 3/4
- Message body: includes `aeId`, `enrollmentId`, `siteId`, grade label

---

## Data Model

### V114 migration — `db/migration/default/V114__ae_safety_officer_config.sql`

```sql
ALTER TABLE clinical_trial ADD COLUMN safety_officer_connector_id VARCHAR(255);
ALTER TABLE clinical_trial ADD COLUMN safety_officer_destination VARCHAR(255);
```

Both columns are nullable — a trial without safety officer config produces a `null` route from the
default policy; the listener skips silently.

### `ClinicalTrial` entity additions
```java
public String safetyOfficerConnectorId;
public String safetyOfficerDestination;
```

---

## Testing

### Unit tests (plain JUnit, no CDI)

| Test class | What it covers |
|---|---|
| `DefaultSafetyOfficerRoutingPolicyTest` | Returns correct route when both fields set; returns `null` when connector fields missing; returns `null` when site/trial not found |
| `DefaultSafetyOfficerNotifierTest` | `[CRITICAL]` prefix for Grade 5; `[Grade 3 AE]` prefix for Grade 3; warning logged when connector not found; no exception on delivery failure |

### `@QuarkusTest` integration tests

| Test class | What it covers |
|---|---|
| `SafetyOfficerNotificationListenerTest` | Full CDI wiring; `@Alternative TestSafetyOfficerRoutingPolicy` returns fixed route; `@Alternative TestSafetyOfficerNotifier` captures request; fire `AdverseEventReportedEvent` async, verify request aeId + grade + route |
| `SafetyOfficerRoutingPolicyTest` | Seed `ClinicalTrial` + `TrialSite` with connector config in H2; `resolve()` returns correct route; `null` when fields unset |

### Out of scope
End-to-end connector delivery (external credentials); DSMB-level escalation notification.

---

## Platform Coherence

- `casehub-connectors-core` already a runtime dep — no new dependency
- `AdverseEventReportedEvent` already exists — no new event type
- `SafetyOfficerRoutingPolicy` in `api/spi/` — consistent with `AdverseEventEscalationPolicy`, `IrbCommitteeAssignmentPolicy`
- `SafetyOfficerNotifier` in `api/` — consistent with `SponsorNotifier`
- `DefaultSafetyOfficerNotifier` uses `@All List<Connector>` — consistent with GE-20260526-1653dc
- No new casehub-work or casehub-engine dependency
