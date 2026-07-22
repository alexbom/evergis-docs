# Компоненты

## Обзор

Переиспользуемые React-компоненты из `components/`. Используются внутри контейнеров, элементов и шапок. Props этих компонентов не входят в типизированные реестры `ContainerComponentRegistry`/`ElementComponentRegistry` (см. [[types|Типы]]) — это внутренние компоненты, не зарегистрированные в registry.

---

## AddFeatureButton

**Назначение:** Кнопка добавления нового объекта на карту-слой.

**Props:**
| Prop | Тип | Default |
|---|---|---|
| `title` | `string?` | — |
| `icon` | `IconTypesKeys?` | `"feature_add"` |
| `layerName` | `string?` | — |
| `geometryType` | `OgcGeometryType \| EditGeometryType?` | — |

> Логика `useFeatureCreator` закомментирована, текущий обработчик — no-op.

```tsx
<AddFeatureButton icon="feature_add" title="Добавить объект" layerName="myLayer" />
```

---

## Chart

**Назначение:** Основной компонент для отображения чартов: bar, line, pie, stack. Большой компонент с поддержкой фильтрации по клику, тултипами, маркерами, легендой.

**Props (`ChartProps`):**
| Prop | Тип |
|---|---|
| `config` | `ConfigContainer` |
| `element` | `ConfigContainerChild` — конфиг чарт-элемента |
| `elementConfig` | `ConfigContainerChild` |
| `type` | `WidgetType` |
| `renderElement` | `RenderElementFunction` |

**Зависимости:** `useChartData`, `useChartChange`, `useWidgetFilters`, `useWidgetContext`, `useGlobalContext`, `useResizeBox` (только в fill-режиме)

