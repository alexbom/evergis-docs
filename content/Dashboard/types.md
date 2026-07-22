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

---

## Дискриминированный union `DashboardChild`

`DashboardChild` — union из 48 ветвей: 15 `<Name>ElementConfig` и 33 `<Name>ContainerConfig`. Каждая ветвь сужена литералом:

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
| `TabsContainerConfig` | `ContainerTemplate.Tabs` |
| `TaskContainerConfig` | `ContainerTemplate.Task` |
| `TitleContainerConfig` | `ContainerTemplate.Title` |
| `TwoColumnContainerConfig` | `ContainerTemplate.TwoColumn` |
| `UploadContainerConfig` | `ContainerTemplate.Upload` |

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
| `ContainerBoxOptions` | `width`, `height`, `overflow` (все — `Pick<ConfigOptions, ...>`) | контейнеры, проходящие через `getWrapperSizeStyle`: `Attachment`, `Camera`, `Chart`, `ContainersGroup`, `DataSource`, `DataSourceProgress`, `Filters`, `Image`, `Layers`, `OneColumn`, `Slideshow`, `Task`, `TwoColumn`, `Upload` | размерная модель корневой обёртки — единая для всех контейнеров; следующее размерное свойство добавляется в одном месте |
| `NumericSizeOptions` | `width?: number`, `height?: number` | `ElementChart`, `ElementSvg`, `ElementControl` (только `width`) | размеры, которые обязаны остаться **числом в пикселях**: значение уходит в вычисления геометрии графика или в HTML-атрибут, где `"100%"` не работает |

`ContainerBoxOptions` даёт `CssSize` (число = px, строка = любое CSS-значение); `"100%"` включает fill-режим обёртки (см. [[utils|`getWrapperSizeStyle`]]). `NumericSizeOptions`, наоборот, сужает те же имена до `number` — поэтому у `ElementChart` в таблице ниже `width`/`height` числовые, а у `ChartContainer` — `CssSize`.

### Элементы

> Колонка `<Name>Options` — это **только** поля внутри JSON-ключа `options`. Корневые поля элемента (`id`, `type`, `value`, `attributeName`, `style`, `relatedDataSource` и т.п.) лежат на верхнем уровне `ConfigContainerChild` — подробности по каждому элементу см. в [[elements]].

| Component | `type` | `<Name>Options` (Pick полей) |
|---|---|---|
| `ElementButton` | `"button"` | — (Record<string, never>) |
| `ElementCamera` | `"camera"` | `expandable`, `expanded` |
| `ElementChart` | `"chart"` | `column`, `markers`, `showLabels`, `showMarkers`, `showTotal`, `totalWord`, `totalAttribute`, `expandable`, `expanded`, `chartType`, `relatedDataSources`, `defaultColor`, `dotSnapping`, `height`, `radius`, `padding`, `fontColor`, `angle`, `barWidth`, `cornerRadius`, `width` |
| `ElementChips` | `"tags"` | `separator`, `bgColor`, `fontColor`, `fontSize`, `colorAttribute`, `variants` |
| `ElementControl` | `"control"` | `relatedDataSource`, `label`, `width`, `control`, `placeholder` |
| `ElementIcon` | `"icon"` | `fontSize`, `fontColor` |
| `ElementImage` | `"image"` | `width`, `height`, `fit` |
| `ElementLegend` | `"legend"` | `twoColumns`, `chartId`, `relatedDataSources`, `fontSize`, `chartType` |
| `ElementLink` | `"link"` | `simple`, `title` |
| `ElementMarkdown` | `"markdown"` | `expandLength`, `noMargin`, `typography` |
| `ElementModal` | `"modal"` | `modalId`, `icon` |
| `ElementSlideshow` | `"slideshow"` | `expandable`, `expanded`, `relatedDataSource`, `controls` |
| `ElementSvg` | `"svg"` | `width`, `height`, `fontColor` |
| `ElementTooltip` | `"tooltip"` | `icon` |
| `ElementUploader` | `"uploader"` | `fileExtensions`, `multiSelect`, `parentResourceId`, `icon`, `title`, `filterName` |

### Контейнеры

