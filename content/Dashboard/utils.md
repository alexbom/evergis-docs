# Утилиты

## Обзор

Утилиты расположены в `D:/projects/spcore_api/packages/react/src/components/Dashboard/utils/`. Чистые функции для обработки, преобразования и проверки данных.

---

## Получение данных

### getAttributeByName

`(attributeName: string | string[], attributes: ClientFeatureAttribute[]) => ClientFeatureAttribute | null`

Поиск атрибута по имени. Возвращает `null` для массива имён или пустого `attributeName`.

```ts
const attr = getAttributeByName("name", attributes);
```

---

### getAttributesConfiguration

`(layer: QueryLayerServiceInfoDc) => AttributesConfigurationDc`

Извлекает `attributesConfiguration` из конфига слоя с fallback на значения по умолчанию (`idAttribute`, `geometryAttribute`).

---

### getChartAxes

`(chartElement: ConfigContainerChild) => ConfigRelatedDataSource[]`

Возвращает оси `chartAxis === "y"` из `elementConfig.options.relatedDataSources`.

---

### getChartFilterName

`(relatedDataSources: ConfigRelatedDataSource[]) => string`

Возвращает `filterName` первой Y-оси чарта.

---

### getChartMarkers

`(items: FilterItem[], markers: BarChartMarker[] | string, dataSources: FetchedDataSource[]) => BarChartMarker[]`

Если `markers` — строка → имя датасорса, загружает маркеры из него. Иначе использует массив маркеров. Вычисляет `value` как индекс в `items`.

---

### getConfigFilter

`(filterName: string, configFilters: ConfigFilter[]) => ConfigFilter`

Поиск конфигурации фильтра по имени.

---

### getContainerComponent

`(innerTemplateName: string) => FC<ContainerProps> | null`

Resolves контейнер из registry. Если не найден — возвращает `default` (ContainersGroupContainer). Если `innerTemplateName` пустой — `null`.

---

### getControlTemplateName

`(type?: ConfigControl["type"]) => ContainerTemplate`

Маппинг типа контрола на `ContainerTemplate`: `chips` → `EditChips`, `checkbox` → `EditCheckbox`, `string` → `EditString`, иначе → `EditDropdown`.

---

### getDashboardHeader

`(templateName: HeaderTemplate) => FC<ContainerProps>`

Возвращает компонент шапки дашборда по `HeaderTemplate`. Поддерживает только `Default`.

---

### getDataFromAttributes

`(t, config, attributes?) => PieChartData[]`

Формирует данные для PieChart из дочерних элементов конфига и атрибутов объекта. Поддерживает сортировку по значению и группировку "других" элементов.

---

### getDataFromRelatedFeatures

`({ t, config, filters, relatedConfig, dataSource, layerInfo }) => FilterItem[] | null`

Формирует данные для чарта из features датасорса. Сортировка, обрезка по `otherItems`, генерация цветового градиента, форматирование значений через `formatAttributeValue`.

---

### getDataSource

`(dataSourceName: string, dataSources: WidgetDataSource[]) => WidgetDataSource`

Поиск загруженного датасорса по имени.

---

### getDataSourceFilterValue

`({ filterName, filterProp, attributeAlias, dataSource, selectedFilters }) => SelectedFilter["value"]`

Получает значение поля `filterProp` из feature датасорса, соответствующего текущему значению фильтра.

---

### getDefaultConfig

`({ title, defaultTitle, items, baseMapName, position, resolution, srid }) => ConfigContainer`

Создаёт полную конфигурацию дашборда по умолчанию с одной страницей и слоями из `items`.

---

### getFeatureAttributes

`(feature, layer?, dataSource?) => ClientFeatureAttribute[]`

Преобразует `FeatureDc` + `QueryLayerServiceInfoDc` в массив `ClientFeatureAttribute` с alias, type, value, readOnly, stringFormat и т.д.

---

### getFilterComponent

`(filterType: FilterType) => FC`

Возвращает React-компонент фильтра: `checkbox` → `CheckboxFilter`, `rangeNumber` → `RangeNumberFilter`, `barChart` → `BarChartFilter`, `rangeDate` → `RangeDateFilter`, `text` → `TextFilter`, `chips` → `ChipsFilter`, `dropdown` → `DropdownFilter` (default).

---

### getFilterSelectedItems

`(filterItems, filters, cardFilters) => ConfigContainerChild[]`

