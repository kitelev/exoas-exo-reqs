---
exo__Asset_uid: ace6df4f-b2c7-4dcb-afb6-bda8b20e7da0
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
exo__Asset_createdAt: 2026-06-21T01:30:00+05:00
exo__Asset_updatedAt: 2026-06-21T01:30:00+05:00
exo__Asset_createdBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253|req__Requirement]]"
exo__Asset_label: "req(exo): creating an instance surfaces the class's SHACL-required properties as form fields"
req__Requirement_status: "[[cb2e9a63-081e-46fa-89b9-7ed479516a62|req__RequirementStatusProposed]]"
req__Requirement_priority: "[[01d50c3d-ade2-4b5f-a35c-f9fd120debd3|req__RequirementPriorityP0]]"
req__Requirement_bindingClass:
  - "[[cc677c5f-b4ac-4ec3-9baa-e7ab79b78113|req__RequirementBindingClassUnit]]"
  - "[[1132e93c-6943-49eb-b7ad-4fbd20eee636|req__RequirementBindingClassGuiBdd]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers:
  - "exo create-instance — SHACL-shape-driven required-property form fields (T3, project bbe40f8c / #3656): a class's minCount>0 properties become create-instance form fields"
req__Requirement_verifiedBy:
  - "packages/exocortex/tests/unit/services/RequiredPropertyResolver.test.ts::createTripleStoreRequiredPropertyResolver > resolves a class's required (minCount>0) properties, skipping non-required"
  - "packages/obsidian-plugin/tests/e2e/eka-gui/create-instance-buttons.spec.ts (gui-bdd, real Obsidian Playwright) — floor binding"
req__Requirement_implementedBy:
  - "RequiredPropertyResolver.createTripleStoreRequiredPropertyResolver (exocortex)"
  - "CommandExecutionFlow required-property field augmentation (T3 #3656)"
---

# req(exo): creating an instance surfaces the class's SHACL-required properties as form fields

## Job Story

When I create an instance of a class that has required properties, I want the create-instance form to prompt me for exactly those properties, so the asset I create is SHACL-valid on the first try instead of failing validation afterwards.

## Statement (Gherkin)

```gherkin
Given a class whose SHACL shape declares one or more required (exo__Property_minCount > 0) properties
  (directly or via transitive exo__Class_superClass closure)
When I create an instance of that class
Then the create-instance form includes a field for each required property (and skips non-required ones)
And each field's renderer is derived from the property range (date / number / boolean / assetRef / text)
```

## Verification

**Revert-verified (unit binding):** `@req:ace6df4f-b2c7-4dcb-afb6-bda8b20e7da0` — inverting the minCount filter in `RequiredPropertyResolver` (`if (mc > 0) continue;`) makes `RequiredPropertyResolver.test.ts "resolves a class's required (minCount>0) properties, skipping non-required"` go **RED** (required properties no longer resolved); restored → **GREEN** (2026-06-21, al-reqmgmt-p2, origin/main `035804ab`).

**GUI-BDD binding (floor, cited):** the end-to-end create-instance form (real Obsidian) is exercised by `packages/obsidian-plugin/tests/e2e/eka-gui/create-instance-buttons.spec.ts` (Playwright, native-amd64 CI, PR #3583/#3659). Its revert-verify requires a Docker/native-amd64 run — **out of scope for this NO-Docker P2 session** and deferred (same precedent as seed `830ef788`). The unit binding above is the revert-verified evidence. Satisfies the P0 binding-class floor via the `gui-bdd` class (RFC 0003 §3.6).

> Migrated requirement (A14): reverse-documented from already-written tests.
