# DurableSponsorNotifier — Design Spec

**Issue:** casehubio/clinical#21  
**Branch:** issue-21-durable-sponsor-notifier  
**Date:** 2026-06-05  
**Revised:** 2026-06-05 (post-review)

---

## Context

`DefaultSponsorNotifier` delivers sponsor notifications fire-and-forget via connector. It satisfies the GCP prompt-notification requirement for the showcase but has no persistence, no retry, and no audit trail beyond a single deviation ledger entry. `DurableSponsorNotifier` replaces it: every notification attempt is persisted, retried on failure, and independently audited.

`DefaultSponsorNotifier` is deleted in this branch. `DurableSponsorNotifier` is the sole `SponsorNotifier` implementation — there is no CDI displacement mechanism; the old class simply does not exist. The fire-and-forget case is expressed via `maxAttempts = 1` in the retry policy preference, not a separate implementation.

---

## Deployment constraints

**Single-node only.** `SponsorNotificationRetryJob` does not implement distributed locking. Two pods polling simultaneously will both find the same PENDING notifications, both enter Phase 2 (connector send), and the sponsor receives duplicates. For the clinical showcase, single-node deployment is the assumed model.

Multi-node safety requires SELECT FOR UPDATE SKIP LOCKED in `findEligibleIds()` — tracked as a future enhancement. This must be an explicit go/no-go gate before any production multi-node deployment.

---

## Architecture

```
SponsorNotificationListener.onDeviationResolved()
    └─ DurableSponsorNotifier.notify()
           └─ SponsorNotificationStore.createPending()  [REQUIRES_NEW — commits before listener's tx]

SponsorNotificationRetryJob  (@Scheduled, not @Transactional, single-node)
    └─ for each id in store.findEligibleIds(now, limit=100):
           try { delivery.attemptDelivery(id) } catch (Exception e) { log.error(...) }

SponsorNotificationDeliveryService.attemptDelivery(id)
    Phase 1: store.load(id)                    [REQUIRED, short tx, entity detaches]
    Phase 2: connector.send(message)            [no transaction]
    Phase 3: store.markDelivered/Failed/Exhausted()  [REQUIRES_NEW]
                 └─ SponsorNotificationLedgerWriter  [REQUIRED joins store's REQUIRES_NEW — atomic]
             deviationLedgerWriter.write*()     [own REQUIRES_NEW immediately after — acknowledged window]
             if EXHAUSTED: fireAsync(SponsorNotificationExhaustedEvent)
```

**XA requirement:** `store.markDelivered/Failed/Exhausted()` write across the default datasource (entity update) and qhorus (via `SponsorNotificationLedgerWriter`). This is a cross-datasource `REQUIRES_NEW` and requires XA on both datasources — already configured in clinical. Do not remove `quarkus.datasource.jdbc.transactions=xa` or `quarkus.datasource.qhorus.jdbc.transactions=xa`; those entries exist partly for this service.

**First-attempt latency:** `notify()` persists PENDING and returns; the scheduler delivers on the next poll tick (default 60s). This is a behaviour change from `DefaultSponsorNotifier`, which delivered synchronously within the listener. The 60s window is acceptable for the showcase — GCP does not mandate sub-minute sponsor notification. Eager first attempt (call delivery inline in `notify()`) is flagged as a future enhancement if showcase fidelity requires it.

---

## `SponsorNotification` entity

Package: `io.casehub.clinical.entity`  
Datasource: default  
Migration: **V115** (`db/migration/default/V115__sponsor_notification.sql`)

