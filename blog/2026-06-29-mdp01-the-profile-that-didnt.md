---
title: "The Profile That Didn't"
date: 2026-06-29
author: mdp
type: diary
projects: [casehub-clinical]
tags: [quarkus, dev-mode, cdi, casehub-pages, demo-ui]
---

## The Profile That Didn't

The clinical demo had 14 pages of TypeScript UI, 7 backend endpoints, and a Quarkus app that had never once started outside of `@QuarkusTest`. Getting `mvn quarkus:dev` to run was supposed to be a config alignment task — diff the test and production `application.properties`, apply the missing `%dev.` entries, done.

It wasn't.

The first surprise was `casehub-engine-persistence-hibernate`. It brings `quarkus-hibernate-reactive-panache` transitively, which demands a reactive persistence unit. Dev mode runs JDBC-only H2 — no reactive PU exists. The engine's JPA stores are `@ApplicationScoped` and auto-discovered via Jandex, so they initialise at startup and immediately crash: `Mutiny.SessionFactory bean not found`.

The fix seemed obvious: select memory store alternatives in dev mode with `%dev.quarkus.arc.selected-alternatives`. Claude and I wrote it, deployed it, and got `UnsatisfiedResolutionException` — the memory beans existed but ArC didn't activate them. We tried removing the non-profiled value. Tried using only profiled values. Nothing.

It turns out `quarkus.arc.selected-alternatives` silently ignores `%dev.` profile overrides. No warning, no error, no log line. The non-profiled value is always used. Meanwhile, `quarkus.arc.exclude-types` *does* support profile overrides — same ArC processor, different behaviour.

The workaround: select everything in the non-profiled config (both JPA and memory stores), then carve out what doesn't belong per profile using `exclude-types`. Dev excludes reactive JPA; prod excludes memory stores. Each profile starts from the same selected set and removes what it can't use.

## Fourteen Pages, Zero Correct DSL Calls

With dev mode running, we turned to the UI. Every page was broken. The casehub-pages DSL had been used with the wrong calling conventions throughout — `columns()` received `{ span: 3 }` objects instead of a `number[]` distribution, `lookup()` got dataset objects instead of string IDs, `page()` had its `PageOptions` as the second argument when it must be the last, and `groupBy()` received `[]` instead of `null`.

Each fix revealed the next error. Column names didn't match the API (`susarOversightStatus` vs `escalationStatus`, `eventType` vs `entryType`). Fetch URLs had `/api/` prefixes from an earlier incarnation that no longer existed. And esbuild was minifying TypeScript constants inside template literal HTML strings — `${TRIAL_ID}` became `${Mr}` in the bundled output, landing inside single quotes where browsers treated it as literal text. Every action button's fetch URL was silently 404-ing.

The deepest issue was that `<script>` tags inside `html()` components never executed at all. The pages runtime uses `innerHTML` to render html content, and browsers don't run scripts added via innerHTML. Every action button, every governance fetch, every Merkle verification — all dead. A MutationObserver that re-creates script elements fixed the remaining html() scripts, and casehub-pages' new native `action-button` component replaced the worst offenders entirely.

## What Actually Shipped

The demo runs. All 14 pages render with live data — metrics pulling from real trial endpoints, tables with filtering and pagination, action buttons that POST to the backend and refresh datasets on success. The Merkle verification button works. The governance page shows a clean alert instead of hanging forever on "Loading governance context..."

The config alignment turned into a full stabilisation pass across config, CDI wiring, TypeScript DSL, API column mapping, fetch URL correction, and native component integration. Eight commits, 24 files changed, net negative 160 lines.
