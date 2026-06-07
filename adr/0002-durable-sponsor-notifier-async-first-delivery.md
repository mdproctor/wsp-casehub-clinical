# 0002 — DurableSponsorNotifier: async-first delivery via scheduler

Date: 2026-06-06
Status: Accepted

## Context and Problem Statement

`DurableSponsorNotifier.notify()` is called from `SponsorNotificationListener`, an
`@ObservesAsync @Transactional` CDI observer. The notifier must persist the notification
attempt and attempt delivery. The question is whether the first delivery attempt happens
synchronously inside `notify()` or is deferred to the scheduler.

## Decision Drivers

* `notify()` is called from an observer with an active DB transaction; a slow connector
  HTTP call (Slack, Teams, SMS) holds the Agroal connection open for its full duration
* Under multi-site load, multiple deviations resolve concurrently, producing concurrent
  `notify()` calls that compete for the same connection pool
* GCP ICH E6(R3) has no sub-minute requirement on sponsor notification timing;
  "as soon as practicable" is satisfied by a 60-second poll interval
* Fire-and-forget behaviour (`maxAttempts=1`) is a configuration preference, not a
  separate code path

## Considered Options

* **Option A: async-first** — `notify()` persists PENDING entity and returns;
  `SponsorNotificationRetryJob` polls for eligible notifications and delivers
* **Option B: sync first attempt** — `notify()` persists PENDING, attempts delivery in
  the same call, leaves entity in FAILED state if unsuccessful; scheduler handles
  retries only

## Decision Outcome

Chosen option: **Option A (async-first)**, because synchronous delivery inside `notify()`
holds an Agroal DB connection open for the duration of every connector HTTP call. Under
concurrent multi-site load this exhausts the connection pool. The 60-second first-attempt
latency is acceptable under GCP; the fire-and-forget case (`maxAttempts=1`) is expressed
via preference, not a separate implementation.

### Positive Consequences

* Agroal connection released immediately after entity persist — no pool contention during
  connector HTTP calls
* All delivery logic (first attempt + retries) in one code path
  (`SponsorNotificationDeliveryService`), not split across `notify()` and the job
* `notify()` is trivially testable: verify PENDING entity created, done

### Negative Consequences / Tradeoffs

* Up to poll-interval (default 60s) latency before first delivery attempt
* Showcase scenario observes a delay that the old `DefaultSponsorNotifier` (synchronous)
  did not have; documented in CLAUDE.md

## Pros and Cons of the Options

### Option A — async-first (chosen)

* ✅ No Agroal pressure under load — connection released after entity persist
* ✅ Single delivery code path — one place to fix, one place to test
* ✅ `notify()` is deterministic and fast
* ❌ Up to 60s latency on first attempt
* ❌ Showcase behaviour changed from immediate to deferred

### Option B — sync first attempt

* ✅ Immediate delivery for the happy path
* ✅ Matches old `DefaultSponsorNotifier` behaviour
* ❌ Holds Agroal connection during connector HTTP call
* ❌ Two delivery code paths: `notify()` for first attempt, job for retries
* ❌ Failure semantics differ between first attempt (inline) and retries (job)

## Links

* casehubio/clinical#21 — DurableSponsorNotifier implementation
* specs/2026-06-05-durable-sponsor-notifier-design.md — full design spec
