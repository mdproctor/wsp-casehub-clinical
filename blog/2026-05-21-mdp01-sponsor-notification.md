---
layout: post
title: "The Sponsor's Notification Address"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [clinical]
tags: [gcp, cdi, connectors, compliance]
---

ICH E6(R3) §4.5 is unambiguous: the investigator must notify the sponsor promptly of any deviation that significantly affects subject safety or data integrity. MAJOR deviations hit that threshold by definition.

What surprised me was the scope of "promptly." I'd assumed it applied to approved deviations — PI signs off, sponsor gets notified. It doesn't. The sponsor needs to know all three outcomes: approval, rejection, and the PI going silent past the response deadline. Especially the last two. A MAJOR deviation where the PI refused to authorise it, or simply didn't respond in time, is a coordination failure at the site level. The sponsor has to know.

The design question was where to put the sponsor's notification endpoint — Slack webhook, Teams channel, email address. The naive answer is `application.properties`, but that breaks as soon as there are two trials: each trial has a different sponsor with different contact details, and per-trial configuration in a deployment config file means a redeployment every time you onboard a new trial. Sponsor contact details are domain data. They belong on `ClinicalTrial` alongside the protocol ID and enrollment target, set at trial registration time. The column is nullable; a trial without connector config produces a warning and skips notification rather than failing.

The delivery layer went behind a `SponsorNotifier` SPI. The interface is a single method — `notify(SponsorNotificationRequest)` — and the default implementation handles the common case. The interesting structural choice was around transactions. Connector delivery (HTTP call to Slack, Teams, whatever) must not run inside a database transaction — you don't want a slow external call holding a connection open. The ledger write that records the delivery attempt should be in its own transaction, independent of whether delivery succeeded. So `DefaultSponsorNotifier` does the HTTP call outside any transaction, then runs the ledger write in `REQUIRES_NEW`:

```java
public void notify(SponsorNotificationRequest req) {
    // connector.send() — outside any transaction
    connector.send(new ConnectorMessage(req.sponsorNotificationDestination(),
        buildTitle(req), buildBody(req)));
    recordAttempt(req, true);
}

@Transactional(Transactional.TxType.REQUIRES_NEW)
protected void recordAttempt(SponsorNotificationRequest req, boolean delivered) {
    ProtocolDeviation dev = ProtocolDeviation.findById(req.deviationId());
    ledgerWriter.writeSponsorNotifiedEntry(dev, clock.instant(), delivered);
}
```

The `REQUIRES_NEW` also means a delivery failure doesn't roll back anything meaningful — the PI resolution entry was committed before the CDI event fired.

## Two CDI surprises

Adding `casehub-connectors-core` to the classpath brings in `TwilioSmsConnector` and `WhatsAppConnector`, both of which inject required credential configuration. In a JDBC-only test environment with no external service config, ArC validates the entire bean graph at build time and fails to start. The error names the connector class but says nothing about missing config keys — you'd have to read the connector source to find them. Fix: `quarkus.arc.exclude-types` for both in test properties. `SlackConnector` avoids the problem because it takes the webhook URL as a runtime argument; the SMS and WhatsApp connectors need account credentials at build augmentation time.

The other issue was in the test capture pattern. The `@Mock` connector used a static `ArrayList` to capture sent messages. `@ObservesAsync` dispatches on a background executor thread; the test thread reads the list in an Awaitility polling loop. A code review pass caught the race — `CopyOnWriteArrayList` and a `volatile` boolean close it. Under Quarkus's default serial test execution the failure window is narrow enough that you'd never see it flake, which is exactly the condition under which latent races survive into production.
