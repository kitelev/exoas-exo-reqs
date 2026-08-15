---
exo__Asset_uid: 5cd9fffe-1fa5-4fda-8e2a-bfe6d4c88379
exo__Asset_createdAt: 2026-08-15T19:24:16
exo__Asset_updatedAt: 2026-08-15T19:24:16
exo__Instance_class:
  - "[[8c5af681-3413-4219-8636-0ac229d1b253]]"
exo__Asset_createdBy: "[[4ef3962d-b8a7-42b5-bd28-88ec846f1d13]]"
exo__Asset_label: "req(exo): host-функции displayName (isEffortBlocked, isEpisodeOngoing) переезжают в core за уже существующий VaultMetadataPort, CLI их регистрирует — оракул имени перестаёт молча пропускать 2 спеки из 35 (83 ассета); потребители плагина не правятся"
aliases:
  - "req(exo): host-функции displayName (isEffortBlocked, isEpisodeOngoing) переезжают в core за уже существующий VaultMetadataPort, CLI их регистрирует — оракул имени перестаёт молча пропускать 2 спеки из 35 (83 ассета); потребители плагина не правятся"
exo__Asset_isDefinedBy: "[[a64ca05b-ed45-4fbc-a8a9-54f9cfcf895c]]"
req__Requirement_status: "[[4bd932c2-2507-4a2d-b3f2-163e096bfa81|req__RequirementStatusApproved]]"
req__Requirement_priority: "[[481c3be1-d05c-4b78-8a3c-c61308d40bf1|req__RequirementPriorityP2]]"
req__Requirement_bindingClass: "[[f8841786-64c2-42a9-8b45-2d33fd6be87c|req__RequirementBindingClassIntegration]]"
req__Requirement_area: "[[bd76637d-5788-4c30-a3d8-c88dcdd9970f|Exocortex Development]]"
req__Requirement_author: "[[de20a3f1-7483-4714-ab28-b45f5cf02c76|ExoAssistant]]"
req__Requirement_covers:
  - exo core — предикаты displayName-матчеров (isEffortBlocked, isEpisodeOngoing) живут в packages/core и обращаются к vault ТОЛЬКО через VaultMetadataPort.resolveLinkpathFrontmatter, введённый req f17f7c57; isEpisodeOngoing не обращается к vault вовсе
  - exo cli — resolve-display-name регистрирует обе host-функции, поэтому спека с _matchHostFunction участвует в составлении имени наравне с плагином; незнакомое имя функции по-прежнему fail-closed (спека молча не участвует, команда не падает)
  - "exo plugin — BlockerHelpers и EpisodePeriodHelpers сохраняют прежние сигнатуры и делегируют в core через Obsidian-адаптер: ни один из 4+4 потребителей BlockerHelpers и 1+1 потребителей EpisodePeriodHelpers не правится"
---

# Job Story

Когда я спрашиваю CLI, как называется ассет, я хочу получить **то же** имя, что покажет Obsidian — включая правила, которые смотрят на **соседний ассет** или на **сегодняшнюю дату**, — чтобы оракул был полным, а не «полным на 33 правила из 35».

# Контекст

Req `f17f7c57` отгрузил оракул `resolve-display-name` и **явно** вывел host-функции за скоуп («Host-функции в CLI не регистрируются … отдельная работа»). Это требование — та самая отдельная работа; по дисциплине `/feature-sdd` оно оформлено **новым sibling-req**, а не расширением покрытия `f17f7c57`, потому что тот req требовал текущее поведение как сценарий и отложил изменение.

## Что именно не работает (замер 2026-08-15)

Спека `exo__DisplayNameSpec` может объявить `_matchHostFunction` — предикат, который **нельзя** выразить сравнением значений, потому что он смотрит наружу: на другой ассет либо на ambient-величину (сегодня). Таких спек **2 из 35**:

| host-функция | что спрашивает | почему не value-matcher |
|---|---|---|
| `isEffortBlocked` | «этот эффорт заблокирован?» | надо резолвить `ems__Effort_blocker` и прочитать статус **другого** ассета |
| `isEpisodeOngoing` | «этот эпизод идёт сейчас?» | комперанд — TODAY, его нет ни в одном frontmatter |

Реестр host-функций **инъектируется из композишн-рута плагина**; CLI не передаёт его вовсе, а движок **fail-closed** ⇒ спека с незарегистрированным именем просто не участвует. CLI не врёт — он **недоговаривает, и молча**.

**Затронуто ассетов: 83** — 74 эффорта несут `ems__Effort_blocker` (58 одиночных + 16 многозначных/пустых) и 9 `life__Episode` несут `life__Episode_start`. Канарейка замера: 35 `exo__DisplayNameSpec` всего, из них 2 с `_matchHostFunction`.

