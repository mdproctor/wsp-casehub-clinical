---
layout: post
title: "Adverse event SLAs are simple. The wiring wasn't."
date: 2026-05-12
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [quarkus, casehub-ledger, casehub-work, adverse-events, jta, flyway]
---

Epic 4 is done. Report a Grade 3 adverse event and the system returns a `workItemId` with a `claimDeadline` 24 hours from `reportedAt`, and writes an `AdverseEventLedgerEntry` into the Merkle audit chain. Grade 5 gets an hour. Grade 1-2 carry a 7-day non-serious reporting window per ICH E6(R3) — correctly now. The `CtcaeGrade` enum was returning `Optional.empty()` for both grades; that got fixed first.

The domain service is about 90 lines. It sets `reportedAt` server-side, computes `slaDeadline` from `grade.sla().orElseThrow()`, creates a WorkItem with `claimDeadline = slaDeadline`, persists the entity, writes the ledger entry. Three infrastructure surprises on the way there.

I brought Claude in to handle execution. Three things came back unexpected.

## Three things casehub-ledger didn't advertise

**beans.xml is silently ignored.** `JpaLedgerEntryRepository` is `@Alternative @ApplicationScoped`. Standard CDI says put the class in `<alternatives>` in `beans.xml`. Quarkus ArC doesn't honour it. The application starts cleanly — no warning, no log entry — then CDI fails with "Unsatisfied dependency for type LedgerEntryRepository." We had to track down the actual fix through Quarkus docs: `quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository` in `application.properties`. Anyone else adding casehub-ledger will hit the same thing.

**Panache entities can't span two persistence units.** We put `AdverseEventLedgerEntry` (which extends `LedgerEntry`) in `io.casehub.clinical.entity` alongside the domain entities, then listed that package in both the default PU and the qhorus PU. Quarkus failed the build immediately: "Panache entities do not support being attached to several persistence units." The ledger subclass moved to `io.casehub.clinical.ledger` — a package listed only under qhorus. Clear constraint once you know it.

**V1004 was taken.** Our plan called for `V1004__ae_ledger_entry.sql` following the documented convention of "V1004+ for consumer-owned ledger joins." Claude inspected the casehub-ledger JAR and found `V1004__actor_identity.sql` already there. The convention had been written before casehub-ledger added that migration. The join table went to V1005.

## The XA problem

`reportAdverseEvent` is one `@Transactional` call that writes to two databases — the default datasource for the domain entity, qhorus for the ledger entry. In production, both are PostgreSQL and handle XA natively. In tests, both are H2.

Agroal's default local-transaction mode won't allow a second datasource to join an existing JTA transaction. The error reads like a connection pool problem: "Failed to enlist. Check if a connection from another datasource is already enlisted to the same transaction." It's a transaction mode mismatch. H2 supports XA; Agroal just doesn't use it unless told:

```properties
quarkus.datasource.jdbc.transactions=xa
quarkus.datasource.qhorus.jdbc.transactions=xa
```

The error message gives no indication this is the fix.

## The compliance gap, in one test

The 3-site oncology showcase now extends to safety events. Site A reports a Grade 3 adverse event; the test asserts `workItemId` is set and `slaDeadline` is within 24 hours. Site B reports Grade 5; deadline is within one hour. That's the structural compliance gap against ClinicalAgent demonstrated in code — a deadline the platform will escalate if it misses, backed by an independently verifiable Merkle chain.

No LLM reasoning is involved. The gap is structural: the presence of `claimDeadline` and the audit chain, not the quality of any safety assessment.
