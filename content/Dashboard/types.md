# Типы

## Обзор

Типизация Dashboard построена в три слоя:

1. **Per-component типы** (`componentTypes.ts`) — для каждого контейнера, элемента и шапки определены три связанные сущности: `<Name>Options` (Pick от `ConfigOptions`), `<Name>Config` (наследник `ConfigContainerChild`/`ConfigContainer`/`ConfigContainerHeader` с литеральным дискриминатором), `<Name>Props` (наследник `ContainerProps`).
2. **Branded keyspaces** (`branded.ts`) — TS-only бренды на строку (`ChartId`, `ModalId`, `LayerName`, ...) для защиты от перепутывания entity-id.
3. **Типизированные реестры** (`ContainerComponentRegistry`, `ElementComponentRegistry`) — `as const satisfies` контракт между ключом (`ContainerTemplate` / `ConfigElementType`) и React-компонентом с правильными props.

Доменная группировка опций — в [[options|Опциях]].

---

## Branded types

Branded type — это строка с уникальным «брендом» на этапе компиляции:

```ts
declare const brand: unique symbol;
export type Brand<T, B extends string> = T & { readonly [brand]: B };
```

Идея: на этапе компиляции `ChartId` и `ModalId` несовместимы (`findChart(modalId)` даёт TS-ошибку), а в runtime — обычные строки (JSON, Redux, fetch — всё работает без обёрток).

### Таблица типов

| Тип | Базовый | Конструктор | Keyspace |
|---|---|---|---|
| `ContainerId` | `Brand<string, "ContainerId">` | `asContainerId(value)` | `ConfigContainer.id` |
| `ChartId` | `ContainerId & Brand<..., "ChartId">` | `asChartId(value)` | Подтип ContainerId — id `ChartContainer` |
| `ModalId` | `ContainerId & Brand<..., "ModalId">` | `asModalId(value)` | Подтип ContainerId — id `ConfigModal` |
| `TabId` | `ContainerId & Brand<..., "TabId">` | `asTabId(value)` | Подтип ContainerId — id вкладки в `TabsContainer` |
| `FilterName` | `Brand<string, "FilterName">` | `asFilterName(value)` | Имя `ConfigFilter` |
| `LayerName` | `Brand<string, "LayerName">` | `asLayerName(value)` | Имя слоя в ГИС |
| `AttributeName` | `Brand<string, "AttributeName">` | `asAttributeName(value)` | Имя атрибута объекта |
| `DataSourceName` | `Brand<string, "DataSourceName">` | `asDataSourceName(value)` | Имя `ConfigDataSource` |
| `ResourceId` | `Brand<string, "ResourceId">` | `asResourceId(value)` | Id Python-ресурса |

### Иерархия `ContainerId`

`ChartId`, `ModalId`, `TabId` — подтипы `ContainerId`. Это отражает реальность: все они хранятся в одном runtime-keyspace (`ConfigContainer.id`), но логически принадлежат разным сущностям. Подтипы позволяют функции, ожидающей просто `ContainerId`, принимать любой из них; функция, ожидающая конкретно `ChartId`, отвергает `ModalId`.

`FilterName`, `LayerName`, `AttributeName`, `DataSourceName`, `ResourceId` — независимые keyspaces (живут в своих неймспейсах, не в `id`).

### Пример

```ts
import { asChartId, asModalId, ChartId } from "@evergis/react";

const chartId = asChartId("revenue_chart");
const modalId = asModalId("details_modal");

function findChart(id: ChartId) { /* ... */ }

findChart(chartId);          // ✓
findChart("revenue_chart");  // ❌ TS error — string не подставится в ChartId
findChart(modalId);          // ❌ TS error — ModalId ≠ ChartId
```

### Slot-id — НЕ branded

