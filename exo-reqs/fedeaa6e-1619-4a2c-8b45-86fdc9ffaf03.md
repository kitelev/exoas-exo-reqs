---
exo__Asset_uid: fedeaa6e-1619-4a2c-8b45-86fdc9ffaf03
exo__Asset_createdAt: 2026-08-12T22:48:28
exo__Asset_updatedAt: 2026-08-13T08:38:35
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
exo__Asset_createdBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13]]"
exo__Asset_label: "req(exo): matchPath supports a dot-notation key-path (cross-asset)"
aliases:
  - "req(exo): matchPath supports a dot-notation key-path (cross-asset)"
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[fccf8fa4-8004-41ee-9102-595a588e9be7|req__RequirementStatusActive]]"
req__Requirement_priority: "[[2c58b8ec-8a68-463b-a694-dfe6afeb861b|req__RequirementPriorityP1]]"
req__Requirement_bindingClass: "[[f8841786-64c2-42a9-8b45-2d33fd6be87c|req__RequirementBindingClassIntegration]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers: exo displayName — exo__DisplayNameSpec_matchPath resolves a dot-notation key-path across a wikilink reference, in both bare and aliased forms
req__Requirement_approvedBy: "[[0aa339bc-9b56-400a-8148-cbde57bbf0b6|a.kitelev]]"
req__Requirement_approvedAt: 2026-08-12T22:52:55+0500
req__Requirement_implementedBy: "PR #4040 (kitelev/exocortex) — squash f54ce463; follow-up #4041"
---

# req(exo): `exo__DisplayNameSpec_matchPath` supports a dot-notation key-path (cross-asset)

## Job Story

Когда я задаю условную displayName-спеку для инстансов, порождённых из прототипов одного
класса, я хочу, чтобы `exo__DisplayNameSpec_matchPath` умел адресовать свойство **связанного**
ассета (`exo__Asset_prototype.exo__Instance_class`) — чтобы ОДНА спека покрывала все прототипы
класса, а не требовала по отдельной спеке (плюс её части) на каждый прототип.

Конкретный триггер: displayName инстансов встреч должен рендериться как
`<label прототипа> <дата из ems__Effort_startTimestamp>` для **всех** инстансов прототипов
класса `ems__MeetingPrototype`. Сегодня `matchValue` принимает РОВНО ОДНО значение
(v2 single-value slice), поэтому покрытие N прототипов стоит N спек.

## Механизм (что уже есть, что меняется)

- `resolvePropertyKey` **уже** пропускает dot-path: алиас `[[uid|a.b]]` берётся дословно, так что
  key-path доезжает до `matcher.matchKey` без изменений.
- Ломается только ЧТЕНИЕ: `matcherSatisfied` делает плоский `metadata[matcher.matchKey]`
  (единственная строка в кодовой базе, читающая `matchKey`).
- Механизм cross-asset dot-notation **уже реализован** для шаблонов
  (`DisplayNameTemplateEngine.getNestedValue`) и имеет живой прецедент авторинга через алиас:
  `exo__PrintedProperty_property: "[[5f830626-…|exo__Statement_predicate.exo__Property_displayName]]"`.

## Scenarios

### Scenario 1 — dot-notation key-path матчит свойство связанного ассета

```gherkin
Given спека exo__DisplayNameSpec с appliesToClass = ems__Meeting,
  matchPath = "[[<property-uid>|exo__Asset_prototype.exo__Instance_class]]"
  и matchValue = "[[<ems__MeetingPrototype uid>]]"
When рендерится инстанс ems__Meeting, чей exo__Asset_prototype указывает на ассет
  класса ems__MeetingPrototype
Then спека участвует в выборе шаблона (её шаблон применяется)
```

### Scenario 2 — fail-closed, когда связанный ассет другого класса или ссылки нет

```gherkin
Given ту же спеку
When рендерится инстанс ems__Meeting, чей exo__Asset_prototype указывает на ассет
  ДРУГОГО класса, ЛИБО у которого exo__Asset_prototype отсутствует
Then спека НЕ участвует
```

### Scenario 3 — обе формы записи wikilink резолвятся одинаково

```gherkin
Given ту же спеку
When значение exo__Asset_prototype записано как "[[<uid>]]" ЛИБО как "[[<uid>|<label>]]"
Then спека участвует в ОБОИХ случаях — форма записи ссылки не влияет на исход матча
```

Основание: резолвер, дереферящий wikilink при обходе key-path, сегодня снимает только
обрамляющие скобки и кавычки и НЕ срезает `|alias`, поэтому aliased-форма даёт молчаливый
не-матч. Это ровно тот dual-IRI класс, который корпус требует закрывать обеими формами сразу.

### Scenario 4 — однокомпонентный matchPath байт-в-байт как раньше (zero-regression)

```gherkin
Given спеку, чей matchPath НЕ содержит точки (например ems__Effort_status)
When рендерится любой инстанс
Then исход матча тождественен поведению до изменения
```

## Non-goals

- НЕ нормализовать значение и НЕ менять эмиссию триплов.
- НЕ вводить конъюнкции/дизъюнкции матчеров — v2 single-matcher slice сохраняется
  (`exo__DisplayNameMatcher` остаётся будущей работой).
- НЕ трогать `matchHostFunction` и его fail-closed семантику.

## Verification

Integration-тест в `packages/obsidian-plugin/tests`, драйвящий реальный
`PrintNameRuleService.scanVault` + `getParticipatingRules` над production-shape фикстурой
(инстанс + прототип + класс-деф), с осями revert-verify:

1. нейтрализовать резолв key-path → Scenario 1/3 RED, Scenario 4 GREEN;
2. нейтрализовать срез `|alias` → Scenario 3 RED, Scenario 1 GREEN.

