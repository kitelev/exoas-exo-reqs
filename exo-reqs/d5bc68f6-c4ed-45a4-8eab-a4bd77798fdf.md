---
exo__Asset_uid: d5bc68f6-c4ed-45a4-8eab-a4bd77798fdf
exo__Asset_createdAt: 2026-08-02T22:11:30
exo__Asset_updatedAt: 2026-08-16T19:16:22
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
  - "[[62464150-2e47-486d-b808-631c9bba10ad]]"
exo__Asset_createdBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13]]"
exo__Asset_label: "req(exo): removing a knowledge pack states its blast radius before the destructive confirm — the still-mounted packs that declare a dependsOn on it, and the resolved links from remaining notes that would be left dangling (P2 of project 17c173dd, re-scoped to the warning half per its P0 falsification verdict)"
aliases:
  - "req(exo): removing a knowledge pack states its blast radius before the destructive confirm — the still-mounted packs that declare a dependsOn on it, and the resolved links from remaining notes that would be left dangling (P2 of project 17c173dd, re-scoped to the warning half per its P0 falsification verdict)"
  - "req(exo): unmount blast-radius warning"
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[fccf8fa4-8004-41ee-9102-595a588e9be7|req__RequirementStatusActive]]"
req__Requirement_priority: "[[481c3be1-d05c-4b78-8a3c-c61308d40bf1|req__RequirementPriorityP2]]"
req__Requirement_bindingClass: "[[f8841786-64c2-42a9-8b45-2d33fd6be87c|req__RequirementBindingClassIntegration]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers: "P2 of project 17c173dd (Profile-based mount management), RE-SCOPED per the explicit verdict of its own falsification gate P0 (task 8b8466bb, 2026-08-02). P0 measured the three real vaults and found that mounting less through profiles does NOT materially help (the active profile already resolves to 100% of vault-my, 100% of vault-tbank, 99.4% of vault-exodev; the limiter is the transitive exo__AssetSpace_dependsOn closure, not the floor; the minimal profile that still has personal data saves 0.7% of files and -0.6% index time). P0's recommendation for P2 was verbatim: lower its priority, because granularity buys nothing where the closure is forced and is useful ONLY together with the reverse-ref warning (risk #1), which P0 measured as real (3684 wikilinks from 1464 personal assets point into shared-private). This requirement implements exactly that warning half. SCOPE: the plugin (packages/obsidian-plugin) ONLY, read-only assessment - nothing about what is mounted changes and the removal is never blocked. MECHANISM (zero new engine code; reuses the shared transitiveDependsOnClosure from core AssetSpaceDependsOn.ts): a pure domain/profile/unmountSafety.ts, sibling of the shipped closureGap.ts (#3956) and indexCost.ts (req 6171f443), exposing findDependentMountedAssetSpaces(targetUid, mountedUids, allInfos) - the REVERSE of detectUnmountedClosureMembers, i.e. every still-mounted AssetSpace whose closure contains the target - plus countIncomingLinks(mountPath, resolvedLinks) reading Obsidian's already-computed metadataCache.resolvedLinks (Record<source, Record<target, count>>) for links from OUTSIDE the mount folder into files INSIDE it, plus formatUnmountRiskWarning(risk) -> string|null. resolvedLinks carries only links whose target exists, which is exactly right: the pack is mounted at assessment time, so those links resolve now and would dangle after removal. No vault walk and no node:fs, so it is free and identical on iOS (Desktop-Mobile parity). TBOX_PROVIDER_NAMESPACES is EXPORTED from closureGap.ts and imported rather than duplicated, so the notion of which packs supply the class/command TBox stays single-sourced across the forward (mount-gap) and reverse (unmount-risk) checks - this matters because exocmd is OPTIONAL, not TS-floor, so it CAN be removed today and its absence leaves homoiconic commands and SHACL validation silently dead. WIRING: UnmountAssetSpaceCommandDeps gains an OPTIONAL assessRisk callback evaluated after the floor refusal and before the destructive confirm; a non-null warning is prepended to the existing confirm body; a throw is swallowed (best-effort). ZERO-REGRESSION is structural: with no dependents and no incoming links the formatter returns null, nothing is prepended, and the confirm body is byte-identical to the shipped wording. TEST (revert-verified, @req-tagged Integration): pure domain cases (reverse-closure detection, outside-in link counting, TBox flag, null-when-safe) plus command-level wiring cases driving the real UnmountAssetSpaceCommand (warning prepended; null -> byte-identical; assessRisk throws -> byte-identical and removal still possible); three revert axes each redden only their own assertions. DELIBERATELY OUT OF SCOPE: a new panel listing every AssetSpace with mount-state and a free-form mount/unmount toggle - the mounted list with own+closure size already exists (Remove knowledge pack, req 6171f443), mounting already exists (Add a knowledge pack / assetspace-add), and P0's verdict is that free-form granular trimming buys ~0.7% while inviting breakage, so building it would ship the misleading affordance the gate warned about; also out of scope: changing what is mounted, a CLI blast-radius equivalent for assetspace-remove (sibling surface), and blocking the removal."
req__Requirement_approvedBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13|ExoAssistant]]"
req__Requirement_approvedAt: 2026-08-02T22:12:28
req__Requirement_implementedBy: "PR #4019 (P2 of project 17c173dd)"
flow__WorkItem_migratedAt: 2026-08-16T19:16:22
---

# req(exo): removing a knowledge pack warns about what it would break

## Job Story

