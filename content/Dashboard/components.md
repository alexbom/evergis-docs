# Компоненты

## Обзор

Переиспользуемые React-компоненты из `components/`. Используются внутри контейнеров, элементов и шапок. Props этих компонентов не входят в типизированные реестры `ContainerComponentRegistry`/`ElementComponentRegistry` (см. [[types|Типы]]) — это внутренние компоненты, не зарегистрированные в registry.

---

## AddButton

**Назначение:** Общая на весь дашборд кнопка «добавить»: серая скруглённая, со значком и подписью. Экспортируется одним styled-компонентом `AddButtonRow` (обёртка над `IconButton` из `@evergis/uilib-gl`) — вложения ([[containers|`AttachmentContainer`]]) и таблица структурированных данных ([[containers|`StructuredDataContainer`]]) добавляют одинаковой кнопкой, разница только в значке.

**Props:** собственных нет — принимает пропсы `IconButton` (`icon`, `onClick`, `children` как подпись).

Размеры сняты с макета: высота 24, поля 10, просвет между значком и подписью 6, значок и подпись по 14. Значок приглушён цветом иконки, подпись остаётся `textPrimary`.

```tsx
<AddButtonRow icon="plus" onClick={onAdd}>{t("addRow")}</AddButtonRow>
```

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

## ContainerBackground

**Назначение:** Фоновое изображение контейнера — универсальный слот `bgImage`. Рендерит ребёнка со слотом `bgImage` внутри абсолютного слоя `ContainerBackgroundLayer`, который лежит **под** содержимым контейнера. Слота нет в конфиге — в DOM не появляется ничего: ни обёртки, ни пустого слоя.

**Props (`ContainerBackgroundProps` = `Pick<ContainerProps, "elementConfig" | "renderElement">`):**
| Prop | Тип | Описание |
|---|---|---|
| `elementConfig` | `ConfigContainerChild?` | конфиг контейнера — в нём ищется ребёнок с `id: "bgImage"` |
| `renderElement` | `RenderElementFunction?` | рендер слота; вызывается как `renderElement({ id: "bgImage", wrap: false })` |

**Требование к хосту.** Ставится **первым** ребёнком корня контейнера, и корень обязан быть хостом слоя: `bgImageHostMixin` + проп `$hasBgImage`. У контейнеров на [[hooks|`useContainerRoot`]] / `useWrapperSize` признак приходит готовым в пропсах корня; те, что ставят `id`/`style` руками, берут его хуком [[hooks|`useBgImageHost`]].

**Как устроен слой (`ContainerBackgroundLayer`, пропсы `BgImageLayerProps`):** `position: absolute; inset: 0; z-index: -1; overflow: hidden; border-radius: inherit; pointer-events: none`, вложенная `img` — `object-fit: cover`. Отрицательный `z-index` ложится под содержимое хоста, но поверх его фона, только внутри собственного stacking-контекста (`isolation: isolate` в миксине) — без изоляции слой ушёл бы под фон ближайшего предка-контекста и пропал.

**`options.outflow` читает слой, а не хост.** Флаг приходит пропом `$outflow` и меняет `inset` на `-1.5rem -1.5rem 0` (`BG_IMAGE_OUTFLOW`): картинка вытекает по бокам и вверх, вниз — никогда, иначе она наползала бы на следующий контейнер колонки. Раскладка хоста при этом не меняется: слой абсолютный, содержимое остаётся в своих границах. `1.5rem` — ровно `padding` карточки контейнера (`ContainerWrapper`), поэтому картинка дотягивается до краёв колонки дашборда.

Парная опция `options.innerPadding` живёт на хосте, а не на слое: она приходит пропом `$innerPadding` и даёт корню `padding: 1rem` (`CONTAINER_INNER_PADDING`) — см. [[hooks|`useBgImageHost`]].

