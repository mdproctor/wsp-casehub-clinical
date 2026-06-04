---
title: "Who notified whom — closing an audit gap in GCP sponsor notifications"
date: 2026-06-04
tags: [casehub-clinical, gcp, audit, spi, pi-identity, compliance]
---

The sponsor notification path in casehub-clinical had a gap I hadn't noticed: the ledger recorded whether a notification was delivered, but not who it was delivered about. `writeSponsorNotifiedEntry` wrote `actorRole: sponsor-notifier`, `sponsorNotifiedAt`, and nothing else. An FDA auditor reading the audit trail could see that a sponsor was notified; they could not reconstruct which PI authorised the corrective action or what name appeared in the notification body.

That's a GCP compliance problem, not a nice-to-have.

The fix required introducing a `PiIdentityResolver` SPI to translate system actor IDs (`claude:pi@v1`, `dr-jones@v1`) into formal names suitable for regulated notification bodies. Where the SPI lives turned out to matter more than the SPI itself.

The obvious placement is `DefaultSponsorNotifier.buildBody()` — it's the method that assembles the message body, so inject the resolver there and call it before formatting. That placement has a real problem: any exception from the resolver propagates into `notify()`'s catch block, which records `delivered: false`. A directory service timeout now produces a ledger entry that says sponsor notification delivery failed — but delivery was never attempted. An FDA auditor cannot distinguish "couldn't reach the PI directory" from "couldn't reach the Slack webhook." The audit trail falsifies the failure reason.

The right placement is `SponsorNotificationListener.onDeviationResolved()`, before the `SponsorNotificationRequest` is built. Moving it there gives us two independent failure paths: a resolver failure writes `sponsor-notifier-pi-resolver-failed` and returns; a delivery failure writes `sponsor-notifier-observer-failed`. Both are auditable and unambiguous. The resolved name flows into the request as `piDisplayName`, and from there into `writeSponsorNotifiedEntry`, which now records both `piId` (the system-authoritative identity) and `piDisplayName` (what the sponsor was told). V1014 adds the `pi_display_name` column to the `protocol_deviation_ledger_entry` join table.

This also required thinking about what a null return from `resolveFormalName()` means. The SPI contract says: "actor not found — return the actorId unchanged." A null return is a contract violation, not a valid "unknown PI" signal. We treat it the same as a throw — write the resolver-failed audit entry and halt. A misbehaving custom resolver that returns null would otherwise silently produce an ESCALATED ledger entry with `piDisplayName = null`, which is the compliance gap we just fixed, via the resolver.

The `EXPIRED` case is simpler: system-initiated expiry has no PI actor, so `piId` and `piDisplayName` are both null and neither the resolver nor the body formatter references them. Recording null in the ledger is accurate — the audit trail correctly shows no PI was involved.

One other gotcha surfaced during testing. Adding `@InjectMock PiIdentityResolver` to `SponsorNotificationListenerTest` broke an existing test that doesn't use the resolver at all. Mockito's default return for a stubbed String method is null, and the null-return guard I'd just added fired — writing `sponsor-notifier-pi-resolver-failed` instead of `sponsor-notifier-observer-failed`. The fix is a default `when(piIdentityResolver.resolveFormalName(any())).thenReturn("Dr. Smith")` in `@BeforeEach`, so all tests in the class get a safe non-null return. Individual tests override where they need to control the output. It's not obvious that `@InjectMock` replaces the CDI bean for every test in the class, not just the test that declares the mock.

The PATCH endpoint for `/trials/{id}/sponsor-config` that landed alongside this is straightforward by comparison — the columns were already there from clinical#13, so it's a single method and four tests. Full replace over partial update: a connector ID without a destination is non-functional, so updating both atomically is the right model.
