# Правила генерации конфигов

> [!danger] Правило №1 — у КАЖДОГО узла конфига обязателен `id`
> Инвариант методологии Dashboard. Любой узел (контейнер, элемент, фильтр, таб, кнопка) **без `id` молча не рендерится**: движок не найдёт контейнер в конфиге, не поместит элемент в нужный slot, не адресует сущность в коллекции. **Типы это НЕ ловят** — `ConfigContainerChild.id` опционален ради legacy-совместимости, поэтому ответственность целиком на том, кто генерирует конфиг. Это правило №1 при любой генерации.

Эту страницу читают **первой** перед генерацией любого конфига дашборда или карточки объекта (FeatureCard). Полные определения — в [[concepts|Основных понятиях]] («ID контейнеров и элементов»), типы — в [[types|Типах]].

## Три смысла `id`

Смысл `id` зависит от роли узла:

| Роль узла | Что такое `id` | Уникальность |
|---|---|---|
| **Контейнер** (узел с `templateName`) | глобально уникальное имя (`page_1`, `chart_floors`) — по нему работают навигация и связи | глобально в `config` |
| **Элемент** (узел с `type` внутри `children`) | **slot** — ключ из фиксированного набора родителя (`alias`, `chart`, `value`, ...) | в пределах `children` |
| **Перечисляемая сущность** (фильтр / таб / кнопка) | уникальное имя в коллекции `children` | в пределах `children` |

## Фиксированные slot-id по контейнерам

У элементов `id` обязан совпадать со slot-id, который ожидает контейнер-родитель. Неверный slot → элемент не отрисуется (`renderElement({ id })` вернёт `null`).

| Контейнер / шапка | Допустимые slot-id |
|---|---|
| `Chart` | `alias`, `chart`, `legend`, `title`, `titleIcon` |
| `TwoColumn` | `alias`, `value`, `units`, `icon`, `tooltip`, `modal` |
| `OneColumn` | `alias`, `value`, `units`, `tooltip`, `modal` |
| `Camera` | `alias`, `value` |
| `Icon` | `icon`, `alias`, `link`, `text` |
| `Image` | `alias`, `text`, `button`, `image` |
| `Slideshow` | `slideshow`, `alias` (опционален) |
| `StructuredData` | `data` — `type: "table"` (обязателен), `alias` |
| `Upload` | `uploader` |
| `Attachment` | `alias`; `value` — если не задан `relatedDataSource` |
| `Edit` (базовый) | `alias`, `value` |
| `EditString`, `EditNumber`, `EditBoolean`, `EditDropdown`, `EditChips`, `EditCheckbox`, `EditDate` | `alias`, `tooltip` (контрол встроен — слота `value` нет) |
| `EditAttachment` | `alias` |
| `EditGroup` | `alias`, `tooltip`, `units`, `icon` |
| `DataSource`, `DataSourceProgress` | slot-id **внутреннего шаблона** `options.innerTemplateName` (не собственные слоты хоста) — см. раздел ниже |
| `Vote` | собственных слотов нет — только универсальные `title`/`titleIcon`/`bgImage`; на узле обязателен `attributeName` |
| `FeatureCardBackgroundHeader` | `title`, `description`, `bgImage`, `icon` |
| `FeatureCardSlideshowHeader` | `title`, `description`, `bgImage`, `slideshow` |
| `DashboardDefaultHeader` | `title`, `icon`, `image` |
| `FeatureCardDefaultHeader` | — кастомных детей нет, структура фиксирована |

### Универсальные слоты

Три slot-id допустимы у **любого** контейнера независимо от его набора, поэтому в таблицу выше не включены:

| Slot-id | Где рендерится | Ограничение |
|---|---|---|
| `title` | заголовок контейнера (`ExpandableTitle`) | нужен контейнер с заголовком |
| `titleIcon` | иконка заголовка | там же |
| `bgImage` | отдельный слой фона **под** содержимым (`ContainerBackground`) | у любого контейнера, **кроме `Divider`** |

Ни один из трёх не рендерится в теле контейнера и не занимает трек сетки (`NON_TRACK_SLOT_IDS`). Слот `icon` на уровне контейнера тоже уходит в заголовок, но, в отличие от этих трёх, входит в наборы слотов конкретных контейнеров.

