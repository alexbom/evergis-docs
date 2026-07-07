# Подключение

## DashboardProvider (client-new)

Файл: `src/components/Dashboard/components/DashboardProvider.tsx`

Клиентская обёртка над `BaseDashboardProvider`. Агрегирует Redux-состояние и результаты хуков, передаёт всё в базовый провайдер.

### Внутренние хуки

| Хук | Что возвращает |
|---|---|
| `useProject()` | `projectInfo`, `updateProject` |
| `useSelectedTab()` | `selectedTabId`, `setSelectedTabId` |
| `useExpandableContainers()` | `expandedContainers`, `expandContainer` |
| `useProjectDataSourceFilters()` | `filters`, `changeFilters` |
| `useDashboardPages()` | `nextPage`, `prevPage`, `changePage` |
| `useDashboardLayers()` | `dashboardLayers`, `setDashboardLayer` |
| `useLayersListVisibility()` | `isVisible`, `toggleVisibility` |
| `useDialog()` | `openDialog` — открытие диалога каталога ресурсов |
| `useValidateDashboardConfig()` | side-effect: прогоняет активный конфиг через клиентский рантайм-валидатор `validateDashboardConfig` (`utils/validateDashboardConfig.ts`) — ловит пропуски `id`, неверные slot-id, фильтры без `filterName`, висячие ссылки (см. [[authoring\|Правила генерации]]) |

Колбэк `selectAttachmentsFromCatalog` открывает диалог `DIALOGS.RESOURCE_CATALOG` (`ResourceCatalogOptions`) и передаёт выбранные `CatalogResourceDc[]` через `onApply`.

### Redux-селекторы

| Селектор | Слайс |
|---|---|
| `getProjectPageIndex` | `dashboard` |
| `getProjectDataSources` | `dashboard` |
| `getProjectDataSourcesAreLoading` | `dashboard` |
| `getProjectGeometryFilter` | `project` |
| `getReferenceLayerInfos` | `dashboard` |

### Кастомные компоненты

Передаются в `BaseDashboardProvider` через `components`:
- `LayerItem` — элемент списка слоёв
- `ProjectPagesMenu` — меню переключения страниц
- `ProjectPanelMenu` — меню панели проекта

```tsx
<BaseDashboardProvider
  config={config}
  pageIndex={pageIndex}
  projectInfo={projectInfo}
  dataSources={dataSources}
  filters={filters}
  selectAttachmentsFromCatalog={selectAttachmentsFromCatalog}
  components={{ LayerItem, ProjectPagesMenu, ProjectPanelMenu }}
  ...
>
  {children}
</BaseDashboardProvider>
```

---

## BaseDashboardProvider (@evergis/react)

Файл: `contexts/DashboardContext/index.tsx`

