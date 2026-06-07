# Handoff — casehub-clinical
2026-06-07

## What happened this session

Shipped #21 — `DurableSponsorNotifier`. Replaces `DefaultSponsorNotifier` (deleted) with a
durable implementation: `SponsorNotification` entity (V115, default datasource),
`SponsorNotificationLedgerEntry` (V2020, qhorus), retry policy via `SponsorNotificationRetryPolicy`
(SingleValuePreference), scheduler-based delivery via `SponsorNotificationDeliveryService` +
`SponsorNotificationRetryJob`. Key design choice: async-first delivery (notify() persists PENDING,
scheduler delivers) to avoid holding Agroal connections during connector HTTP calls under multi-site
load. 226 tests, 0 failures.

Code review found one critical bug: deviation ledger writes inside the connector try-catch caused
DELIVERED→FAILED regression after `store.markDelivered()` committed. Fixed by moving secondary
writes outside the try-catch with isolated try-catch-log, plus terminal-status guards on
`markFailed/Exhausted`. Garden entry GE-20260607-0bfc83, protocol PP-20260607-697a78.

Two snapshot regressions also fixed: `ledger_subject_sequence` missing (switched to
`InMemoryLedgerEntryRepository` in tests); qhorus `ChannelSlugValidator` rejecting UUID segments
(`dev-<UUID>` prefix fix). Both filed as clinical#63 and issues in garden.

`IrbDecisionListenerTest` fails due to `CallerRef.encode()` removed in new workadapter snapshot —
pre-existing regression, tracked as clinical#67.

## Current state

- **Project repo:** `main` — pushed to fork + upstream (squashed to 1 commit)
- **Workspace:** `main`

## Outstanding

- `IrbDecisionListenerTest` compile failure (casehubio/clinical#67) — pre-existing workadapter snapshot regression, not caused by #21
- Workspace branch `epic-multi-site-sub-case` past deletion window — retained per policy

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #67 | Fix IrbDecisionListenerTest — CallerRef.encode() removed in workadapter snapshot | XS | Low | Compile error; find new CallerRef API in workadapter source |
| Layer 7 | Trust routing | XL | High | Blocked on engine#387 (open) |
| #62 | Renumber qhorus migrations V1005–V1014 → V2000–V2009 (protocol compliance) | S | Low | Mechanical rename; requires Flyway repair or clean rebuild |
| #60 | WorkItem escalation on SponsorNotificationExhaustedEvent | S | Low | Extension point exists; wire a WorkItem on exhaustion |
