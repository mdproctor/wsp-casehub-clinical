# Handoff — casehub-clinical
2026-05-08

## What changed this session

Epics 1 and 2 complete. First code in the repo. Pushed to `mdproctor/clinical` main, issues #1 and #2 closed on `casehubio/clinical`.

**What was built:**
- `api/` — 11 enums (CtcaeGrade with CTCAE v5.0 SLAs, FHIR-aligned TrialPhase/EnrollmentStatus, etc.), ClinicalCapabilities + ClinicalTrustDimensions constants
- `runtime/` — 6 Panache Active Record entities, 6 Flyway migrations V1–V6, 3 REST resources (POST/GET trials, sites, patients), 17 tests
- CLAUDE.md — Name field, build commands, development workflow, external reference standards

**Key design decisions locked:**
- No POJO layer — JPA Active Record entities ARE the domain objects (platform rule exception documented in CLAUDE.md, not in PLATFORM.md)
- `quarkus-hibernate-validator` required explicitly — not bundled with `quarkus-rest`
- Two-phase Maven build for local dev: `mvn install -pl api` then `mvn test -pl runtime`
- UUID-suffix business keys in tests to prevent H2 shared-state conflicts

## Current state

```
clinical/
├── api/          casehub-clinical-api  (enums + constants)
└── runtime/      casehub-clinical      (entities, migrations, REST)
```

17 tests passing. 3-site showcase scenario (register trial → 3 sites → 3 patients → full verification) works end to end.

## What's next

**Epic 3: Multi-site sub-case structure** (casehubio/clinical#3)

Each trial site becomes a sub-case via `SubCase.builder().namespace(...).name(...).version(...)` with `waitForCompletion=false` (sites are long-running). The DSMB rollup binding on the parent trial case fires when safety signals accumulate across ≥ 2 sites — cross-site pattern detection from blackboard state.

Sub-case API is production-ready (casehubio/engine#195). Foundation gate: none.

Before starting Epic 3: fetch platform docs and run Platform Coherence Protocol per CLAUDE.md.

## References

- Spec: `specs/2026-05-07-epics-1-2-scaffold-domain-model-design.md`
- Plan: `plans/2026-05-07-epics-1-2-scaffold-domain-model.md`
- Blog: `blog/2026-05-08-mdp01-clinical-foundation.md`
- Platform docs: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`
- Engine sub-case: casehubio/engine#195 (closed — implementation done)
- Open epics: casehubio/clinical#3 through #10
