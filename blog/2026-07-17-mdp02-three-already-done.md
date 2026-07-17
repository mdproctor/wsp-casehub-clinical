---
layout: post
title: "Three Already Done and One That Wasn't"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [webui, dual-mode, work-item-inbox, blocks-ui]
---

The branch covered four issues: #125 (CSV field names), #126 (GDPR erasure dialog), #127 (demo mode fixes), and #123 (work queue inbox). I expected a session of small UI fixes. What I found was that three of the four were already resolved — the implementations had landed in previous sessions but the issues were never closed.

#125's field alignment (CSV `indStatus` → `regulatorySubmissionStatus`, adding `slaTimeRemainingHours` to `AdverseEventRow`) was complete. #126's confirmation dialog and receipt fields were wired exactly to spec, including the in-component state-based confirmation the spec explicitly chose over `<blocks-confirm-dialog>`. #127's JSONata expressions were already JSONata, and the CBR precedents panel already had demo mode.

What #127 *did* need was the CSV column alignment that the issue title buries after "JSONata expressions." Six mock CSV files used display-friendly column names (`enrolledCount`, `activeAeCount`, `targetCount`, `irbStatus`, `timestamp`) while the backend records used different names (`totalEnrolled`, `totalAdverseEvents`, `targetEnrollment`, `irbDecision`, `occurredAt`). In demo mode nobody notices — the CSV feeds the columns and everything matches. Switch `VITE_DEMO_MODE=false` and the columns go empty because the API JSON field names don't match the column IDs.

The fix was mechanical: rename CSV headers to match the Java records, update the `col()` and `groupBy()` references in the frontend views, and add `siteName` to `SiteRow` (populated from `investigatorId` — `TrialSite` doesn't have a name field yet, but the investigator ID is already being used as the display name in the AE and deviation endpoints). One near-miss: `ide_replace_text_in_file` with `enrolledCount` hit both the `TrialSummary` metric reference (correct rename to `totalEnrolled`) *and* the sites bar chart reference (wrong — sites use `enrolledCount` as their field name). Caught it before it shipped.

The real work was #123 — migrating the Work Queue from a pages-ui `table()` to the blocks-ui `<work-item-inbox>` component. The component needs `WorkIdentity` as a JS property (can't be set as an HTML attribute — complex object), and the previous blocker (blocks-ui#42, `WorkItemResponse` missing the `types` field) is now closed. The wiring: `loadSite()` completion callback sets the identity, provides demo data in DEMO_MODE, and registers a `work-item:selected` event handler that does type discrimination — `adverse-event` routes to Safety Workbench, `deviation-review` routes to Protocol Workbench via hash navigation (`#/page/Safety%20Workbench`).

The gap between "issue open" and "work done" is a recurring pattern. Issues get filed from design reviews, the implementation lands as part of a larger epic, and nobody goes back to close the review-filed issues individually. It's not a process failure worth fixing — the cost of hunting them down exceeds the cost of rediscovering they're done.
