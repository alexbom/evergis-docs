# Опции

## Обзор

`ConfigOptions` — общий словарь конфигурационных опций для всех контейнеров, элементов и шапок Dashboard. Исторически это был один плоский интерфейс с 100+ полями. После рефакторинга типизации интерфейс разбит на **12 доменных миксинов**:

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
    ConfigMiscOptions {}
```

> `ConfigEntityRefOptions` — двенадцатый миксин — в `extends` **не входит**: это документирующий «highlight»-интерфейс, чьи поля (`chartId`, `modalId`, `tabId`, ...) уже продублированы в других миксинах (`ConfigChartOptions`, `ConfigEditOptions`, `ConfigMapLayerOptions`, `ConfigMiscOptions`, `ConfigDataSourceBindingOptions`). Он подсвечивает entity-ref природу этих полей — см. раздел ниже.

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
| `width` | `number` | Ширина в px |
| `height` | `number` | Высота в px |
| `padding` | `number` | Внутренние отступы |
| `radius` | `number` | Радиус (для PieChart — относительно контейнера; для табов — `border-radius`) |
| `cornerRadius` | `number` | Закругление углов столбцов BarChart |
| `column` | `boolean` | Вертикальная раскладка детей (один в столбик) |
| `twoColumns` | `boolean` | Двухколоночная раскладка (Chart-легенда / fallback ChartContainer) |
| `align` | `"left" \| "center" \| "right"` | Выравнивание текста/блоков |
| `center` | `boolean` | Центрирование контента |
| `innerTemplateStyle` | `CSSProperties` | Кастомные CSS-стили внутреннего шаблона (`OneColumn`, `TwoColumn`, `Progress`, `RoundedBackground`) |
| `withPadding` | `boolean` | Дополнительный внутренний padding (шапки FeatureCard) |
| `withDivider` | `boolean` | Показывать разделитель |
| `bottomBlur` | `boolean` | Эффект размытия снизу (`FeatureCardBackgroundHeader`) |
| `noMargin` | `boolean` | Убрать внешний margin |
| `maxTextWidth` | `number` | Максимальная ширина текста в px |
| `barWidth` | `number` | Ширина столбца BarChart |
| `barHeight` | `number` | Высота StackBar |

**Используется в:** `ElementChart`, `ElementImage`, `ElementSvg`, `ElementControl`, `ChartContainer`, `ContainersGroupContainer`, `DataSourceContainer`, `DataSourceInnerContainer`, `OneColumnContainer`, `PagesContainer`, `ProgressContainer`, `RoundedBackgroundContainer`, `TabsContainer`, `TaskContainer`, `TwoColumnContainer`, `FeatureCardBackgroundHeader`, `FeatureCardDefaultHeader`, `FeatureCardSlideshowHeader`.

---

## ConfigTypographyOptions

Типографика и цвета.

| Поле | Тип | Описание |
|---|---|---|
| `fontSize` | `string` | CSS-длина (`"14px"`, `"0.875rem"`, `"larger"`) — см. `FontSizeToken` |
| `fontColor` | `string` | Цвет текста (CSS) |
| `bgColor` | `string` | Цвет фона |
| `backgroundColor` | `string` | Альтернативное имя для bgColor |
| `primaryColor` | `string` | Основной акцентный цвет |
| `defaultColor` | `string` | Цвет по умолчанию (PieChart-секторов и т.п.) |
| `noBg` | `boolean` | Отключить фон |
| `colors` | `string[]` | Массив цветов (Progress, multi-series Chart) |
| `colorAttribute` | `string` | Имя атрибута, из которого брать цвет |
| `statusColors` | `Record<RemoteTaskStatus, string>` | Цвета по статусам задачи |

**Используется в:** `ElementChart`, `ElementChips`, `ElementIcon`, `ElementLegend`, `ElementSvg`, `DividerContainer`, `FiltersContainer`, `ProgressContainer`, `RoundedBackgroundContainer`, `TabsContainer`, `TaskContainer`, `FeatureCardBackgroundHeader`, `FeatureCardSlideshowHeader`.

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
| `relatedDataSource` | `string` | ⚠️ entity-ref на `[[concepts#Источники данных\|ConfigDataSource]]` (см. `[[types#Branded types\|DataSourceName]]`) |
| `relatedDataSources` | `ConfigRelatedDataSource[]` | Несколько источников с alias/axis для серий графика |
| `relatedAttributes` | `ConfigRelatedAttribute[]` | Атрибуты из связанных слоёв (join) |
| `relatedResources` | `ConfigRelatedResource[]` | Связанные Python-ресурсы (TaskContainer) |
| `responseFilters` | `Record<string, string>` | Фильтры ответа задачи |
| `hideIfEmptyDataSource` | `string` | Скрыть контейнер, если указанный источник пуст |

**Используется в:** `ElementChart`, `ElementControl`, `ElementLegend`, `ElementSlideshow`, `AttachmentContainer`, `DataSourceContainer`, `DataSourceInnerContainer`, `DataSourceProgressContainer`, `EditAttachmentContainer`, `TaskContainer`.

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
| `overlay` | `string` | Наложение поверх изображения (CSS gradient, цвет) |
| `onlyIcon` | `boolean` | Показывать только иконку (без текста) |
| `bigIcon` | `boolean` | Большой размер иконки |
| `big` | `boolean` | Большой размер компонента |
| `tagView` | `boolean` | Отображать в виде «тэга» |

**Используется в:** `ElementModal`, `ElementUploader`, `AddFeatureButton`, `ExportPdfContainer`, `RoundedBackgroundContainer`, `TabsContainer` (`onlyIcon`), `TaskContainer`, `FeatureCardBackgroundHeader`, `FeatureCardDefaultHeader`, `FeatureCardSlideshowHeader`.

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
| `viewMode` | `"grid" \| "list"` | Режим отображения коллекции |
| `hideEmpty` | `boolean` | Скрывать пустые элементы |
| `limit` | `number` | Лимит на количество элементов |

**Используется в:** `AttachmentContainer`, `ChartContainer` (`hideEmpty`), `DataSourceProgressContainer` (`shownItems`), `EditAttachmentContainer`, `OneColumnContainer` (`hideEmpty`), `RoundedBackgroundContainer` (`hideEmpty`), `TabsContainer` (`shownItems`), `TwoColumnContainer` (`hideEmpty`).

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

**Используется в:** `ElementControl` (`control`), `ElementUploader` (`fileExtensions`, `multiSelect`), `ElementSlideshow` (`controls`), `AttachmentContainer` (`controls`), `EditGroupContainer`, `EditBooleanContainer`, `EditStringContainer`, `EditNumberContainer`, `EditDropdownContainer`, `EditChipsContainer`, `EditCheckboxContainer`, `EditDateContainer` (`withTime`, `controls`), `EditAttachmentContainer` (`controls`, `fileExtensions`), `FilterChild`, `DataSourceProgressContainer` (`maxValue`), `ProgressContainer` (`maxValue`).

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
| `innerTemplateName` | `ContainerTemplate` | **Обязателен для `DataSource`/`DataSourceProgress`.** Шаблон рендеринга каждой записи источника — читается пайплайном ([[containers\|`getRenderElement`]]) и конвертируется в проп `innerComponent` через `getContainerComponent`. Без него записи не рендерятся |
| `themeName` | `"light" \| "dark"` | Принудительная тема для шапки |
| `url` | `string` | URL (для DashboardDefaultHeader) |
| `inlineUnits` | `boolean` | Показывать единицы измерения inline |
| `noUnits` | `boolean` | Не показывать единицы измерения |
| `attributes` | `string[]` | Список атрибутов для отображения |
| `useProjectHiddenAttributes` | `boolean` | Учитывать `hiddenAttributes` проекта (PUBGL-658) |
| `innerValue` | `boolean` | Показывать значение внутри Progress |
| `groupTooltip` | `boolean` | Группировать tooltip |
| `wrap` | `boolean` | Переносить содержимое |
| `modalId`, `tabId`, `downloadById`, `parentResourceId` | `string` | Дубликаты entity-ref для удобства Pick |
| `useNotifications` | `boolean` | Показывать прогресс-уведомления о выполнении задачи (`TaskContainer`) |

**Используется в:** `DataSourceContainer`, `DataSourceProgressContainer` (`innerTemplateName` — через пайплайн рендера, не через Pick), `DashboardDefaultHeader` (`url`), `EditGroupContainer`, `OneColumnContainer`, `TwoColumnContainer` (`attributes`, `useProjectHiddenAttributes`), `ProgressContainer` (`innerValue`), `RoundedBackgroundContainer` (`inlineUnits`), `TitleContainer` (`downloadById`), `ElementModal` (`modalId`), `ElementUploader` (`parentResourceId`), `EditAttachmentContainer` (`parentResourceId`), `TaskContainer` (`useNotifications`), `FeatureCardBackgroundHeader`, `FeatureCardDefaultHeader`, `FeatureCardSlideshowHeader` (`themeName`).

> [!note] Опции, потребляемые пайплайном рендера (не через Pick)
> Часть полей `ConfigMiscOptions` читается пайплайном рендера напрямую из `options`, а не через `<Name>Options`-Pick конкретного компонента — их «Используется в» определяется по фактическому чтению в коде, а не по Pick. Ключевой пример — **`innerTemplateName`**: его читает [[containers|`getRenderElement`]] и конвертирует в `innerComponent` (`getContainerComponent`) для `DataSourceContainer`/`DataSourceProgressContainer`. Такие поля не значатся ни в одном `<Name>Options`, но обязательны — не считай их неиспользуемыми.

---

## Связанные разделы

[[types|Типы]] | [[concepts|Основные понятия]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[headers|Шапки]]
