# Опции

## Обзор

`ConfigOptions` — общий словарь конфигурационных опций для всех контейнеров, элементов и шапок Dashboard. Исторически это был один плоский интерфейс с 100+ полями. После рефакторинга типизации интерфейс разбит на **13 доменных миксинов**:

```ts
interface ConfigOptions
  extends
    ConfigLayoutOptions,
    ConfigTypographyOptions,
    ConfigExpandableOptions,
    ConfigDataSourceBindingOptions,
    ConfigChartOptions,
    ConfigVisualOptions,
    ConfigTextDisplayOptions,
    ConfigCollectionOptions,
    ConfigMapLayerOptions,
    ConfigEditOptions,
    ConfigVoteOptions,
    ConfigMiscOptions {}
```

> `ConfigEntityRefOptions` — тринадцатый миксин — в `extends` **не входит**: это документирующий «highlight»-интерфейс, чьи поля (`chartId`, `modalId`, `tabId`, ...) уже продублированы в других миксинах (`ConfigChartOptions`, `ConfigEditOptions`, `ConfigMapLayerOptions`, `ConfigMiscOptions`, `ConfigDataSourceBindingOptions`). Он подсвечивает entity-ref природу этих полей — см. раздел ниже.

Каждый компонент использует только часть полей. В `componentTypes.ts` для каждого компонента определён `<Name>Options = Pick<ConfigOptions, ...>` — список фактически читаемых полей. Это даёт:

- **Прозрачность контракта** — по типу `<Name>Options` сразу видно, какие опции компонент реально читает.
- **Безопасность** — при изменении конфига TS подскажет, если опция не поддерживается компонентом.
- **Документируемость** — в `[[types|сводной таблице]]` `[[types#Per-component типы|Per-component типы]]` перечислены Pick для каждого компонента.

