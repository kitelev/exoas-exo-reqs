---
exo__Asset_uid: de7131ae-f9e5-4498-bf06-41ccbaadc7de
exo__Asset_createdAt: 2026-09-16T13:12:51
exo__Asset_updatedAt: 2026-09-16T13:46:30
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
exo__Asset_createdBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76]]"
exo__Asset_label: "req(exo): plugin property surfaces (layout cell-edit writer, Property Editor form) are in parity with the archive-flag chokepoint — cell-edit writes canonicalYamlKey + drops the legacy spelling (canonical-wins on collision), Property Editor seeds the Archived checkbox from MetadataHelpers.isAssetArchived and its Save writes only the canonical key (tickets 3aa8a7dd / 24d7edcc)"
aliases:
  - "req(exo): plugin property surfaces (layout cell-edit writer, Property Editor form) are in parity with the archive-flag chokepoint — cell-edit writes canonicalYamlKey + drops the legacy spelling (canonical-wins on collision), Property Editor seeds the Archived checkbox from MetadataHelpers.isAssetArchived and its Save writes only the canonical key (tickets 3aa8a7dd / 24d7edcc)"
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[4bd932c2-2507-4a2d-b3f2-163e096bfa81|req__RequirementStatusApproved]]"
req__Requirement_priority: "[[2c58b8ec-8a68-463b-a694-dfe6afeb861b|req__RequirementPriorityP1]]"
req__Requirement_bindingClass: "[[f8841786-64c2-42a9-8b45-2d33fd6be87c|req__RequirementBindingClassIntegration]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers: "Plugin-side property writers/readers parity with the core archive-flag chokepoint (req 960d7a3f) and the canonical-YAML-key rule (req 869561bf): ObsidianVaultAdapter.updateFrontmatter maps every written key through canonicalYamlKey(normalizeIRI(key)), drops LEGACY_YAML_KEYS of the written canonical key from the live frontmatter, and resolves a payload that carries both spellings canonical-wins (same priority as NoteToRDFConverter guard M1 / MetadataHelpers.ARCHIVED_FLAG_KEYS); LayoutService.handleCellEdit canonicalises the edited column's property name BEFORE building the update payload so a dual carrier edited through the bare `archived` column keeps the edit; editing ANY column on a legacy `archived:` carrier migrates it to exo__Asset_archived (value preserved, graph-neutral, ORCH decision eb07dc18); exo__Asset_aliases written by the cell editor lands on `aliases` (869561bf §Scope remainder). PropertyEditorForm seeds formData.exo__Asset_archived = MetadataHelpers.isAssetArchived(frontmatter) ONLY when the canonical key is absent and a legacy/alias carrier (exo__Asset_isArchived / archived) is present, and drops those legacy keys from the seed so Save (PropertyEditorModal.handleSave → FrontmatterService.updateProperty) writes the canonical key only; no archive key is invented for an asset that has none. Source: tickets 3aa8a7dd, 24d7edcc (S7 da0f73a3, PR #4240 review), ems__Bug 43e41c8f; approved by ORCH main-35699 mandate at the code-batch plan-gate 2026-09-16 (decision eb07dc18)."
req__Requirement_approvedBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_approvedAt: 2026-09-16T13:13:15
---
## Job story

When I edit an asset's properties through the plugin's own property surfaces — an inline cell in a layout table (`LayoutService.handleCellEdit` → `ObsidianVaultAdapter.updateFrontmatter`) or the Property Editor modal — I want those surfaces to speak the SAME frontmatter-key dialect as the core chokepoint (`canonicalYamlKey` / `LEGACY_YAML_KEYS`, req `960d7a3f`; `UNPREFIXED_ASSET_FIELDS`, req `869561bf`), so that an asset never carries two spellings of one flag on disk and the Archived checkbox shows what every other reader (`MetadataHelpers.isAssetArchived`, exocmd preconditions, CLI) already sees.

## Context

