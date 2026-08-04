---
title: "What the Case Remembers"
date: 2026-08-04
entry_type: note
subtype: diary
type: phase-update
author: mdp
projects: [casehub-clinical]
tags: [cbr, feature-extraction, compaction, trust-routing]
---

# casehub-clinical — What the Case Remembers

**Date:** 2026-08-04
**Type:** phase-update

---

## What I was trying to achieve: richer precedent and leaner storage

CBR Phase 7 shipped the multi-scope DSMB memory. The CBR layer stores AE precedents, retrieves similar past cases, and adapts escalation plans from them. But the feature vector was blind to two things that materially affect how similar two AE cases really are: site context and agent reliability. A Grade 3 hepatotoxicity at a site with 8 enrolled patients and a safety agent with a 0.82 trust score is a fundamentally different situation from the same grade at a site with 200 patients and an agent in its bootstrap phase. The feature vector didn't know.

The second problem was volume. Every completed AE escalation stores a CBR case. Over time, the `clinical-ae` domain accumulates cases that are functionally identical — same grade, same event type, same trial phase, same unexpected/suspected flags — but with slightly different numeric values. The retention purge job handles TTL and count limits, but it doesn't reduce redundancy. You end up with fifty Grade 3 Neutropenia PHASE_III cases that all look the same to a retriever.

## What we believed going in: additive features and a batch job

The feature enrichment felt straightforward — add three numeric fields to `AeCbrFeatureBuilder`, update the schema, update the observer that stores cases. The compaction job felt like a smaller version of the retention purge: scan, group, merge, store.

Both turned out to be more involved than expected, but not in the ways I anticipated.

## The parameter explosion and AeCbrContext

`AeCbrFeatureBuilder.buildFeatures()` was a static method taking six parameters: the AE entity, enrollment, trial, safety review outcome, a boolean, and a prior-AE count. Adding three more (site enrollment count, target enrollment, agent trust score) would push it to nine. Nine parameters on a static utility method is the kind of API that breeds bugs silently — you swap two `long` values and nothing catches it until production.

We introduced `AeCbrContext` — a Java record that bundles all nine parameters with named fields. The builder methods become single-argument: `buildFeatures(AeCbrContext ctx)`. Callers construct the record with named fields, which is self-documenting and swap-proof. The record also carries through to `buildQueryFeatures`, `buildProblemSummary`, and `buildSolutionSummary` — a consistent API surface where there was previously a grab-bag of overloads.

## Where trust scores actually live

I expected the trust score lookup to be a Panache `find()` call. It isn't. `ActorTrustScore` is a regular JPA entity in the qhorus persistence unit — not a Panache entity, not in the default PU. The clean path is `ActorTrustScoreRepository.findCapabilityDimension(actorId, capabilityKey, dimensionKey)`, which returns `Optional<ActorTrustScore>` and handles the cross-PU plumbing internally. The safety-monitoring agent's trust score on the `safety-accuracy` dimension is the signal we capture — it reflects the routing confidence at the time the case completed.

When no trust score exists (agent still in bootstrap), we store 0.5 — the uninformative prior. This is deliberate: a brand-new agent hasn't earned trust or distrust. The CBR retriever will treat 0.5-trust cases as neutral evidence, which is exactly right.

## The merge key question

Compaction needed a definition of "similar enough to merge." I considered three approaches: configurable subset matching, similarity-threshold clustering via `retrieveSimilar`, and exact categorical key matching. The first adds config surface for marginal flexibility. The second depends on retrieval infrastructure being fast enough for batch scanning. The third is deterministic, simple, and cheap.

We went with exact categorical match on five features: grade, eventType, trialPhase, unexpected, suspected. Cases sharing all five are functionally identical from a clinical similarity perspective — the only differences are numeric (enrollment counts, trust scores) and outcome categoricals (safety review, DSMB escalation). Those differences get weighted-averaged and majority-voted respectively in the merged representative.

## Erase before store

The failure semantics of compaction matter. If you store the merged representative first and then erase the originals, a crash between the two operations leaves duplicates — the representative plus some originals. Duplicates pollute retrieval permanently. If you erase first and then store, a crash loses the group's data. But lost compaction candidates can be re-derived from future case ingestion. Duplicates cannot be un-derived.

We chose erase-before-store. Each group's operation is wrapped in try-catch so a failure on one group doesn't block others. The `mergeCount` field on the representative tracks how many original cases it represents — a compact case with `mergeCount=5` that gets re-compacted with two new singles produces a representative with `mergeCount=7`, and the weighted average respects the 5:1:1 ratio.

## What changed and why: FeatureValue naming mismatch

The schema DSL and the runtime types don't share names. `FeatureField.categorical()` declares a field in the schema, but at runtime the value type is `FeatureValue.StringVal`, not `FeatureValue.Categorical`. `FeatureField.numeric()` maps to `FeatureValue.NumberVal`, not `FeatureValue.Numeric`. Pattern matching on `FeatureValue.Numeric n` gives a compile error with no helpful message pointing you at `NumberVal`.

Claude caught the priorAeCount misclassification during code review — it was listed in the numeric averaging fields, but `bucketPriorAeCount()` stores it as a categorical string ("NONE", "ONE", "MULTIPLE"). The weighted average would have silently skipped it (not a `NumberVal`), which is functionally harmless but conceptually wrong. Moving it to the categorical majority-vote list fixed the intent.

## What I'd tell someone building a CBR compaction layer

Three things to know before you start. First, `CbrCaseMemoryStore.scan()` returns `CbrCaseSummary` — metadata only, no feature data. If you need features for grouping, use `retrieveSimilar()` with `minSimilarity(0.0)` and an empty feature map. Second, `ScoredCbrCase` puts the case object first in its constructor, not the ID — `new ScoredCbrCase<>(cbrCase, caseId, score)`. Third, design your merge key from the problem features, not the outcome features. Outcome features are what you're averaging — they vary within a group. Problem features are what define the group — they must be constant.

## Where this leaves the CBR roadmap

Epic #115 had seven phases, all shipped. The four follow-on issues are enhancements: #132 (feature enrichment) and #144 (compaction) are done. #145 (AE regrade capability) and #146 (DSMB WorkItem for batch signals) remain. The compaction job ships disabled by default — it activates when case volume warrants it, not before.
