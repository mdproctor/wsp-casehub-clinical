# Design Journal — epic-protocol-deviation-pi-auth

## Epic 5: PI Authorisation for Protocol Deviations

**Branch:** `epic-protocol-deviation-pi-auth`
**Issue:** casehubio/clinical#5
**Dates:** 2026-05-15 – 2026-05-17
**Design spec:** `specs/2026-05-15-epic5-pi-authorisation-design.md`

---

### What was built

Formal PI (Principal Investigator) authorisation for protocol deviations via pure qhorus COMMAND/Commitment lifecycle. When a deviation is reported, a per-deviation qhorus channel is created and a COMMAND is issued to the site PI. The PI's structured JSON response updates the deviation status through a 6-state machine. Downstream epics consume `ProtocolDeviationResolvedEvent` without modifying this layer.

**Closes the structural ClinicalAgent gap:** ClinicalAgent logs deviations. A log proves notice. A COMMAND creates a formal obligation — a named PI on record with a deadline.

---

### Key architecture decisions

**Per-deviation channels, not per-site.**
Channel naming: `clinical/deviation/{deviationId}/pi-oversight`. Per-site channels were considered but rejected because `ChannelGateway.receiveHumanMessage()` passes `correlationId=null` to MessageService (qhorus#154) — the deviation can only be identified from the channel name when the CDI event path is used. Per-deviation channels make the mapping unambiguous regardless of this gap.

**`MessageService.send()` auto-opens Commitment on COMMAND type.**
No explicit `commitmentService.open()` call needed. The MessageService switch statement handles it automatically for COMMAND type + non-null correlationId. The correlationId = `deviation.id.toString()` across all commitment operations (fulfill, decline, fail).

**`PiResponseListener` must close the Commitment explicitly.**
Because `receiveHumanMessage()` passes `correlationId=null`, the auto-state-machine in MessageService does NOT fire for PI responses. `PiResponseListener.process()` calls `commitmentService.fulfill()` or `.decline()` directly using the known correlationId.

**`DeviationResponsePolicy` SPI — deadline + escalation configurable per deployment.**
`DefaultDeviationResponsePolicy @DefaultBean` reads from MicroProfile Config. Deployers override via `@ApplicationScoped` without `@DefaultBean`. Deadline and escalation path are determined by the same policy call — a deployer who needs "MAJOR oncology deviations at German sites require regulatory authority notification within 48h" implements one SPI, not three.

**`ClinicalInboundNormaliser` scoped to `/pi-oversight` channels.**
The `InboundNormaliser` SPI is application-wide. Scoping detection to channel names containing `/pi-oversight` prevents misclassifying messages on other channels. Whitespace-tolerant JSON matching added after code review.

**`ProtocolDeviationResolvedEvent` CDI event — no service modification on future epics.**
Fires on APPROVED, REJECTED, ESCALATED, EXPIRED. Epic 6 (IRB gate) and Epic 13 (sponsor notification) observe this event — no changes to this service when they ship.

**Flyway migration subdirectory restructure.**
Adding casehub-qhorus to the classpath alongside casehub-work causes a version collision at `classpath:db/migration` (both ship V1+). Fix: clinical migrations moved to datasource-scoped subdirectories (`db/migration/default/`, `db/migration/qhorus/`). Tests use drop-and-create + Flyway disabled. AML has the same latent issue — casehubio/aml#20.

---

### Deferred

| Issue | What | When |
|---|---|---|
| casehubio/qhorus#153 | MessageReceivedEvent CDI hook — unblocks PiResponseListenerIntegrationTest | Before Epic 5 integration test runs end-to-end |
| casehubio/qhorus#154 | InboundHumanMessage.correlationId gap — enables per-message commitment auto-close | Optional — clinical works around it |
| casehubio/clinical#6 | IRB gate — consumes ProtocolDeviationResolvedEvent with IRB_REVIEW | Epic 6 |
| casehubio/clinical#13 | Sponsor notification — consumes ProtocolDeviationResolvedEvent with SPONSOR_NOTIFICATION | After connectors pattern established |
| casehubio/clinical#14 | Ledger entries on PI response and expiration | Follow-up |
| casehubio/clinical#15 | sequenceNumber hardcoded to 1 | With #14 |
| casehubio/aml#20 | AML Flyway classpath collision — same fix as clinical | AML session |
| casehubio/parent#22 | casehub-clinical.md layer table update | Next parent session |
| casehubio/parent#23 | PLATFORM.md cross-repo dep table: casehub-qhorus → casehub-clinical | After Epic 5 merges |