Фильтрует items фильтра, оставляя только те, для которых выбрано значение (или есть `defaultValue` в конфиге).

---

### getFilterValue

`<T>({ selectedFilters, configFilters, filterName, newValue? }) => T | T[]`

Вычисляет новое значение фильтра при клике. Учитывает `valueType` (`single` / `range` / `array`), toggles значение в массиве.

---

### getFormattedAttributes

`(t, data, attributes, config) => ClientFeatureAttribute[]`

Добавляет к атрибутам «Другое» если `otherItems < data.length`.

---

### getLayerDefinition

`(layer?: QueryLayerServiceInfoDc) => LayerDefinitionDc`

Извлекает `layerDefinition` из слоя с normalization атрибутов и fallback для пустого `attributes`.

---

### getLayerInfo

`(layer?: QueryLayerServiceInfoDc) => QueryLayerServiceInfoDc`

Нормализует объект `QueryLayerServiceInfoDc`: заполняет `configuration`, `layerDefinition`, `idAttribute`, `geometryAttribute`, `proxy` и т.д.

---

### getLayerInfoFromDataSources

`(layerInfos, dataSources, relatedDataSource) => QueryLayerServiceInfoDc | undefined`

Находит `layerInfo` по имени слоя из датасорса `relatedDataSource`.

---

### getPagesFromConfig

`(config: ConfigContainer) => ConfigContainerChild[]`

Возвращает список страниц: `config.children[0].children`.

---

### getPagesFromProjectInfo

`(projectInfo: ExtendedProjectInfoDc) => ConfigContainerChild[]`

`getPagesFromConfig(projectInfo.content.dashboardConfiguration)`.

---

### getRelatedAttribute

`(layerInfo, sourceAttributeName, relatedLayerName) => ConfigRelatedAttribute | undefined`

Находит связанный атрибут из конфига карточки слоя (`cardConfiguration`).

---

### getResourceUrl

`(url?: string) => string`

Если URL начинается с `http` — возвращает как есть. Иначе добавляет префикс `/sp/resources/file/`. Пустая строка при `!url`.

---

### getRootElementId

`(type?: WidgetType) => string`

`"${type}-root"` — id корневого элемента для экспорта в PDF.

---

### getSelectedFilterValue

`(filterName, selectedFilters, defaultValue) => SelectedFilter["value"]`

Возвращает текущее значение фильтра (или `defaultValue`). Нормализует single-value в массив при `Array.isArray(defaultValue)`.

---

### getSlideshowImages

`({ element, attribute }) => string[]`

Возвращает массив URL изображений: из `element.value`, `attribute.value.split(separator)` или `element.defaultValue`.

---

### getSvgUrl

`({ elementConfig, layerInfo, attributes }) => string | null`

Получает URL SVG из `attributeIcon` → иконки слоя, или из `attributeName` → значения атрибута, или из `elementConfig.value`. Применяет `getResourceUrl`.

---

### getTemplateNameFromAttribute

`(attribute: ClientFeatureAttribute) => ContainerTemplate`

Маппинг `AttributeType` на `ContainerTemplate` для Edit-контейнеров: `Boolean` → `EditBoolean`, `Int32/Int64/Double` → `EditNumber`, `DateTime` → `EditDate`, иначе → `EditString`.

---

### getTotalFromAttributes

`(children, attributes) => number | string`

Суммирует числовые значения атрибутов из `attributes` по `attributeName` в дочерних. Возвращает `.toFixed(0)`.

---

### getTotalFromRelatedFeatures

`(data: FilterItem[]) => number | string`

Суммирует `value` из массива `FilterItem`. Возвращает `.toFixed(0)`.

---

## Форматирование

### formatChartRelatedValue

`(t, value, layerInfo, relatedAttributes) => string | number`

Форматирует значение чарта с учётом типа атрибута Y-оси через `formatAttributeValue`.

---

### formatConditionValue

`({ value, attributeType, defaultValue, checkQuotes?, isSetParams? }) => string`

Форматирует значение фильтра для вставки в EQL-условие. Оборачивает строки в кавычки, форматирует массивы как `[v1,v2,v3]`, даты как `#'...'`.

---

### formatDataSourceCondition

`({ condition, configFilters, filters, attributes, geometry, layerParams?, eqlParameters? }) => string`

Основная утилита подстановки фильтров в EQL-условие. Обрабатывает `$(params)` секции и основную часть условия через `applyVarsToCondition`. Заменяет `%filterName`, `%filterName.min`, `%filterName.max`, `%geometry`, `{attributeName}`.

