---
exo__Asset_uid: ace6df4f-b2c7-4dcb-afb6-bda8b20e7da0
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
exo__Asset_createdAt: 2026-06-21T01:30:00+05:00
exo__Asset_updatedAt: 2026-09-18T09:33:55
exo__Asset_createdBy: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253|req__Requirement]]"
  - "[[62464150-2e47-486d-b808-631c9bba10ad]]"
exo__Asset_label: "req(exo): creating an instance surfaces the class's SHACL-required properties as form fields"
req__Requirement_status: "[[fccf8fa4-8004-41ee-9102-595a588e9be7|req__RequirementStatusActive]]"
req__Requirement_priority: "[[01d50c3d-ade2-4b5f-a35c-f9fd120debd3|req__RequirementPriorityP0]]"
req__Requirement_bindingClass:
  - "[[cc677c5f-b4ac-4ec3-9baa-e7ab79b78113|req__RequirementBindingClassUnit]]"
  - "[[d58c8af5-6366-4b79-aa2a-74b187e461f0|req__RequirementBindingClassE2e]]"
  - "[[cc677c5f-b4ac-4ec3-9baa-e7ab79b78113|req__RequirementBindingClassUnit]]"
  - "[[1132e93c-6943-49eb-b7ad-4fbd20eee636|req__RequirementBindingClassGuiBdd]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers:
  - "exo create-instance — SHACL-shape-driven required-property form fields (T3, project bbe40f8c / #3656): a class's minCount>0 properties become create-instance form fields"
req__Requirement_verifiedBy:
  - packages/exocortex/tests/unit/services/RequiredPropertyResolver.test.ts::createTripleStoreRequiredPropertyResolver > resolves a class's required (minCount>0) properties, skipping non-required
  - packages/obsidian-plugin/tests/e2e/eka-gui/create-instance-buttons.spec.ts (gui-bdd, real Obsidian Playwright) — floor binding
  - packages/core/tests/unit/services/RequiredPropertyResolver.test.ts::createTripleStoreRequiredPropertyResolver > CURIE-literal datatype range xsd:<local> (ticket 5380e7fd) > C1 / C2 / C3
req__Requirement_implementedBy:
  - RequiredPropertyResolver.createTripleStoreRequiredPropertyResolver (exocortex)
  - "CommandExecutionFlow required-property field augmentation (T3 #3656)"
  - "PR kitelev/exocortex#4254 (merge 72b1f66d, release v16.240.8): RequiredPropertyResolver.fieldTypeFromRange derives targetClassUid (label form) from a symbolic Property_range via iriToObsidianName; axes S1-S3 (core), S4 (plugin production-shape); 3 mutants — ticket dc04eded"
  - "PR kitelev/exocortex#4262 (ticket 5380e7fd): RequiredPropertyResolver.xsdLocalName recognises the CURIE-literal datatype range xsd:<local> (the form 100 % of live datatype ranges carry) like the full XSD IRI; axes C1-C3 (core)"
flow__WorkItem_migratedAt: 2026-08-16T19:16:22
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

