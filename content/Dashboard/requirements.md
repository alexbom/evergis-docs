# Системные требования

## @evergis/react

Хуки:
`useWidgetPage`, `useWidgetContext`, `useWidgetConfig`, `useWidgetFilters`, `useGlobalContext`, `useDataSources`, `useChartData`, `useChartChange`, `useDashboardHeader`, `useEditControl`, `useUpdateDataSource`, `useExportPdf`, `useFetchWithAuth`, `useFetchImageWithAuth`, `useGetConfigLayer`, `useHeaderRender`, `useHideIfEmptyDataSource`, `useRelatedDataSourceAttributes`, `useRenderContainerItem`, `useRenderElement`, `useShownOtherItems`, `useExpandableContainers`, `useAutoCompleteControl`, `useDiffPage`, `useProjectDashboardInit`, `useServerNotificationsContext`, `useAttachmentItems`, `useAttachmentPreviewImages`, `useContainerAttributes`, `useEditGroupAttributes`, `useFeatureSaveHooks`, `useBeforeSave`, `useAfterSave`, `useSavePrototypeBuilder`, `useResizeBox`, `useWrapperSize`

Компоненты и провайдеры:
`DashboardProvider` (BaseDashboardProvider), `FeatureCardProvider`, `GlobalProvider`, `ConfigContainer`, `ContainerTemplate`, `HeaderTemplate`, `WidgetType`

Типы и branded keyspaces (см. [[types|Типы]]):
`DashboardChild`, `DashboardHeaderConfig`, `ContainerComponentRegistry`, `ElementComponentRegistry`, `Brand`, `ContainerId`, `ChartId`, `ModalId`, `TabId`, `FilterName`, `LayerName`, `AttributeName`, `DataSourceName`, `ResourceId`. Конструкторы: `asContainerId`, `asChartId`, `asModalId`, `asTabId`, `asFilterName`, `asLayerName`, `asAttributeName`, `asDataSourceName`, `asResourceId`.

Утилиты:
`formatDataSourceCondition`, `applyTreeFilterToCondition`, `applyQueryFilters`, `getContainerComponent`, `getRenderElement`, `getFilterComponent`, `getDashboardHeader`, `getFeatureCardHeader`, `getDataSourceLayerInfo`, `mergeAttributeConfigurations`, `getWrapperSizeStyle`, `toCssSize`, `isVisibleContainer`, `checkEqualOrIncludes`

## @evergis/api

`AttributeDefinitionDc`, `AttributeFormatDefinitionDc`, `AttributeType`, `CatalogResourceDc`, `EqlRequestDc`, `ExtendedProjectInfoDc`, `FeatureDc`, `LayerDefinitionDc`, `OgcGeometryType`, `PagedFeaturesListDc`, `PositionDc`, `ProjectContentItemDc`, `QueryLayerServiceConfigurationDc`, `QueryLayerServiceInfoDc`, `RemoteTaskStatus`, `StringSubType`

## @evergis/uilib-gl

`Dropdown`, `Flex`, `FlexSpan`, `Icon`, `IconButton`, `IconToggle`, `IconTypesKeys`, `IOption`, `ITheme`, `LegendToggler`, `LinearProgress`, `Preview`, `Tooltip`, `Uploader`, `UploaderItemProps`, `Dialog`, `DialogContent`, `DialogTitle`

## @evergis/charts

`BarChartMarker`, `LineChart`, `PieChart`, `PieChartData`

## Redux

Слайс `dashboard`:
- `getProjectDataSources`, `setProjectDataSources`
- `getProjectDataSourcesAreLoading`, `setProjectDataSourcesAreLoading`
- `getProjectPageIndex`
- `getReferenceLayerInfos`

Слайс `project`:
- `getProjectGeometryFilter`

## Прочие зависимости

| Пакет | Использование |
|---|---|
| `lodash` | `isEmpty`, `isEqual`, `isNil`, `reduce` |
| `i18next` | `i18n["t"]` — переводы, namespace `dashboard` |
| `styled-components` | Стилизация всех компонентов (`useTheme`, `ThemeProvider`) |
| `react-markdown` | `ElementMarkdown` — рендеринг Markdown |
| `rehype-raw`, `rehype-sanitize`, `remark-gfm` | Плагины для `react-markdown` |
| `swiper` | `TabsContainer` — горизонтальный скролл вкладок |
| `javascript-color-gradient` | Генерация градиентов цветов для чартов |
| `find-and` | `ElementLegend` — `returnFound` для поиска в дереве конфига |
| `jsPDF`, `html2canvas` | `useExportPdf` — экспорт в PDF |
| `d3` | `FEATURE_CARD_DEFAULT_COLORS` (`d3.schemeAccent`) для цветовой палитры карточки |
| `maplibre-gl` | Типы `CircleLayerSpecification`, `FillLayerSpecification`, `LineLayerSpecification` для `CustomFeatureSelect` |
| `@xterm/xterm`, `@xterm/addon-fit` | `LogTerminal` — вывод лога Python-задачи в `TaskContainer` |
| `geojson` | Тип `Geometry` для `SaveHookInput.changedGeometry` (серверные хуки сохранения) |

Браузерные API: `ResizeObserver` — [[hooks\|`useResizeBox`]] (измерение ячейки для `fill`-режима графика).

## Связанные разделы

[[setup|Подключение]] | [[architecture|Архитектура]] | [[types|Типы]]
