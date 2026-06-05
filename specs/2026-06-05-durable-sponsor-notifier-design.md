# DurableSponsorNotifier — Design Spec

**Issue:** casehubio/clinical#21  
**Branch:** issue-21-durable-sponsor-notifier  
**Date:** 2026-06-05

---

## Context

`DefaultSponsorNotifier` delivers sponsor notifications fire-and-forget via connector. It satisfies the GCP prompt-notification requirement for the showcase but has no persistence, no retry, and no audit trail beyond a single deviation ledger entry. `DurableSponsorNotifier` replaces it for production-grade regulated deployments: every notification attempt is persisted, retried on failure, and independently audited.

`DefaultSponsorNotifier` is deleted in this branch. `DurableSponsorNotifier` is the sole `SponsorNotifier` implementation. The fire-and-forget case is expressed as a preference setting (`maxAttempts = 1`), not a separate class.

---

## Architecture

```
SponsorNotificationListener.onDeviationResolved()
    └─ DurableSponsorNotifier.notify()
           └─ SponsorNotificationStore.createPending()  [REQUIRES_NEW]

SponsorNotificationRetryJob  (@Scheduled, not @Transactional)
    └─ for each eligible notification:
           SponsorNotificationDeliveryService.attemptDelivery(id)
               Phase 1: store.load(id)                  [REQUIRED, short tx]
               Phase 2: connector.send(message)          [no transaction]
               Phase 3: store.markDelivered/Failed/Exhausted()  [REQUIRES_NEW]
                             └─ SponsorNotificationLedgerWriter   [REQUIRED joins store's tx]
                      + DeviationLedgerWriter.write*()    [own REQUIRES_NEW, called after]
                      + Event.fireAsync(Failed) if EXHAUSTED
```

**CDI activation:** `DurableSponsorNotifier` is `@ApplicationScoped` (non-`@DefaultBean`), displacing `DefaultSponsorNotifier`. No explicit `selected-alternatives` entry needed.

---

## `SponsorNotification` entity

Package: `io.casehub.clinical.entity`  
Datasource: default  
Migration: **V115** (`db/migration/default/V115__sponsor_notification.sql`)

```java
@Entity @Table(name = "sponsor_notification")
public class SponsorNotification extends PanacheEntityBase {
    @Id public UUID id;

    // Subject references — no FK constraints (cross-datasource)
    public UUID deviationId;
    public UUID trialId;
    public UUID siteId;

    @Enumerated(EnumType.STRING)
    public SponsorNotificationStatus status;  // PENDING | FAILED | DELIVERED | EXHAUSTED

    public int     attempts;
    public Instant createdAt;
    public Instant lastAttemptedAt;
    public Instant nextRetryAfter;   // null = eligible immediately
    public Instant deliveredAt;
    public String  failureReason;    // last failure message, overwritten per attempt

    // Payload snapshot — resolved at notify() time, used verbatim on retry
    public String            piId;
    public String            piDisplayName;
    public String            connectorId;
    public String            destination;
    @Enumerated(EnumType.STRING)
    public DeviationSeverity severity;
    @Enumerated(EnumType.STRING)
    public PiApprovalStatus  terminalStatus;
    public String            deviationType;
}
```

`SponsorNotificationStatus` enum in `api/model/`: `PENDING`, `FAILED`, `DELIVERED`, `EXHAUSTED`.

**Payload is snapshotted at `notify()` time** because: `piDisplayName` is resolved once by `PiIdentityResolver` (directory may differ on retry); connector config reflects where notification was originally directed (trial config may change mid-retry window).

---

## `DurableSponsorNotifier`

Package: `io.casehub.clinical.service`

```java
@ApplicationScoped
public class DurableSponsorNotifier implements SponsorNotifier {
    @Inject SponsorNotificationStore store;

    @Override
    public void notify(SponsorNotificationRequest request) {
        store.createPending(request);
    }
}
```

`notify()` does one thing: delegate to the store. No connector logic, no transaction annotation. The listener's skip paths (site-not-found, no-config, PI-resolver-failed) continue to bypass `notify()` entirely and write directly to the deviation ledger — unchanged.

---

## `SponsorNotificationStore`

Package-private `@ApplicationScoped` bean. Owns all `SponsorNotification` entity mutations and calls `SponsorNotificationLedgerWriter` from within its outcome-recording methods — making entity update and notification ledger write atomic. The writer's `@Transactional(REQUIRED)` joins the store's surrounding `REQUIRES_NEW` rather than starting a new transaction.

**Key methods:**

