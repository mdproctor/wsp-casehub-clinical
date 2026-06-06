# Design Journal — issue-21-durable-sponsor-notifier

### 2026-06-06 · §9.4·DomainBaseline

`SponsorNotification` entity added to the default datasource (V115). Payload snapshot at `notify()` time — `piDisplayName`, `connectorId`, `destination`, `severity`, `terminalStatus`, `deviationType` — so retries do not re-resolve PI identity or re-read connector config. Re-resolution on retry would introduce a race with directory service availability and changed trial config between attempts. Snapshotting at listener time eliminates that ambiguity.

`SponsorNotificationLedgerEntry` added to the qhorus datasource (V2020). Subject is `notificationId`, not `deviationId`: (1) GDPR Art.17 — if a deviation is expunged, the notification audit chain survives independently; (2) the deviation's ledger chain carries the FDA summary (delivered/exhausted, final timestamp) while the notification chain carries per-attempt operational detail. Cross-traversal uses the `deviationId` field stored on the entry. V2020 leaves a gap above the #62 renumbering range (V2000–V2009).

Channel name format changed from `clinical/deviation/<UUID>/pi-oversight` to `clinical/deviation/dev-<UUID>/pi-oversight`. A new qhorus snapshot tightened `ChannelSlugValidator` to require `[a-z][a-z0-9]*(-[a-z0-9]+)*` on all segments. UUID segments start with hex digits, violating the `[a-z]` start requirement. The `dev-` prefix satisfies the validator while preserving the UUID as a parseable group-1 capture. `PiResponseListener.CHANNEL_PATTERN` updated to `dev-([0-9a-f-]+)`. Tracked: casehubio/clinical#63.

`InMemoryLedgerEntryRepository` added to test `selected-alternatives`. A new `casehub-ledger` snapshot introduced `LedgerSequenceAllocator` (native SQL `MERGE INTO ledger_subject_sequence`) which is not a Hibernate-mapped entity and therefore not created by `drop-and-create`. Switching to the in-memory alternative avoids the missing-table failure without enabling Flyway in tests.

### 2026-06-06 · §9.4·Layer4Sponsor

`DefaultSponsorNotifier` (fire-and-forget) replaced by `DurableSponsorNotifier` as the sole `SponsorNotifier` implementation. `notify()` persists PENDING and returns immediately; delivery runs via `SponsorNotificationRetryJob` on the next poll tick (default 60s). Synchronous first attempt was rejected because it holds an Agroal DB connection open during the connector HTTP call for every concurrent `ProtocolDeviationResolvedEvent` — pool-exhausting under multi-site load.

Retry policy via `SponsorNotificationRetryPolicy` (`SingleValuePreference`, `casehub-platform-config`). Format `"maxAttempts,retryIntervalMinutes"` — plain string, no jackson-datatype-jsr310 dependency. `maxAttempts = 1` is the fire-and-forget equivalent with full audit trail.

`SponsorNotificationStore.markDelivered/Failed/Exhausted()` call `SponsorNotificationLedgerWriter` from within their `REQUIRES_NEW` transaction, making entity state update and notification ledger write atomic. Terminal-status guards prevent post-commit status regression: if `deviationLedgerWriter.writeSponsorNotifiedEntry()` throws after `store.markDelivered()` committed, the catch block previously called `store.markFailed()`, downgrading DELIVERED to FAILED. Deviation ledger writes are now outside the connector try-catch with isolated try-catch-log.