```java
@Entity
@Table(
    name = "sponsor_notification",
    indexes = {
        @Index(name = "idx_sn_eligible", columnList = "status, next_retry_after")
    }
)
public class SponsorNotification extends PanacheEntityBase {

    @Id
    @Column(nullable = false)
    public UUID id;

    // Subject references — no FK constraints (cross-datasource)
    @Column(name = "deviation_id", nullable = false)
    public UUID deviationId;

    @Column(name = "trial_id", nullable = false)
    public UUID trialId;

    @Column(name = "site_id", nullable = false)
    public UUID siteId;

    // Future multi-tenancy — nullable; findEligibleIds() must filter by tenantId when set
    @Column(name = "tenant_id", length = 64)
    public String tenantId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 32)
    public SponsorNotificationStatus status;   // PENDING | FAILED | DELIVERED | EXHAUSTED

    // attempts = number of completed delivery attempts
    // Phase 3 sets n.attempts = attemptNumber (the value incremented from n.attempts + 1 in Phase 1)
    @Column(nullable = false)
    public int attempts;

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @Column(name = "last_attempted_at")
    public Instant lastAttemptedAt;

    @Column(name = "next_retry_after")
    public Instant nextRetryAfter;    // null = eligible immediately; set on failure

    @Column(name = "delivered_at")
    public Instant deliveredAt;

    @Column(name = "failure_reason", length = 1000)
    public String failureReason;     // last failure message, overwritten per attempt

    // Payload snapshot — resolved at notify() time, used verbatim on retry.
    // piDisplayName resolved by PiIdentityResolver at listener time (directory may differ on retry).
    // connectorId/destination snapshotted to preserve original delivery target if trial config changes.
    @Column(name = "pi_id", length = 255)
    public String piId;

    @Column(name = "pi_display_name", length = 255)
    public String piDisplayName;

    @Column(name = "connector_id", nullable = false, length = 128)
    public String connectorId;

    @Column(name = "destination", nullable = false, length = 2048)
    public String destination;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 32)
    public DeviationSeverity severity;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 32)
    public PiApprovalStatus terminalStatus;

    @Column(name = "deviation_type", nullable = false, length = 128)
    public String deviationType;
}
```

**Invariant:** `SponsorNotification` must never gain `@OneToMany`, `@ManyToOne`, or any other association mapping. `store.load(id)` returns the entity after its read transaction commits; any lazy-loaded association would throw `LazyInitializationException` in Phase 2. If an association is ever added, the load pattern must be revised.

`SponsorNotificationStatus` enum in `api/model/`: `PENDING`, `FAILED`, `DELIVERED`, `EXHAUSTED`.

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

`notify()` does one thing: persist a PENDING entity. The listener's skip paths (site-not-found, no-config, PI-resolver-failed) bypass `notify()` and write to the deviation ledger directly — unchanged.

**CDI binding:** `DurableSponsorNotifier` is `@ApplicationScoped`. `DefaultSponsorNotifier` is deleted. No `@Alternative` / `selected-alternatives` entry is needed — `DurableSponsorNotifier` is the only `SponsorNotifier` bean on the classpath. The integration test must verify CDI lookup resolves to `DurableSponsorNotifier` with no ambiguity.

**Listener rollback:** `store.createPending()` uses `REQUIRES_NEW` and commits before the listener's outer transaction. If the listener's transaction subsequently rolls back (CDI dispatch failure after `notify()` returns), the PENDING entity survives. This is intentional: delivery must proceed independently of the listener's transaction outcome.

---

## `SponsorNotificationStore`

Package: `io.casehub.clinical.service` (package-private `@ApplicationScoped`)

All beans that inject the store (`DurableSponsorNotifier`, `SponsorNotificationDeliveryService`, `SponsorNotificationRetryJob`) must remain in the same package — Quarkus ArC can proxy package-private classes only when the injecting class is co-located.

The store owns all `SponsorNotification` entity mutations **and** calls `SponsorNotificationLedgerWriter` from within its outcome methods, making entity update and notification ledger write atomic in the same `REQUIRES_NEW` transaction. The writer's `@Transactional(REQUIRED)` joins the surrounding `REQUIRES_NEW`.

```java
@Inject SponsorNotificationLedgerWriter ledgerWriter;
@Inject Clock clock;
```

**Key methods:**

| Method | Transaction | Purpose |
|--------|-------------|---------|
| `createPending(request)` | `REQUIRES_NEW` | Create PENDING entity; commits independently of listener's outer tx |
| `load(id)` | `REQUIRED` | Short read tx; entity detaches on return — scalar fields only (see invariant above) |
| `findEligibleIds(now, limit)` | `REQUIRED` | Query `status IN (PENDING, FAILED) AND (nextRetryAfter IS NULL OR nextRetryAfter <= now) LIMIT limit` |
| `markDelivered(id, snapshot, attemptNumber, deliveredAt)` | `REQUIRES_NEW` | Entity → DELIVERED (attempts = attemptNumber, deliveredAt = connector ack time) + ledger write, atomic |
| `markFailed(id, snapshot, reason, attemptNumber, nextRetry)` | `REQUIRES_NEW` | Entity → FAILED (attempts = attemptNumber, nextRetryAfter = nextRetry) + ledger write, atomic |
| `markExhausted(id, snapshot, reason, attemptNumber)` | `REQUIRES_NEW` | Entity → EXHAUSTED (attempts = attemptNumber) + ledger write, atomic |

