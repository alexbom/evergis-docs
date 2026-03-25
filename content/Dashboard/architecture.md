# Архитектура

## Паттерн конфигурационного управления

Dashboard использует registry-pattern: каждый контейнер и элемент регистрируется по ключу (`ContainerTemplate` enum или строковое название) в `registry.ts`. При рендеринге компонент определяется динамически по значению `templateName` / `type` из конфига.

```ts
// containers/registry.ts
export const containerComponents: Record<string, FC<ContainerProps>> = {
  [ContainerTemplate.Chart]: ChartContainer,
  [ContainerTemplate.DataSource]: DataSourceContainer,
  [ContainerTemplate.Filters]: FiltersContainer,
  // ...30+ записей
  default: ContainersGroupContainer,
};

// elements/registry.ts
export const elementComponents: Record<string, FC<ContainerProps>> = {
  chart: ElementChart,
  image: ElementImage,
  // ...15 записей
};
```

Разрешение компонента:
- Контейнер: `getContainerComponent(templateName)` → `containerComponents[name] || default`
- Элемент: `getRenderElement(...)` возвращает функцию, которая по `id` находит дочерний элемент в конфиге и рендерит нужный компонент

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

[[concepts|Основные понятия]] | [[setup|Подключение]] | [[hooks|Хуки]]
