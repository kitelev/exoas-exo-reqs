---
exo__Asset_uid: cef65fd5-8bf0-4082-afcc-9d0fb3fe68ac
exo__Asset_createdAt: 2026-08-20T05:54:18
exo__Asset_updatedAt: 2026-08-20T05:54:18
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
exo__Asset_createdBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13]]"
exo__Asset_label: EARL-исход выводится из severity, а не назначается failed всем находкам
aliases:
  - EARL-исход выводится из severity, а не назначается failed всем находкам
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[cb2e9a63-081e-46fa-89b9-7ed479516a62|req__RequirementStatusProposed]]"
req__Requirement_covers: buildEARLReport отображает sh:Violation в earl:failed, а sh:Warning в earl:cantTell; conforms и код возврата не меняются
req__Requirement_author: "[[0aa339bc-9b56-400a-8148-cbde57bbf0b6|a.kitelev]]"
---

## Контекст

`validate schema --format earl` отображает **каждую** запись `report.violations` в `earl:failed`, независимо от `severity`. Предупреждения попадают на EARL-поверхность как отказы, тогда как `conforms` остаётся `true`, а код возврата — `0`.

⇒ **Три поверхности одной проверки противоречат друг другу**, и все три одновременно верными быть не могут.

## Где

`packages/cli/src/commands/validate-schema.ts` → `buildEARLReport`.

⛤ **`severity` доступна в том же объекте** и уже используется строкой ниже — для поля `sh:resultSeverity`. То есть данные для верного маппинга были на месте всегда; игнорировалось только `earl:outcome`.

## Свидетельство

Отчёт, чьи находки — два `sh:Warning`, даёт EARL-исходы `["earl:failed", "earl:failed"]` при `conforms: true`. До появления первого предупреждения та же команда эмитит одну запись `earl:passed` («All nodes conform to shapes») — то есть **приобретение предупреждения переворачивает ранее проходивший EARL-отчёт**.

## Масштаб — пред-существующий, не регрессия

Слепой к severity маппинг **старше** проверки коллизий term-IRI (PR #4070): `report.violations` уже несёт cross-vault unresolvable-ref предупреждения — **640** в vault-exodev на 2026-08-20. Значит правка меняет поведение **для всех** из них, а не для одной фичи.

⛤ Именно поэтому #4070 это не тронул: ограничение фаундера там было «предупреждение не должно переворачивать `conforms`», и оно выполняется на `json` и `text`; `earl` — единственная поверхность, где не выполняется.

## Почему важно

EARL существует, чтобы его **потреблял другой инструмент**. Потребитель, гейтящийся на `earl:failed`, видит жёсткий отказ там, где сам CLI классифицирует находку как не-отказ и не роняет собственный код возврата. Любой vault с одной cross-vault ссылкой уже сегодня отчитывается так.

## Требование

Исход EARL выводится из `severity`, а не назначается:

| `severity` | `earl:outcome` |
|---|---|
| `sh:Violation` | `earl:failed` |
| `sh:Warning` | `earl:cantTell` |

`earl:cantTell` — исход EARL для «утверждение сделано, но результат не является ни прохождением, ни отказом», что и есть смысл предупреждения SHACL.

## Gherkin

```gherkin
Feature: the EARL surface agrees with the other two

  Scenario: a warning is not reported as a failure
    Given a report whose only findings are sh:Warning
    When validate schema --format earl runs
    Then no assertion carries earl:failed
    And conforms stays true and the exit code stays 0
    # These three already agree on json and text; earl was the outlier.

  Scenario: a violation is still a failure
    Given a report containing an sh:Violation
    Then its assertion carries earl:failed
    # ~all existing consumers gate on this; it must not move.

  Scenario: a mixed report distinguishes the two
    Given a report with one sh:Violation and one sh:Warning
    Then exactly one assertion is earl:failed
    And the other is not
    # A fix that mapped everything to cantTell would pass a violations-only
    # axis and a warnings-only axis while being wrong.

  Scenario: a clean vault is unchanged
    Given no findings at all
    Then the single earl:passed assertion is emitted as before
```

## ⛤ Ось, без которой набор был бы вакуумным

Оси «только предупреждения» и «только нарушения» по отдельности проходят и у **неверной** реализации, отображающей всё в один исход. Различает их только **смешанный** отчёт — он и есть несущая ось.

Issue #4072.