`snapshot` is the `SponsorNotification` value read in Phase 1 — passed to the ledger writer for field values without a second DB read inside the `REQUIRES_NEW`.

---

## `SponsorNotificationRetryJob` and `SponsorNotificationDeliveryService`

**`SponsorNotificationRetryJob`** — `@ApplicationScoped`, `@Scheduled`, **not** `@Transactional`:

```java
@ApplicationScoped
class SponsorNotificationRetryJob {
    @Inject SponsorNotificationStore store;
    @Inject SponsorNotificationDeliveryService delivery;
    @Inject Clock clock;

    @ConfigProperty(name = "casehub.clinical.sponsor-notifier.batch-size", defaultValue = "100")
    int batchSize;

    @Scheduled(every = "${casehub.clinical.sponsor-notifier.poll-interval:60}s")
    void tick() {
        for (UUID id : store.findEligibleIds(clock.instant(), batchSize)) {
            try {
                delivery.attemptDelivery(id);
            } catch (Exception e) {
                LOG.errorf(e, "SponsorNotificationRetryJob: unhandled error for notificationId=%s — skipping", id);
            }
        }
    }
}
```

Per GE-20260522-44bbf3: the loop is not transactional. One notification throwing must not roll back or abandon others — hence the per-iteration try-catch-log. Added to `quarkus.arc.exclude-types` in test `application.properties`.

**`SponsorNotificationDeliveryService`** — package-private `@ApplicationScoped`, **not** `@Transactional` at the outer method. Injects `Clock clock` and `Preferences preferences`. Builds the connector registry via constructor injection (same pattern as the deleted `DefaultSponsorNotifier`):

```java
private final Map<String, Connector> connectorRegistry;

@Inject
SponsorNotificationDeliveryService(@All List<Connector> connectors, ...) {
    this.connectorRegistry = connectors.stream()
        .collect(Collectors.toMap(Connector::id, Function.identity()));
}
```

"Connector not found" means `connectorRegistry.get(snapshot.connectorId)` returns null — treated as a failure with reason `"connector-not-found: " + snapshot.connectorId`, same code path as a connector throw.

Retry policy is resolved once per `attemptDelivery()` call:
```java
SponsorNotificationRetryPolicy policy = preferences
    .get(RETRY_POLICY, new SettingsScope(Path.of("casehubio", "clinical"), clock.instant()))
    .orElse(SponsorNotificationRetryPolicy.DEFAULT);
int      maxAttempts   = policy.maxAttempts();
Duration retryInterval = policy.retryInterval();
```

Three-phase pattern:

1. **Phase 1** — `store.load(id)`: REQUIRED tx reads entity, commits on return. Entity detaches; scalar fields accessible. Check `status` — if DELIVERED or EXHAUSTED, skip (guards against concurrent pickup, though single-node deployment makes this rare).
2. **Phase 2** — `connector.send(message)`: no transaction. Connector call outside tx boundary; `buildTitle()`/`buildBody()` are package-private methods moved from the deleted `DefaultSponsorNotifier`.
3. **Phase 3** — outcome recording:

**Success path — timestamp capture:** `Instant deliveredAt = clock.instant()` is captured **immediately after `connector.send()` returns successfully** — before any Phase 3 work begins. This records when the connector acknowledged delivery, not when the ledger write happened. The failure paths each capture their own `clock.instant()` at failure-detection time (correct — records when the failure was observed, not when delivery was attempted).

**Success (Phase 3):**
- `store.markDelivered(id, snapshot, attemptNumber, deliveredAt)` — entity → DELIVERED + ledger write, atomic (REQUIRES_NEW)
- `deviationLedgerWriter.writeSponsorNotifiedEntry(snapshot.deviationId, snapshot.siteId, snapshot.severity, deliveredAt, snapshot.piId, snapshot.piDisplayName)` — deviation chain final outcome, own REQUIRES_NEW

