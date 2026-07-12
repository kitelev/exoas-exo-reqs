---
exo__Asset_uid: b2a33efc-1b6c-4c9a-ab76-ecff66ffab08
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
exo__Asset_createdAt: 2026-07-12T19:37:07+05:00
exo__Asset_updatedAt: 2026-07-12T19:39:20+05:00
exo__Asset_createdBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253|req__Requirement]]"
exo__Asset_label: "req(exo): a pn__DailyNote exo__Layout can render a 'Closed today' section — a new 'closed' daily-efforts partition listing efforts whose ems__Effort_resolutionTimestamp (or ems__Effort_endTimestamp) falls on the note's day (so Trashed-only closures surface, not just Done), computed local-tz, additive to the existing Actions/Tasks/Projects class partitions with zero regression, homoiconic (Layout+Block are vault assets), desktop+mobile"
req__Requirement_status: "[[4bd932c2-2507-4a2d-b3f2-163e096bfa81|req__RequirementStatusApproved]]"
req__Requirement_approvedBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_approvedAt: 2026-07-12T19:39:20+05:00
req__Requirement_priority: "[[3c28e0c7-7cf8-4041-9230-d37a04d3981e|req__RequirementPriorityP3]]"
req__Requirement_bindingClass:
  - "[[aefdb1a3-ae87-4d4e-93fb-fb989a243c0c|req__RequirementBindingClassComponent]]"
  - "[[cc677c5f-b4ac-4ec3-9baa-e7ab79b78113|req__RequirementBindingClassUnit]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers:
  - "GitHub issue #3781 (LOW/QoL). A pn__DailyNote has no 'closed today' feed: tasks closed on a given day are not surfaced on that day's note, so the note can't be used for daily review of what got done. The homoiconic daily-note feed already exists (exo__Layout daily-efforts-by-class block, req a38ac95b) but (i) its provider filters via isEffortInDay, which matches start/end/plannedStart/plannedEnd timestamps but NOT ems__Effort_resolutionTimestamp — so a TRASHED-only closure (DefaultWorkflows sets ONLY resolutionTimestamp on Trashed; DONE sets endTimestamp+resolutionTimestamp) is invisible on the daily note; and (ii) the block's partitions are class-based (actions/tasks/projects), so there is no 'closed today' axis. This requirement adds a fourth daily-efforts partition, 'closed', to the LayoutBlock model + the exo__Layout render pipeline: a daily-efforts-by-class block with partition 'closed' lists the efforts whose ems__Effort_resolutionTimestamp OR ems__Effort_endTimestamp falls on the note's pn__DailyNote_day (the '(fallback endTimestamp)' of the issue AC). The composition and visibility of the Closed block live in the exo__Layout RDF asset + per-note frontmatter (override key pn__DailyNote_showClosed), never in plugin code or plugin settings — the Layout is user-configurable homoiconic semantics (Homoiconicity Invariant, Q1=yes; same treatment as the a38ac95b Actions/Tasks/Projects blocks). The date-match ALGORITHM (which timestamp is a closure + which day it belongs to) stays in the engine as a core-processing concern (Homoiconicity Q3 exception a) — matching the existing isEffortInDay precedent — because it MUST compute the note's LOCAL day interval: a store-side SPARQL day-filter (STRSTARTS on the UTC xsd:dateTime literal) is technically feasible on the clean timestamp literals (the #3805 dual-IRI налог applies to class/enum IRIs, not timestamp literals) but would match the UTC day, off-by-tz from the note's local day in the ~5h window near local midnight; the TS interval-match is tz-correct. So AC2 'exo__Layout + SPARQL / no TS hardcode' is satisfied in the sense that the feed IS a vault-asset Layout+Block (homoiconic), while the closure date-match is engine core-algorithm for local-tz correctness (documented deviation from the issue's literal 'SPARQL block', approved by the orchestrator under Andrey's authority). The new 'closed' axis is ADDITIVE and orthogonal to the class partitions: the collector broadens to collect efforts that are isEffortInDay OR closed-in-day, but the class buckets (actions/tasks/projects) are computed only over the isEffortInDay subset, so the a38ac95b Actions/Tasks/Projects behavior is byte-identical (zero regression) and a Trashed-only closure appears ONLY in the Closed section. A day with nothing closed renders an empty Closed section ('No efforts'), no error. Read-path is Obsidian-core via vault.adapter with no Platform gate → identical on desktop and iPhone (Desktop↔Mobile Command Parity invariant). Deployment note: activating any pn__DailyNote exo__Layout suppresses the legacy DailyTasksRenderer catch-all feed, so a live layout must carry the full block set (Actions/Tasks/Projects + Closed); the live-activating pn__DailyNote exo__Layout + LayoutBlock vault assets are authored + temp-vault-verified, with the live pixel-visual on a real daily note deferred to a ~1-min user-confirm (ui-smoke-transitive-proof: the daily-efforts render + DailyEffortsBlockView are the same code-path a38ac95b verified; the delta is the added partition)."
