# Основные понятия

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

Результат загрузки — `WidgetDataSource`: `{ name, features, layerName, attributeDefinition }`.

**Умная инвалидация:** при изменении значения фильтра система вычисляет список изменившихся фильтров → `getUpdatingDataSources()` возвращает только те источники, в `condition` или `parameters` которых есть ссылка `%filterName` → перезагружаются только они. Остальные источники не тронуты.

Хук: [[hooks|хук]] `useDataSources`. Имя источника типизируется branded-типом [[types#Branded types|DataSourceName]] (`asDataSourceName`).

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
  value: string | number | string[] | number[] | Date | Date[];
  min?: string | number | Date;
  max?: string | number | Date;
}
```

**ConfigFilter** — описание фильтра в конфиге страницы: `name`, `defaultValue`, `valueType`, `relatedDataSource` (откуда брать список вариантов), `resetFilters` (сбрасываемые при изменении фильтры). Имя фильтра типизируется branded-типом [[types#Branded types|FilterName]] (`asFilterName`).

**Геометрический фильтр:** специальный фильтр с именем `geometry`. Позволяет фильтровать объекты на карте по нарисованной области (polygon, bbox). Передаётся как `ewktGeometry` через [[setup|GlobalContext]]. При вставке в условие: `%geometry` → `'SRID=4326;POLYGON(...)'`. Пример: пользователь рисует прямоугольник на карте → все источники данных с `%geometry` в условии перезапрашиваются для выбранного района.

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
| `TwoColumnContainer`, `OneColumnContainer` | `alias`, `value`, `units` |
| `CameraContainer` | `alias`, `value` |
| `IconContainer` | `icon`, `alias`, `link`, `text` |
| `ImageContainer` | `alias`, `text`, `button`, `image` |
| `AttachmentContainer` (без `relatedDataSource`) | `value` |
| `Edit*Container` | `alias` (label поля редактирования) |
| `AddFeatureContainer` | **дети — кнопки `AddFeatureButtonChild` с уникальным `id`** (перечисляемые сущности — см. подраздел выше) |
| `TabsContainer` | **дети — табы `TabChild` с уникальным `id`** (тип `TabId` — перечисляемые сущности) |
| `FiltersContainer` | **дети — фильтры `FilterChild` с уникальным `id`** + обязательный `options.filterName` (перечисляемые сущности) |
| `FeatureCardBackgroundHeader` | `title`, `description`, `bgImage`, `icon` |
| `FeatureCardSlideshowHeader` | `title`, `description`, `bgImage`, `slideshow` |
| `DashboardDefaultHeader`, `FeatureCardDefaultHeader` | `title`, `description`, `icon` |

Типизация slot-id — литеральные string'и в parent-specific child-типах (`ChartAliasChild`, `ChartChartChild`, `ChartLegendChild`, ...). См. [[types#Slot-id — НЕ branded|Slot-id]].

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
- **Карта** (`ewktGeometry`) — геометрический фильтр из карты
- **Тема** (`themeName`) — `"light"` | `"dark"`
- **Язык** (`language`) — текущий язык интерфейса

Нужен везде: каждый компонент, делающий запрос или выводящий текст, зависит от GlobalContext.

Все три контекста агрегируются через `useWidgetContext(type)`, который переключается между `DashboardContext` и `FeatureCardContext` по значению `WidgetType` (`Dashboard` | `FeatureCard`).

---

## Real-time обновления

**Real-time** позволяет источнику данных автоматически обновляться при изменении объектов слоя в бэкенде — без перезагрузки страницы.

Механизм:
1. В конфиге источника данных указывается `"autoSyncLayer": true`
2. При монтировании `useProjectDataSources` подписывается на WebSocket-событие: `addSubscription({ tag: "feature_layer_updated", resources: [layerName] })`
3. Когда другой пользователь изменяет объект слоя — бэкенд отправляет `ReceiveFeaturesUpdateNotification`
4. Хук сбрасывает `features: null` для нужных источников и вызывает `fetchData(updatingDataSources)` для перезагрузки

```json
{
  "name": "incidents",
  "layerName": "incidents_layer",
  "autoSyncLayer": true
}
```

Практический пример: диспетчерский дашборд — операторы видят новые инциденты в реальном времени без перезагрузки страницы.

Использует `useServerNotificationsContext` из `@evergis/react`.

---

## Registry

**Registry** — механизм, связывающий строковое имя типа из JSON-конфига с React-компонентом. Это позволяет декларативно описывать UI через JSON-конфиг, не упоминая компоненты напрямую.

**Регистрация контейнеров** в `containers/registry.ts`:
```ts
export const containerComponents = {
  [ContainerTemplate.Chart]: ChartContainer,
  [ContainerTemplate.DataSource]: DataSourceContainer,
  [ContainerTemplate.Filters]: FiltersContainer,
  // ... 34 контейнера
  default: ContainersGroupContainer, // если templateName не найден
} as const satisfies ContainerComponentRegistry;
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

**Расширение:** клиент может добавить свои контейнеры/элементы, зарегистрировав их под новыми именами `ContainerTemplate` / `ElementTemplate`. Рендеринг автоматически использует новые компоненты.

---

## Связанные разделы

[[architecture|Архитектура]] | [[options|Опции]] | [[types|Типы]] | [[hooks|Хуки]] | [[containers|Контейнеры]] | [[elements|Элементы]]
