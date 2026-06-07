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

- Workspace branch `epic-multi-site-sub-case` past deletion window — retained per policy

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| Layer 7 | Trust routing | XL | High | Blocked on engine#387 (open) |

*Updated: #67, #62, #60 closed — removed from backlog.*