**Failure with retries remaining (Phase 3 — `attemptNumber < maxAttempts`):**
- `store.markFailed(id, snapshot, reason, attemptNumber, now.plus(retryInterval))` — entity → FAILED + ledger write, atomic (REQUIRES_NEW)

**Exhaustion (Phase 3 — `attemptNumber >= maxAttempts`):**
- `store.markExhausted(id, snapshot, reason, attemptNumber)` — entity → EXHAUSTED + ledger write, atomic (REQUIRES_NEW)
- `deviationLedgerWriter.writeExhaustedNotificationEntry(snapshot.deviationId, snapshot.siteId, snapshot.severity, now)` — deviation chain, own REQUIRES_NEW
- `exhaustedEvents.fireAsync(new SponsorNotificationExhaustedEvent(...))`

**Connector not found:** treated as failure — same code path as connector throw.

**`attempts` / `attemptNumber` definition:** `attempts` on the entity = number of completed delivery attempts. In `attemptDelivery()`, `int attemptNumber = snapshot.attempts + 1` at the start of Phase 1. Phase 3 stores `n.attempts = attemptNumber`. Exhaustion check: `if (attemptNumber >= maxAttempts)`.

---

## `DeviationLedgerWriter` changes

The existing `writeSponsorNotifiedEntry(ProtocolDeviation dev, ...)` overload is **deleted**. Its only caller was `DefaultSponsorNotifier.recordAttempt()`, which is deleted by this branch. The listener's skip paths call `writeSkippedSponsorEntry()` and `writeObserverFailureEntry()` — they never called `writeSponsorNotifiedEntry()`. Zero callers remain post-deletion.

Two new methods added:

```java
// Called by delivery service on successful delivery — takes fields from SponsorNotification snapshot
// rather than loading ProtocolDeviation (avoids cross-entity load in the delivery service)
@Transactional(REQUIRES_NEW)
public void writeSponsorNotifiedEntry(
    UUID deviationId, UUID siteId, DeviationSeverity severity,
    Instant notifiedAt, String piId, String piDisplayName) { ... }

// Called by delivery service on EXHAUSTED — distinct from writeObserverFailureEntry(),
// which is reserved for listener-level CDI/infrastructure failures only
@Transactional(REQUIRES_NEW)
public void writeExhaustedNotificationEntry(
    UUID deviationId, UUID siteId, DeviationSeverity severity, Instant occurredAt) {
    // actorRole = "sponsor-notifier-exhausted"
    // actorId = ClinicalActors.CLINICAL_SERVICE, actorType = ActorType.SYSTEM
}
```

Unchanged: `writeSkippedSponsorEntry()`, `writeObserverFailureEntry()` — both called by `SponsorNotificationListener` skip/catch paths.

---

## `SponsorNotificationLedgerEntry` and writer

**`SponsorNotificationLedgerEntry`** in `io.casehub.clinical.ledger`:

```java
@Entity
@Table(name = "sponsor_notification_ledger_entry")
@DiscriminatorValue("SponsorNotification")
public class SponsorNotificationLedgerEntry extends LedgerEntry {

    @Column(name = "notification_id", nullable = false)
    public UUID notificationId;     // same as subjectId, explicit for query clarity

    @Column(name = "deviation_id", nullable = false)
    public UUID deviationId;        // cross-reference; no FK (cross-datasource)

    @Column(name = "attempt_number", nullable = false)
    public int attemptNumber;

    @Column(name = "delivered", nullable = false)
    public boolean delivered;

    @Column(name = "failure_reason", length = 1000)
    public String failureReason;    // null on success
}
```

Migration: **V2020** on qhorus datasource (`db/migration/qhorus/V2020__sponsor_notification_ledger_entry.sql`).

V2020 chosen to leave a safe gap above the V2000–V2009 range reserved for the #62 renumbering of V1005–V1014. This branch must not land before #62 defines its final range, or V2020 must be confirmed clear at merge time.

