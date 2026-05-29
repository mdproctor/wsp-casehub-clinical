---
layout: post
title: "A domain event fires once"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [gcp, cdi, connectors, compliance, casehub-work]
---

Issue #11 — AE safety officer notification — is in main.

The compliance requirement is ICH E6(R3) §5.17 and 21 CFR 312.32: when a serious adverse event is reported, the safety officer must be reachable, and the fact that they were notified (or that delivery failed) must appear in the tamper-evident audit trail. The ledger write was non-negotiable. Delivery success and failure both get entries.

The interesting decision was the trigger.

The issue specification said to observe `WorkItemLifecycleEvent` for `category = "adverse-event"` and `status = PENDING` — fire when the WorkItem is created. That's the natural thing to reach for. But Grade 4/5 AE engine cases create two WorkItems: one for the senior safety monitor and one for the DSMB escalation, both from YAML `humanTask` bindings. `WorkItemLifecycleEvent` would fire twice per AE. The notification would go out twice. For a death notification, that's not a delivery detail — it's a compliance error.

The fix was to use `AdverseEventReportedEvent` instead. It fires once, at service boundary, before engine orchestration begins. Grade 3+ only. It carries the AE id, grade, enrollment id, and site id directly — everything the notification needs without a secondary lookup. The `SafetyOfficerNotificationListener` observes it and does the rest: site → trial → connector config → SPI call.

The design went through a short iteration on the SPI shape. The first sketch had two SPIs: a `SafetyOfficerRoutingPolicy` that resolved the connector id and destination, and `SafetyOfficerNotifier` that dispatched. The routing policy was justified by "the user said SPI-configurable." Code review pushed back: the routing SPI creates asymmetry with `SponsorNotificationListener`, which does inline entity lookup with no routing layer. And `SafetyOfficerNotifier` already takes the grade in the request, so a deployer who wants Grade 5 routed to an emergency pager just overrides the notifier — the routing policy solves nothing the notifier doesn't already solve. Dropped it. One SPI, one listener, inline lookup. Same shape as the sponsor notification.

`DefaultSponsorNotifier` was using `@Any Instance<Connector>` while the new `DefaultSafetyOfficerNotifier` used `@All List<Connector>`. Two injection patterns for the same dependency. I updated `DefaultSponsorNotifier` in the same commit — constructor injection, `Collectors.toMap`, done. The `@All List<>` pattern is already in the garden (GE-20260526-1653dc) because it's more testable: you can hand it a `List.of(fakeConnector)` in a plain JUnit test without touching CDI.

The compliance surface is `SafetyOfficerNotificationLedgerEntry` — JOINED inheritance on the qhorus datasource, V1011 migration. Same pattern as `AeEscalationLedgerEntry` and `AdverseEventLedgerEntry`. The only wrinkle was the `delivered` boolean: true on successful connector send, false if the connector isn't found or throws. Either way the entry is written. `DefaultSafetyOfficerNotifier` runs the connector call and the ledger write in `REQUIRES_NEW` so a delivery exception doesn't roll back either.

Two follow-on issues filed: #45 (no ledger entry if the async observer itself throws — the try/catch only covers connector delivery, not upstream DB errors in the listener) and #46 (actorId "system" vs "clinical-service" inconsistency across all ledger writers — pre-existing, replicated here).

`ClinicalTrial` got two new columns: `safety_officer_connector_id` and `safety_officer_destination` (V114, VARCHAR(2048) to match the existing sponsor notification column). Nullable — missing config logs a warning and skips notification rather than failing hard.

152 tests passing.
