# Design Spec — clinical#14 + clinical#15: Protocol Deviation Resolution Ledger Entries

**Date:** 2026-05-18
**Issues:** casehubio/clinical#14 (resolution entries), casehubio/clinical#15 (sequenceNumber fix)

---

## Problem

`ProtocolDeviationLedgerEntry` is written once — when a deviation is commanded to the PI
(sequence 1, type COMMAND). The Merkle chain has the obligation but no resolution. An FDA
inspector sees the PI was formally commanded; they cannot see from the tamper-evident chain
how it was resolved: approved, rejected, or expired. The audit trail is structurally
incomplete.

Additionally, `sequenceNumber` is hardcoded to 1. A second entry for the same deviation
would produce a duplicate sequence number, breaking chain integrity.

---

## Out of Scope

- Linking clinical ledger entries to qhorus normative records via `commitmentId` — the
  normative link already exists through `ProtocolDeviation.commitmentId`. The normative
  layer's structural value is demonstrated by the architecture (COMMAND/Commitment lifecycle,
  expiration detection, named obligation), not by a foreign key annotation in the ledger entry.

---

## Schema — Flyway V1007 (qhorus datasource)

Two nullable columns added to `protocol_deviation_ledger_entry`:

```sql
ALTER TABLE protocol_deviation_ledger_entry
    ADD COLUMN terminal_status VARCHAR(50),
    ADD COLUMN resolved_at     TIMESTAMP WITH TIME ZONE;
```

Nullable because existing COMMAND entries predate these columns and carry no resolution state.
New COMMAND entries also leave these null — resolution is populated only by resolution entries.

No new join table. No new entity class.

---

## Entity Changes — `ProtocolDeviationLedgerEntry`

```java
@Column(name = "terminal_status")
public String terminalStatus;   // null for COMMAND entries

@Column(name = "resolved_at")
public Instant resolvedAt;      // null for COMMAND entries
```

---

## New Component — `DeviationLedgerWriter @ApplicationScoped`

Centralises all ledger writing for protocol deviations. Previously `LedgerEntryRepository`
was injected directly into `ProtocolDeviationService`; this component owns it instead.

```
writeCommandEntry(ProtocolDeviation dev, String piId)
    entryType = COMMAND
    actorId   = "clinical-service"
    actorType = SYSTEM
    actorRole = "deviation-reporter"
    sequenceNumber = nextSequenceNumber(dev.id)   ← fixes clinical#15
    terminalStatus = null
    resolvedAt     = null

writeResolutionEntry(ProtocolDeviation dev, PiApprovalStatus terminalStatus,
                     String actorId, ActorType actorType, String actorRole)
    entryType      = EVENT
    sequenceNumber = nextSequenceNumber(dev.id)
    terminalStatus = terminalStatus.name()
    resolvedAt     = Instant.now()
    occurredAt     = Instant.now()

nextSequenceNumber(UUID deviationId)
    → ledgerEntryRepository.findLatestBySubjectId(deviationId)
          .map(e -> e.sequenceNumber + 1)
          .orElse(1)
```

Common fields populated in all entries: `id` (random UUID), `subjectId = dev.id`,
`deviationId = dev.id`, `siteId = dev.siteId`, `severity = dev.severity.name()`,
`piId = piId` (command) or null (resolution).

---

## Call Sites

### `ProtocolDeviationService`

Replace private `writeLedgerEntry()` with `deviationLedgerWriter.writeCommandEntry(deviation, piId)`.
Remove `@Inject LedgerEntryRepository` — it moves to `DeviationLedgerWriter`.

### `PiResponseListener.process()`

After `deviation.piApprovalStatus` is set (APPROVED, ESCALATED, or REJECTED), before
`resolvedEvent.fireAsync(...)`:

```java
deviationLedgerWriter.writeResolutionEntry(
    deviation, deviation.piApprovalStatus,
    senderId, ActorType.HUMAN, "pi-authoriser");
```

`senderId` is the PI's actor ID from the channel message. `deviation.piApprovalStatus` at
this point is already the terminal state (APPROVED, ESCALATED, or REJECTED).

### `DeviationExpirationJob.checkExpiredCommitments()`

Inside the try block, after `d.piApprovalStatus = PiApprovalStatus.EXPIRED`, before
`resolvedEvent.fireAsync(...)`:

```java
deviationLedgerWriter.writeResolutionEntry(
    d, PiApprovalStatus.EXPIRED,
    "system", ActorType.SYSTEM, "deviation-expiration-job");
```

---

## Entry Shapes

| Field | COMMAND | PI approved/escalated | PI rejected | Expired |
|-------|---------|----------------------|-------------|---------|
| `entryType` | COMMAND | EVENT | EVENT | EVENT |
| `actorId` | `"clinical-service"` | PI sender ID | PI sender ID | `"system"` |
| `actorType` | SYSTEM | HUMAN | HUMAN | SYSTEM |
| `actorRole` | `"deviation-reporter"` | `"pi-authoriser"` | `"pi-authoriser"` | `"deviation-expiration-job"` |
| `terminalStatus` | null | APPROVED or ESCALATED | REJECTED | EXPIRED |
| `resolvedAt` | null | Instant.now() | Instant.now() | Instant.now() |
| `sequenceNumber` | computed | computed | computed | computed |

---

## Testing

### `DeviationLedgerWriterTest` (unit — mocked `LedgerEntryRepository`)

- `writeCommandEntry`: entry has `entryType=COMMAND`, `terminalStatus=null`,
  `resolvedAt=null`, `sequenceNumber=1` when no prior entries exist
- `writeCommandEntry`: `sequenceNumber=3` when prior max is 2
- `writeResolutionEntry` APPROVED: `entryType=EVENT`, `terminalStatus="APPROVED"`,
  `actorType=HUMAN`, `resolvedAt` non-null, `sequenceNumber=2` when prior COMMAND at seq 1
- `writeResolutionEntry` ESCALATED: `terminalStatus="ESCALATED"`
- `writeResolutionEntry` REJECTED: `terminalStatus="REJECTED"`, `actorType=HUMAN`
- `writeResolutionEntry` EXPIRED: `terminalStatus="EXPIRED"`, `actorId="system"`,
  `actorType=SYSTEM`

### `ProtocolDeviationServiceTest` (existing integration — add assertions)

- After `reportDeviation()`: assert COMMAND ledger entry exists with `terminalStatus=null`,
  `sequenceNumber=1`

### `PiResponseListenerTest` (existing — add assertions)

- APPROVED response: assert resolution entry written with `terminalStatus=APPROVED`,
  `actorId=senderId`, `sequenceNumber=2`
- ESCALATED response (CRITICAL deviation): assert `terminalStatus=ESCALATED`
- REJECTED response: assert `terminalStatus=REJECTED`, `actorId=senderId`

### `DeviationExpirationJobTest` (existing — add assertions)

- After `checkExpiredCommitments()` for one overdue deviation: assert EXPIRED entry written
  with `actorId="system"`, `actorType=SYSTEM`, `sequenceNumber=2`
- Two overdue deviations: each gets its own EXPIRED entry (sequenceNumbers independent per
  deviation)

---

## Deferred

| Issue | What |
|-------|------|
| casehubio/qhorus#153 | `MessageReceivedEvent` — when shipped, `PiResponseListenerIntegrationTest` can run end-to-end and the full resolution chain (channel message → ledger entry) will be testable |
