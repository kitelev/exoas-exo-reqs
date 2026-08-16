---
exo__Asset_uid: 1da8e1bf-22e7-4f2b-b0ec-86dde1adf3e8
exo__Asset_createdAt: 2026-08-16T11:41:43
exo__Asset_updatedAt: 2026-08-16T11:47:22
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
exo__Asset_createdBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13]]"
exo__Asset_label: "req(exo): DisplayNameResolver читает classTemplates как СОБСТВЕННОЕ свойство — метка класса, равная члену Object.prototype, больше не роняет резолвер TypeError (пятая и единственная БРОСАЮЩАЯ точка класса)"
aliases:
  - "req(exo): DisplayNameResolver читает classTemplates как СОБСТВЕННОЕ свойство — метка класса, равная члену Object.prototype, больше не роняет резолвер TypeError (пятая и единственная БРОСАЮЩАЯ точка класса)"
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[4bd932c2-2507-4a2d-b3f2-163e096bfa81|req__RequirementStatusApproved]]"
req__Requirement_priority: "[[481c3be1-d05c-4b78-8a3c-c61308d40bf1]]"
req__Requirement_bindingClass: "[[f8841786-64c2-42a9-8b45-2d33fd6be87c]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76]]"
req__Requirement_covers:
  - exo core — DisplayNameResolver.resolveRenderSpec читает settings.classTemplates как СОБСТВЕННОЕ свойство, поэтому метка класса с именем члена Object.prototype даёт undefined вместо унаследованной функции и резолвер не бросает TypeError
  - "exo core — штатный путь не меняется: собственный ключ в classTemplates применяется байт-в-байт, обычные метки классов резолвятся как сегодня"
---

# Job Story

Когда я открываю ассет, чей `exo__Instance_class` я задал сам, я хочу, чтобы резолвер имени **никогда не падал** из-за того, как названа метка класса, — потому что упавший резолвер ломает отображение имён целиком, а не портит одно имя.

# Контекст

`DisplayNameResolver.resolveRenderSpec` индексирует `this.settings.classTemplates` ключом, полностью авторским: `firstClass` приходит из `exo__Instance_class` через `cleanClassValue`, который возвращает **алиас после `|`**. `classTemplates` — обычный объектный литерал, с #3838 part 3 пустой по умолчанию (все per-class шаблоны переехали в vault-спеки). Пустой литерал всё равно наследует `Object.prototype`.

Замер (2026-08-16, воспроизведён дважды — ревьюером и мной независимо):

```
guard passes: true | typeof: function
downstream: TypeError: ct[k].trim is not a function
```

Гвард `if (firstClass && this.settings.classTemplates[firstClass])` **проходит**, потому что `Function.prototype.toString` истинен, и наружу уходит **функция** в поле `template`; затем `DisplayNameTemplateEngine:63` / `:565` зовут `this.template.trim()`.

## Чем этот случай отличается от четырёх уже закрытых

Класс тот же — «пользовательский ключ индексирует обычный объект», — и четыре его точки уже закрыты: реестр host-функций (req `5cd9fffe`, issue #4052), обход key-path (`4a2e6b80`, #4058), плоское чтение матчера (#4060), алиас блокера (#4062). Но у всех четырёх отказ **fail-closed**: не-строка отбрасывается ниже по течению (`extractClassKeys` → `[]`), и худший исход — «спека не участвует».

Здесь отказ **БРОСАЕТ**. Ни `BodyLinkPatch:389`, ни `GraphViewPatch:416` не оборачивают `resolve()` в try/catch — то есть непойманное исключение ломает именование целиком, ровно как об этом уже написано в докстринге `keyPathResolver`.

## Почему отдельным требованием, а не расширением `4a2e6b80`

По тому же дискриминатору, которым реестр отделялся от #4052 — отличаются **все три** оси:

| ось | `4a2e6b80` | здесь |
|---|---|---|
| объект | метаданные ассета | `settings.classTemplates` (настройки) |
| функция | `matcherSatisfied` / `resolveKeyPath` | `DisplayNameResolver.resolveRenderSpec` |
| поле | `exo__DisplayNameSpec_matchPath` | `exo__Instance_class` |

# Statement (Gherkin)

```gherkin
Scenario: метка класса, совпадающая с членом Object.prototype, не роняет резолвер
  Given ассет, чей exo__Instance_class записан как [[<uid>|toString]]
  When резолвится его отображаемое имя
  Then резолвер возвращает шаблон по умолчанию, а не бросает TypeError
  And то же для constructor, valueOf, hasOwnProperty, toLocaleString

Scenario: чтение classTemplates — только СОБСТВЕННОЕ свойство
  Given classTemplates не содержит ключа, равного метке класса
  When метка класса совпадает с именем члена Object.prototype
  Then значение читается как undefined, а не как унаследованная функция

Scenario: настроенный per-class шаблон продолжает применяться байт-в-байт
  Given classTemplates содержит СОБСТВЕННЫЙ ключ для метки класса
  When резолвится имя ассета этого класса
  Then применяется этот шаблон — изменение не касается штатного пути

Scenario: не-строковое значение не доезжает до движка шаблонов
  Given значение по ключу существует, но не является строкой
  When резолвится имя
  Then резолвер уходит на шаблон по умолчанию, а не передаёт значение дальше
```

# Verification

**Revert-verify (production-shape):** ось драйвит реальный `DisplayNameResolver.resolve` над ассетом, чей `exo__Instance_class` — `[[<uid>|toString]]`; ⛔ не через прямой вызов внутреннего метода, иначе ось не наблюдает маршрут.

- снять own-property-гвард → ось на `toString` **RED** (возвращается `TypeError`);
- CONTROL: собственный ключ в `classTemplates` продолжает применяться — зелен в обе стороны;
- CONTROL: обычная метка класса (`ems__Task`) резолвится как сегодня — зелен в обе стороны.

⛔ Ломать **копию** вне репозитория (`ZERO-GIT-TOUCH`), и прогнать немутированную копию первой: сьют, не запустившийся из-за резолюции модулей, рапортует `Tests: 0` и выглядит как «ничего не упало».

# Non-goals

- ⛔ **Не** менять семантику штатного пути: собственный ключ в `classTemplates` обязан применяться байт-в-байт.
- ⛔ **Не** трогать четыре уже закрытые точки того же класса (`5cd9fffe`, `4a2e6b80`, #4060, #4062).
- ⛔ **Не** оборачивать `resolve()` в try/catch у вызывающих: это спрятало бы класс, а не закрыло. Если такой barrier нужен — отдельное требование со своим обоснованием.
- Унификация трёх парсеров статуса (#4056) — отдельная работа.

# Известные границы

- Достижимость **низкая**: нужна метка класса, дословно равная члену `Object.prototype`, а метки следуют форме `prefix__Name`. Закрывается **по принципу**, как и сам `4a2e6b80`, — сказано вслух, а не подразумевается.
- Изменение **видимо** только для такой метки; для всех остальных путь байт-в-байт прежний.
- ⛔ Считать инцидентность по СЫРОМУ frontmatter, не по SPARQL: store эмитит таргет wikilink и **выбрасывает алиас**, а `cleanClassValue` читает ключ именно из алиаса, — то есть SPARQL-замер этого вопроса структурно слеп и вернёт ноль при любых данных.

