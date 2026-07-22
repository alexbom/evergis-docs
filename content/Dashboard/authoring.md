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
| `TwoColumn`, `OneColumn` | `alias`, `value`, `units` |
| `Camera` | `alias`, `value` |
| `Icon` | `icon`, `alias`, `link`, `text` |
| `Image` | `alias`, `text`, `button`, `image` |
| `Attachment` (без `relatedDataSource`) | `value` |
| `Edit*` | `alias` |
| `DataSource`, `DataSourceProgress` | slot-id **внутреннего шаблона** `options.innerTemplateName` (не собственные слоты хоста) — см. раздел ниже |
| `FeatureCardBackgroundHeader` | `title`, `description`, `bgImage`, `icon` |
| `FeatureCardSlideshowHeader` | `title`, `description`, `bgImage`, `slideshow` |
| `DashboardDefaultHeader`, `FeatureCardDefaultHeader` | `title`, `description`, `icon` |

**Перечисляемые контейнеры** — дети имеют произвольный уникальный `id` (не slot из фикс. набора):

| Контейнер | Дети | Доп. требование |
|---|---|---|
| `Tabs` | табы с уникальным `id` (тип `TabId`) | — |
| `AddFeature` | кнопки с уникальным `id` | — |
| `Filters` | фильтры с уникальным `id` | у каждого обязателен `options.filterName` |

## DataSource-хосты и `innerTemplateName`

Контейнеры `DataSource` и `DataSourceProgress` не рендерят детей напрямую — они проходят по **каждой записи** (`feature`) источника и рендерят её через **внутренний шаблон**, заданный `options.innerTemplateName` (пайплайн конвертирует имя в проп `innerComponent` через `getContainerComponent`). Поэтому:

- **`options.innerTemplateName` обязателен.** Без него `getContainerComponent(undefined) === null` → ни одна запись не рендерится, контейнер визуально пуст. Обязательность **типами не ловится**: поле объявлено в `ConfigMiscOptions` и, хотя и входит в `DataSourceContainerOptions`/`DataSourceProgressContainerOptions`, остаётся необязательным.
- **`children` хоста — это slot-id выбранного внутреннего шаблона**, а не собственные слоты `DataSource`.
- Неизвестное имя шаблона откатывается на `ContainersGroup` (реестровый `default`).

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
- [ ] Все ссылки `chartId` / `tabId` / `modalId` / `downloadById` резолвятся в существующий `id`.
- [ ] Новые `id` не конфликтуют с `id` из уже существующего конфига.

## Для TS-авторинга — строгие типы

Если конфиг пишется на TypeScript, типизируй узлы строгим authoring-типом — тогда пропуск `id` станет **ошибкой компиляции** (базовый `ConfigContainerChild.id` опционален и пропуск не ловит):

- `StrictConfigContainerChild` — `id` обязателен рекурсивно у узла и всех детей;
- `StrictDashboardChild` — то же на верхнем уровне, но с сужением дискриминированного union (проверка полей `options` по `type`/`templateName`).

Оба экспортируются из `@evergis/react`. Для JSON-конфигов типы не работают — их закрывает рантайм-валидатор `validateDashboardConfig` и скилл `dashboard-container-gen`.

## Связанные разделы

[[concepts|Основные понятия]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[headers|Шапки]] | [[types|Типы]]
