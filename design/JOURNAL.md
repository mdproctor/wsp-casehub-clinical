# Design Journal — issue-51-sx-sweep

### 2026-06-04 · §9.4·Domain Baseline

`PiIdentityResolver` SPI in `api/spi/` — resolves PI actor IDs to formal names for GCP-regulated sponsor notifications. Resolution happens in `SponsorNotificationListener` (not the notifier), so resolution failures get a distinct audit role (`sponsor-notifier-pi-resolver-failed`) from delivery failures (`sponsor-notifier-observer-failed`). Resolved name flows into `SponsorNotificationRequest.piDisplayName` and is recorded in `ProtocolDeviationLedgerEntry.piDisplayName` (V1014 migration), closing the GCP compliance gap where the audit trail recorded delivery status but not which PI was notified or by what name. Default implementation is a passthrough; production deployments override. Null-return from resolver is treated as a contract violation — same halt path as throw.

### 2026-06-03 · §9.4·Domain Baseline

Added `PATCH /trials/{id}/sponsor-config` to `TrialResource` — full-replace semantics for sponsor notification config (`connectorId` + `destination`). Both fields nullable to allow clearing the config. No migration: columns were already present from clinical#13. `SponsorConfigRequest` record is a nested type in `TrialResource`, consistent with `RegisterTrialRequest`. Full-replace over partial-update: the two fields are co-dependent (a connector without a destination is non-functional) so atomically replacing both avoids half-configured state.
