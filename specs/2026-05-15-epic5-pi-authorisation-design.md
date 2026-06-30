# Design Spec — Epic 5: PI Authorisation for Protocol Deviations

**Date:** 2026-05-15
**Issue:** casehubio/clinical#5
**Branch:** `epic-protocol-deviation-pi-auth`

---

## Problem

ICH E6(R3) §4.5 requires a named Principal Investigator to take formal responsibility for protocol deviations at their site. A deviation log proves the deviation was noticed — not that anyone was accountable for it. ClinicalAgent (peer-reviewed baseline) logs deviations; it has no named PI, no formal commitment, no deadline. Those are different things.

CaseHub's COMMAND lifecycle closes this gap structurally: a COMMAND issued to a named participant creates a formal obligation with a deadline, tracked through a 7-state Commitment lifecycle, recorded as a tamper-evident `MessageLedgerEntry`.

---

## Architecture Decision: Approach A — Pure Qhorus Channel

The PI's formal response is captured entirely within the qhorus normative layer. No REST endpoint on clinical acts as a channel bypass.

**Why not Approach B (COMMAND via qhorus + clinical REST response endpoint):**
When a real `HumanParticipatingChannelBackend` is registered (Slack, email, future PI dashboard), the REST endpoint becomes dead code or creates a dual-entry problem. Worse: the RESPONSE is recorded in clinical's database via the REST handler, not as a RESPONSE message in the qhorus channel. The COMMAND exists; the Commitment exists; the fulfillment arrives through a side channel that bypasses the ledger. The audit trail is structurally incomplete — a GCP violation.

**Critical path dependency:**
`PiResponseListener` (which observes inbound PI RESPONSE messages) requires **casehubio/qhorus#153** — CDI `MessageReceivedEvent` fired by `MessageService` on every inbound message. Everything else in Epic 5 (service, channel creation, COMMAND issuance, SPI, expiration job, unit tests) is independent of that issue and proceeds now.

---

## Domain Model Changes

### `api/` module

**`PiApprovalStatus`** — three new values:

| Value | Meaning |
|---|---|
| `PENDING` | Deviation reported; COMMAND not yet issued (transient — service issues COMMAND in same transaction) |
| `COMMANDED` | COMMAND issued; Commitment OPEN; awaiting PI response |
| `APPROVED` | PI responded positively; Commitment FULFILLED; MINOR deviations close here |
| `REJECTED` | PI declined or contested the classification; Commitment DECLINED |
| `EXPIRED` | Deadline passed without response; Commitment FAILED; GCP SLA breach |
| `ESCALATED` | PI approved; forwarded to IRB (CRITICAL) or sponsor (MAJOR); Commitment FULFILLED |

`ESCALATED` is a domain state, not a Commitment state. It is set immediately when the deviation is forwarded downstream — it does not wait for confirmation from the downstream epic.

**`EscalationRequirement`** (new enum in `api/`):

```java
public enum EscalationRequirement { NONE, SPONSOR_NOTIFICATION, IRB_REVIEW }
```

**`DeviationResponsePolicy` SPI** (`api/spi/`):

```java
public interface DeviationResponsePolicy {
    DeviationResponseRequirements evaluate(DeviationContext context);
}
```

**`DeviationContext`** (new record in `api/spi/`):

```java
public record DeviationContext(
    UUID deviationId,
    UUID siteId,
    UUID trialId,
    String protocolId,
    TrialPhase phase,
    DeviationSeverity severity,
    String deviationType
) {}
```

Carries enough context for a deployer to scope policy by trial, site, sponsor, or phase — without requiring JPA in the SPI implementation.

**`DeviationResponseRequirements`** (new record in `api/spi/`):

```java
public record DeviationResponseRequirements(
    Duration piResponseDeadline,
    EscalationRequirement escalationRequirement
) {}
```

The policy owns both the deadline AND the escalation path. A deployer who needs "MAJOR oncology trial deviations at German sites require regulatory authority notification within 48h" implements this SPI — the service is unchanged.

**`ProtocolDeviationResolvedEvent`** (new record in `api/`):

```java
public record ProtocolDeviationResolvedEvent(
    UUID deviationId,
    UUID siteId,
    DeviationSeverity severity,
    EscalationRequirement escalationRequirement,
    PiApprovalStatus terminalStatus   // APPROVED, REJECTED, EXPIRED, ESCALATED
) {}
```

Fired by `PiResponseListener` and `DeviationExpirationJob`. Consumed by:
- Epic 6 (`casehubio/clinical#6`): observes `IRB_REVIEW` → creates IrbApproval WorkItem
- Epic 13 (`casehubio/clinical#13`): observes `SPONSOR_NOTIFICATION` → notifies sponsor via casehub-connectors