| Method | Transaction | Purpose |
|--------|-------------|---------|
| `createPending(request)` | `REQUIRES_NEW` | Create entity on notify(); commits independently of listener's outer tx |
| `load(id)` | `REQUIRED` | Short read tx; entity detaches on return, fields accessible in Phase 2 |
| `findEligibleIds(now)` | `REQUIRED` | Query PENDING∪FAILED where nextRetryAfter ≤ now or null |
| `markDelivered(id, snapshot, attempts, now)` | `REQUIRES_NEW` | Entity → DELIVERED + ledger write, atomic |
| `markFailed(id, snapshot, reason, attempts, nextRetry)` | `REQUIRES_NEW` | Entity → FAILED + ledger write, atomic |
| `markExhausted(id, snapshot, reason, attempts)` | `REQUIRES_NEW` | Entity → EXHAUSTED + ledger write, atomic |

`@Inject SponsorNotificationLedgerWriter` inside the store gives atomicity: entity state and notification ledger commit together. The deviation ledger write (`deviationLedgerWriter`, its own `REQUIRES_NEW`) is a separate commit called from the delivery service immediately after — a small acknowledged window.

---

## `SponsorNotificationRetryJob` and `SponsorNotificationDeliveryService`

**`SponsorNotificationRetryJob`** — `@ApplicationScoped`, `@Scheduled`, **not** `@Transactional`:

```java
@Scheduled(every = "${casehub.clinical.sponsor-notifier.poll-interval:60}s")
void tick() {
    for (UUID id : store.findEligibleIds(clock.instant())) {
        delivery.attemptDelivery(id);
    }
}
```

The loop is not transactional. Per GE-20260522-44bbf3: one notification failing must not roll back others. Added to `quarkus.arc.exclude-types` in test `application.properties`.

**`SponsorNotificationDeliveryService`** — package-private `@ApplicationScoped`, **not** `@Transactional` at the outer method:

Three-phase pattern (connector call never inside a transaction):

1. **Phase 1** — `store.load(id)`: REQUIRED tx reads entity, commits on return. Entity detaches; fields are accessible.
2. **Phase 2** — `connector.send(message)`: no transaction. Connector call outside tx boundary prevents Agroal pool exhaustion under concurrent load.
3. **Phase 3** — `store.markDelivered/Failed/Exhausted(...)`: REQUIRES_NEW. Entity update and `SponsorNotificationLedgerWriter` commit atomically inside the store's REQUIRES_NEW. `DeviationLedgerWriter` runs in its own REQUIRES_NEW call immediately after (acknowledged small window).

**Success Phase 3:**
- `store.markDelivered(id, snapshot, attemptNumber, now)` — entity → DELIVERED + ledger write, atomic (REQUIRES_NEW)
- `deviationLedgerWriter.writeSponsorNotifiedEntry(...)` — deviation audit chain, own REQUIRES_NEW

**Failure Phase 3 (retries remain — `attemptNumber < maxAttempts`):**
- `store.markFailed(id, snapshot, reason, attemptNumber, nextRetryAfter)` — entity → FAILED + ledger write, atomic (REQUIRES_NEW)

**Exhaustion Phase 3 (`attemptNumber >= maxAttempts`):**
- `store.markExhausted(id, snapshot, reason, attemptNumber)` — entity → EXHAUSTED + ledger write, atomic (REQUIRES_NEW)
- `deviationLedgerWriter.writeObserverFailureEntry(...)` — deviation audit chain, own REQUIRES_NEW
- `failedEvents.fireAsync(new SponsorNotificationFailedEvent(...))`

`buildTitle()` and `buildBody()` (moved from `DefaultSponsorNotifier`) live in the delivery service as package-private methods.

---

## `SponsorNotificationLedgerEntry` and writer

**`SponsorNotificationLedgerEntry`** in `io.casehub.clinical.ledger`:

```java
@Entity
@Table(name = "sponsor_notification_ledger_entry")
public class SponsorNotificationLedgerEntry extends LedgerEntry {
    public UUID   notificationId;  // same as subjectId, explicit for query clarity
    public UUID   deviationId;     // cross-reference; no FK (cross-datasource)
    public int    attemptNumber;
    public boolean delivered;
    public String  failureReason;  // null on success
}
```

Migration: **V2000** on qhorus datasource (`db/migration/qhorus/V2000__sponsor_notification_ledger_entry.sql`).

`subjectId = notificationId` — notification attempt chain is isolated from the deviation's ledger chain. `deviationId` provides the cross-reference for audit traversal.

**Actor roles:**
- Delivered: `"sponsor-notifier"`
- Failed attempt: `"sponsor-notifier-attempt-failed"`
- Exhausted: `"sponsor-notifier-exhausted"`

`actorId = ClinicalActors.CLINICAL_SERVICE`, `actorType = ActorType.SYSTEM` on all entries (per observer-failure-actor-role-naming protocol).