Тонкий `memo`-компонент, создающий `DashboardContext` из всех переданных props. Конфиг `config: ConfigContainer` содержит `children: DashboardChild[]` (discriminated union из [[types#Дискриминированный union DashboardChild|типов]]).

### Props (`DashboardContextProps`)

| Prop | Тип | Описание |
|---|---|---|
| `config` | `ConfigContainer` | Конфигурация дашборда |
| `containerIds` | `string[]` | Список ID контейнеров |
| `projectInfo` | `ExtendedProjectInfoDc` | Информация о проекте |
| `updateProject` | `(info) => void` | Обновить проект |
| `pageIndex` | `number` | Текущий индекс страницы |
| `layerInfos` | `QueryLayerServiceInfoDc[]` | Информация о слоях |
| `dataSources` | `WidgetDataSource[]` | Загруженные источники данных |
| `geometryFilter` | `boolean` | Геометрический фильтр активен |
| `loading` | `boolean` | Идёт загрузка данных |
| `editMode` | `boolean` | Режим редактирования |
| `filters` | `SelectedFilters` | Активные фильтры |
| `dashboardLayers` | `DashboardState["layers"]` | Состояние слоёв |
| `setDashboardLayer` | `(payload) => void` | Установить параметры слоя |
| `selectedTabId` | `string` | ID выбранной вкладки |
| `setSelectedTabId` | `(tab) => void` | Выбрать вкладку |
| `changeFilters` | `(filters, reset?) => void` | Изменить фильтры |
| `expandedContainers` | `Record<string, boolean>` | Раскрытые контейнеры |
| `expandContainer` | `(id, expanded?) => void` | Раскрыть/свернуть контейнер |
| `nextPage` | `(total) => void` | Перейти на следующую страницу |
| `prevPage` | `(total) => void` | Перейти на предыдущую страницу |
| `changePage` | `(index) => void` | Перейти на конкретную страницу |
| `visibleLayers` | `boolean` | Список слоёв видим |
| `toggleLayersVisibility` | `VoidFunction` | Переключить видимость слоёв |
| `selectAttachmentsFromCatalog` | `(onApply: (resources: CatalogResourceDc[]) => void) => void` | Выбор вложений из каталога ресурсов |
| `components` | `{ LayerItem?, ProjectPanelMenu?, ProjectPagesMenu? }` | Кастомные компоненты |

---

## FeatureCardProvider (@evergis/react)

Файл: `contexts/FeatureCardContext/index.tsx`

Провайдер контекста карточки объекта. Принимает данные выбранного feature и информацию слоя.

### Ключевые props (`FeatureCardContextSettings`)

| Prop | Тип | Описание |
|---|---|---|
| `config` | `ConfigContainer` | Конфиг карточки |
| `attributes` | `ClientFeatureAttribute[]` | Атрибуты объекта |
| `layerInfo` | `QueryLayerServiceInfoDc` | Информация о слое |
| `feature` | `SelectedFeature` | Выбранный объект |
| `pageIndex` | `number` | Текущая страница |
| `isRaster` | `boolean` | Карточка растрового объекта |
| `editMode` | `boolean` | Режим редактирования |
| `isFeatureEditable` | `boolean` | Можно ли редактировать объект |
| `hasCopyRights` | `boolean` | Есть ли права на копирование |
| `editOnly` | `boolean` | Режим «только редактирование» |
| `selectedTabId` | `string` | ID выбранной вкладки |
| `setSelectedTabId` | `(id) => void` | Выбрать вкладку |
| `dataSources` | `WidgetDataSource[]` | Источники данных карточки |
| `loading` | `boolean` | Идёт загрузка данных |
| `filters` | `SelectedFilters` | Активные фильтры |
| `controls` | `Record<string, EditAttributeValue>` | Значения edit-контролов |
| `changeControls` | `(controls) => void` | Обновить контролы |
| `changeFilters` | `(filters) => void` | Изменить фильтры |
| `closeFeatureCard` | `VoidFunction` | Закрыть карточку |
| `expandedContainers` | `Record<string, boolean>` | Раскрытые контейнеры |
| `expandContainer` | `(id, expanded?) => void` | Раскрыть/свернуть контейнер |
| `nextPage`, `prevPage`, `changePage` | функции | Навигация по страницам |

---

## GlobalProvider (@evergis/react)

Файл: `contexts/GlobalContext/index.tsx`

Глобальный контекст с данными, необходимыми всем компонентам: i18n, API, геометрия, тема.

### Props (`GlobalContextProps`)

| Prop | Тип | Описание |
|---|---|---|
| `t` | `i18n["t"]` | Функция перевода |
| `language` | `string` | Язык интерфейса |
| `ewktGeometry` | `string` | EWKT-геометрия для геофильтра |
| `themeName` | `ThemeName` | Тема (`Dark` / `Light`) |
| `api` | `Api` | Экземпляр API-клиента (`@evergis/api`) |
| `notification` | `{ add, update, close }` | API уведомлений (`INotificationItem`). Нужен для прогресс-уведомлений серверных [[hooks|хуков]] `beforeSave`/`afterSave` |

```tsx
<GlobalProvider api={api} t={t} ewktGeometry={geometry} themeName="Light">
  <DashboardProvider config={config} ...>
    ...
  </DashboardProvider>
</GlobalProvider>
```

---

## Главный компонент Dashboard (client-new)

Файл: `src/components/Dashboard/index.tsx`

Условный рендеринг на основе состояния Redux:

- `!isOpen` → `null` (дашборд закрыт)
- `isEmpty` → `<DashboardSoon />` (нет конфигурации)
- Иначе → `<DashboardWrapper>` с `<FiltersUpdatingOverlay />` (если `filtersUpdating`) + `<DashboardBase />`

---

## Связанные разделы

[[hooks|Хуки]] | [[components|Компоненты]] | [[requirements|Системные требования]] | [[architecture|Архитектура]] | [[types|Типы]]