- Source: tickets `3aa8a7dd-6e8c-47d0-934d-c16923bdbb52` (cell-edit writer bypasses the chokepoint) and `24d7edcc-f265-4cae-ab20-e1ce555148fb` (Property Editor reads the raw `exo__Asset_archived` key), both found by the review of PR kitelev/exocortex#4240 (S7 `da0f73a3`), tracked on `ems__Bug` `43e41c8f`. Approved for implementation by ORCH `main-35699` mandate at the code-batch plan-gate 2026-09-16 (decision `eb07dc18`).
- Why a NEW sibling requirement (not an extension of an Active one): `960d7a3f` Scenario C names its writers exhaustively (`ArchiveAssetService.archiveAsset`, the CLI `archive` executor, batch `archive`, grounding `property_set`) — the cell-edit writer and the Property Editor are not among them; `869561bf` §Scope EXPLICITLY excludes `ObsidianVaultAdapter.updateFrontmatter` («tracked on ems__Bug 43e41c8f, not closed here»). Binding new tests to either would test a clause those specs do not make.
- Mechanism measured on `origin/main` `b2a9391f` (2026-09-16): `ObsidianVaultAdapter.updateFrontmatter` applies only `FrontmatterService.normalizeIRI` to each key inside `processFrontMatter`; `LayoutService.handleCellEdit` returns `{...current, [propertyName]: value}`, i.e. RE-EMITS every existing key, so a legacy `archived: true` carrier edited through an `exo__Asset_archived` column ends up with BOTH spellings on disk. `PropertyEditorForm` seeds `formData = {...frontmatter}` and `BooleanField` reads `formData["exo__Asset_archived"]`, so a legacy carrier renders the Archived checkbox unchecked while `isAssetArchived` says true. The modal's Save already writes every key through `FrontmatterService.updateProperty` (the chokepoint) — the read side is the defect.
- Read-side priority when two spellings coexist is already fixed by `NoteToRDFConverter` (canonical wins, guard M1) and `MetadataHelpers.ARCHIVED_FLAG_KEYS` (`exo__Asset_archived` → `exo__Asset_isArchived` → `archived`); this requirement makes the plugin WRITERS agree with that priority.
- Exposure at authoring time: 0 `exo__Layout*` triples in the three canonical vaults (vault-exodev / vault-my / vault-tbank; canary `ems__Task` 2327 / 562), and the ~1270 legacy `archived:` carriers were migrated in S7 Phase B. The class of defect stays (any layout column bound to `archived` / `exo__Asset_archived` / `exo__Asset_aliases`; any carrier produced by an external tool or the Obsidian Properties panel).

## Scenarios

### Scenario A — cell edit writes the canonical key and drops the legacy spelling
Given an asset whose frontmatter carries the legacy bare key `archived: true`
When a layout-table cell bound to the property `exo__Asset_archived` is edited to `false` (`LayoutService.handleCellEdit` → `ObsidianVaultAdapter.updateFrontmatter`)
Then the frontmatter carries exactly ONE archive-flag key, `exo__Asset_archived: false`
And it carries NO bare `archived` key.

### Scenario B — a column bound to the bare legacy name is upgraded, never downgraded
Given the same legacy carrier (`archived: true`)
When a layout-table cell bound to the bare property name `archived` is edited to `false`
Then the frontmatter carries `exo__Asset_archived: false` and NO bare `archived` key
(`canonicalYamlKey("archived")` = `"exo__Asset_archived"`, the same upgrade rule as req `960d7a3f` Scenario C).

