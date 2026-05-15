# Handoff — casehub-clinical
2026-05-15

## What changed this session

Short setup session. Cleaned up orphaned `design/.meta` + `design/JOURNAL.md` left on workspace `main` from Epic 4's fast-forward merge. Started Epic 5: both repos branched to `epic-protocol-deviation-pi-auth`, workspace scaffold committed, linked to casehubio/clinical#5.

Blog entry added: `blog/2026-05-15-mdp01-protocol-deviation-accountability.md` — Day Zero framing for PI authorisation, covering the GCP accountability gap vs ClinicalAgent.

## Current state

Both repos on `epic-protocol-deviation-pi-auth`. No code written yet — brainstorm is the next step.

```
design/.meta   epic: epic-protocol-deviation-pi-auth, issue: 5
design/JOURNAL.md   stub only
```

## Open design question

COMMAND routing for deviations isn't uniform: site-level deviations go to the site PI; protocol-significant deviations may also require sponsor notification with different deadlines and responsible parties. Classification needs to be resolved in the brainstorm before implementation.

## What's next

1. **Brainstorm** — invoke `superpowers:brainstorming` at the start of next session to design Epic 5 before writing any code
2. Run `work-start` first for platform coherence check

## References

- Blog: `blog/2026-05-15-mdp01-protocol-deviation-accountability.md`
- Epic 5 issue: casehubio/clinical#5 (open)
- Engine sub-case tracking: casehubio/engine#112 (open — Epic 3 still blocked)
- Previous session detail: `git show HEAD~3:HANDOFF.md`
