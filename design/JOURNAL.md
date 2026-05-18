# Design Journal — epic-deviation-resolution-ledger

### 2026-05-18 · §Platform Integration

The `ProtocolDeviationLedgerEntry` Merkle chain previously had only a COMMAND entry — the obligation was recorded but not its resolution. An FDA audit trail without a terminal entry is structurally incomplete: the inspector can see a PI was formally commanded but cannot see from the tamper-evident chain whether the obligation was discharged, rejected, or expired. This epic closes the gap.

`DeviationLedgerWriter @ApplicationScoped` now owns all ledger writes for the deviation lifecycle. The centralisation decision was driven by sequenceNumber: three services write entries for the same subject (`ProtocolDeviationService`, `PiResponseListener`, `DeviationExpirationJob`) and the chain integrity requires that sequenceNumber increments correctly across all of them. Each service calling `findLatestBySubjectId` independently works, but with no shared ownership the invariant is invisible. `DeviationLedgerWriter` makes it explicit and testable in isolation.

Resolution entries use `LedgerEntryType.EVENT` (not a new type), with `terminal_status` and `resolved_at` as nullable columns (V1007). COMMAND entries leave both null; resolution entries populate them. This keeps a single entity class covering the full lifecycle without a subclass split, at the cost of two nullable columns — an acceptable trade-off given ledger entry schemas are already sparse by design.

### 2026-05-18 · §Key Architecture Decisions

Two new decisions established this epic:

**`DeviationLedgerWriter` as the canonical pattern for multi-service ledger writes.** When a ledger subclass is written from more than one service, a dedicated writer bean should own sequenceNumber computation and entry construction. Without this, each service independently reads the latest sequence and there is no place to test the invariant in isolation. The pattern should be followed for any future subclass written from multiple call sites.

**`quarkus.arc.exclude-types` for ledger SNAPSHOT reactive services.** casehub-ledger 0.2-SNAPSHOT ships services (`LedgerVerificationService` et al.) that inject `ReactiveLedgerEntryRepository`, which is `@Vetoed` in the JDBC-only test environment. The CDI context fails to start without the exclusion. The fix is a test `application.properties` entry — not a code change — but it must be updated each time the ledger SNAPSHOT adds a new reactive-dependent service. Tracked for upstream resolution as clinical#17.
