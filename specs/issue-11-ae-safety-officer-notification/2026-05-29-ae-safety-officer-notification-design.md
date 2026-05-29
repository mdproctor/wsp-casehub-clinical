# Design: AE Safety Officer Notification — Issue #11

**Date:** 2026-05-29 (revised after spec review)
**Branch:** `issue-11-ae-safety-officer-notification`
**Issue:** casehubio/clinical#11

---

## Summary

When a serious adverse event (Grade ≥ 3) is reported, deliver a notification to the safety officer
via casehub-connectors. Grade 5 (death) notifications carry a `[CRITICAL]` prefix. Delivery
success or failure is written to the tamper-evident ledger (ICH E6(R3) §5.17, 21 CFR 312.32).
Routing is inline entity lookup (mirrors `SponsorNotificationListener`); deployers who need custom
routing override `SafetyOfficerNotifier` directly.

---

## Architecture and Trigger

```
AdverseEventService.reportAdverseEvent()
  → fires AdverseEventReportedEvent (Grade 3+ only — existing event)
  → SafetyOfficerNotificationListener (@ObservesAsync)
      → inline lookup: siteId → TrialSite → ClinicalTrial
      → SafetyOfficerNotifier.notify(SafetyOfficerNotificationRequest)
          → Connector.send(ConnectorMessage)
          → SafetyOfficerNotificationLedgerWriter.writeEntry(...)
```

**Trigger rationale:** `AdverseEventReportedEvent` fires exactly once per AE and only for Grade ≥ 3.
Grade 4/5 AEs create two engine WorkItems (senior-safety-monitor + DSMB) — observing
`WorkItemLifecycleEvent` would send duplicate notifications. `AdverseEventReportedEvent` is the
correct single trigger.

---

## API Types (`api/` module)

### `SafetyOfficerNotifier` — `api/`
Dispatch SPI (parallel to `SponsorNotifier`). Deployers override for custom routing or delivery.

```java
public interface SafetyOfficerNotifier {
    void notify(SafetyOfficerNotificationRequest request);
}
```

### `SafetyOfficerNotificationRequest` — `api/`
Carries everything the notifier needs.

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

No routing policy SPI — the `SafetyOfficerNotifier` SPI is the customisation point for routing.
Deployers who need grade-based channel splitting override `SafetyOfficerNotifier` and use
`request.grade()` to decide.

---

## Runtime Components (`runtime/` module)

### `SafetyOfficerNotificationListener` — `service/`
Observes `AdverseEventReportedEvent`, does inline entity lookup (mirrors `SponsorNotificationListener`
lines 24–39), delegates to notifier.

```java
@ApplicationScoped
public class SafetyOfficerNotificationListener {

    @Inject SafetyOfficerNotifier notifier;

    @Transactional
    public void onAeReported(@ObservesAsync AdverseEventReportedEvent event) {
        if (event.siteId() == null) {
            Log.errorf("AE %s has no siteId — safety officer notification skipped", event.aeId());
            return;
        }
        TrialSite site = TrialSite.findById(event.siteId());
        if (site == null) {
            Log.warnf("TrialSite %s not found — safety officer notification skipped", event.siteId());
            return;
        }
        ClinicalTrial trial = ClinicalTrial.findById(site.trialId);
        if (trial == null) {
            Log.warnf("Trial %s not found — safety officer notification skipped", site.trialId);
            return;
        }
        if (trial.safetyOfficerConnectorId == null || trial.safetyOfficerDestination == null) {
            Log.warnf("Trial %s has incomplete safety officer notification config — skipping",
                site.trialId);
            return;
        }
        notifier.notify(new SafetyOfficerNotificationRequest(
            event.aeId(), event.enrollmentId(), event.siteId(), event.grade(),
            trial.safetyOfficerConnectorId, trial.safetyOfficerDestination));
    }
}
```

**Null siteId guard:** `siteId` can be null when `AdverseEventService.resolveSiteId()` fails
(enrollment not found). Log as error (not warn — a serious AE with no site is a data integrity
problem) and return immediately.

### `DefaultSafetyOfficerNotifier` — `service/`
`@ApplicationScoped @DefaultBean`. Dispatches via `Connector` and writes ledger entry.

- Injects `@All List<Connector>` (see pattern alignment note below)
- Resolves connector by `connectorId`; logs `errorf` and records failed ledger entry if not found
- Message title: `"[CRITICAL] " + grade.label() + " Adverse Event — aeId: " + aeId` for Grade 5;
  `"[" + grade.label() + " AE] — aeId: " + aeId` for Grade 3/4
- Message body: enrollmentId, siteId, grade label, aeId
- Writes ledger entry via `SafetyOfficerNotificationLedgerWriter` on both success and failure
  (mirrors `DefaultSponsorNotifier.recordAttempt()` with `REQUIRES_NEW`)
