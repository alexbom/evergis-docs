# Элементы

> [!danger] У каждого элемента обязателен `id` = slot-id родителя
> Значение `id` элемента должно совпадать со slot-id, который ожидает контейнер-родитель (`alias`, `chart`, `value`, ...). Неверный slot → элемент не отрисуется. Таблица slot-id и чек-лист — [[authoring|Правила генерации]].

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
| `table` | `ElementTable` |
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

**Опции (`options`):** нет — `ElementButtonOptions = Record<string, never>`.

**Поведение:** читает `attributeName` → `attribute.value` (URL) → `window.open(url)`. Если значение атрибута не строка или пустое — не рендерится.

```tsx
{ id: "value", type: "button", attributeName: "report_url", value: "Открыть отчёт" }
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
| `expanded` | `boolean` | Развёрнута ли галерея по умолчанию |

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
| `shownItems` | `number` | Сколько категорий показать на графике |
| `otherItems` | `number` | Лимит категорий, после которого остаток сворачивается в «Другое» (обрезка и группировка идут в `getDataFromAttributes` / `getDataFromRelatedFeatures` внутри [[hooks\|`useChartData`]]) |

```tsx
{
  id: "chart",
  type: "chart",
  options: {
    chartType: "bar",
    relatedDataSources: [{ name: "floors_ds", chartAxis: "y", attributeName: "count" }],
    showLabels: true,
    showTotal: true
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
| `variants` | `IOption[] \| ChipOption[]` | Варианты соответствия значение → цвет/подпись чипа |

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

**Назначение:** Изображение из атрибута объекта, файлового ресурса, произвольного URL или вложения. Загружает с авторизацией через [[hooks|хук]] `useFetchImageWithAuth`.

**Поля конфига (корневой уровень `ConfigContainerChild`):**

| Поле | Тип | Описание |
|---|---|---|
| `value` | `string` | Статический URL или путь ресурса |
| `attributeName` | `string` | Имя атрибута, **значение** которого содержит URL изображения (или атрибут-вложение) |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `resourceId` | `string` | Id файлового ресурса — разворачивается в `/sp/resources/file/<id>` |
| `url` | `string` | Адрес картинки; проходит через `getResourceUrl` — `http…` берётся как есть, остальное разворачивается в `/sp/resources/file/<url>` |
| `width` | `CssSize` | Ширина изображения: число — px, строка — любое CSS-значение |
| `height` | `CssSize` | Высота изображения |
| `fit` | `"cover" \| "contain" \| "fill" \| "none" \| "scale-down"` | CSS `object-fit` — как изображение масштабируется внутри своего бокса |

**Поведение:** адрес резолвится в [[utils|`getImageUrl`]] — первый непустой источник выигрывает:

```
options.resourceId → options.url → value → attributeName
```

Все источники проходят через `getResourceUrl` (как и у [[elements#ElementSvg|ElementSvg]]): `http…` берётся как есть, остальное разворачивается в `/sp/resources/file/<url>`. Из значения атрибута берётся первый адрес до разделителя `;`. Если атрибут имеет `subType === Attachments`, строковый канал даёт `null` и картинка грузится каналом вложений (`useAttachmentItems` + `useAttachmentPreviewImages`, берётся первое изображение). Настройки атрибута в слое (`attributesConfiguration.attributes[].icon`) элемент `image` **не читает** — это источник только для `attributeIcon` у [[elements#ElementSvg|ElementSvg]]. Если адреса нет — элемент не рендерится. Размеры собираются через [[utils|`getWrapperSizeStyle`]] и уходят в CSS; в HTML-атрибут `width` попадает только числовое (пиксельное) значение.

SVG-ресурс отображается как обычная картинка (`<img>`), без перекраски — для inline-SVG с управлением цветом есть [[elements#ElementSvg|ElementSvg]].

```tsx
// статический URL
{ id: "bgImage", type: "image", value: "https://example.com/bg.png", options: { width: 200 } }

// из атрибута объекта, вписать в бокс без искажений
{ id: "image", type: "image", attributeName: "photoUrl", options: { width: "100%", height: 160, fit: "cover" } }

// готовый адрес из конфига
{ id: "image", type: "image", options: { url: "https://example.com/logo.png", width: 24, height: 24 } }

// файловый ресурс по id
{ id: "image", type: "image", options: { resourceId: "3f1c...", width: 48 } }
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
| `chartType` | `ChartType` | Тип привязанного чарта (`bar` \| `line` \| `pie` \| `stack`) |

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
| `noMargin` | `boolean` | Убрать внешний margin обёртки |
| `typography` | `MarkdownTypography` | Пер-тег типографика: `{ h1..h6, p, li, code }` → `{ fontSize, lineHeight, fontWeight, marginTop, marginBottom }`. Незаданные теги/свойства берут дефолт `MarkdownWrapper` (см. [[types#Типографика markdown\|Типы]]) |

**Поведение:** контент из `elementConfig.value` или `attributes[attributeName].value`. Рендерит через `react-markdown` с `rehype-raw`, `rehype-sanitize` (схема расширена — разрешён атрибут `style`), `remark-gfm`. При `expandLength > 0` и превышении длины показывает `LegendToggler` «Подробнее» / «Свернуть».

```tsx
{
  id: "value",
  type: "markdown",
  attributeName: "description",
  options: {
    expandLength: 300,
    typography: {
      h2: { fontSize: "1.25rem", marginBottom: "0.5rem" },
      p: { fontSize: "0.875rem", lineHeight: "1.4" }
    }
  }
}
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

**Загрузка:** содержимое заменяется на `DashboardLoading` только когда данных нет вообще — по [[hooks#useDataSourceLoading|`useDataSourceLoading`]], тому же условию, что и корневой `Dashboard`. При частичном обновлении источника (смена фильтра, autoSync) модалка остаётся на экране: перерисовываются лишь контейнеры, зависящие от обновившегося ИД, каждый со своим локальным скелетоном.

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
| `attributeIcon` | `string` | Имя атрибута, из **настроек** которого берётся иконка (`icon.resourceId \|\| icon.url` в `attributesConfiguration` слоя) |
| `attributeName` | `string` | Имя атрибута, **значение** которого содержит URL SVG |
| `value` | `string` | Статический путь к SVG-ресурсу |

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `width` | `number \| string` | Ширина |
| `height` | `number \| string` | Высота |
| `fontColor` | `string` | Цвет заливки SVG (через CSS `color`) |

**Поведение:** [[utils|`getSvgUrl`]]`({ elementConfig, layerInfo, attributes })` → `getResourceUrl(url)` → `SvgImage`. Порядок источников — первый непустой выигрывает: `attributeIcon` → `attributeName` → `value`. Разбор `attributeIcon` вынесен в утилиту [[utils|`getAttributeIconUrl`]] (читает `icon` из `attributesConfiguration` слоя).

```tsx
{ id: "icon", type: "svg", attributeName: "iconUrl", options: { width: 32, height: 32, fontColor: "#2980b9" } }
```

---

## ElementTable

**Назначение:** Представление структурированных данных таблицей — единственный поддерживаемый вид для [[containers#StructuredDataContainer|StructuredDataContainer]]. Данные, схему и колбэки правки берёт из контекста родительского контейнера, поэтому вне него не рендерится.

**Props:** `ElementTableProps` (`ContainerProps` с `elementConfig: ElementTableConfig`)

**Slot-id:** `data` — единственное место, где элемент работает.

**Колонки — это схема контейнера** (`attributesDescription`) как есть: сколько атрибутов описано и в каком порядке, столько колонок и в том же порядке. Своего отбора у представления нет — состав колонок правится схемой контейнера, а не опциями элемента.

**Опции (`options`):**

| Опция | Тип | Описание |
|---|---|---|
| `sort` | `boolean` | Разрешить сортировку кликом по заголовку: по возрастанию → по убыванию → исходный порядок. Сортировка локальная и только визуальная — в фильтр уходит исходный порядок строк |
| `width` | `CssSize` | Ширина САМОЙ ТАБЛИЦЫ. Задана — колонки встают по содержимому, предел ячейки снимается, лишнее уходит в горизонтальную прокрутку таблицы |
| `height` | `CssSize` | Высота САМОЙ ТАБЛИЦЫ: строки прокручиваются внутри неё, шапка липнет к верху. Только конкретные единицы — процент не разрешается |

**Редакторы ячеек по типу атрибута:** `String` → `Input`, `Int32`/`Int64`/`Double` → `NumberInput`, `Boolean` → `Checkbox`, `DateTime` → `DatePicker`. Редактор появляется при трёх условиях сразу: у контейнера включён `editMode`, атрибут помечен `isEditable`, тип редактируем. `isEditable` берётся от источника только там, где в него и пишут: у [[containers#StructuredDataContainer|источника без слоя]] его запрет не действует, и колонку закрывает лишь `attributesDescription`. Атрибут любого другого типа показывается только на чтение, даже с `isEditable: true`; логическое значение и на чтение рисуется галкой, но неактивной.

**Значок сортировки не двигает таблицу.** Пока `sort` включён, место под него в заголовке занято всегда — сам значок только прячется. Иначе он менял бы ширину колонки, появляясь и исчезая, и таблицу дёргало бы на каждый клик по заголовку, вместе с телом (по замеру — колонки уезжают на 3–16 px). Оба направления рисуются значком в `1rem`, поэтому переключение asc/des ширину тоже не трогает.

**Формат значения.** Вне фокуса ячейка всегда показывает значение через `stringFormat` (`formatAttributeValue`), редактор подставляется на фокус — иначе форматирование пришлось бы дублировать внутри инпута. Пустые ячейки при сортировке всегда внизу, независимо от направления.

**Правку открывает фокус.** Ячейка вне правки — кнопка, и она же таб-стоп: Tab по таблице ставит курсор сразу в поле нужной ячейки, отдельный `Enter` не нужен, а из открытой ячейки Tab идёт в следующую, закрывая предыдущую. `Enter` и `Escape` закрывают редактор, но фокус оставляют на ячейке — иначе он падал бы в `body` и следующий Tab пошёл бы с начала страницы; с закрытой ячейки `Enter` открывает поле снова. Поэтому состояний у ячейки три, а не два: закрыта, закрыта под фокусом (вышли по `Enter`/`Escape`) и открыта — без среднего возврат фокуса на кнопку тут же открывал бы правку заново. Действие по умолчанию у закрывающего `Enter` гасится: что нажать, браузер смотрит уже после обработчиков, а к этому моменту в фокусе стоит кнопка ячейки, и правка открылась бы тем же нажатием.

Колонки с `isEditable: false` Tab пропускает — таб-стопа у них нет. Логическая колонка зависит от версии `@evergis/uilib-gl`: пока её `Checkbox` прятал свой инпут атрибутом `hidden`, тот выпадал из таб-порядка и галка правилась только мышью; после фикса (инпут не скрыт, а прозрачен и вынесен из потока) колонка проходится Tab-ом, переключается пробелом, а фокус видно по рамке вокруг галки.

**Ширина ячейки при правке.** Редактор кладётся поверх невидимой копии значения: копия остаётся в потоке и держит ширину колонки прежней, а поле ввода занимает ровно ширину ячейки (собственная ширина инпута по содержимому снимается `min-width: 0`). Без этого таблица дёргалась бы на каждый вход в правку — контролы uilib по умолчанию просят ~14rem.

**Выход из правки — клик снаружи**, а не `blur`: `DatePicker` уводит фокус из поля уже на клике по иконке календаря, и по `blur` редактор закрывался бы раньше, чем календарь успевал открыться. Слушается `mousedown` на документе, клики внутри `#portal-root` (там живут выпадающие слои uilib, включая календарь) считаются «своими». С клавиатуры уход виден по `relatedTarget` события `focusout`: фокус ушёл на соседнюю ячейку — правка закрыта, фокус «в никуда» (внутренности `DatePicker`) уходом не считается.

**Дата и время.** Выбор времени в `DatePicker` включается, только если время есть в `stringFormat.format` (`hh:mm…`) — иначе правилось бы то, чего в ячейке не видно. `DatePicker` повторяет текущее значение на каждый уход фокуса из поля, поэтому повтор отбрасывается сравнением по моменту времени — иначе строка помечалась бы изменённой от одного лишь открытия календаря. Пропы `withTime`/`withHeader` передаются явно: `DatePicker` из `@evergis/uilib-gl` добирает дефолты через лодашевый `defaults(props, …)`, который пишет прямо в объект пропсов, а React в dev-режиме его замораживает — без явных пропов рендер падает с «Cannot add property withTime, object is not extensible».

**Собственный бокс таблицы.** `width`/`height` здесь — размер САМОЙ ТАБЛИЦЫ, а не контейнера: она получает собственный скролл-контейнер, и то, что в него не влезло, прокручивается внутри. Размеров нет — бокса нет: таблица растёт под содержимое, а лишнее, как и раньше, прокручивает область представления контейнера (`options.width`/`height` [[containers#StructuredDataContainer|контейнера]] отмеряют место всему контейнеру, вместе с заголовком и панелью кнопок).

Что меняет каждый из размеров:

- **`width`** — колонки перестают втискиваться в выданную ширину и встают по содержимому (`width: max-content`), предел ячейки в `20rem` снимается, значения не режутся многоточием. Иначе вышла бы бессмыслица: бокс с прокруткой при обрезанных значениях. При fill-ширине (`"100%"`) бокс вдобавок получает `contain: inline-size` — без этого процент от раскладки, которая меряет себя содержимым (ряд плиток, трек `auto`), вырождается в `auto`, и таблица распирает её изнутри.
- **`height`** — строки прокручиваются внутри бокса, липкая шапка липнет к его верху, а заголовок контейнера и панель кнопок остаются на месте. Только конкретные единицы (`240`, `"14rem"`, `"50vh"`): между областью представления и таблицей стоит блочная обёртка элемента с высотой `auto`, и процент в ней не разрешается. «Занять всю выданную высоту» — это `height` контейнера, в том числе `"100%"` для трека сетки.

Один заданный размер включает прокрутку по обеим осям — `overflow-x: auto` с `overflow-y: visible` в CSS невозможен.

**Липкая шапка прозрачна.** Своего фона у неё нет и быть не может: контейнер красится цветом из конфига, и закрашенная шапка осталась бы прямоугольником чужого цвета, как только этот цвет поменяли. Строки под неё не заезжают потому, что тело таблицы отсекается (`clip-path`) ровно на то, на сколько шапка его накрыла: низ шапки минус верх тела. Разница считается по факту, а не по величине прокрутки, поэтому одинаково работает и когда прокручивает собственный бокс таблицы, и когда её прокручивает область представления контейнера.

```tsx
{ id: "data", type: "table", options: { sort: true, width: "100%", height: "14rem" } }
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
