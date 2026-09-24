# Системные требования

## @evergis/react

Хуки:
`useWidgetPage`, `useWidgetContext`, `useWidgetConfig`, `useWidgetFilters`, `useGlobalContext`, `useDataSources`, `useDataSourceLoading`, `useChartData`, `useChartChange`, `useChartAxisAttributes`, `useChartAxisTickFormat`, `useChartAxisTitles`, `useDashboardHeader`, `useEditControl`, `useUpdateDataSource`, `useExportPdf`, `useFetchWithAuth`, `useFetchImageWithAuth`, `useGetConfigLayer`, `useHeaderRender`, `useHideIfEmptyDataSource`, `useRelatedDataSourceAttributes`, `useRenderContainer`, `useRenderContainerItem`, `useRenderElement`, `useShownOtherItems`, `useExpandableContainers`, `useAutoCompleteControl`, `useDiffPage`, `useProjectDashboardInit`, `useServerNotificationsContext`, `useAttachmentItems`, `useAttachmentPreviewImages`, `useContainerAttributes`, `useContainerRoot`, `useEditGroupAttributes`, `useEqualTileWidth`, `useFeatureSaveHooks`, `useBeforeSave`, `useAfterSave`, `useSavePrototypeBuilder`, `useResizeBox`, `useWrapperSize`, `useBgImageHost`, `useResizeDrag`, `useConfigDataSources`, `useAttachmentDownload`, `useAttachmentsView`

Компоненты и провайдеры:
`Dashboard`, `DashboardProvider` (BaseDashboardProvider), `FeatureCardProvider`, `GlobalProvider`, `ContainerBackground`, `ContainerWrapper`, `ConfigContainer`, `ContainerTemplate`, `HeaderTemplate`, `WidgetType`

Константы и дефолты:
`DEFAULT_DASHBOARD_CONFIG`, `DEFAULT_PAGES_CONFIG`, `CONFIG_PAGES_ID`, `CONFIG_PAGE_ID`, `TITLE_SLOT_IDS`, `BG_IMAGE_SLOT_ID`, `NON_TRACK_SLOT_IDS`, `ROOT_OWNING_TEMPLATES`, `BASE_CONTAINER_STYLE`, `CONTAINER_BODY_ATTRIBUTE`, `CONTAINERS_GROUP_DEFAULTS`

Типы и branded keyspaces (см. [[types|Типы]]):
`DashboardChild`, `StrictDashboardChild`, `StrictConfigContainerChild`, `DashboardHeaderConfig`, `ContainerComponentRegistry`, `ElementComponentRegistry`, `Brand`, `ContainerId`, `ChartId`, `ModalId`, `TabId`, `FilterName`, `LayerName`, `AttributeName`, `DataSourceName`, `ResourceId`, `ConfigNotifications` (фильтр уведомлений по автору). Конструкторы: `asContainerId`, `asChartId`, `asModalId`, `asTabId`, `asFilterName`, `asLayerName`, `asAttributeName`, `asDataSourceName`, `asResourceId`.

