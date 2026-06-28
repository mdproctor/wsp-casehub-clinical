# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

## Data entry application UI

**Logged:** 2026-06-28

The demo UI (Epic #93) shows a pre-seeded trial with guided action buttons — a showcase. A separate epic could add form-based data entry pages so someone installing casehub-clinical has a functional application: trial registration, site/patient enrollment, AE reporting, PI response, IRB decision, safety officer WorkItem management. The REST API already supports all of these. casehub-pages has form components (textInput, dropdown, datePicker, dataScope, REST save adapters). The work is wiring forms to existing endpoints. Significant data volume needed to get something useful — trial setup alone requires protocol, phase, sponsor, sites with investigators, then patients with consent. Worth scoping carefully before committing.
