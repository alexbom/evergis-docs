# Подключение

## DashboardProvider (client-new)

Файл: `src/providers/DashboardProvider/index.tsx`

Клиентская обёртка над `DashboardProvider` из `@evergis/react` (импортируется под алиасом `BaseDashboardProvider`). Агрегирует Redux-состояние и результаты хуков, передаёт всё в базовый провайдер. Принимает единственный собственный проп — `config?: ConfigContainer` (нужен превью редактора контейнеров, которое подставляет свой конфиг вместо страничного).

### Внутренние хуки

| Хук | Откуда | Что возвращает |
|---|---|---|
| `useProject()` | `src/hooks` | `projectInfo`, `updateProject` |
| `useDialog()` | `src/hooks` | `openDialog` — открытие диалога каталога ресурсов |
| `useExpandableContainers()` | `@evergis/react` | `expandedContainers`, `expandContainer` |
| `useSelectedTab()` | `components/Dashboard/hooks/useSelectTab` | `selectedTabId`, `setSelectedTabId` |
| `useProjectModals()` | `components/Dashboard/hooks/useProjectModals` | `[openedModalIds, onModalToggle]` — открытые модалки в слайсе `dashboard` (`toggleProjectModal`); `onModalToggle` уходит в базовый провайдер |
| `useProjectDataSourceFilters()` | `components/Dashboard/hooks` | `filters`, `changeFilters` |
| `useDashboardPages()` | `components/Dashboard/hooks` | `nextPage`, `prevPage`, `changePage` |
| `useDashboardLayers()` | `components/Dashboard/hooks` | `dashboardLayers`, `setDashboardLayer` |
| `useValidateDashboardConfig()` | `components/Dashboard/hooks` | side-effect: прогоняет активный конфиг через клиентский рантайм-валидатор `validateDashboardConfig` (`components/Dashboard/utils/validateDashboardConfig.ts`) — ловит пропуски `id`, неверные slot-id, фильтры без `filterName`, висячие ссылки, слот `bgImage` у `Divider` и `options.outflow` без этого слота (`orphan-option`). Источник берётся из стора (`content.dashboardConfiguration`), а не из пропа `config` провайдера; в production — no-op (см. [[authoring\|Правила генерации]]) |
| `useLayersListVisibility()` | `components/MainPanel/hooks` | `isVisible`, `toggleVisibility` |

Колбэк `selectAttachmentsFromCatalog` открывает диалог `DIALOGS.RESOURCE_CATALOG` (`ResourceCatalogOptions`) и передаёт выбранные `CatalogResourceDc[]` через `onApply`.

### Redux-селекторы