Поля, помеченные ⚠️ **entity-ref**, концептуально принадлежат [[types#Branded types|branded keyspaces]] (`ChartId`, `ModalId`, `LayerName`, ...) — даже если в типе сейчас обычная строка.

---

## ConfigLayoutOptions

Размеры, раскладка, отступы.

| Поле | Тип | Описание |
|---|---|---|
| `width` | `CssSize` | Ширина корневой обёртки. Число — px, строка — любое CSS-значение; `"100%"` включает fill-режим (контейнер занимает ячейку целиком). Значение в `fr` (`"2fr"`) — доля трека сетки: её применяет родитель, самому узлу достаётся fill |
| `height` | `CssSize` | Высота корневой обёртки. `"100%"` — заполнить ячейку родителя. Значение в `fr` — доля трека сетки, как у `width` |
| `overflow` | `"visible" \| "hidden" \| "scroll" \| "auto"` | CSS `overflow` корневой обёртки. Не задана — браузерный `visible`: контент крупнее бокса вытекает на соседние слоты. Значения кроме `visible` обрезают и абсолютных потомков — у `FiltersContainer` это открытый список фильтра |
| `padding` | `number` | Внутренние отступы |
| `innerPadding` | `boolean` | Фиксированный внутренний отступ `1rem` по всем краям корня контейнера (`CONTAINER_INNER_PADDING`). Пара к слоту `bgImage`: без отступа содержимое прилипает к краям картинки. От `padding` отличается тем, что не требует числа и не участвует в расчёте размеров обёртки (`getWrapperSizeStyle`). Гейт **не** привязан к наличию фона — отступ работает и без него. Читается [[hooks\|`useBgImageHost`]] / [[hooks\|`useWrapperSize`]] напрямую из `options` |
| `outflow` | `boolean` | Слой фонового изображения (`bgImage`) вытекает за края контейнера на `1.5rem` (`BG_IMAGE_OUTFLOW`) — по бокам и **вверх**, вниз никогда: там начинается следующий контейнер колонки. Раскладку не меняет — слой абсолютный, содержимое остаётся в своих границах. Без слота `bgImage` не делает ничего; обрезается любым предком с `overflow` кроме `visible`, включая собственный `options.overflow`. Читает сам слой ([[components\|`ContainerBackground`]]), а не хост |
| `radius` | `number` | Радиус (для PieChart — относительно контейнера; для табов — `border-radius`) |
| `cornerRadius` | `number` | Закругление углов столбцов BarChart |
| `column` | `boolean` | Вертикальная раскладка детей (один в столбик) |
| `grid` | `boolean` | Переключает `ContainersGroup` в режим CSS-сетки: дети — строки (`GridRow`), дети строк — ячейки, вложенность не ограничена. См. [[containers#Режим сетки grid\|Режим сетки]] |
| `editMode` | `boolean` | Разрешение править содержимое контейнера. У `ContainersGroup` с `grid` — раскладку мышью (ресайз границ, выделение, контекстное меню; читается только у внешнего узла). У `StructuredData` — таблицу (ячейки, добавление и удаление строк, «Отменить»/«Сохранить»); не задан — только чтение. **Не путать** с пропом `editMode` провайдеров — тот про редактирование атрибутов объекта (в контейнерах `isEditing`) |
| `autoHeight` | `boolean` | Сетка растёт под содержимое: `height` становится минимумом (`min-height`), треки — `minmax(auto, Nfr)`, внутренний скролл трека снимается. Читается у **каждого** узла (сетка, `GridRow`, вложенная сетка) и вверх не поднимается. См. [[containers#Рост под содержимое autoHeight\|Рост под содержимое]] |
| `fixedHeight` | `boolean` | Высотой сетки распоряжается внешняя раскладка: у корневой строки не рендерится ручка нижней границы, и высоту нельзя утянуть мышью. Нужен сеткам, растянутым на бокс панели фиксированного экрана или дока. Читается только у **корневого** узла сетки в `editMode`; у вложенных ручки высоты нет и так |
| `twoColumns` | `boolean` | Двухколоночная раскладка (Chart-легенда / fallback ChartContainer) |
| `fill` | `boolean` | Вписать график `ChartContainer` в контейнер: по ширине всегда, по высоте — при заданной `height` (аналог `object-fit: contain`) |
| `align` | `"left" \| "center" \| "right" \| "stretch"` | Выравнивание текста/блоков; для плиток (`DataSource` + `RoundedBackground`) — положение плиток в ряду/столбце |
| `columns` | `number` | Число плиток в строке ряда плиточного контейнера (включает grid-режим) |
| `gap` | `number` | Отступ **между плитками**, px. Читает `DataSourceContainer`. Дефолт `8` (в ряду без `columns` — `0.5rem`). В сетке (`grid`) — зазор между треками: у сетки между строками, у `GridRow` между ячейками, дефолт `0` |
| `innerGap` | `number` | Отступ **между элементами внутри плитки** (иконка ↔ значение ↔ подпись, значение ↔ единицы), px. Читает `RoundedBackgroundContainer`. Не задан — исторические значения (`0.25rem` над подписью и до единиц в строку, `0.5rem` в режиме `big` между блоком иконки и значением; под иконкой и под значением у единиц зазора нет) |
| `alignItems` | `"flex-start" \| "center" \| "flex-end" \| "stretch" \| "baseline"` | Выравнивание детей по поперечной оси. В ряду по умолчанию `center` |
| `fit` | `"cover" \| "contain" \| "fill" \| "none" \| "scale-down"` | CSS `object-fit` изображения элемента `image` |
| `center` | `boolean` | Центрирование контента |
| `innerTemplateStyle` | `CSSProperties` | Кастомные CSS-стили внутреннего шаблона (`OneColumn`, `TwoColumn`, `Progress`, `RoundedBackground`) |
| `withPadding` | `boolean` | Дополнительный внутренний padding (шапки FeatureCard) |
| `withDivider` | `boolean` | Показывать разделитель |
| `bottomBlur` | `boolean` | Эффект размытия снизу (`FeatureCardBackgroundHeader`) |
| `noMargin` | `boolean` | Убрать внешний margin |
| `maxTextWidth` | `number` | Максимальная ширина текста в px |
| `barWidth` | `number` | Ширина столбца BarChart |
| `barHeight` | `number` | Высота StackBar |

**Используется в:** `ElementChart`, `ElementImage`, `ElementSvg`, `ElementControl`, `ElementTable`, `ChartContainer`, `ContainersGroupContainer` (`grid`, `editMode`, `autoHeight`, `fixedHeight`, `gap`), `DataSourceContainer`, `DataSourceInnerContainer`, `GridRowContainer` (`gap`, `alignItems`, `autoHeight`), `OneColumnContainer`, `PagesContainer`, `ProgressContainer`, `RoundedBackgroundContainer`, `StructuredDataContainer` (`editMode`), `TabsContainer`, `TaskContainer`, `TwoColumnContainer`, `FeatureCardBackgroundHeader`, `FeatureCardDefaultHeader`, `FeatureCardSlideshowHeader`.

`innerPadding` и `outflow` — **любой** контейнер: ни один `<Name>Options` их не Pick'ает, значения читают хук хоста и слой фона (см. врезку про пайплайн в [[options#ConfigMiscOptions|ConfigMiscOptions]]).

---

## ConfigTypographyOptions

Типографика и цвета.

| Поле | Тип | Описание |
|---|---|---|
| `fontSize` | `string` | CSS-длина (`"14px"`, `"0.875rem"`, `"larger"`) — см. `FontSizeToken` |
| `fontColor` | `string` | Цвет текста (CSS) |
| `bgColor` | `string` | Цвет фона. В плитке (`RoundedBackground`) — сплошная заливка, перебивающая тонированный фон от цвета иконки/текста, см. [[containers#RoundedBackgroundContainer\|приоритет цвета фона плитки]] |
| `backgroundColor` | `string` | Альтернативное имя для bgColor |
| `primaryColor` | `string` | Основной акцентный цвет |
| `defaultColor` | `string` | Цвет по умолчанию (PieChart-секторов и т.п.) |
| `noBg` | `boolean` | Отключить фон |
| `colors` | `string[]` | Массив цветов (Progress, multi-series Chart) |
| `colorAttribute` | `string` | Имя атрибута, из которого брать цвет |
| `statusColors` | `Record<RemoteTaskStatus, string>` | Цвета по статусам задачи |
| `typography` | `MarkdownTypography` | Пер-тег типографика markdown-виджета: `{ h1..h6, p, li, code }` → `{ fontSize, lineHeight, fontWeight, marginTop, marginBottom }`. Отсутствующий тег или свойство — дефолт `MarkdownWrapper`. См. [[types#Типографика markdown\|Типографика markdown]] |

**Используется в:** `ElementChart`, `ElementChips`, `ElementIcon`, `ElementLegend`, `ElementMarkdown` (`typography`), `ElementSvg`, `DividerContainer`, `FiltersContainer`, `ProgressContainer`, `RoundedBackgroundContainer`, `TabsContainer`, `TaskContainer`, `FeatureCardBackgroundHeader`, `FeatureCardSlideshowHeader`.

---

## ConfigExpandableOptions

Поведение «раскрытия» (свернуть/развернуть).

| Поле | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Возможность сворачивать/разворачивать содержимое |
| `expanded` | `boolean` | Начальное состояние (раскрыто/свёрнуто) |
| `expandLength` | `number` | Длина текста, после которой показывается кнопка «показать ещё» (Markdown) |

**Используется в:** `ElementCamera`, `ElementChart`, `ElementMarkdown`, `ElementSlideshow`, `AttachmentContainer`, `CameraContainer`, `ContainersGroupContainer`, `DataSourceContainer`, `DataSourceProgressContainer`, `EditGroupContainer`, `FiltersContainer`, `LayersContainer`, `SlideshowContainer`, `UploadContainer`.

---

## ConfigDataSourceBindingOptions

Связи с источниками данных.

| Поле | Тип | Описание |
|---|---|---|
| `attributesDescription` | `ConfigAttributeDescription[]` | Схема структурированных данных [[containers#StructuredDataContainer\|StructuredDataContainer]]. С источником — переопределение его схемы (состав, `alias`, `stringFormat`, `isEditable`; типы — от источника), без источника — единственная схема и обязательна |
| `relatedDataSource` | `string` | ⚠️ entity-ref на `[[concepts#Источники данных\|ConfigDataSource]]` (см. `[[types#Branded types\|DataSourceName]]`) |
| `relatedDataSources` | `ConfigRelatedDataSource[]` | Несколько источников с alias/axis для серий графика |
| `relatedAttributes` | `ConfigRelatedAttribute[]` | Атрибуты из связанных слоёв (join) |
| `relatedResources` | `ConfigRelatedResource[]` | Связанные Python-ресурсы (TaskContainer) |
| `responseFilters` | `Record<string, string>` | Фильтры ответа задачи |
| `hideIfEmptyDataSource` | `string` | Скрыть контейнер, если указанный источник пуст |

**Используется в:** `ElementChart`, `ElementControl`, `ElementLegend`, `ElementSlideshow`, `AttachmentContainer`, `DataSourceContainer`, `DataSourceInnerContainer`, `DataSourceProgressContainer`, `EditAttachmentContainer`, `StructuredDataContainer` (`attributesDescription`, `relatedDataSource`), `TaskContainer`.

---

## ConfigChartOptions

Опции графиков.

| Поле | Тип | Описание |
|---|---|---|
| `chartType` | `"bar" \| "line" \| "pie" \| "stack"` | Тип диаграммы |
| `chartId` | `string` | ⚠️ entity-ref на `ChartContainer` ребёнка по id (см. `[[types#Branded types\|ChartId]]`) |
| `markers` | `BarChartMarker[] \| string` | Маркеры на BarChart |
| `showLabels` | `boolean` | Показывать подписи столбцов |
| `showMarkers` | `number` | Сколько маркеров показать |
| `showTotal` | `boolean` | Показывать итог |
| `totalWord` | `string` | Слово для итога (например, «Всего») |
| `totalAttribute` | `string` | Имя атрибута для подсчёта итога |
| `dotSnapping` | `boolean` | Привязка точек LineChart к ближайшему значению |
| `drawMinMax` | `boolean` | Рисовать min/max-маркеры |
| `angle` | `number` | Угол поворота подписей оси (BarChart) |

**Используется в:** `ElementChart`, `ElementLegend`, `DataSourceProgressContainer`.

---

## ConfigVisualOptions

Иконки и изображения.

| Поле | Тип | Описание |
|---|---|---|
| `icon` | `IconTypesKeys` | Имя иконки из дизайн-системы |
| `iconAttribute` | `string` | Имя атрибута, из которого брать иконку |
| `image` | `string` | URL изображения |
| `resourceId` | `string` | Id файлового ресурса с картинкой — разворачивается в `/sp/resources/file/<id>` |
| `overlay` | `string` | Наложение поверх изображения (CSS gradient, цвет) |
| `onlyIcon` | `boolean` | Показывать только иконку (без текста) |
| `bigIcon` | `boolean` | Большой размер иконки. В плитке (`RoundedBackground`) — водяной знак `3rem` в правом верхнем углу, прозрачность `0.12` |
| `big` | `boolean` | Большой размер компонента |
| `tagView` | `boolean` | Отображать в виде «тэга» |

**Используется в:** `ElementImage` (`resourceId`), `ElementModal`, `ElementUploader`, `AddFeatureButton`, `ExportPdfContainer`, `RoundedBackgroundContainer`, `TabsContainer` (`onlyIcon`), `TaskContainer`, `FeatureCardBackgroundHeader`, `FeatureCardDefaultHeader`, `FeatureCardSlideshowHeader`.

---

## ConfigTextDisplayOptions

Текст и заголовки.

| Поле | Тип | Описание |
|---|---|---|
| `title` | `string` | Заголовок |
| `label` | `string` | Подпись (label инпута) |
| `placeholder` | `string` | Плейсхолдер инпута |
| `hideTitle` | `boolean` | Скрыть заголовок |
| `simple` | `boolean` | Упрощённый вариант (Title/Link без обвязки) |
| `maxLength` | `number` | Максимальная длина текста до обрезки |
| `maxLines` | `number` | Обрезка по числу строк (CSS line-clamp) вместо обрезки по символам. Читает [[components\|`TextTrim`]]: при заданном `maxLines` он игнорирует `maxLength` и показывает тултип с полным текстом только при реальном переполнении. В плитке (`RoundedBackground`) применяется к подписи; дефолт для плиточного ряда с `columns` — `2` |
| `wordBreak` | `"break-word" \| "break-all"` | Стратегия переноса длинного текста |
| `separator` | `string` | Разделитель элементов (Chips) |
| `lineBreak` | `string` | Кастомный перевод строки |

**Используется в:** `ElementChips`, `ElementControl`, `ElementLink`, `ElementUploader`, `AddFeatureButton`, `ExportPdfContainer`, `FilterChild`, `ProgressContainer`, `RoundedBackgroundContainer`, `TabsContainer`, `TaskContainer`, `TitleContainer`.

---

## ConfigCollectionOptions

Списки и коллекции.

| Поле | Тип | Описание |
|---|---|---|
| `shownItems` | `number` | Сколько элементов показывать сразу |
| `otherItems` | `number` | Лимит «остальных» в развёрнутом списке |
| `orderByValue` | `boolean` | Сортировать по значению |
| `orderByTitle` | `boolean` | Сортировать по заголовку |
| `sort` | `boolean` | Разрешить сортировку представления кликом по заголовку колонки. Сортировка локальная — только вид |
| `viewMode` | `"grid" \| "list"` | Режим отображения коллекции |
| `hideEmpty` | `boolean` | Скрывать пустые элементы |
| `limit` | `number` | Лимит на количество элементов |

**Используется в:** `ElementChart` (`shownItems`, `otherItems` — сколько категорий показать на графике и сколько свернуть в «Другое»), `AttachmentContainer`, `ChartContainer` (`hideEmpty`), `DataSourceContainer` (`shownItems`, `otherItems`), `DataSourceProgressContainer` (`shownItems`, `otherItems`), `EditAttachmentContainer`, `OneColumnContainer` (`hideEmpty`), `RoundedBackgroundContainer` (`hideEmpty`), `TabsContainer` (`shownItems`), `TwoColumnContainer` (`hideEmpty`), `ElementTable` (`sort`).

---

## ConfigMapLayerOptions

Карта и слои.

| Поле | Тип | Описание |
|---|---|---|
| `layerName` | `string` | ⚠️ entity-ref на `[[concepts#Слои\|ConfigLayer]]` (см. `[[types#Branded types\|LayerName]]`) |
| `layerNames` | `string[]` | ⚠️ entity-ref на несколько слоёв |
| `geometryType` | `OgcGeometryType \| EditGeometryType` | Тип геометрии (Point/Polygon/LineString/...) |
| `baseMapName` | `string` | Имя базовой карты |
| `baseMapSettings` | `Record<string, BaseMapSettings>` | Настройки базовых карт (opacity, showBuildings) |
| `expandedLayers` | `boolean` | Раскрытое состояние дерева слоёв |
| `customFeatureSelect` | `CustomFeatureSelect` | Кастомные стили выделения объекта на карте |
| `pitch` | `number` | Угол наклона карты |
| `bearing` | `number` | Поворот карты |
| `srid` | `string` | SRID проекции |
| `maxZoomTo` | `number` | Максимальный зум при `zoomTo` |
| `position` | `PositionDc` | Координаты центра |
| `resolution` | `number` | Разрешение |

**Используется в:** `AddFeatureButton` (`layerName`, `geometryType`), `LayersContainer` (`layerNames`).

---

## ConfigEditOptions

Контролы редактирования и фильтрации.

| Поле | Тип | Описание |
|---|---|---|
| `control` | `ConfigControl` | Описание одного контрола (тип, атрибут, параметры) |
| `controls` | `ConfigControl[]` | Несколько контролов (Edit-группы, чек-боксы) |
| `filterName` | `string` | ⚠️ entity-ref на `[[concepts#Фильтры\|ConfigFilter]]` (см. `[[types#Branded types\|FilterName]]`) |
| `searchFilterName` | `string` | ⚠️ entity-ref на поисковый фильтр |
| `variants` | `IOption[] \| ChipOption[]` | Список вариантов для Chips/Dropdown |
| `multiSelect` | `boolean` | Множественный выбор |
| `withTime` | `boolean` | Включить выбор времени (EditDate) |
| `step` | `number` | Шаг RangeNumber |
| `minValue` | `number \| Date` | Минимум диапазона |
| `maxValue` | `number \| Date` | Максимум диапазона |
| `noEmptyOption` | `boolean` | Запретить пустой выбор |
| `fileExtensions` | `string` | Допустимые расширения файлов (например, `".pdf,.png"`) |
| `editOnly` | `boolean` | Оставить таблице ([[elements#ElementTable\|`table`]]) только правку существующих строк: кнопки «Добавить» и колонки удаления не будет. Поверх `editMode` контейнера. Опция объявлена у представления, а читает её [[containers#StructuredDataContainer\|контейнер]] — кнопка добавления его |

**Используется в:** `ElementControl` (`control`), `ElementUploader` (`fileExtensions`, `multiSelect`), `ElementSlideshow` (`controls`), `AttachmentContainer` (`controls`), `EditGroupContainer`, `EditBooleanContainer`, `EditStringContainer`, `EditNumberContainer`, `EditDropdownContainer`, `EditChipsContainer`, `EditCheckboxContainer`, `EditDateContainer` (`withTime`, `controls`), `EditAttachmentContainer` (`controls`, `fileExtensions`), `FilterChild`, `DataSourceProgressContainer` (`maxValue`), `ProgressContainer` (`maxValue`), `ElementTable` (`editOnly`).

---

## ConfigVoteOptions

Слои-таблицы контейнера [[containers#VoteContainer|Vote]]. Все четыре обязательны — без любой из них контейнер отдаёт ошибку блока.

| Поле | Тип | Описание |
|---|---|---|
| `categoryDataSource` | `string` | Слой-справочник категорий: `id`, `name` |
| `questionDataSource` | `string` | Слой голосований: `id`, `text`, `category_id`, `user_name`, `multi_select`. На запись |
| `variantDataSource` | `string` | Слой вариантов ответа: `id`, `question_id`, `text`. На запись |
| `answerDataSource` | `string` | Слой ответов: `id`, `question_id`, `user_name`, `variant_id`. На запись |

> [!warning] Это имена СЛОЁВ, а не источников данных страницы
> Несмотря на суффикс `DataSource`, контейнер обращается к ним через `api.layers.*` напрямую, мимо механики [[concepts#Источники данных|источников данных]]: имя из `dataSources` страницы он не найдёт. Схема полей захардкожена в контейнере (`VOTE_FIELDS`) и конфигом не переопределяется.

Имя атрибута объекта с `question_id` в этот миксин **не входит**: оно живёт свойством `attributeName` самого узла конфига, а не в `options`.

**Используется в:** `VoteContainer`.

---

## ConfigEntityRefOptions

Ссылки на сущности по id/name. **Дублирует** поля из других миксинов — этот миксин подсвечивает их «entity-ref» природу.

| Поле | Тип | Branded keyspace |
|---|---|---|
| `chartId` | `string` | `[[types#Branded types\|ChartId]]` |
| `modalId` | `string` | `[[types#Branded types\|ModalId]]` |
| `tabId` | `string` | `[[types#Branded types\|TabId]]` |
| `filterName` | `string` | `[[types#Branded types\|FilterName]]` |
| `searchFilterName` | `string` | `[[types#Branded types\|FilterName]]` |
| `layerName` | `string` | `[[types#Branded types\|LayerName]]` |
| `layerNames` | `string[]` | `[[types#Branded types\|LayerName]]` |
| `relatedDataSource` | `string` | `[[types#Branded types\|DataSourceName]]` |
| `parentResourceId` | `string` | `[[types#Branded types\|ResourceId]]` |
| `downloadById` | `string` | `[[types#Branded types\|ResourceId]]` |

При написании нового кода для entity-id рекомендуется использовать конструкторы из `branded.ts` (`asChartId`, `asLayerName`, ...) — см. `[[types|Типы]]`.

---

## ConfigMiscOptions

Поля без явного домена, обычно широкого назначения.

| Поле | Тип | Описание |
|---|---|---|
| `innerTemplateName` | `ContainerTemplate` | **Обязателен для `DataSource`/`DataSourceProgress`** (входит в их `<Name>Options`, но как необязательное поле — типы пропуск не ловят). Шаблон рендеринга каждой записи источника: читается пайплайном ([[containers\|`getRenderElement`]]) и конвертируется в проп `innerComponent` через `getContainerComponent`. Без него записи не рендерятся |
| `themeName` | `"light" \| "dark"` | Принудительная тема для шапки |
| `url` | `string` | URL: ссылка по клику на лого в `DashboardDefaultHeader`; адрес картинки в `ElementImage` (через `getResourceUrl`: `http…` как есть, остальное — `/sp/resources/file/<url>`) |
| `inlineUnits` | `boolean` | Показывать единицы измерения inline |
| `noUnits` | `boolean` | Не показывать единицы измерения |
| `attributes` | `string[]` | Список атрибутов для отображения |
| `useProjectHiddenAttributes` | `boolean` | Учитывать `hiddenAttributes` проекта (PUBGL-658) |
| `innerValue` | `boolean` | Показывать значение внутри Progress |
| `groupTooltip` | `boolean` | Группировать tooltip |
| `wrap` | `boolean` | Переносить содержимое |
| `modalId`, `tabId`, `downloadById`, `parentResourceId` | `string` | Дубликаты entity-ref для удобства Pick |
| `useNotifications` | `boolean` | Показывать прогресс-уведомления о выполнении задачи (`TaskContainer`) |

**Используется в:** `DataSourceContainer`, `DataSourceProgressContainer` (`innerTemplateName` — значение потребляет пайплайн рендера, а не пропсы контейнера), `DashboardDefaultHeader` (`url`), `EditGroupContainer`, `OneColumnContainer`, `TwoColumnContainer` (`attributes`, `useProjectHiddenAttributes`), `ProgressContainer` (`innerValue`), `RoundedBackgroundContainer` (`inlineUnits`), `TitleContainer` (`downloadById`), `ElementModal` (`modalId`), `ElementUploader` (`parentResourceId`), `EditAttachmentContainer` (`parentResourceId`), `TaskContainer` (`useNotifications`), `FeatureCardBackgroundHeader`, `FeatureCardDefaultHeader`, `FeatureCardSlideshowHeader` (`themeName`).

> [!note] Опции, потребляемые пайплайном рендера
> Часть полей читается **пайплайном рендера** напрямую из `options`, а не пропсами компонента — их «Используется в» определяется по фактическому чтению в коде, а не по тому, чей `<Name>Options` их Pick'ает. Ключевой пример — **`innerTemplateName`**: его читает [[containers|`getRenderElement`]] и конвертирует в `innerComponent` (`getContainerComponent`) для `DataSourceContainer`/`DataSourceProgressContainer`; сам контейнер этой опции в пропсах не видит, хотя она и перечислена в его `<Name>Options`. Не считай такие поля неиспользуемыми только потому, что компонент к ним не обращается.
>
> Тем же путём идут **`innerPadding`** и **`outflow`** из [[options#ConfigLayoutOptions|ConfigLayoutOptions]] — с той разницей, что их не Pick'ает **ни один** `<Name>Options`: `innerPadding` читает [[hooks|`useBgImageHost`]] (через него — `useWrapperSize`/`useContainerRoot`) и кладёт на корень пропом `$innerPadding`, `outflow` читает сам слой [[components|`ContainerBackground`]] пропом `$outflow`. Поэтому обе опции допустимы у любого контейнера, а не у перечисленного списка.

---

## Связанные разделы

[[types|Типы]] | [[concepts|Основные понятия]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[headers|Шапки]]
