# Хуки

## Обзор

Хуки расположены в `D:/projects/spcore_api/packages/react/src/components/Dashboard/hooks/`. Предоставляют бизнес-логику для компонентов Dashboard и FeatureCard.

Публичные экспорты — см. `hooks/index.ts`. Internal-хуки (`useEditControl`, `useRenderContainer`, `useRenderContainerItem`) используются внутренними контейнерами и не экспортируются наружу.

Серверные хуки сохранения (`useFeatureSaveHooks`, `useBeforeSave`, `useAfterSave`, `useSavePrototypeBuilder`) реализуют механизм `beforeSave`/`afterSave` python-скриптов — концептуально описан в [[concepts#Серверные хуки сохранения (beforeSave / afterSave)|Основных понятиях]].

---

## useAfterSave

**Назначение:** Фабрика fire-and-forget действия после успешного сохранения объекта в FeatureCard. Запускает python-скрипт (`afterSave`) через remote task; показывает уведомление о прогрессе/успехе/ошибке, но **не блокирует** сохранение и ничего не возвращает в поток сохранения.

**Параметры:**

| Параметр | Тип |
|---|---|
| `hook` | `ConfigRelatedResource \| undefined` — описание `afterSave`-скрипта из `editConfiguration.options` |
| `buildPrototype` | `(hook, input: SaveHookInput) => TaskPrototypeDto` — билдер прототипа задачи (из `useSavePrototypeBuilder`) |

**Возвращает:** `(input: SaveHookInput) => Promise<void>`

Если `hook` неактивен (нет `resourceId` и `fileName` — см. `isHookActive`) — действие пропускается. См. [[types|тип]] `ConfigRelatedResource`, `SaveHookInput`.

```ts
const runAfterSave = useAfterSave(options?.afterSave, buildPrototype);
await runAfterSave({ featureId, changedProperties });
```

---

## useAttachmentItems

**Назначение:** Извлечение списка вложений (`Attachment[]`) из атрибута объекта или связанного источника данных. Используется в `AttachmentContainer` и `EditAttachmentContainer`.

**Параметры:**

| Параметр | Тип |
|---|---|
| `type` | `WidgetType?` |
| `elementConfig` | `ConfigContainerChild?` |
| `valueOverride` | `unknown?` |

**Возвращает:** `{ items: Attachment[], attributeName?: string, rawValue: unknown }`

Если `options.relatedDataSource` задан — берёт features из источника через `attachmentsFromFeatures(features, mapping)` (mapping в `controls[0]`). Иначе парсит сырое значение через `parseAttachments(rawValue)`.

```ts
const { items } = useAttachmentItems({ type, elementConfig });
```

---

## useAttachmentPreviewImages

**Назначение:** Превращает список `Attachment[]` в `IPreviewImage[]` для компонента `Preview` из `@evergis/uilib-gl`. Для изображений из защищённого хранилища загружает blob через `api.catalog.getFile`, кэширует `URL.createObjectURL`, отслеживает loading/error. Для внешних URL — отдаёт ссылку напрямую. Для не-изображений — иконку типа файла через `getFileTypeIcon`.

**Параметры:**

| Параметр | Тип |
|---|---|
| `items` | `Attachment[]` |
| `active` | `boolean` |

**Возвращает:** `IPreviewImage[]` — `{ src, fileName, hasError?, isLoading? }[]`

При размонтировании автоматически вызывает `URL.revokeObjectURL` для всех созданных blob URL.

---

## useAutoCompleteControl

**Назначение:** Состояние автозаполнения для edit-контрола. Хранит текущее введённое значение и список опций; статические опции собираются из переданного списка значений, динамические — задаются через `setOptions`.

**Параметры:**

- `items` — `(string | number | boolean | null | undefined)[]?` — значения для статических опций (фильтруются по `Boolean`)

**Возвращает:**

| Поле | Тип |
|---|---|
| `value` | `string` |
| `setValue` | `(value: string) => void` |
| `onChange` | `(value: string) => void` |
| `options` | `IOption[]` — статические (если есть) либо динамические |
| `setOptions` | `(options: IOption[]) => void` |

```ts
const { value, options, onChange } = useAutoCompleteControl(variants);
```

---

## useBeforeSave

**Назначение:** Фабрика синхронной серверной проверки перед сохранением объекта в FeatureCard. Запускает python-скрипт (`beforeSave`) через remote task и **ждёт** его завершения: при `RemoteTaskStatus.Completed` resolves `true` (сохранение продолжается), иначе — `false` (сохранение должно быть отменено). Показывает уведомление о прогрессе/успехе/ошибке (текст ошибки берётся из `log` задачи).

**Параметры:**

| Параметр | Тип |
|---|---|
| `hook` | `ConfigRelatedResource \| undefined` — описание `beforeSave`-скрипта из `editConfiguration.options` |
| `buildPrototype` | `(hook, input: SaveHookInput) => TaskPrototypeDto` — билдер прототипа задачи (из `useSavePrototypeBuilder`) |

**Возвращает:** `(input: SaveHookInput) => Promise<boolean>` — `false` означает «проверка не пройдена, отменить save»

Если `hook` неактивен (нет `resourceId` и `fileName`) — возвращает `true` (проверка пропускается, сохранение не блокируется). См. [[concepts#Серверные хуки сохранения (beforeSave / afterSave)|Основные понятия]].

```ts
const runBeforeSave = useBeforeSave(options?.beforeSave, buildPrototype);

if (!(await runBeforeSave({ featureId, changedProperties }))) return; // save отменён
```

---

## useChartChange

**Назначение:** Кастомизация визуального отображения чарта (цвета, ширина, маркеры). Использует `@evergis/charts` customize API.

**Параметры:**
| Параметр | Тип | Описание |
|---|---|---|
| `dataSources` | `ConfigDataSource[]` | Источники данных страницы |
| `chartId` | `string` | ID элемента чарта |
| `width`, `height` | `number \| string` | Размеры |
| `fontColor` | `string` | Цвет текста осей |
| `relatedAttributes` | `ConfigRelatedDataSource[]` | Связанные атрибуты |
| `defaultColor` | `string` | Цвет по умолчанию |
| `markers` | `BarChartMarker[] \| string` | Маркеры |
| `showMarkers` | `number` | Шаг показа маркеров |

**Возвращает:** `[customize]` — функция кастомизации для передачи в чарт

---

## useChartData

**Назначение:** Получение и форматирование данных для чарта из источников данных или атрибутов объекта.

**Параметры:**
| Параметр | Тип |
|---|---|
| `element` | `ConfigContainerChild` — конфиг чарт-элемента |
| `type` | `WidgetType` |

**Возвращает:** `{ data: ChartDataProps[], loading: boolean }`

`ChartDataProps`: `{ items, layerInfo, attributeName, attributeUnits, dataSourceName, color }`

```ts
const { data, loading } = useChartData({ element: chartElement, type });
```

---

## useContainerAttributes

**Назначение:** Подготовка списка атрибутов и фабрики рендера для `OneColumnContainer` и `TwoColumnContainer`. Учитывает `options.attributes` (явный список) или все доступные `attributes` контекста + фильтр `useProjectHiddenAttributes` (см. `useLayerHiddenAttributes`).

**Параметры:** `Pick<ContainerProps, "elementConfig" | "type" | "renderElement">`

**Возвращает:** `{ getRenderContainerItem, attributesToRender }`
- `getRenderContainerItem` — фабрика из `useRenderContainerItem`
- `attributesToRender` — `string[] | null` (null если `options.attributes` не задан)

---

## useDashboardHeader

**Назначение:** Данные для шапки дашборда (заголовок, иконка, изображение, тема).

**Параметры:** нет

**Возвращает:**
| Поле | Тип |
|---|---|
| `pageId` | `string` |
| `image` | `string \| null` |
| `icon` | `ReactNode` |
| `title` | `ReactNode` |
| `url` | `string` |
| `tooltip` | `ReactNode` |
| `themeName` | `ThemeName` |
| `onClickLogo` | `VoidFunction` |

```ts
const { title, icon, onClickLogo } = useDashboardHeader();
```

---

## useDataSources

**Назначение:** Базовый хук получения данных. Поддерживает EQL-запросы, layer features, Python remote tasks, URL-эндпоинты.

**Параметры:**
| Параметр | Тип |
|---|---|
| `type` | `WidgetType` |
| `config` | `ConfigContainerChild` — конфиг страницы |
| `attributes` | `ClientFeatureAttribute[]?` |
| `filters` | `SelectedFilters` |
| `layerParams` | `Record<string, string>?` |
| `eqlParameters` | `QueryLayerServiceConfigurationDc["eqlParameters"]?` |

**Возвращает:**
| Поле | Описание |
|---|---|
| `getDataSourcePromises(ds, newFilters?, offset?)` | Загрузить один источник данных |
| `getUpdatingDataSources()` | Вернуть источники, затронутые изменившимися фильтрами |
| `getUpdatedDataSources(responses, current, other)` | Смерджить ответы в массив `FetchedDataSource` |
| `zoomToLayersExtent(layers)` | Zoom to экстент слоёв |

```ts
const { getDataSourcePromises, getUpdatingDataSources } = useDataSources({ type, config: currentPage, filters });
```

---

## useDiffPage

**Назначение:** Определяет, изменилась ли страница (используется для показа лоадера при переходе).

**Параметры:** `type: WidgetType`

**Возвращает:** `boolean` — `true` если переход на другую страницу ещё не завершён

---

## useEditControl

**Назначение:** Логика для edit-контролов. Находит контрол по `targetAttributeName`, разрешает текущее значение (из `controls` или `attributes`, с фоллбэком на `defaultValue`), собирает список вариантов (из `variants` контрола или features связанного источника) и обработчик изменения. При наличии `defaultValue` и отсутствии значения проставляет его в `controls`. Internal-хук (не экспортируется).

**Параметры:**
| Параметр | Тип |
|---|---|
| `type` | `WidgetType` |
| `elementConfig` | `ConfigContainerChild` |

**Возвращает:** `{ control, value, dataSource, items, onChange }`
- `control` — найденный `ConfigControl | undefined`
- `value` — текущее значение атрибута (с учётом `defaultValue`)
- `dataSource` — связанный источник данных контрола
- `items` — `string[]` варианты значений
- `onChange` — `(value: EditAttributeValue) => void`

---

## useEditGroupAttributes

**Назначение:** Фильтрация атрибутов и контролов для `EditGroupContainer`. Исключает `idAttribute` слоя и (если `useProjectHiddenAttributes`) скрытые атрибуты проекта.

**Параметры:** `Pick<ContainerProps, "elementConfig" | "type">`

**Возвращает:** `{ filteredAttributes, filteredControls }`
- `filteredAttributes` — `ClientFeatureAttribute[]` без `idAttribute` и скрытых
- `filteredControls` — `ConfigControl[] | undefined` без контролов, чьи `targetAttributeName` входят в `hiddenAttributes`

---

## useExpandableContainers

**Назначение:** Управление состоянием раскрытых/свёрнутых контейнеров.

**Параметры:** нет

**Возвращает:** `[expandedContainers: Record<string, boolean>, expandContainer: (id: string, expanded?: boolean) => void]`

```ts
const [expandedContainers, expandContainer] = useExpandableContainers();
```

---

## useExportPdf

**Назначение:** Экспорт DOM-элемента в PDF через `jsPDF` + `html2canvas`. Пагинирует по высоте дочерних элементов.

**Параметры:**
| Параметр | Тип | Default |
|---|---|---|
| `id` | `string` | — |
| `margin` | `number` | `20` |

**Возвращает:** `{ loading: boolean, onExport: VoidFunction }`

Имя файла: `yyyy-MM-dd_HH:mm:ss.pdf`

```ts
const { loading, onExport } = useExportPdf(getRootElementId(type));
```

---

## useFeatureSaveHooks

**Назначение:** Orchestrator-хук серверных `beforeSave` / `afterSave` python-скриптов. Читает их описание из `layerInfo.configuration.editConfiguration.options` (тип [[types|EditConfigurationOptions]]) текущего слоя FeatureCard и собирает готовые к вызову функции через **useBeforeSave** / **useAfterSave** и билдер из **useSavePrototypeBuilder**.

Активируется только если в [[setup|GlobalProvider]] переданы `api`, `notification`, `t`, а приложение обёрнуто в `<ServerNotificationsProvider>` (SignalR-подписка на прогресс задачи).

**Параметры:** нет

**Возвращает:**

| Поле | Тип | Описание |
|---|---|---|
| `runBeforeSave` | `(input: SaveHookInput) => Promise<boolean>` | Проверка перед сохранением; `false` → отменить save |
| `runAfterSave` | `(input: SaveHookInput) => Promise<void>` | Действие после успешного save (fire-and-forget) |

Вспомогательные функции и константы — в `useFeatureSaveHooks.utils.ts`: `isHookActive(hook)` (хук активен при наличии `resourceId` или `fileName`), `createSaveNotificationId()` (уникальный id уведомления), `SAVE_HOOK_RESULT_DURATION` (4000 мс — длительность показа результата).

```ts
const { runBeforeSave, runAfterSave } = useFeatureSaveHooks();

const onSave = async (input: SaveHookInput) => {
  if (!(await runBeforeSave(input))) return;
  await save();
  runAfterSave(input);
};
```

---

## useFetchImageWithAuth

**Назначение:** Загрузка изображения по URL с авторизацией (Bearer token из localStorage). Возвращает object URL.

**Параметры:** `url: string | null`

**Возвращает:** `string | null` — blob URL или `null`

```ts
const blobUrl = useFetchImageWithAuth(imageUrl);
```

---

## useFetchWithAuth

**Назначение:** Generic хук загрузки данных с авторизацией.

**Параметры:**
| Параметр | Тип |
|---|---|
| `url` | `string \| null` |
| `transform` | `(response: Response) => Promise<T>` |
| `cleanup` | `(data: T) => void` |

**Возвращает:** `T | null`

```ts
const data = useFetchWithAuth<MyType>(url, resp => resp.json(), () => {});
```

---

## useGetConfigLayer

**Назначение:** Возвращает геттер `ConfigLayer` из слоёв текущей страницы (через **useWidgetPage**) по имени слоя.

**Параметры:** нет

**Возвращает:** `(layerName: string) => ConfigLayer | undefined`

```ts
const getConfigLayer = useGetConfigLayer();
const layer = getConfigLayer("myLayer");
```

---

## useGlobalContext

**Назначение:** Доступ к `GlobalContext` (api, t, ewktGeometry, themeName, language).

**Параметры:** нет

**Возвращает:** `GlobalContextProps` (без `children`)

```ts
const { api, t, ewktGeometry } = useGlobalContext();
```

---

## useHeaderRender

**Назначение:** Создаёт функцию `renderElement` для шапок (header) дашборда / FeatureCard. Аналог `getRenderElement` для `ConfigContainerHeader`.

**Параметры:** `elementConfig: ConfigContainerHeader`, `type?: WidgetType` (default `Dashboard`)

**Возвращает:** `RenderElementFunction`

```ts
const renderElement = useHeaderRender(header);
// renderElement({ id: "title", wrap: false })
```

---

## useHideIfEmptyDataSource

**Назначение:** Возвращает функцию проверки — нужно ли скрыть элемент при пустом источнике данных.

**Параметры:** `type: WidgetType`

**Возвращает:** `(dataSourceName?: string) => boolean`

```ts
const checkIfEmpty = useHideIfEmptyDataSource(type);
if (checkIfEmpty(item.options?.hideIfEmptyDataSource)) return null;
```

---

## useProjectDashboardInit

**Назначение:** Инициализация данных дашборда при монтировании (загрузка данных текущей страницы). Используется в клиентской части `client-new`.

---

## useRelatedDataSourceAttributes

**Назначение:** Получение атрибутов объекта из источника данных (для контейнеров типа DataSource, RoundedBackground).

**Параметры:**
| Параметр | Тип |
|---|---|
| `type` | `WidgetType` (default `Dashboard`) |
| `elementConfig` | `ConfigContainerChild` |
| `dataSources` | `FetchedDataSource[]` |
| `feature` | `FeatureDc?` |

**Возвращает:** `{ attributes: ClientFeatureAttribute[], layerInfo: QueryLayerServiceInfoDc, dataSource?: FetchedDataSource }`

---

## useRenderContainer

**Назначение:** Internal-хук для контейнеров `OneColumnContainer` и `TwoColumnContainer`. Объединяет `useContainerAttributes` и `useRenderContainerItem`; поддерживает переопределение `templateName` по атрибуту через `getDisplayTemplateNameFromAttribute`.

**Параметры:**
| Параметр | Тип |
|---|---|
| `elementConfig`, `type`, `renderElement` | из `ContainerProps` |
| `renderBody` | `(item, attribute?) => ReactNode` — функция отрисовки тела |

**Возвращает:** `{ renderContainer(attribute?), attributesToRender }`

Если у атрибута есть override-templateName — рендерится соответствующий `<OverrideContainer>` через `getContainerComponent`; иначе вызывается `renderBody(item, attribute)`.

---

## useRenderContainerItem

**Назначение:** Фабрика для рендеринга одного элемента контейнера (OneColumn / TwoColumn). Internal-хук. Возвращает `getRenderContainerItem`.

**Параметры:** `type: WidgetType`, `renderElement: RenderElementFunction`

**Возвращает:** функция `(elementConfig, attributeName?) => { id, value, hideEmpty, style, hasIcon, hasUnits, render }`

---

## useRenderElement

**Назначение:** Хук, возвращающий `renderElement` функцию для рендеринга дочерних элементов по `id`. Собирает контекст (config, layerInfo, attributes, табы, страница) из **useWidgetContext** / **useWidgetConfig** / **useWidgetPage** и передаёт в `getRenderElement`.

**Параметры:** `type?: WidgetType` (default `Dashboard`), `elementConfig: ConfigContainerChild`

**Возвращает:** `RenderElementFunction`

---

## useSavePrototypeBuilder

**Назначение:** Билдер `TaskPrototypeDto` для запуска `beforeSave`/`afterSave` python-скрипта. Собирает параметры скрипта из контекста FeatureCard и Dashboard: подставляет фильтры/геометрию в `hook.parameters` через [[utils|утилиту]] `applyQueryFilters`, добавляет служебные `projectName`, `layerName`, `featureId`, объект `edit` (изменённые атрибуты + геометрия) и, при активном геометрическом фильтре карты, `selectionGeometry`. Используется внутри **useFeatureSaveHooks**.

**Параметры:** нет

**Возвращает:** `(hook: ConfigRelatedResource, input: SaveHookInput) => TaskPrototypeDto`

Структура собираемого payload (`scriptParameters`):

| Поле | Источник | Условие |
|---|---|---|
| `projectName` | `projectInfo.name` (Dashboard) | всегда |
| `layerName` | `layerInfo.name` (FeatureCard) | всегда |
| `featureId` | `input.featureId` | всегда (`null` при создании) |
| `selectionGeometry` | `ewktGeometry` из [[setup\|GlobalContext]] | только если геом-фильтр карты активен |
| `edit.attributes` | `input.changedProperties` | всегда |
| `edit.featureGeometry` | `geometryToEwkt(input.changedGeometry)` | только если геометрия объекта менялась |
| `...resolvedParameters` | `applyQueryFilters(hook.parameters, ...)` | подстановка фильтров/геометрии |

Прототип содержит один `pythonService`-subtask (`method: "pythonrunner/run"`) с `resourceId`/`fileName`/`methodName` из `hook` и собранными `parameters`. Полное описание того, что python-скрипт получает на входе, — [[concepts#Дата-контракт python-скрипта|дата-контракт]] в Основных понятиях.

```ts
const buildPrototype = useSavePrototypeBuilder();
const prototype = buildPrototype(hook, { featureId, changedProperties, changedGeometry });
```

---

## useShownOtherItems

**Назначение:** Управление пагинацией списков (shownItems / otherItems).

**Параметры:** `options: ConfigOptions`

**Возвращает:** `{ sliceItems(data), checkIsSliced(data), showMore, onShowMore }`

---

## useUpdateDataSource

**Назначение:** Перезагрузка данных одного источника (например, после изменения записи или для пагинации). Использует **useDataSources** под капотом.

**Параметры:** объект
| Параметр | Тип |
|---|---|
| `dataSource` | `ConfigDataSource` |
| `config` | `ConfigContainerChild` |
| `filters` | `SelectedFilters` |
| `attributes` | `ClientFeatureAttribute[]?` |
| `layerParams` | `Record<string, string>?` |
| `eqlParameters` | `QueryLayerServiceConfigurationDc["eqlParameters"]?` |

**Возвращает:** `(newFilters: SelectedFilters, offset?: number) => Promise<{ items, totalCount }>`

```ts
const updateDataSource = useUpdateDataSource({ dataSource, config, filters });
const { items, totalCount } = await updateDataSource(newFilters, offset);
```

---

## useWidgetConfig

**Назначение:** Доступ к конфигурации виджета — список страниц, текущий конфиг, header.

**Параметры:** `type?: WidgetType` (default `Dashboard`)

**Возвращает:** `{ config: ConfigContainer, pages: ConfigContainerChild[], header: ConfigContainerHeader }`

```ts
const { config, pages } = useWidgetConfig(type);
```

---

## useWidgetContext

**Назначение:** Единая точка доступа к состоянию виджета. Агрегирует `DashboardContext` или `FeatureCardContext` в зависимости от `type`.

**Параметры:** `type?: WidgetType` (default `Dashboard`)

**Возвращает:** объединённый объект контекста — `dataSources`, `filters`, `attributes`, `layerInfo`, `selectedTabId`, `expandedContainers`, `changeFilters`, `setSelectedTabId` и т.д.

```ts
const { dataSources, filters, attributes } = useWidgetContext(type);
```

---

## useWidgetFilters

**Назначение:** Логика применения фильтра к чарту — форматирование цвета, проверка активности, обработчик клика по элементу.

**Параметры:** `type: WidgetType`, `filterName: string`, `items?: FilterItem[]`

**Возвращает:** `{ filters, formatFilterColor, hasAnyFilter, isFiltered, onFilter }`
- `filters` — текущие выбранные фильтры
- `formatFilterColor` — `(name, color, defaultColor?) => string`
- `hasAnyFilter` — `boolean`, есть ли активное значение фильтра
- `isFiltered` — `(name: string) => boolean`
- `onFilter` — `(name: string) => void`

---

## useWidgetPage

**Назначение:** Доступ к текущей странице и навигации.

**Параметры:** `type?: WidgetType` (default `Dashboard`)

**Возвращает:**
| Поле | Описание |
|---|---|
| `pageIndex` | `number` |
| `currentPage` | `ConfigContainerChild` (merged dataSources + filters) |
| `changePage(index)` | Перейти на страницу |
| `updateConfigPage(data)` | Обновить конфиг страницы |
| `addConfigPage()` | Добавить страницу |
| `deleteConfigPage(index)` | Удалить страницу |
| `updateConfigLayer(name, data)` | Обновить слой страницы |
| `updateConfigLayers(layers)` | Обновить все слои |

```ts
const { pageIndex, currentPage } = useWidgetPage(type);
```

---

## Связанные разделы

[[components|Компоненты]] | [[utils|Утилиты]] | [[concepts|Основные понятия]] | [[setup|Подключение]] | [[types|Типы]]
