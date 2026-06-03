# Design Journal — issue-51-sx-sweep

### 2026-06-03 · §9.4·Domain Baseline

Added `PATCH /trials/{id}/sponsor-config` to `TrialResource` — full-replace semantics for sponsor notification config (`connectorId` + `destination`). Both fields nullable to allow clearing the config. No migration: columns were already present from clinical#13. `SponsorConfigRequest` record is a nested type in `TrialResource`, consistent with `RegisterTrialRequest`. Full-replace over partial-update: the two fields are co-dependent (a connector without a destination is non-functional) so atomically replacing both avoids half-configured state.