**`subjectId` = `notificationId`** — the notification entity is the audit subject. This keeps the per-attempt chain isolated from the deviation's lifecycle chain for two reasons: (1) GDPR expungement — if a deviation record is erased under Art.17, the notification's tamper-evident chain survives independently; (2) the deviation chain carries the FDA summary (was the sponsor reached: yes/no, when) and the notification chain carries per-attempt operational detail. Cross-traversal from deviation → notification attempts: query `SponsorNotificationLedgerEntry` where `deviationId = ?` — the `deviationId` field exists specifically for this.

**Crash recovery / audit gap:** if the process crashes between `store.markDelivered()` (notification chain committed) and `deviationLedgerWriter.writeSponsorNotifiedEntry()` (deviation chain), the notification chain says DELIVERED but the deviation chain has no `sponsor-notifier` entry. Recovery: the notification chain is the primary FDA audit record. The deviation chain entry is supplementary cross-reference. An auditor needing the complete picture queries `SponsorNotificationLedgerEntry.deviationId`. No startup reconciliation scan is implemented in this issue; the acknowledged window is acceptable given single-node deployment and the short time between the two commits.

**Actor roles** (per `observer-failure-actor-role-naming` protocol):
- Delivered: `"sponsor-notifier"`
- Failed attempt: `"sponsor-notifier-attempt-failed"`
- Exhausted: `"sponsor-notifier-exhausted"`

`actorId = ClinicalActors.CLINICAL_SERVICE`, `actorType = ActorType.SYSTEM` on all entries.

**`SponsorNotificationLedgerWriter`** — `@ApplicationScoped`. Injects `@Inject Clock clock`. Follows harness-ledger-writer protocol — single owner of sequence number computation:

```java
@ApplicationScoped
public class SponsorNotificationLedgerWriter {
    @Inject LedgerEntryRepository repo;
    @Inject Clock clock;

    // deliveredAt is caller-supplied (connector acknowledgement time) — same pattern as
    // DeviationLedgerWriter.writeSponsorNotifiedEntry(), so both audit chains record identical timestamps
    @Transactional public void writeDelivered(SponsorNotification n, int attemptNumber, Instant deliveredAt) { ... }
    @Transactional public void writeFailed(SponsorNotification n, int attemptNumber, String reason) { ... }
    @Transactional public void writeExhausted(SponsorNotification n, int attemptNumber, String reason) { ... }

    private int nextSequenceNumber(UUID notificationId) {
        return repo.findLatestBySubjectId(notificationId)
            .map(e -> e.sequenceNumber + 1).orElse(1);
    }
}
```

---

## Retry policy preferences

**`SponsorNotificationRetryPolicy`** in `api/`:

```java
public record SponsorNotificationRetryPolicy(
    int      maxAttempts,   // total attempts including first; minimum 1
    Duration retryInterval  // fixed wait between attempts
) implements Preference {
    public static final SponsorNotificationRetryPolicy DEFAULT =
        new SponsorNotificationRetryPolicy(3, Duration.ofMinutes(30));
}

public static final PreferenceKey<SponsorNotificationRetryPolicy> RETRY_POLICY =
    PreferenceKey.of("sponsor-notification.retry-policy", SponsorNotificationRetryPolicy.class);
```

`maxAttempts = 1` → one attempt total; on failure → EXHAUSTED immediately. Equivalent to old fire-and-forget but with full audit trail. Up to one poll interval (default 60s) of first-attempt latency; accepted for showcase (see Deployment constraints).

**YAML config** (`casehub-platform-config` backend, `config/preferences.yaml`):
```yaml
casehubio:
  clinical:
    sponsor-notification.retry-policy:
      maxAttempts: 3
      retryInterval: PT30M
```

**New dependency:** `casehub-platform-config` added to `runtime/pom.xml`. This is the first use of the YAML preference backend in clinical. **Verify before implementing:**
- Does `casehub-platform-config` require a `quarkus.index-dependency` entry in `application.properties`? Almost certainly yes — follow the pattern from engine modules.
- Fallback behaviour when `preferences.yaml` is absent: expected to return `DEFAULT` silently (standard `casehub-platform-config` behaviour) — confirm this before relying on it in tests.
- **Duration deserialisation:** `retryInterval` is `java.time.Duration`; the YAML value is ISO 8601 (`PT30M`). Confirm `casehub-platform-config`'s Jackson mapper registers `jackson-datatype-jsr310` (or equivalent). If not, `Duration` deserialization fails at runtime with a confusing mapping error. Verify during the first integration test run; if absent, either register the module or change the preference type to `long` minutes.

