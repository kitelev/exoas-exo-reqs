---
exo__Asset_uid: 454ccedf-fefe-4cfe-bdf7-704f050c1f34
exo__Asset_createdAt: 2026-09-17T10:45:01
exo__Asset_updatedAt: 2026-09-17T10:46:39
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
exo__Asset_createdBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13]]"
exo__Asset_label: "req(exo): GroundingExecutor stamps exo__Asset_updatedAt on every mutating grounding write (property_set / property_delete / property_append / property_increment / property_shift / body_template / class-flip / workflow_transition), only when the content actually changed — so the 19 single-step commands (set-parent, set-criticality-*, archive, shift-day-*, …) record a modification without a data-side bump step"
aliases:
  - "req(exo): GroundingExecutor stamps exo__Asset_updatedAt on every mutating grounding write (property_set / property_delete / property_append / property_increment / property_shift / body_template / class-flip / workflow_transition), only when the content actually changed — so the 19 single-step commands (set-parent, set-criticality-*, archive, shift-day-*, …) record a modification without a data-side bump step"
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[4bd932c2-2507-4a2d-b3f2-163e096bfa81|req__RequirementStatusApproved]]"
req__Requirement_priority: "[[481c3be1-d05c-4b78-8a3c-c61308d40bf1|req__RequirementPriorityP2]]"
req__Requirement_bindingClass:
  - "[[f8841786-64c2-42a9-8b45-2d33fd6be87c|req__RequirementBindingClassIntegration]]"
  - "[[cc677c5f-b4ac-4ec3-9baa-e7ab79b78113|req__RequirementBindingClassUnit]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
exo__Asset_relates:
  - "[[f7790000-3779-4aaa-8aaa-000000000001]]"
  - "[[3800d995-2bae-401f-a23a-dac914505e9d]]"
  - "[[664123d3-5b91-4793-8085-485d48471546]]"
  - "[[1af85afd-c7c6-4cf2-a59f-353f23aaa5e9]]"
req__Requirement_covers:
  - "exocortex core GroundingExecutor — every mutating grounding branch (property_set incl. the workflow_transition status write, property_delete, property_append, property_increment, property_shift, body_template, convert-to-task / convert-to-project) stamps exo__Asset_updatedAt = DateFormatter.toLocalTimestamp(clock.now()) (YYYY-MM-DDTHH:mm:ss, the same form `$nowLocal` / create_instance write) on the content it is about to write; adds the key when absent. Closes the class behind ticket 533856e4 (19 non-composite commands never bumped: 11 property_set — archive, mark-done-post-factum, move-to-backlog, plan-on-today, set-blocker x2, set-criticality-high/low/medium, set-draft-status, set-parent — plus append-instance-class, copy-label-to-aliases, shift-day-backward/forward, un-archive, vote-on-effort, re-open, rollback-to-backlog). Plugin buttons run through the same executor — CLI/UI and Desktop/Mobile parity by construction."
  - "Skip rules that make the stamp an invariant rather than noise: (1) a branch whose OWN target property is exo__Asset_updatedAt (composite step 49e00287 `$nowLocal`) owns the key — the executor writes no second one (exactly one key line, idempotent); (2) a write that leaves the content UNCHANGED (property_set to the value already on disk, property_append of an alias already present, property_delete of an absent key) is byte-identical — no spurious ExoSync delta, same rule as remove-property 'bumps only when a change occurs'; (3) refusals (missing named input, unquoted-wikilink guard req 29e0d1b6, precondition) write nothing and stamp nothing; (4) composite rollback restores the pre-composite bytes unstamped; (5) a target without a frontmatter block (plain-markdown body_template) gets no invented frontmatter."
  - "Not governed: service_call groundings (17 commands mutate inside packages/services past these write points — separate follow-up under bbac67ce); create_instance (already writes updatedAt = createdAt, task 1af85afd)."
req__Requirement_approvedBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_approvedAt: 2026-09-17T10:46:36
---

## Job Story

When I mutate an existing asset through ANY homoiconic command — `apply set-parent`, `set-criticality-*`, `archive`, `move-to-backlog`, `plan-on-today`, `shift-day-*`, `vote-on-effort`, `rollback-to-backlog`, … — from the CLI or from a plugin button, I want the asset's `exo__Asset_updatedAt` to record that modification, so that "when did this asset last change" is an invariant the graph can rely on (ExoSync deltas, staleness audits, `updatedAt`-ordered views) instead of a courtesy that only the 18 composite groundings carrying the data-side step `49e00287` ("Bump updatedAt") happen to provide.

## Context

