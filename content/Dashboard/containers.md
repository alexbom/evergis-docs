# Контейнеры

## Обзор

Контейнеры — компоненты верхнего уровня, управляющие расположением и логикой отображения контента. Регистрируются в `containers/registry.ts` (типобезопасно через `as const satisfies ContainerComponentRegistry`) по значению `ContainerTemplate`. Рендерятся динамически через [[utils|утилиту]] `getContainerComponent`. Все принимают `ContainerProps`.

Каждый контейнер имеет тройку типов `<Name>ContainerOptions` / `<Name>ContainerConfig` / `<Name>ContainerProps` (см. [[types#Контейнеры|сводную таблицу]]). В карточках ниже указан литерал `templateName` и поля `<Name>Options` — какие опции из [[options|`ConfigOptions`]] контейнер реально читает.

**Каждый контейнер обязан иметь поле `id`** — уникальное имя в keyspace `ConfigContainer.id` (типы `ContainerId` / `ChartId` / `ModalId` / `TabId`). Через него работают навигация, связи (`tabId`, `modalId`, `chartId`, `downloadById`), управление состоянием (`expandedContainers`, `selectedTabId`). Подробности и таблица slot-id для `children` — в [[concepts#ID контейнеров и элементов|разделе про id]]. В примерах ниже `id` контейнеров — иллюстративные доменные имена; в реальном конфиге они должны быть уникальны.

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
  children: [{
    type: "button",
    options: { icon: "feature_add", title: "Добавить здание", layerName: "buildings", geometryType: "Polygon" }
  }]
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

Без `relatedDataSource` источник вложений — значение `attributeName` дочернего элемента с `id: "value"` (через `parseAttachments`).

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

**Props:** `ContainerProps` (использует `elementConfig.children` с id `alias`, `chart`, `legend`)

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `twoColumns` | `boolean` | Разместить легенду и график в две колонки |
| `hideEmpty` | `boolean` | Скрыть контейнер при отсутствии данных |

При ошибке источника данных рендерит `<DataSourceError />`.

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
| `column` | `boolean` | `true` — дочерние контейнеры в колонку, `false` — в строку |
| `expandable` | `boolean` | Разрешить сворачивание группы |
| `expanded` | `boolean` | Развёрнута ли группа по умолчанию |

Рендерит дочерние элементы через `ContainerChildren`. Если `id` начинается с `"page_"` — рендерится как корневой контейнер страницы.

```tsx
{
  id: "main_group",
  templateName: "ContainersGroup",
  options: { column: false },
  children: [leftPanel, rightPanel]
}
```

---

### DataSourceContainer

**Назначение:** Отображает список объектов из источника данных — рендерит каждый `feature` через `DataSourceInnerContainer`. Используется для карточек списков: объекты недвижимости, инциденты, объекты мониторинга.

**Props:** `ContainerProps` + `innerComponent?: FC<InnerContainerProps>`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | **Обязательный.** Имя источника данных из `dataSources` страницы |
| `column` | `boolean` | Располагать элементы в колонку (`true`) или строку (`false`) |
| `expandable` | `boolean` | Разрешить сворачивание контейнера |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |
| `shownItems` | `number` | Количество отображаемых элементов до кнопки «Показать ещё» |
| `otherItems` | `number` | Максимум элементов вместе с «Другое» |

При `!relatedDataSource` → `null`. При ошибке → `<DataSourceError />`. До загрузки → `<ContainerLoading />`.

```tsx
{
  id: "buildings_list",
  templateName: "DataSource",
  options: { relatedDataSource: "buildings_ds", shownItems: 5 },
  children: [{ id: "status_card", templateName: "RoundedBackground", ... }]
}
```

---

### DataSourceInnerContainer

**Назначение:** Рендерит один элемент из источника данных — оборачивает контейнер с атрибутами конкретного `feature`. Используется внутри `DataSourceContainer` и `DataSourceProgressContainer`.

**Props:** `InnerContainerProps` (`type, config, elementConfig, feature, index, maxValue, innerComponent`)

---

### DataSourceProgressContainer

**Назначение:** Список прогресс-баров из источника данных с опциональным итогом. Полезен для рейтингов: топ зданий по этажности, топ районов по объёму сделок.

**Props:** `ContainerProps` + `innerComponent`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | **Обязательный.** Имя источника данных |
| `maxValue` | `number \| string` | Максимальное значение для расчёта ширины бара. Если строка — имя атрибута |
| `showTotal` | `boolean` | Показывать итоговую сумму под списком |
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |
| `shownItems` | `number` | Элементов до кнопки «Показать ещё» |
| `otherItems` | `number` | Максимум элементов с «Другое» |

Использует [[hooks|хук]] `useShownOtherItems` для пагинации. Вычисляет `totalValue` и `currentMaxValue` из features.

```tsx
{
  id: "districts_progress",
  templateName: "DataSourceProgress",
  options: { relatedDataSource: "districts_ds", showTotal: true, shownItems: 5 }
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

Фильтрует дочерние элементы по наличию `options.filterName`.

#### Типы фильтров (FilterType)

| Тип | Компонент | Описание |
|---|---|---|
| `checkbox` | `CheckboxFilter` | Список чекбоксов из `relatedDataSource` |
| `rangeNumber` | `RangeNumberFilter` | Числовой диапазон (ползунок или два инпута) |
| `rangeDate` | `RangeDateFilter` | Диапазон дат (два date-picker) |
| `text` | `TextFilter` | Текстовый поиск с debounce |
| `chips` | `ChipsFilter` | Мультиселект в виде чипов |
| `barChart` | `BarChartFilter` | Интерактивный барчарт — клик по бару выбирает значение |
| `dropdown` | `DropdownFilter` | Выпадающий список (default) |

**Опции дочернего элемента фильтра:**

| Опция | Тип | Описание |
|---|---|---|
| `filterName` | `string` | **Обязательный.** Имя фильтра из `ConfigFilter.name` |
| `filterType` | `FilterType` | Тип компонента фильтра |
| `relatedDataSource` | `string` | Источник данных для вариантов фильтра |
| `label` | `string` | Подпись фильтра |
| `placeholder` | `string` | Placeholder |
| `showSearch` | `boolean` | Показать поле поиска внутри фильтра |
| `multiSelect` | `boolean` | Разрешить множественный выбор (для dropdown) |

```tsx
{
  id: "page_filters",
  templateName: "Filters",
  options: { expandable: true, expanded: true },
  children: [
    { options: { filterName: "floors", filterType: "rangeNumber", label: "Этажность" } },
    { options: { filterName: "type", filterType: "chips", relatedDataSource: "types_ds", label: "Тип" } }
  ]
}
```

> **Исключение:** дочерние элементы `FiltersContainer` идентифицируются **не по `id`**, а по обязательному `options.filterName` — поэтому `id` у них не нужен. Это единственный контейнер с таким поведением (см. [[concepts#ID контейнеров и элементов|про id]]).

---

### IconContainer

**Назначение:** Блок с иконкой, заголовком, описанием и ссылкой. Используется для виджетов-карточек типа «быстрых ссылок».

**Props:** `ContainerProps` (дочерние элементы по id: `icon`, `alias`, `link`, `text`)

---

### ImageContainer

**Назначение:** Блок с фоновым изображением, заголовком, текстом и кнопкой действия.

**Props:** `ContainerProps` (дочерние по id: `alias`, `text`, `button`, `image`)

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

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributes` | `string[]` | Список имён атрибутов для отображения. Пустой массив = все атрибуты слоя |
| `useProjectHiddenAttributes` | `boolean` | Если `true` — список `attributes` фильтруется по `hiddenAttributes` слоя проекта (хук [[hooks\|`useLayerHiddenAttributes`]]). По умолчанию `false` — атрибуты, скрытые в проекте, всё равно отображаются |
| `hideEmpty` | `boolean` | Скрыть пустые атрибуты |
| `fontSize` | `string \| number` | Размер шрифта значения |
| `fontColor` | `string` | Цвет шрифта значения |

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

**Назначение:** Прогресс-бар с тултипом. Вычисляет ширину полосы по `value / maxValue * 100%`. Используется внутри `DataSourceProgressContainer`.

**Props:** `InnerContainerProps` (только как дочерний `DataSourceProgressContainer`)

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

**Назначение:** Компактный блок с округлым фоновым блоком: иконка + значение + подпись. Используется внутри `DataSourceContainer` для списков.

**Props:** `InnerContainerProps`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `center` | `boolean` | Центрировать содержимое |
| `fontColor` | `string` | Цвет текста |
| `colorAttribute` | `string` | Имя атрибута, определяющего цвет фона |
| `big` | `boolean` | Увеличенный размер блока |
| `bigIcon` | `boolean` | Увеличенная иконка |
| `inlineUnits` | `boolean` | Отображать единицы рядом со значением (а не под ним) |
| `hideEmpty` | `boolean` | Скрыть блок при пустом значении |
| `maxLength` | `number` | Максимальная длина текста (обрезка с многоточием) |

```tsx
{
  id: "status_card",
  templateName: "RoundedBackground",
  options: { big: true, colorAttribute: "status_color", inlineUnits: true }
}
```

---

### SlideshowContainer

**Назначение:** Контейнер для слайдшоу изображений, связанных с атрибутом объекта или источником данных.

**Props:** `ContainerProps`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

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

Дочерние элементы — вкладки: `{ id: "tab_1", value: "Общая информация", options: { icon: "info" } }`.

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
| `fontColor` | `string` | Цвет текста заголовка |

```tsx
{ id: "section_title", templateName: "Title", options: { downloadById: "pdf_export", fontColor: "#2c3e50" } }
```

---

### TwoColumnContainer

**Назначение:** Двухколоночный блок: подпись (`alias`) слева + значение (`value`) справа. Основной контейнер для отображения атрибутов объекта. Поддерживает `options.attributes`.

**Props:** `ContainerProps`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributes` | `string[]` | Список имён атрибутов для отображения. Пустой массив = все атрибуты слоя |
| `useProjectHiddenAttributes` | `boolean` | Если `true` — список `attributes` фильтруется по `hiddenAttributes` слоя проекта (хук [[hooks\|`useLayerHiddenAttributes`]]). По умолчанию `false` — атрибуты, скрытые в проекте, всё равно отображаются |
| `hideEmpty` | `boolean` | Скрыть строки с пустым значением |
| `fontColor` | `string` | Цвет значений |
| `fontSize` | `string \| number` | Размер шрифта значений |

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

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

---

## Связанные разделы

[[elements|Элементы]] | [[concepts|Основные понятия]] | [[hooks|Хуки]] | [[components|Компоненты]] | [[options|Опции]] | [[types|Типы]]
