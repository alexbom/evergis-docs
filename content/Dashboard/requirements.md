# Системные требования

## @evergis/react

Хуки:
`useWidgetPage`, `useWidgetContext`, `useWidgetConfig`, `useWidgetFilters`, `useGlobalContext`, `useDataSources`, `useChartData`, `useChartChange`, `useDashboardHeader`, `useEditControl`, `useUpdateDataSource`, `useExportPdf`, `useFetchWithAuth`, `useFetchImageWithAuth`, `useGetConfigLayer`, `useHeaderRender`, `useHideIfEmptyDataSource`, `useRelatedDataSourceAttributes`, `useRenderContainerItem`, `useRenderElement`, `useShownOtherItems`, `useExpandableContainers`, `useAutoCompleteControl`, `useDiffPage`, `useProjectDashboardInit`, `useServerNotificationsContext`

Компоненты и провайдеры:
`DashboardProvider` (BaseDashboardProvider), `FeatureCardProvider`, `GlobalProvider`, `ConfigContainer`, `ContainerTemplate`, `WidgetType`

Утилиты:
`formatDataSourceCondition`, `applyQueryFilters`, `getContainerComponent`, `getRenderElement`, `getFilterComponent`, `getDashboardHeader`, `getFeatureCardHeader`, `isVisibleContainer`, `checkEqualOrIncludes`

## @evergis/api

`AttributeDefinitionDc`, `AttributeFormatDefinitionDc`, `AttributeType`, `EqlRequestDc`, `ExtendedProjectInfoDc`, `FeatureDc`, `LayerDefinitionDc`, `OgcGeometryType`, `PagedFeaturesListDc`, `PositionDc`, `ProjectContentItemDc`, `QueryLayerServiceConfigurationDc`, `QueryLayerServiceInfoDc`, `RemoteTaskStatus`, `StringSubType`

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

## Связанные разделы

[[setup|Подключение]] | [[architecture|Архитектура]]
