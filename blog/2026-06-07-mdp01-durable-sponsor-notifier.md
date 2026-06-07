# The Durable Notifier

The last session shipped `PiIdentityResolver` — the SPI that resolves PI actor IDs to formal names so that sponsor notifications carry a real name instead of a system ID. At the time, the underlying notifier was still `DefaultSponsorNotifier`: fire-and-forget via a Slack connector, with a deviation ledger entry and that's it. Good enough for the showcase. Not good enough for production.

This session closed that gap.

## The design question that took the most rounds

Five rounds of spec review. The fundamental issue wasn't the implementation — it was the transaction model. We're calling `notify()` from an `@ObservesAsync @Transactional` listener. The listener has an active database connection when `notify()` fires. If we attempt connector delivery synchronously inside `notify()`, that connection stays open across the HTTP call to Slack or Teams. Under multi-site load, with dozens of deviations resolving concurrently, you exhaust the Agroal pool.

So we went async-first: `notify()` persists a PENDING entity and returns immediately. A scheduler picks up eligible notifications and delivers. The connector call happens with no database connection held.

The tradeoff is a delay — up to the poll interval, default 60 seconds — before first delivery. For GCP compliance this is fine. ICH E6(R3) says "as soon as practicable", not "within 30 seconds". The showcase changes behaviour — the old `DefaultSponsorNotifier` delivered synchronously — but correctness beats showcase fidelity.

## The bug that came out of code review

The reviewer caught one that would have caused duplicate delivery in production.

We had the deviation ledger write — `deviationLedgerWriter.writeSponsorNotifiedEntry()` — inside the connector try-catch. Here's why that's wrong: `store.markDelivered()` runs in its own `REQUIRES_NEW` transaction. That transaction commits when the method returns. If the deviation ledger write then throws, the catch block calls `store.markFailed()` — another `REQUIRES_NEW` that overwrites DELIVERED with FAILED. The scheduler picks it up, delivers again, and the sponsor gets a duplicate notification.

The fix: move deviation ledger writes outside the try-catch, with their own isolated try-catch-log that records the failure without touching entity state. Add a terminal-status guard to `markFailed()` and `markExhausted()` — if the entity is already DELIVERED or EXHAUSTED, return immediately. Belt and suspenders.

This ended up as a garden entry (`GE-20260607-0bfc83`) and a protocol (`PP-20260607-697a78`). The pattern generalises: any time you have a committed REQUIRES_NEW followed by secondary audit writes, those secondary writes must be outside the primary try-catch. Obvious in hindsight.

## What the snapshot changes broke

Midway through the session, the casehub-ledger snapshot updated with a `LedgerSequenceAllocator`. This is a native SQL `MERGE INTO ledger_subject_sequence` — not a Hibernate entity, not created by `drop-and-create`. Every test that actually wrote to the ledger started failing with "Table LEDGER_SUBJECT_SEQUENCE not found".

The fix: switch from `JpaLedgerEntryRepository` to `InMemoryLedgerEntryRepository` in test `selected-alternatives`. The engine's own tests have always done this; downstream consumers hadn't needed to until now. Took some digging to connect "SQL error deep in the ledger internals" with "switch your selected alternative" — the stack trace doesn't help you there.

The qhorus snapshot also tightened `ChannelSlugValidator` to reject UUID channel name segments. Our PI oversight channels were named `clinical/deviation/<UUID>/pi-oversight`. A UUID starts with a hex digit, which fails `[a-z]`. We prefixed with `dev-` to satisfy the validator and updated `PiResponseListener.CHANNEL_PATTERN` to match.

Both are now garden entries. Neither is the kind of thing you find in the docs.

## Preferences, not framework coupling

One question I wanted to get right: how do you make the retry policy configurable without coupling to jackson-datatype-jsr310 or some complex preference format?

The answer is the simplest possible: the preference value is a plain string `"maxAttempts,retryIntervalMinutes"` — e.g. `"3,30"`. The `PreferenceKey<T>` constructor takes a parser function that turns this string into a typed `SponsorNotificationRetryPolicy` record. No Duration parsing, no JSON, no extra dependencies.

`maxAttempts=1` gives you fire-and-forget semantics — one attempt, on failure the entity goes straight to EXHAUSTED, `SponsorNotificationExhaustedEvent` fires. The old `DefaultSponsorNotifier` (now deleted) had no audit trail; `maxAttempts=1` gives you identical delivery behaviour with a full per-attempt ledger chain.

One thing I didn't realise until I tried it: `Preferences` is not a CDI bean. You can't `@Inject Preferences`. The pattern is to inject `PreferenceProvider` and call `preferenceProvider.resolve(SettingsScope.of("casehubio", "clinical")).getOrDefault(KEY)`. The name makes it sound like a service, but it's a value factory. That's also a garden entry now.

## What's left

Layer 7 — trust routing — is still the stub. The next substantive issue after the bug cleanup (casehubio/clinical#63 for the qhorus channel slug regression) is probably the trust infrastructure.

The durable notifier also deferred three things deliberately: WorkItem escalation when all retries are exhausted (#60), exponential backoff (#61), and renumbering the qhorus ledger migrations from V1005 to V2000+ to match the platform protocol (#62). All filed, none urgent for the showcase.
