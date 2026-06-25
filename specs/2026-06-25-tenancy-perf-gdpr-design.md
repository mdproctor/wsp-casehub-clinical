# Design: #89 MissingTenancyExceptionMapper + #87 Listener TX Fix + #79 GDPR Erasure

**Branch:** issue-89-tenancy-perf-gdpr
**Covers:** casehubio/clinical#89, #87, #79
**Date:** 2026-06-25

---

## §1 — #89: MissingTenancyExceptionMapper

### Context

Platform#111 shipped `io.casehub.platform.api.identity.MissingTenancyException` (extends `RuntimeException`, carries `actorId()`). `OidcCurrentPrincipal.tenancyId()` throws it when the JWT lacks the `tenancyId` claim. Without an ExceptionMapper, RESTEasy maps it to HTTP 500 — wrong semantics for a client error.

The issue was filed as `MissingTenancyClaimException` — the actual class name is `MissingTenancyException`.

### Design

**Class:** `io.casehub.clinical.resource.MissingTenancyExceptionMapper implements ExceptionMapper<MissingTenancyException>`

**Location:** `runtime/src/main/java/io/casehub/clinical/resource/`

**Behavior:** HTTP 400 with JSON body:
```json
{
  "error": "missing_tenancy_claim",
  "message": "JWT does not contain required tenancyId claim",
  "actorId": "<from exception>"
}
```

**Test:** `MissingTenancyExceptionMapperTest` — unit test, no Quarkus CDI. Construct mapper, pass exception, assert 400 + JSON body.

### Follow-up

File on `casehubio/platform`: provide this mapper from `casehub-platform-oidc` so consumers don't duplicate it. PLATFORM.md Step 6.

---

## §2 — #87: CaseLifecycleEvent Listener Transaction Fix

### Root Cause

`CaseLifecycleEvent` is a thin record — `(caseId, tenancyId, commandType, eventType, caseStatus, actorId, actorRole, traceId)`. No case context. All three listeners must query the reactive `CaseInstanceRepository` (`Uni<T>`) to discriminate and extract data. They do this inside `@Transactional`, holding a JDBC connection from the Agroal pool during `.await().atMost(5s)`. Under sustained load, this exhausts the pool.

### Design

**Principle:** Remove `@Transactional` from all three observer methods. The reactive `.await()` call runs without holding a JDBC connection. Each listener's writes manage their own transaction boundaries.

#### SusarOversightListener

Remove `@Transactional`. The only write is `statusUpdater.markCompleted(aeId)` — already `REQUIRES_NEW`. No other changes.

#### AeEscalationListener

Remove `@Transactional` from `onCaseLifecycle()`.

Add `@Transactional(REQUIRES_NEW)` to `AeEscalationLedgerWriter`:
- `writeCompletionEntry()`
- `writeObserverFailureEntry()`

Preserves the FDA gap design: `statusUpdater.markCompleted()` commits first (REQUIRES_NEW), then `writeCompletionEntry()` commits independently (now also REQUIRES_NEW). If ledger write fails, `writeObserverFailureEntry()` runs in yet another REQUIRES_NEW. Each step is independently durable.

`memoryService.storeAeOutcome()` and `completedEvents.fireAsync()` don't need a transaction. Memory store is best-effort (try/catch). No behavior change.

#### ProtocolAmendmentListener

Remove `@Transactional` from `onCaseLifecycle()`.

**New class: `ProtocolAmendmentStatusUpdater`** — `@ApplicationScoped`:
- `@Transactional(REQUIRES_NEW) public void applyRecommendation(UUID amendmentId, String recommendation)`
- Loads `ProtocolAmendment.findById(amendmentId)`
- Idempotency guard: `supervisorRecommendation != null` → return
- Applies status/recommendation fields per the switch logic
- Calls `ledgerWriter.writeResolutionEntry(amendment)` — participates in the same REQUIRES_NEW transaction
- Entity state updates and ledger write commit atomically

`ProtocolAmendmentLedgerWriter` — stays WITHOUT `@Transactional` on its methods. It participates in whatever transaction the caller provides. This is a deliberate design difference from `AeEscalationLedgerWriter`: the amendment entity writes and ledger write must be *atomic*, so they share a transaction. The AE design *intentionally separates* status from ledger (FDA gap handling).

#### Listener simplification

After extracting write logic to the updater, `ProtocolAmendmentListener.onCaseLifecycle()` becomes:
1. Filter by eventType (GoalReached/CaseCompleted)
2. Reactive query for case instance (no TX held)
3. Discriminate by `amendmentId` in context
4. Extract `advisorRecommendation` from context
5. Call `statusUpdater.applyRecommendation(amendmentId, recommendation)`

**`default` case (unknown recommendation):** stays in the listener, NOT in the updater. It deliberately does NOT set `supervisorRecommendation` (allowing redelivery) and only sets `amendmentCaseStatus = FAILED`. This is a direct Panache write that needs a transaction — wrap it in `QuarkusTransaction.requiringNew().run(...)` inline, since it's a single-field error-path write that doesn't warrant a separate service method.

### Test changes

- `AeEscalationLedgerWriterTest` — unit test with mocks, add verification that REQUIRES_NEW is declared (annotation assertion)
- `ProtocolAmendmentListenerTest` — existing tests call listener directly. Add `ProtocolAmendmentStatusUpdaterTest` for the write path.
- Integration tests — REQUIRES_NEW methods manage their own transactions. No structural test changes needed.

