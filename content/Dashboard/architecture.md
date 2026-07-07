# Архитектура

## Паттерн конфигурационного управления

Dashboard использует registry-pattern: каждый контейнер и элемент регистрируется по ключу (`ContainerTemplate` enum для контейнеров, строковый `type` для элементов) в `registry.ts`. При рендеринге компонент определяется динамически по значению `templateName` / `type` из конфига.

```ts
// containers/registry.ts — as const satisfies ContainerComponentRegistry
export const containerComponents = {
  [ContainerTemplate.Chart]: ChartContainer,
  [ContainerTemplate.DataSource]: DataSourceContainer,
  [ContainerTemplate.Filters]: FiltersContainer,
  // ...34 записи + default
  default: ContainersGroupContainer,
} as const satisfies ContainerComponentRegistry;

// elements/registry.ts — as const satisfies ElementComponentRegistry
export const elementComponents = {
  chart: ElementChart,
  image: ElementImage,
  // ...15 записей
} as const satisfies ElementComponentRegistry;
```

Реестры — типобезопасные: `ContainerComponentRegistry` и `ElementComponentRegistry` (см. [[types#Типизированные реестры|Типизированные реестры]]) гарантируют, что каждый зарегистрированный компонент принимает корректные `<Name>Props`.

Разрешение компонента:
- Контейнер: `getContainerComponent(templateName)` → `containerComponents[name] || default`
- Элемент: `getRenderElement(...)` возвращает функцию, которая по `id` находит дочерний элемент в конфиге и рендерит нужный компонент

## Типизация

Типы сгруппированы в три слоя:

1. **Per-component тройка** `<Name>Options` / `<Name>Config` / `<Name>Props` — для каждого контейнера, элемента и шапки в `componentTypes.ts`. Сводная таблица — [[types#Per-component типы|Per-component типы]].
2. **Branded keyspaces** — `ChartId`, `ModalId`, `TabId`, `FilterName`, `LayerName`, `AttributeName`, `DataSourceName`, `ResourceId` в `branded.ts`. Защищают от перепутывания entity-id на этапе компиляции. См. [[types#Branded types|Branded types]].
3. **Доменная группировка опций** — `ConfigOptions extends` 12 миксинов (`ConfigLayoutOptions`, `ConfigTypographyOptions`, `ConfigChartOptions`, ...). См. [[options|Опции]].

Discriminated union `DashboardChild` (15 вариантов элементов + 33 варианта контейнеров) и `DashboardHeaderConfig` (4 ветви шапок) позволяют TS сужать ветвь по полю `type` / `templateName` и валидировать `options`. Для авторинга новых конфигов есть строгие варианты `StrictConfigContainerChild` / `StrictDashboardChild`, у которых `id` обязателен (пропуск = ошибка компиляции). Детали — [[types#Дискриминированный union DashboardChild|Дискриминированный union]].

## Иерархия компонентов

```
GlobalProvider (contexts/GlobalContext)
└── BaseDashboardProvider / FeatureCardProvider (contexts/*Context)
    └── Dashboard (components/Dashboard/index.tsx)
        ├── DashboardHeader (components/DashboardHeader)
        │   └── DashboardDefaultHeader / FeatureCardDefaultHeader / ...
        └── PagesContainer (containers/PagesContainer)
            └── ContainerChildren (components/ContainerChildren)
                └── [ContainerComponent по registry]
                    │   (ContainersGroupContainer, ChartContainer, ...)
                    └── renderElement({ id })
                        └── [ElementComponent по registry]
                            (ElementChart, ElementImage, ...)
```

Для `FeatureCard` схема аналогична, но использует `FeatureCardContext` и `WidgetType.FeatureCard`.

## Контексты

### DashboardContext
Хранит конфиг дашборда, состояние страниц, источники данных, фильтры, слои, expandedContainers, кастомные компоненты. Читается через `useWidgetContext(WidgetType.Dashboard)`.

### FeatureCardContext
Хранит атрибуты и конфиг карточки объекта, layerInfo, controls, edit-состояние. Читается через `useWidgetContext(WidgetType.FeatureCard)`.

### GlobalContext
Глобальные настройки: `api`, `t`, `ewktGeometry`, `themeName`, `language`. Читается через `useGlobalContext()`.

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

## Связанные разделы

[[concepts|Основные понятия]] | [[options|Опции]] | [[types|Типы]] | [[setup|Подключение]] | [[hooks|Хуки]]
