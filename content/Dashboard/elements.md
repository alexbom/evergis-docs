# Элементы

## Обзор

Элементы — «листовые» компоненты, отображающие отдельные значения внутри контейнеров. Регистрируются в `elements/registry.ts` по строковому ключу (`type`). Рендерятся через [[utils|утилиту]] `getRenderElement`.

Все принимают `ContainerProps` (`type`, `elementConfig`, `renderElement`).

---

## ElementButton

**Назначение:** Кнопка, открывающая URL из атрибута объекта в новой вкладке.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `value` | `string` | Текст кнопки |
| `attributeName` | `string` | Имя атрибута, содержащего URL для открытия |
| `icon` | `IconTypesKeys` | Иконка кнопки |

**Поведение:** читает `attributeName` → `attribute.value` (URL) → `window.open(url)`. Если значение атрибута не строка или пустое — не рендерится.

```tsx
{ type: "button", attributeName: "report_url", value: "Открыть отчёт", options: { icon: "open_in_new" } }
```

---

## ElementCamera

**Назначение:** Галерея снимков с камеры видеонаблюдения. Асинхронно подгружает снимки и таймлайн.

**Props:** `CameraAttributeProps` (`type, elementConfig, renderElement, field?`)

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего URL камеры |
| `expandable` | `boolean` | Разрешить раскрытие галереи |

**Зависимости:** `useCameraAttribute(cameraUrl)` — `galleryImages`, `totalCount`, `isLoadingSnapshot`, `isLoadingTimeline`

**Поведение:** `isLoadingSnapshot` → `LinearProgress`; нет снимков → `NoLiveSnapshot`; есть → `SmallPreview` + `Preview` (галерея)

```tsx
{ type: "camera", attributeName: "cameraUrl", options: { expandable: true } }
```

---

## ElementChart

**Назначение:** Обёртка-делегат для [[components|компонента]] `Chart`. Получает конфиг через `useWidgetConfig`. Поддерживает типы: `bar`, `pie`, `line`, `stackBar`.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `chartType` | `"bar" \| "pie" \| "line" \| "stackBar"` | Тип графика |
| `relatedDataSources` | `ConfigRelatedDataSource[]` | Источники данных для осей: `{ name, chartAxis: "x" \| "y", attributeName, attributeAlias }` |
| `otherItems` | `number` | Максимальное кол-во элементов, остальные группируются в «Другое» |
| `shownItems` | `number` | Кол-во отображаемых элементов (с кнопкой «Ещё») |
| `sortByValue` | `boolean` | Сортировать данные по значению |
| `markers` | `BarChartMarker[] \| string` | Маркеры на барчарте (массив или имя датасорса) |
| `horizontal` | `boolean` | Горизонтальный барчарт |
| `stackedBar` | `boolean` | Столбчатый с накоплением |

```tsx
{
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

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `separator` | `string` | Разделитель тегов (default: `,`) |
| `bgColor` | `string` | Цвет фона чипов |
| `fontColor` | `string` | Цвет текста чипов |
| `fontSize` | `string \| number` | Размер шрифта чипов |
| `colorAttribute` | `string` | Имя атрибута, определяющего цвет чипа |

**Поведение:** `attribute.value.split(separator)` → массив тегов → `DashboardChip`

```tsx
{ type: "tags", attributeName: "categories", options: { separator: ";", bgColor: "#e3f2fd", fontColor: "#1565c0" } }
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

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `value` | `IconTypesKeys` | Статическое имя иконки из библиотеки |
| `attributeName` | `string` | Имя атрибута, содержащего имя иконки |
| `fontSize` | `string \| number` | Размер иконки |
| `fontColor` | `string` | Цвет иконки |

**Поведение:** если указан `attributeName` — берёт значение атрибута как `IconTypesKeys`; иначе `elementConfig.value`.

```tsx
{ type: "icon", value: "star", options: { fontSize: "24px", fontColor: "#f39c12" } }
```

---

## ElementImage