**`SponsorNotificationLedgerWriter`** — `@ApplicationScoped`, follows harness-ledger-writer protocol:

```java
@ApplicationScoped
public class SponsorNotificationLedgerWriter {
    @Inject LedgerEntryRepository repo;
    @Inject Clock clock;

    @Transactional public void writeDelivered(SponsorNotification n, int attempt) { ... }
    @Transactional public void writeFailed(SponsorNotification n, int attempt, String reason) { ... }
    @Transactional public void writeExhausted(SponsorNotification n, int attempt, String reason) { ... }

    private int nextSequenceNumber(UUID notificationId) {
        return repo.findLatestBySubjectId(notificationId)
            .map(e -> e.sequenceNumber + 1).orElse(1);
    }
}
```

Writer methods are `@Transactional` (REQUIRED) — they join the surrounding REQUIRES_NEW transaction from Phase 3, so entity update and ledger write commit atomically.

---

## Retry policy preferences

**`SponsorNotificationRetryPolicy`** in `api/`:

```java
public record SponsorNotificationRetryPolicy(
    int      maxAttempts,   // total attempts including first; minimum 1
    Duration retryInterval  // wait between attempts (fixed interval)
) implements Preference {
    public static final SponsorNotificationRetryPolicy DEFAULT =
        new SponsorNotificationRetryPolicy(3, Duration.ofMinutes(30));
}

public static final PreferenceKey<SponsorNotificationRetryPolicy> RETRY_POLICY =
    PreferenceKey.of("sponsor-notification.retry-policy", SponsorNotificationRetryPolicy.class);
```

`maxAttempts = 1` → try once, on failure → EXHAUSTED immediately. Equivalent to old fire-and-forget behaviour, but with full audit trail.

**YAML config** (`casehub-platform-config` backend, `config/preferences.yaml`):
```yaml
casehubio:
  clinical:
    sponsor-notification.retry-policy:
      maxAttempts: 3
      retryInterval: PT30M
```

**New dependency:** `casehub-platform-config` added to `runtime/pom.xml` — first use of YAML preference backend in clinical.

The scheduler's polling frequency (`casehub.clinical.sponsor-notifier.poll-interval`, default 60s) is a MicroProfile Config property, not a preference — it must be known at startup.

---

## `SponsorNotificationFailedEvent`

In `api/`:

```java
public record SponsorNotificationFailedEvent(
    UUID              notificationId,
    UUID              deviationId,
    UUID              trialId,
    UUID              siteId,
    DeviationSeverity severity,
    PiApprovalStatus  terminalStatus,
    String            failureReason,
    int               totalAttempts
) {}
```

Fired via `Event.fireAsync()` on EXHAUSTED. No consumer defined in this issue — extension point for casehubio/clinical#60 (WorkItem escalation).

---

## Deletions

- `DefaultSponsorNotifier` — deleted
- `DefaultSponsorNotifierTest` — deleted
- `DefaultSponsorNotifierBodyTest` — deleted

---

## Testing

| Test class | Type | Covers |
|---|---|---|
| `DurableSponsorNotifierTest` | Unit | `notify()` calls `store.createPending()` |
| `SponsorNotificationLedgerWriterTest` | Unit (`@InjectMock LedgerEntryRepository`) | sequence number, actor roles, subjectId |
| `SponsorNotificationStoreTest` | `@QuarkusTest` | createPending fields, state transitions, findEligibleIds query |
| `SponsorNotificationDeliveryServiceTest` | `@QuarkusTest` | success/failure/exhaustion paths; maxAttempts=1; connector-not-found |
| `SponsorNotificationIntegrationTest` | `@QuarkusTest` | CDI event → PENDING entity → delivery → DELIVERED state + ledger |

Retry job excluded via `quarkus.arc.exclude-types` in test `application.properties`. Tests drive delivery via direct `deliveryService.attemptDelivery(id)` calls. Preferences overridden per test via `@InjectMock Preferences`.

**Robustness cases:** concurrent pickup (entity reloaded in Phase 1 — status check skips if no longer eligible); `nextRetryAfter` boundary; listener skip paths unchanged.

---

## Flyway version plan

| Datasource | Migration | Description |
|---|---|---|
| default | V115 | `sponsor_notification` table |
| qhorus | V2000 | `sponsor_notification_ledger_entry` join table |

Existing V1005–V1014 (qhorus) are a protocol violation; cleanup tracked in casehubio/clinical#62.

---

## Deferred

| Issue | Description |
|---|---|
| casehubio/clinical#60 | WorkItem escalation on `SponsorNotificationFailedEvent` |
| casehubio/clinical#61 | Exponential backoff for retry interval |
| casehubio/clinical#62 | Renumber V1005–V1014 to V2000+ |