- No exception on delivery failure — notification is best-effort; ledger records the outcome

**Pattern alignment:** `DefaultSponsorNotifier` currently uses `@Any Instance<Connector>`. This
issue updates it to `@All List<Connector>` (GE-20260526-1653dc: more testable, no CDI context
required for unit tests). Both notifiers must use the same injection pattern.

---

## Ledger Integration

### `SafetyOfficerNotificationLedgerEntry` — `io.casehub.clinical.ledger`
JOINED inheritance on qhorus datasource. Must live in `io.casehub.clinical.ledger` — never in
`io.casehub.clinical.entity`. Mirrors `AeEscalationLedgerEntry`.

Fields:
- `aeId` UUID — subject of the notification
- `enrollmentId` UUID
- `siteId` UUID
- `ctcaeGrade` VARCHAR(50) — `grade.name()`
- `connectorId` VARCHAR(255)
- `destination` VARCHAR(2048)
- `delivered` BOOLEAN — true = connector.send() succeeded
- `notifiedAt` TIMESTAMP WITH TIME ZONE

### `SafetyOfficerNotificationLedgerWriter` — `service/`
Injected into `DefaultSafetyOfficerNotifier`. Mirrors `AeEscalationLedgerWriter`:
`LedgerEntryType.EVENT`, `actorId = "system"`, `actorType = ActorType.SYSTEM`,
`actorRole = "SafetyOfficerNotification"`. `sequenceNumber` from `findLatestBySubjectId(aeId)`.

---

## Data Model

### V114 — `db/migration/default/V114__ae_safety_officer_config.sql`
```sql
ALTER TABLE clinical_trial ADD COLUMN safety_officer_connector_id VARCHAR(255);
ALTER TABLE clinical_trial ADD COLUMN safety_officer_destination  VARCHAR(2048);
```
Both nullable — missing config causes listener to skip with a warn log.

### V1011 — `db/migration/qhorus/V1011__safety_officer_notification_ledger_entry.sql`
```sql
CREATE TABLE safety_officer_notification_ledger_entry (
    id           UUID        NOT NULL,
    ae_id        UUID        NOT NULL,
    enrollment_id UUID       NOT NULL,
    site_id      UUID        NOT NULL,
    ctcae_grade  VARCHAR(50) NOT NULL,
    connector_id VARCHAR(255),
    destination  VARCHAR(2048),
    delivered    BOOLEAN     NOT NULL DEFAULT FALSE,
    notified_at  TIMESTAMP WITH TIME ZONE NOT NULL,
    CONSTRAINT pk_so_notification_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_so_notification_le_ledger FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

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
| `DefaultSafetyOfficerNotifierTest` | `[CRITICAL]` prefix for Grade 5 via `grade.label()`; `[grade.label() AE]` for Grade 3/4; `errorf` logged when connector not found; no exception on delivery failure; ledger entry written on success (delivered=true); ledger entry written on failure (delivered=false) |
| `SafetyOfficerNotificationLedgerWriterTest` | Entry fields set correctly (subjectId=aeId, entryType=EVENT, actorRole, delivered flag); sequence number increments from `findLatestBySubjectId` |

### `@QuarkusTest` integration tests

| Test class | What it covers |
|---|---|
| `SafetyOfficerNotificationListenerTest` | Full CDI wiring; `@InjectMock SafetyOfficerNotifier` (consistent with `SponsorNotificationListenerTest`); fire `AdverseEventReportedEvent` async → verify `notify()` called with correct aeId, grade, connectorId, destination; fire with null siteId → verify `notify()` not called |

**Unit vs integration distinction:**
- Unit tests (`DefaultSafetyOfficerNotifierTest`, `SafetyOfficerNotificationLedgerWriterTest`) cover
  null-handling logic and field-mapping without a CDI container
- `@QuarkusTest` (`SafetyOfficerNotificationListenerTest`) covers CDI wiring, async observer delivery,
  and the null-siteId guard path through the full container

### Out of scope
End-to-end connector delivery (external credentials); DSMB-level escalation notification.

---

## Platform Coherence

- `casehub-connectors-core` already a runtime dep — no new dependency
- `AdverseEventReportedEvent` already exists — no new event type
- `SafetyOfficerNotifier` in `api/` — consistent with `SponsorNotifier`
- `SafetyOfficerNotificationLedgerEntry` in `io.casehub.clinical.ledger` — consistent with all
  other clinical ledger entries; V1011 on qhorus datasource
- `@All List<Connector>` alignment — `DefaultSponsorNotifier` updated in same issue
- No new casehub-work or casehub-engine dependency
