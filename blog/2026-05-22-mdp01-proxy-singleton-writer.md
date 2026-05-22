---
layout: post
title: "Proxy, Singleton, and a Preemptive Writer"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [clinical]
tags: [cdi, quarkus, refactoring, ledger]
---

The most revealing part of this cleanup was what happened when we converted `TestSlackConnector` from static fields to instance fields.

`@Mock` without an explicit scope inherits the replaced bean's scope. `SlackConnector` is `@ApplicationScoped`, so the test double gets wrapped in a CDI client proxy — a generated subclass that delegates method calls to the real instance. Java field access is not virtual. `proxy.sent` reads the proxy's own `sent` field, which is initialised to an empty list and never populated. `proxy.send()` routes to the real instance and works. `proxy.sent` always looks empty. The tests fail in a way that points directly to "connector isn't being called", which is wrong.

`@Singleton` is the fix:

```java
@Mock
@Singleton  // not @ApplicationScoped — ArC proxies application-scoped beans;
            // field access on the proxy reads the proxy's field, not the delegate's
public class TestSlackConnector implements Connector {
    private final List<ConnectorMessage> sent = new CopyOnWriteArrayList<>();
    ...
}
```

ArC doesn't proxy singletons — they're injected directly, and field access and method calls both hit the real instance. We also moved to accessor methods (`sent()`, `setShouldThrow()`, `reset()`), which is cleaner for test infrastructure anyway, but the scope annotation is what makes it actually work.

Claude flagged one more issue during review: the backing list should stay `CopyOnWriteArrayList`. The integration test uses `@ObservesAsync`, which dispatches on a managed executor thread — the test thread reading the list through Awaitility is a genuine cross-thread read. `ArrayList` has no JMM visibility guarantee across threads. Under Quarkus's default serial test execution the failure window is narrow enough you'd never see it flake, which is exactly the condition under which latent races outlive the tests that should catch them.

## The same ledger pattern, applied one domain earlier

`AdverseEventService` was writing its ledger entry inline, `sequenceNumber` hardcoded to 1. One entry per adverse event — correct today, wrong the moment safety monitoring starts writing resolution and escalation entries for the same subject.

`DeviationLedgerWriter` exists because the deviation lifecycle eventually involved three separate services, each writing entries for the same subject. Without a shared owner, sequenceNumber computation scattered, untestable in isolation, prone to duplicates under concurrent writes. Centralising in one class, with `nextSequenceNumber()` reading from `findLatestBySubjectId()`, made the invariant explicit and testable.

I wanted `AdverseEventLedgerWriter` extracted now, not when the new entries need to be written. Retrofitting a pattern under pressure from a new feature is harder than establishing it while the motivation is still visible.

The unplanned item: `ActorType` moved from `io.casehub.ledger.api.model` to `io.casehub.platform.api.identity` in a SNAPSHOT update. Nine files. The compiler caught it as a missing symbol rather than a silent behaviour change, but the error didn't say where the type went. Filed a cross-repo audit issue before the sibling modules hit the same build failure.