Scenario: an object range in SYMBOLIC IRI form yields a picker-keyed assetRef field (2026-09-17, ticket dc04eded, PR #4254)
  Given a required property whose exo:Property_range object is the symbolic ontology IRI <ns>#<Local>
    (the form the converter emits for every class with a prefix__LocalName label — ALL 22 required
    object ranges on vault-exodev on 2026-09-17, 0 path-form; vault-my 13/0, vault-tbank 17/0)
  When the create-instance form's required-property fields are resolved
  Then the field is assetRef with targetClassUid = the class LABEL <ns>__<Local>
    (via the shared core inverse iriToObsidianName → Namespace.fromTermIRI — every registered AND ad-hoc namespace)
    And the plugin's DynamicFormModal builds that field's candidates from the class AND its subclasses (req 15f48fa1 closure)
    And a path-form range obsidian://…/<uid>.md still yields the bare (lower-cased) class UID, exactly as before
    And a datatype (xsd) range is still never an assetRef

Scenario: a CURIE-literal datatype range `xsd:<local>` renders the same field as the full XSD IRI (2026-09-18, ticket 5380e7fd, PR #4262)
  Given a required property whose exo:Property_range is the CURIE literal "xsd:dateTime" (or "xsd:integer" / "xsd:boolean" / "xsd:string")
    (the form every datatype range on the live vaults carries — vault-exodev 134, vault-my 48, vault-tbank 44 on 2026-09-18; the full http://www.w3.org/2001/XMLSchema# literal has 0 live carriers)
  When the create-instance form's required-property fields are resolved
  Then the field is date (/ number / boolean / text) exactly as for the full XSD IRI, and never an assetRef
    And a range with a foreign CURIE prefix (ex:date), a bare "xsd:" or an unknown xsd local (xsd:gYear) still renders as text
```

## Verification

**Revert-verified (unit binding):** `@req:ace6df4f-b2c7-4dcb-afb6-bda8b20e7da0` — inverting the minCount filter in `RequiredPropertyResolver` (`if (mc > 0) continue;`) makes `RequiredPropertyResolver.test.ts "resolves a class's required (minCount>0) properties, skipping non-required"` go **RED** (required properties no longer resolved); restored → **GREEN** (2026-06-21, al-reqmgmt-p2, origin/main `035804ab`).

**GUI-BDD binding (floor, cited):** the end-to-end create-instance form (real Obsidian) is exercised by `packages/obsidian-plugin/tests/e2e/eka-gui/create-instance-buttons.spec.ts` (Playwright, native-amd64 CI, PR #3583/#3659). Its revert-verify requires a Docker/native-amd64 run — **out of scope for this NO-Docker P2 session** and deferred (same precedent as seed `830ef788`). The unit binding above is the revert-verified evidence. Satisfies the P0 binding-class floor via the `gui-bdd` class (RFC 0003 §3.6).

**Gap closed 2026-09-17 (ticket `dc04eded`, PR #4254, bug-fix under this req — no new req):**
`RequiredPropertyResolver.fieldTypeFromRange` derived the picker key with `uidFrom`, which understands
only the path-form / bare-UID range, so a SYMBOLIC range (`…/ontology/ems#Effort` — 22 of 22 required
object ranges on the live vault-exodev, incl. `exo__Setting_key → exo#SettingKey`) reached the form as
`assetRef` WITHOUT `targetClassUid` and the picker degraded to a plain text input: the assetRef renderer
this req promises was effectively unmet for every required object property; the gap surfaced in the IRI
form (dual-IRI, `sparql-iri-form-pre-verify`). The spec is extended above, not replaced. Axes
`@req:ace6df4f-…` S1–S3 in `packages/core/tests/unit/services/RequiredPropertyResolver.test.ts`
(registered ns / ad-hoc ns / path-form regression) and S4 in
`packages/obsidian-plugin/tests/unit/presentation/modals/DynamicFormModal.test.ts` (production shape:
REAL store → REAL resolver → REAL `CommandExecutionFlow.applyRequiredPropertyFields` → REAL
`DynamicFormModal` with its production candidate resolver over a fake `app.metadataCache`); mutant
driver: drop the symbolic branch → S1/S2/S4 RED, registered-only inverse → S2 RED, path-form branch
broken → S3 RED.

**Gap closed 2026-09-18 (ticket `5380e7fd`, PR #4262, bug-fix under this req — no new req):**
`RequiredPropertyResolver.fieldTypeFromRange` recognised a datatype range only by the full
`http://www.w3.org/2001/XMLSchema#` prefix; every datatype range on the live vaults is the CURIE
literal `"xsd:<local>"` (134 / 48 / 44 on the three vaults, 0 full-IRI literals), so `ems__Reminder_at`
(`xsd:dateTime`, required) rendered as a plain text field instead of a date field — the "renderer derived
from the property range" promise was unmet for 100 % of live datatype ranges. Fixed by `xsdLocalName`
(full IRI ∪ `xsd:` CURIE); axes `@req:ace6df4f-…` C1–C3 in
`packages/core/tests/unit/services/RequiredPropertyResolver.test.ts`; mutant driver: CURIE branch removed
→ C1 RED, any-prefix → C3 RED, CURIE local not lower-cased → C1 RED, full-IRI branch removed → C2 RED.
Out of scope: `ShapeLoader.loadFromRDFGraph` drops the same CURIE literal (sh:datatype never checked for
those ranges) — separate follow-up ticket with the measured blast-radius.

> Migrated requirement (A14): reverse-documented from already-written tests.
