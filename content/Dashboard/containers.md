# Контейнеры

> [!danger] У каждого контейнера обязателен уникальный `id`
> Без него не работают навигация, связи и управление состоянием — контейнер выпадает из рендера. Правила и чек-лист генерации — [[authoring|Правила генерации]].

## Обзор

Контейнеры — компоненты верхнего уровня, управляющие расположением и логикой отображения контента. Регистрируются в `containers/registry.ts` (типобезопасно через `as const satisfies ContainerComponentRegistry`) по значению `ContainerTemplate`. Рендерятся динамически через [[utils|утилиту]] `getContainerComponent`. Все принимают `ContainerProps`.

Каждый контейнер имеет тройку типов `<Name>ContainerOptions` / `<Name>ContainerConfig` / `<Name>ContainerProps` (см. [[types#Контейнеры|сводную таблицу]]). В карточках ниже указан литерал `templateName` и поля `<Name>Options` — какие опции из [[options|`ConfigOptions`]] контейнер реально читает.

**Каждый контейнер обязан иметь поле `id`** — уникальное имя в keyspace `ConfigContainer.id` (типы `ContainerId` / `ChartId` / `ModalId` / `TabId`). Через него работают навигация, связи (`tabId`, `modalId`, `chartId`, `downloadById`), управление состоянием (`expandedContainers`, `selectedTabId`). Подробности и таблица slot-id для `children` — в [[concepts#ID контейнеров и элементов|разделе про id]]. В примерах ниже `id` контейнеров — иллюстративные доменные имена; в реальном конфиге они должны быть уникальны.

## Размерная модель обёртки (`ContainerBoxOptions`)

Три опции размера корневой обёртки общие для контейнеров и в таблицах ниже отдельно не повторяются — они подмешиваются в `<Name>Options` миксином `ContainerBoxOptions` (см. [[types#Общие миксины размеров|Типы]]):

| Опция | Тип | Описание |
|---|---|---|
| `width` | `CssSize` | Ширина обёртки. Число — px, строка — любое CSS-значение. Не задана — контейнер занимает всю ширину ячейки (`100%` по умолчанию) |
| `height` | `CssSize` | Высота обёртки. `"100%"` — заполнить ячейку родителя |
| `overflow` | `"visible" \| "hidden" \| "scroll" \| "auto"` | Что делать с контентом, вылезающим за бокс. Не задана — браузерный `visible`: контент вытекает на соседние слоты. Значения кроме `visible` обрезают и абсолютно спозиционированных потомков — у `Filters` с заданной высотой обрежется открытый список фильтра |

Поддерживают: `Attachment`, `Camera`, `Chart`, `ContainersGroup`, `DataSource`, `DataSourceProgress`, `Filters`, `Image`, `Layers`, `OneColumn`, `Slideshow`, `Task`, `TwoColumn`, `Upload`. Значение `"100%"` включает **fill-режим**: обёртка занимает ячейку целиком, конфликтующие внутренние дефолты (собственные `width`/`height`/`margin`) снимаются, а для fill-высоты добавляется `flex: 1 1 auto`. Реализация — [[utils|`getWrapperSizeStyle`]] + [[hooks|`useWrapperSize`]]; размеры уходят styled-пропом, поэтому перебиваются снаружи обычной специфичностью, без `!important`.

Остальные контейнеры (`AddFeature`, `DefaultAttributes`, `Divider`, `Edit*`, `ExportPdf`, `Icon`, `Progress`, `RoundedBackground`, `Tabs`, `Title`) своей размерной модели не имеют; у `Pages` из размеров есть только `width`.

> [!info] Размер в `fr` — это доля трека сетки
> `width`/`height` вида `"2fr"` применяет не сам узел, а его родитель — [[containers#Режим сетки grid|сетка]] или [[containers#GridRowContainer|строка сетки]] — при сборке `grid-template-*`. Самому узлу такое значение даёт fill-режим, как `"100%"`.

---

## Список контейнеров

### AddFeatureContainer

**Назначение:** Отображает кнопки добавления объектов на слой карты. При нажатии активирует инструмент рисования для указанного слоя и типа геометрии.

**Типы:** `templateName = "AddFeature"` · `AddFeatureContainerOptions` (Record<string, never>) · `AddFeatureContainerProps`. Дети — `AddFeatureButtonChild` (`type: "button"`, `AddFeatureButtonOptions`).

**Props:** `ContainerProps` (использует `elementConfig.children` с `type === "button"`)

**Опции дочерних элементов (`AddFeatureButtonOptions`):**

| Опция | Тип | Описание |
|---|---|---|
| `icon` | `IconTypesKeys` | Иконка кнопки |
| `title` | `string` | Подпись кнопки |
| `layerName` | `string` | Имя слоя, на который добавляется объект (см. [[types#Branded types\|LayerName]]) |
| `geometryType` | `OgcGeometryType \| EditGeometryType` | Тип геометрии: `"Point"`, `"LineString"`, `"Polygon"`, ... |

```tsx
{
  id: "buildings_add",
  templateName: "AddFeature",
  children: [
    {
      id: "add_building_polygon",
      type: "button",
      options: { icon: "feature_add", title: "Добавить здание", layerName: "buildings", geometryType: "Polygon" }
    },
    {
      id: "add_road_line",
      type: "button",
      options: { icon: "feature_add", title: "Добавить дорогу", layerName: "roads", geometryType: "LineString" }
    }
  ]
}
```

---

### AttachmentContainer

**Назначение:** Отображение списка вложений (документы, изображения, ссылки) объекта или источника данных. Поддерживает превью изображений с авторизованной загрузкой через `api.catalog.getFile`, fallback-иконки по типу файла, переключение `viewMode` (`grid` / `list`), пагинацию `shownItems`/`otherItems`.

**Типы:** `templateName = "Attachment"` · `AttachmentContainerOptions` · `AttachmentContainerConfig`. Локальные типы — `FileType` enum (XLSX/PDF/CSV/...), `IMAGE_FILE_TYPES`, `AttachmentViewMode`, `Attachment` (в `containers/AttachmentContainer/types.ts`).

**Props:** `AttachmentContainerProps`

**Опции (`AttachmentContainerOptions`):**

| Опция | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |
| `viewMode` | `"grid" \| "list"` | Режим отображения коллекции |
| `shownItems` | `number` | Сколько элементов показывать сразу |
| `otherItems` | `number` | Лимит «остальных» в развёрнутом списке |
| `relatedDataSource` | `string` | Источник данных вложений (см. [[types#Branded types\|DataSourceName]]) |
| `controls` | `ConfigControl[]` | Маппинг полей источника на поля `Attachment`: `[{ attributeLink, attributeName, attributeMime, attributeDate }]` |

**Зависимости:**
- [[hooks|хук]] `useAttachmentItems` — извлекает список `Attachment[]` из атрибута или источника
- [[hooks|хук]] `useAttachmentPreviewImages` — превращает в `IPreviewImage[]` с загрузкой blob

**Слоты (`children`):** `alias` — подпись в шапке списка вложений; `value` — источник вложений, когда `relatedDataSource` не задан (значение его `attributeName` разбирается через `parseAttachments`).

```tsx
{
  id: "files_attachments",
  templateName: "Attachment",
  options: {
    viewMode: "grid",
    shownItems: 6,
    relatedDataSource: "files_ds",
    controls: [{ attributeLink: "url", attributeName: "name", attributeMime: "mime_type", attributeDate: "uploaded_at" }]
  }
}
```

---

### CameraContainer

**Назначение:** Обёртка для элемента камеры — отображает галерею снимков камеры видеонаблюдения с таймлайном.

**Типы:** `templateName = "Camera"` · `CameraContainerOptions` · `CameraContainerProps`.

**Props:** `ContainerProps` + поддержка `expandable/expanded`

**Опции (`CameraContainerOptions`):**

| Опция | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Разрешить сворачивание контейнера |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

```tsx
{
  id: "entry_camera",
  templateName: "Camera",
  options: { expandable: true, expanded: true },
  children: [
    { id: "alias", value: "Камера въезда" },
    { id: "value", type: "camera", attributeName: "cameraUrl" }
  ]
}
```

---

### ChartContainer

**Назначение:** Контейнер для отображения графика с легендой и псевдонимом. Поддерживает Bar, Pie, Line и другие типы через дочерний элемент `chart`. Использует [[hooks|хук]] `useChartData`.

**Типы:** `templateName = "Chart"` · `ChartContainerOptions` · `ChartContainerProps`. Дети — `ChartContainerChild`: `ChartAliasChild` (`id: "alias"`), `ChartChartChild` (`id: "chart"`, `type: "chart"`), `ChartLegendChild` (`id: "legend"`, `type: "legend"`, опции `ChartLegendChildOptions` = `ElementLegendOptions` + `center`), `ChartTitleChild` (`id: "title"`, `type: "text"`) и `ChartTitleIconChild` (`id: "titleIcon"`, `type: "icon"`) — два последних от ExpandableTitle. См. [[types#Контейнеры|сводную таблицу]].

**Props:** `ContainerProps` (использует `elementConfig.children` с id `alias`, `chart`, `legend`)

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `twoColumns` | `boolean` | Разместить легенду и график в две колонки |
| `hideEmpty` | `boolean` | Скрыть контейнер при отсутствии данных |
| `fill` | `boolean` | Вписать **тело графика** в контейнер (аналог `object-fit: contain`): по ширине — всегда, по высоте — только если задана `height` контейнера |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`) — см. [[containers#Размерная модель обёртки ContainerBoxOptions|размерную модель]].

При ошибке источника данных рендерит `<DataSourceError />`.

#### Как работает `fill`

Элемент `chart` рендерится через `renderElement({ id: "chart" })` и опций контейнера не видит, поэтому `ChartContainer` передаёт режим через контекст `FillContext` (`{ fill, fitHeight }`, где `fitHeight = fill && height != null`), а компонент `Chart` читает его через `useContext`.

- **По ширине** обёртка растягивается на `100%`, а телу графика отдаётся реально измеренная ширина ячейки: d3-графики требуют пиксельное число, а не CSS `100%`. Измерение — [[hooks|хук]] `useResizeBox` (ResizeObserver) на внешней обёртке `ChartFillMeasure`; без `fill` наблюдение отключено и лишних ре-рендеров нет.
- **По высоте** фит включается только при заданной `height` контейнера. Тогда `chartHeight` означает не «высоту тела», а **всю доступную высоту ячейки**, из которой вычитается обвес: подписи оси X и внешний отступ у BarChart, строка итога у StackBar (её высота не прибавляется, а забирается флексом). Круглый график (`pie`) вписывается по короткой стороне и центрируется.
- Без `fill` поведение прежнее: `options.width`/`options.height` элемента `chart` задают фиксированную геометрию.

```tsx
{
  id: "floors_chart",
  templateName: "Chart",
  options: { fill: true, height: 240 },     // fill + height → фит по обеим осям
  children: [
    { id: "alias", value: "Этажность" },
    { id: "chart", type: "chart", options: { chartType: "bar" } },
    { id: "legend", type: "legend", options: { chartId: "chart" } }
  ]
}
```

```tsx
{
  id: "types_chart",
  templateName: "Chart",
  options: { twoColumns: true },
  children: [
    { id: "alias", value: "Распределение по типам" },
    { id: "chart", type: "chart", options: { chartType: "pie", relatedDataSources: [{ name: "types_ds", chartAxis: "y" }] } },
    { id: "legend", type: "legend", options: { chartId: "chart" } }
  ]
}
```

---

### ContainersGroupContainer

**Назначение:** Группа контейнеров — базовый составной контейнер, используемый как страница дашборда. Является `default` в registry (применяется когда `templateName` не найден).

**Props:** `ContainerProps`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `column` | `boolean` | `true` (default) — дочерние контейнеры в колонку, `false` — в строку |
| `expandable` | `boolean` | Разрешить сворачивание группы |
| `expanded` | `boolean` | Развёрнута ли группа по умолчанию |
| `alignItems` | `"flex-start" \| "center" \| "flex-end" \| "stretch" \| "baseline"` | Выравнивание детей по поперечной оси (CSS `align-items`). В ряду по умолчанию `center` |
| `grid` | `boolean` | Переключает контейнер в режим сетки — см. раздел ниже |
| `editMode` | `boolean` | Редактирование раскладки сетки мышью. Работает только вместе с `grid` |
| `gap` | `number` | Зазор между строками сетки, px. Дефолт `0` — сетка бесшовная. Читается только в режиме `grid` |
| `autoHeight` | `boolean` | Сетка растёт под содержимое вместо внутреннего скролла — см. раздел ниже. Работает только вместе с `grid` |
| `fixedHeight` | `boolean` | Высота сетки задана снаружи: у корневой строки не рендерится ручка нижней границы. Читается только у корневого узла сессии `editMode` |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`).

Рендерит дочерние элементы через `ContainerChildren`. Если `id` начинается с `"page_"` — рендерится как корневой контейнер страницы.

Корневая обёртка группы всегда `box-sizing: border-box`, поэтому `padding` из авторского `style` не раздувает бокс сверх выданного размера — указывать `boxSizing` в конфиге не нужно.

```tsx
{
  id: "main_group",
  templateName: "ContainersGroup",
  options: { column: false },
  children: [leftPanel, rightPanel]
}
```

#### Режим сетки (`grid`)

При `options.grid` дети контейнера трактуются как **строки** ([[containers#GridRowContainer|`GridRow`]]), дети строки — как **ячейки** (снова `ContainersGroup`). Любая ячейка может включить `grid` у себя — вложенность не ограничена.

Доли треков задаются в `fr`: у строки — `options.height`, у ячейки — `options.width`. **Долю применяет родитель**, собирая из детей `grid-template-rows` / `grid-template-columns`; сам узел растягивается на выделенный ему трек (`fr` на узле трактуется как fill, см. [[utils#getWrapperSizeStyle|getWrapperSizeStyle]]). Значения не в `fr` сеткой не поддерживаются и считаются за `1fr`.

> [!warning] Внешней сетке нужна заданная высота
> Без `options.height` (у сетки или у её родителя) высота контейнера не определена, и `1fr`-строки по спецификации CSS раскладываются по содержимому, а не пропорционально.
>
> Внутрь высота передаётся сама: сетка, строка и вложенная сетка по умолчанию занимают выделенный им бокс целиком (`height: 100%`), поэтому задавать её на каждом уровне не нужно — достаточно внешнего узла.

> [!info] Содержимое заперто в треке
> Трек по содержимому не растёт (`minmax(0, Nfr)`), поэтому содержимое ячейки не может выйти за её границы: не влезло — внутри ячейки появляется скролл. Без этого лишнее рисовалось бы поверх соседней строки, причём поверх оказывалась бы нижняя — она позже в DOM.
>
> Побочный эффект тот же, что у [[options#ConfigLayoutOptions|`options.overflow`]]: абсолютно спозиционированные потомки (например, открытый список фильтра) обрезаются по боксу ячейки.

Настоящего `rowspan` в модели нет: вертикальное объединение достигается вложенностью — ячейка, поделённая на собственные строки, даёт тот же результат и всегда остаётся валидной раскладкой.

#### Рост под содержимое (`autoHeight`)

Договор «содержимое заперто в треке» подходит не всем секциям. Витрина, доращивающая контент уже после рендера («Показать ещё»), в нём получает внутренний скролл вместо того, чтобы растянуть секцию. `options.autoHeight` переворачивает договор:

| Что | Обычный режим | `autoHeight` |
|---|---|---|
| `options.height` узла | жёсткая `height` | минимум: `min-height` |
| Дефолт сетки, строки, вложенной сетки | `height: 100%` | `min-height: 100%` |
| Строки сетки | `minmax(0, Nfr)` | `minmax(auto, Nfr)` |
| Ячейки строки, вертикально | `minmax(0, 1fr)` | `minmax(auto, 1fr)` |
| Колонки строки | `minmax(0, Nfr)` | без изменений — ширину растить нечем |
| Содержимое трека | `overflow: auto` | `overflow: visible` |
| Ручка нижней границы в `editMode` | пишет высоту | пишет тот же `options.height`, но как минимум |

> [!warning] Опция читается у КАЖДОГО узла и вверх не поднимается
> Сетка объявляет, что её **строки** могут расти; строка — что её **ячейки** могут растить строку. Включённая только у корня, опция упрётся в первую же строку, оставшуюся с `height: 100%`, — рост от ячейки до сетки не дойдёт. Ставить её нужно на всей цепочке предков растущего узла.

Два следствия, из-за которых режим не сделан поведением по умолчанию:

- **Точен для секции с одной строкой.** При нескольких переросшая строка раздувает соседей пропорционально их долям: это семантика `fr`, и «сосед держит свои пиксели» в CSS Grid не выражается. Сетка с `min-height: 300px` и строками `1fr`/`2fr` пустой даёт `100`/`200`; контент `500px` в первой строке даёт `500`/`1000` и итог `1500`.
- **Горизонтальный клип уходит вместе с вертикальным.** Пары `overflow-x: auto` + `overflow-y: visible` в CSS не существует — вторая ось поднимает первую до `auto`. Широкое содержимое в этом режиме вытекает вбок.

Технически минимум держится ещё и тем, что обёртка трека теряет свой `min-height: 0`: этот ноль — «минимальный вклад» grid-элемента, из которого сетка считает `auto`-минимум трека, и с ним `minmax(auto, Nfr)` не сработал бы вовсе.

```tsx
{
  id: "main_grid",
  templateName: "ContainersGroup",
  options: { grid: true, height: "22rem" },
  children: [
    {
      id: "row_1", templateName: "GridRow", options: { height: "1fr" },
      children: [
        { id: "cell_a", templateName: "ContainersGroup", options: { width: "2fr" }, children: [chart] },
        { id: "cell_b", templateName: "ContainersGroup", options: { width: "1fr" }, children: [filters] }
      ]
    },
    {
      id: "row_2", templateName: "GridRow", options: { height: "2fr" },
      children: [
        {
          id: "cell_c", templateName: "ContainersGroup",
          options: { width: "1fr", grid: true },   // вложенная сетка
          children: [innerRow1, innerRow2]
        }
      ]
    }
  ]
}
```

#### Редактирование раскладки (`editMode`)

Включается на **внешнем** grid-узле; вложенные сетки наследуют режим через контекст, их собственный `editMode` игнорируется — сессия редактирования одна на дерево.

| Взаимодействие | Результат |
|---|---|
| Наведение на ячейку | Её границы обозначаются пунктиром `primary`-цвета. Вне `editMode` сетка ничем себя не обозначает; у ячейки-сетки пунктир не рисуется — под курсором всегда лежит какая-то её внутренняя ячейка |
| Наведение на границу треков | Вдоль границы появляется ползунок `primary`-цвета. Есть он и на нижней границе сетки — соседа снизу там нет, поэтому последняя строка растёт вместе с самой сеткой |
| Перетаскивание ползунка | Меняются доли ровно двух смежных треков, их сумма сохраняется — остальные границы не двигаются. Каждому треку пары оставляется минимум `40px`, а в тесной паре — четверть её суммы, чтобы граница не запиралась намертво |
| Клик по ячейке | Выделение одной ячейки. Повторный клик по ней же снимает выделение; клик по одной из нескольких выделенных сначала схлопывает выделение до неё. Клик не всплывает, поэтому во вложенных сетках выделяется самая глубокая ячейка под курсором |
| Shift + клик | Добавить/убрать ячейку из выделения |
| Перетаскивание ячейки | Две ячейки меняются местами. Жест начинается после сдвига курсора на `4px`, поэтому обычный клик по-прежнему выделяет, а нажатие на ручке ресайза тянет границу, а не ячейку. Пока жест идёт, источник приглушается, а ячейка под курсором обводится сплошной рамкой, если обмен с ней возможен; `Escape` и отпускание мимо цели отменяют жест. Нажатие в поле ввода перетаскивание не начинает, а на время жеста выделение текста на странице отключается — в покое текст в ячейках выделяется и копируется как обычно |
| `Escape` | Снять выделение |
| Правая кнопка на ячейке | Контекстное меню операций (невыделенная ячейка сначала выделяется) |

Пункты меню и условия доступности:

| Пункт | Доступен |
|---|---|
| Разделить по вертикали / по горизонтали | Выбрана ровно одна ячейка |
| Добавить строку выше / ниже, ячейку слева / справа | Выбрана ровно одна ячейка |
| Объединить | Выбраны ≥2 **смежные** ячейки одной строки, ни одна из них не является сеткой |
| Удалить | Всегда |

Семантика операций: направление в подписи — это то, **как встанут половинки**, а не куда пройдёт граница между ними. «Разделить по горизонтали» ставит их в ряд: вставляет соседа в ту же строку и делит долю пополам. «По вертикали» ставит их друг под друга: у единственной в строке ячейки делит саму строку, у ячейки с соседями — превращает её во вложенную сетку, унося прежнее содержимое в первую строку. При объединении доли складываются, а содержимое всех выбранных ячеек склеивается в первую — в столбик, как у обычной группы, поэтому объединённая ячейка требует больше высоты, чем каждая из исходных: если строка её не вмещает, ячейка прокручивается. При удалении освободившаяся доля отдаётся левому соседу (у первого трека — правому), опустевшая строка удаляется, а опустевшая корневая сетка восстанавливается до одной пустой ячейки.

Обмен ячеек местами устроен иначе, чем остальные операции: доля трека остаётся за **позицией**, а не едет с ячейкой, поэтому после перестановки границы стоят там же, где стояли, — переезжает только содержимое. Меняться можно с любой ячейкой дерева сессии, включая ячейки соседних строк и вложенных сеток. Запрещён единственный случай — обмен ячейки с её же внутренней ячейкой: ячейка оказалась бы внутри самой себя. Такая цель под курсором не подсвечивается, и отпускание над ней ничего не меняет.

Нижняя граница сетки — особый случай: она меняет `options.height` корневого узла (в пикселях) и одновременно пересчитывает доли всех строк, чтобы верхние остались прежней высоты. Доступна только у корневой сетки сессии: у вложенной снизу лежит трек чужой раскладки, и её высотой распоряжается родитель. В режиме [[containers#Рост под содержимое autoHeight|`autoHeight`]] жест задаёт не высоту, а её минимум — так же ведёт себя и превью под курсором. Опция `options.fixedHeight` убирает эту ручку и у корневой сетки: высотой распоряжается внешняя раскладка (панель фиксированного экрана, док), тянуть там нечего.

После каждой правки вызывается `onChange` контейнера — он приходит из пропа `onContainerChange` провайдера, см. [[setup#Сохранение изменений раскладки|Подключение]]. Во время перетаскивания `onChange` не дёргается: раскладка меняется инлайн-стилем, а конфиг обновляется один раз на отпускание мыши.

#### Интеграционный API для хостов

Вход в сетку один — `ContainersGroup` с `options.grid`. Хост со своим реестром содержимого (редактор дашборда, Storybook) расширяет его двумя пропсами `ContainerProps`; сами компоненты сетки и сессия редактирования наружу из пакета не отдаются.

| Проп | Тип | Назначение |
|---|---|---|
| `createRenderElement` | `(node: ConfigContainerChild) => RenderElementFunction` | Фабрика рендера содержимого по **черновику** сессии. Готовый `renderElement` из пропсов для этого не годится: он замкнут на исходный узел и не найдёт ячейки, созданные `split`/`add`. Без фабрики сессия строит рендер сама — штатным реестром контейнеров ([[hooks\|`useRenderElement`]]) |
| `onGridSelectionChange` | `(cellIds: string[]) => void` | Зеркало выделения наружу: вызывается на каждую смену выделения списком id выделенных ячеек (пустой список — выделения нет). Само выделение живёт в сессии; колбэк ни на что не влияет, он только сообщает |

Публичная поверхность модуля `grid/` (`@evergis/react`) — типы операций, константы, утилиты треков и дерева, фабрика id: см. [[types#Публичная поверхность сетки|Типы]] и [[utils#Утилиты сетки grid|Утилиты]].

---

### DataSourceContainer

**Назначение:** Отображает список объектов из источника данных — проходит по каждой записи (`feature`) и рендерит её через **внутренний шаблон** `options.innerTemplateName` (обёрнутый в `DataSourceInnerContainer`). Используется для карточек списков: объекты недвижимости, инциденты, объекты мониторинга.

**Props:** `ContainerProps` + `innerComponent?: FC<InnerContainerProps>`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | **Обязательный.** Имя источника данных из `dataSources` страницы |
| `innerTemplateName` | `ContainerTemplate` | **Обязательный.** Шаблон рендеринга каждой записи источника — конвертируется в проп `innerComponent`. Значение потребляет пайплайн рендера ([[utils\|`getRenderElement`]] → [[utils\|`getContainerComponent`]]), сам контейнер его в пропсах не видит. Типовые значения: `RoundedBackground`, `Progress`, `OneColumn`, `TwoColumn`, `ContainersGroup` |
| `column` | `boolean` | Располагать элементы в колонку (`true`, default) или строку (`false`) |
| `columns` | `number` | Число плиток в строке (ряд переключается в grid-режим) |
| `gap` | `number` | Отступ между плитками, px. Дефолт `8`; в ряду без `columns` — `0.5rem` |
| `innerGap` | `number` | Отступ между элементами внутри плитки — потребляет внутренний шаблон, см. [[containers#RoundedBackgroundContainer\|RoundedBackgroundContainer]] |
| `align` | `"left" \| "center" \| "right" \| "stretch"` | Положение плиток в ряду/столбце |
| `shownItems` | `number` | Сколько записей показывать до кнопки «Показать все» ([[hooks\|`useShownOtherItems`]]) |
| `otherItems` | `number` | Максимум записей в развёрнутом списке |
| `expandable` | `boolean` | Разрешить сворачивание контейнера |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`).

При `!relatedDataSource` → `null`. При ошибке → `<DataSourceError />`. До загрузки → `<ContainerLoading />`.

> [!info] Ширина плитки в grid-режиме ограничена ячейкой
> При `columns` и выравнивании кроме «Растянуть» ряд — grid с треками `auto`, а ширину плиток уравнивает [[hooks#useEqualTileWidth\|`useEqualTileWidth`]]: измеряет `max-content` самой широкой и отдаёт результат в CSS-переменную `--tile-width`.
>
> Результат **обязательно** ограничивается шириной ячейки — `(clientWidth − gap × (columns − 1)) / columns`. Плитка задаёт ширину в px и в grid не сжимается вместе с треком: `flex-shrink` там не действует, а `min-width: 0` базового `Container` (ветка `isColumn=false`) разрешает треку ужаться ниже min-content. Без ограничения плитки вылезали за свои ячейки и наезжали друг на друга. Вторая линия защиты — `max-width: 100%` у `RoundedBackgroundContainerWrapper`.
>
> При «Растянуть» замер отключён: треки `minmax(0, 1fr)`, плитка `width: 100%`.

> [!warning] Без `innerTemplateName` контейнер визуально пуст
> `getContainerComponent(undefined) === null` → `DataSourceInnerContainer` возвращает `null` → **ни одна запись не рендерится**. Типы это не ловят: поле объявлено в `ConfigMiscOptions` и, хотя и входит в `DataSourceContainerOptions`, остаётся **необязательным** — пропуск компилируется молча. Неизвестное имя шаблона откатывается на `ContainersGroup` (реестровый `default`).
>
> В dev-режиме пропуск ловит рантайм-валидатор `validateDashboardConfig` — ошибкой `missing-inner-template` с путём до узла; он же резолвит заданное имя и проверяет `children` по слотам внутреннего шаблона (см. [[authoring|Правила генерации]]).

**`children` DataSource-хоста — это slot-id выбранного `innerTemplateName`** (например для `RoundedBackground`: `icon`, `alias`, `value`, `units`; для `OneColumn`/`TwoColumn`: `alias`, `value`, `units`), а не собственные слоты `DataSource` и не вложенный контейнер.

```tsx
{
  id: "buildings_list",
  templateName: "DataSource",
  options: { relatedDataSource: "buildings_ds", column: true, innerTemplateName: "RoundedBackground" },
  children: [
    { id: "icon", type: "icon", options: { icon: "building" } },
    { id: "alias", attributeName: "name" },
    { id: "value", attributeName: "floors" }
  ]
}
```

---

### DataSourceInnerContainer

**Назначение:** Рендерит один элемент из источника данных — оборачивает `innerComponent` (внутренний шаблон, полученный из `innerTemplateName`) с атрибутами конкретного `feature`. Используется внутри `DataSourceContainer` и `DataSourceProgressContainer`. Если `innerComponent` не передан (не задан `innerTemplateName`) → возвращает `null`.

**Типы:** собственного `templateName` нет — контейнер не регистрируется в реестре и в конфиге не встречается. `DataSourceInnerContainerOptions` · `DataSourceInnerContainerConfig` · `InnerContainerProps`.

**Props:** `InnerContainerProps` (`type, config, elementConfig, feature, index, maxValue, innerComponent`) — это `ContainerProps & { feature?: FeatureDc }`.

**Опции (`DataSourceInnerContainerOptions`):** читаются из `options` **хоста** (`DataSource`/`DataSourceProgress`) — своего узла в конфиге у этого контейнера нет.

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | Источник, из которого берётся запись: по нему собирается значение записи через [[utils\|`getDataFromRelatedFeatures`]] |
| `filterName` | `string` | Делает запись кликабельной: клик отправляет значение слота `alias` в фильтр ([[hooks\|`useWidgetFilters`]]), а невыбранные записи приглушаются. Не задан — клик не обрабатывается |
| `column` | `boolean` | Раскладка содержимого записи в колонку. Не задана — `true` |

---

### DataSourceProgressContainer

**Назначение:** Список прогресс-баров из источника данных с опциональным итогом. Как и `DataSourceContainer`, рендерит каждую запись через внутренний шаблон `options.innerTemplateName` (обычно `Progress`). Полезен для рейтингов: топ зданий по этажности, топ районов по объёму сделок.

**Props:** `ContainerProps` + `innerComponent`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | **Обязательный.** Имя источника данных |
| `innerTemplateName` | `ContainerTemplate` | **Обязательный.** Шаблон рендеринга каждой записи (обычно `Progress`) — конвертируется в `innerComponent`. Объявлен в `ConfigMiscOptions` (см. [[options\|Опции]]) и необязателен по типу, поэтому пропуск компилируется молча — без него записи не рендерятся |
| `maxValue` | `number \| string` | Максимальное значение для расчёта ширины бара. Если строка — имя атрибута |
| `showTotal` | `boolean` | Показывать итоговую сумму под списком |
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |
| `shownItems` | `number` | Элементов до кнопки «Показать ещё» |
| `otherItems` | `number` | Максимум элементов с «Другое» |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`).

Использует [[hooks|хук]] `useShownOtherItems` для пагинации. Вычисляет `totalValue` и `currentMaxValue` из features. `children` — slot-id внутреннего шаблона (`Progress`: `icon`, `alias`, `value`, `units`).

```tsx
{
  id: "districts_progress",
  templateName: "DataSourceProgress",
  options: { relatedDataSource: "districts_ds", showTotal: true, shownItems: 5, innerTemplateName: "Progress" },
  children: [
    { id: "alias", attributeName: "district_name" },
    { id: "value", attributeName: "deals_count" }
  ]
}
```

---

### DefaultAttributesContainer

**Назначение:** Рендерит все атрибуты объекта по умолчанию без явного конфига (для FeatureCard). Итерирует `attributes` из контекста и рендерит каждый через `TwoColumnContainer`.

**Props:** `ContainerProps`

---

### DividerContainer

**Назначение:** Горизонтальный разделитель между контейнерами.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `bgColor` | `string` | Цвет разделителя (CSS-значение, например `"#e0e0e0"`) |

```tsx
{ id: "section_divider", templateName: "Divider", options: { bgColor: "#e0e0e0" } }
```

---

### EditContainer

**Назначение:** Обёртка для поля редактирования атрибута объекта. Отображает `alias` (подпись) + поле ввода `value`. Используется только внутри FeatureCard в режиме редактирования.

**Props:** `ContainerProps`

**Слоты (`children`):** `alias`, `value`. У **подтипов** `Edit*` набор другой — `alias` и `tooltip`: контрол (инпут, свитч, дропдаун, календарь) встроен в сам контейнер, поэтому слота `value` у них нет. `EditAttachment` рендерит только `alias`, а `EditGroup` клонирует своих детей в разрешённый по типу атрибута `Edit*`-шаблон (плюс `units`/`icon` из [[hooks|`useRenderContainerItem`]]).

**Типы EditContainer:** `templateName = "Edit"` · `EditContainerOptions` (Record<string, never>) · `EditContainerProps`.

#### Подтипы EditContainer

Все подтипы принимают `ContainerProps` и обращаются к контексту FeatureCard для получения/установки значения атрибута. Тип поля задаётся через `templateName`. Большинство подтипов используют только опцию `controls: ConfigControl[]` (соответствующая `<Name>Options`).

| Подтип | `templateName` | `<Name>Options` Pick | Описание |
|---|---|---|---|
| `EditBooleanContainer` | `EditBoolean` | `controls` | Переключатель для boolean-атрибута |
| `EditCheckboxContainer` | `EditCheckbox` | `controls` | Чекбокс с кастомным label |
| `EditChipsContainer` | `EditChips` | `controls` | Мультиселект в виде чипов |
| `EditDateContainer` | `EditDate` | `controls`, `withTime` | Выбор даты/времени через календарь |
| `EditDropdownContainer` | `EditDropdown` | `controls` | Выпадающий список значений |
| `EditGroupContainer` | `EditGroup` | `controls`, `useProjectHiddenAttributes`, `expandable`, `expanded` | Группа edit-полей (использует [[hooks\|`useEditGroupAttributes`]] для фильтрации) |
| `EditNumberContainer` | `EditNumber` | `controls` | Числовой инпут |
| `EditStringContainer` | `EditString` | `controls` | Строковый текстовый инпут |
| `EditAttachmentContainer` | `EditAttachment` | `parentResourceId`, `fileExtensions`, `viewMode`, `shownItems`, `otherItems`, `relatedDataSource`, `controls` | Редактируемый список вложений (см. ниже) |

**Общие опции для полей редактирования:**

| Опция | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, значение которого редактируется |
| `label` | `string` | Подпись поля (если не задан alias) |
| `readOnly` | `boolean` | Только чтение — поле отображается, но недоступно для ввода |
| `required` | `boolean` | Обязательное поле |
| `placeholder` | `string` | Placeholder для текстовых полей |

**Опции EditDropdown / EditChips:**

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | Источник данных для вариантов выбора |
| `items` | `{ text, value }[]` | Статический список вариантов |

```tsx
{
  id: "status_dropdown",
  templateName: "EditDropdown",
  options: { attributeName: "status", relatedDataSource: "statuses_ds" },
  children: [{ id: "alias", value: "Статус" }]
}
```

#### EditAttachmentContainer (детально)

**Назначение:** Редактируемый список вложений — `AttachmentContainer` + кнопки добавления/удаления файлов. Используется внутри FeatureCard в режиме редактирования.

**Опции (`EditAttachmentContainerOptions`):**

| Опция | Тип | Описание |
|---|---|---|
| `parentResourceId` | `string` | Id родительского ресурса для загрузки файлов (см. [[types#Branded types\|ResourceId]]) |
| `fileExtensions` | `string` | Допустимые расширения файлов (например `".pdf,.png,.jpg"`) |
| `viewMode` | `"grid" \| "list"` | Режим отображения коллекции |
| `shownItems` | `number` | Сколько элементов показывать сразу |
| `otherItems` | `number` | Лимит «остальных» |
| `relatedDataSource` | `string` | Источник данных вложений |
| `controls` | `ConfigControl[]` | Маппинг полей источника на `Attachment` |

```tsx
{
  id: "attachments_edit",
  templateName: "EditAttachment",
  options: {
    parentResourceId: "documents_root",
    fileExtensions: ".pdf,.docx",
    viewMode: "list",
    relatedDataSource: "object_attachments"
  }
}
```

---

### ExportPdfContainer

**Назначение:** Кнопка экспорта текущего виджета в PDF-файл. Использует [[hooks|хук]] `useExportPdf` с `getRootElementId(type)`.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `icon` | `IconTypesKeys` | Иконка кнопки (default: `"download"`) |
| `title` | `string` | Подпись кнопки (default: из локализации) |

```tsx
{ id: "pdf_export", templateName: "ExportPdf", options: { icon: "download", title: "Скачать PDF" } }
```

---

### FiltersContainer

**Назначение:** Контейнер фильтров. Динамически рендерит компоненты фильтров через **`getFilterComponent(filterType)`**. Показывает текущие активные фильтры через `HiddenTitleItems`.

**Props:** `ContainerProps`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `padding` | `string \| number` | Внутренние отступы контейнера |
| `bgColor` | `string` | Цвет фона |
| `fontColor` | `string` | Цвет текста |
| `fontSize` | `string \| number` | Размер шрифта |
| `expandable` | `boolean` | Разрешить сворачивание панели фильтров |
| `expanded` | `boolean` | Развёрнута ли панель по умолчанию |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`) — учти, что `overflow` не `visible` при заданной высоте обрежет раскрытый список фильтра.

Фильтрует дочерние элементы по наличию `options.filterName`.

#### Типы фильтров (FilterType)

| Тип | Компонент | Описание |
|---|---|---|
| `checkbox` | `CheckboxFilter` | Список чекбоксов из `relatedDataSource` |
| `rangeNumber` | `RangeNumberFilter` | Числовой диапазон (ползунок или два инпута) |
| `rangeDate` | `RangeDateFilter` | Диапазон дат (два date-picker) |
| `text` | `TextFilter` | Текстовое поле с автодополнением (`AutoComplete`). Режим свободного ввода (значение сохраняется строкой), если нет `variants` и у `searchFilterName`-фильтра не заданы `attributeAlias`/`attributeValue`; иначе — автодополнение по `searchFilterName`/`variants` с infinite scroll (значение — массив) |
| `chips` | `ChipsFilter` | Мультиселект в виде чипов |
| `barChart` | `BarChartFilter` | Интерактивный барчарт — клик по бару выбирает значение |
| `tree` | `TreeFilter` | Иерархический справочник — поэлементный выбор узлов дерева из `relatedDataSource` (см. [[concepts\|Основные понятия]]) |
| `dropdown` | `DropdownFilter` | Выпадающий список (default) |

**Опции дочернего элемента фильтра (`FilterChildOptions`):**

Тип фильтра задаётся полем `type` дочернего элемента (один из **FilterType**), не через опции.

Набор `FilterChildOptions` — объединение того, что реально читают все восемь реализаций фильтра: `dropdown` берёт `variants`/`noEmptyOption`, `text` — `searchFilterName` и `multiSelect`, `chips` — цвета и иконку, `rangeNumber` — `step` и границы, `rangeDate` — `withTime`, `barChart` — геометрию столбцов и палитру.

| Опция | Тип | Описание |
|---|---|---|
| `filterName` | `string` | **Обязательный.** Имя фильтра из `ConfigFilter.name` |
| `searchFilterName` | `string` | Фильтр-источник автодополнения (`text`) |
| `relatedDataSource` | `string` | Источник данных для вариантов фильтра |
| `label` | `string` | Подпись фильтра |
| `placeholder` | `string` | Placeholder |
| `control` | `ConfigControl` | Одиночный маппинг поля источника |
| `controls` | `ConfigControl[]` | Маппинг полей источника |
| `minValue` | `number \| Date` | Минимум диапазона (`rangeNumber`/`rangeDate`) |
| `maxValue` | `number \| Date` | Максимум диапазона (`rangeNumber`/`rangeDate`) |
| `step` | `number` | Шаг ползунка (`rangeNumber`) |
| `withTime` | `boolean` | Включить выбор времени (`rangeDate`) |
| `multiSelect` | `boolean` | Множественный выбор |
| `variants` | `IOption[] \| ChipOption[]` | Статический список вариантов (`dropdown`, `chips`) |
| `noEmptyOption` | `boolean` | Запретить пустой выбор (`dropdown`) |
| `shownItems` | `number` | Сколько вариантов показывать сразу |
| `width` | `CssSize` | Ширина контрола фильтра |
| `height` | `CssSize` | Высота контрола фильтра |
| `align` | `"left" \| "center" \| "right"` | Выравнивание содержимого |
| `maxTextWidth` | `number` | Максимальная ширина текста в px |
| `icon` | `IconTypesKeys` | Иконка (`chips`) |
| `iconAttribute` | `string` | Атрибут, из которого брать иконку |
| `colorAttribute` | `string` | Атрибут, определяющий цвет |
| `fontColor` | `string` | Цвет текста |
| `backgroundColor` | `string` | Цвет фона |
| `barWidth` | `number` | Толщина столбца (`barChart`). Ширина графика **считается**: `столбцы × (barWidth + padding)` |
| `padding` | `number` | Зазор между столбцами (`barChart`) |
| `barHeight` | `number` | Высота столбца (`barChart`) |
| `radius` | `number` | Скругление столбца (`barChart`) |
| `markers` | `BarChartMarker[] \| string` | Отметки на графике (`barChart`) |
| `colors` | `string[]` | Палитра столбцов (`barChart`) |
| `defaultColor` | `string` | Цвет невыбранного столбца (`barChart`) |
| `primaryColor` | `string` | Цвет выбранного столбца (`barChart`) |
| `drawMinMax` | `boolean` | Показать подписи минимума и максимума (`barChart`) |

```tsx
{
  id: "page_filters",
  templateName: "Filters",
  options: { expandable: true, expanded: true },
  children: [
    { id: "floors_filter", type: "rangeNumber", options: { filterName: "floors", label: "Этажность" } },
    { id: "type_filter", type: "chips", options: { filterName: "type", relatedDataSource: "types_ds", label: "Тип" } }
  ]
}
```

> У каждого фильтра в `children` обязательны **оба** поля: `id` (уникальное имя узла в коллекции, как у перечисляемой сущности) и `options.filterName` (ключ фильтра из `ConfigFilter.name` страницы — связывает фильтр с конфигом, по нему работает подстановка в `condition` и сброс). `id` и `filterName` — разные вещи: `id` адресует узел в `children`, `filterName` связывает фильтр со страничным конфигом. См. [[concepts#Перечисляемые сущности в children|подраздел про перечисляемые сущности]].

---

### GridRowContainer

**Назначение:** Строка сетки. Служебный контейнер: появляется только внутри [[containers#ContainersGroupContainer|`ContainersGroup`]] с `options.grid` и самостоятельно не используется.

**Типы:** `templateName = "GridRow"` · `GridRowContainerOptions` · `GridRowContainerProps`. Дети — ячейки (`ContainersGroup`).

**Props:** `ContainerProps`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `gap` | `number` | Зазор между ячейками строки, px. Дефолт `0` — ячейки стыкуются вплотную |
| `alignItems` | `AlignItems` | Наследуется от базового `Container`; в сетке треки и так растягиваются |
| `autoHeight` | `boolean` | Разрешает **ячейкам** растить строку своим содержимым — см. [[containers#Рост под содержимое autoHeight\|Рост под содержимое]]. Родительская сетка увидит рост, только если включила `autoHeight` у себя |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`). **`height` строки читает не она сама, а родительская сетка** при сборке `grid-template-rows`; строке достаётся уже готовая высота трека.

Заголовка у строки нет — это структурный узел, весь контент лежит в ячейках. Слоты `title` / `icon` / `titleIcon` в строке игнорируются.

```tsx
{
  id: "row_1",
  templateName: "GridRow",
  options: { height: "1fr" },
  children: [
    { id: "cell_a", templateName: "ContainersGroup", options: { width: "2fr" }, children: [...] },
    { id: "cell_b", templateName: "ContainersGroup", options: { width: "1fr" }, children: [...] }
  ]
}
```

---

### IconContainer

**Назначение:** Блок с иконкой, заголовком, описанием и ссылкой. Используется для виджетов-карточек типа «быстрых ссылок».

**Props:** `ContainerProps` (дочерние элементы по id: `icon`, `alias`, `link`, `text`)

---

### ImageContainer

**Назначение:** Блок с фоновым изображением, заголовком, текстом и кнопкой действия.

**Props:** `ContainerProps` (дочерние по id: `alias`, `text`, `button`, `image`)

**Опции:** собственных нет — только `ContainerBoxOptions` (`width`, `height`, `overflow`).

---

### LayersContainer

**Назначение:** Дерево слоёв карты текущей страницы. Позволяет пользователю управлять видимостью слоёв. Рендерит `<LayerTree />`.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `layerNames` | `string[]` | Фильтр — показывать только указанные слои |
| `expandable` | `boolean` | Разрешить сворачивание панели слоёв |
| `expanded` | `boolean` | Развёрнута ли панель по умолчанию |

```tsx
{ id: "map_layers", templateName: "Layers", options: { layerNames: ["buildings", "roads"], expandable: true } }
```

---

### OneColumnContainer

**Назначение:** Одноколоночный блок: подпись (`alias`) + значение (`value`) + единицы измерения (`units`). Поддерживает `options.attributes` для рендеринга нескольких атрибутов в одну колонку.

**Props:** `ContainerProps`

**Слоты (`children`):** `alias`, `value`, `units`, а также `tooltip` и `modal` — они рендерятся рядом с подписью (`renderElement({ id: "tooltip" })` / `{ id: "modal" }`).

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributes` | `string[]` | Список имён атрибутов для отображения. Пустой массив = все атрибуты слоя |
| `useProjectHiddenAttributes` | `boolean` | Если `true` — список `attributes` фильтруется по `hiddenAttributes` слоя проекта (хук [[hooks\|`useLayerHiddenAttributes`]]). По умолчанию `false` — атрибуты, скрытые в проекте, всё равно отображаются |
| `hideEmpty` | `boolean` | Скрыть пустые атрибуты |
| `innerTemplateStyle` | `object` | Кастомные CSS-стили внутренних элементов |

```tsx
{
  id: "area_attr",
  templateName: "OneColumn",
  children: [
    { id: "alias", value: "Площадь" },
    { id: "value", attributeName: "area" },
    { id: "units", value: "м²" }
  ]
}
```

---

### PagesContainer

**Назначение:** Корневой контейнер страниц дашборда. Фильтрует дочерние страницы по `selectedTabId` и рендерит через `ContainerChildren`.

**Props:** `ContainerProps` + `noBorders?: boolean`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `column` | `boolean` | Расположение страниц в колонку |
| `width` | `string \| number` | Ширина контейнера страниц |

---

### ProgressContainer

**Назначение:** Прогресс-бар с тултипом. Вычисляет ширину полосы по `value / maxValue * 100%`. Используется как **внутренний шаблон** `DataSourceProgressContainer` — подключается через `options.innerTemplateName: "Progress"` на хосте (не как обычный вложенный контейнер).

**Props:** `InnerContainerProps` (только как внутренний шаблон `DataSourceProgressContainer`)

**Слоты (`children` хоста):** `icon`, `alias`, `value`, `units`.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `hideTitle` | `boolean` | Скрыть подпись над прогресс-баром |
| `innerValue` | `string` | Имя атрибута для значения внутри бара |
| `bgColor` | `string` | Цвет фона бара |
| `colors` | `string[]` | Массив цветов для разных значений |
| `colorAttribute` | `string` | Имя атрибута, определяющего цвет бара |
| `maxValue` | `number` | Максимальное значение (100%) |
| `innerTemplateStyle` | `object` | Кастомные CSS-стили для внутренних элементов |

---

### RoundedBackgroundContainer

**Назначение:** Компактный блок с округлым фоновым блоком: иконка + значение + подпись. Используется как **внутренний шаблон** `DataSourceContainer` для списков — подключается через `options.innerTemplateName: "RoundedBackground"` на хосте.

**Props:** `InnerContainerProps`

**Слоты (`children` хоста):** `icon`, `alias`, `value`, `units`.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `center` | `boolean` | Центрировать содержимое |
| `fontColor` | `string` | Цвет текста; он же — источник тонированного фона, если не заданы `bgColor` и цвет иконки |
| `bgColor` | `string` | Свой цвет фона плитки. Кладётся **сплошным** и перебивает наследование от иконки/текста |
| `colorAttribute` | `string` | Имя атрибута, значение которого заменяет `fontColor` (цвет текста, а с ним и тонированный фон) |
| `big` | `boolean` | Увеличенный размер блока |
| `bigIcon` | `boolean` | Иконка-водяной знак: уходит из потока в правый верхний угол плитки, `3rem`, прозрачность `0.12` |
| `inlineUnits` | `boolean` | Отображать единицы рядом со значением (а не под ним) |
| `hideEmpty` | `boolean` | Скрыть блок при пустом значении |
| `maxLength` | `number` | Максимальная длина подписи в символах (обрезка с многоточием). Дефолт — `28` |
| `maxLines` | `number` | Обрезка подписи по числу строк (CSS line-clamp) — перебивает `maxLength`. Не задан: в плиточном ряду с `columns` действует дефолт `2`, иначе обрезки по строкам нет. Тултип с полным текстом показывается только при реальном переполнении |
| `wordBreak` | `"break-word" \| "break-all"` | Стратегия переноса длинного текста |
| `innerGap` | `number` | Отступ между элементами внутри плитки, px: иконка ↔ значение ↔ подпись и значение ↔ единицы измерения. Не путать с [[containers#DataSourceContainer\|`gap`]] — тот задаёт расстояние **между плитками** |
| `innerTemplateStyle` | `CSSProperties` | Стили самой плитки. Заданы — перебивают авторский `style` узла целиком, а не мержатся с ним |

Опции `align`, `columns` и `gap` тоже входят в `RoundedBackgroundContainerOptions`, но отвечают за раскладку **ряда плиток**, а не одной плитки — описаны у [[containers#DataSourceContainer|DataSourceContainer]].

> [!info] Опции читаются из `options` хоста
> Плитка получает `elementConfig` контейнера `DataSource`, поэтому `innerGap` (как и `big`, `inlineUnits`, `hideEmpty`) указывается в `options` **хоста**, а не отдельного узла `RoundedBackground`.
>
> Если `innerGap` не задан, действуют исторические значения: `0.25rem` над подписью и между значением и единицами в строку, `0.5rem` в режиме `big` (между блоком иконки и значением), а между иконкой и значением в колоночной плитке и под значением у единиц зазора нет вовсе. Поэтому добавление опции не меняет вид существующих конфигов.
>
> Вертикальный зазор под иконкой задаётся на её обёртке в `RoundedBackgroundContainer`, а не в стилях плитки: элемент `icon` рендерится через `ElementIcon`/`ElementSvg` со своими styled-компонентами, и селекторы на `ContainerIcon`/`SvgContainer` из `containers/styled.ts` до него не достают.

> [!info] Приоритет цвета фона плитки
> Отдельной опции «тип цвета фона» нет — фон выводится по приоритету:
>
> 1. `options.bgColor` хоста — сплошная заливка выбранным цветом;
> 2. цвет узла `icon` (`options.fontColor`, фолбэк `style.color`) — тонировка 6%;
> 3. `colorAttribute`/`fontColor` (цвет текста) — тонировка 6%;
> 4. ничего не задано — токен темы `palette.element`.
>
> Цвет иконки влияет **только** на фон: сам глиф красят `ElementIcon`/`ElementSvg` своим `fontColor`, а текст плитки — `colorAttribute`/`fontColor` хоста. Отсюда следствие: у плитки с цветной иконкой фон тонируется её цветом, даже если цвет текста задан.
>
> Редактор дашбордов (client-new, `useTilesBackground`) показывает это как выпадающий список «Цвет фона»: «Как в иконке» / «Как в тексте» (что из них — зависит от того, какой цвет задан) и «Свой» с пикером, пишущим `bgColor`.

```tsx
{
  id: "status_card",
  templateName: "RoundedBackground",
  options: { big: true, colorAttribute: "status_color", inlineUnits: true, innerGap: 6 }
}
```

---

### SlideshowContainer

**Назначение:** Контейнер для слайдшоу изображений, связанных с атрибутом объекта или источником данных.

**Props:** `ContainerProps`

**Слоты (`children`):** `slideshow` (элемент `type: "slideshow"`) и опциональный `alias` — подпись над галереей. Без `alias` обёртка подписи не рендерится вовсе: пустой блок отъедал бы половину ширины у галереи.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`).

---

### StructuredDataContainer

**Назначение:** Таблица структурированных данных со схемой в конфиге. С `editMode` строки правятся, добавляются и удаляются прямо в дашборде, а результат уходит потребителям через фильтр значением `valueType: "features"` (FeatureCollection); без него таблица работает как представление данных только на чтение. Режимов данных два: с источником (строки грузятся из него, «Сохранить» пишет и в источник, и в фильтр) и без источника (пустая структура набирается руками, «Сохранить» пишет только в фильтр).

**Типы:** `templateName = "StructuredData"` · `StructuredDataContainerOptions` · `StructuredDataContainerProps`. Дети — `StructuredDataContainerChild`.

**Props:** `ContainerProps`

**Слоты (`children`):** `data` — представление структуры, элемент `type: "table"` (см. [[elements#ElementTable|ElementTable]]); `alias`; `title` / `titleIcon` — заголовок контейнера. Слот `data` обязателен: без него данные негде показать.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributesDescription` | `ConfigAttributeDescription[]` | Схема данных, она же набор колонок таблицы: что описано, то и показано, в порядке описания. С источником — переопределение его схемы: задаёт состав, порядок, `alias`, `stringFormat`, `isEditable`; типы приходят от источника. Без источника — единственная схема, обязательна |
| `relatedDataSource` | `string` | Имя источника данных. Не задан — структура набирается вручную |
| `filterName` | `string` | Имя фильтра-приёмника. **Обязателен при `editMode`** — иначе правкам некуда уходить. Фильтр должен быть объявлен на странице с `valueType: "features"` |
| `editMode` | `boolean` | Разрешить правку таблицы: редакторы в ячейках, добавление и удаление строк, панель отката и записи. Не задан или `false` — таблица только на чтение. Та же опция, что у сетки [[containers#ContainersGroupContainer\|ContainersGroup]], но читает её этот контейнер сам |
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`).

**Поля `ConfigAttributeDescription`:** `attributeName` (обязателен), `type`, `alias`, `description`, `isEditable`, `stringFormat`. Форма повторяет `AttributeConfigurationDc` из `@evergis/api`.

**Панель действий.** Слева — общая кнопка «Добавить»: серая скруглённая, со значком `feature_add` и подписью, та же, что добавляет файл во [[containers#AttachmentContainer|вложениях]] (`AddButtonRow` из `Dashboard/components/AddButton`). Размеры сняты с макета: высота `1.5rem`, поля `0.625rem`, просвет до подписи `0.375rem`, значок и подпись по `0.875rem`; фон `elementDark`, значок цветом иконки, подпись — `textPrimary` (сам `IconButton` красит подпись заодно со значком, поэтому цвет задан отдельно). Справа, и только пока черновик грязный, — пара `IconButton`: красный крест отката (`kind="close"` с пропом `error`) и галка записи основным цветом (`kind="success"` с пропом `primary`). Та же пара правит объект в карточке (`EditControls`), поэтому подписи ей не нужны — они ушли в `title`. Всем трём кнопкам `tabIndex` задан явно: `IconButton` по умолчанию ставит -1, а по таблице ходят Tab-ом, и с ячеек он обязан доходить до панели.

**Черновик и сохранение.** Правки копятся в локальном черновике: ни источник, ни потребители фильтра не узнают о них до нажатия «Сохранить», «Отменить» возвращает черновик к последнему сохранённому состоянию (прецедент — dirty/onSave у [[containers#FiltersContainer|tree-фильтра]]). Порядок записи: `deleteFeatures` → `updateFeature` → `createFeatures`, затем `changeFilters`. Созданные строки получают `id` из `createdIds` ответа — повторное сохранение обновляет их, а не плодит дубли.

**Что уходит в features-API.** Только атрибуты с `isEditable: true`. Нередактируемые поля не передаются вовсе — их значение принадлежит источнику, а у новой строки его и нет. У источника со слоем конфиг может правку запретить, но не разрешить там, где источник её не допускает (id-атрибут, вычисляемые поля).

**Начальное наполнение без источника.** Ручная структура (без `relatedDataSource`) гидратируется из фильтра-приёмника один раз при монтировании: сначала из выбранного значения фильтра, а пока пользователь ничего не сохранил — из `defaultValue` этого фильтра в конфиге страницы. Общего стейта, залитого дефолтами, в дашборде нет: `defaultValue` живёт только в конфиге, и фолбэк на него делает каждый потребитель фильтра сам. Так таблица получает наполнение по умолчанию, а после первого «Сохранить» его заменяет сохранённое значение. Грязный черновик гидратация не затирает, и повторно она не срабатывает — иначе пересоздались бы ключи уже показанных строк. У таблицы с источником этот путь не работает вовсе: строки приходят из источника, а фильтр служит только выходом.

**Что уходит в фильтр.** FeatureCollection, где `features[].properties` — строка ровно по схеме (объявленный атрибут уходит всегда, даже незаполненным; лишние ключи черновика отбрасываются), `geometry: null` (геометрия не поддерживается), а порядок строк — исходный, без учёта сортировки представления.

**Размеры и прокрутка.** Размеров два набора, и они про разное:

- **`width`/`height` контейнера** (`ContainerBoxOptions`) отмеряют место ВСЕМУ контейнеру — заголовку, таблице и панели кнопок вместе. Заданная высота включает fill-режим у тела: область представления сжимается и прокручивает строки внутри себя. `height: "100%"` — то же самое там, где высоту выдаёт раскладка (трек сетки): без него корень контейнера перерастает трек, и в прокрутку уезжает весь контейнер целиком, вместе с заголовком.
- **`width`/`height` у ребёнка `data`** ограничивают САМУ ТАБЛИЦУ: она получает собственный скролл-бокс, а контейнер остаётся ровно таким, каким его сделала таблица. Заданная ширина вдобавок ставит колонки по содержимому и снимает предел ячейки в `20rem`; высота принимает только конкретные единицы — процент в блочной обёртке элемента не разрешается. Подробности — в [[elements#ElementTable|ElementTable]].

Отдельных опций прокрутки нет и не нужно: скролл появляется сам, как только у бокса есть определённый размер — что у контейнера, что у таблицы.

**Режим только на чтение.** Без `editMode` контейнер показывает значения и ничего больше: ячейки — форматированный текст (логические — неактивная галка), колонки удаления и панели с кнопками нет, черновик не может стать грязным, а значит и в фильтр ничего не уйдёт. Такой таблице `filterName` не нужен.

**Источник без записи.** Вид источника решает не *что* можно править, а *куда* уедет результат. Запись через features-API возможна только в источник со слоем (`layerName`); у EQL-, python- и url-источников её нет — «Сохранить» пишет только значение фильтра. Права правки при этом те же: ячейки, добавление и удаление строк. Ограничения источника на `isEditable` (нередактируемый атрибут, вычисляемое поле, id-атрибут) там тоже не действуют — они про запись, которой не будет, поэтому запретить колонку может только `attributesDescription`. Удаление строки такой таблицы убирает её лишь из значения фильтра: следующая загрузка источника вернёт её обратно.

```tsx
// в filters страницы
{ name: "type_1", valueType: "features" }

// контейнер
{
  id: "Table-1",
  templateName: "StructuredData",
  options: {
    relatedDataSource: "source-data",
    filterName: "type_1",
    editMode: true,
    attributesDescription: [
      { type: "Int64", attributeName: "number", alias: "Номер", isEditable: false },
      { type: "String", attributeName: "name", alias: "Название", isEditable: true },
      {
        type: "Double",
        attributeName: "value",
        alias: "Значение",
        isEditable: true,
        stringFormat: { culture: "ru-RU", rounding: 8, unitsLabel: "руб.", scalingFactor: 1, splitDigitGroup: true }
      }
    ]
  },
  children: [
    { id: "data", type: "table", options: { sort: true } }
  ]
}
```

---

### TabsContainer

**Назначение:** Горизонтальные вкладки (Swiper). Каждая вкладка — дочерний элемент с `id`. Управляет `selectedTabId` через контекст — при клике на вкладку показывается соответствующая страница/контейнер.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `radius` | `number` | Радиус скругления вкладок |
| `column` | `boolean` | Вертикальные вкладки вместо горизонтальных |
| `bgColor` | `string` | Цвет фона панели вкладок |
| `noBg` | `boolean` | Прозрачный фон (без подложки) |
| `onlyIcon` | `boolean` | Показывать только иконку без текста |
| `shownItems` | `number` | Количество видимых вкладок (остальные — в overflow) |
| `maxLength` | `number` | Максимальная длина текста вкладки |
| `wordBreak` | `string` | CSS `word-break` для текста вкладок |

**Опции дочерних элементов (`TabOptions`):**

| Опция | Тип | Описание |
|---|---|---|
| `icon` | `IconTypesKeys` | Иконка вкладки. При `onlyIcon` у контейнера подпись не рендерится и остаётся только она |

Дочерний узел вкладки — `TabChild`: `id` (уникальное имя, на которое ссылается `options.tabId` [[containers#PagesContainer|страницы]]), `value` (подпись) и `options.icon`. Своих `children` у вкладки нет.

`{ id: "tab_1", value: "Общая информация", options: { icon: "info" } }`

```tsx
{
  id: "main_tabs",
  templateName: "Tabs",
  options: { noBg: false, onlyIcon: false },
  children: [
    { id: "tab_info", value: "Информация", options: { icon: "info" } },
    { id: "tab_docs", value: "Документы", options: { icon: "attachment" } }
  ]
}
```

---

### TaskContainer

**Назначение:** Кнопка запуска/остановки Python-задачи через `usePythonTask`. Показывает лог выполнения задачи в реальном времени.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `title` | `string` | Текст кнопки запуска |
| `relatedResources` | `ConfigRelatedResource[]` | Список Python-ресурсов задачи |
| `center` | `boolean` | Центрировать кнопку |
| `icon` | `IconTypesKeys` | Иконка кнопки |
| `statusColors` | `Record<string, string>` | Цвет по статусу задачи (`{ "running": "#f39c12", "done": "#27ae60" }`) |
| `responseFilters` | `Record<string, string>` | Маппинг полей ответа задачи на фильтры |
| `useNotifications` | `boolean` | Показывать прогресс-уведомления о выполнении задачи |

```tsx
{
  id: "python_run",
  templateName: "Task",
  options: {
    title: "Запустить расчёт",
    relatedResources: [{ resourceId: "calc_script_id" }],
    statusColors: { "running": "#f39c12", "done": "#27ae60", "error": "#e74c3c" }
  }
}
```

---

### TitleContainer

**Назначение:** Заголовок контейнера с поддержкой collapse (LegendToggler) и кнопкой управления видимостью слоёв.

**Props:** `ContainerProps` + `containerId?`, `templateName?`, `layerNames?`, `fontColor?`, `expandable?`, `expanded?`, `isVisible?`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `simple` | `boolean` | Простой заголовок без кнопок управления |
| `downloadById` | `string` | ID элемента `ExportPdf` — добавляет кнопку скачивания рядом с заголовком |
| `align` | `"left" \| "center" \| "right"` | Горизонтальное выравнивание содержимого заголовка |

`fontColor` задаётся не опцией, а одноимённым prop контейнера (см. Props выше).

```tsx
{ id: "section_title", templateName: "Title", options: { downloadById: "pdf_export" } }
```

---

### TwoColumnContainer

**Назначение:** Двухколоночный блок: подпись (`alias`) слева + значение (`value`) справа. Основной контейнер для отображения атрибутов объекта. Поддерживает `options.attributes`.

**Props:** `ContainerProps`

**Слоты (`children`):** `alias`, `value`, `units`, `icon`, `tooltip`, `modal`. Слот `icon` рендерится перед подписью, и его тип со значением подставляются из настроек атрибута в слое (`attributesConfiguration.attributes[].icon`) через [[hooks|`useRenderContainerItem`]] + [[utils|`getAttributeIconElement`]]; `tooltip` и `modal` встают рядом с подписью.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributes` | `string[]` | Список имён атрибутов для отображения. Пустой массив = все атрибуты слоя |
| `useProjectHiddenAttributes` | `boolean` | Если `true` — список `attributes` фильтруется по `hiddenAttributes` слоя проекта (хук [[hooks\|`useLayerHiddenAttributes`]]). По умолчанию `false` — атрибуты, скрытые в проекте, всё равно отображаются |
| `hideEmpty` | `boolean` | Скрыть строки с пустым значением |
| `innerTemplateStyle` | `object` | Кастомные CSS-стили внутренних элементов |

```tsx
{
  id: "type_attr",
  templateName: "TwoColumn",
  options: { hideEmpty: true },
  children: [
    { id: "alias", value: "Тип объекта" },
    { id: "value", attributeName: "object_type" }
  ]
}
```

---

### UploadContainer

**Назначение:** Обёртка для элемента загрузки файлов.

**Props:** `ContainerProps`

**Слоты (`children`):** `uploader` — элемент `type: "uploader"` (см. [[elements#ElementUploader|ElementUploader]]).

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

Плюс `ContainerBoxOptions` (`width`, `height`, `overflow`).

---

## Связанные разделы

[[elements|Элементы]] | [[concepts|Основные понятия]] | [[hooks|Хуки]] | [[components|Компоненты]] | [[options|Опции]] | [[types|Типы]]
