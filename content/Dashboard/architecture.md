# Архитектура

## Паттерн конфигурационного управления

Dashboard использует registry-pattern: каждый контейнер и элемент регистрируется по ключу (`ContainerTemplate` enum для контейнеров, строковый `type` для элементов) в `registry.ts`. При рендеринге компонент определяется динамически по значению `templateName` / `type` из конфига.

```ts
// containers/registry.ts — реестр строится ЛЕНИВО и кэшируется (циклический импорт → TDZ)
const createContainerComponents = () =>
  ({
    [ContainerTemplate.Chart]: ChartContainer,
    [ContainerTemplate.DataSource]: DataSourceContainer,
    [ContainerTemplate.Filters]: FiltersContainer,
    // ...37 записей + default
    default: ContainersGroupContainer,
  }) as const satisfies ContainerComponentRegistry;

export const getContainerComponents = () => { /* кэширует createContainerComponents() */ };

// elements/registry.ts — обычная константа, цикла нет
export const elementComponents = {
  chart: ElementChart,
  image: ElementImage,
  // ...16 записей
} as const satisfies ElementComponentRegistry;
```

Реестры — типобезопасные: `ContainerComponentRegistry` и `ElementComponentRegistry` (см. [[types#Типизированные реестры|Типизированные реестры]]) гарантируют, что каждый зарегистрированный компонент принимает корректные `<Name>Props`. Почему реестр контейнеров ленивый — [[types#Типизированные реестры|там же]].

Разрешение компонента:
- Контейнер: `getContainerComponent(templateName)` → `getContainerComponents()[name] || default`
- Элемент: `getRenderElement(...)` возвращает функцию, которая по `id` находит дочерний элемент в конфиге и рендерит нужный компонент

## Типизация

Типы сгруппированы в три слоя:

1. **Per-component тройка** `<Name>Options` / `<Name>Config` / `<Name>Props` — для каждого контейнера, элемента и шапки в `componentTypes.ts`. Сводная таблица — [[types#Per-component типы|Per-component типы]].
2. **Branded keyspaces** — `ChartId`, `ModalId`, `TabId`, `FilterName`, `LayerName`, `AttributeName`, `DataSourceName`, `ResourceId` в `branded.ts`. Защищают от перепутывания entity-id на этапе компиляции. См. [[types#Branded types|Branded types]].
3. **Доменная группировка опций** — `ConfigOptions extends` 13 миксинов (`ConfigLayoutOptions`, `ConfigTypographyOptions`, `ConfigChartOptions`, `ConfigButtonOptions`, ...) плюс документирующий `ConfigEntityRefOptions`. См. [[options|Опции]].

Discriminated union `DashboardChild` (16 вариантов элементов + 36 вариантов контейнеров) и `DashboardHeaderConfig` (4 ветви шапок) позволяют TS сужать ветвь по полю `type` / `templateName` и валидировать `options`. Для авторинга новых конфигов есть строгие варианты `StrictConfigContainerChild` / `StrictDashboardChild`, у которых `id` обязателен (пропуск = ошибка компиляции). Детали — [[types#Дискриминированный union DashboardChild|Дискриминированный union]].

## Иерархия компонентов

```
GlobalProvider (contexts/GlobalContext)
└── TaskExecutionProvider
    ├── TaskExecutionHost (общие логи Python-задач)
    └── BaseDashboardProvider / FeatureCardProvider (contexts/*Context)
        └── WidgetModalsProvider → WidgetActionsProvider
            ├── WidgetModalHost (окна + контекст открытия + локальные источники)
            ├── DashboardHeader / FeatureCardHeader (хост размещает отдельно)
            │   └── DashboardDefaultHeader / FeatureCardDefaultHeader / ...
            └── Dashboard (components/Dashboard/index.tsx)
                └── PagesContainer (containers/PagesContainer)
                    └── ContainerChildren (components/ContainerChildren)
                        └── [ContainerComponent по registry]
                            │   (ContainersGroupContainer, ChartContainer, ...)
                            ├── ContainerBackground → слот bgImage (слой под содержимым)
                            ├── ExpandableTitle    → слоты title / titleIcon
                            └── renderElement({ id })
                                └── [ElementComponent по registry]
                                    (ElementChart, ElementImage, ...)
```

Главный `Dashboard` рендерит только тело `PagesContainer` либо loading-заглушку. Шапку хост размещает отдельно в пределах того же контекста: её точка входа — `DashboardHeader` / `FeatureCardHeader` (см. [[components|Компоненты]]).

Три slot-id универсальны: контейнер читает их по `id` сам и не отдаёт ни в общий рендер тела, ни в треки сетки (`NON_TRACK_SLOT_IDS`). `title` / `titleIcon` уходят в `ExpandableTitle`, `bgImage` — в `ContainerBackground`. Детали — [[containers#Универсальные слоты|Контейнеры]].

Для `FeatureCard` схема аналогична, но использует `FeatureCardContext` и `WidgetType.FeatureCard`.

`ContainersGroupContainer` — диспетчер двух раскладок: обычной (`FlowGroup`) и сеточной (`GridGroup`, при `options.grid`). Собственных хуков у диспетчера нет намеренно — `options.grid` переключается на лету, и любой хук до ветвления менял бы их порядок между рендерами.

```
ContainersGroupContainer
├── FlowGroup                                 ← обычная раскладка
└── GridGroup (options.grid)                  ← модуль grid/
    ├── GridEditSession (options.editMode у ВНЕШНЕГО узла)
    │   └── GridEditContext → GridGroupView + GridContextMenu
    └── GridGroupView                         ← вложенные сетки наследуют сессию
        └── GridTracks → GridTrack → [содержимое ячейки]
```

Вход в сетку из конфига один — `ContainersGroup` с `options.grid`; наружу отдаётся только листовая часть модуля (типы, константы, утилиты) плюс два хостовых пропа, см. [[containers#Интеграционный API для хостов|Контейнеры]] и [[types#Публичная поверхность сетки|Типы]].

## Контексты

### DashboardContext
Хранит конфиг дашборда, состояние страниц, источники данных, фильтры, слои, expandedContainers, кастомные компоненты. Читается через `useWidgetContext(WidgetType.Dashboard)`.

### FeatureCardContext
Хранит атрибуты и конфиг карточки объекта, layerInfo, controls, edit-состояние. Читается через `useWidgetContext(WidgetType.FeatureCard)`.

### GlobalContext
Глобальные настройки: `api`, `t`, `language`, `themeName`, состояние карты для системных плейсхолдеров (`ewktGeometry`, `ewktExtent` → `%extent`, `zoomLevel` → `%zoom`), проект (`projectName` → `%project` / `%project.name`, `projectAlias` → `%project.alias`) и API уведомлений `notification` (`add` / `update` / `close`). Читается через `useGlobalContext()`, полная таблица — в [[setup#GlobalProvider (@evergis/react)|Подключении]].

Подробнее о контекстах — в [[concepts|Основных понятиях]].

## Поток данных

```
projectInfo.content.dashboardConfiguration (ConfigContainer)
  ↓
getPagesFromConfig() → ConfigContainerChild[] (страницы)
  ↓
useWidgetPage() → currentPage (активная страница)
  ↓
currentPage.dataSources → useDataSources()
  ↓
Promise.allSettled(...) → getUpdatedDataSources() → dispatch(setProjectDataSources)
  ↓
useWidgetContext() → dataSources
  ↓
DataSourceContainer / ContainerChildren → renderElement()
  ↓
getContainerComponent(templateName) → ContainerComponent
  ↓
getRenderElement() → ElementComponent
```

Изменение фильтров — только затронутые источники данных перезагружаются (умная инвалидация через `getUpdatingDataSources()`).

Источники модалок (`config.modals[].dataSources`) в `currentPage.dataSources` не пишутся и грузятся лениво через `WidgetModalHost` → [[hooks#useModalSources / useModalAutoSync (internal)|useModalSources]] → внутренний `useDataSourceRequests`. `ModalDataContext` перекрывает источники и loading своего окна в [[hooks#useWidgetContext|useWidgetContext]], а [[hooks#useConfigDataSources|useConfigDataSources]] внутри окна видит страницу и источники этой модалки. Страничный источник с тем же именем сохраняет приоритет. При открытии `onModalToggle(..., { managed: true })` исключает повторную клиентскую загрузку; интеграция — [[setup#Подключение Actions|Подключение Actions]].

## Связанные разделы

[[concepts|Основные понятия]] | [[options|Опции]] | [[types|Типы]] | [[setup|Подключение]] | [[hooks|Хуки]]

## Исполнение Actions

Общий runtime в `components/Dashboard/actions` выполняет действия последовательно: callback предыдущего вызова завершается до следующего вызова массива. `WidgetActionsProvider` создаёт области root/page своего `WidgetType`; окно дополняет их реестром модалки. Разрешение ID идёт от внутренней области к внешней. [[actions|Контракт и сценарии]].

```text
Клик элемента / записи / сегмента
  → useActionBindings (контекст записи + sourceKey)
  → createActionRuntime → executeActionSequence
      → resolveActionInvocation (ссылка/inline, scopes)
      → resolveActionParameters (живые фильтры + снимок клика)
      → исполнитель runTask / setFilters / openUrl / openModal
      → callback: первый истинный вариант или else
```

`TaskExecutionProvider` внутри `GlobalProvider` наблюдает независимые Python-задачи дольше жизни страницы. `usePythonTask` и Actions используют один сервис, сборку параметров ресурса и уведомления. Уход из контекста отменяет продолжения; повторный клик или ручная остановка дополнительно запрашивают остановку серверной задачи. Смена страницы сама по себе её не останавливает.

`WidgetModalsProvider` и `WidgetModalHost` хранят одно окно на ID и контекст каждой ревизии открытия. `ActionContextProvider` несёт данные клика/задачи/параметров, `ActionScopeProvider` — реестры и сигнал отмены, `ModalDataContext` — данные источников окна. Загрузка учитывает резолвленные параметры, условия, `debounce` и autoSync; закрытие либо смена ревизии исключает применение поздних ответов. Поиск визуального узла через [[utils#findDashboardNode|findDashboardNode]] обходит только `children`, `header`, `modals` и пропускает реестры Actions.