---

### formatElementValue

Форматирует значение элемента для отображения (с учётом `stringFormat`, типа атрибута).

---

## Создание

### createConfigLayer

`(layerName: string) => ConfigLayer`

Создаёт `ConfigLayer` с дефолтными значениями: `opacity: 1`, `isVisible: true`, `selectable: false`, `filterZoomTo: false`.

---

### createConfigPage

`(props?: CreateConfigPageProps) => ConfigContainerChild`

Создаёт новую страницу с ID `page_${n}`, пустыми `children/layers/dataSources/filters/tasks`, дефолтными `position`, `resolution`, `baseMapName`.

---

### createNewPageId

`(pages: ConfigContainerChild[]) => number`

Вычисляет следующий свободный ID страницы (max существующего + 1).

---

## Проверки

### checkEqualOrIncludes

`<T>(arrayOrSingle: T | T[], value: T) => boolean`

`Array.includes` если массив, иначе `===`.

---

### checkIsLoading

`(dataSources, config, filters) => boolean`

Проверяет, загружаются ли источники данных для фильтров контейнера.

---

### isEmptyElementValue

`(value?: unknown) => boolean`

`value === "" || value === null || value === undefined`

---

### isEmptyValue

Аналог `isEmptyElementValue`.

---

### isHiddenEmptyValue

`({ value, children, hideEmpty, renderElement }) => boolean`

Проверяет, нужно ли скрыть элемент (`hideEmpty: true`) при пустом значении. Учитывает дочерний `id: "value"`.

---

### isNotValidSelectedTab

`(tabId: string, selectedTabId: string) => boolean`

Возвращает `true` если `selectedTabId` не входит в `tabId` (поддерживает массив).

---

### isVisibleContainer

`(id, expandable, expanded, expandedContainers) => boolean`

Если не expandable → всегда `true`. Иначе смотрит в `expandedContainers[id]`, fallback на `expanded`.

---

## Операции с источниками данных

### addDataSource

`(config, pageIndex, query, additional) => ConfigContainer["children"]`

Добавляет EQL-датасорс на страницу. Автоматически присваивает `name: "datasource_N"`.

---

### addDataSources

`(config, pageIndex, layerNames) => ConfigContainer["children"]`

Добавляет layer-датасорсы для массива `layerNames`.

---

### applyQueryFilters

`({ parameters, filters, selectedFilters, dataSources, geometry }) => Record<string, any>`

Заменяет `%filterName` в `parameters` на значения фильтров. Поддерживает `.min`, `.max`, `.property`, геометрию.

---

### eqlParametersToPayload

`(parameters: ConfigDataSource["parameters"]) => Record<string, any>`

Преобразует EQL-параметры (`{ key: { default: value } }`) в плоский объект.

---

### removeDataSource

`(config, name, pageIndex) => ConfigContainer`

Удаляет датасорс по имени из конфига страницы и глобального конфига.

---

### updateDataSource

`(config, name, pageIndex, data) => ConfigContainer`

Обновляет поля датасорса (`Partial<Omit<ConfigDataSource, "name">>`) в конфиге страницы.

---

## Прочие

### pieChartTooltipFromAttributes

`(t, data: PieChartData[], attributes: FeatureAttribute[]) => string`

Форматирует строку тултипа PieChart из атрибутов объекта.

---

### roundTotalSum

`(value: number) => string | number`

`>= 1 000 000` → `"1.0M"`, `>= 10 000` → `"10.0K"`, иначе число без изменений.

---

### sliceShownOtherItems

`<T>(data: T, options: ConfigOptions, showMore?: boolean) => T`

Обрезает массив по `min(shownItems, otherItems)` если `!showMore`.

---

### tooltipNameFromAttributes

`(name: string, attributes: ClientFeatureAttribute[]) => string`

Возвращает `attribute.alias || name`.

---

### tooltipValueFromAttributes

`(t, value, name?, attributes?) => number | string | ReactNode`

Форматирует значение тултипа через `formatAttributeValue`.

---

### tooltipValueFromRelatedFeatures

`(t, value, relatedAttributes, layerInfo) => number | string | ReactNode`

Делегирует `formatChartRelatedValue`.

---

## Связанные разделы

[[hooks|Хуки]] | [[concepts|Основные понятия]] | [[containers|Контейнеры]] | [[elements|Элементы]]