### Follow-up

File on `casehubio/engine`: "Enrich `CaseLifecycleEvent` with case context snapshot — eliminates reactive `CaseInstanceRepository` query in `@ObservesAsync` listeners."

---

## §3 — #79: GDPR Art.17 Patient-Scoped Erasure

### Context

`ConsentWithdrawalService.withdraw(enrollmentId, tenantId)` already does full GDPR erasure for a single enrollment: pseudonymizes `patientId`, writes `ConsentWithdrawalLedgerEntry`, calls `LedgerErasureService.erase()`, erases case memories. `PatientResource.withdrawConsent()` exposes this as a REST endpoint.

Gaps:
1. Enrollment-scoped only — no patient-level "erase all my data" endpoint
2. Foundation `ErasureReceiptLedgerEntry` not enabled (`casehub.ledger.erasure-receipt.enabled` defaults to `false`)
3. Flyway path for ledger V1010 migration not in clinical's qhorus locations
4. No traceability link between domain receipt (`ConsentWithdrawalLedgerEntry`) and foundation receipt (`ErasureReceiptLedgerEntry`)

### Config Changes

**Production `application.properties`:**
```properties
casehub.ledger.erasure-receipt.enabled=true
```

Add `classpath:db/ledger/migration` to qhorus Flyway locations:
```properties
quarkus.flyway.qhorus.locations=classpath:db/migration,classpath:db/migration/qhorus,classpath:db/engine-ledger/migration,classpath:db/ledger/migration
```

**Test `application.properties`:**
```properties
casehub.ledger.erasure-receipt.enabled=true
```
No Flyway path change — `drop-and-create` handles table creation.

### GdprErasureService

**Class:** `io.casehub.clinical.service.GdprErasureService` — `@ApplicationScoped`

**Method:** `@Transactional public int erasePatient(String patientId, String tenantId)`

1. Find all non-withdrawn enrollments: `PatientEnrollment.find("patientId = ?1 AND tenantId = ?2 AND consentStatus != ?3", patientId, tenantId, ConsentStatus.WITHDRAWN).list()`
2. If empty → throw `PatientNotFoundException(patientId)`
3. For each enrollment: `consentWithdrawalService.withdraw(enrollment.id, tenantId)`
4. Return count

Critical: step 1 completes before any withdrawal starts. After the first `withdraw()`, that enrollment's `patientId` becomes `"erased-<UUID>"`, but since we already have the full list, the lookup is not affected. All withdrawals participate in the outer XA transaction (REQUIRED propagation) — atomic commit.

### GdprErasureResource

**Class:** `io.casehub.clinical.resource.GdprErasureResource`
- `@Path("/api/gdpr/erasure")`
- `@ApplicationScoped`

**Endpoint:** `@DELETE @Path("/patients/{patientId}") @RolesAllowed({SPONSOR, COORDINATOR})`

- 204 No Content on success (header `X-Enrollments-Erased: N`)
- 404 if no non-withdrawn enrollments found

**RBAC:** SPONSOR/COORDINATOR (cross-site authority). The per-enrollment `withdrawConsent` remains INVESTIGATOR (site-level).

### PatientNotFoundException

`io.casehub.clinical.service.PatientNotFoundException` — extends `RuntimeException`, carries `patientId`. Follows `PatientEnrollmentNotFoundException` pattern.

### receiptEntryId Traceability

Add to `ConsentWithdrawalLedgerEntry`:
```java
@Column(name = "receipt_entry_id")
public UUID receiptEntryId;
```

Migration: `V2023__consent_withdrawal_receipt_entry_id.sql` in `db/migration/qhorus/`:
```sql
ALTER TABLE consent_withdrawal_ledger_entry ADD COLUMN receipt_entry_id UUID;
```

Update `ConsentWithdrawalService.withdraw()`:
```java
entry.receiptEntryId = erasureResult.receiptEntryId().orElse(null);
```

Update `domainContentBytes()` to include `receiptEntryId`.

### Test Plan

- `GdprErasureServiceTest` — unit test with mocked `ConsentWithdrawalService`. Cases: single enrollment, multiple enrollments across sites/trials, no enrollments found.
- `GdprErasureResourceTest` — `@QuarkusTest` + `@TestSecurity`. Cases: 204 success, 404 unknown patient, RBAC boundary (INVESTIGATOR denied, SPONSOR/COORDINATOR allowed).
- `ConsentWithdrawalServiceTest` — add test verifying `receiptEntryId` is set when `erasure-receipt.enabled=true`.

---

## CLAUDE.md Updates

- Update issue #89 references from `MissingTenancyClaimException` to `MissingTenancyException`
- Update "19 REST endpoints" count to 20 (new `GdprErasureResource`)
- Document `classpath:db/ledger/migration` addition in Flyway migration structure
- Note SPONSOR/COORDINATOR RBAC scope on erasure endpoint

## Issues to File Before Implementation

1. `casehubio/platform`: "Provide MissingTenancyExceptionMapper from casehub-platform-oidc"
2. `casehubio/engine`: "Enrich CaseLifecycleEvent with case context snapshot"
