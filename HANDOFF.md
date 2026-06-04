# Handoff — casehub-clinical
2026-06-04

## What happened this session

Swept the S/XS open issues. #51 was already done (CaseLifecycleEvent tenancyId update). Shipped #22 (`PATCH /trials/{id}/sponsor-config`, full-replace semantics, no migration) and #23 (`PiIdentityResolver` SPI — PI actor ID to formal name resolution for GCP-regulated sponsor notifications).

#23 required three spec review rounds. The key design insight: resolution belongs in `SponsorNotificationListener`, not `DefaultSponsorNotifier` — resolution failures get a distinct audit role (`sponsor-notifier-pi-resolver-failed`) from delivery failures. Resolved name flows into `SponsorNotificationRequest.piDisplayName` and into `ProtocolDeviationLedgerEntry.piDisplayName` (V1014 migration), closing a GCP compliance gap where the audit trail recorded delivery status but not which PI was notified.

Both commits landed on upstream/main. ARC42STORIES.MD stale scan cleared 5 closed issues from Active Risks / Technical Debt tables.

## Current state

- **Project repo:** `main` — pushed to fork + upstream
- **Workspace:** `main`

## Outstanding

- Workspace branch `epic-multi-site-sub-case` past deletion window (scaffold-only, EPIC-CLOSED.md present) — retained per policy, no action needed

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| AML migration | Bootstrap ARC42STORIES.MD for AML following clinical pattern | L | Med | arc42stories-spec.md gate now in place |
| Layer 7 | Trust routing | XL | High | Blocked on engine#387 (open) |
| #21 | DurableSponsorNotifier — entity + retry semantics + own ledger subtype | M | Med | `PiIdentityResolver` already wired in listener — notifier gets piDisplayName via request field |