No code changes to clinical's deviation service when those epics ship.

### `runtime/` module

**`ProtocolDeviation` entity** — five new fields (Flyway V107):

```java
public UUID commitmentId;               // qhorus Commitment correlator
public String piCommandChannelName;     // e.g. "clinical/site/{siteId}/pi-oversight"
public Instant commandedAt;
public Instant responseDeadline;

@Enumerated(EnumType.STRING)
public EscalationRequirement escalationRequirement;
```

**`ProtocolDeviationLedgerEntry`** (`io.casehub.clinical.ledger`, Flyway V1006):

```java
@Entity @Table(name = "protocol_deviation_ledger_entry")
@DiscriminatorValue("PROTOCOL_DEVIATION")
public class ProtocolDeviationLedgerEntry extends LedgerEntry {
    public UUID deviationId;
    public UUID siteId;
    public String severity;
    public String piId;
    public Instant commandedAt;
    public Instant responseDeadline;
    public String escalationRequirement;
}
```

---

## Service Design

### `DefaultDeviationResponsePolicy @ApplicationScoped @DefaultBean` (`runtime/`)

Reads deadline from MicroProfile Config with hardcoded fallbacks:

```properties
casehub.clinical.deviation.minor.deadline=168h   # 7 days
casehub.clinical.deviation.major.deadline=72h    # 3 days
casehub.clinical.deviation.critical.deadline=24h # 1 day
```