## Замер связанности (проведён ДО формулировки требования)

| функция | обращений к `app` | какие именно | строк | импортёров (src + тесты) |
|---|---|---|---|---|
| `EpisodePeriodHelpers.isEpisodeOngoing` | **0** | — (читает своё же поле + `new Date()`) | 114 | 1 + 1 |
| `BlockerHelpers.isEffortBlocked` | **2** | `getFirstLinkpathDest(p,"")` → `getFileCache(f)?.frontmatter` | 44 | 4 + 4 |

⛤ **Оба вызова `isEffortBlocked` — это ровно `VaultMetadataPort.resolveLinkpathFrontmatter(linkpath)`**, метод, уже существующий в core после `f17f7c57`. Ничего нового к порту добавлять не нужно; `EffortStatus` там уже импортируется **из core**. Т.е. это перенос, а не переписывание.

⛤ У `BlockerHelpers` **4 src-потребителя** (`ExocortexAPI`, `DailyTasksRenderer`, `RelationsRenderer`, `AssetMetadataService`) — выше cascade-cap. Их не правим: плагинская обёртка сохраняет сигнатуру `(app, metadata)` и делегирует в core через Obsidian-адаптер — **тот же приём шима**, которым `f17f7c57` дал 0 правок у 30 импортёров.

# Scenarios

## Scenario 1: CLI применяет спеку с host-функцией
Given vault со спекой `exo__DisplayNameSpec`, объявляющей `_matchHostFunction: isEffortBlocked`, и эффортом, чей блокер активен
When я выполняю `exocortex resolve-display-name <путь> --vault <V> --json`
Then имя составлено **с** вкладом этой спеки
  And `source` = `spec`

## Scenario 2: обе поверхности через ОДИН код
Given host-функции живут в `packages/core` и вызываются через `VaultMetadataPort`
When плагин и CLI резолвят имя одного и того же ассета
Then обе поверхности проходят через **одну и ту же** реализацию предиката
  And расхождение вывода означает расхождение **адаптеров**, а не двух реализаций

## Scenario 3: ноль правок у потребителей
Given `BlockerHelpers` импортируют 4 src-файла плагина и 4 теста, `EpisodePeriodHelpers` — 1 и 1
When предикаты перенесены в core, а на прежних путях оставлены делегирующие обёртки
Then ни один импортёр не изменён
  And сигнатура `(app, metadata)` на стороне плагина сохранена

## Scenario 4: fail-closed сохраняется
Given спека объявляет `_matchHostFunction` с именем, которого нет в реестре
When CLI резолвит имя такого ассета
Then спека **не участвует** (движок fail-closed), команда не падает
  And имя составляется остальными правилами

## Scenario 5: `isEpisodeOngoing` не требует vault-доступа
Given предикат читает только собственный период инстанса и текущую дату
When он вызван в core
Then он **не обращается** к `VaultMetadataPort` вовсе

# Non-goals

- ⛔ **Не чинить dual-IRI дефект `isEffortBlocked`** (см. «Известные границы»). Перенос — **предусловие** его починки: после переноса один фикс в core лечит обе поверхности разом. Фикс меняет то, что пользователь видит в vault ⇒ свой req.
- ⛔ **Не расширять сигнатуру реестра** параметрами. Обе функции сегодня хардкодят имена свойств, которые читают; параметризация оправдана, когда второй period-подобный класс захочет тот же маркер.
- **Не менять семантику отображения.** Перенос обязан быть поведенчески нейтральным; любое расхождение вывода — дефект переноса, а не улучшение.

# Известные границы

- ⛔ **CLI унаследует живой дефект `isEffortBlocked`, и это НАМЕРЕННО — в этом и состоит паритет.** Предикат сравнивает статус блокера со строкой `ems__EffortStatusDone`, но `exocortex-cli` пишет статус **голым `[[uid]]`**; после strip'а брекетов остаётся UID, сравнение не срабатывает, и **завершённый блокер считается активным**. Замер 2026-08-15 на живых данных: у 49 из 58 одиночных блокеров статус в bare-UID-форме; **8 эффортов** прямо сейчас несут 🚩 ошибочно. Это класс dual-IRI, лечится матчем **обеих** форм — отдельным req.
- `EpisodePeriodHelpers.localToday` берёт **локальную** дату процесса. У CLI и у плагина это одна машина, но при запуске из CI в UTC день может отличаться.
- Многозначный `ems__Effort_blocker` (16 из 74) сворачивается `String(...)` в строку с запятыми и ссылку не резолвит ⇒ предикат вернёт «не заблокирован». Тоже пере-носится вербатим.