`bgImage` — это обычный элемент `type: "image"` (URL в корневом `value`, `attributeName` или `options.resourceId`). Слоя нет в DOM вовсе, пока слот не объявлен в конфиге: пустой абсолютный `div` не создаётся. По умолчанию картинка растягивается `object-fit: cover`; перебить можно `options.fit` у самого элемента.

К слоту прилагаются две опции, тоже допустимые у **любого** контейнера (их не Pick'ает ни один `<Name>Options` — читают хук хоста и слой фона):

| Опция | Что делает | Оговорка |
|---|---|---|
| `options.innerPadding` | Внутренний отступ `1rem` по краям корня — содержимое не липнет к краям картинки | Работает и без `bgImage`. Для произвольного отступа — `options.padding` |
| `options.outflow` | Слой фона вытекает за края на `1.5rem` по бокам и вверх (вниз — никогда) | Без слота `bgImage` бессмысленна; обрезается предком с `overflow` кроме `visible` — в том числе собственным `options.overflow` |

```tsx
{
  id: "stats_card",
  templateName: "ContainersGroup",
  options: { column: true, height: 240, innerPadding: true, outflow: true },
  children: [
    { id: "bgImage", type: "image", options: { resourceId: "1f2e...", fit: "contain" } },
    { id: "total_value", templateName: "OneColumn", children: [{ id: "value", attributeName: "total" }] },
  ],
}
```

> [!warning] `bgImage` у `DataSource` / `DataSourceProgress` остаётся на хосте
> У DataSource-хостов `children` — слоты внутреннего шаблона, поэтому слот фона достался бы и хосту, и каждой записи: одна картинка нарисовалась бы N+1 раз. `DataSourceInnerContainer` вырезает `bgImage` из конфига записи — фон принадлежит хосту, записи его не наследуют.

**Перечисляемые контейнеры** — дети имеют произвольный уникальный `id` (не slot из фикс. набора):

| Контейнер | Дети | Доп. требование |
|---|---|---|
| `Tabs` | табы с уникальным `id` (тип `TabId`) | — |
| `AddFeature` | кнопки с уникальным `id` | — |
| `Filters` | фильтры с уникальным `id` | у каждого обязателен `options.filterName` |
| `ContainersGroup` с `options.grid` | строки `GridRow` с уникальным `id` | см. раздел про сетку ниже |
| `GridRow` | ячейки `ContainersGroup` с уникальным `id` | у каждой — `options.width` в `fr` |

## Сетка (`options.grid`)

Строгая трёхуровневая структура — нарушение любого пункта ломает раскладку молча:

1. **Сетка → строки → ячейки.** Дети сетки обязаны быть `templateName: "GridRow"`, дети строки — `templateName: "ContainersGroup"`. Промежуточных узлов быть не может, «ячейка сразу в сетке» не работает.
2. **Доли только в `fr`.** У строки `options.height: "1fr"`, у ячейки `options.width: "2fr"`. Значения в px и процентах сетка не поддерживает — они считаются за `1fr`.
3. **У внешней сетки обязательна `options.height`** (или определённая высота у родителя). Иначе `1fr`-строки раскладываются по содержимому, а не пропорционально. Строкам и вложенным сеткам высоту задавать не нужно — они по умолчанию занимают выделенный им трек целиком.
4. **Вложенность — через `grid` у ячейки.** Ячейка с `options.grid: true` сама становится сеткой, и её дети снова обязаны быть строками.
5. **Универсальные слоты не занимают треки.** Дети с `id` из `title` / `icon` / `titleIcon` уходят в заголовок, `bgImage` — в слой фона; треки считаются по остальным (`NON_TRACK_SLOT_IDS`). У строки заголовка нет вовсе.
6. **`id` уникальны глобально.** Правило №1 здесь особенно чувствительно: `returnFound` ищет вглубь, и на дубликате движок молча возьмёт первое совпадение. Автогенерируемые id имеют вид `gridRow_N` / `gridCell_N`.
7. **`id` ячейки не должен начинаться с `page`** — по этому префиксу контейнер опознаётся как корневой блок страницы и оборачивается в карточку.

По умолчанию сетка бесшовная: `gap` равен нулю, ячейки стыкуются вплотную и ничем себя не обозначают. Зазор задаётся явно — у сетки он разводит строки, у строки — ячейки.

Две опции меняют договор о высоте — ставить их надо осознанно:

| Опция | Где читается | Что делает |
|---|---|---|
| `autoHeight` | у **каждого** узла цепочки (сетка, `GridRow`, вложенная сетка) | Секция растёт под содержимое: `height` становится минимумом, треки — `minmax(auto, Nfr)`. Включённая только у корня — не сработает, рост упрётся в первую же строку |
| `fixedHeight` | только у **корневого** узла сетки | Убирает ручку нижней границы в `editMode`: высотой распоряжается внешняя раскладка (панель, док), тянуть её мышью нельзя |

```tsx
{
  id: "main_grid",
  templateName: "ContainersGroup",
  options: { grid: true, height: "22rem" },
  children: [
    { id: "grid_row_1", templateName: "GridRow", options: { height: "1fr" }, children: [
      { id: "grid_cell_a", templateName: "ContainersGroup", options: { width: "2fr" }, children: [chart] },
      { id: "grid_cell_b", templateName: "ContainersGroup", options: { width: "1fr" }, children: [filters] }
    ]}
  ]
}
```

## DataSource-хосты и `innerTemplateName`

Контейнеры `DataSource` и `DataSourceProgress` не рендерят детей напрямую — они проходят по **каждой записи** (`feature`) источника и рендерят её через **внутренний шаблон**, заданный `options.innerTemplateName` (пайплайн конвертирует имя в проп `innerComponent` через `getContainerComponent`). Поэтому:

- **`options.innerTemplateName` обязателен.** Без него `getContainerComponent(undefined) === null` → ни одна запись не рендерится, контейнер визуально пуст. Обязательность **типами не ловится**: поле объявлено в `ConfigMiscOptions` и, хотя и входит в `DataSourceContainerOptions`/`DataSourceProgressContainerOptions`, остаётся необязательным.
- **`children` хоста — это slot-id выбранного внутреннего шаблона**, а не собственные слоты `DataSource`.
- Неизвестное имя шаблона откатывается на `ContainersGroup` (реестровый `default`).
- Оба правила проверяет рантайм-валидатор `validateDashboardConfig`: пропуск опции — ошибка `missing-inner-template`, чужой slot у ребёнка — `unexpected-slot` с перечнем слотов **внутреннего** шаблона.

| Внутренний шаблон (`innerTemplateName`) | Его слоты (`children` хоста) |
|---|---|
| `RoundedBackground` | `icon`, `alias`, `value`, `units` |
| `Progress` (обычно для `DataSourceProgress`) | `icon`, `alias`, `value`, `units` |
| `OneColumn`, `TwoColumn` | `alias`, `value`, `units` |
| `ContainersGroup` | произвольная вёрстка (дети — вложенные контейнеры, а не слоты) |

## Фильтры

У фильтра в `FiltersContainer` — **два разных ключа, оба обязательны**:
- `id` — идентифицирует узел в `children` (для React key и адресации),
- `options.filterName` — связывает фильтр со страничным `ConfigFilter.name` (подстановка `%filterName` в `condition`, сброс).

### Фильтр-приёмник `StructuredData`

Контейнер [[containers#StructuredDataContainer|StructuredData]] не читает фильтр, а **пишет** в него результат правки. Пара нужна только редактируемой таблице: без `options.editMode` править нечего, и фильтр не понадобится. Требования к паре:

- `options.editMode` и `options.filterName` у контейнера — оба обязательны;
- фильтр с таким `name` объявлен в `filters` страницы и имеет `valueType: "features"`. Без этого вид значения определяется структурной догадкой, и FeatureCollection может быть разобран как значение tree-фильтра;
- плейсхолдер `%<filterName>` ставится только в `parameters` потребителя (питон-таска, url-источник). В `condition` он не подставляется никогда и останется в запросе нетронутым.

```jsonc
// filters страницы
{ "name": "type_1", "valueType": "features" }

// потребитель — параметры таски
{ "parameters": { "rows": "%type_1" } }
```

## Ссылки между узлами (entity-ref)

Поля `options`, ссылающиеся на `id`, должны резолвиться в существующий узел:

| Поле | Ссылается на |
|---|---|
| `options.chartId` | `id` `ChartContainer` или slot-id дочернего чарта (`"chart"`) |
| `options.tabId` | `id` вкладки в `TabsContainer` |
| `options.modalId` | `id` модала в `config.modals[]` |
| `options.downloadById` | `id` `ExportPdfContainer` |

## ❌ / ✅

```ts
// ❌ так НЕ работает — узлы без id: контейнер и его элементы не отрендерятся
{
  templateName: "Chart",
  children: [{ type: "chart" }, { type: "legend", options: { chartId: "chart" } }],
}

// ✅ так работает — id у контейнера и корректные slot-id у элементов
{
  id: "chart_floors",
  templateName: "Chart",
  options: { twoColumns: true },
  children: [
    { id: "alias", value: "Этажность" },
    { id: "chart", type: "chart", options: { chartType: "bar" } },
    { id: "legend", type: "legend", options: { chartId: "chart" } },
  ],
}
```

```ts
// ❌ фильтр без filterName — не подставится в condition
{ id: "page_filters", templateName: "Filters", children: [{ id: "f1", type: "rangeNumber" }] }

// ✅ у фильтра есть и id, и options.filterName
{
  id: "page_filters",
  templateName: "Filters",
  children: [{ id: "floors_filter", type: "rangeNumber", options: { filterName: "floors" } }],
}
```

```ts
// ❌ DataSource без innerTemplateName — записи источника не рендерятся, контейнер пуст
{
  id: "buildings_list",
  templateName: "DataSource",
  options: { relatedDataSource: "buildings_ds" },
  children: [{ id: "alias", attributeName: "name" }, { id: "value", attributeName: "floors" }],
}

// ✅ задан innerTemplateName, а children — слоты этого внутреннего шаблона
{
  id: "buildings_list",
  templateName: "DataSource",
  options: { relatedDataSource: "buildings_ds", innerTemplateName: "RoundedBackground" },
  children: [
    { id: "icon", type: "icon", options: { icon: "building" } },
    { id: "alias", attributeName: "name" },
    { id: "value", attributeName: "floors" },
  ],
}
```

## Чек-лист перед выдачей конфига

Прогони сгенерированный конфиг по пунктам — **прежде чем отдавать**:

- [ ] У **каждого** контейнера (узел с `templateName`) есть уникальный `id`.
- [ ] У **каждого** элемента (узел с `type`) есть `id`, равный корректному slot-id родителя (см. таблицу).
- [ ] У **каждой** перечисляемой сущности (таб / кнопка / фильтр) есть уникальный `id`.
- [ ] У **каждого** фильтра дополнительно есть `options.filterName`.
- [ ] У **каждого** `DataSource` / `DataSourceProgress` задан `options.innerTemplateName`, а slot-id детей соответствуют этому внутреннему шаблону.
- [ ] Слот `bgImage` (если он есть) стоит у контейнера, а не у `Divider`, и не продублирован внутрь записей DataSource-хоста.
- [ ] `options.outflow` задан только там, где у контейнера есть слот `bgImage` (иначе валидатор отдаёт `orphan-option` — вытекать нечему).
- [ ] У `Vote` заданы свойство узла `attributeName` и все четыре слоя (`categoryDataSource`, `questionDataSource`, `variantDataSource`, `answerDataSource`) — это **имена слоёв**, а не источников данных страницы (иначе валидатор отдаёт `missing-vote-binding` / `missing-vote-source`).
- [ ] Все ссылки `chartId` / `tabId` / `modalId` / `downloadById` резолвятся в существующий `id`.
- [ ] Новые `id` не конфликтуют с `id` из уже существующего конфига.

## Для TS-авторинга — строгие типы

Если конфиг пишется на TypeScript, типизируй узлы строгим authoring-типом — тогда пропуск `id` станет **ошибкой компиляции** (базовый `ConfigContainerChild.id` опционален и пропуск не ловит):

- `StrictConfigContainerChild` — `id` обязателен рекурсивно у узла и всех детей;
- `StrictDashboardChild` — то же на верхнем уровне, но с сужением дискриминированного union (проверка полей `options` по `type`/`templateName`).

Оба экспортируются из `@evergis/react`. Для JSON-конфигов типы не работают — их закрывает рантайм-валидатор `validateDashboardConfig` и скилл `dashboard-container-gen`.

## Связанные разделы

[[concepts|Основные понятия]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[headers|Шапки]] | [[types|Типы]]