### Scenario C — a dual carrier edited through the bare column keeps the EDIT, not the stale canonical value
Given an asset that already carries BOTH `archived: true` and `exo__Asset_archived: false` (a state reachable only past the chokepoint)
When a cell bound to `archived` is edited to `true`
Then the frontmatter carries `exo__Asset_archived: true` only
(the edited property name is canonicalised BEFORE the update payload is built — `LayoutService.handleCellEdit` — so the edit lands on the canonical key; the adapter's collision rule alone would let the stale canonical value win).

### Scenario D — the adapter's collision rule is canonical-wins, matching the readers
Given `ObsidianVaultAdapter.updateFrontmatter` receives an update payload that contains BOTH `archived: true` and `exo__Asset_archived: false` (e.g. re-emitted from a dual carrier while another column was edited)
When the payload is written
Then `exo__Asset_archived` is `false` (the canonical spelling wins, the same priority as `NoteToRDFConverter` guard M1 and `MetadataHelpers.ARCHIVED_FLAG_KEYS`)
And the bare `archived` key is removed.

### Scenario E — editing an UNRELATED column on a legacy carrier migrates the flag as a side effect (accepted, graph-neutral)
Given a legacy carrier `archived: true` with `exo__Asset_label: "Old"`
When a cell bound to `exo__Asset_label` is edited to `"New"`
Then the frontmatter carries `exo__Asset_label: "New"` AND `exo__Asset_archived: true` (value preserved) and NO bare `archived` key
(the cell-edit writer re-emits every current key; per req `960d7a3f` writers never emit the bare form, and the predicate `exo:Asset_archived` is unchanged — the migration is graph-neutral and idempotent; accepted by ORCH decision `eb07dc18`).

### Scenario F — the `exo__Asset_aliases` remainder of req `869561bf` §Scope is closed for the cell-edit writer
Given an update payload with the prefixed key `exo__Asset_aliases: ["x"]` (the form a layout property editor can produce)
When it is written through `ObsidianVaultAdapter.updateFrontmatter`
Then the frontmatter carries `aliases: ["x"]` and NO literal `exo__Asset_aliases` key.

### Scenario G — negative control: keys outside the canonical-key rule pass through unchanged
Given an update payload with `exo__Asset_isDefinedBy: "[[x]]"` and `ems__Effort_status: "[[y]]"`
When it is written through `ObsidianVaultAdapter.updateFrontmatter`
Then both keys are written under their own names (canonicalisation does not rewrite arbitrary keys).

### Scenario H — the Property Editor's Archived checkbox reflects the legacy carrier
Given the Property Editor form is opened on an asset whose frontmatter carries the legacy bare key `archived: true` and NO `exo__Asset_archived` — or an EMPTY one (`exo__Asset_archived:` with no value, i.e. `null`, which every reader skips exactly like an absent key)
When the form renders the `exo__Asset_archived` boolean field
Then the "Archived" checkbox is checked (the form seeds `exo__Asset_archived` from `MetadataHelpers.isAssetArchived(frontmatter)`; "present" uses the readers' predicate `raw !== undefined && raw !== null`, not key existence — PR #4241 review LOW-2).

### Scenario I — saving the form writes only the canonical key
Given the same legacy carrier opened in the Property Editor
When the form is saved without touching the checkbox
Then the save payload contains `exo__Asset_archived: true`
And it contains NO bare `archived` key (the legacy spelling is dropped from the seed; the modal's chokepoint write then removes it from disk).

### Scenario J — negative control: no archive-flag key is invented
Given an asset with NO archive-flag key at all (`exo__Asset_archived`, `exo__Asset_isArchived`, `archived` all absent)
When the form is saved
Then the save payload contains NO `exo__Asset_archived` key (Save must not stamp `exo__Asset_archived: false` on every asset).

### Scenario K — a dual carrier renders by the readers' priority, and Save never re-emits the legacy key
Given an asset carrying BOTH `archived: true` and `exo__Asset_archived: false`
When the Property Editor form renders
Then the "Archived" checkbox is unchecked (the canonical value is kept — the same priority as `ARCHIVED_FLAG_KEYS`)
And when the form is saved without touching the checkbox, the save payload carries `exo__Asset_archived: false` and NO `archived` key
(the legacy/alias keys are dropped from the seed in EVERY case, not only when the canonical key is absent: the modal's Save writes payload keys in FILE order through `FrontmatterService.updateProperty`, and a re-emitted `archived: true` would canonicalise into an `exo__Asset_archived` write that silently overwrites a canonical `false` sitting above it — PR #4241 review MEDIUM).

## Non-goals

- The CLI `FileSystemVaultAdapter.updateFrontmatter` (same `IVaultAdapter` signature, no production caller — the only consumer of the interface is the plugin's `LayoutService`) is not changed here; parity note in PR.
- No change to `FrontmatterService`, `canonicalYamlKey`, `LEGACY_YAML_KEYS` (req `960d7a3f`) or to `PropertyEditorModal.handleSave` (already writes through the chokepoint).
- `exo__Asset_isArchived` stays a read-only compatibility alias (req `960d7a3f`); the form never writes it.
- The Obsidian Properties panel and external tools writing bare `archived:` are outside the plugin's writers and not addressed.

## Refs

- Tickets `3aa8a7dd`, `24d7edcc` (parent techbacklog `bbac67ce`); `ems__Bug` `43e41c8f`; req `960d7a3f` (chokepoint, Active), req `869561bf` §Scope (Active); PR kitelev/exocortex#4240 review (LOW, out of diff).
