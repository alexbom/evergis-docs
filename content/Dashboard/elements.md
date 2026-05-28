# Элементы

## Обзор

Элементы — «листовые» компоненты, отображающие отдельные значения внутри контейнеров. Регистрируются в `elements/registry.ts` (типобезопасно через `as const satisfies ElementComponentRegistry`) по строковому ключу (`type`). Рендерятся через [[utils|утилиту]] `getRenderElement`.

Все принимают `ContainerProps` (`type`, `elementConfig`, `renderElement`). Каждый элемент имеет тройку типов `<Name>Options` / `<Name>Config` / `<Name>Props` (см. [[types#Элементы|сводную таблицу]]). В карточках ниже указан литерал `type` и поля, реально читаемые из `<Name>Options`.

**Каждый элемент обязан иметь поле `id`** — это **slot** (зарезервированное место) внутри parent-контейнера. Контейнер рендерит конкретный slot через `renderElement({ id: "<slot>" })` и ожидает фиксированные ключи (`alias`, `value`, `chart`, `legend`, `title`, `description`, `bgImage`, `icon`, `units`, ...). Если `id` элемента не совпадает с ожидаемым slot-id контейнера — элемент не отрисуется. Таблица slot-id по контейнерам и шапкам — в [[concepts#ID контейнеров и элементов|разделе про id]]. В примерах ниже slot-id выбран под типовой контейнер-родитель.

Список `type`-литералов и их `<Name>` (для совместимости с legacy — некоторые типы носят историческое имя):

| `type` | `<Name>` |
|---|---|
| `button` | `ElementButton` |
| `camera` | `ElementCamera` |
| `chart` | `ElementChart` |
| `tags` | `ElementChips` (исторически `chips`) |
| `control` | `ElementControl` |
| `icon` | `ElementIcon` |
| `image` | `ElementImage` |
| `legend` | `ElementLegend` |
| `link` | `ElementLink` |
| `markdown` | `ElementMarkdown` |
| `modal` | `ElementModal` |
| `slideshow` | `ElementSlideshow` |
| `svg` | `ElementSvg` |
| `tooltip` | `ElementTooltip` |
| `uploader` | `ElementUploader` |

---

## ElementButton

**Назначение:** Кнопка, открывающая URL из атрибута объекта в новой вкладке.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `value` | `string` | Текст кнопки |
| `attributeName` | `string` | Имя атрибута, содержащего URL для открытия |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `icon` | `IconTypesKeys` | Иконка кнопки |

**Поведение:** читает `attributeName` → `attribute.value` (URL) → `window.open(url)`. Если значение атрибута не строка или пустое — не рендерится.

```tsx
{ id: "value", type: "button", attributeName: "report_url", value: "Открыть отчёт", options: { icon: "open_in_new" } }
```

---

## ElementCamera

**Назначение:** Галерея снимков с камеры видеонаблюдения. Асинхронно подгружает снимки и таймлайн.

**Props:** `CameraAttributeProps` (`type, elementConfig, renderElement, field?`)

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего URL камеры |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `expandable` | `boolean` | Разрешить раскрытие галереи |

**Зависимости:** `useCameraAttribute(cameraUrl)` — `galleryImages`, `totalCount`, `isLoadingSnapshot`, `isLoadingTimeline`

**Поведение:** `isLoadingSnapshot` → `LinearProgress`; нет снимков → `NoLiveSnapshot`; есть → `SmallPreview` + `Preview` (галерея)

```tsx
{ id: "value", type: "camera", attributeName: "cameraUrl", options: { expandable: true } }
```

---

## ElementChart

**Назначение:** Обёртка-делегат для [[components|компонента]] `Chart`. Получает конфиг через `useWidgetConfig`. Поддерживает типы: `bar`, `pie`, `line`, `stack`.

**Типы:** `type = "chart"` · `ElementChartOptions` · `ElementChartProps`.

**Опции (`ElementChartOptions` Pick):**

| Опция | Тип | Описание |
|---|---|---|
| `chartType` | `"bar" \| "line" \| "pie" \| "stack"` | Тип графика (`ChartType` union) |
| `relatedDataSources` | `ConfigRelatedDataSource[]` | Источники данных для осей: `{ name, chartAxis: "x" \| "y", attributeName, attributeAlias }` |
| `column` | `boolean` | Вертикальная раскладка |
| `markers` | `BarChartMarker[] \| string` | Маркеры на барчарте (массив или имя датасорса) |
| `showLabels` | `boolean` | Подписи столбцов |
| `showMarkers` | `number` | Шаг показа маркеров |
| `showTotal` | `boolean` | Показывать итог |
| `totalWord` | `string` | Слово для итога |
| `totalAttribute` | `string` | Имя атрибута для итога |
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |
| `defaultColor` | `string` | Цвет по умолчанию (PieChart-секторов) |
| `dotSnapping` | `boolean` | Привязка точек LineChart |
| `height`, `width` | `number` | Размеры |
| `radius` | `number` | Радиус PieChart |
| `padding` | `number` | Внутренние отступы |
| `fontColor` | `string` | Цвет текста |
| `angle` | `number` | Угол поворота подписей оси |
| `barWidth` | `number` | Ширина столбца BarChart |
| `cornerRadius` | `number` | Закругление столбцов BarChart |

```tsx
{
  id: "chart",
  type: "chart",
  options: {
    chartType: "bar",
    relatedDataSources: [{ name: "floors_ds", chartAxis: "y", attributeName: "count" }],
    otherItems: 10,
    sortByValue: true
  }
}
```

---

## ElementChips

**Назначение:** Теги-чипы, разобранные из строки атрибута по разделителю. Используется для отображения категорий, тегов, статусов.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута со строкой тегов |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `separator` | `string` | Разделитель тегов (default: `,`) |
| `bgColor` | `string` | Цвет фона чипов |
| `fontColor` | `string` | Цвет текста чипов |
| `fontSize` | `string \| number` | Размер шрифта чипов |
| `colorAttribute` | `string` | Имя атрибута, определяющего цвет чипа |

**Поведение:** `attribute.value.split(separator)` → массив тегов → `DashboardChip`

```tsx
{ id: "value", type: "tags", attributeName: "categories", options: { separator: ";", bgColor: "#e3f2fd", fontColor: "#1565c0" } }
```

---

## ElementControl

**Назначение:** Управляющий элемент для редактирования атрибута объекта (только в FeatureCard). Загружает варианты из источника данных.

**Props:** `ContainerProps` (всегда использует `WidgetType.FeatureCard`)

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | Источник данных для вариантов выбора |
| `label` | `string` | Подпись элемента управления |
| `width` | `string \| number` | Ширина элемента |
| `placeholder` | `string` | Placeholder |
| `control` | `ConfigControl` | Конфиг контрола: `{ type, targetAttributeName, attributeName }` |

`ConfigControl.type`: `"dropdown"` \| `"chips"` \| `"checkbox"` \| `"string"`.

**Поведение:** загружает items из `dataSource`, по выбору вызывает `changeControls({ [targetAttributeName]: value })`. Проверяет `isEditable` из `layerInfo`.

```tsx
{
  id: "value",
  type: "control",
  options: {
    relatedDataSource: "statusDs",
    control: { type: "dropdown", targetAttributeName: "status", attributeName: "id" },
    label: "Статус объекта"
  }
}
```

---

## ElementIcon

**Назначение:** Иконка из библиотеки `@evergis/uilib-gl` или из значения атрибута.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `value` | `IconTypesKeys` | Статическое имя иконки из библиотеки |
| `attributeName` | `string` | Имя атрибута, содержащего имя иконки |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `fontSize` | `string \| number` | Размер иконки |
| `fontColor` | `string` | Цвет иконки |

**Поведение:** если указан `attributeName` — берёт значение атрибута как `IconTypesKeys`; иначе `elementConfig.value`.

```tsx
{ id: "icon", type: "icon", value: "star", options: { fontSize: "24px", fontColor: "#f39c12" } }
```

---

## ElementImage

**Назначение:** Изображение из ресурса или атрибута. Загружает с авторизацией через [[hooks|хук]] `useFetchImageWithAuth`.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `value` | `string` | Статический URL или путь ресурса |
| `attributeName` | `string` | Имя атрибута, содержащего URL изображения |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `width` | `number \| string` | Ширина изображения в px или CSS-значение |

**Поведение:** `value` → `getResourceUrl(value)`; `attributeName` → первый элемент из значения атрибута (разделённый `;`). Если URL не получен — не рендерится.

```tsx
// статический URL
{ id: "bgImage", type: "image", value: "https://example.com/bg.png", options: { width: 200 } }

// из атрибута объекта
{ id: "image", type: "image", attributeName: "photoUrl", options: { width: 200 } }
```

---

## ElementLegend

**Назначение:** Легенда чарта. Отображает список элементов с цветовой меткой и подписью. Берёт данные из `useChartData` по `chartId`.

**Props:** `ContainerProps` + `element: ConfigContainerChild`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `chartId` | `string` | `id` дочернего элемента-чарта, к которому привязана легенда |
| `twoColumns` | `boolean` | Отображать легенду в две колонки |
| `relatedDataSources` | `ConfigRelatedDataSource[]` | Источники данных (для line-чарта — оси Y) |
| `fontSize` | `string \| number` | Размер шрифта элементов легенды |

**Поведение:** для line-чарта — показывает оси Y как элементы легенды; для bar/pie — items из `data[0].items` с alias из атрибутов.

```tsx
{ id: "legend", type: "legend", options: { chartId: "chart", twoColumns: false, fontSize: 12 } }
```

---

## ElementLink

**Назначение:** Ссылка из значения атрибута. Внешние ссылки (`http...`) открываются в новой вкладке, внутренние — через `LocalLink` (SPA-навигация).

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего URL |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `simple` | `boolean` | Простой `<a>` без стилизации |
| `title` | `string` | Статический текст ссылки (вместо URL из атрибута) |

**Поведение:** `attribute.value` → `getResourceUrl` → если начинается с `http` → `ExternalLink`; иначе → `LocalLink`.

```tsx
{ id: "value", type: "link", attributeName: "docUrl", options: { simple: true, title: "Документация" } }
```

---

## ElementMarkdown

**Назначение:** Рендеринг Markdown-контента с поддержкой раскрытия по длине. Полезен для описаний объектов, инструкций.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `value` | `string` | Статический Markdown-контент |
| `attributeName` | `string` | Имя атрибута, содержащего Markdown-текст |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `expandLength` | `number` | Число символов до скрытия текста. `0` = не скрывать |

**Поведение:** контент из `elementConfig.value` или `attributes[attributeName].value`. Рендерит через `react-markdown` с `rehype-raw`, `rehype-sanitize`, `remark-gfm`.

```tsx
{ id: "value", type: "markdown", attributeName: "description", options: { expandLength: 300 } }
```

---

## ElementModal

**Назначение:** Иконка-кнопка, открывающая диалоговое окно с конфигом из `config.modals[modalId]`.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `modalId` | `string` | **Обязательный.** ID модала из `config.modals` |
| `icon` | `IconTypesKeys` | Иконка кнопки (default: `"new_window"`) |

**Поведение:** находит `ConfigModal` по `modalId`, рендерит `ContainerChildren` внутри `Dialog`.

**Конфигурация модалки (`config.modals[]`, `ConfigModal`):**

| Поле | Тип | Описание |
|---|---|---|
| `id` | `ModalId` | Id модалки, на который ссылается `ElementModal.options.modalId` |
| `options.title` | `string` | Заголовок диалога |
| `options.maxWidth` | `string` | CSS `max-width` диалога |
| `options.minWidth` | `string` | CSS `min-width` диалога |
| `options.minHeight` | `string` | CSS `min-height` диалога |
| `children` | `ConfigContainerChild[]` | Содержимое модалки (рендерится через `ContainerChildren`) |

```tsx
// Кнопка-открытие в children контейнера
{ id: "open_details", type: "modal", options: { modalId: "detailsModal", icon: "info" } }

// Сама модалка — на корневом уровне config
modals: [
  {
    id: "detailsModal",
    options: { title: "Подробности", maxWidth: "800px", minHeight: "400px" },
    children: [
      { id: "chart", templateName: "Chart", options: { chartType: "bar", relatedDataSource: "sales" } },
      { id: "txt", type: "markdown", attributeName: "description" }
    ]
  }
]
```

---

## ElementSlideshow

**Назначение:** Слайдшоу изображений с поддержкой полноэкранного просмотра. Может использовать источник данных или атрибут объекта.

**Типы:** `type = "slideshow"` · `ElementSlideshowOptions` · `ElementSlideshowProps`. Локальный тип — `DashboardSlideshowProps` (Pick от `ElementSlideshowProps`) в `elements/ElementSlideshow/types.ts`.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего URL изображений (через разделитель) |

**Опции (`options`, `ElementSlideshowOptions` Pick):**

| Опция | Тип | Описание |
|---|---|---|
| `relatedDataSource` | `string` | Источник данных с изображениями (приоритет над `attributeName`) |
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |
| `controls` | `ConfigControl[]` | Маппинг полей источника на поля слайдшоу |

**Поведение:** если `relatedDataSource` → из features датасорса; иначе `getSlideshowImages({ element, attribute })`. Открывает `Preview` галерею по клику.

```tsx
{ id: "slideshow", type: "slideshow", attributeName: "photos", options: { relatedDataSource: "photosDs", expandable: true } }
```

---

## ElementSvg

**Назначение:** SVG-изображение, загружаемое по URL. Поддерживает три источника URL: атрибут объекта, иконка слоя, статическое значение.

**Props:** `ContainerProps` + опционально `layerInfo`, `attributes`

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `value` | `string` | Статический путь к SVG-ресурсу |
| `attributeName` | `string` | Имя атрибута с URL SVG |
| `attributeIcon` | `string` | Имя атрибута для поиска иконки слоя |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `width` | `number \| string` | Ширина |
| `height` | `number \| string` | Высота |
| `fontColor` | `string` | Цвет заливки SVG (через CSS `color`) |

**Поведение:** `getSvgUrl({ elementConfig, layerInfo, attributes })` → `getResourceUrl(url)` → `SvgImage`.

```tsx
{ id: "icon", type: "svg", attributeName: "iconUrl", options: { width: 32, height: 32, fontColor: "#2980b9" } }
```

---

## ElementTooltip

**Назначение:** Иконка с всплывающей подсказкой при наведении. Текст берётся из атрибута или статического значения.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `value` | `string` | Статический текст тултипа |
| `attributeName` | `string` | Имя атрибута, содержащего текст тултипа |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `icon` | `IconTypesKeys` | Иконка (default: `"question"`) |

```tsx
{ id: "value", type: "tooltip", attributeName: "hint", options: { icon: "info" } }
```

---

## ElementUploader

**Назначение:** Компонент загрузки файлов на сервер. После загрузки может обновить значение фильтра для перезапроса данных.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `fileExtensions` | `string` | Допустимые расширения файлов (default: `".txt,.csv,.py"`) |
| `multiSelect` | `boolean` | Разрешить загрузку нескольких файлов |
| `parentResourceId` | `string` | ID родительского ресурса (папка на сервере) |
| `icon` | `IconTypesKeys` | Иконка кнопки загрузки |
| `title` | `string` | Текст кнопки загрузки |
| `filterName` | `string` | Имя фильтра, обновляемого после загрузки файла |

**Поведение:** загружает файл через `api.file.upload`, после успеха обновляет `filters[filterName]` через `changeFilters`. Удаление через `api.file.deleteResource`.

```tsx
{
  id: "value",
  type: "uploader",
  options: {
    fileExtensions: ".csv,.xlsx",
    filterName: "dataFile",
    parentResourceId: "upload_folder_id",
    title: "Загрузить данные"
  }
}
```

---

## Связанные разделы

[[containers|Контейнеры]] | [[concepts|Основные понятия]] | [[hooks|Хуки]] | [[options|Опции]] | [[types|Типы]]