**Режим `fill`:** читается из контекста `FillContext`, который выставляет `ChartContainer` (опции контейнера до элемента `chart` не доходят). При `fill` тело графика оборачивается в измеряемый `ChartFillMeasure`, а размеры берутся из `useResizeBox`, а не из `options.width`/`options.height`. Подробно — [[containers#Как работает fill|ChartContainer]].

**Типы чартов** (через `options.chartType`):
- `bar` (default) — StyledBarChart из `@evergis/charts`
- `line` — LineChart из `@evergis/charts`
- `pie` — PieChart из `@evergis/charts`
- `stack` — StackBar (кастомный)

### ChartWrapper

Обёртка с loading skeleton и позиционированием (width, height, column).

---

## ChartLegend

**Назначение:** Список легенды для чарта с цветными маркерами и значениями.

**Props:**
| Prop | Тип |
|---|---|
| `data` | `PieChartDisplayedData` |
| `loading` | `boolean` |
| `chartElement` | `ConfigContainerChild` |
| `twoColumns` | `boolean?` |
| `fontSize` | `string?` |
| `type` | `WidgetType` |

---

## ContainerChildren

**Назначение:** Рендерит список дочерних элементов контейнера. Исключает служебные (`title`, `icon`, `titleIcon`), проверяет `hideIfEmptyDataSource`.

**Props (`ContainerChildrenProps`):**
| Prop | Описание |
|---|---|
| `type` | `WidgetType` |
| `items` | `ConfigContainerChild[]` |
| `isColumn` | `boolean?` |
| `isMain` | `boolean?` |
| `renderElement` | `RenderElementFunction` |

При `isMain` оборачивает каждый item в `ContainerWrapper > DashboardWrapper`.

---

## Dashboard (главный)

**Назначение:** Корневой компонент виджета. Управляет отображением лоадера и рендерит `PagesContainer`.

**Props:** `{ type?: WidgetType, noBorders?: boolean }`

**Логика:**
- `dataSourceLoading || isDiffPage` → `<DashboardLoading />`
- Иначе → `<PagesContainer type={type} noBorders={noBorders} />`

```tsx
<Dashboard type={WidgetType.Dashboard} />
```

---

## DashboardCheckbox

**Назначение:** Стилизованный чекбокс для Dashboard. В режиме просмотра (без `onChange`) выводит локализованную метку «Да»/«Нет», в режиме редактирования — `CardCheckbox`.

**Props:**
| Prop | Тип |
|---|---|
| `title` | `ReactElement \| string` |
| `checked` | `boolean` |
| `onChange` | `VoidFunction?` |

---

## DashboardHeader

**Назначение:** Точка входа для шапки дашборда. Resolves тип шапки через `getDashboardHeader(currentPage.header.templateName)`.

**Props:** нет (данные из контекста)

```tsx
<DashboardHeader />
```

---

## DataSourceError

**Назначение:** Заглушка при ошибке загрузки датасорса (features === null).

**Props:** `{ name: string }`

---

## ExpandableTitle

**Назначение:** Заголовок с возможностью раскрытия/сворачивания контейнера. Рендерит `TitleContainer`.

**Props:** `ExpandableTitleProps` (elementConfig, type, renderElement)

Ищет дочерний элемент `id === "title"`. Если не найден — не рендерится. Управляет `expandContainer` через `TitleContainer`.

---

## FeatureCardButtons

**Назначение:** Кнопки действий карточки объекта (редактировать, закрыть, сохранить и т.п.).

---

## FeatureCardHeader

**Назначение:** Точка входа для шапки FeatureCard. Resolves тип через `getFeatureCardHeader(currentPage.header.templateName)`.

```tsx
<FeatureCardHeader />
```

---

## FeatureCardTitle

**Назначение:** Заголовок и описание в шапке FeatureCard. Заголовок разрешается из пропса `title`, иначе из значения `titleAttribute` (источник или layer definition), иначе `feature.id`.

**Props:** `{ title: string, description: string }`

---

## HiddenTitleItems

**Назначение:** Отображает выбранные фильтры в свёрнутом состоянии контейнера (под заголовком) в виде чипов с кнопкой сброса.

**Props (`ContainerProps & { filter? }`):** `{ elementConfig, config, type, renderElement, filter?: string }`

---

## Loading

### ContainerLoading

**Назначение:** Скелетон-лоадер для контейнеров данных.

### ChartLoading

**Назначение:** Скелетон-лоадер для чартов.

### DashboardLoading

**Назначение:** Полноэкранный лоадер при смене страницы.

---

## LogTerminal

**Назначение:** Терминал (xterm.js) для вывода лога выполнения Python-задачи. Используется в `TaskContainer`. Поддерживает инкрементальную дозапись строкового лога, вывод JSON-результата и Ctrl+C для копирования выделения.

**Props (`TaskLogTerminalProps`):**
| Prop | Тип |
|---|---|
| `log` | `string \| Record<string, any>?` |
| `className` | `string?` |
| `styles` | `CSSProperties?` |
| `terminalOptions` | `ITerminalOptions & ITerminalInitOnlyOptions?` |

---

## Pagination

**Назначение:** Кнопки «Назад» / «Вперёд» для навигации между страницами дашборда.

**Props:** `{ type?: WidgetType }`

Использует `useWidgetContext` (nextPage, prevPage) и `useWidgetConfig` (pages.length).

```tsx
<Pagination type={WidgetType.Dashboard} />
```

---

## StackBar

**Назначение:** Горизонтальный стек-бар чарт. Альтернатива `PieChart` для отображения долей.

**Props:**
| Prop | Тип |
|---|---|
| `data` | `ChartDataItem[]` |
| `filterName` | `string` |
| `type` | `WidgetType` |
| `alias` | `ConfigContainerChild?` |
| `options` | `ConfigOptions` |
| `renderTooltip` | функция |
| `renderElement` | `RenderElementFunction` |

---

## SvgImage

**Назначение:** Загружает и отображает SVG-файл с заменой цвета через `currentColor`.

**Props:**
| Prop | Тип |
|---|---|
| `url` | `string` |
| `width` | `number?` |
| `height` | `number?` |
| `fontColor` | `string?` |

```tsx
<SvgImage url="/sp/resources/file/icon.svg" width={24} fontColor="#333" />
```

---

## TextTrim

**Назначение:** Обрезает текст до `maxLength` символов с добавлением `...`. При `expandable` показывает кнопку «Подробнее»/«Свернуть», иначе полный текст в тултипе. Поддерживает перенос строк через `lineBreak` и режим переноса слов `wordBreak`.

**Props (`TextTrimProps`):**
| Prop | Тип |
|---|---|
| `maxLength` | `number?` |
| `expandable` | `boolean?` |
| `lineBreak` | `string?` |
| `wordBreak` | `ConfigTextDisplayOptions["wordBreak"]?` |
| `children` | `string \| number?` |

```tsx
<TextTrim maxLength={20}>{longTitle}</TextTrim>
```

---

## Связанные разделы

[[hooks|Хуки]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[headers|Шапки]] | [[types|Типы]]