Наличие слота определяет [[utils|утилита]] `hasContainerBgImage`. Полное описание механики — в [[concepts#Универсальные слоты и фон контейнера|Основных понятиях]].

```tsx
<ContainerRoot {...root}>
  <ContainerBackground elementConfig={elementConfig} renderElement={renderElement} />
  <ExpandableTitle ... />
  <ContainerChildren ... />
</ContainerRoot>
```

---

## ContainerChildren

**Назначение:** Рендерит список дочерних элементов контейнера. Исключает универсальные слоты, которые контейнер читает по `id` сам (`NON_TRACK_SLOT_IDS` — `title`, `titleIcon`, `bgImage`), проверяет `hideIfEmptyDataSource`.

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
- `useDataSourceLoading(type) || isDiffPage` → `<DashboardLoading />`
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

**Назначение:** Полноэкранный лоадер при смене страницы и пока не пришёл ни один источник данных. Условие показа — [[hooks#useDataSourceLoading|`useDataSourceLoading`]]; используется корневым `Dashboard` и `ElementModal`.

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

## ResizeHandle

**Назначение:** Ручка перетаскивания границы. Общая для [[containers#Режим сетки grid|сетки]] дашборда и колонок таблицы [[elements|ElementTable]]: жест один и тот же, и выглядеть он обязан одинаково. Сам жест — [[hooks|хук]] `useResizeDrag`.

**Props:**
| Prop | Тип | Описание |
|---|---|---|
| `$axis` | `ResizeAxis` — `"row" \| "column"` | Ось: колонка тянется по горизонтали, строка — по вертикали |
| `ref` | `(element: HTMLElement \| null) => void` | `setHandle` из `useResizeDrag` |
| `data-dragging` | `boolean` | Жест идёт: ползунок подсвечен, пока мышь не отпущена |

Ручка абсолютная и целиком лежит в зазоре, поэтому включение режима редактирования не сдвигает раскладку ни на пиксель. Сам ползунок (`::after`) появляется только при наведении и во время перетаскивания, цвет — `palette.primary`; в покое раскладка выглядит как обычно. Ширину зазора берёт из переменной `--grid-gap`: у сетки её объявляет тело сетки, у таблицы зазора нет вовсе и зона захвата сжимается до одного захода на содержимое с каждой стороны.

Модуль экспортирует ещё два имени:

| Имя | Значение |
|---|---|
| `HANDLE_BLEED_PX` | `3` — заход зоны захвата на содержимое с каждой стороны границы. Без него попасть курсором ровно в границу невозможно: у таблицы её толщина один пиксель, а у сетки зазора может не быть вовсе |
| `RESIZE_HANDLE_ATTR` | `"data-grid-handle"` — маркер ручки, компонент ставит его сам. По нему сетка отличает нажатие на границе от нажатия на ячейке (`NO_CELL_DRAG_SELECTOR`), иначе ручка колонки внутри ячейки сетки начинала бы перетаскивание всей ячейки |

```tsx
<ResizeHandle ref={setHandle} $axis="column" data-dragging={dragging} onClick={stopPropagation} />
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

**Назначение:** Обрезает текст — по числу строк (`maxLines`) либо по числу символов (`maxLength`). При `expandable` показывает кнопку «Подробнее»/«Свернуть», иначе полный текст в тултипе. Поддерживает перенос строк через `lineBreak` и режим переноса слов `wordBreak`.

**Props (`TextTrimProps`):**
| Prop | Тип |
|---|---|
| `maxLength` | `number?` |
| `maxLines` | `number?` |
| `expandable` | `boolean?` |
| `lineBreak` | `string?` |
| `wordBreak` | `ConfigTextDisplayOptions["wordBreak"]?` |
| `children` | `string \| number?` |

**Два режима обрезки:**

- `maxLines` **перебивает** `maxLength`: текст уходит во внутренний `TextTrimLineClamp` с CSS line-clamp. Тултип с полным текстом показывается **только при реальном переполнении** (`scrollHeight > clientHeight`), которое отслеживается `ResizeObserver` — у поместившегося текста тултипа нет.
- `maxLength` (без `maxLines`) обрезает по символам с многоточием; полный текст уходит в тултип либо раскрывается кнопкой `LegendToggler` при `expandable`.

```tsx
<TextTrim maxLength={20}>{longTitle}</TextTrim>
<TextTrim maxLines={2} wordBreak="break-word">{description}</TextTrim>
```

---

## Связанные разделы

[[hooks|Хуки]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[headers|Шапки]] | [[types|Типы]]
