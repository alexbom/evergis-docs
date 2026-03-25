# Хуки

## Обзор

Хуки расположены в `D:/projects/spcore_api/packages/react/src/components/Dashboard/hooks/`. Предоставляют бизнес-логику для компонентов Dashboard и FeatureCard.

---

## useAutoCompleteControl

**Назначение:** Автозаполнение для edit-контрола. Загружает варианты значений из источника данных.

**Параметры:**
| Параметр | Тип |
|---|---|
| `type` | `WidgetType` |
| `elementConfig` | `ConfigContainerChild` |

**Возвращает:** `{ items: IOption[], value, onChange }`

```ts
const { items, value, onChange } = useAutoCompleteControl(type, elementConfig);
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

**Возвращает:** `{ data: ChartDataItem[], loading: boolean }`

`ChartDataItem`: `{ items, layerInfo, attributeName, attributeUnits, dataSourceName, color }`

```ts
const { data, loading } = useChartData({ element: chartElement, type });
```

---

## useDashboardHeader

**Назначение:** Данные для шапки дашборда (заголовок, иконка, изображение, тема).

**Параметры:** нет

**Возвращает:**
| Поле | Тип |
|---|---|
| `title` | `string` |
| `pageId` | `string` |
| `image` | `string` |
| `icon` | `ReactNode` |
| `tooltip` | `string` |
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

**Назначение:** Логика для edit-контейнеров. Получает значение атрибута и обработчик его изменения.

**Параметры:**
| Параметр | Тип |
|---|---|
| `type` | `WidgetType` |
| `elementConfig` | `ConfigContainerChild` |

**Возвращает:** `{ value, onChange, attribute, isDisabled }`

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

**Назначение:** Получение `ConfigLayer` из конфига текущей страницы по имени слоя.

**Параметры:** `type: WidgetType`, `layerName: string`

**Возвращает:** `ConfigLayer | undefined`

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

**Назначение:** Создаёт функцию `renderElement` для шапок FeatureCard. Аналог getRenderElement для `ConfigContainerHeader`.

**Параметры:** `header: ConfigContainerHeader`

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
| `type` | `WidgetType` |
| `elementConfig` | `ConfigContainerChild` |
| `dataSources` | `WidgetDataSource[]` |
| `feature` | `FeatureDc?` |

**Возвращает:** `{ attributes: ClientFeatureAttribute[], layerInfo: QueryLayerServiceInfoDc, dataSource: WidgetDataSource }`

---

## useRenderContainerItem

**Назначение:** Фабрика для рендеринга одного элемента контейнера (OneColumn / TwoColumn). Возвращает `getRenderContainerItem`.

**Параметры:** `type: WidgetType`, `renderElement: RenderElementFunction`

**Возвращает:** функция `(elementConfig, attributeName?) => { id, value, hideEmpty, style, hasIcon, hasUnits, render }`

---

## useRenderElement

**Назначение:** Хук, возвращающий `renderElement` функцию для рендеринга дочерних элементов по `id`.

**Параметры:** `GetRenderElementProps`

**Возвращает:** `RenderElementFunction`

---

## useShownOtherItems

**Назначение:** Управление пагинацией списков (shownItems / otherItems).

**Параметры:** `options: ConfigOptions`

**Возвращает:** `{ sliceItems(data), checkIsSliced(data), showMore, onShowMore }`

---

## useUpdateDataSource

**Назначение:** Обновление данных одного источника данных (например, после изменения записи).

**Параметры:** `type: WidgetType`

**Возвращает:** `updateDataSource: (dataSourceName: string) => Promise<void>`

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

**Назначение:** Логика применения фильтра к чарту — форматирование цвета, обработчик клика по элементу.

**Параметры:** `type: WidgetType`, `filterName: string`, `items: FilterItem[]`

**Возвращает:** `{ formatFilterColor: (name, color, default?) => string, onFilter: (name: string) => void }`

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
| `addConfigPage(props?)` | Добавить страницу |
| `deleteConfigPage(index)` | Удалить страницу |
| `updateConfigLayer(name, data)` | Обновить слой страницы |
| `updateConfigLayers(layers)` | Обновить все слои |

```ts
const { pageIndex, currentPage } = useWidgetPage(type);
```

---

## Связанные разделы

[[components|Компоненты]] | [[utils|Утилиты]] | [[concepts|Основные понятия]] | [[setup|Подключение]]