The scheduler's polling frequency (`casehub.clinical.sponsor-notifier.poll-interval`, default 60s) and batch size (`casehub.clinical.sponsor-notifier.batch-size`, default 100) are MicroProfile Config properties — they must be known at startup.

---

## `SponsorNotificationExhaustedEvent`

In `api/`:

```java
public record SponsorNotificationExhaustedEvent(
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

Named `Exhausted` (not `Failed`) — FAILED is an intermediate status meaning retries remain; this event fires only when all attempts are consumed. Fired via `Event.fireAsync()` after `store.markExhausted()` commits. No consumer defined in this issue — extension point for casehubio/clinical#60 (WorkItem escalation).

---

## Deletions

- `DefaultSponsorNotifier` — deleted
- `DefaultSponsorNotifierTest` — deleted
- `DefaultSponsorNotifierBodyTest` — deleted

`buildBody()` and `buildTitle()` move into `SponsorNotificationDeliveryService` as package-private methods.

---

## Testing

| Test class | Type | Covers |
|---|---|---|
| `DurableSponsorNotifierTest` | Unit | `notify()` calls `store.createPending()` (Mockito mock on store) |
| `SponsorNotificationLedgerWriterTest` | Unit (`@InjectMock LedgerEntryRepository`) | sequence number, actor roles, `subjectId = notificationId` |
| `SponsorNotificationStoreTest` | `@QuarkusTest` | createPending fields; state transitions; findEligibleIds batch limit; excludes DELIVERED/EXHAUSTED; respects nextRetryAfter |
| `SponsorNotificationDeliveryServiceTest` | `@QuarkusTest` | success/failure/exhaustion paths; `maxAttempts = 1` direct exhaustion; connector-not-found; status-check skip in Phase 1; `attempts` field post-Phase-3 |
| `SponsorNotificationIntegrationTest` | `@QuarkusTest` | CDI event → PENDING entity → `attemptDelivery()` → DELIVERED + ledger entries + deviation chain entry; CDI lookup resolves `SponsorNotifier` to `DurableSponsorNotifier` with no ambiguity |
| `DeviationLedgerWriterTest` | existing | Extend to cover new `writeSponsorNotifiedEntry` and `writeExhaustedNotificationEntry` overloads |

Scheduler excluded via `quarkus.arc.exclude-types`. Tests drive delivery via direct `deliveryService.attemptDelivery(id)` calls. Preferences overridden via `@InjectMock Preferences`.

**Robustness cases:** Phase 1 status check skips already-terminal notifications; loop try-catch isolates per-notification failures; `nextRetryAfter` boundary; listener skip paths unchanged.

---

## Flyway version plan

| Datasource | Migration | Description |
|---|---|---|
| default | V115 | `sponsor_notification` table + `idx_sn_eligible` index |
| qhorus | V2020 | `sponsor_notification_ledger_entry` join table |

**V2020 ordering:** must not conflict with #62 (renumbering V1005–V1014 → V2000–V2009). Confirm V2010–V2019 are clear at merge time; adjust if #62 uses a wider range.

Existing V1005–V1014 (qhorus) are a protocol violation; cleanup tracked in casehubio/clinical#62.

---

## Deferred

| Issue | Description |
|---|---|
| casehubio/clinical#60 | WorkItem escalation on `SponsorNotificationExhaustedEvent` |
| casehubio/clinical#61 | Exponential backoff for retry interval |
| casehubio/clinical#62 | Renumber V1005–V1014 to V2000+ |
| (future) | SELECT FOR UPDATE SKIP LOCKED in `findEligibleIds()` for multi-node safety |
| (future) | Eager first attempt in `notify()` to eliminate poll-interval latency |
| (future) | Startup reconciliation scan for notification→deviation audit gap |
| (future) | Retention policy for DELIVERED/EXHAUSTED records — accumulate without bound; correct for tamper-evident audit but may need archiving for regulated long-running deployments |

**Implementation note:** `createPending()` must set `n.attempts = 0` explicitly (not rely on Java's default zero) — in regulated audit-trail code, "initialised to zero" and "forgotten to set" must be distinguishable at review time.
