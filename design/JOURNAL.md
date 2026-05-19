# Design Journal — issue-18-deviation-expiration-requires-new

### 2026-05-19 · §Key Architecture Decisions

DeviationExpirationJob's per-deviation isolation required a structural fix: the original single `@Transactional` batch loop with a try/catch was aspirationally correct but architecturally broken. Any JPA exception inside the loop marked the entire transaction rollback-only — the catch block's status reset also rolled back, silently undoing all previously-expired deviations. Extracting `DeviationExpirer` with `@Transactional(REQUIRES_NEW)` makes the isolation real: each deviation commits or rolls back independently.

Also surfaced: XA configuration is required in production `application.properties`, not just test properties. Any service writing to both datasources (default + qhorus) in a single transaction needs Agroal's XA mode in both environments. `ProtocolDeviationService`, `DeviationExpirer`, and `AdverseEventService` all write cross-datasource. This was a latent production gap now closed.