- Ticket `533856e4-622d-4256-91fa-bb0235e83316` (parent techbacklog `bbac67ce`), found while closing `4f226028` (PR #4250). Repro on the published CLI 16.240.4 over a temp vault: `apply set-parent`, `apply set-criticality-low` and `apply rollback-to-backlog` each wrote their property and left `exo__Asset_updatedAt` at the seeded value; the control `apply set-label` (composite with step `49e00287`) bumped it. n = 3 by execution.
- Mechanism (`packages/core/src/services/GroundingExecutor.ts` @ `031f26ec`): there is NO executor-side bump — `git grep updatedAt` hits only the create path (`executeCreateInstance`, `updatedAt = createdAt`, task `1af85afd`). Every mutating branch reads → `FrontmatterService.updateProperty/removeProperty/replaceBody` → `fileWriter.updateFile` without touching `updatedAt`: `property_set` (`:813`, also the target of `workflow_transition` `:3184`), `property_delete` (`:949`), `convert-to-task`/`convert-to-project` (`:1247`/`:1260`), `body_template` (`:1333`), `property_append` (`:2910`), `property_increment` (`:2981`), `property_shift` (`:3066`).
- Blast radius (SPARQL over the 74 `exocmd__Command` with a cliName): **19** commands mutate through a SINGLE non-composite grounding and therefore never bump — 11 `property_set` (archive, mark-done-post-factum, move-to-backlog, plan-on-today, set-blocker ×2, set-criticality-high/low/medium, set-draft-status, set-parent) + append-instance-class, copy-label-to-aliases (`property_append`), shift-day-backward/forward (`property_shift`), un-archive (`property_delete`), vote-on-effort (`property_increment`), re-open / rollback-to-backlog (`workflow_transition`). The 18 composites bump only where a data author remembered the step. `1af85afd` (Done) closed the CREATE side only; it is not reopened — its closure is incomplete by n = 19, recorded in the ticket.
- Homoiconicity Q3: "when was this asset last modified" is a structural integrity invariant of the graph, not user-configurable domain semantics → it belongs to the executor (guard rail), not to 19 more data-side composite steps that would regress the moment a 20th command forgets the step. The plugin executes buttons through the same `GroundingExecutor`, so Desktop/Mobile/CLI parity holds by construction.
- Not governed: `service_call` groundings (17 commands) mutate inside `packages/services` past these write points — a separate follow-up under `bbac67ce`; `create_instance` (already stamps `updatedAt = createdAt`); the composite rollback (restores the pre-composite bytes verbatim).

## Statement (Gherkin)

```gherkin
Given an existing asset whose frontmatter carries exo__Asset_updatedAt "2020-01-01T00:00:00"
When a property_set grounding (e.g. `apply set-parent … --input '{"parent":"<uid>"}'`) writes that asset
Then the target property is written
  And exo__Asset_updatedAt is no longer "2020-01-01T00:00:00" and has the local-timestamp shape YYYY-MM-DDTHH:mm:ss (DateFormatter.toLocalTimestamp of the executor clock — the same form `$nowLocal` and create_instance write)

Given the same seeded asset
When the write comes from property_delete, property_append, property_increment, property_shift, body_template, convert-to-task / convert-to-project, or a workflow_transition status mutation
Then exo__Asset_updatedAt is stamped exactly the same way (every mutating branch of GroundingExecutor)

Given an asset with NO exo__Asset_updatedAt key
When a mutating grounding writes it
Then the key is added with the local-timestamp value

Given a composite grounding whose own steps already write exo__Asset_updatedAt (step 49e00287 `$nowLocal`)
When the composite runs
Then the file carries exactly ONE exo__Asset_updatedAt key (the explicit step is the stamp; the executor does not write a second one)

Given a grounding that REFUSES before writing (missing named input, unquoted-wikilink guard req 29e0d1b6, precondition failure)
When it is applied
Then the file is byte-identical — no stamp without a write

Given a grounding whose write would leave the content UNCHANGED (property_set to the value already on disk, property_append of an alias already present, property_delete of an absent key)
When it is applied
Then the file is byte-identical and exo__Asset_updatedAt is NOT touched (an idempotent re-apply must not create a spurious ExoSync delta — same rule as remove-property "bumps only when a change occurs")

Given a composite whose later step fails after an earlier step mutated the click-target
When the composite rolls back
Then exo__Asset_updatedAt is restored to its pre-composite value (rollback writes the original bytes, unstamped)

Given a file WITHOUT a frontmatter block (a plain markdown target of body_template)
When the body is written
Then no frontmatter block is invented — the stamp applies only where a frontmatter block exists after the mutation
```

## Verification

Binding: integration (real `apply` over a temp vault, `packages/cli/tests/integration/apply-mutation-parity.integration.test.ts`) + unit (`packages/core/tests/unit/services/GroundingExecutor.updatedat-stamp.test.ts`, one axis per mutating branch). Revert-verify + mutant matrix (M1 stamp no-op · M2 stamp only in property_set · M3 stamp before the refuse guard · M4 stamp on rollback · M5 stamp on a no-op write) recorded in the PR body. The former `:436` axis "legacy 2-step set-label does NOT bump updatedAt" (#3798 BUG 2) is rewritten under this requirement: the executor bumps even without the data step; the aliases-accumulation half (BUG 1) stays a red axis of the data.

## Related

- `[[f7790000-3779-4aaa-8aaa-000000000001]]` (set-parent — the command that surfaced the gap), `[[3800d995-2bae-401f-a23a-dac914505e9d]]` (set-property bumps via the CLI verb), `[[664123d3-5b91-4793-8085-485d48471546]]` (set-body bumps via the CLI verb) — three explicit per-verb bumps this invariant generalises to every grounding write.
- `[[1af85afd-c7c6-4cf2-a59f-353f23aaa5e9]]` — create-side half of the same invariant (Done).

