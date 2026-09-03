# Основные понятия

> [!danger] У каждого узла конфига обязателен `id`
> И как `slot` (у элементов), и как уникальный ключ (у контейнеров и перечисляемых сущностей). Узел без `id` молча не рендерится; типы это не ловят. Подробно — ниже в разделе «[[concepts#ID контейнеров и элементов|ID контейнеров и элементов]]», кратко с чек-листом — на странице [[authoring|Правила генерации]].

## Страницы

**Страница** — это независимая единица отображения в дашборде, содержащая собственный набор слоёв, источников данных, фильтров и контейнеров. Технически — `ConfigContainerChild` с `templateName: ContainerTemplate.ContainersGroup`, хранящийся в `config.children[0].children`.

Каждая страница изолирована: при переключении страницы меняются слои на карте, загружаются свои источники данных, применяются свои фильтры. Например, первая страница показывает жилые объекты с фильтрами по этажности и году постройки, вторая — коммерческие объекты с фильтрами по площади.

```json
{
  "id": "page_1",
  "templateName": "ContainersGroup",
  "dataSources": [{ "name": "buildings", "layerName": "buildings_layer" }],
  "filters": [{ "name": "floors", "defaultValue": [], "valueType": "array" }],
  "layers": [{ "name": "buildings_layer", "isVisible": true }],
  "children": [...]
}
```

**Навигация:** `pageIndex` (0-based) — текущая страница. Методы из [[setup|DashboardContextProps]]: `nextPage(total)`, `prevPage(total)`, `changePage(index)`. Хук: [[hooks|хук]] `useWidgetPage` — возвращает `pageIndex`, `currentPage` (с объединёнными dataSources/filters из конфига), методы обновления конфига страницы.

Связь страницы с табом — поле `options.tabId` ребёнка `PagesContainer` (тип `PageChildOptions`, см. [[types|Типы]]).

Создание новой страницы: [[utils|утилита]] `createConfigPage(props)` → `ConfigContainerChild` с ID формата `page_${n}`.

---

## Источники данных

**Источник данных** (`ConfigDataSource`) — это описание запроса, результат которого передаётся в контейнеры для отображения. Когда открывается страница со списком зданий, каждый источник данных — это один запрос: «дай мне все здания в этом квартале» или «дай мне топ-10 зданий по высоте».

Поддерживаемые типы запросов:

| Тип | Поля | Описание |
|---|---|---|
| **Layer features** | `layerName` | Объекты из карт-слоя |
| **EQL-запрос** | `query`, `ds`, `condition`, `parameters` | Запрос к датасету через EQL |
| **Python task** | `resourceId` | Запуск Python-скрипта, получение результата |
| **URL endpoint** | `url` | GET-запрос к внешнему эндпоинту |

```json
{
  "name": "expensive_apartments",
  "query": "SELECT * FROM real_estate",
  "ds": "real_estate_ds",
  "condition": "price > 10000000 AND floors >= %minFloors",
  "parameters": { "limit": { "default": 50 } }
}
```

Результат загрузки — `WidgetDataSource`: `{ name, features, layerName?, attributes? }`, где `attributes` — описание атрибутов из ответа (`AttributesConfigurationDc["attributes"]`) для источников без слоя (EQL, python).

**Умная инвалидация:** при изменении значения фильтра система вычисляет список изменившихся фильтров → `getUpdatingDataSources()` возвращает только те источники, в `condition` или `parameters` которых есть ссылка `%filterName` → перезагружаются только они. Остальные источники не тронуты.

Хук: [[hooks|хук]] `useDataSources`. Имя источника типизируется branded-типом [[types#Branded types|DataSourceName]] (`asDataSourceName`).

**Задержка запроса — `debounce`.** Необязательное поле источника (мс, по умолчанию `0` — запрос сразу). Дебаунсит **любой** перезапрос этого источника: смену фильтров, движение карты (`%extent`/`%zoom`), autoSync-уведомления, правку конфига и первичную загрузку. Серия быстрых событий схлопывается в один сетевой вызов — нужно тяжёлым источникам.

```json
{ "name": "heavy_stats", "layerName": "deals", "condition": "ST_Intersects(geom, %extent)", "debounce": 500 }
```

Во время задержки контейнер показывает прежние данные, а не спиннер: `setProjectDataSourcesAreLoading` взводится уже при отправке запроса. Источник, которого ещё нет в сторе, всё это время рисует свой `ContainerLoading`.

### Настройка атрибутов источника — секция `attributes`

Иногда атрибуты приходят «сырыми»: у EQL- и python-источника слоя нет вообще, а у слоя формат может не совпадать с тем, как значение нужно показать в конкретном дашборде. Для этого у источника есть секция `attributes` (`ConfigDataSourceAttribute[]`) — она **накладывается поверх** атрибутов слоя или ответа запроса.

```json
{
  "name": "deals_ds",
  "query": "SELECT district, amount FROM deals",
  "ds": "analytics",
  "attributes": [
    { "attributeName": "amount", "alias": "Сумма сделок", "stringFormat": { "unitsLabel": "млн ₽", "decimals": 1 } }
  ]
}
```

Правила наложения ([[utils|`mergeAttributeConfigurations`]] внутри [[utils|`getDataSourceLayerInfo`]]):

- **`stringFormat` мержится по полям** — в конфиге задаётся только переопределяемое, остальное (в том числе критичный для форматирования `type`) наследуется от базы. Если формата нет ни у одной из сторон, поле остаётся `undefined`, а не пустым объектом: `{}` прошёл бы гейты `attribute?.stringFormat` и прогнал значение через форматирование чисел.
- **Атрибут, которого нет ни у слоя, ни в ответе, добавляется** с `isDisplayed: true` — иначе его отфильтрует `getFeatureAttributes`.
- Для источника **без слоя** база берётся из `attributes` ответа, и собирается «синтетический» `layerInfo`. Ему намеренно не задаётся `name`: по имени слоя ищут скрытые атрибуты проекта и рендерят элемент `layerName` — имя источника дало бы там ложные срабатывания.
- Если накладывать нечего, возвращается исходный объект слоя — идентичность сохраняется ради мемоизации.

> Не путать с `options.attributes` (`string[]`) у `OneColumn`/`TwoColumn` — там это список **имён** атрибутов для отображения, а здесь — их метаданные и формат. См. [[types#Типы источника данных|Типы]].

### Рендеринг записей источника — `innerTemplateName`

Контейнеры `DataSource` и `DataSourceProgress` (см. [[containers|Контейнеры]]) не рендерят детей напрямую: они проходят по **каждой записи** (`feature`) источника и рендерят её через **внутренний шаблон**, заданный `options.innerTemplateName`. Пайплайн рендера ([[utils|`getRenderElement`]] → [[utils|`getContainerComponent`]]) конвертирует это имя в проп `innerComponent`, который `DataSourceInnerContainer` применяет к каждой записи, подставляя её атрибуты.

- **`innerTemplateName` обязателен.** Без него `getContainerComponent(undefined) === null` → `innerComponent` не передан → `DataSourceInnerContainer` возвращает `null` → записи не рендерятся, контейнер визуально пуст. Опция объявлена в `ConfigMiscOptions` (см. [[options|Опции]]) и остаётся необязательной по типу, поэтому обязательность **типами не ловится**.
- **`children` DataSource-хоста — это slot-id выбранного внутреннего шаблона** (`RoundedBackground`/`Progress` → `icon`/`alias`/`value`/`units`; `OneColumn`/`TwoColumn` → `alias`/`value`/`units`), а не собственные слоты хоста.

```json
{
  "id": "buildings_list",
  "templateName": "DataSource",
  "options": { "relatedDataSource": "buildings_ds", "innerTemplateName": "RoundedBackground" },
  "children": [
    { "id": "icon", "type": "icon", "options": { "icon": "building" } },
    { "id": "alias", "attributeName": "name" },
    { "id": "value", "attributeName": "floors" }
  ]
}
```

---

## Фильтры

**Фильтр** — это именованный параметр, значение которого пользователь задаёт через UI, а система подставляет его в EQL-условие источника данных. Например, фильтр `floors` с диапазоном 5–15 ограничивает показ объектов недвижимости по этажности: только здания от 5 до 15 этажей попадают в датасорс и отображаются на карте и в списке.

```json
{
  "name": "floors",
  "defaultValue": [1, 30],
  "valueType": "range"
}
```

В источнике данных условие: `condition: "floors BETWEEN %floors.min AND %floors.max"`.

Типы значений (`valueType`):

| Значение | Описание | Пример значения |
|---|---|---|
| `single` | Одно значение | `"Жилой"` |
| `range` | Диапазон min/max | `[5, 15]` |
| `array` | Массив выбранных значений | `["Жилой", "Коммерческий"]` |

**SelectedFilters** — текущие значения всех фильтров страницы:
```ts
type SelectedFilters = Record<string, SelectedFilter>;

interface SelectedFilter {
  value: string | number | string[] | number[] | Date | Date[] | TreeFilterValue;
  min?: string | number | Date;
  max?: string | number | Date;
}
```

Объектный вариант `TreeFilterValue` — только у иерархического фильтра «tree» (см. ниже); остальные фильтры хранят скалярное или массивное значение (`ScalarFilterValue`).

**ConfigFilter** — описание фильтра в конфиге страницы: `name`, `defaultValue`, `valueType`, `relatedDataSource` (откуда брать список вариантов), `resetFilters` (сбрасываемые при изменении фильтры). Имя фильтра типизируется branded-типом [[types#Branded types|FilterName]] (`asFilterName`).

> [!warning] `defaultValue` не попадает в состояние фильтров
> Состояние выбранных фильтров (`filters` виджет-контекста) стартует пустым и наполняется только выбором пользователя — заливки дефолтов из конфига в него нет. `defaultValue` живёт исключительно в конфиге страницы, поэтому фолбэк `filters[name]?.value ?? configFilter?.defaultValue` делает каждый потребитель фильтра сам: и компоненты фильтров, и подстановка в условия источников, и контейнеры, которые читают значение фильтра напрямую (например, [[containers#StructuredDataContainer|StructuredData]]).

**Тип контрола** (`FilterType`) — каким UI-виджетом рисуется фильтр: `"checkbox"`, `"rangeNumber"`, `"rangeDate"`, `"text"`, `"dropdown"`, `"barChart"`, `"chips"`, `"tree"`.

**Иерархический фильтр `"tree"`:** для справочников «уровень → подуровень» (регион → район → населённый пункт). В дополнение к базовым полям **ConfigFilter** задаёт атрибуты дерева:

| Поле | Назначение |
|---|---|
| `attributeValue` | атрибут хранимого значения — то, что фильтр отдаёт потребителям; для дерева служит идентичностью узла (`TreeItemProps.id`) |
| `attributeName` | атрибут-имя записи — служебная связь дерева (`child.attributeParentName == parent.attributeName`) |
| `attributeParentName` | атрибут-имя родителя — связывает узел с родительским |
| `attributeLevel` | атрибут числового уровня узла (корень = 1) |
| `attributeHasChildren` | атрибут-флаг наличия детей (показ шеврона раскрытия) |
| `limit` | единый лимит на запросы дерева (дети уровня, поиск) |

Значение tree-фильтра — `TreeFilterValue`: объект «уровень → массив id», ключи вида `l{N}` (`l1`, `l3`, ...); уровни без выбранных элементов не включаются. Например, выбор двух регионов и одного города даёт `{ l1: [1, 2], l3: [1023] }`. Это объектный вариант `SelectedFilter["value"]` (отличается от скалярных/массивных значений — `ScalarFilterValue`).

```json
{
  "name": "territory",
  "valueType": "array",
  "attributeValue": "id",
  "attributeName": "code",
  "attributeParentName": "parent_code",
  "attributeLevel": "level",
  "attributeHasChildren": "hasChildren",
  "limit": 100
}
```

Подстановка tree-значения в условие — [[utils|утилита]] `applyTreeFilterToCondition`: заменяет плейсхолдеры уровней `%name.l{N}` на список id (`[id1,id2,...]` для оператора `IN`). Пустые/отсутствующие уровни не подставляются. Вызывается из `applyFiltersToCondition` (которую агрегирует `formatDataSourceCondition`) для значений, прошедших проверку `isTreeFilterValue`.

### Системные фильтры карты

Три имени зарезервированы: их значения приходят не из `SelectedFilters`, а из [[setup|GlobalContext]], и подставляются до пользовательских фильтров. Одноимённый фильтр в конфиге будет перехвачен системной подстановкой.

| Плейсхолдер | Источник в GlobalContext | Что подставляется | Меняется |
|---|---|---|---|
| `%geometry` | `ewktGeometry` | `'SRID=3857;POLYGON(...)'` — область, **нарисованная пользователем** (polygon, bbox, зоны с буфером) | по завершении рисования |
| `%extent` | `ewktExtent` | `'SRID=3857;POLYGON(...)'` — прямоугольник **видимой области карты** | по событию `idle` карты (пан, зум) |
| `%zoom` | `zoomLevel` | целое число без кавычек, `Math.round(map.getZoom())` | там же |

Работают и в `condition`, и в `parameters` (включая секции `$(param=...)`). В `parameters` `%zoom` уходит **числом**, а не строкой — единственная нестроковая системная подстановка.

Граница имени: составные формы достаются пользовательским фильтрам — `%zoomLevel`, `%extent_id`, `%zoom.min` системная замена не трогает.

```json
{
  "name": "visible_buildings",
  "layerName": "buildings",
  "condition": "ST_Intersects(geom, %extent) AND detail_level <= %zoom",
  "debounce": 500
}
```

**Перезапрос по движению карты.** Изменение вида карты перезапрашивает **только** источники, у которых `%extent`/`%zoom` реально встречаются в `condition` или `parameters` — их отбирает [[utils|утилита]] `getMapViewDataSources`. Стор при этом не сбрасывается, поэтому соседние контейнеры не мигают «Блок не загружен». Геометрический фильтр устроен иначе: смена `ewktGeometry` перезагружает страницу целиком со сбросом стора.

**Слои карты.** Те же три плейсхолдера работают в `query` и `parameters` слоя (`ConfigLayer`) — их резолвят хуки client-new `useTempLayerConditions` и `useTempLayerParams`, читая значения из того же `GlobalContext`. Результат уходит в серверный фильтр слоя (`api.filters.create` / `update`), после чего тайлы перерисовываются. Отдельная опция `debounce` слою не нужна: создание фильтра уже дебаунсится на 150 мс в `useLayerFilterQuery`, а `idle` срабатывает только после остановки карты.

Пример `%geometry`: пользователь рисует прямоугольник на карте → все источники данных с `%geometry` в условии перезапрашиваются для выбранного района.

### Текущий проект

Ещё одно зарезервированное имя — `project`. Значения приходят из [[setup|GlobalContext]] (`projectName`, `projectAlias` — из описания открытого проекта) и подставляются так же, как системные значения карты: до цикла по пользовательским фильтрам, поэтому одноимённый фильтр конфига будет перехвачен.

| Плейсхолдер | Источник в GlobalContext | Что подставляется |
|---|---|---|
| `%project` | `projectName` | системное имя открытого проекта |
| `%project.name` | `projectName` | то же самое, полная форма |
| `%project.alias` | `projectAlias` | алиас проекта; при пустом алиасе — системное имя |

Работают и в `condition`, и в `parameters` (включая секции `$(param=...)`), а также в `query` и `parameters` слоя карты. Значение строковое — в условии уходит в кавычках, как `%geometry`.

Граница имени та же, что у значений карты: `%project_id` и `%projects` достаются пользовательским фильтрам. Чужое свойство (`%project.foo`) не резолвится вовсе.

Когда значения нет — проект ещё не загружен или свойство чужое — подстановка ведёт себя как у незаполненного фильтра: ключ параметра выпадает из запроса, а плейсхолдер в условии остаётся на месте. Перезапрос по смене значения не нужен: проект меняется только вместе с перезагрузкой страницы.

```json
{
  "name": "project_docs",
  "layerName": "documents",
  "condition": "project_name = %project AND title = %project.alias"
}
```

**Подстановка фильтров:** [[utils|утилита]] `formatDataSourceCondition` — заменяет все вхождения `%filterName`, `%filterName.min`, `%filterName.max`, `{attributeName}` в condition на текущие значения фильтров.

---

## Слои

**Слой** (`ConfigLayer`) — описание карт-слоя, который должен быть активирован на текущей странице дашборда. Когда пользователь переходит на страницу «Жилые объекты», система включает слой `residential_buildings` и выключает все остальные слои этой страницы согласно `isVisible`.

```ts
interface ConfigLayer {
  name: string;          // идентификатор слоя в ГИС
  opacity: number;       // прозрачность 0–1
  condition?: string;    // EQL-фильтр для отображения объектов
  query?: string;        // EQL-запрос
  parameters?: object;   // параметры запроса
  isVisible: boolean;    // видимость при загрузке страницы
  selectable: boolean;   // можно ли кликать по объектам
  filterZoomTo: boolean; // приблизить карту к результатам фильтра
  searchFields?: string[]; // поля для поиска по слою
  minScale?: number;     // минимальный масштаб отображения
  maxScale?: number;     // максимальный масштаб отображения
}
```

`query` и `parameters` слоя проходят ту же подстановку, что и условия источников данных: фильтры страницы (`%name`, `%name.min` / `.max`, `%name.lN`), атрибуты карточки (`{attributeName}`, `$card:<layer>:<field>`), системные значения карты `%geometry` / `%extent` / `%zoom` (см. [[#Системные фильтры карты]]) и значения проекта `%project` / `%project.name` / `%project.alias` (см. [[#Текущий проект]]). В UI значение параметра переключается на подстановку кнопкой «%» в панели фильтров слоя — список предлагает фильтры текущей страницы плюс шесть системных плейсхолдеров.

**DashboardLayerPayload** — обновление состояния слоя в runtime: `{ name: string, isVisible?: boolean, condition?: string, ... }`. Вызывается через `setDashboardLayer` из контекста. Имя слоя типизируется branded-типом [[types#Branded types|LayerName]] (`asLayerName`).

**layerInfos** — метаданные слоёв (`QueryLayerServiceInfoDc[]`), передаются через DashboardProvider. Используются для: форматирования значений атрибутов по их типу (`Int32`, `DateTime`, `String` и т.д.), отображения иконки слоя в шапке, резолва связанных атрибутов (join).

---

## ID контейнеров и элементов

**Каждый узел конфига обязан иметь поле `id`** — это инвариант методологии Dashboard. Без `id` движок не сможет ни найти контейнер в большом конфиге, ни поместить элемент в нужное место рендера, ни адресовать конкретный фильтр/таб/кнопку в коллекции. Смысл `id` зависит от роли узла — есть **три случая**.

### Три смысла `id`

| Роль узла | Что значит `id` | Тип | Уникальность |
|---|---|---|---|
| **Контейнер** (`templateName`-узел) | Уникальное имя для быстрого поиска и связей | `ContainerId` (бренд `string`), подтипы — `ChartId`, `ModalId`, `TabId` | Глобально уникален в `config` |
| **Элемент** (`type`-узел внутри `children`) | Слот — зарезервированное место в parent-контейнере | Литеральный `string` из фиксированного набора | Уникален в пределах `children` одного контейнера |
| **Перечисляемая сущность в `children`** (фильтр в `FiltersContainer`, таб в `TabsContainer`, кнопка в `AddFeatureContainer` — узлы со специализированными child-типами без своего `templateName`/`type`) | Уникальное имя, как у контейнера | `string` (для табов — `TabId`) | Уникален в пределах `children` одного контейнера |

### `id` контейнера — уникальное имя

Контейнер — это узел с полем `templateName` (`"Chart"`, `"Tabs"`, `"DataSource"`, ...). Его `id` живёт в едином keyspace `ConfigContainer.id` и должен быть **глобально уникален**. Через него работают:

- **Навигация по страницам:** `id: "page_1"` — корневой контейнер страницы (ContainersGroup), используется `changePage`, `pageIndex`.
- **Связи между узлами:**
  - `PageChild.options.tabId` → `id` вкладки в `TabsContainer` (тип `TabId`)
  - `ElementModal.options.modalId` → `id` модала в `config.modals[]` (тип `ModalId`)
  - `ElementLegend.options.chartId` → `id` `ChartContainer` либо `id` дочернего элемента-чарта внутри него (тип `ChartId`)
  - `TitleContainer.options.downloadById` → `id` `ExportPdfContainer`
- **Управление состоянием:** `expandedContainers: ContainerId[]`, `expandContainer(id)`, `selectedTabId`.
- **Утилиты:** `getRootElementId(type)` строит `${type}-root` для PDF-экспорта.

Типизация — branded types (см. [[types#Branded types|Branded types]]).

### `id` элемента — слот в контейнере

Элемент — это узел с полем `type` (`"button"`, `"chart"`, `"icon"`, ...) внутри `children`. Его `id` — это **слот**, ключ из фиксированного набора, который контейнер ожидает у своих детей. Контейнер рендерит конкретный слот через `renderElement({ id: "<slot>" })`.

Например, `ChartContainer` рендерит детей через `renderElement({ id: "chart" })`, `renderElement({ id: "alias" })`, `renderElement({ id: "legend" })`. Если у дочернего элемента `id` не совпадает с ожидаемым slot-id — элемент не отрисуется.

### Перечисляемые сущности в `children`

Некоторые контейнеры держат в `children` не элементы и не вложенные контейнеры, а **специализированные child-типы**:

- `FiltersContainer` → `FilterChild` (фильтр с `options.filterName`)
- `TabsContainer` → `TabChild` (вкладка)
- `AddFeatureContainer` → `AddFeatureButtonChild` (кнопка добавления фичи)

Это **перечисляемые сущности** — каждая своя со своим действием/конфигом, не slot из фиксированного набора. У них `id` — **уникальное имя в пределах `children`**, как у контейнера. По нему React строит `key` для списка, а внешний код адресует конкретный экземпляр (например, `PageChild.options.tabId` ссылается на `id` нужной вкладки в `TabsContainer`).

У **фильтра** в придачу к `id` обязателен `options.filterName` — это ключ фильтра из `ConfigFilter.name` на странице, по которому работает логика подстановки в `condition` и сброса. `id` и `filterName` — **разные вещи**: `id` идентифицирует узел в `children`, `filterName` связывает фильтр со страничным конфигом.

#### Таблица фиксированных slot-id

| Контейнер / шапка | Slot-id (обязательные и опциональные) |
|---|---|
| `ChartContainer` | `alias`, `chart`, `legend`, `title`, `titleIcon` |
| `TwoColumnContainer` | `alias`, `value`, `units`, `icon`, `tooltip`, `modal` |
| `OneColumnContainer` | `alias`, `value`, `units`, `tooltip`, `modal` |
| `CameraContainer` | `alias`, `value` |
| `IconContainer` | `icon`, `alias`, `link`, `text` |
| `ImageContainer` | `alias`, `text`, `button`, `image` |
| `SlideshowContainer` | `slideshow`, `alias` (опционален) |
| `StructuredDataContainer` | `data` — представление `type: "table"` (обязателен), `alias` |
| `UploadContainer` | `uploader` |
| `AttachmentContainer` | `alias`; `value` — источник вложений, если не задан `relatedDataSource` |
| `EditContainer` (базовый, `templateName: "Edit"`) | `alias`, `value` |
| Подтипы `Edit*` (`EditString`, `EditNumber`, `EditBoolean`, `EditDropdown`, `EditChips`, `EditCheckbox`, `EditDate`) | `alias`, `tooltip` — контрол встроен в контейнер, слота `value` у них **нет** |
| `EditAttachmentContainer` | `alias` |
| `EditGroupContainer` | `alias`, `tooltip` (+ `units`, `icon` — их читает [[hooks\|`useRenderContainerItem`]]); дети клонируются в разрешённый по типу атрибута `Edit*`-шаблон |
| `DataSourceContainer`, `DataSourceProgressContainer` | slot-id **внутреннего шаблона** `options.innerTemplateName`, а не собственные слоты хоста — см. раздел «[[concepts#Рендеринг записей источника — innerTemplateName\|Рендеринг записей источника]]» |
| `AddFeatureContainer` | **дети — кнопки `AddFeatureButtonChild` с уникальным `id`** (перечисляемые сущности — см. подраздел выше) |
| `TabsContainer` | **дети — табы `TabChild` с уникальным `id`** (тип `TabId` — перечисляемые сущности) |
| `FiltersContainer` | **дети — фильтры `FilterChild` с уникальным `id`** + обязательный `options.filterName` (перечисляемые сущности) |
| `VoteContainer` | собственных слотов **нет** — экран рисует сам контейнер; допустимы только универсальные `title`/`titleIcon`/`bgImage`. Обязательно свойство узла `attributeName` — атрибут объекта с `question_id` |
| `FeatureCardBackgroundHeader` | `title`, `description`, `bgImage`, `icon` |
| `FeatureCardSlideshowHeader` | `title`, `description`, `bgImage`, `slideshow` |
| `DashboardDefaultHeader` | `title`, `icon`, `image` (логотип; при отсутствии — иконка `logo` и `options.title` страницы) |
| `FeatureCardDefaultHeader` | — структура фиксирована, кастомные дети не рендерятся |

Слоты `title`, `icon`, `titleIcon` на уровне контейнера уходят в заголовок (`TITLE_SLOT_IDS`): `ContainerChildren` исключает их из рендера тела, а сетка — из подсчёта треков. `title` и `titleIcon` допустимы у **любого** контейнера с заголовком (`ExpandableTitle`) — `Chart`, `Camera`, `Attachment`, `Slideshow`, `StructuredData`, `Upload`, `Edit`, `Filters`, `Layers`, `Task`, `Vote`, `DataSource`, `DataSourceProgress`, `ContainersGroup` — и поэтому в наборы слотов отдельных контейнеров не входят.

#### Универсальные слоты и фон контейнера

Помимо слотов заголовка у любого контейнера допустим ещё один универсальный slot-id — `bgImage`. Вместе они образуют `NON_TRACK_SLOT_IDS` (`= TITLE_SLOT_IDS + BG_IMAGE_SLOT_ID`): набор slot-id, которые контейнер читает по `id` сам и не отдаёт ни в общий рендер тела (`ContainerChildren`), ни в треки сетки (`gridTracks` / `gridTree`).

| Slot-id | Кто рендерит | Где рисуется | Ограничение |
|---|---|---|---|
| `title`, `titleIcon` | [[components\|`ExpandableTitle`]] | заголовок контейнера | нужен контейнер с заголовком |
| `bgImage` | [[components\|`ContainerBackground`]] | отдельный слой **под** содержимым | у любого контейнера, кроме `Divider` |

**Как устроен фон.** Слот `bgImage` — обычный элемент `type: "image"` (URL берётся из корневого `value`, `attributeName` или `options.resourceId`). Компонент `ContainerBackground` ставится первым ребёнком корня контейнера и рендерит `renderElement({ id: "bgImage", wrap: false })` внутри абсолютного слоя `ContainerBackgroundLayer` (`inset: 0`, `z-index: -1`, `border-radius: inherit`, `pointer-events: none`). Картинка по умолчанию заполняет бокс через `object-fit: cover`; авторский `options.fit` у элемента перебивает дефолт.

**Гейт по наличию слота.** Слой рисуется **только** если узел действительно несёт ребёнка с `id: "bgImage"` — это проверяет [[utils|утилита]] `hasContainerBgImage`. Без слота в DOM не появляется ничего: ни обёртки, ни пустого абсолютного `div`. Тот же признак включает и хост: корень контейнера получает `$hasBgImage` и вместе с ним `position: relative` + `isolation: isolate` (`bgImageHostMixin`). Изоляция принципиальна — отрицательный `z-index` слоя ложится под содержимое хоста только внутри его собственного stacking-контекста, иначе ушёл бы под фон ближайшего предка и пропал. Гейт по `$hasBgImage` тоже не декоративный: `position: relative` меняет containing block для абсолютно позиционированных потомков (контролы слайдшоу, подписи прогресса), поэтому включается лишь там, где автор конфига попросил фон.

Признак попадает на корень двумя путями: контейнеры на [[hooks|`useContainerRoot`]] / `useWrapperSize` получают `$hasBgImage` готовым в пропсах корня, а те, что ставят `id`/`style` руками (`Title`, `Icon`, `Tabs`, `AddFeature`, `ExportPdf`, `Progress`, `RoundedBackground`, `OneColumn`, `TwoColumn`, `DefaultAttributes`, `PagesContainer`, подтипы `Edit*`), берут его хуком [[hooks|`useBgImageHost`]].

**Две опции-компаньона фона.** Обе живут в [[options#ConfigLayoutOptions|`ConfigLayoutOptions`]] и, в отличие от остальных опций контейнера, **не входят ни в один `<Name>Options`** — их читают хук хоста и слой фона напрямую из `options`, поэтому они допустимы у любого контейнера:

| Опция | Кто читает | Что делает |
|---|---|---|
| `innerPadding` | [[hooks\|`useBgImageHost`]] → проп `$innerPadding` на корне | Фиксированный внутренний отступ `1rem` (`CONTAINER_INNER_PADDING`) по всем краям корня — содержимое не липнет к краям картинки |
| `outflow` | [[components\|`ContainerBackground`]] → проп `$outflow` на слое | Слой вытекает за края контейнера на `1.5rem` (`BG_IMAGE_OUTFLOW`) — по бокам и вверх |

`innerPadding` — булев по замыслу: автор конфига включает «отступ от краёв фона», а не подбирает число (для произвольного отступа есть `options.padding`). Значение ставится под селектором `&&`, чтобы перебить `padding` из `$sizeCss` и внутренних `defaults` контейнера; авторский inline-`style` по-прежнему сильнее. Гейт у него **отдельный** от `$hasBgImage`: отступ содержимого нужен и без картинки, поэтому фоном он не обусловлен.

`outflow` читает слой, а не хост: вылет за края — свойство картинки, раскладка контейнера от него не меняется (слой абсолютный, содержимое остаётся в своих границах). `1.5rem` — ровно `padding` карточки контейнера (`ContainerWrapper`), поэтому картинка дотягивается до краёв колонки дашборда. Вниз слой не вытекает никогда: там начинается следующий контейнер колонки. Без слота `bgImage` опция не делает ничего — вытекать нечему; и её обрезает любой предок с `overflow` кроме `visible`, включая собственный `options.overflow` контейнера.

```tsx
{
  id: "hero_card",
  templateName: "ContainersGroup",
  options: { column: true, height: 240, innerPadding: true, outflow: true },
  children: [
    { id: "bgImage", type: "image", options: { resourceId: "1f2e..." } },
    { id: "title", type: "text", value: "Сводка по округу" }
  ]
}
```

**Исключение — `Divider`.** У разделителя нет ни бокса, ни собственного содержимого: одна линия. Слой фона там молча потерялся бы, поэтому слот у него не поддерживается — клиентский валидатор отдаёт `unexpected-slot`.

**DataSource-хосты.** У `DataSource` / `DataSourceProgress` дети — слоты **внутреннего шаблона**, поэтому `bgImage` достался бы и хосту, и каждой записи: одна картинка нарисовалась бы N+1 раз. `DataSourceInnerContainer` вырезает слот из конфига записи — фон остаётся за хостом, он и есть контейнер.

**Шапки FeatureCard.** `FeatureCardBackgroundHeader` и `FeatureCardSlideshowHeader` рисуют свой `bgImage` тем же компонентом `ContainerBackground` — прежний локальный `ImageContainerBg` убран, а маска `bottomBlur` теперь целится в `ContainerBackgroundLayer`.

```tsx
{
  id: "stats_card",
  templateName: "ContainersGroup",
  options: { column: true, height: 240 },
  children: [
    { id: "bgImage", type: "image", options: { resourceId: "1f2e...", fit: "cover" } },
    { id: "total", templateName: "OneColumn", children: [{ id: "value", attributeName: "total" }] }
  ]
}
```

Типизация slot-id — литеральные string'и в parent-specific child-типах (`ChartAliasChild`, `ChartChartChild`, `ChartLegendChild`, ...). См. [[types#Slot-id — НЕ branded|Slot-id]].

> [!info] Что из таблицы проверяет клиентский валидатор
> `CONTAINER_SLOT_MAP` (`client-new/src/components/Dashboard/utils/constants.ts`) — зеркало этой таблицы; мастер-источник — документация, при правке обновляй обе стороны.
>
> - Универсальные слоты (`title`, `titleIcon`, `bgImage` — `UNIVERSAL_SLOT_IDS`) пропускаются у всех контейнеров: ни в набор слотов, ни в требование `filterName` они не входят.
> - Слот `bgImage` у `Divider` — ошибка `unexpected-slot`: единственный контейнер, который фон не поддерживает (`NO_BG_IMAGE_TEMPLATE`).
> - `options.outflow` без слота `bgImage` — ошибка `orphan-option`: вытекать нечему, опция молча не работает. `options.innerPadding` так не проверяется — это обычный отступ, осмысленный и без картинки.
> - У `DataSource`/`DataSourceProgress` валидатор **резолвит `options.innerTemplateName`** и проверяет детей по слотам внутреннего шаблона; пропуск опции — ошибка `missing-inner-template`.
> - Контейнер без записи в карте (например `ContainersGroup` — в том числе как внутренний шаблон с произвольной вёрсткой) на слоты не проверяется. Туда же попадает опечатка в `innerTemplateName`: неизвестное имя правила не находит, и дети не проверяются — как и рантайм, который молча откатывается на `ContainersGroup`.
> - У `StructuredData` сверх слотов проверяется собственный набор инвариантов (`validateStructuredData.ts`): есть ребёнок `data` с `type: "table"` (`missing-view`); задан `options.filterName` (`missing-filter-name`), и такой фильтр объявлен на странице с `valueType: "features"` (`invalid-filter-value-type`); без `relatedDataSource` описана схема (`missing-schema`); имена атрибутов уникальны (`duplicate-attribute`).

### Сводный пример с двумя уровнями `id`

```tsx
{
  id: "chart_floors",                                                  // id контейнера: уникальное имя в keyspace
  templateName: "Chart",
  options: { twoColumns: true },
  children: [
    { id: "alias", value: "Этажность" },                               // slot
    { id: "chart", type: "chart", options: { chartType: "bar" } },     // slot
    { id: "legend", type: "legend", options: { chartId: "chart" } }    // slot; chartId ссылается на slot брата-чарта
  ]
}
```

### Что произойдёт без `id`

| Узел | Симптом |
|---|---|
| Контейнер без `id` | Поломается навигация; связи `tabId`/`modalId`/`chartId`/`downloadById` не разрезолвятся; `expandedContainers` и `selectedTabId` не смогут управлять состоянием |
| Элемент без `id` | Не попадёт в ожидаемый slot — `renderElement({ id })` вернёт `null`, контейнер пропустит элемент |
| Перечисляемая сущность в `children` без `id` | Невозможно адресовать конкретный фильтр/таб/кнопку в коллекции — поломаются связи (`tabId` со стороны `PageChild`, обращение к фильтру), нарушится React `key` для списка рендеринга |

```ts
// ❌ так НЕ работает — ни контейнер, ни его элементы не отрендерятся
{ templateName: "Chart", children: [{ type: "chart" }, { type: "legend" }] }

// ✅ так работает — id у контейнера и корректные slot-id у элементов
{
  id: "chart_floors",
  templateName: "Chart",
  children: [
    { id: "alias", value: "Этажность" },
    { id: "chart", type: "chart" },
    { id: "legend", type: "legend", options: { chartId: "chart" } },
  ],
}
```

Готовые правила и чек-лист генерации — на странице [[authoring|Правила генерации]].

---

## Контексты

Дашборд использует три вложенных контекста. Каждый контекст предоставляет свой слой данных компонентам-потребителям.

### DashboardContext

Создаётся `BaseDashboardProvider`. Содержит всё необходимое для работы дашборда:

- **Конфиг** (`config: ConfigContainer`) — полная конфигурация дашборда
- **Навигация** (`pageIndex`, `nextPage`, `prevPage`, `changePage`) — управление текущей страницей
- **Данные** (`dataSources: WidgetDataSource[]`) — загруженные источники данных
- **Фильтры** (`filters: SelectedFilters`, `changeFilters`) — текущие значения и метод изменения
- **Слои** (`dashboardLayers`, `setDashboardLayer`) — состояние слоёв карты
- **UI** (`expandedContainers`, `expandContainer`, `selectedTabId`, `setSelectedTabId`) — состояние раскрытых контейнеров и активных вкладок
- **Компоненты** (`components: { LayerItem, ProjectPagesMenu, ProjectPanelMenu }`) — кастомные компоненты от клиента

Нужен, когда компонент отображает данные дашборда, реагирует на фильтры или меняет страницу.

### FeatureCardContext

Создаётся `FeatureCardProvider`. Содержит данные конкретного выбранного объекта на карте:

- **Атрибуты** (`attributes: ClientFeatureAttribute[]`) — все атрибуты объекта с alias, type, value
- **Редактирование** (`controls`, `changeControls`, `saveControls`) — изменение значений атрибутов
- **Метаданные** (`layerInfo`, `feature`) — информация о слое и самом объекте
- **Состояние** (`isEditable`, `isLoading`, `isEdit`) — режим просмотра/редактирования

Нужен, когда компонент находится внутри карточки объекта (FeatureCard) и отображает или редактирует атрибуты.

### GlobalContext

Создаётся `GlobalProvider`. Содержит глобальные зависимости приложения:

- **API** (`api`) — все методы API для запросов к бэкенду
- **Локализация** (`t`) — функция перевода `i18next`
- **Карта** (`ewktGeometry`, `ewktExtent`, `zoomLevel`) — геометрический фильтр, экстент видимой области и уровень зума
- **Тема** (`themeName`) — `"light"` | `"dark"`
- **Язык** (`language`) — текущий язык интерфейса

Нужен везде: каждый компонент, делающий запрос или выводящий текст, зависит от GlobalContext.

Все три контекста агрегируются через `useWidgetContext(type)`, который переключается между `DashboardContext` и `FeatureCardContext` по значению `WidgetType` (`Dashboard` | `FeatureCard`).

---

## Real-time обновления

**Real-time** позволяет источнику данных автоматически обновляться при изменении объектов слоя в бэкенде — без перезагрузки страницы.

Механизм:
1. В конфиге источника данных указывается `"autoSyncLayer": true`
2. Клиентский хук `useDataSourceSubscriptions` (вызывается из `useProjectDataSources`, см. [[setup|Подключение]]) подписывается на WebSocket-событие для каждого такого источника: `addSubscription({ tag: "feature_layer_updated", resources: [layerName] })`
3. Когда другой пользователь изменяет объект слоя — бэкенд отправляет `ReceiveFeaturesUpdateNotification`
4. Хук убирает записи нужных источников из стора и вызывает `fetchData(updatingDataSources)` для перезагрузки — зависимые контейнеры показывают свой `ContainerLoading`, соседние не мигают. Ставить `features: null` нельзя: в семантике контейнеров это ошибка («Блок не загружен»), а не загрузка
5. При смене проекта подписки снимаются (`unsubscribeById`) и оформляются заново — иначе остались бы висеть на слоях прежнего проекта. Снимаются они и на размонтировании

```json
{
  "name": "incidents",
  "layerName": "incidents_layer",
  "autoSyncLayer": true
}
```

Практический пример: диспетчерский дашборд — операторы видят новые инциденты в реальном времени без перезагрузки страницы.

Использует `useServerNotificationsContext` из `@evergis/react`.

### Real-time в карточке объекта

`useDataSourceSubscriptions` обслуживает только страницу виджета Dashboard. У карточки объекта свой набор хуков (client-new, `components/FeatureCard/hooks`):

- `useFeatureCardSync` — подписан на `feature_layer_updated` всех слоёв текущего выбора. Когда `updatedIds` нотификации содержит `currentId`, объект перезапрашивается через `layers.getFeatures1` и обновляется в сторе (`updateCurrentFeature`, только `properties` и `geometry` — `id` и `layer` менять нельзя, на них завязаны выбор и пагинация). Пока пользователь редактирует объект, обновление пропускается: объект из стора наполняет форму, и внешние данные затёрли бы несохранённый ввод
- Удалённые объекты (`deletedIds`) уходят из выбора через `removeFeatures` (`hooks/map/useSelectFeatures`) — чистая функция `getSelectFeaturesAfterRemove` (`utils/selectFeatures`) убирает их из списка слоя, уменьшает `totalCounts` и переводит `currentId` на соседний объект (следующий, иначе предыдущий). Если в слое не осталось объектов, `currentId` обнуляется и карточка показывает `NoFeatureCard`. Удаление применяется и при открытой форме правки: объекта больше нет, сохранение в него всё равно не пройдёт
- `useFeatureDataSourceSubscriptions` — аналог дашбордового хука для источников карточки: одна подписка на все слои источников с `autoSyncLayer`, при нотификации `fetchData(updatingDataSources)` из `useFeatureDataSources`
- Отдельный эффект в `useFeatureDataSources` перезапрашивает источники, когда у того же объекта изменились значения атрибутов: они подставляются в параметры и условия запросов (`%attributeName`), а `feature.id` при внешнем обновлении не меняется и эффект первичной загрузки не срабатывает

Оба хука держат payload подписки пустым, пока `connection` не поднят: карточка смонтирована с самого старта приложения, а `useServerNotification` оформляет подписку один раз на текущий payload.

---

## Серверные хуки сохранения (beforeSave / afterSave)

**Save-хуки** — это серверные python-скрипты, которые выполняются при сохранении объекта в FeatureCard: один до сохранения (валидация), другой после (побочное действие). Они описываются в конфигурации слоя `layerInfo.configuration.editConfiguration.options` и не требуют изменения клиентского кода.

- **`beforeSave`** — синхронная серверная проверка **перед** сохранением. Save ждёт завершения задачи: при статусе `Completed` сохранение продолжается, при `Error` — отменяется. Текст ошибки показывается из `log` задачи. Практический пример: проверка пересечения геометрии нового участка с уже существующими — если пересечение есть, сохранение блокируется.
- **`afterSave`** — fire-and-forget действие **после** успешного сохранения. Не блокирует UI и не влияет на результат save. Практический пример: пересчёт связанной агрегации или запуск нотификации смежной службе.

```json
{
  "editConfiguration": {
    "options": {
      "beforeSave": {
        "resourceId": "validate-geometry-script",
        "methodName": "validate",
        "parameters": { "tolerance": { "default": 0.5 } }
      },
      "afterSave": {
        "fileName": "recalc.py",
        "methodName": "run",
        "parameters": {}
      }
    }
  }
}
```

Описание скрипта — тип `ConfigRelatedResource` (`resourceId`/`fileName`, `methodName`, `parameters`, `script`). Контейнер `editConfiguration.options` — тип `EditConfigurationOptions` (`{ beforeSave?, afterSave? }`). Вход хуков — `SaveHookInput`:

```ts
interface SaveHookInput {
  featureId: number | string | null;          // null при создании нового объекта
  changedProperties: Record<string, unknown>; // изменённые атрибуты
  changedGeometry?: Geometry;                  // новая/изменённая GeoJSON-геометрия (WGS84)
}
```

### Дата-контракт python-скрипта

`SaveHookInput` — это вход хука **на клиенте**; на сервер он попадает не напрямую. Билдер [[hooks|хук]] `useSavePrototypeBuilder` собирает из него (и из контекста FeatureCard/Dashboard) объект `parameters`, который передаётся python-скрипту как аргумент вызова `pythonrunner/run`. Именно эту структуру скрипт читает на входе — и для `beforeSave`, и для `afterSave` контракт одинаковый.

```json
{
  "projectName": "city_cadastre",
  "layerName": "parcels",
  "featureId": null,
  "selectionGeometry": "SRID=4326;POLYGON((37.6 55.7, ...))",
  "edit": {
    "attributes": { "name": "Участок №5", "area": 1240 },
    "featureGeometry": "SRID=4326;POLYGON((37.6 55.7, ...))"
  },
  "tolerance": 0.5
}
```

| Поле | Тип | Источник | Наличие |
|---|---|---|---|
| `projectName` | `string` | `projectInfo.name` (контекст Dashboard) | всегда |
| `layerName` | `string` | `layerInfo.name` (контекст FeatureCard) | всегда |
| `featureId` | `number \| string \| null` | `SaveHookInput.featureId` | всегда; `null` при создании нового объекта |
| `selectionGeometry` | `string` (EWKT, `SRID=4326;...`) | геометрический фильтр карты `ewktGeometry` из [[setup\|GlobalContext]] | только если фильтр на карте активен |
| `edit.attributes` | `Record<string, unknown>` | `SaveHookInput.changedProperties` | всегда (может быть `{}`) |
| `edit.featureGeometry` | `string` (EWKT, `SRID=4326;...`) | `geometryToEwkt(SaveHookInput.changedGeometry)` | только если геометрия объекта менялась |
| *(пользовательские поля)* | любые | `hook.parameters` после подстановки фильтров через `applyQueryFilters` | как заданы в конфиге |

**Ключевые моменты контракта:**

- **`edit` — это «что изменил пользователь».** `edit.attributes` содержит только изменённые атрибуты, `edit.featureGeometry` — только изменённую геометрию. Если геометрия не редактировалась — ключ `featureGeometry` **отсутствует** (а не приходит `null`); скрипт должен проверять его наличие.
- **Геометрия — в EWKT, WGS84.** На клиенте `changedGeometry` — это GeoJSON `Geometry`; билдер конвертирует его в строку EWKT (`SRID=4326;...`) через `geometryToEwkt`. То же касается `selectionGeometry`.
- **`selectionGeometry` ≠ `edit.featureGeometry`.** Первое — область выделения/фильтра на карте (контекст «где смотрит пользователь»), второе — собственная геометрия сохраняемого объекта.
- **Пользовательские `parameters` мёржатся на верхний уровень.** Поля из `hook.parameters` (в примере конфига выше — `tolerance`) проходят через `applyQueryFilters` (подстановка `%filterName`, геометрии и т.п.) и добавляются в payload **последними** — при совпадении имён они переопределяют служебные поля, поэтому не называйте свои параметры `projectName`/`layerName`/`featureId`/`edit`.

Практический пример: `beforeSave`-скрипт читает `edit.featureGeometry` нового участка, сверяет его с уже существующими объектами слоя `layerName` в проекте `projectName` и возвращает ошибку (статус задачи `Error` + текст в `log`), если есть пересечение — клиент отменяет сохранение.

Механизм реализован хуками [[hooks|Хуки]] `useFeatureSaveHooks` (orchestrator), `useBeforeSave`, `useAfterSave`, `useSavePrototypeBuilder`. Параметры скрипта собираются через [[utils|утилиту]] `applyQueryFilters` (подстановка фильтров/геометрии) и запускаются как remote task (`pythonrunner/run`). Имя ресурса типизируется branded-типом [[types#Branded types|ResourceId]].

---

## Registry

**Registry** — механизм, связывающий строковое имя типа из JSON-конфига с React-компонентом. Это позволяет декларативно описывать UI через JSON-конфиг, не упоминая компоненты напрямую.

**Регистрация контейнеров** в `containers/registry.ts` — объект собирается лениво и кэшируется (иначе циклический импорт `containers → utils → registry` даёт TDZ):
```ts
const createContainerComponents = () =>
  ({
    [ContainerTemplate.Chart]: ChartContainer,
    [ContainerTemplate.DataSource]: DataSourceContainer,
    [ContainerTemplate.Filters]: FiltersContainer,
    // ... всего 35 записей
    default: ContainersGroupContainer, // если templateName не найден
  }) as const satisfies ContainerComponentRegistry;

export const getContainerComponents = () => { /* кэш + createContainerComponents() */ };
```

**Регистрация элементов** в `elements/registry.ts`:
```ts
export const elementComponents = {
  chart: ElementChart,
  image: ElementImage,
  icon: ElementIcon,
  // ... 15 элементов
} as const satisfies ElementComponentRegistry;
```

Контракт `ContainerComponentRegistry` / `ElementComponentRegistry` (см. [[types#Типизированные реестры|Типизированные реестры]]) на этапе компиляции проверяет, что каждый зарегистрированный компонент принимает корректные `<Name>Props`.

Разрешение:
- [[utils|утилита]] `getContainerComponent(templateName)` → `FC<ContainerProps> | null`
- [[utils|утилита]] `getRenderElement(props)` → `ReactNode`

**Расширение:** клиент может добавить свои контейнеры под новыми значениями `ContainerTemplate`, а элементы — под новыми строковыми литералами `type` (`ConfigElementType`). Рендеринг автоматически использует новые компоненты.

---

## Связанные разделы

[[architecture|Архитектура]] | [[options|Опции]] | [[types|Типы]] | [[hooks|Хуки]] | [[containers|Контейнеры]] | [[elements|Элементы]]