Публичная поверхность сетки (`grid/index.ts`, см. [[types#Публичная поверхность сетки|Типы]]):
типы `GridAxis`, `GridInsertSide`, `GridSplitDirection`, `GridEditAction`, `GridMenuState`, `GridMenuPosition`, `GridEditSessionValue`, `GridCellContext`, `GridIdFactory`; константы `MIN_TRACK_PX`, `MAX_TRACKS`, `DEFAULT_GRID_GAP`, `GRID_CELL_ATTR`, `GRID_ROW_ID_PREFIX`, `GRID_CELL_ID_PREFIX`, ...; утилиты `getLayoutChildren`, `getTrackSizes`, `buildGridTemplate`, `isGridNode`, `findCellContext`, `createGridCell`, `createGridRow`, `collectConfigIds`, `createGridIdFactory`.

Утилиты:
`formatDataSourceCondition`, `applyTreeFilterToCondition`, `applyQueryFilters`, `fetchQueryDescription`, `getContainerComponent`, `isRootOwningContainer`, `getRenderElement`, `getFilterComponent`, `getDashboardHeader`, `getFeatureCardHeader`, `getDataSourceLayerInfo`, `mergeAttributeConfigurations`, `getWrapperSizeStyle`, `hasContainerBgImage`, `isTreeFilterValue`, `isFeaturesFilterValue`, `isFillSize`, `isFrSize`, `toCssSize`, `toRenderableValue`, `isVisibleContainer`, `checkEqualOrIncludes`, `createConfigPage`, `createConfigLayer`

## @evergis/api

`Api`, `ArchiveTimelineItemDc`, `AttributeConfigurationDc`, `AttributeConfigurationType`, `AttributeIconDc`, `AttributeIconType`, `AttributeType`, `AttributesConfigurationDc`, `CatalogResourceDc`, `EqlRequestDc`, `ExtendedProjectInfoDc`, `FeatureDc`, `OgcGeometryType`, `PagedFeaturesListDc`, `PositionDc`, `ProxyServiceInfoDc`, `QueryLayerServiceConfigurationDc`, `QueryLayerServiceInfoDc`, `RemoteTaskStatus`, `STORAGE_TOKEN_KEY`, `StringAttributeConfigurationDc`, `StringSubType`, `TaskPrototypeDto`

## @evergis/uilib-gl

Контролы и ввод: `AutoComplete`, `Checkbox`, `ColorPicker`, `DatePicker`, `Dropdown`, `DropdownField`, `Input`, `NumberInput`, `NumberRangeSlider`, `RangeNumberInput`, `Slider`, `Switch`, `TreeDropdown`, `TreeId`, `TreeItemProps`, `MultiSelectContainer`, `ComplexOptionText`, `PartialLoadData`, `useAsyncAutocomplete`.

Кнопки и действия: `ActionsGroup`, `FlatButton`, `IconButton`, `IconButtonButton`, `IconButtonInnerChild`, `IconToggle`, `IconToggleButton`, `RaisedButton`, `Menu`, `Popover`, `Popup`.

Раскладка и типографика: `Blank`, `Chip`, `Description`, `Divider`, `Flex`, `FlexSpan`, `H2`, `Icon`, `IconTypesKeys`, `LegendToggler`, `Tooltip`, `Dialog`, `DialogActions`, `DialogContent`, `DialogTitle`.

Загрузка и файлы: `CircularProgress`, `LinearProgress`, `Preview`, `IPreviewImage`, `Uploader`, `UploaderItemArea`, `UploaderItemProps`, `UploaderTitleWrapper`.

Тема и утилиты: `ITheme`, `ThemeProvider`, `defaultTheme`, `darkTheme`, `shadows`, `transition`, `getLocale`, `dateFormat`, `IOption`, `IJSXOption`, `INotificationItem`, `DraggableTreeContainer`, `useDragAndDropEffect`.

## @evergis/charts

`BarChart`, `BarChartData`, `BarChartMarker`, `BarChartMarshalledGroup`, `BarChartMergedData`, `LineChart`, `LineChartProps`, `PieChart`, `PieChartData`, `barChartClassNames`, `lineChartClassNames`

## @evergis/color

`Color` — разбор строки цвета (`hex`, `rgb`, `rgba`) и проверка её валидности. Нужен колонке цвета таблицы [[containers|StructuredDataContainer]] (`colorPicker` у атрибута): значение остаётся обычной строкой, и что она значит цвет, говорит только схема. Оттуда же собран `colorToHex` (`utils/color`), которым выбранный палитрой цвет уходит в черновик — `#rrggbb`, а при неполной непрозрачности `#rrggbbaa`.

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
| `find-and` | `ElementLegend` — `returnFound` для поиска в дереве конфига |
| `jsPDF`, `html2canvas` | `useExportPdf` — экспорт в PDF |
| `d3` | `FEATURE_CARD_DEFAULT_COLORS` (`d3.schemeAccent`) для цветовой палитры карточки |
| `maplibre-gl` | Типы `CircleLayerSpecification`, `FillLayerSpecification`, `LineLayerSpecification` для `CustomFeatureSelect` |
| `@xterm/xterm`, `@xterm/addon-fit` | `LogTerminal` — вывод лога Python-задачи в `TaskContainer` |
| `geojson` | Типы `Geometry`, `FeatureCollection` — `SaveHookInput.changedGeometry` и значение фильтра `valueType: "features"` |
| `date-fns` | Форматирование дат: таймлайн `ElementCamera` (`useCameraAttribute`), имя PDF-файла в `useExportPdf` |
| `uuid` | Клиентские id строк таблицы `StructuredDataContainer` и id уведомления об ошибке скачивания вложения ([[hooks\|`useAttachmentDownload`]]) |

Браузерные API: `ResizeObserver` — [[hooks\|`useResizeBox`]] (измерение ячейки для `fill`-режима графика).

## Связанные разделы

[[setup|Подключение]] | [[architecture|Архитектура]] | [[types|Типы]]
