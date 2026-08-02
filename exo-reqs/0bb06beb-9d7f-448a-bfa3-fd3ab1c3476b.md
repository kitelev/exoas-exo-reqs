---
exo__Asset_uid: 0bb06beb-9d7f-448a-bfa3-fd3ab1c3476b
exo__Asset_createdAt: 2026-08-02T22:41:46
exo__Asset_updatedAt: 2026-08-02T23:48:01
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
exo__Asset_createdBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13]]"
exo__Asset_label: "req(exo): the legacy prototype backlink fires only when the target IS a prototype"
aliases:
  - "req(exo): the legacy prototype backlink fires only when the target IS a prototype"
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[fccf8fa4-8004-41ee-9102-595a588e9be7|req__RequirementStatusActive]]"
req__Requirement_priority: "[[2c58b8ec-8a68-463b-a694-dfe6afeb861b|req__RequirementPriorityP1]]"
req__Requirement_bindingClass: "[[f8841786-64c2-42a9-8b45-2d33fd6be87c|req__RequirementBindingClassIntegration]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers: exo create_instance — legacy exo__Asset_prototype backlink top-up gated on target being a prototype
req__Requirement_approvedBy: "[[0aa339bc-9b56-400a-8148-cbde57bbf0b6|a.kitelev]]"
req__Requirement_approvedAt: 2026-08-02T22:47:00+05:00
req__Requirement_implementedBy: "PR #4020 (squash f6cbe26f)"
---
# req(exo): the legacy prototype backlink fires only when the target IS a prototype, and links by UID

## Job Story

When I create an asset from a page that is **not** a prototype (a daily note, an
area, any class without a backlink InheritanceRule), I want the new asset to carry
no invented prototype link, so my graph never asserts a relationship that does not exist.
And when the link IS written, I want it in UID-canon form, so renaming or moving the
target never breaks it.

## Context

`GroundingExecutor.applyMissingBacklinkTopUp` is a degraded-mode safety net: when no
InheritanceRule wrote a backlink and no property already references the click-target,
it writes `exo__Asset_prototype` pointing at that target and logs an error naming
itself «Bug #5 may surface if target is not a prototype-instance».

The code comment frames the trade-off as «a corrupt asset is preferable to creation
failure». That dilemma is **false**: the third option — write nothing — was never on
the table. Omitting a backlink is not a failure; a missing link is a normal state,
while an invented one is a lie the graph then carries.

The net is still worth keeping where it is truthful. When the target genuinely IS a
prototype, `exo__Asset_prototype` is the correct property and preserving it protects
the prototype chain if the Universal InheritanceRules are absent.

**Measured blast radius** (2026-08-02, live vaults): **112 assets** already carry an
`exo__Asset_prototype` pointing at a non-prototype (my 86 / tbank 17 / exodev 9) —
the parasitic class this requirement ends. The legitimate case is far larger and
untouched: 282 in vault-my alone point at real `ems__TaskPrototype` assets
(«Покурить ploom», «Lunch», «Процессинг»), and those keep working.

## Statement (Gherkin)

```gherkin
Given a create_instance grounding runs on a click-target
  And no InheritanceRule wrote a backlink to that target
  And the grounding declares no explicit linkBackProperty
  And no already-written property references the target

When the click-target is NOT a prototype
  (none of its exo__Instance_class refs reaches exo__Prototype
   by walking exo__Class_superClass — a name-suffix test is NOT used;
   req 5579ffa1 de-hacked exactly that heuristic)
Then the created asset carries NO exo__Asset_prototype
  And no "No backlink rule fired" error is logged
  And every other property the pipeline wrote is unchanged

When the click-target IS a prototype
Then the created asset carries exo__Asset_prototype pointing at that target
  And the reference is the target's exo__Asset_uid (UID-canon), never a vault path
  And the degraded-mode safety net is otherwise preserved unchanged
```

## Why not "never write the fallback"

The originating task asked to drop the fallback entirely. That would also remove the
net in the case where it is **correct** — a real prototype whose Universal
InheritanceRules are missing would silently lose its prototype link. Gating on what
the property actually means (`exo__Asset_prototype` points at a prototype) fixes the
lie without discarding the protection.

## Why the reference form is a separate clause

`extractBacklinkTarget` returns the basename when it is a UUID and the **full vault
path** otherwise — so a label-named target produced
`[[assetspaces/kitelev/exoas-my/my-pns/2026-08-02 Note]]`, which breaks the moment the
note is renamed or moved.

Empirically every real prototype is UID-named (a SPARQL sweep for label-named
prototypes returns only symbolic-IRI duals — `exo#RootPrototype`, `kitelev#MyStuff` —
not files), so the class-gate alone already eliminates the path form in practice.
Reading the uid from the target's own frontmatter makes that a **guarantee** rather
than a consequence, and keeps the UID-canon invariant true even if a label-named
prototype ever appears.

## Verification plan

Revert-verified integration test over the real `GroundingExecutor.executeCreateInstance`:

- non-prototype target → no `exo__Asset_prototype` (RED when the class-gate is neutralized)
- prototype target → `exo__Asset_prototype` still written (negative control: stays GREEN
  under the class-gate revert, proving the delta is exactly the non-prototype case)
- prototype target → the written value is `[[<uid>]]`, not a path (RED when the
  uid-preference is neutralized; independent axis from the gate)
- an already-linked target (e.g. Area with `ems__Effort_area`) → unchanged (#3561 behaviour)

Task: 626a630c-9780-45b6-acd6-e3b8dca55c5a
