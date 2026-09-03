# Утилиты

## Обзор

Утилиты расположены в `D:/projects/spcore_api/packages/react/src/components/Dashboard/utils/`. Чистые функции для обработки, преобразования и проверки данных.

---

## Получение данных

### fetchQueryDescription

`(api: Api, { ds?, query?, parameters? }) => Promise<EqlDataSource["attributes"]>`

Запрашивает описание атрибутов EQL-запроса (`api.eql.getQueryDescription`) — им закрывается отсутствие слоя у EQL- и python-источников (см. [[concepts#Настройка атрибутов источника — секция attributes|секцию `attributes`]]).

Результат кэшируется по паре `ds` + `query` (без `parameters` — на состав атрибутов они не влияют) на `QUERY_DESCRIPTION_CACHE_TTL` = 60 000 мс: одна и та же страница обычно тянет описание для нескольких контейнеров разом. Кэшируется **промис**, поэтому параллельные вызовы делят один запрос; упавший запрос из кэша удаляется, чтобы следующий вызов попробовал снова.

---

### getAttributeByName

`(attributeName: string | string[], attributes?: ClientFeatureAttribute[]) => ClientFeatureAttribute | null | undefined`

Поиск атрибута по имени. Возвращает `null` для массива имён или пустого `attributeName`.

```ts
const attr = getAttributeByName("name", attributes);
```

---

### getAttributeConfigurationByName

`(attributes?: AttributesConfigurationDc["attributes"], name?: string) => AttributeConfigurationDc | undefined`

Находит конфигурацию атрибута по имени. `AttributesConfigurationDc["attributes"]` объявлен массивом, но часть источников отдаёт атрибуты объектом-мапой (нормализация в `useDataSources`), поэтому поддерживаются **обе формы** — массив ищется `find`, мапа читается по ключу.

---

### getAttributeIconElement

`(icon?: AttributeIconDc) => Pick<ConfigContainerChild, "type" | "value">`

Определяет, каким элементом рендерить иконку атрибута из настроек слоя: `Icon` → `icon` (значение — `iconName`, имя из библиотеки EverGIS), `PNG` → `image`, `SVG` → `svg` (значение — `resourceId || url`). Для `Unknown` и отсутствующего типа возвращает `type: undefined` — элемент не рендерится. Маппинг типов лежит в `constants.ts` (`ATTRIBUTE_ICON_ELEMENT_TYPES`). Используется в [[hooks#useRenderContainerItem|useRenderContainerItem]] для slot-id `icon`.

---

### getAttributeIconUrl

`({ attributeName, layerInfo }) => string | null`

Достаёт адрес картинки из **настроек** атрибута слоя: `attributesConfiguration.attributes[].icon` → `resourceId || url`. Обслуживает поле `attributeIcon` у [[elements#ElementSvg|ElementSvg]]; [[elements#ElementImage|ElementImage]] этот источник не использует. Форма `icon.iconName` (иконка из библиотеки EverGIS) не поддерживается — это не файл. Построена на `getAttributesConfiguration` + `getAttributeConfigurationByName`.

---

### getAttributesConfiguration

`(layer?: QueryLayerServiceInfoDc) => AttributesConfigurationDc`

Извлекает `attributesConfiguration` из конфига слоя с fallback на значения по умолчанию (`idAttribute`, `geometryAttribute`). Источники без слоя (EQL, python) резолвятся в `layerInfo` без поля `name` и не проходят `isLayerService`, поэтому дополнительно принимается любой объект с готовой конфигурацией атрибутов.

---

### getAttributeValue

`(element: ConfigContainerChild, attributes: ClientFeatureAttribute[]) => ReactNode | string`

Извлекает значение атрибута элемента для отображения. Boolean-атрибут → [[components|компонент]] `DashboardCheckbox`; массив `attributeName` → конкатенация значений через `separator`; объект → `JSON.stringify`; при заданном `options.maxLength` оборачивает в [[components|`TextTrim`]]. Используется в `getElementValue` для `type === "attributeValue"`.

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

Resolves контейнер из реестра — через `getContainerComponents()` (реестр строится лениво, см. [[types#Типизированные реестры|Типы]]). Если ключ не найден — возвращает `default` (ContainersGroupContainer). Если `innerTemplateName` пустой — `null`. Так `options.innerTemplateName` [[containers|контейнеров]] `DataSource`/`DataSourceProgress` превращается в проп `innerComponent` (шаблон рендеринга каждой записи источника); пустое значение → `null` → записи не рендерятся.

---

### isRootOwningContainer

`(templateName?: string) => boolean`

Контейнер этого шаблона сам держит `id`, `data-templatename`, авторский `style` и `$sizeCss` на своём единственном корне — внешняя обёртка `ElementValueWrapper` ему не нужна (иначе в DOM появится второй узел-близнец с теми же атрибутами). Проверяет принадлежность к набору [[types#ROOT_OWNING_TEMPLATES — контейнеры со своим корнем|`ROOT_OWNING_TEMPLATES`]]; результат уходит флагом `hasOwnRoot` в **formatElementValue**.

Неизвестный шаблон (`templateName` не найден в реестре) считается владельцем корня: он резолвится в реестровый `default` — `ContainersGroupContainer`, а тот корнем владеет. Пустой `templateName` — `false`. Живёт рядом с **getContainerComponent** ровно ради этого правила.

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

### getDataSourceLayerInfo

`({ layerInfos, configDataSource, fetchedDataSource }) => QueryLayerServiceInfoDc | null`

Резолвит `layerInfo` источника данных и накладывает поверх атрибуты из его конфига (см. [[concepts#Настройка атрибутов источника — секция attributes|секцию `attributes`]]). База — реальный слой по `layerName`, а если слоя нет (EQL, python) — атрибуты из ответа; наложение делегируется **mergeAttributeConfigurations**.

Возвращает `null`, когда атрибутов нет вообще: потребители отличают по этому «источник без описания атрибутов» от загруженного, и non-null сломал бы их loading/empty-state. Синтетическому `layerInfo` намеренно **не задаётся `name`** — по нему ищут скрытые атрибуты слоя и рендерят элемент `layerName`, имя источника дало бы ложные срабатывания. Если накладывать нечего, возвращается исходный объект слоя (идентичность сохраняется ради мемоизации).

---

### mergeAttributeConfigurations

`(base?: AttributesConfigurationDc["attributes"], overlay?: ConfigDataSourceAttribute[]) => AttributesConfigurationDc["attributes"]`

Накладывает атрибуты из конфига источника поверх атрибутов слоя или ответа EQL. `stringFormat` мержится **по полям** — в конфиге задают только переопределяемое, остальное (в т.ч. критичный для форматирования `type`) наследуется от базы. Если формата нет ни у одной из сторон — возвращается `undefined`, а не `{}`: пустой объект truthy и прошёл бы гейты `attribute?.stringFormat`, прогнав значение через форматирование чисел. Атрибут, которого нет ни у слоя, ни в ответе, добавляется с `isDisplayed: true` — иначе его отфильтрует `getFeatureAttributes`.

---

### getDefaultConfig

`({ title, defaultTitle, items, baseMapName, position, resolution, srid }) => ConfigContainer`

Создаёт полную конфигурацию дашборда по умолчанию с одной страницей и слоями из `items`.

---

### getElementValue

`({ type, config, elementConfig, renderElement, layerInfo, attributes, element, getDefaultContainer? }) => RenderElementValue`

Центральный диспетчер рендера элемента по полю `type`. Text/attribute-рендеры обрабатываются напрямую: `text` → `value`; `attributeAlias` → alias атрибута (с `TextTrim`); `attributeValue` → делегирует `getAttributeValue`; `attributeUnits` → `stringFormat.unitsLabel`; `attributeDescription` → описание атрибута из конфига слоя; `layerName` → `layerInfo.name`. Иначе ищет компонент в `elements/registry.ts` (`elementComponents[type]`) и рендерит его; при отсутствии — `getDefaultContainer()`. Используется внутри `getRenderElement`.

---

### getFeatureAttributes

`(feature, layer?, dataSource?) => ClientFeatureAttribute[]`

Преобразует `FeatureDc` + `QueryLayerServiceInfoDc` в массив `ClientFeatureAttribute` с alias, type, value, readOnly, stringFormat и т.д.

---

### getFeatureCardHeader

`(templateName: HeaderTemplate) => FC<ContainerProps>`

Возвращает компонент [[headers|шапки]] FeatureCard по `HeaderTemplate`: `Slideshow` → `FeatureCardSlideshowHeader`, `Background` → `FeatureCardBackgroundHeader`, `Default` (и fallback) → `FeatureCardDefaultHeader`. Парный к `getDashboardHeader`; используется в [[components|компоненте]] `FeatureCardHeader`.

---

### getFilterComponent

`(filterType: FilterType) => FC`

Возвращает React-компонент фильтра: `checkbox` → `CheckboxFilter`, `rangeNumber` → `RangeNumberFilter`, `barChart` → `BarChartFilter`, `rangeDate` → `RangeDateFilter`, `text` → `TextFilter`, `chips` → `ChipsFilter`, `tree` → `TreeFilter`, `dropdown` → `DropdownFilter` (default).

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

### getImageUrl

`({ elementConfig, attributes }) => string | null`

Резолвит адрес картинки [[elements#ElementImage|ElementImage]] — первый непустой источник выигрывает: `options.resourceId` → `options.url` → `value` → `attributeName`. Читается **значение** атрибута объекта, настройки атрибута в слое не задействованы. Все источники проходят через `getResourceUrl`; из значения атрибута берётся первый адрес до `;`. Для атрибута с `subType === Attachments` возвращает `null` — вложения грузятся отдельным каналом в `useElementImage`.

---

### getLayerInfo

`(layer?: QueryLayerServiceInfoDc) => QueryLayerServiceInfoDc`

Нормализует объект `QueryLayerServiceInfoDc`: заполняет `configuration`, `layerDefinition`, `idAttribute`, `geometryAttribute`, `proxy` и т.д.

---

### getLayerInfoAttribute

`(layerInfo?: QueryLayerServiceInfoDc, name?: string) => AttributeConfigurationDc | undefined`

Достаёт конфигурацию атрибута по имени прямо из `layerInfo`, снимая повторяющийся каст конфигурации. Тонкая обёртка над **getAttributeConfigurationByName**.

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

### getProjectValue

`({ prop?, projectName?, projectAlias? }) => string | undefined`

Значение системной подстановки текущего проекта (см. [[concepts#Текущий проект|Основные понятия]]). Голый `%project` и `%project.name` дают системное имя, `%project.alias` — алиас с фолбэком на имя. Чужое свойство возвращает `undefined`, чтобы плейсхолдер остался нетронутым. Используется и в **applyQueryFilters**, и в **formatDataSourceCondition**, а в client-new — хуком `useTempLayerParams` для параметров слоя.

---

### getRelatedAttribute

`(layerInfo, sourceAttributeName, relatedLayerName) => ConfigRelatedAttribute | undefined`

Находит связанный атрибут из конфига карточки слоя (`cardConfiguration`).

---

### getRenderElement

`(props: GetRenderElementProps) => RenderElementFunction`

Фабрика функции `renderElement({ id?, index?, wrap? })` для рендера дочерних элементов контейнера. Находит ребёнка по `id` (через `returnFound` из `find-and`) или по `index`; резолвит ссылки `containerId` на контейнер верхнего уровня; рекурсивно строит вложенный `renderElement`; делегирует значение `getElementValue`; скрывает пустые элементы (`isHiddenEmptyValue`) и форматирует результат через `formatElementValue`. Ключевая утилита registry-рендера — см. [[architecture#Поток данных|Поток данных]]. Парный хук — [[hooks|`useRenderElement`]].

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

Получает URL SVG — первый непустой источник выигрывает: `attributeIcon` (иконка из настроек атрибута слоя, через `getAttributeIconUrl`) → `attributeName` (значение атрибута, через `getAttributeByName`) → `value`. Применяет `getResourceUrl`.

---

### getTemplateNameFromAttribute

`(attribute: ClientFeatureAttribute) => ContainerTemplate`

Маппинг `AttributeType` на `ContainerTemplate` для Edit-контейнеров: `Boolean` → `EditBoolean`, `Int32/Int64/Double` → `EditNumber`, `DateTime` → `EditDate`, `Resource` → `EditAttachment`, иначе → `EditString`.

---

### getDisplayTemplateNameFromAttribute

`(attribute?: ClientFeatureAttribute) => ContainerTemplate | undefined`

Для display-режима (не edit): возвращает специальный `ContainerTemplate` для атрибута, если он требует особого рендера (например, `AttributeType.Resource` → `ContainerTemplate.Attachment`). Используется в [[hooks|хуке]] `useRenderContainer` для переопределения template дочернего контейнера в `OneColumn`/`TwoColumn`.

---

### getThemeByName

`(themeName?: ThemeName) => DefaultTheme`

Возвращает объект темы (`ITheme`) для `ThemeProvider`. Используется в шапках `FeatureCard*Header` для принудительной темы шапки независимо от глобальной.

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

`({ condition, configFilters, filters, attributes, geometry, extent?, zoomLevel?, projectName?, projectAlias?, layerParams?, eqlParameters? }) => string`

Основная утилита подстановки фильтров в EQL-условие. Обрабатывает `$(params)` секции и основную часть условия через `applyVarsToCondition`. Заменяет `%filterName`, `%filterName.min`, `%filterName.max`, `%geometry`, `%extent`, `%zoom`, `%project`, `{attributeName}`.

Системные фильтры карты (`%geometry`, `%extent`, `%zoom`, см. [[concepts#Системные фильтры карты|Основные понятия]]) подставляются **до** цикла по `configFilters` — одноимённый пользовательский фильтр их не перехватит. `%extent` уходит в кавычках, `%zoom` — числом без кавычек; составные формы (`%zoomLevel`, `%zoom.min`) системная замена не трогает.

Там же резолвится текущий проект (`%project`, `%project.name`, `%project.alias`, см. [[concepts#Текущий проект|Основные понятия]]) — значение строковое, поэтому уходит в кавычках. Свойства заменяются раньше голого плейсхолдера, значение берёт **getProjectValue**; `%project_id` и `%project.foo` замена не трогает.

---

### applyTreeFilterToCondition

`(condition: string, name: string, value: TreeFilterValue, isSingle?: boolean) => string`

Подставляет в EQL-условие плейсхолдеры уровней иерархического фильтра «tree» — `%name.l{N}` — значениями id соответствующего уровня (формат `[id1,id2,...]` для оператора `IN`). Уровни, отсутствующие в значении (или пустые), не подставляются — плейсхолдер остаётся нетронутым. Форматирование значений уровня делегирует **formatConditionValue**.

Type-guards самого значения живут в соседнем модуле — **filterValueKind** (см. ниже).

---

### filterValueKind

Два type-guard'а для объектных значений фильтра. Угадывать вид значения по форме нельзя: `tree` и `features` оба объекты — поэтому дискриминатором служит `ConfigFilter.valueType` (см. [[concepts|Основные понятия]], «Фильтры»).

| Функция | Сигнатура и назначение |
|---|---|
| `isFeaturesFilterValue` | `(value?) => value is FeaturesFilterValue` — значение `valueType: "features"` (строки контейнера [[containers\|`StructuredData`]]). Форма самодостаточна (`type: "FeatureCollection"` + массив `features`), поэтому конфиг для проверки не нужен |
| `isTreeFilterValue` | `(value?, configFilter?) => value is TreeFilterValue` — объектное значение иерархического фильтра «уровень → массив id». Решают два значения `valueType`: `"tree"` → да, `"features"` → нет |

Фолбэк `isTreeFilterValue` (когда `valueType` не задан) намеренно узкий — все ключи вида `l{N}`, все значения массивы. Он нужен существующим конфигам: у tree-фильтров там `valueType` либо отсутствует, либо объявлен как `"array"` (значение при этом всё равно объектное), и трактовать такой фильтр как не-tree значило бы сломать их. Пустой объект — это tree с пустым выбором, а не «неизвестное значение». `Date` отсекается явно.

---

### getMapViewDataSources

`(dataSources?: ConfigDataSource[]) => ConfigDataSource[]`

Источники, чей запрос зависит от текущего вида карты: в `parameters` или `condition` встречаются `%extent` / `%zoom`. При движении карты перезапрашивают только их — иначе каждый пан дёргал бы все запросы страницы. Граница имени та же, что при подстановке: `%zoomLevel` и `%extent_id` в выборку не попадают.

Потребитель — клиентский [[hooks|хук]] `useMapViewRefetch` (client-new).

---

### toConditionsArray

`(value?: string | string[]) => string[]`

Приводит `condition` источника (строка либо массив строк) к массиву для единообразного обхода.

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

## Размеры и стили

### getWrapperSizeStyle

`({ style, width, height, overflow, defaults, defaultWidth, heightAsMin }) => CSSObject | undefined`

Собирает CSS-объект корневой обёртки контейнера: внутренние дефолты плюс размеры из `options` (см. [[containers#Размерная модель обёртки ContainerBoxOptions|размерную модель]]). Результат уходит в styled-проп, а не в inline-style, поэтому перебивается снаружи обычной специфичностью — без `!important`. Опции перекрывают одноимённые поля авторского `style`.

Размер `"100%"` включает **fill-режим**: контейнер занимает ячейку целиком, для чего снимаются конфликтующие внутренние дефолты обёртки (`width`/`minWidth`/`maxWidth`/`marginLeft`/`marginRight` по горизонтали, `height`/`minHeight`/`maxHeight`/`marginTop`/`marginBottom` по вертикали), а контент лишается возможности её распирать (`min-width`/`min-height: 0`). Для fill-высоты добавляется `flex: 1 1 auto` — в колонке контейнер забирает остаток ячейки под заголовком, а не переполняет её на его высоту. `overflow` уходит в CSS как есть и ничем не подменяется.

`heightAsMin` (приходит из [[options#ConfigLayoutOptions|`options.autoHeight`]]) переводит высоту в `min-height`: узел не опускается ниже неё, но перерастает под содержимое. Парного `min-height: 0` в fill-режиме при этом нет — он немедленно погасил бы сам минимум, ради которого режим и включают.

Рядом экспортируются константа `FILL_SIZE` (`"100%"`), предикат `isFillSize(size)` и предикат `isFrSize(size)` — распознаёт долю трека сетки (`"2fr"`, `"1.5fr"`). Доля на самом узле трактуется как fill: применяет её родитель, собирая `grid-template-*` (см. **buildGridTemplate**). Парный хук — [[hooks|`useWrapperSize`]] (он передаёт `defaultWidth: FILL_SIZE`, поэтому контейнеры по умолчанию занимают всю ширину ячейки, а элементы — нет).

---

### toCssSize

`(size?: CssSize) => string | undefined`

Приводит размер из конфига к CSS-значению: число → пиксели (`"12px"`), строка → как есть. Нужен там, где значение подставляется в CSS-строку (`width: ${...}`) — в отличие от inline-style и css-объекта styled-components, которые добавляют `px` к числам сами.

Рядом — `toPxNumber(size)`: пиксельное число для API, которым нужен именно `number` (геометрия графиков). Относительные значения (`"100%"`, `"10rem"`) числом не выражаются и дают `undefined`, чтобы вызывающий код взял свой дефолт.

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

### hasContainerBgImage

`(elementConfig?: Pick<ConfigContainerChild, "children">) => boolean`

Есть ли у узла конфига слот фонового изображения — ребёнок с `id: "bgImage"` (`BG_IMAGE_SLOT_ID`).

Слой фона рисуется **только** по этому признаку: без узла `bgImage` в конфиге пустой абсолютно позиционированный `div` в DOM не появляется вовсе. Тот же признак поднимает корень контейнера в хост слоя (`$hasBgImage` → `bgImageHostMixin`). Потребители — [[components|`ContainerBackground`]], [[hooks|`useWrapperSize`]] и [[hooks|`useBgImageHost`]]. Механика целиком — в [[concepts#Универсальные слоты и фон контейнера|Основных понятиях]].

---

### isCrossOriginUrl

`(url?: string | null) => boolean`

Ведёт ли адрес на **чужой** origin — сервер, который ничего не знает ни про наш токен, ни про наши заголовки. Относительный адрес (`/sp/resources/file/<id>`) всегда свой; абсолютный сравнивается с `location.origin`, поэтому `https://<наш хост>/sp/...` тоже считается своим.

Проверка не сводится к `^https?://`, как флаг `isExternal` у вложений: там достаточно отличить ссылку от файла каталога, а здесь решается, отправлять ли `Authorization`. Разобрать адрес не удалось — считаем его своим, пусть с ним разбирается обычный путь загрузки. Потребители — [[hooks|`useFetchWithAuth`]] и [[hooks|`useFetchImageWithAuth`]].

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

`({ parameters, filters, selectedFilters, geometry, extent?, zoomLevel?, projectName?, projectAlias?, attributes?, layerInfo?, dataSources, projectDataSources? }) => Record<string, any>`

Резолвит значения `parameters` (EQL-параметров или параметров python-скрипта) из нескольких источников:

- `%filterName` → значение фильтра; поддерживает `.min`, `.max`, `.property`, а также системные `%geometry`, `%extent` (строка EWKT) и `%zoom` (**число**);
- `%project`, `%project.name`, `%project.alias` → имя и алиас открытого проекта (см. [[concepts#Текущий проект|Основные понятия]]); значение берёт **getProjectValue**, при отсутствии значения ключ выпадает из результата;
- `$card:layerName:fieldName` → значение атрибута текущего объекта FeatureCard (из `attributes`, если `layerInfo.name === layerName`);
- `$left:layerName:fieldName` → значение из первого feature `projectDataSources` по имени слоя;
- `{attributeName}` → подстановка/интерполяция значений атрибутов объекта.

Используется в источниках данных, а также билдером [[hooks|хука]] `useSavePrototypeBuilder` для сборки параметров `beforeSave`/`afterSave` скриптов.

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

## Утилиты AttachmentContainer

Локальные в `containers/AttachmentContainer/utils/`:

### parseAttachments

`(rawValue: unknown) => Attachment[]`

Парсит сырое значение атрибута (строка, массив, JSON) в массив `Attachment` (`{ link, name, mimeType, uploadedAt, isExternal }`).

---

### attachmentsFromFeatures

`(features?: FeatureDc[], mapping?: { attributeLink?, attributeName?, attributeMime?, attributeDate? }) => Attachment[]`

Извлекает вложения из features связанного источника данных согласно маппингу полей.

---

### getFileType

`(mimeType?: string, fileName?: string) => FileType`

Определяет `FileType` (`XLSX`, `PDF`, `CSV`, `PNG`, ...) из MIME-типа или расширения файла.

---

### getFileTypeIcon

`(fileType: FileType) => string`

Возвращает URL иконки-заглушки по типу файла (для preview, когда сам файл не загружен или не является изображением).

---

### getMimeTypeFromUrl

`(url: string) => string`

Определяет MIME-тип по расширению файла в URL (с отбрасыванием query/hash). Возвращает пустую строку, если расширение неизвестно или отсутствует. Рядом экспортируется `getFileNameFromUrl(url)` — извлекает имя файла из URL (через `new URL`, с fallback на последний сегмент пути).

---

### saveBlobAsFile

`(blob: Blob, fileName: string) => void`

Отдаёт браузеру уже загруженный файл под именем `fileName`: создаёт object URL, кликает по скрытой ссылке с атрибутом `download` и убирает её из DOM. Нужна вложениям за авторизацией — их не отдать простой ссылкой, файл приходит через `api.catalog.getFile` (см. [[hooks|`useAttachmentDownload`]]).

Адрес отзывается не сразу, а через секунду: часть браузеров дочитывает поток уже после клика, и мгновенный `revokeObjectURL` обрывает сохранение.

---

## Утилиты сетки (grid)

Утилиты [[containers#Режим сетки grid|режима сетки]]. Часть отдаётся наружу через `grid/index.ts` — только листовые модули (`gridTracks`, `gridTree`, `createGridNodeId`): баррель `utils` оттуда не тянут, иначе цикл `getRenderElement → registry → контейнеры → grid` уронит инициализацию в TDZ. Операции раскладки (`gridOperations`) и DOM-чтение остаются внутренними. Типы и константы — [[types#Публичная поверхность сетки|Типы]].

### Треки (`gridTracks`)

| Функция | Сигнатура и назначение |
|---|---|
| `getLayoutChildren` | `(node?) => ConfigContainerChild[]` — дети, которые реально становятся треками: универсальные слоты (`NON_TRACK_SLOT_IDS` — `title`, `titleIcon`, `bgImage`) исключаются. `ContainerChildren` их не рендерит, и в `grid-template-*` их быть не должно — иначе треков окажется больше, чем ячеек, и раскладка съедет |
| `getTrackSizeKey` | `(axis) => "height" \| "width"` — в какой опции лежит доля трека: у строки — высота, у ячейки — ширина |
| `parseFrValue` | `(size?) => number` — доля числом. Значения не в `fr` (px, проценты, `auto`) сеткой не поддерживаются и считаются за `1fr`: смешение фиксированных и резиновых треков сломало бы инвариант ресайза «сумма долей пары неизменна» |
| `getTrackSizes` | `(children, axis) => number[]` — доли всех треков одного родителя в порядке следования |
| `roundFr` / `toFrSize` | `(value: number) => number` / `=> string` — округление до трёх знаков и сборка `"1.5fr"`. Без округления в конфиг попадают хвосты вида `1.0000000000000002fr` |
| `buildTrackTemplate` | `(sizes: string[], autoMin?) => string` — значение `grid-template-*` из готовых размеров. `minmax(0, X)` вместо голого размера: у голого трека неявный минимум `auto`, и широкий контент (таблица, график) распирает его изнутри, ломая пропорции. `autoMin` возвращает этот минимум — ровно то, ради чего включают `autoHeight` |
| `buildGridTemplate` | `(children, axis, autoHeight?) => string` — то же по долям детей. `autoHeight` отпускает минимум только у **строк**: ширину растить нечем, а `auto`-минимум колонки просто сломал бы пропорции |

### Дерево (`gridTree`)

| Функция | Сигнатура и назначение |
|---|---|
| `isGridNode` | `(node?) => boolean` — узел-сетка: `ContainersGroup` с `options.grid` |
| `setLayoutChildren` | `(node, children) => ConfigContainerChild` — подменяет «раскладочных» детей, сохраняя универсальные слоты заголовка и фона (кладутся первыми — их порядок относительно тела ни на что не влияет: тело их отфильтровывает, а `ExpandableTitle` и `ContainerBackground` ищут по `id`) |
| `mapNodeById` | `(node, id, mapper) => ConfigContainerChild` — иммутабельно применяет `mapper` к узлу с заданным `id` где угодно в поддереве; `null` из маппера удаляет узел. Незатронутые ветки возвращаются по прежней ссылке, поэтому `memo`-контейнеры не перерисовываются. Пустой `id` — no-op: иначе `undefined === undefined` совпало бы с корнем и правка ушла бы не туда |
| `containsNodeId` / `replaceNodesByIds` | Поиск id в поддереве и пакетная замена узлов |
| `findCellContext` | `(grid, cellId) => GridCellContext \| null` — ищет ячейку по всему дереву сеток, включая вложенные; возвращает ближайший к ней контекст (`grid`, `row`, `rows`, `cells`, `rowIndex`, `cellIndex`) |
| `getTrackSize` / `withTrackSize` | Прочитать и задать долю трека узла |
| `getAverageTrackSize` | `(tracks, axis) => number` — средняя доля: база для вновь добавляемого трека, чтобы он не выбивался из масштаба |
| `removeTracks` | `(tracks, indexes, axis) => ConfigContainerChild[]` — убирает треки, отдавая их доли соседу слева (у первого — соседу справа). Без передачи доли `fr` перенормируются сами, и вместе с удалённым треком визуально сместятся все остальные границы |
| `createGridCell` / `createGridRow` | `(factory, ...) => ConfigContainerChild` — пустая ячейка (`ContainersGroup` с `width`) и строка (`GridRow` с `height`, по умолчанию с одной пустой ячейкой) |

### Идентификаторы (`createGridNodeId`)

| Функция | Сигнатура и назначение |
|---|---|
| `collectConfigIds` | `(source: unknown, acc?) => Set<string>` — все `id` в поддереве конфига. Обходит объект целиком, а не только `children`: узлы встречаются и в других коллекциях (`modals`, вложенные структуры), и `returnFound` из `find-and` ищет так же. Дубликат id заставил бы движок молча взять первое совпадение |
| `createGridIdFactory` | `(usedIds: Set<string>) => GridIdFactory` — фабрика уникальных id со сквозной нумерацией (`gridRow_1`, `gridCell_1`). Счётчик общий на страницу: сеток на ней может быть несколько, и локальные счётчики выдали бы им одинаковые id |

### Операции и DOM (внутренние)

| Модуль | Содержимое |
|---|---|
| `gridOperations` | `normalizeGrid`, `deleteCells`, `canMergeCells` / `mergeCells`, `canSwapCells` / `swapCells`, `getGridMenuState`, `applyGridAction` — применение [[types#Публичная поверхность сетки\|`GridEditAction`]] к черновику. Семантика операций — [[containers#Редактирование раскладки editMode\|Редактирование раскладки]] |
| `gridResizeOperations` | Пересчёт долей пары треков и высоты корня по итогу жеста |
| `readTrackPixels` | `(grid, axis) => number[]` — пиксельные размеры треков из computed `grid-template-*`, а не из прямоугольников ячеек: у отрисованного грида браузер отдаёт уже разрешённые used values, без зазоров |
| `readCellAtPoint` | Ячейка под курсором по маркеру `data-grid-cell` — цель перетаскивания |

---

## Прочие

### pieChartTooltipFromAttributes

`(t, data: PieChartData[], attributes: FeatureAttribute[]) => string`

Форматирует строку тултипа PieChart из атрибутов объекта.

---

### pieChartTooltipFromRelatedFeatures

`(t, data: PieChartData[], relatedAttributes, layerInfo) => string`

Форматирует строку тултипа PieChart из features связанного источника данных (аналог `pieChartTooltipFromAttributes` для related-features-сценария).

---

### roundTotalSum

`(value: number, fractionDigits?: number) => string | number`

`>= 1 000 000` → `"1.0M"`, `>= 10 000` → `"10.0K"`, иначе число без изменений; `0`/пустое значение → `""`. Число знаков после запятой — `fractionDigits`, по умолчанию `COMPACT_FRACTION_DIGITS` (`1`). Лежит не в `Dashboard/utils/`, а в общем `packages/react/src/utils/` — используется компонентом [[components|`Chart`]] для итога в центре PieChart и форматтером `formatAttributeValue` (там `fractionDigits` приходит из `stringFormat.rounding`).

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

### toRenderableValue

`(value: unknown) => ReactNode`

Возвращает безопасное для рендера в React значение. Если на вход пришёл не-примитивный объект (массив или plain-object), не являющийся React-элементом, — возвращает пустую строку, чтобы избежать runtime-ошибки «Objects are not valid as a React child». React-элементы и примитивы проходят без изменений. Нужно там, где слот получает сырое значение атрибута, которое может быть структурированным payload (например, атрибут с `subType: Attachments`, чьё значение — `Attachment[]`).

---

## Связанные разделы

[[hooks|Хуки]] | [[concepts|Основные понятия]] | [[containers|Контейнеры]] | [[elements|Элементы]] | [[types|Типы]]
