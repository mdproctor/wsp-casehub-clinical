---
layout: post
title: "The Substring That Ate Sortable"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [snapshot-sync, intellij-mcp, pages-ui]
---

`table` was renamed to `dataTable` in `@casehubio/pages-ui`. A routine rename across four view files — except `ide_replace_text_in_file` does substring matching, not word matching. `sortable` became `sordataTable` in every file it touched.

The tool has no `wholeWord` parameter. Its sibling `ide_search_text` does, which creates an expectation that replacement works the same way. It doesn't. The replacement count comes back higher than expected, but there's no warning — you only catch it by reading the output.

The broader fix session was mechanical: syncing clinical with current platform and engine SNAPSHOTs. The three items from the original issue — `GroupMembershipProvider.membersOf()` gaining a `tenancyId` parameter, `CaseHub.startCase()` return type changing, CBR constructor updates — had already been fixed in prior commits. What remained was the frontend: `table` to `dataTable`, `emptyMessage` dropped from `DataTableProps`, and `inputSchema` deprecated in favour of `inputProjection` in the engine's YAML capability schema.

The `inputSchema` rename is worth noting because it's embedded in CLAUDE.md's ecosystem conventions section — a reference to "define `spec.capabilities[- name/inputSchema]`" that silently becomes wrong when the engine SNAPSHOT ships. The kind of drift that costs someone thirty minutes of debugging six months from now.

The TypeScript side also had pre-existing gaps: `?raw` CSV imports and `import.meta.env` lacked type declarations because the project uses esbuild, not Vite. A small `env.d.ts` closes all twelve of those errors permanently.

The substring gotcha went into the garden. It's a clean example of a tool that works correctly by its own definition but not by the user's expectation — and the gap between those two things is where the damage happens.