---

# req(exo): pn__DailyNote Layout — a 'Closed today' daily-efforts partition (resolution/end-on-day, homoiconic, desktop + mobile)

## Job Story

When I open a daily note, I want a "Closed today" section listing the efforts I resolved or ended that day — including tasks I trashed (which carry only a resolution timestamp) — so I can review at a glance what got closed, without editing plugin code, and see it the same on my iPhone.

## Statement (Gherkin)

```gherkin
Given a daily-efforts-by-class block on a pn__DailyNote with partition "closed"
And an effort whose ems__Effort_resolutionTimestamp falls on the note's pn__DailyNote_day
Then the effort is listed in the "Closed today" section

Given an effort whose ems__Effort_endTimestamp falls on the note's day (fallback signal)
Then the effort is listed in the "Closed today" section
# DONE sets both end+resolution; TRASHED sets only resolution — both surface

Given an effort resolved/ended on a DIFFERENT day than the note
Then it is NOT listed in the "Closed today" section

Given an effort closed near local midnight (UTC day differs from local day)
Then it is matched against the note's LOCAL day interval, not the UTC day
# tz-correctness — the reason the date-match is engine TS, not a store SPARQL STRSTARTS

Given a pn__DailyNote whose day has nothing closed
Then the "Closed today" section renders empty ("No efforts"), with no error

Given a TRASHED-only effort (resolutionTimestamp on the day, no start/end/plannedStart/plannedEnd)
Then it appears ONLY in the "Closed today" section
And the Actions / Tasks / Projects class partitions are byte-identical to before (zero regression to req a38ac95b)
# additive orthogonal axis — the class buckets are computed over the isEffortInDay subset only

Given the "closed" partition
Then its built-in visibility default is shown
And a per-note frontmatter override key pn__DailyNote_showClosed can hide/show it (override > Layout default > built-in)

Given the daily-note Layout read-path
Then it uses only Obsidian-core APIs via vault.adapter with no Platform gate
And it renders identically on desktop and on mobile (Desktop↔Mobile Command Parity invariant)

Given the Closed section's composition and visibility
Then they live in the exo__Layout RDF asset + per-note frontmatter — never in plugin settings (Homoiconicity Invariant)
```

## Notes

- Design fork (SPARQL vs TS date-match) was escalated to and ruled by the orchestrator (Option A) under Andrey's approve authority: SPARQL day-filtering IS feasible on the clean `xsd:dateTime` timestamp literals (`STRSTARTS(STR(?r), "YYYY-MM-DD")` verified against the live vault-my store), so the #3805 dual-IRI налог does NOT block it — but it matches the UTC day, an off-by-tz regression near local midnight. The engine `isEffortInDay`-style local-interval match is tz-correct; the feed remains homoiconic because the Layout + Block are vault assets.
- Builds on req `a38ac95b` (daily-efforts-by-class blocks). That req deferred the live pn__DailyNote layout deployment to "Phase 4-5"; this req authors + temp-vault-verifies that layout (with the added Closed block) and defers the live pixel-visual to a user-confirm.
