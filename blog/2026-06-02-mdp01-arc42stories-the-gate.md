---
layout: post
title: "ARC42STORIES.MD: the gate, not the advice"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [clinical]
tags: [arc42stories, write-content, documentation, casehub]
---

Before generating a single line of `ARC42STORIES.MD` for clinical, I wanted to understand whether the arc42stories spec actually enforced the write-content modes — or just mentioned them.

The spec's Writing Style section said: "Constraint sets for each mode live in the `write-content` skill (`modes/` directory). When using the skill, it loads the correct mode file automatically. When writing without the skill, apply the structural rules below directly."

That's advisory. A generator reads it, registers where the files live, and proceeds without loading them. The phrase "apply the structural rules below directly" actively gives permission to skip loading entirely. Without the files in context, wrong-mode generation produces plausible-looking output: prose where tables should be, hedging where direct statements are required. The failure is invisible.

I had Claude load all eight write-content files before generating a word — `voice/anti-slop.md`, `voice/mandatory-rules.md`, `forms/technical-documentation.md`, and all five mode files. Then we added a `### Generator pre-conditions` section to the spec that names each file explicitly with "do not generate any content until all are loaded." The gate is what the advisory was pretending to be.

The clinical document has six complete layers, five chapters, and eight anti-patterns in §8. What made it more interesting to write than devtown is what it doesn't have: no `@DefaultBean` displacement pattern, no three-tier module structure, no port interfaces in `api/`. Clinical uses Panache Active Record entities directly as domain objects — no downstream JPA consumers exist, so the whole apparatus of CDI displacement is unnecessary. Devtown's `## Solution Strategy` section is built around that displacement pattern. Clinical's needed to explain the exception instead.

The other structural difference: Layer 6 in clinical is trial-level blackboard aggregation — a domain-specific layer not in the standard CaseHub Foundation taxonomy, which names Layer 6 as trust routing. Trust routing becomes Layer 7 for clinical. The chapter structure had to accommodate an extra layer that wasn't in the profile.

We ran the three post-generation quality checks. One was productive: issue references in §12 Active Risks. One issue was closed — a documentation item about the Grade 5 regulatory gap, not the gap itself. We removed it from Active Risks and noted the gap against the Layer 7 stub instead.

The generation also surfaced a spec problem: a merge commit had duplicated the entire `## Writing Style` section. Clinical was the first document generated under the revised spec, which is why we caught it.