When I am about to remove a knowledge pack from my vault, I want to be told **what removing it would break** — which still-mounted packs declare a dependency on it, and how many links from my remaining notes point into it — so that I decide with the blast radius in front of me instead of discovering silently-dead commands and dangling links afterwards.

## Why this, and why NOT a new toggle panel (P0 falsification, 2026-08-02)

This is P2 of project `17c173dd` (Profile-based mount management), **re-scoped per the explicit verdict of its own falsification gate P0** (task `8b8466bb`).

P0 measured the three real vaults and concluded that mounting less through profiles does **not** materially help: the active profile already resolves to 100 % of vault-my (8 738 / 8 738), 100 % of vault-tbank and 99.4 % of vault-exodev, because the transitive `exo__AssetSpace_dependsOn` closure — not the always-mounted floor — is the limiter. Its recommendation for P2 was verbatim:

> **P2 (гранулярный менеджер) — понизить приоритет.** Гранулярность не даёт выигрыша там, где закрытие принудительно; полезна **только вместе с reverse-ref предупреждением** (риск №1), которое здесь измерено как реальное (3 684 ссылки).

So the load-bearing half of P2 is the **warning**, and the panel half is not merely unnecessary — it is actively harmful:

- a panel promising "trim what is mounted to index faster" sells an outcome P0 measured at **0.7 % of files / −0.6 % index time**;
- a free-form toggle over *all* AssetSpaces invites exactly the removal that breaks the vault, because the closure is forced. Unmounting `exocmd` is permitted today (it is **optional**, not TS-floor) and, per the shipped `closureGap.ts`, its absence leaves homoiconic commands and SHACL validation **silently** dead.

P0 also quantified the data-level half on the real vault: **3 684 wikilinks from 1 464 personal assets** point into `shared-private` — so "just unmount the big shared pack" would break 39 % of the personal corpus.

## Scenarios

```gherkin
Feature: Removing a knowledge pack states its blast radius before the destructive confirm

  Background:
    Given the «Remove knowledge pack (advanced)» command lists the mounted AssetSpaces
    And I have picked one that is not TS-floor protected

  Scenario: A still-mounted pack declares a dependency on the one I am removing
    Given another mounted AssetSpace has the picked one in its transitive exo__AssetSpace_dependsOn closure
    When the destructive confirmation is raised
    Then it names that dependent pack before the removal wording
    And when the picked pack supplies the class / command TBox it says so explicitly

  Scenario: Notes that stay behind link into the pack I am removing
    Given resolved links point from files outside the pack's folder into files inside it
    When the destructive confirmation is raised
    Then it states how many links from how many files would be left dangling

  Scenario: Nothing depends on the pack and nothing links into it
    When the destructive confirmation is raised
    Then its wording is byte-identical to the wording shipped before this requirement

  Scenario: The assessment itself fails
    Given computing the blast radius throws
    When the destructive confirmation is raised
    Then it falls back to the byte-identical wording and the removal remains possible
```

## Mechanism (zero new engine code)

A pure `domain/profile/unmountSafety.ts` — sibling of the shipped `closureGap.ts` (issue #3956) and `indexCost.ts` (req `6171f443`) — reusing the SAME shared `transitiveDependsOnClosure` helper from core that `ProfileApplyManager.resolveDeclaredAndEffective` and `CliProfileResolver` consume:

- `findDependentMountedAssetSpaces(targetUid, mountedUids, allInfos)` — the **reverse** of `detectUnmountedClosureMembers`: every still-mounted AssetSpace whose closure contains the target. `closureGap` asks "what did I forget to mount?"; this asks "who still needs what I am about to remove?".
- `countIncomingLinks(mountPath, resolvedLinks)` — links from sources **outside** the mount folder into files **inside** it, plus the number of distinct source files. Read straight off Obsidian's already-computed `metadataCache.resolvedLinks` (`Record<source, Record<target, count>>`) — no vault walk, no `node:fs`, so it costs nothing extra and works identically on iOS (Desktop↔Mobile parity). `resolvedLinks` only carries links whose target exists, which is exactly right: the pack is mounted at assessment time, so a link into it resolves now and would dangle after removal.
- `formatUnmountRiskWarning(risk)` → `string | null`.

`TBOX_PROVIDER_NAMESPACES` is **exported from `closureGap.ts` and imported** rather than duplicated, so "which packs supply the class / command TBox" stays single-sourced across the forward (mount-gap) and reverse (unmount-risk) checks.

Wiring: `UnmountAssetSpaceCommandDeps` gains an **optional** `assessRisk?: (chosen) => Promise<string | null>`, evaluated after the floor refusal and before the destructive confirm; a non-null warning is prepended to the existing confirm body. The dep is best-effort (a throw is swallowed) and optional, so an unwired or failing assessment degrades to today's behaviour rather than blocking a removal.

**Zero-regression is structural, not argued**: with no dependents and no incoming links the formatter returns `null`, the caller prepends nothing, and the confirm body is the same string as before this requirement.

## Deliberately NOT in scope

- **A new panel listing every AssetSpace with mount-state + toggle.** The mounted list with its size already exists («Remove knowledge pack», with own + closure cost from req `6171f443`), mounting already exists («Add a knowledge pack» / `assetspace-add`), and P0's verdict is that free-form granular trimming buys ~0.7 % while inviting breakage. Building it would ship the misleading affordance the gate warned about.
- Changing what is mounted, or how the closure is resolved.
- A CLI equivalent (`assetspace-remove` blast-radius) — sibling surface, separate requirement.
- Blocking the removal. The warning informs a destructive confirm the user already has to pass; it never refuses.

