---
exo__Asset_uid: dd9d1be5-16d4-49be-bbde-1e4fdd53c8ee
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
exo__Asset_createdAt: 2026-06-21T12:32:45+05:00
exo__Asset_updatedAt: 2026-06-21T12:32:45+05:00
exo__Asset_createdBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253|req__Requirement]]"
exo__Asset_label: "req(exo): audit co-location reports a violation when an asset's folder differs from its isDefinedBy ontology folder"
req__Requirement_status: "[[601bb075-ccf6-4d70-8a05-f6fd669c42b7|req__RequirementStatusDraft]]"
req__Requirement_priority: "[[01d50c3d-ade2-4b5f-a35c-f9fd120debd3|req__RequirementPriorityP0]]"
req__Requirement_bindingClass:
  - "[[f8841786-64c2-42a9-8b45-2d33fd6be87c|req__RequirementBindingClassIntegration]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers:
  - "exo audit co-location (RFC 0b7a2fad CR-1): the audit scans the vault and reports a violation for every asset whose on-disk folder differs from the folder of the ontology its exo__Asset_isDefinedBy resolves to. Co-located assets pass (0 violations); fail-open assets (bang-prefix / unresolvable / empty isDefinedBy) and Templater template folders (09 Templates/, templates/) are skipped, never reported"
req__Requirement_verifiedBy:
  - "packages/cli/tests/integration/commands/audit-co-location.integration.test.ts::audit co-location — revert→fail / restore→pass (integration) > PASS when co-located, FAIL when moved away, PASS again when restored"
req__Requirement_implementedBy:
  - "scanVaultForCoLocation (packages/cli/src/commands/audit-co-location.ts) — compares normalizePath(dirname(ontologyPath)) vs normalizePath(dirname(relPath)) and pushes a violation on mismatch; the same resolver used by `apply repair-folder` / co-location placement on create"
---

# req(exo): audit co-location reports a violation when an asset's folder differs from its isDefinedBy ontology folder

## Job Story

When I run the co-location audit on my vault, I want every asset whose physical folder no longer matches its `exo__Asset_isDefinedBy` ontology folder flagged as a violation — while fail-open and Templater assets are skipped — so the co-location invariant (CR-1) is mechanically enforceable and silent drift can't accumulate.

## Statement (Gherkin)

```gherkin
Given an asset co-located in its ontology's folder (its exo__Asset_isDefinedBy resolves to an ontology in the same directory)
When I run `audit co-location`
Then the audit reports 0 violations and counts the asset as checked

Given the same asset is physically moved out of its ontology folder (e.g. into 03 Knowledge/inbox)
When I run `audit co-location`
Then the audit reports exactly 1 violation, with actualFolder = the wrong folder and expectedFolder = the ontology folder

Given the asset is moved back into its ontology folder
When I run `audit co-location`
Then the audit reports 0 violations again
```

## Verification

**Revert-verified (integration binding):** `@req:dd9d1be5-16d4-49be-bbde-1e4fdd53c8ee` — disabling the mismatch check in `scanVaultForCoLocation` (`if (false && expectedFolder !== actualFolder)`, so no violation is ever pushed) makes `audit-co-location.integration.test.ts "PASS when co-located, FAIL when moved away, PASS again when restored"` go **RED** (the moved-away asset is not flagged → `expect(r.violations).toHaveLength(1)` receives 0 at line 72); restored → **GREEN** (2026-06-21, al-reqmgmt-ramp, origin/main `1a6915d5`). The two folder-exclusion cases (09 Templates / templates) stayed GREEN under the revert (clean isolation — they assert 0 violations regardless).

> Migrated requirement (A14): reverse-documented from an already-written test (test-first-then-spec). This binding is itself a revert→fail / restore→pass test (it physically moves the file out of co-location and back), so it is `integration`-class (real `scanVaultForCoLocation` over a temp vault with real fs moves), satisfying the P0 binding-class floor (RFC 0003 §3.6) — no deferred floor.