Default escalation mapping (overridable via custom SPI impl):
- `MINOR` → `NONE` (site-level resolution)
- `MAJOR` → `SPONSOR_NOTIFICATION` (casehubio/clinical#13)
- `CRITICAL` → `IRB_REVIEW` (casehubio/clinical#6)

Displaced by any `@ApplicationScoped` implementation without `@DefaultBean`.

### `ProtocolDeviationService @ApplicationScoped` (`runtime/`)

Single public method: `reportDeviation(ProtocolDeviation deviation, UUID siteId, UUID trialId)`

Executes in one `@Transactional` call:

1. Validate site belongs to trial (`TrialSite.findById(siteId)`)
2. Resolve `ClinicalTrial` for `protocolId` and `phase`
3. Set `deviation.id`, `deviation.siteId`, `deviation.piApprovalStatus = PENDING`
4. Build `DeviationContext` from resolved entities
5. Call `deviationResponsePolicy.evaluate(context)` → `requirements`
6. Create PI oversight channel if absent: `clinical/site/{siteId}/pi-oversight`
   - `allowedTypes = QUERY, COMMAND` (per protocol PP-20260508-a15390)
   - `semantics = APPEND`
   - If channel already exists (idempotent re-creation on restart or second deviation at the same site): use existing. Verify the correct qhorus API for create-if-absent at implementation time — handle `ChannelAlreadyExistsException` or use a `findOrCreate` variant if available.
7. Issue COMMAND message on channel via `MessageService`:
   - `sender` = `"clinical-service"` (the clinical harness's qhorus instance id — registered once at startup via `InstanceService.register()` or equivalent; verify injectable API at implementation time)
   - `content` = structured JSON: `{ "deviationId": "...", "deviationType": "...", "severity": "MAJOR", "responseDeadline": "..." }`
   - `correlationId` = `deviation.id.toString()` — the key correlator for PI response lookup
8. Open Commitment via `CommitmentService`:
   - `debtorId` = `site.investigatorId`
   - `creditorId` = `"clinical-service"`
   - `deadline` = `Instant.now() + requirements.piResponseDeadline()`
   - **Note:** `CommitmentService` is documented in qhorus as an MCP tool surface. Verify at implementation time that it is also a CDI-injectable Java bean — if not, raise an issue on casehubio/qhorus before proceeding.
9. Set `deviation.commitmentId`, `deviation.piCommandChannelName`, `deviation.commandedAt = Instant.now()`, `deviation.responseDeadline`, `deviation.escalationRequirement`
10. Set `deviation.piApprovalStatus = COMMANDED`
11. Persist `deviation`
12. Write `ProtocolDeviationLedgerEntry` with all audit fields

### `PiResponseListener @ApplicationScoped` (`runtime/`)

**Blocked on casehubio/qhorus#153 for integration.** Implement and unit-test now; integration test gated on qhorus#153 merge.

```java
void onMessage(@ObservesAsync MessageReceivedEvent event) {
    if (event.messageType() != MessageType.RESPONSE) return;
    if (event.correlationId() == null) return;

    UUID deviationId = UUID.fromString(event.correlationId());
    ProtocolDeviation deviation = ProtocolDeviation.findById(deviationId);
    if (deviation == null || deviation.piApprovalStatus != PiApprovalStatus.COMMANDED) return;

    // PI response content format: JSON {"decision":"APPROVED","comment":"..."}
    // comment is optional; decision is required. REJECTED if decision != "APPROVED".
    boolean approved = "APPROVED".equals(parseDecision(event.content()));
    updateDeviationFromResponse(deviation, approved, event.senderId());
}
```

`updateDeviationFromResponse` (in `@Transactional` — `@ObservesAsync` fires without the original transaction context; `@Transactional` here creates a new transaction, which is correct):
- APPROVED + `escalationRequirement != NONE` → `piApprovalStatus = ESCALATED`
- APPROVED + `escalationRequirement == NONE` → `piApprovalStatus = APPROVED`
- REJECTED → `piApprovalStatus = REJECTED`
- Call `CommitmentService.fulfill(deviation.commitmentId)` or `.decline()`
- Fire `deviationResolvedEvent.fireAsync(new ProtocolDeviationResolvedEvent(...))`
- Persist deviation

### `DeviationExpirationJob @ApplicationScoped` (`runtime/`)

No qhorus CDI event needed for expiry — periodic reconciliation:

```java
@Scheduled(every = "${casehub.clinical.deviation.expiration-check-interval:1h}",
           identity = "deviation-expiration")
@Transactional
void checkExpiredCommitments() {
    List<ProtocolDeviation> expired = ProtocolDeviation
        .find("piApprovalStatus = ?1 and responseDeadline < ?2",
              PiApprovalStatus.COMMANDED, Instant.now())
        .list();

    for (ProtocolDeviation d : expired) {
        d.piApprovalStatus = PiApprovalStatus.EXPIRED;
        commitmentService.fail(d.commitmentId);
        deviationResolvedEvent.fireAsync(
            new ProtocolDeviationResolvedEvent(d.id, d.siteId, d.severity,
                                               d.escalationRequirement,
                                               PiApprovalStatus.EXPIRED));
    }
}
```

---

## REST Layer

**`DeviationResource @Path("/trials/{trialId}/sites/{siteId}/deviations")`**

```
POST   /trials/{trialId}/sites/{siteId}/deviations           → 201 Created + Location
GET    /trials/{trialId}/sites/{siteId}/deviations/{id}      → 200 with full deviation state
```

`POST` body:
```json
{ "deviationType": "sample-collection-window", "severity": "MAJOR" }
```

Response includes `piApprovalStatus`, `responseDeadline`, `commitmentId`. Client can poll GET for status transitions.

No PI response endpoint. The PI's formal response arrives via `HumanParticipatingChannelBackend` registered at deployment.

---

## Infrastructure Changes

### `runtime/pom.xml` — add direct qhorus dependency
```xml
<!-- Layer 3: formal PI obligation tracking via COMMAND lifecycle -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus</artifactId>
</dependency>
```

### `api/pom.xml` — add qhorus-api for SPI types
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-api</artifactId>
</dependency>
```

### `application.properties` — add qhorus runtime scan packages
```properties
# Extend qhorus PU to include qhorus entity package
quarkus.hibernate-orm.qhorus.packages=\
  io.casehub.qhorus.runtime,\
  io.casehub.ledger.runtime.model,\
  io.casehub.ledger.model,\
  io.casehub.clinical.ledger
```

### `test/application.properties` — reactive suppression (verify if needed)
If qhorus#GE-20260508-492336 is still active, add:
```properties
casehub.qhorus.reactive.enabled=false
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.reactive=false
```
Verify against current qhorus version before adding — may already be resolved.

---

## Flyway Migrations

**V107** — `alter_protocol_deviation_add_commitment_fields.sql`:
```sql
ALTER TABLE protocol_deviation
    ADD COLUMN commitment_id          UUID,
    ADD COLUMN pi_command_channel_name VARCHAR(500),
    ADD COLUMN commanded_at           TIMESTAMP WITH TIME ZONE,
    ADD COLUMN response_deadline      TIMESTAMP WITH TIME ZONE,
    ADD COLUMN escalation_requirement VARCHAR(50);
```

**V1006** — `protocol_deviation_ledger_entry.sql`:
```sql
CREATE TABLE protocol_deviation_ledger_entry (
    id                    UUID         NOT NULL,
    deviation_id          UUID         NOT NULL,
    site_id               UUID         NOT NULL,
    severity              VARCHAR(50)  NOT NULL,
    pi_id                 VARCHAR(255),
    commanded_at          TIMESTAMP WITH TIME ZONE,
    response_deadline     TIMESTAMP WITH TIME ZONE,
    escalation_requirement VARCHAR(50),
    CONSTRAINT pk_pd_ledger PRIMARY KEY (id),
    CONSTRAINT fk_pd_ledger_entry FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

---

## Testing

### Unit tests (`api/`)
- `DefaultDeviationResponsePolicyTest` — each severity produces correct deadline and escalation requirement; config overrides hardcoded defaults; custom impl displaces default via CDI
- `DeviationContextTest` — record equality, all fields set

### Integration tests (`runtime/`, `@QuarkusTest`)

**`ProtocolDeviationServiceTest`**
- Report MINOR deviation → `piApprovalStatus = COMMANDED`; `commitmentId` non-null; `responseDeadline = commandedAt + 7d`; `escalationRequirement = NONE`; `ProtocolDeviationLedgerEntry` written
- Report MAJOR deviation → `responseDeadline = commandedAt + 72h`; `escalationRequirement = SPONSOR_NOTIFICATION`
- Report CRITICAL deviation → `responseDeadline = commandedAt + 24h`; `escalationRequirement = IRB_REVIEW`
- `POST` to non-existent site → 404
- `POST` to site from wrong trial → 404
- Channel created with `allowedTypes = QUERY, COMMAND` (verify via qhorus channel query)
- COMMAND message present in channel with correct `correlationId` (verify via `MessageService`)

**`PiResponseListenerTest`** — unit test (no CDI event trigger needed):
- APPROVED response on MINOR deviation → `piApprovalStatus = APPROVED`; `ProtocolDeviationResolvedEvent` with `escalationRequirement = NONE`
- APPROVED response on CRITICAL deviation → `piApprovalStatus = ESCALATED`; `ProtocolDeviationResolvedEvent` with `IRB_REVIEW`
- REJECTED response → `piApprovalStatus = REJECTED`; `ProtocolDeviationResolvedEvent` with `REJECTED`
- Non-RESPONSE message type → no state change
- Unknown correlationId → no state change, no exception
- Already-terminal deviation (APPROVED) → no state change (idempotent)

**`PiResponseListenerIntegrationTest`** — `@QuarkusTest` full channel flow:
- **Gated on casehubio/qhorus#153** — annotate `@Disabled("Requires qhorus#153 — MessageReceivedEvent CDI hook")`
- Report deviation → simulate PI RESPONSE via `ChannelGateway.receiveHumanMessage()` → assert status transition + Commitment FULFILLED + `ProtocolDeviationResolvedEvent` fired

**`DeviationExpirationJobTest`**
- Insert `COMMANDED` deviation with `responseDeadline = Instant.now().minus(1h)` → call `checkExpiredCommitments()` directly → assert `piApprovalStatus = EXPIRED` + `ProtocolDeviationResolvedEvent(EXPIRED)` fired
- Insert `COMMANDED` deviation with future deadline → assert no state change

**`DeviationResourceTest`** — REST layer:
- `POST` deviation → 201 + `Location` header + `piApprovalStatus = COMMANDED` in body
- `GET` deviation → 200 with all fields including `responseDeadline`
- `GET` non-existent → 404
- `POST` missing `severity` → 400
- `POST` missing `deviationType` → 400

### End-to-end (`ShowcaseScenarioTest`)
Extend existing 3-site test:
- Site C reports a MINOR deviation → `COMMANDED`
- Site C reports a CRITICAL deviation → `COMMANDED`, `escalationRequirement = IRB_REVIEW`
- Simulate PI RESPONSE (APPROVED) on MINOR → `APPROVED`
- Simulate PI RESPONSE (APPROVED) on CRITICAL → `ESCALATED`
- Simulate PI RESPONSE (REJECTED) on any → `REJECTED`

---

## Deferred — Tracked Issues

| Issue | What | When |
|---|---|---|
| casehubio/qhorus#153 | `MessageReceivedEvent` CDI hook — unblocks `PiResponseListenerIntegrationTest` | Before Epic 5 integration test can run end-to-end |
| casehubio/clinical#6 | IRB gate — consumes `ProtocolDeviationResolvedEvent` with `IRB_REVIEW` | Epic 6 |
| casehubio/clinical#13 | Sponsor notification — consumes `ProtocolDeviationResolvedEvent` with `SPONSOR_NOTIFICATION` | After Epic 7 / connectors pattern established |
| casehubio/parent#22 | `casehub-clinical.md` layer table update | Next parent session |
| casehubio/parent#23 | PLATFORM.md cross-repo dep table: add casehub-qhorus → casehub-clinical | After Epic 5 merges |