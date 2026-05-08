---
layout: post
title: "Laying the clinical domain model"
date: 2026-05-08
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [quarkus, domain-model, clinical-trials]
---

Starting a new Quarkus application from scratch means making a lot of
small decisions before writing any domain logic. casehub-clinical is the
clinical trial coordination system I've been planning as CaseHub's first
regulated-domain application — the one that demonstrates what the engine
is actually for.

## The module split that wasn't

The CaseHub convention is `api/` (pure Java, no JPA) plus `runtime/` (Quarkus
app, entities, migrations). The reason: JPA entity classes force datasource
config onto every downstream consumer. casehub-clinical has no downstream
consumers — it's an application tier — so there's a case for skipping the
split entirely and using Panache Active Record entities as the domain objects.

I added an exception to the platform protocol documenting exactly this. Then
reverted it.

The rule exists because the problem it prevents is real, and a general-purpose
exception invites drift. The better approach: document the reasoning in
casehub-clinical's own CLAUDE.md and keep the platform rule clean. So `api/`
has enums and constants only — `CtcaeGrade`, `TrialPhase`, `EnrollmentStatus`,
and the capability tag strings the engine needs. The JPA entities in `runtime/`
are the domain objects.

## What FHIR told us about our own model

Before writing entities we validated the field definitions against FHIR R5 —
ResearchStudy, ResearchSubject, and AdverseEvent. FHIR doesn't define CTCAE
grading (it defers to implementation guides for that), but it surfaced three
gaps in what I'd planned:

`targetEnrollment` was missing from `ClinicalTrial`. FHIR has this as
`recruitment.targetNumber` — you need it to know when the `enrollment-complete`
goal is met.

`EnrollmentStatus` needed to be richer than a simple `ConsentStatus`. FHIR's
ResearchSubject has a full state machine: CANDIDATE → SCREENING → ELIGIBLE
→ ENROLLED → ON_STUDY → OFF_STUDY → WITHDRAWN. Consent and participation
status are separate things.

`AdverseEvent` needed two timing fields, not one. `occurredAt` is when the
event happened; `reportedAt` is when it was documented. GCP tracks both.

`CtcaeGrade` needed careful sourcing. Grades 3 and 4 trigger 24-hour
expedited reporting to the sponsor, per ICH E6(R3) §5.17. Grade 5 (death)
uses a 1-hour internal SLA — stricter than the ICH minimum. That's product
policy, not a GCP requirement, and the Javadoc says so explicitly.

## Two things Quarkus didn't volunteer

`quarkus-rest` doesn't include Bean Validation. `@NotBlank`, `@NotNull`,
`@Valid` — all compiled, all wired, all silently did nothing. A missing
required field returned 200. Adding `quarkus-hibernate-validator` to the
pom fixed it. Worth knowing before you spend time chasing phantom validation
failures.

In a greenfield multi-module repo with no sources in `api/` yet, `mvn compile
-pl api,runtime` crashes with `NoSuchFileException`. Quarkus's `generate-code`
goal on `runtime/` scans `api/target/classes` at startup to detect extensions —
before the Java compiler has run. The fix is a two-phase build: `mvn install
-pl api` first, then proceed. It resolves itself once `api/` has real sources.

## The gap the tests missed

The review caught something the TDD cycle hadn't. `POST
/trials/{t}/sites/{s}/patients` validated that the site belonged to the given
trial before enrolling. `GET .../patients/{id}` only checked that the
enrollment belonged to the given siteId — not that the site belonged to the
given trialId. A caller with a valid siteId could reach any patient. One extra
lookup fixed it, but it's the kind of ownership chain asymmetry that goes
unnoticed unless someone specifically looks for it.

Six entities, six Flyway migrations, three REST resources. A 3-site showcase
scenario registers a trial, adds independent sites, enrolls patients, and
verifies all state. The domain layer is ready; the engine wiring comes next.