| Component | `templateName` | `<Name>Options` (Pick полей) |
|---|---|---|
| `AddFeatureContainer` | `AddFeature` | — (опции у `AddFeatureButtonChild`: `icon`, `title`, `layerName`, `geometryType`) |
| `AttachmentContainer` | `Attachment` | `expandable`, `expanded`, `viewMode`, `shownItems`, `otherItems`, `relatedDataSource`, `controls` + `ContainerBoxOptions` |
| `CameraContainer` | `Camera` | `expandable`, `expanded` + `ContainerBoxOptions` |
| `ChartContainer` | `Chart` | `twoColumns`, `hideEmpty`, `fill` + `ContainerBoxOptions` (+ дети: `ChartAliasChild`, `ChartChartChild`, `ChartLegendChild`, `ChartTitleChild`, `ChartTitleIconChild`) |
| `ContainersGroupContainer` | `ContainersGroup` | `column`, `expandable`, `expanded`, `alignItems` + `ContainerBoxOptions` |
| `DataSourceContainer` | `DataSource` | `column`, `relatedDataSource`, `innerTemplateName`, `expandable`, `expanded` + `ContainerBoxOptions` |
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
| `RoundedBackgroundContainer` | `RoundedBackground` | `maxLength`, `wordBreak`, `center`, `fontColor`, `innerTemplateStyle`, `inlineUnits`, `big`, `bigIcon`, `hideEmpty`, `colorAttribute` |
| `SlideshowContainer` | `Slideshow` | `expandable`, `expanded` + `ContainerBoxOptions` |
| `TabsContainer` | `Tabs` | `radius`, `column`, `bgColor`, `noBg`, `onlyIcon`, `shownItems`, `maxLength`, `wordBreak` (+ `TabChild`: `icon`) |
| `TaskContainer` | `Task` | `title`, `relatedResources`, `center`, `icon`, `statusColors`, `responseFilters`, `useNotifications` + `ContainerBoxOptions` |
| `TitleContainer` | `Title` | `simple`, `downloadById`, `align` |
| `TwoColumnContainer` | `TwoColumn` | `attributes`, `useProjectHiddenAttributes`, `hideEmpty`, `innerTemplateStyle` + `ContainerBoxOptions` |
| `UploadContainer` | `Upload` | `expandable`, `expanded` + `ContainerBoxOptions` |

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
  // ... все 34 ключа
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

---

## Per-feature локальные типы

Некоторые контейнеры и элементы имеют свои `types.ts` и `constants.ts` рядом с компонентом — для типов, специфичных только для них.

| Файл | Содержимое |
|---|---|
| `containers/ChartContainer/types.ts` | `ChartProps`, `ChartDataProps` (`BarChartData[]`, `PieChartData[]`, `FilterItem[]`) |
| `containers/DataSourceInnerContainer/types.ts` | `InnerContainerProps` — `ContainerProps + feature?: FeatureDc` |
| `containers/FiltersContainer/types.ts` | `FilterOption` (`text`, `value`, `min`, `max`), `WidgetFilterProps` (`type`, `filter`, `config`) |
| `containers/FiltersContainer/constants.ts` | константы для рендера фильтров |
| `containers/AttachmentContainer/types.ts` | `FileType` enum (`XLSX`, `PDF`, `CSV`, ... — 16 типов), `IMAGE_FILE_TYPES`, `AttachmentViewMode`, `Attachment` |
| `containers/AttachmentContainer/constants.ts` | MIME-типы, лимиты, расширения |
| `elements/ElementCamera/types.ts` | `SmallPreviewProps` (`images`, `totalCount`, `currentIndex`), `CameraAttributeProps` |
| `elements/ElementSlideshow/types.ts` | `DashboardSlideshowProps` — Pick от `ElementSlideshowProps` |
| `components/Chart/FillContext.ts` | `FillContextValue` (`fill`, `fitHeight`) — контекст вписывания графика; `ChartContainer` кладёт в него `options.fill`, `Chart` читает через `useContext` (опции контейнера до элемента `chart` иначе не доходят) |
| `components/Chart/types.ts` | `ChartContainerProps` обёртки графика (`width`, `height`, `column`, `loading`) |

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
| `ConfigDataSource` | `name`, `alias`, `attributes?`, `condition`, `ds`, `layerName`, `limit`, `offset`, `query`, `parameters`, `resourceId`, `fileName`, `methodName`, `url`, `type`, `autoSyncLayer` | Описание запроса в конфиге страницы (см. [[concepts#Источники данных\|Основные понятия]]) |
| `ConfigDataSourceAttribute` | `attributeName`, `alias?`, `type?`, `stringFormat?: AttributeFormatConfigurationDc` | Элемент `ConfigDataSource.attributes` — настройки атрибута источника, накладываемые поверх атрибутов слоя/ответа EQL. `stringFormat` мержится по полям, поэтому задаётся только переопределяемое |
| `EqlDataSource` | `items: FeatureDc[]`, `attributes?` | Ответ EQL-запроса |
| `FetchedDataSource` / `WidgetDataSource` | `name`, `features`, `layerName?`, `attributes?` | Загруженный источник в состоянии виджета |

> Не путать `ConfigDataSource.attributes` (`ConfigDataSourceAttribute[]` — метаданные и формат атрибутов источника) с `options.attributes` (`string[]` — список имён атрибутов для отображения в `OneColumn`/`TwoColumn`).

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
