---
layout: post
title: "A deviation log isn't accountability"
date: 2026-05-15
type: day-zero
entry_type: note
subtype: diary
projects: [clinical]
---

Every clinical trial runs on a protocol — a precise specification approved by the IRB and, for IND trials, by the FDA. Every procedure, every measurement window, every patient selection criterion is spelled out and frozen at approval time. When execution departs from any of it, that's a protocol deviation.

Deviations happen in practice. A blood sample arrives outside its collection window. Equipment failure delays a required imaging scan. A marginal eligibility criterion gets interpreted differently at different sites. These aren't automatically disqualifying — what matters is whether they're acknowledged, documented, and assessed for impact on data integrity. ICH E6(R3) is explicit: a named Principal Investigator must take formal responsibility for deviations at their site. Not a system log entry. A person, on record, who has reviewed what happened and committed to a response.

ClinicalAgent — the peer-reviewed baseline I'm building against — logs deviations. The agent detects a departure from protocol and records it. There's no named PI, no formal commitment, no deadline. In a GCP audit, that log entry proves the deviation was noticed. It doesn't prove anyone was accountable for it. Those are different things.

The gap isn't technical sophistication. An agent can log a deviation with perfect precision and still satisfy none of the GCP requirement, because the requirement is about accountability — a named person making a formal commitment with a traceable record.

CaseHub's COMMAND lifecycle was built for this pattern. A COMMAND issued to a named participant creates a formal obligation with a deadline. The system tracks whether it's been acknowledged, acted on, and resolved. What the clinical layer needs is the domain binding: a `ProtocolDeviation` entity, a case plan binding that fires when an agent classifies a departure, and a COMMAND routed to the PI responsible for that site.

One thing I want to get right before building: some deviations affect only the site where they occur, and the PI's acknowledgement closes them. Others — significant enough to require protocol amendment — may need to go to the sponsor and back to the IRB. Those carry different deadlines and different responsible parties. The COMMAND routing isn't uniform. Getting the classification right before writing a line of code matters more than moving fast.