**Назначение:** Изображение из ресурса или атрибута. Загружает с авторизацией через [[hooks|хук]] `useFetchImageWithAuth`.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `value` | `string` | Статический URL или путь ресурса |
| `attributeName` | `string` | Имя атрибута, содержащего URL изображения |
| `width` | `number \| string` | Ширина изображения в px или CSS-значение |

**Поведение:** `value` → `getResourceUrl(value)`; `attributeName` → первый элемент из значения атрибута (разделённый `;`). Если URL не получен — не рендерится.

```tsx
{ type: "image", attributeName: "photoUrl", options: { width: 200 } }
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
{ type: "legend", options: { chartId: "myChart", twoColumns: false, fontSize: 12 } }
```

---

## ElementLink

**Назначение:** Ссылка из значения атрибута. Внешние ссылки (`http...`) открываются в новой вкладке, внутренние — через `LocalLink` (SPA-навигация).

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего URL |
| `simple` | `boolean` | Простой `<a>` без стилизации |
| `title` | `string` | Статический текст ссылки (вместо URL из атрибута) |

**Поведение:** `attribute.value` → `getResourceUrl` → если начинается с `http` → `ExternalLink`; иначе → `LocalLink`.

```tsx
{ type: "link", attributeName: "docUrl", options: { simple: true, title: "Документация" } }
```

---

## ElementMarkdown

**Назначение:** Рендеринг Markdown-контента с поддержкой раскрытия по длине. Полезен для описаний объектов, инструкций.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего Markdown-текст |
| `value` | `string` | Статический Markdown-контент |
| `expandLength` | `number` | Число символов до скрытия текста. `0` = не скрывать |

**Поведение:** контент из `elementConfig.value` или `attributes[attributeName].value`. Рендерит через `react-markdown` с `rehype-raw`, `rehype-sanitize`, `remark-gfm`.

```tsx
{ type: "markdown", attributeName: "description", options: { expandLength: 300 } }
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

```tsx
{ type: "modal", options: { modalId: "detailsModal", icon: "info" } }
```

---

## ElementSlideshow

**Назначение:** Слайдшоу изображений с поддержкой полноэкранного просмотра. Может использовать источник данных или атрибут объекта.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего URL изображений (через разделитель) |
| `relatedDataSource` | `string` | Источник данных с изображениями (приоритет над `attributeName`) |
| `expandable` | `boolean` | Разрешить сворачивание |
| `expanded` | `boolean` | Развёрнут ли по умолчанию |

**Поведение:** если `relatedDataSource` → из features датасорса; иначе `getSlideshowImages({ element, attribute })`. Открывает `Preview` галерею по клику.

```tsx
{ type: "slideshow", attributeName: "photos", options: { relatedDataSource: "photosDs", expandable: true } }
```

---

## ElementSvg

**Назначение:** SVG-изображение, загружаемое по URL. Поддерживает три источника URL: атрибут объекта, иконка слоя, статическое значение.

**Props:** `ContainerProps` + опционально `layerInfo`, `attributes`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута с URL SVG |
| `attributeIcon` | `string` | Имя атрибута для поиска иконки слоя |
| `value` | `string` | Статический путь к SVG-ресурсу |
| `width` | `number \| string` | Ширина |
| `height` | `number \| string` | Высота |
| `fontColor` | `string` | Цвет заливки SVG (через CSS `color`) |

**Поведение:** `getSvgUrl({ elementConfig, layerInfo, attributes })` → `getResourceUrl(url)` → `SvgImage`.

```tsx
{ type: "svg", attributeName: "iconUrl", options: { width: 32, height: 32, fontColor: "#2980b9" } }
```

---

## ElementTooltip

**Назначение:** Иконка с всплывающей подсказкой при наведении. Текст берётся из атрибута или статического значения.

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `attributeName` | `string` | Имя атрибута, содержащего текст тултипа |
| `value` | `string` | Статический текст тултипа |
| `icon` | `IconTypesKeys` | Иконка (default: `"question"`) |

```tsx
{ type: "tooltip", attributeName: "hint", options: { icon: "info" } }
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

[[containers|Контейнеры]] | [[concepts|Основные понятия]] | [[hooks|Хуки]]
