# Design Spec — clinical#18: DeviationExpirationJob Per-Deviation Transaction Isolation

**Date:** 2026-05-19
**Issue:** casehubio/clinical#18

---

## Problem

`DeviationExpirationJob.checkExpiredCommitments()` runs the entire per-deviation
loop inside a single `@Transactional` method with a try/catch. The method Javadoc
claims "one failure must not roll back status updates for other deviations" — but
this guarantee does not hold at the JPA level.

If any write inside the try block causes a JPA/JDBC exception (constraint violation,
deadlock, connection timeout), Hibernate marks the entire transaction rollback-only.
The catch block cannot recover from this — it sets `d.piApprovalStatus = COMMANDED`
but that write is in the same doomed transaction. On commit, every deviation that
succeeded in the loop also rolls back.

In a regulated clinical trial context, silently losing expiration updates for
multiple deviations in a batch — with no trace of which ones succeeded — is not
acceptable for FDA audit.

---

## Decision

**Approach A — `DeviationExpirer @ApplicationScoped` helper bean.**

Extract a second CDI bean that owns the per-deviation transaction boundary using
`@Transactional(TxType.REQUIRES_NEW)`. The job becomes a non-transactional
orchestrator that reads IDs and delegates. Each deviation commits or rolls back
independently.

Alternatives considered and rejected:
- `QuarkusTransaction.requiringNew()` inline — equivalent semantics, less CDI-idiomatic,
  harder to unit test in isolation, adds Quarkus-specific API dependency to the job
- Leave current structure — the Javadoc guarantee is structurally false; documenting the
  lie instead of fixing it is not acceptable

---

## Design

### `DeviationExpirer @ApplicationScoped` (new)

```java
@ApplicationScoped
public class DeviationExpirer {

    @Inject CommitmentService commitmentService;
    @Inject Event<ProtocolDeviationResolvedEvent> resolvedEvent;
    @Inject DeviationLedgerWriter ledgerWriter;

    @Transactional
    public List<UUID> findOverdueIds() {
        return ProtocolDeviation
            .find("piApprovalStatus = ?1 and responseDeadline < ?2",
                  PiApprovalStatus.COMMANDED, Instant.now())
            .<ProtocolDeviation>list()
            .stream().map(d -> d.id).toList();
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void expireOne(UUID deviationId) {
        ProtocolDeviation d = ProtocolDeviation.findById(deviationId);
        if (d == null || d.piApprovalStatus != PiApprovalStatus.COMMANDED) return;

        d.piApprovalStatus = PiApprovalStatus.EXPIRED;
        commitmentService.fail(d.id.toString());
        ledgerWriter.writeResolutionEntry(d, PiApprovalStatus.EXPIRED,
            "system", ActorType.SYSTEM, "deviation-expiration-job");
        resolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(
            d.id, d.siteId, d.severity,
            d.escalationRequirement != null ? d.escalationRequirement : EscalationRequirement.NONE,
            PiApprovalStatus.EXPIRED
        ));
    }
}
```

`expireOne` reloads the deviation from DB. This is required: REQUIRES_NEW suspends
the caller's transaction and creates a new one on a separate JDBC connection. The
entity from the caller's transaction context is detached in the new connection.
The null/status guard makes `expireOne` idempotent — safe if called twice.

### `DeviationExpirationJob @ApplicationScoped` (revised)

Remove `@Transactional`. Inject `DeviationExpirer`. The scheduler method becomes
a non-transactional orchestrator:

```java
@Inject DeviationExpirer expirer;

@Scheduled(every = "${casehub.clinical.deviation.expiration-check-interval:1h}",
           identity = "deviation-expiration")
public void checkExpiredCommitments() {
    for (UUID id : expirer.findOverdueIds()) {
        try {
            expirer.expireOne(id);
        } catch (Exception e) {
            Logger.getLogger(DeviationExpirationJob.class)
                .errorf(e, "Failed to expire deviation %s — will retry next run", id);
        }
    }
}
```

The Javadoc comment now holds: if `expireOne` throws a JPA exception, only that
deviation's REQUIRES_NEW sub-transaction rolls back. Other deviations already
committed are unaffected.

### Test restructure

Current tests use `@Transactional` at the test method level and persist deviations
inline. With REQUIRES_NEW, `expireOne` runs on a separate DB connection and cannot
see the test's uncommitted `dev.persist()`.

Fix: deviations must be persisted in a committed transaction before `checkExpiredCommitments()`
is called. Add a package-private `@ApplicationScoped TestDeviationPersister` in
`src/test/java/io/casehub/clinical/service/`:

```java
@ApplicationScoped
class TestDeviationPersister {
    @Transactional
    UUID persistCommanded(UUID siteId, DeviationSeverity severity,
                          EscalationRequirement esc, Instant deadline) {
        ProtocolDeviation d = new ProtocolDeviation();
        d.id = UUID.randomUUID();
        d.siteId = siteId;
        d.deviationType = "test";
        d.severity = severity;
        d.piApprovalStatus = PiApprovalStatus.COMMANDED;
        d.escalationRequirement = esc;
        d.piCommandChannelName = "clinical/deviation/" + d.id + "/pi-oversight";
        d.commandedAt = Instant.now().minus(10, ChronoUnit.DAYS);
        d.responseDeadline = deadline;
        d.persist();
        return d.id;
    }
}
```

Test methods remove `@Transactional`. They inject `TestDeviationPersister`, call
`persister.persistCommanded(...)` (commits on return), then call
`job.checkExpiredCommitments()`, then assert directly via `ProtocolDeviation.findById`
(in an auto-managed Panache transaction) and `ledgerRepo.findBySubjectId`.

Post-test cleanup: REQUIRES_NEW commits independently, so test data persists after
the test method. This is safe — each test uses unique UUIDs and the schema is
`drop-and-create` per test suite run.

---

## Testing

### `DeviationExpirationJobTest` (revised)

- `overdueCommandedDeviationIsMarkedExpired` — persist via `TestDeviationPersister`,
  call `checkExpiredCommitments()`, assert EXPIRED status and EXPIRED ledger entry
- `twoOverdueDeviationsEachGetIndependentLedgerEntry` — persist two via
  `TestDeviationPersister`, assert each has exactly one ledger entry
- `futureDeadlineDeviationIsNotExpired` — persist future-deadline via
  `TestDeviationPersister`, assert COMMANDED status unchanged

### `DeviationExpirerTest` (new unit test)

Unit test `expireOne` directly using `@QuarkusTest`:

- `expireOne_skiipsNullDeviation` — call with non-existent UUID, assert no exception
- `expireOne_skipsAlreadyTerminalDeviation` — call with APPROVED deviation, assert no change
- `expireOne_expiresCommandedDeviation` — persist via `TestDeviationPersister`, call
  `expirer.expireOne(id)`, assert EXPIRED + ledger entry

---

## Files

| Action | File |
|--------|------|
| CREATE | `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirer.java` |
| MODIFY | `runtime/src/main/java/io/casehub/clinical/service/DeviationExpirationJob.java` |
| CREATE | `runtime/src/test/java/io/casehub/clinical/service/TestDeviationPersister.java` |
| MODIFY | `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirationJobTest.java` |
| CREATE | `runtime/src/test/java/io/casehub/clinical/service/DeviationExpirerTest.java` |