Slot-id обязателен у каждого элемента — без него контейнер не разместит элемент в нужное место (см. [[concepts#ID контейнеров и элементов|семантику id]]).

Литеральные slot-id (`"alias"`, `"chart"`, `"legend"`, `"title"`, `"value"`, `"units"`, ...) — это не entity-id, а ключи фиксированного набора. Они сужаются через literal `id` в parent-specific child-типах (см. `ChartAliasChild`, `ChartChartChild`, `ChartLegendChild` в `componentTypes.ts`).

Три slot-id универсальны и в parent-specific child-типах не перечисляются — их наборы задаются константами: `TITLE_SLOT_IDS` (`title`, `titleIcon`), `BG_IMAGE_SLOT_ID` (`bgImage`) и их объединение `NON_TRACK_SLOT_IDS` — узлы, которые контейнер читает по `id` сам и не отдаёт ни в рендер тела, ни в треки сетки (см. [[containers#Универсальные слоты|Контейнеры]]).

---

## Дискриминированный union `DashboardChild`

`DashboardChild` — union из 51 ветви: 16 `<Name>ElementConfig` и 35 `<Name>ContainerConfig`. Каждая ветвь сужена литералом:

- **элементы** — поле `type` (`"button"`, `"camera"`, `"chart"`, ...);
- **контейнеры** — поле `templateName` (`"AddFeature"`, `"Attachment"`, `"Chart"`, ...).

Оба поля помечены опциональными (`?`) — это **мягкая совместимость** с legacy-конфигами, где дискриминатор может отсутствовать. При наличии литерала TS сужает union до соответствующей ветви и проверяет поле `options` на соответствие `<Name>Options`.

### Ветви (элементы)

| Config | `type` |
|---|---|
| `ElementButtonConfig` | `"button"` |
| `ElementCameraConfig` | `"camera"` |
| `ElementChartConfig` | `"chart"` |
| `ElementChipsConfig` | `"tags"` (исторически — не `"chips"`) |
| `ElementControlConfig` | `"control"` |
| `ElementIconConfig` | `"icon"` |
| `ElementImageConfig` | `"image"` |
| `ElementLegendConfig` | `"legend"` |
| `ElementLinkConfig` | `"link"` |
| `ElementMarkdownConfig` | `"markdown"` |
| `ElementModalConfig` | `"modal"` |
| `ElementSlideshowConfig` | `"slideshow"` |
| `ElementSvgConfig` | `"svg"` |
| `ElementTableConfig` | `"table"` |
| `ElementTooltipConfig` | `"tooltip"` |
| `ElementUploaderConfig` | `"uploader"` |

### Ветви (контейнеры)

| Config | `templateName` |
|---|---|
| `AddFeatureContainerConfig` | `ContainerTemplate.AddFeature` |
| `AttachmentContainerConfig` | `ContainerTemplate.Attachment` |
| `CameraContainerConfig` | `ContainerTemplate.Camera` |
| `ChartContainerConfig` | `ContainerTemplate.Chart` |
| `ContainersGroupContainerConfig` | `ContainerTemplate.ContainersGroup` |
| `GridRowContainerConfig` | `ContainerTemplate.GridRow` |
| `DataSourceContainerConfig` | `ContainerTemplate.DataSource` |
| `DataSourceProgressContainerConfig` | `ContainerTemplate.DataSourceProgress` |
| `DefaultAttributesContainerConfig` | `ContainerTemplate.DefaultAttributes` |
| `DividerContainerElementConfig` | `ContainerTemplate.Divider` |
| `EditContainerConfig` | `ContainerTemplate.Edit` |
| `EditGroupContainerConfig` | `ContainerTemplate.EditGroup` |
| `EditBooleanContainerConfig` | `ContainerTemplate.EditBoolean` |
| `EditStringContainerConfig` | `ContainerTemplate.EditString` |
| `EditNumberContainerConfig` | `ContainerTemplate.EditNumber` |
| `EditDropdownContainerConfig` | `ContainerTemplate.EditDropdown` |
| `EditChipsContainerConfig` | `ContainerTemplate.EditChips` |
| `EditCheckboxContainerConfig` | `ContainerTemplate.EditCheckbox` |
| `EditDateContainerConfig` | `ContainerTemplate.EditDate` |
| `EditAttachmentContainerConfig` | `ContainerTemplate.EditAttachment` |
| `ExportPdfContainerConfig` | `ContainerTemplate.ExportPdf` |
| `FiltersContainerConfig` | `ContainerTemplate.Filters` |
| `IconContainerConfig` | `ContainerTemplate.Icon` |
| `ImageContainerConfig` | `ContainerTemplate.Image` |
| `LayersContainerConfig` | `ContainerTemplate.Layers` |
| `OneColumnContainerConfig` | `ContainerTemplate.OneColumn` |
| `ProgressContainerConfig` | `ContainerTemplate.Progress` |
| `RoundedBackgroundContainerConfig` | `ContainerTemplate.RoundedBackground` |
| `SlideshowContainerConfig` | `ContainerTemplate.Slideshow` |
| `StructuredDataContainerConfig` | `ContainerTemplate.StructuredData` |
| `TabsContainerConfig` | `ContainerTemplate.Tabs` |
| `TaskContainerConfig` | `ContainerTemplate.Task` |
| `TitleContainerConfig` | `ContainerTemplate.Title` |
| `TwoColumnContainerConfig` | `ContainerTemplate.TwoColumn` |
| `UploadContainerConfig` | `ContainerTemplate.Upload` |
| `VoteContainerConfig` | `ContainerTemplate.Vote` |

### Union шапок: `DashboardHeaderConfig`

```ts
type DashboardHeaderConfig =
  | DashboardDefaultHeaderConfig
  | FeatureCardBackgroundHeaderConfig
  | FeatureCardDefaultHeaderConfig
  | FeatureCardSlideshowHeaderConfig;
```

`HeaderTemplate.Default` сразу матчится двумя ветвями (`DashboardDefaultHeaderConfig` и `FeatureCardDefaultHeaderConfig`) — конкретная выбирается на уровне виджета (`WidgetType.Dashboard` vs `WidgetType.FeatureCard`).

### Строгие authoring-типы

Базовый `ConfigContainerChild.id` — **опционален** (`id?`) ради legacy-совместимости с рантайм-парсером, поэтому TS **не ловит** пропуск `id`. Для авторинга новых конфигов и примеров в документации есть два строгих типа, делающих пропуск `id` ошибкой компиляции:

| Тип | Что требует | Когда использовать |
|---|---|---|
| `StrictConfigContainerChild` | `id` обязателен **рекурсивно** — у узла и всех потомков в `children` | Авторинг дерева конфига целиком; `ts-check` ловит пропуск `id` на любой глубине |
| `StrictDashboardChild` | `DashboardChild & { id: string }` — сохраняет сужение дискриминированного union (проверка `options` по `type`/`templateName`) + требует `id` на верхнем узле | Авторинг одного контейнера/элемента, когда важна проверка опций |

```ts
const child: StrictConfigContainerChild = {
  id: "chart_floors",          // ← без id — ошибка компиляции
  templateName: ContainerTemplate.Chart,
  children: [{ id: "chart", type: "chart" }],
};
```

Оба экспортируются из `@evergis/react`. Для JSON-конфигов (где типы не работают) пропуск `id` закрывает рантайм-валидатор `validateDashboardConfig` и скилл `dashboard-container-gen` — см. [[authoring|Правила генерации]].

---

## Per-component типы

Для каждого компонента из реестра определены `<Name>Options`, `<Name>Config`, `<Name>Props`. Сводная таблица — какие поля `ConfigOptions` (см. [[options|Опции]]) использует каждый компонент.

### Общие миксины размеров

Два размерных набора не перечисляются в каждом `Pick`, а подмешиваются к `<Name>Options` целиком:

| Миксин | Поля | Кто подмешивает | Зачем |
|---|---|---|---|
| `ContainerBoxOptions` | `width`, `height`, `overflow` (все — `Pick<ConfigOptions, ...>`) | контейнеры, проходящие через `getWrapperSizeStyle`: `Attachment`, `Camera`, `Chart`, `ContainersGroup`, `DataSource`, `DataSourceProgress`, `Filters`, `GridRow`, `Image`, `Layers`, `OneColumn`, `Slideshow`, `StructuredData`, `Task`, `TwoColumn`, `Upload`, `Vote` | размерная модель корневой обёртки — единая для всех контейнеров; следующее размерное свойство добавляется в одном месте |
| `NumericSizeOptions` | `width?: number`, `height?: number` | `ElementChart`, `ElementSvg`, `ElementControl` (только `width`) | размеры, которые обязаны остаться **числом в пикселях**: значение уходит в вычисления геометрии графика или в HTML-атрибут, где `"100%"` не работает |

`ContainerBoxOptions` даёт `CssSize` (число = px, строка = любое CSS-значение); `"100%"` включает fill-режим обёртки (см. [[utils|`getWrapperSizeStyle`]]). `NumericSizeOptions`, наоборот, сужает те же имена до `number` — поэтому у `ElementChart` в таблице ниже `width`/`height` числовые, а у `ChartContainer` — `CssSize`.

### Элементы

> Колонка `<Name>Options` — это **только** поля внутри JSON-ключа `options`. Корневые поля элемента (`id`, `type`, `value`, `attributeName`, `style`, `relatedDataSource` и т.п.) лежат на верхнем уровне `ConfigContainerChild` — подробности по каждому элементу см. в [[elements]].

| Component | `type` | `<Name>Options` (Pick полей) |
|---|---|---|
| `ElementButton` | `"button"` | — (Record<string, never>) |
| `ElementCamera` | `"camera"` | `expandable`, `expanded` |
| `ElementChart` | `"chart"` | `column`, `markers`, `showLabels`, `showMarkers`, `showTotal`, `totalWord`, `totalAttribute`, `expandable`, `expanded`, `chartType`, `relatedDataSources`, `defaultColor`, `dotSnapping`, `height`, `radius`, `padding`, `fontColor`, `angle`, `barWidth`, `cornerRadius`, `shownItems`, `otherItems`, `axis`, `width` |
| `ElementChips` | `"tags"` | `separator`, `bgColor`, `fontColor`, `fontSize`, `colorAttribute`, `variants` |
| `ElementControl` | `"control"` | `relatedDataSource`, `label`, `width`, `control`, `placeholder` |
| `ElementIcon` | `"icon"` | `fontSize`, `fontColor` |
| `ElementImage` | `"image"` | `width`, `height`, `fit`, `resourceId`, `url` |
| `ElementLegend` | `"legend"` | `twoColumns`, `chartId`, `relatedDataSources`, `fontSize`, `chartType`, `column` |
| `ElementLink` | `"link"` | `simple`, `title` |
| `ElementMarkdown` | `"markdown"` | `expandLength`, `noMargin`, `typography` |
| `ElementModal` | `"modal"` | `modalId`, `icon` |
| `ElementSlideshow` | `"slideshow"` | `expandable`, `expanded`, `relatedDataSource`, `controls` |
| `ElementSvg` | `"svg"` | `width`, `height`, `fontColor` |
| `ElementTable` | `"table"` | `sort`, `editOnly`, `width`, `height` |
| `ElementTooltip` | `"tooltip"` | `icon` |
| `ElementUploader` | `"uploader"` | `fileExtensions`, `multiSelect`, `parentResourceId`, `icon`, `title`, `filterName` |

### Контейнеры

| Component | `templateName` | `<Name>Options` (Pick полей) |
|---|---|---|
| `AddFeatureContainer` | `AddFeature` | — (опции у `AddFeatureButtonChild`: `icon`, `title`, `layerName`, `geometryType`) |
| `AttachmentContainer` | `Attachment` | `expandable`, `expanded`, `viewMode`, `shownItems`, `otherItems`, `relatedDataSource`, `controls` + `ContainerBoxOptions` |
| `CameraContainer` | `Camera` | `expandable`, `expanded` + `ContainerBoxOptions` |
| `ChartContainer` | `Chart` | `twoColumns`, `legendInline`, `hideEmpty`, `fill` + `ContainerBoxOptions` (+ дети: `ChartAliasChild`, `ChartChartChild`, `ChartLegendChild`, `ChartTitleChild`, `ChartTitleIconChild`) |
| `ContainersGroupContainer` | `ContainersGroup` | `column`, `expandable`, `expanded`, `alignItems`, `grid`, `editMode`, `fixedHeight`, `autoHeight`, `gap` + `ContainerBoxOptions` |
| `GridRowContainer` | `GridRow` | `gap`, `alignItems`, `autoHeight` + `ContainerBoxOptions` |
| `DataSourceContainer` | `DataSource` | `column`, `relatedDataSource`, `innerTemplateName`, `expandable`, `expanded`, `columns`, `gap`, `innerGap`, `align`, `shownItems`, `otherItems` + `ContainerBoxOptions` |
| `DataSourceInnerContainer` | — | `relatedDataSource`, `filterName`, `column` |
| `DataSourceProgressContainer` | `DataSourceProgress` | `maxValue`, `showTotal`, `relatedDataSource`, `innerTemplateName`, `expandable`, `expanded`, `shownItems`, `otherItems` + `ContainerBoxOptions` |
| `DefaultAttributesContainer` | `DefaultAttributes` | — |
| `DividerContainer` | `Divider` | `bgColor` (из глобального `config.options`) |
| `EditContainer` | `Edit` | — |
| `EditGroupContainer` | `EditGroup` | `controls`, `useProjectHiddenAttributes`, `expandable`, `expanded` |
| `EditBooleanContainer` | `EditBoolean` | `controls` |
| `EditStringContainer` | `EditString` | `controls` |
| `EditNumberContainer` | `EditNumber` | `controls` |
| `EditDropdownContainer` | `EditDropdown` | `controls` |
| `EditChipsContainer` | `EditChips` | `controls` |
| `EditCheckboxContainer` | `EditCheckbox` | `controls` |
| `EditDateContainer` | `EditDate` | `withTime`, `controls` |
| `EditAttachmentContainer` | `EditAttachment` | `parentResourceId`, `fileExtensions`, `viewMode`, `shownItems`, `otherItems`, `relatedDataSource`, `controls` |
| `ExportPdfContainer` | `ExportPdf` | `icon`, `title` |
| `FiltersContainer` | `Filters` | `padding`, `bgColor`, `fontColor`, `fontSize`, `expandable`, `expanded` + `ContainerBoxOptions` (+ `FilterChildOptions` — см. [[containers#FiltersContainer\|полный список]]) |
| `IconContainer` | `Icon` | — |
| `ImageContainer` | `Image` | `ContainerBoxOptions` (собственных полей нет) |
| `LayersContainer` | `Layers` | `layerNames`, `expandable`, `expanded` + `ContainerBoxOptions` |
| `OneColumnContainer` | `OneColumn` | `attributes`, `useProjectHiddenAttributes`, `hideEmpty`, `innerTemplateStyle` + `ContainerBoxOptions` |
| `PagesContainer` | `Pages` | `column`, `width` (+ `PageChild.options.tabId` связывает страницу с табом) |
| `ProgressContainer` | `Progress` | `bgColor`, `innerTemplateStyle`, `maxValue`, `hideTitle`, `innerValue`, `colors`, `colorAttribute` |
| `RoundedBackgroundContainer` | `RoundedBackground` | `maxLength`, `maxLines`, `wordBreak`, `center`, `fontColor`, `bgColor`, `innerTemplateStyle`, `inlineUnits`, `big`, `bigIcon`, `hideEmpty`, `colorAttribute`, `align`, `columns`, `gap`, `innerGap` |
| `SlideshowContainer` | `Slideshow` | `expandable`, `expanded` + `ContainerBoxOptions` |
| `StructuredDataContainer` | `StructuredData` | `attributesDescription`, `relatedDataSource`, `filterName`, `editMode`, `saveToLayer`, `expandable`, `expanded` + `ContainerBoxOptions` (+ дети: `StructuredDataTableChild` (`id: "data"`, `type: "table"`, опции — `ElementTableOptions`), `StructuredDataAliasChild`, `StructuredDataTitleChild`, `StructuredDataTitleIconChild`) |
| `TabsContainer` | `Tabs` | `radius`, `column`, `bgColor`, `noBg`, `onlyIcon`, `shownItems`, `maxLength`, `wordBreak` (+ `TabChild`: `icon`) |
| `TaskContainer` | `Task` | `title`, `relatedResources`, `center`, `icon`, `statusColors`, `responseFilters`, `useNotifications` + `ContainerBoxOptions` |
| `TitleContainer` | `Title` | `simple`, `downloadById`, `align` |
| `TwoColumnContainer` | `TwoColumn` | `attributes`, `useProjectHiddenAttributes`, `hideEmpty`, `innerTemplateStyle` + `ContainerBoxOptions` |
| `UploadContainer` | `Upload` | `expandable`, `expanded` + `ContainerBoxOptions` |
| `VoteContainer` | `Vote` | `categoryDataSource`, `questionDataSource`, `variantDataSource`, `answerDataSource`, `expandable`, `expanded` + `ContainerBoxOptions` (+ на узле: `attributeName` — атрибут объекта с `question_id`) |

### Шапки

| Component | `templateName` | `<Name>Options` (Pick полей) |
|---|---|---|
| `DashboardDefaultHeader` | `Default` (в `WidgetType.Dashboard`) | `url` |
| `FeatureCardBackgroundHeader` | `Background` | `fontColor`, `bgColor`, `height`, `overlay`, `bigIcon`, `withPadding`, `bottomBlur`, `themeName`, `column` |
| `FeatureCardDefaultHeader` | `Default` (в `WidgetType.FeatureCard`) | `themeName`, `withPadding`, `column`, `height`, `overlay` |
| `FeatureCardSlideshowHeader` | `Slideshow` | `height`, `fontColor`, `withPadding`, `themeName`, `column`, `overlay` |

---

## Типизированные реестры

`ContainerComponentRegistry` и `ElementComponentRegistry` — `as const satisfies` контракт между ключами и React-компонентами.

```ts
export type ContainerTemplateToProps = {
  [ContainerTemplate.AddFeature]: AddFeatureContainerProps;
  [ContainerTemplate.Attachment]: AttachmentContainerProps;
  [ContainerTemplate.Chart]: ChartContainerProps;
  // ... все 36 ключей
};

export type ContainerComponentRegistry = {
  [K in keyof ContainerTemplateToProps]: import("react").FC<ContainerTemplateToProps[K]>;
} & { default: import("react").FC<ContainersGroupContainerProps> };
```

`containers/registry.ts` использует `as const satisfies ContainerComponentRegistry`, но собирает объект **лениво** — через функцию с кэшем, а не константой на инициализации модуля:

```ts
const createContainerComponents = () =>
  ({
    [ContainerTemplate.Chart]: ChartContainer,
    [ContainerTemplate.DataSource]: DataSourceContainer,
    // ...
    default: ContainersGroupContainer,
  }) as const satisfies ContainerComponentRegistry;

let cachedContainerComponents = null;

export const getContainerComponents = () => {
  if (!cachedContainerComponents) {
    cachedContainerComponents = createContainerComponents();
  }

  return cachedContainerComponents;
};
```

> [!warning] Почему лениво — циклический импорт
> Контейнеры импортируют баррели `../../components` и `../../utils`, а баррель utils через `getContainerComponent` тянет реестр обратно — получается цикл. Константа на этапе инициализации модуля читала бы `const` из ещё выполняющихся модулей контейнеров и падала с TDZ («Cannot access 'AddFeatureContainer' before initialization»): rollup спасает переупорядочиванием модулей, webpack — нет. К первому рендеру все модули уже инициализированы, поэтому чтение внутри функции безопасно; результат кэшируется и объект строится один раз. Потребители обращаются к реестру только через [[utils|`getContainerComponent`]] / `getContainerComponents()`, а не к экспортированной константе.

TypeScript на этапе компиляции проверяет, что каждый компонент в реестре действительно принимает props, соответствующие `<Name>Props` своего ключа. Если кто-то добавит новый `ContainerTemplate`, но забудет зарегистрировать компонент — компиляция упадёт.

Реестр элементов (`elements/registry.ts`) цикла не образует и остаётся обычной константой `elementComponents`.

Аналогично `ElementComponentRegistry`:

```ts
export type ElementTypeToProps = {
  control: ElementControlProps;
  chart: ElementChartProps;
  // ...
};

export type ElementComponentRegistry = {
  [K in keyof ElementTypeToProps]: import("react").FC<ElementTypeToProps[K]>;
};
```

### Как добавить новый контейнер

1. Описать `<Name>Options`, `<Name>Config`, `<Name>Props` в `componentTypes.ts`.
2. Добавить ветвь в `DashboardChild` union.
3. Добавить ключ в `ContainerTemplate` enum (`types.ts`).
4. Добавить запись в `ContainerTemplateToProps` (`componentTypes.ts`).
5. Зарегистрировать компонент в `containers/registry.ts` — TS-компилятор подскажет, если что-то пропущено.

### Особый случай: `Progress` и `RoundedBackground`

Эти контейнеры исторически принимают `InnerContainerProps` (из `DataSourceInnerContainer/types.ts`), а не свои `<Name>Props`. Регистрация работает за счёт контравариантности `FC<P>`: `FC<InnerContainerProps>` совместим с `FC<<Name>Props>`, пока `<Name>Props` ⊆ `InnerContainerProps`. В `registry.ts` применяется явный каст:

```ts
const ProgressContainerTyped = ProgressContainer as unknown as FC<ProgressContainerProps>;
const RoundedBackgroundContainerTyped =
  RoundedBackgroundContainer as unknown as FC<RoundedBackgroundContainerProps>;
```

### `ROOT_OWNING_TEMPLATES` — контейнеры со своим корнем

Рядом с реестром лежит `ROOT_OWNING_TEMPLATES: ReadonlySet<string>` — перечень шаблонов, чей компонент держит `id`, `data-templatename`, авторский `style` и `$sizeCss` на **единственном** собственном корне (через [[hooks|`useContainerRoot`]] / [[hooks|`useWrapperSize`]]). Такому контейнеру внешняя обёртка `ElementValueWrapper` не нужна: она добавляла бы в DOM второй узел с теми же атрибутами и стилями.

Читает набор [[utils#isRootOwningContainer|`isRootOwningContainer`]], а результат уходит флагом `hasOwnRoot` в [[utils#formatElementValue|`formatElementValue`]]. Неизвестный шаблон считается владельцем корня: он резолвится в реестровый `default` (= `ContainersGroupContainer`), а тот корнем владеет.

Сейчас в наборе: `ContainersGroup`, `GridRow`, `Attachment`, `Camera`, `Chart`, `DataSource`, `DataSourceProgress`, `Edit`, `Filters`, `Image`, `Layers`, `Slideshow`, `StructuredData`, `Task`, `Upload`, `Vote`.

Остальных там нет намеренно: `DefaultAttributes`, `EditGroup` и `OneColumn`/`TwoColumn` в режиме `attributesToRender` возвращают **несколько** корней, а `Title`, `Icon`, `Divider`, `Tabs`, `AddFeature`, `ExportPdf`, `Progress`, `RoundedBackground` ставят `id`/`style` руками — для них обёртка остаётся единственным одиночным узлом. Переводишь очередной контейнер на `useContainerRoot` — добавь его в набор.

---

## Пропсы корня контейнера

Корневая обёртка контейнера типизирована `ContainerRootProps` (`Dashboard/styled.ts`) — её собирают [[hooks|`useWrapperSize`]] и [[hooks|`useContainerRoot`]] (там тип называется `WrapperRootProps`).

| Поле | Тип | Назначение |
|---|---|---|
| `id` / `data-id` | `string?` | идентификатор узла и его дубль для внешних селекторов |
| `data-templatename` | `string?` | шаблон контейнера — точка зацепки для внешних стилей |
| `style` | `CSSProperties?` | авторский `style` из конфига, остаётся inline |
| `$sizeCss` | `CSSObject?` | размеры из `options` — уходят классом, а не inline, поэтому перебиваются без `!important` |
| `$noMargin` | `boolean?` | снять базовый отступ обёртки |
| `$hasBgImage` | `boolean?` | узел несёт слот `bgImage` — корень становится хостом фонового слоя |
| `$innerPadding` | `boolean?` | `options.innerPadding` — фиксированный внутренний отступ `1rem` на корне, чтобы содержимое не липло к краям фона |

Оба `$`-пропа приходят из отдельного мини-интерфейса `BgImageHostProps` (`components/ContainerBackground/styled.ts`), который `ContainerRootProps` расширяет:

```ts
interface BgImageHostProps {
  $hasBgImage?: boolean;
  $innerPadding?: boolean;
}

/** Пропсы самого слоя — в отличие от хоста, `options.outflow` читает только он. */
interface BgImageLayerProps {
  $outflow?: boolean;
}
```

Вместе с ними идёт `bgImageHostMixin`. Гейты в нём **раздельные**: `$hasBgImage` включает `position: relative` + `isolation: isolate`, `$innerPadding` — `padding: 1rem` (`CONTAINER_INNER_PADDING`) под селектором `&&`, чтобы перебить `padding` из `$sizeCss` и внутренних `defaults` контейнера, оставив авторский inline-`style` сильнее. Отступ намеренно не привязан к наличию фона — он нужен и без картинки. Модуль намеренно листовой: его импортирует `Dashboard/styled.ts`, и обратная зависимость замкнула бы цикл, уронив миксин в TDZ на инициализации.

Сам слой рендерит [[components|`ContainerBackground`]] с пропсами `ContainerBackgroundProps = Pick<ContainerProps, "elementConfig" | "renderElement">`, а styled-узел слоя (`ContainerBackgroundLayer`) типизирован `BgImageLayerProps`: `$outflow` растягивает его отрицательным `inset` на `1.5rem` (`BG_IMAGE_OUTFLOW`) по бокам и вверх. Признак наличия слота считает [[utils|`hasContainerBgImage`]]; корни, которые не берут пропсы из `useWrapperSize`, получают пропсы хоста хуком [[hooks|`useBgImageHost`]] (его вход — `BgImageHostConfig = Pick<ConfigContainerChild, "children" | "options">`).

---

## Per-feature локальные типы

Некоторые контейнеры и элементы имеют свои `types.ts` и `constants.ts` рядом с компонентом — для типов, специфичных только для них.

| Файл | Содержимое |
|---|---|
| `containers/ChartContainer/types.ts` | `ChartProps`, `ChartDataProps` (`BarChartData[]`, `PieChartData[]`, `FilterItem[]`, `axisSide` — сторона шкалы серии) |
| `containers/DataSourceInnerContainer/types.ts` | `InnerContainerProps` — `ContainerProps + feature?: FeatureDc` |
| `containers/FiltersContainer/types.ts` | `FilterOption` (`text`, `value`, `min`, `max`), `WidgetFilterProps` (`type`, `filter`, `config`) |
| `containers/FiltersContainer/constants.ts` | константы для рендера фильтров |
| `containers/AttachmentContainer/types.ts` | `FileType` enum (`XLSX`, `PDF`, `CSV`, ... — 16 типов), `IMAGE_FILE_TYPES`, `AttachmentViewMode`, `Attachment` |
| `containers/AttachmentContainer/constants.ts` | MIME-типы, лимиты, расширения |
| `containers/StructuredDataContainer/types.ts` | `DraftRowState` (`"pristine" \| "created" \| "updated" \| "deleted"`), `DraftRow` (`key`, `featureId?`, `properties`, `state`), `StructuredDataAttribute` — атрибут схемы после слияния `attributesDescription` с атрибутами источника (плюс `control` и `listOptions` — они есть только у схемы **представления**, см. [[containers#StructuredDataContainer\|Контейнеры]]), `StructuredDataContextValue` (`schema`, `rows`, `loading`, `canEdit`, `canDelete`, `onCellChange`, `onRowDelete`) |
| `containers/StructuredDataContainer/constants.ts` | `STRUCTURED_DATA_VIEW_SLOT` (`"data"`), типы атрибутов по группам, `DEFAULT_ATTRIBUTE_TYPE`, `DEFAULT_ATTRIBUTE_CONTROL` (`"dropdown"` — контрол колонки со списком, когда `control` не задан), `SAVE_ERROR_DURATION` |
| `containers/DataSourceContainer/constants.ts` | константы раскладки плиток источника |
| `containers/VoteContainer/types.ts` | `VoteScreen` (`"loading" \| "create" \| "voting" \| "voted" \| "unauthenticated"`), сущности БД `VoteCategory` / `VoteQuestion` / `VoteVariant` / `VoteVariantResult`, форма `VoteFormValues` + `VoteFormVariant`, `VoteDataSources` — `Required<Pick<VoteContainerOptions, ...четыре слоя>>` (только имена таблиц, без размеров и заголовка), пропсы экранов `VoteCreateFormProps` / `VoteResultsProps` / `VoteScreenProps` |
| `containers/VoteContainer/constants.ts` | имена атрибутов таблиц голосования и лимиты формы |
| `elements/ElementTable/types.ts` | `SortDirection`, `TableSort` (одна колонка за раз, `null` — исходный порядок), `CellMode` (`"idle" \| "focused" \| "editing"`), `TableCellProps`, `TableHeadRowProps`, `TableColumnResizerProps`, `AttachmentsCellState` — состояние колонки вложений целиком (список, вид поп-апа, галерея, загрузка, удаление) и `AttachmentsCellPopupProps` |
| `elements/ElementTable/constants.ts` | геометрия ячейки и шапки, `ATTACHMENTS_INLINE_LIMIT` / `ATTACHMENTS_SHOWN_ITEMS` / `ATTACHMENTS_VIEW_MODE`, `EMPTY_LIST_OPTION` (`{ text: "—", value: "" }` — пустой пункт списка, снимающий значение) |
| `elements/ElementCamera/types.ts` | `SmallPreviewProps` (`images`, `totalCount`, `currentIndex`), `CameraAttributeProps` |
| `elements/ElementSlideshow/types.ts` | `DashboardSlideshowProps` — Pick от `ElementSlideshowProps` |
| `components/Chart/FillContext.ts` | `FillContextValue` (`fill`, `fitHeight`) — контекст вписывания графика; `ChartContainer` кладёт в него `options.fill`, `Chart` читает через `useContext` (опции контейнера до элемента `chart` иначе не доходят) |
| `components/Chart/ChartPlotContext.ts` | `ChartPlotInsets` (`left`, `right`, `width`) и `ChartPlotContextValue` (`reportPlotInsets`) — обратный канал: `Chart` после отрисовки d3 сообщает `ChartContainer`, где лежит поле графика, и тот выравнивает по нему подпись оси X и легенду |
| `hooks/useChartAxisTitles.ts` | `ChartAxisTitle` (`title`, `color?`) — подпись оси одной стороны; `color` задан, только если на стороне одна серия |
| `components/Chart/types.ts` | `ChartContainerProps` обёртки графика (`width`, `height`, `column`, `loading`) |
| `components/ContainerBackground/types.ts` | `ContainerBackgroundProps` — `Pick<ContainerProps, "elementConfig" \| "renderElement">` |
| `components/ContainerBackground/styled.ts` | `BgImageHostProps` (`$hasBgImage`), слой `ContainerBackgroundLayer`, миксин `bgImageHostMixin` |
| `grid/types.ts` | Типы сетки: `GridAxis` (`"row"` \| `"column"`), `GridEditAction` (union операций `delete`/`merge`/`split`/`swap`/`addRow`/`addCell`), `GridMenuState`, `GridMenuPosition`, `GridEditSessionValue` — значение контекста сессии редактирования |

---

## Колбэк изменения конфига

`ContainerProps` содержит опциональный `onChange?: (config: ConfigContainerChild) => void` — контейнер сообщает наружу новую версию собственного узла. Сейчас источник один: [[containers#Редактирование раскладки editMode|сетка в режиме редактирования]].

Путь колбэка: проп `onContainerChange` у `DashboardProvider` / `FeatureCardProvider` → контекст → `useWidgetContext` → `PagesContainer` кладёт его в `getRenderElement({ onChange })` → движок передаёт каждому контейнеру пропом `onChange`. Поскольку `GetRenderElementProps extends Omit<ContainerProps, "renderElement">`, поле появилось в параметрах `getRenderElement` автоматически.

Идентификатор узла лежит внутри payload (`next.id`), поэтому хосту достаточно `replaceObject(config, { id: next.id }, next)` из `find-and`.

---

## Публичная поверхность сетки

Модуль `grid/` отдаётся наружу через `grid/index.ts` — только листовые модули: типы, константы, утилиты треков и дерева, фабрика id. Компоненты и сессия редактирования из пакета **не экспортируются**: вход в сетку один — [[containers#Режим сетки grid|`ContainersGroup` с `options.grid`]]. Баррель `utils` оттуда не тянут — иначе цикл `getRenderElement → registry → контейнеры → grid` уронил бы инициализацию в TDZ.

### Хостовые пропсы `ContainerProps`

Помимо `onChange` (см. раздел выше) сетка читает два пропа, предназначенных хосту с собственным реестром содержимого:

| Проп | Тип |
|---|---|
| `createRenderElement` | `(node: ConfigContainerChild) => RenderElementFunction` — фабрика рендера по черновику сессии; без неё сессия строит рендер штатным реестром |
| `onGridSelectionChange` | `(cellIds: string[]) => void` — зеркало выделения ячеек наружу |

Подробности — [[containers#Интеграционный API для хостов|Контейнеры]].

### Типы `grid/types.ts`

| Тип | Значение |
|---|---|
| `GridAxis` | `"row" \| "column"` — ось раскладки треков. `"row"`: треками управляет grid-узел, треки — строки (`grid-template-rows`, доля в `options.height`). `"column"`: треками управляет строка, треки — ячейки (`grid-template-columns`, доля в `options.width`) |
| `GridInsertSide` | `"before" \| "after"` — куда вставлять новый трек относительно опорного |
| `GridSplitDirection` | `"vertical" \| "horizontal"` — как встанут половинки после деления ячейки (расположение результата, а не линия разреза) |
| `GridEditAction` | Union операций: `{ type: "delete"; cellIds }`, `{ type: "merge"; cellIds }`, `{ type: "split"; cellId; direction }`, `{ type: "swap"; sourceId; targetId }`, `{ type: "addRow"; cellId; side }`, `{ type: "addCell"; cellId; side }` |
| `GridMenuPosition` | `{ x, y }` — координаты курсора для контекстного меню |
| `GridMenuState` | `{ canMerge, canSplit, canAdd }` — что доступно при текущем выделении |
| `GridEditSessionValue` | Значение контекста сессии: `draft`, `selectedIds`, `selectCell`, `clearSelection`, `beginCellDrag`, `consumeDragClick`, `commitResize`, `commitHeight`, `applyAction`, `openMenu`, `menuState` |
| `GridCellContext` | (`utils/gridTree`) окружение ячейки в дереве: её строка и позиция в ней |
| `GridIdFactory` | (`utils/createGridNodeId`) `{ createRowId, createCellId }` — генератор id создаваемых узлов |

### Константы `grid/constants.ts`

| Константа | Значение | Назначение |
|---|---|---|
| `MIN_TRACK_PX` | `40` | Верхняя граница минимального размера трека при ресайзе |
| `MIN_TRACK_RATIO` | `0.25` | Минимальная доля трека в тесной паре — чтобы граница не запиралась намертво |
| `DEFAULT_GRID_GAP` | `0` | Зазор между треками по умолчанию: сетка бесшовная |
| `DRAG_THRESHOLD_PX` | `4` | Сдвиг курсора, после которого нажатие считается перетаскиванием ячейки |
| `MAX_TRACKS` | `12` | Потолок на число треков у одного родителя — страховка от бесконечного `split` |
| `DEFAULT_TRACK_FR` | `1` | Доля нового трека, если среднее посчитать не из чего |
| `FR_PRECISION` | `1000` | Знаменатель округления долей — три знака после запятой |
| `GRID_FILL_DEFAULTS` / `GRID_AUTO_FILL_DEFAULTS` | `height: 100%` / `min-height: 100%` | Дефолты корня сетки и строки; второй — для режима `autoHeight` |
| `GRID_CELL_ATTR` | `data-grid-cell` | Маркер ячейки в DOM: по нему ищется цель перетаскивания под курсором. Маркер ручки (`RESIZE_HANDLE_ATTR`) и её заход на содержимое (`HANDLE_BLEED_PX`) живут у общего компонента ручки — см. [[components\|ResizeHandle]] |
| `GRID_DRAG_SOURCE_ATTR`, `GRID_DROP_TARGET_ATTR`, `GRID_DRAGGING_ATTR` | `data-grid-*` | Разметка идущего жеста — атрибутами, а не пропсами: цель меняется десятки раз за жест |
| `NO_CELL_DRAG_SELECTOR` | селектор | Что перетаскиванием ячейки не считается: ручка, `input`, `textarea`, `select`, `contenteditable` |
| `GRID_ROW_ID_PREFIX`, `GRID_CELL_ID_PREFIX` | `gridRow_`, `gridCell_` | Префиксы id создаваемых узлов. Не начинаются с `"page"` — по `id.startsWith("page")` `ContainersGroupContainer` опознаёт корневой блок страницы |

---

## Типографика markdown

Опция `typography` (`ConfigTypographyOptions`, читает только `ElementMarkdown`) переопределяет размер/интервал/начертание отдельных markdown-тегов:

```ts
type MarkdownTypographyTag = "h1" | "h2" | "h3" | "h4" | "h5" | "h6" | "p" | "li" | "code";

interface MarkdownTagTypography {
  fontSize?: FontSizeToken | string;
  lineHeight?: string;
  fontWeight?: number | string;
  /** Отступ снизу — расстояние до следующего блока. */
  marginBottom?: string;
  /** Отступ сверху — расстояние до предыдущего блока. */
  marginTop?: string;
}

type MarkdownTypography = Partial<Record<MarkdownTypographyTag, MarkdownTagTypography>>;
```

Отсутствующий тег или отдельное свойство берут дефолт `MarkdownWrapper` (`elements/ElementMarkdown/styled.ts`) — задавать нужно только переопределяемое.

---

## Типы источника данных

| Тип | Содержимое | Назначение |
|---|---|---|
| `ConfigDataSource` | `name`, `alias`, `attributes?`, `condition`, `ds`, `layerName`, `limit`, `offset`, `query`, `parameters`, `resourceId`, `fileName`, `methodName`, `url`, `type`, `autoSyncLayer`, `autoSyncLayers?`, `notifications?: ConfigNotifications`, `debounce?` | Описание запроса в конфиге страницы (см. [[concepts#Источники данных\|Основные понятия]]) |
| `ConfigDataSourceAttribute` | `attributeName`, `alias?`, `type?`, `stringFormat?: AttributeFormatConfigurationDc` | Элемент `ConfigDataSource.attributes` — настройки атрибута источника, накладываемые поверх атрибутов слоя/ответа EQL. `stringFormat` мержится по полям, поэтому задаётся только переопределяемое |
| `ConfigAttributeDescription` | `attributeName`, `type?`, `subType?`, `alias?`, `description?`, `isEditable?`, `stringFormat?`, `width?`, `resizable?`, `multiline?`, `colorPicker?`, `style?`, `parentResourceId?`, `fileExtensions?`, `control?: ConfigAttributeControl`, `variants?: ChipOption[]`, `relatedDataSource?`, `attributeValue?`, `attributeAlias?` | Элемент `options.attributesDescription` — описание атрибута структуры [[containers#StructuredDataContainer\|StructuredDataContainer]]. Повторяет форму `AttributeConfigurationDc`, но со `stringFormat` пакета и полями раскладки, вида, загрузки вложений и списка значений, которых в серверном контракте нет |
| `ConfigAttributeControl` | `"dropdown" \| "chips"` | Контрол выбора значения колонки из списка (`variants` либо справочник `relatedDataSource`): выпадающий список (по умолчанию) или ряд кнопок |
| `ConfigNotifications` | `"all" \| "my" \| string[]` | Фильтр серверных уведомлений по автору изменения (`senderName`): `"all"` — любые (по умолчанию), `"my"` — только текущего пользователя, массив — только перечисленных (текущий пользователь неявно не добавляется, `"my"` внутри массива — обычное имя). Уведомление без автора (системная задача) применяется всегда. Поле с этим типом есть у `ConfigDataSource`, `ConfigLayer` и `ConfigMiscOptions` (`options` корня); приоритет — источник → слой → корень, см. [[concepts#Фильтр уведомлений по автору — notifications\|Основные понятия]] |
| `EqlDataSource` | `items: FeatureDc[]`, `attributes?` | Ответ EQL-запроса |
| `FetchedDataSource` / `WidgetDataSource` | `name`, `features`, `layerName?`, `attributes?` | Загруженный источник в состоянии виджета |

> Не путать `ConfigDataSource.attributes` (`ConfigDataSourceAttribute[]` — метаданные и формат атрибутов источника) с `options.attributes` (`string[]` — список имён атрибутов для отображения в `OneColumn`/`TwoColumn`).

---

## Типы осей графика

### ConfigAxis

Настройки одной оси и привязанной к ней серии. Лежат в `ConfigRelatedDataSource.axis` — по одному объекту на источник.

```ts
interface ConfigAxis {
  type?: "x" | "y";              // роль источника, по умолчанию "y"
  side?: "left" | "right";       // сторона шкалы, по умолчанию "left"
  color?: string;                // цвет линии, заливки и метки в легенде
  title?: string;                // подпись оси; цвет — color серии (одна на стороне) или нейтральный
  hide?: boolean;                // скрыть ось целиком
}
```

Заменяет прежние плоские поля `ConfigRelatedDataSource`: `chartAxis` → `axis.type`, `axisColor` → `axis.color`, `hideAxis` → `axis.hide`.

> [!info] Старые конфиги работают через слой совместимости
> Плоские поля остались в типе как `@deprecated` и читаются утилитой [[utils#resolveAxis|`resolveAxis`]]: при отсутствии `axis` она собирает объект из них. Заданный `axis` приоритетнее — узел, который уже правили, не откатывается к старым значениям.
>
> Слой временный. Каждый источник со старой формой один раз сообщает о себе в консоль в dev-сборке — по этим сообщениям видно, когда легаси закончилось и слой можно снять. Редактор дашборда нормализует конфиг на входе, поэтому открытый и сохранённый контейнер переезжает на `axis` сам.

Отличие в умолчании: раньше источник **без** `chartAxis` из графика выпадал, теперь отсутствие `axis.type` означает `"y"`. Обратной дороги к «источник без оси игнорируется» нет.

`side` — не только про то, с какой стороны нарисованы значения: серия строится **по шкале своей стороны**, домены левой и правой считаются независимо. `alias` (имя серии в легенде) к оси не относится и остаётся снаружи `axis`.

### ConfigChartAxisOptions

Общие настройки осей графика, `ElementChart.options.axis`:

```ts
interface ConfigChartAxisOptions {
  titlePosition?: "side" | "top";                          // общий режим подписей осей Y
  x?: { title?: string; align?: "left" | "center" | "right" };  // подпись оси X
}
```

Типы-алиасы: `ConfigAxisType`, `ConfigAxisSide`, `ConfigAxisAlign`, `ConfigAxisTitlePosition`.

Подробности рендеринга — [[elements#ElementChart|ElementChart]] (подписи осей Y) и [[containers#ChartContainer|ChartContainer]] (подпись оси X и легенда).

---

## Типы фильтров

| Тип | Содержимое | Назначение |
|---|---|---|
| `ConfigFilterValueType` | `"single" \| "range" \| "array" \| "tree" \| "features"` | Вид значения фильтра, объявленный в конфиге. Дискриминатор для type guard-ов |
| `TreeFilterValue` | `Record<"l{N}", Array<string \| number>>` | Значение иерархического фильтра: «уровень → массив id». Плейсхолдеры `%name.lN` |
| `FeaturesFilterValue` | `FeatureCollection<null, Record<string, FeatureAttributeValue>>` | Значение фильтра `"features"` — строки [[containers#StructuredDataContainer\|StructuredDataContainer]]. `geometry` всегда `null` |
| `SelectedFilter` | `value`, `min?`, `max?` | Выбранное значение фильтра в состоянии виджета |
| `ScalarFilterValue` | `SelectedFilter["value"]` без `TreeFilterValue` и `FeaturesFilterValue` | Значение скалярных/массивных фильтров |

**Куда подставляется значение.** `single`/`range`/`array` — и в `condition` источника, и в `parameters`. `tree` — в `condition` через `applyTreeFilterToCondition` (`%name.lN`) и в `parameters`. `features` — **только** в `parameters` (питон-таска, url-источник); в `condition` не попадает никогда: там значение прошло бы через `formatConditionValue` и выродилось в `[object Object]`.

**Как определяется вид значения.** По полю `valueType` конфига фильтра, а не по форме значения: `tree` и `features` оба объекты, и структурная догадка их не различает. Отсюда сигнатура `isTreeFilterValue(value, configFilter?)` — второй аргумент передают везде, где конфиг фильтра под рукой. Решают ровно два значения `valueType`: `"tree"` — это tree, `"features"` — точно не tree. Любое другое (`"array"`, не задано вовсе) уходит в структурный фолбэк: в существующих конфигах tree-фильтры объявлены как раз так, и трактовать их как не-tree значило бы их сломать. Фолбэк намеренно узкий: ключи вида `l{N}`, значения — массивы. `isFeaturesFilterValue(value)` конфига не требует: форма `FeatureCollection` самодостаточна.

> Объявляя фильтр-приёмник для `StructuredData`, обязательно ставь `valueType: "features"` — без него значение уйдёт в структурный фолбэк, а `condition` источника не получит гейта по конфигу. Валидатор конфига (`validateDashboardConfig`) проверяет это отдельно.

---

## Типы серверных хуков сохранения

Типы для механизма `beforeSave`/`afterSave` (см. [[concepts#Серверные хуки сохранения (beforeSave / afterSave)|Основные понятия]] и [[hooks|хук]] `useFeatureSaveHooks`). Определены в `types.ts`.

| Тип | Содержимое | Назначение |
|---|---|---|
| `ConfigRelatedResource` | `resourceId`, `parameters`, `script?`, `fileName?`, `methodName?` | Описание серверного python-ресурса. Используется и в `TaskContainer.options.relatedResources`, и в save-хуках |
| `EditConfigurationOptions` | `beforeSave?: ConfigRelatedResource`, `afterSave?: ConfigRelatedResource` | Контейнер `editConfiguration.options` слоя |
| `SaveHookInput` | `featureId: number \| string \| null`, `changedProperties: Record<string, unknown>`, `changedGeometry?: Geometry` | Вход `runBeforeSave`/`runAfterSave`; `featureId === null` при создании нового объекта; `changedGeometry` — новая/отредактированная GeoJSON-геометрия (WGS84), опущена если геометрия не менялась |

```ts
interface EditConfigurationOptions {
  beforeSave?: ConfigRelatedResource; // синхронная валидация перед save
  afterSave?: ConfigRelatedResource;  // fire-and-forget после save
}

interface SaveHookInput {
  featureId: number | string | null;          // null при создании нового объекта
  changedProperties: Record<string, unknown>; // изменённые атрибуты
  changedGeometry?: Geometry;                  // GeoJSON-геометрия (WGS84); опущена если не менялась
}
```

Хук считает скрипт активным (`isHookActive`), если у `ConfigRelatedResource` задан `resourceId` или `fileName`. Имя ресурса типизируется branded-типом [[#Branded types|ResourceId]].

---

## CSS-токены

В `types.ts` определены литеральные шаблонные типы для CSS-значений:

```ts
type CssLength = `${number}rem` | `${number}px` | `${number}em` | `${number}%`;
type DesignToken = `var(--${string})`;
type FontSizeToken = CssLength | DesignToken | "larger" | "smaller";

type CssColor =
  | `#${string}`
  | `rgb(${string})` | `rgba(${string})`
  | `hsl(${string})` | `hsla(${string})`
  | `var(--${string})`
  | "transparent" | "currentColor" | "inherit";

type FileExtensions = `.${string}` | `.${string},${string}`;
```

Применяются как opt-in для нового кода. Существующий `ConfigOptions.fontSize` пока остаётся `string` ради совместимости с legacy-конфигами.

---

## Связанные разделы

[[options|Опции]] | [[architecture|Архитектура]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[headers|Шапки]]