| Селектор | Слайс |
|---|---|
| `getProjectPageIndex` | `dashboard` |
| `getProjectDataSources` | `dashboard` |
| `getProjectDataSourcesAreLoading` | `dashboard` |
| `getProjectOpenedModalIds` | `dashboard` |
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
| `editMode` | `boolean` | Режим редактирования **атрибутов объекта** (в контейнерах — `isEditing`). К раскладке отношения не имеет |
| `onContainerChange` | `(config: ConfigContainerChild) => void` | Контейнер изменил свой конфиг — см. раздел ниже |
| `onModalToggle` | `(modalId: string, isOpen: boolean) => void` | Модалка `config.modals[].id` открылась или закрылась — хост грузит её `dataSources`, пока она открыта. См. [[setup#Ленивые источники модалок (client-new)\|раздел ниже]] |
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

### Сохранение изменений раскладки

`onContainerChange` — единственная точка, через которую контейнер сообщает наружу новую версию собственного конфига. Сейчас его вызывает только [[containers#Редактирование раскладки editMode|сетка в режиме редактирования]]: после перетаскивания границы или операции над ячейками.

Внутрь дашборда колбэк уходит по цепочке «контекст → `useWidgetContext` → `PagesContainer` → `getRenderElement({ onChange })` → проп `onChange` контейнера». Тот же проп есть у `FeatureCardProvider`.

Приходит **весь узел целиком** и уже с новым содержимым; его `id` лежит внутри, поэтому применяется точечной заменой:

```tsx
import { replaceObject } from "find-and";

<DashboardProvider
  config={config}
  onContainerChange={next => {
    const newProjectInfo = JSON.parse(JSON.stringify(projectInfo));

    newProjectInfo.content.dashboardConfiguration = replaceObject(
      newProjectInfo.content.dashboardConfiguration,
      { id: next.id },
      next,
    );

    updateProject(newProjectInfo);
  }}
  ...
/>
```

> [!info] Редактирование работает и без обработчика
> Сетка держит собственный черновик раскладки, поэтому без `onContainerChange` правки видны, но не сохраняются. Если хост кладёт результат обратно в конфиг, зацикливания не будет: контейнер узнаёт по ссылке конфиг, который сам же и отдал.

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
| `editMode` | `boolean` | Режим редактирования атрибутов объекта |
| `onContainerChange` | `(config: ConfigContainerChild) => void` | Контейнер изменил свой конфиг — см. [[setup#Сохранение изменений раскладки\|раздел выше]] |
| `onModalToggle` | `(modalId: string, isOpen: boolean) => void` | Модалка карточки открылась или закрылась — см. [[setup#Ленивые источники модалок (client-new)\|раздел ниже]]. В client-new передаётся из `useFeatureModals()` (слайс `feature`, `toggleFeatureModal`) |
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

Глобальный контекст с данными, необходимыми всем компонентам: i18n, API, геометрия, текущий проект, тема.

### Props (`GlobalContextProps`)

| Prop | Тип | Описание |
|---|---|---|
| `t` | `i18n["t"]` | Функция перевода |
| `language` | `string` | Язык интерфейса |
| `ewktGeometry` | `string` | EWKT-геометрия геофильтра — область, нарисованная пользователем (`SRID=3857`) |
| `ewktExtent` | `string` | EWKT-экстент видимой области карты (`SRID=3857`) — плейсхолдер `%extent` |
| `zoomLevel` | `number` | Целый уровень зума карты — плейсхолдер `%zoom` |
| `projectName` | `string` | Системное имя открытого проекта — плейсхолдеры `%project` и `%project.name` |
| `projectAlias` | `string` | Алиас открытого проекта — плейсхолдер `%project.alias`, при пустом значении используется `projectName` |
| `themeName` | `ThemeName` | Тема (`Dark` / `Light`) |
| `api` | `Api` | Экземпляр API-клиента (`@evergis/api`) |
| `notification` | `{ add, update, close }` | API уведомлений (`INotificationItem`). Нужен для прогресс-уведомлений серверных [[hooks\|хуков]] `beforeSave`/`afterSave` |

```tsx
<GlobalProvider api={api} t={t} ewktGeometry={geometry} ewktExtent={extent} zoomLevel={zoom} themeName="Light">
  <DashboardProvider config={config} ...>
    ...
  </DashboardProvider>
</GlobalProvider>
```

### Кто заполняет свойства карты (client-new)

Файл: `src/providers/GlobalProvider/index.tsx` — обёртка отрендерена внутри `MapProvider`, поэтому обоим хукам доступен `useMapContext()`.

| Проп | Хук | Что делает |
|---|---|---|
| `ewktGeometry` | `useEwktGeometry()` (`src/hooks/map`) | EWKT нарисованного выделения с буфером |
| `ewktExtent`, `zoomLevel` | `useMapView()` (`src/hooks/map`) | Снимок вида карты по событию `idle`; `getMapView` (`src/evergis/map`) клампит bounds по границам меркатора и конвертирует через `geometryToEwkt` |

Подписка идёт на `idle`, а не на `moveend`: последний ловит и синтетические события от `stop()` во время программного `flyTo` и отдавал бы промежуточные координаты интерполяции.

Перезапрос источников при смене вида карты делает `useMapViewRefetch` (`components/Dashboard/hooks`), отбирая источники через [[utils|`getMapViewDataSources`]]; пер-источниковую задержку `debounce` применяет `useDataSourceDebounce` — общая обёртка над загрузчиком, через которую проходят все триггеры перезапроса.

Слои карты берут `ewktExtent` и `zoomLevel` из этого же контекста — `useTempLayerConditions` и `useTempLayerParams` (`src/hooks/project`) вызывают `useGlobalContext()`, поэтому подписка на `idle` в приложении ровно одна.

---

## Главный компонент Dashboard (client-new)

Файл: `src/components/Dashboard/index.tsx`

Условный рендеринг по результату хука `useDashboardStatus()`:

- `!isOpen` → `null` (дашборд закрыт)
- `isEmpty` → `<DashboardSoon />` (у текущей страницы нет конфигурации)
- Иначе → `<DashboardWrapper>` с `<FiltersUpdatingOverlay />` (если `filtersUpdating` из слайса `dashboard`) + `<DashboardBase />` из `@evergis/react`

`useDashboardStatus` не только считает флаги (`useDashboardsOpen` + `isEmpty(currentPage)`), но и запускает загрузку: внутри он вызывает `useReferenceLayerInfos()` (метаданные слоёв страницы и открытых модалок) и `useProjectDataSources()` (источники данных страницы, вместе с подписками `autoSyncLayer` / `autoSyncLayers` через `useDataSourceSubscriptions` — см. [[concepts#Real-time обновления|Real-time обновления]]).

Какие именно источники уйдут в запрос, решает утилита `getUnloadedDataSources` (`components/Dashboard/utils`): источник считается загруженным, только если в сторе под его именем лежит **массив** `features` и сигнатура запроса совпадает с текущей. Сигнатура собирается из `DATA_SOURCE_REQUEST_FIELDS` (`ds`, `query`, `parameters`, `condition`, `layerName`, `limit`, `url`, `resourceId`, `fileName`, `methodName`). После ошибки в сторе остаётся `null` — такой источник запрашивается снова. Отдельный вход `outdatedNames` добивает случай, когда сигнатура та же, а значения фильтров уже другие.

**Правка источников в конфиге текущей страницы** (например, в редакторе) перезапрашивает только источники с изменившимся запросом, без сброса стора — иначе обнулялись бы соседние графики. Старый и новый наборы сравниваются утилитой `getQueryRelevantDataSources` (`components/Dashboard/utils`), которая выбрасывает из каждого источника секцию `attributes`: оверлей атрибутов (`alias`, `type`, `stringFormat`, `description`) на серверный запрос не влияет и рендерится прямо из конфига (`getDataSourceLayerInfo`), поэтому правка формата или подписи атрибута в сеть не ходит. Смена страницы этот механизм не задействует — там своя загрузка.

---

## Ленивые источники модалок (client-new)

Источники из `config.modals[].dataSources` (см. [[elements#ElementModal|ElementModal]]) грузятся не со страницей, а при открытии модалки. `ElementModal` сообщает об открытии и закрытии через проп `onModalToggle`; провайдеры кладут id в слайсы (`dashboard.openedModalIds` / `feature.openedModalIds`). Загрузчики — `useProjectDataSources` (дашборд) и `useFeatureDataSources` (карточка) — работают через общие хуки из `components/Dashboard/hooks`:

| Хук | Что делает |
|---|---|
| `useModalDataSources(type)` | Наборы источников: `activeDataSources` (страница + открытые модалки — то, что грузится), `allDataSources` (страница + все модалки, из [[hooks#useConfigDataSources\|useConfigDataSources]]), `openedModalDataSources`, `closedModalNames`, `configWithModals` (`currentPage` с `allDataSources` — конфиг для [[hooks#useDataSources\|useDataSources]]) |
| `useModalAwareFetch(type, fetchData)` | Обёртка загрузчика для событий перезапроса (фильтры, `%extent`/`%zoom`, autoSync): источники **закрытых** модалок не грузятся, а **вытесняются из стора** |
| `useOpenedModalsFetch(type, fetchData, isBlocked?)` | При открытии модалки запрашивает её источники, которых нет в сторе (`getUnloadedDataSources`). Запрошенные имена помнит до закрытия — упавший запрос не зацикливается; `isBlocked` откладывает загрузку, пока карточка грузит объект |

Жизненный цикл данных модалки:

- **Открыта** — источники ведут себя как страничные: фильтры, движение карты, autoSync, `debounce`.
- **Закрыта** — данные остаются в сторе (повторное открытие мгновенное). Событие, которое перезапросило бы источник, вместо запроса удаляет его данные — при следующем открытии он окажется незагруженным и запросится с актуальными фильтрами и видом карты.
- **Смена страницы / контекста / объекта карточки** — стор сбрасывается целиком, модальный кэш вместе с ним. Размонтированная открытая модалка сама сообщает о закрытии.

Подписки autoSync (`useDataSourceSubscriptions`, `useFeatureDataSourceSubscriptions`) оформляются сразу на слои **всех** модалок: подписка ставится один раз на соединение, и модалка, открытая позже, иначе осталась бы без неё.

Клиентский валидатор ([[authoring|Правила генерации]]) предупреждает `modal-datasource-shadowed`, если имя источника модалки совпадает со страничным или корневым: такой источник перекрыт и грузится вместе со страницей.

---

## Наполнение текущей страницы (client-new)

Два хука дописывают сущности в конфиг текущей страницы из панелей каталога — оба ходят через `updateConfigPage` из [[hooks|`useWidgetPage`]] и подсвечивают счётчик новинок соответствующей панели (`useNewItemsUpdate`):

| Хук | Что добавляет | Особенности |
|---|---|---|
| `useAddCurrentPageDataSources()` | `(sourcesToAdd: ConfigDataSource[]) => void` — источники в `currentPage.dataSources` | дедупликации нет: вызывающая сторона отдаёт уже отобранное; счётчик панели `Data` |
| `useAddCurrentPageTasks()` | `(resources: CatalogResourceDc[]) => void` — задачи в `currentPage.tasks` | ресурсы каталога конвертируются `catalogResourceToConfigTask`; дубли отсекаются по `systemName` / `resourceId` уже добавленных задач, задачи без `name` отбрасываются; счётчик панели `Tools`. Пустой результат — выход без записи в конфиг |

---

## Связанные разделы

[[hooks|Хуки]] | [[components|Компоненты]] | [[requirements|Системные требования]] | [[architecture|Архитектура]] | [[types|Типы]]
