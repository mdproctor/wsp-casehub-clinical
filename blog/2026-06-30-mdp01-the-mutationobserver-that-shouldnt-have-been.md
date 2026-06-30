---
layout: post
title: "The MutationObserver That Shouldn't Have Been"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [web-components, demo-ui, playwright]
---

The demo UI had a problem hiding in plain sight. Four pages — PI authorisation, SUSAR gate approval, Merkle verification, and the audit trail — all embedded JavaScript inside `html()` template literals. Browsers don't execute `<script>` tags inserted via `innerHTML`. The workaround was a `MutationObserver` in `index.ts` that watched for new DOM nodes, found script elements, and re-created them so they'd run.

It worked. It was also the wrong abstraction entirely.

The issue crystallised when I looked at what these scripts actually needed. Step 4 fetches the deviations list, finds a COMMANDED deviation, then enables an approval button. Step 7 reads an AE ID from `sessionStorage` — except step 5 uses `actionButton()`, which discards the response body. The AE ID was never stored. The code looked correct on both sides; the failure was silent. Step 8 makes a GET request and displays a structured result. The existing `actionButton()` helper can't do any of these things — it fires a POST, throws away the response, and refreshes datasets.

The right answer was custom web components. Three of them: `<clinical-pi-approval>`, `<clinical-susar-gate>`, `<clinical-merkle-verify>`. Light DOM, no Shadow DOM, no framework — vanilla `HTMLElement` subclasses with `connectedCallback` for init, `AbortController` for cleanup, and HTML attributes for configuration. They're registered in `index.ts` before `loadSite()`, embedded via `html()` with just the tag, and the MutationObserver is gone.

The `sessionStorage` dependency disappeared too. `ClinicalSusarGate` fetches the AE list directly and finds the first REQUESTED entry — same pattern as `ClinicalPiApproval` finding COMMANDED deviations. Navigation order no longer matters.

The design review surfaced a question I'd glossed over: why not extend `actionButton()` in casehub-pages to support response bodies? The answer is layering. PI approvals, SUSAR gates, and Merkle verification are clinical domain concepts. The pages DSL shouldn't know about them. The components belong in the application layer, and the generic capability gap — response body surfacing — is a separate issue for casehub-pages.

The other two items on this branch were straightforward. The sites list endpoint (`GET /trials/{trialId}/sites`) went on `TrialDashboardResource` — every enriched dashboard dataset lives there, and the two-hop join for per-site AE counts follows the exact same `enrollmentToSite` map pattern as the existing `adverseEvents()` method. Step 1's bar chart and sites table, previously TODO-commented, now render live data.

Playwright smoke tests round it out — 28 tests across page reachability, navigation, clipping at two viewports, and the full action flows (step 3→4 deviation→PI approval, step 5→7 AE→SUSAR gate, step 8 Merkle verify). The test infrastructure lives in `runtime/src/test/playwright/` with its own `package.json` to avoid bloating the production webui with Playwright's browser binaries.

The `actionButton()` response body gap is worth coming back to. Right now the clinical components handle it at the application layer — correct for domain-specific interactions. But a generic pages-level capability for "POST, read the response, display formatted output" would eliminate the need for custom components in simpler cases across all CaseHub harnesses.
