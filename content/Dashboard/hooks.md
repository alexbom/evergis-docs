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

## useAttachmentDownload

**Назначение:** Скачивание вложения по требованию — файл запрашивается в момент клика, а не заранее. Отдаётся кнопке «Скачать» просмотрщика `Preview` в [[containers#AttachmentContainer|`AttachmentContainer`]] и `EditAttachmentContainer`.

**Параметры:** `items: Attachment[]` — тот же список, что рисует контейнер (индекс приходит от `Preview`)

**Возвращает:** `(index: number) => void`

Свой файл лежит за авторизацией, поэтому ссылкой его не отдать: он тянется через `api.catalog.getFile` и сохраняется из памяти утилитой [[utils|`saveBlobAsFile`]]. Внешнее вложение (`isExternal`) просто открывается в новой вкладке — кросс-доменный атрибут `download` браузер всё равно игнорирует.

Повторный клик по тому же файлу во время загрузки игнорируется (`inFlightRef` по `item.link`). Ошибка запроса уходит в уведомление из [[setup|GlobalContext]] (`notification.add`, длительность `DOWNLOAD_ERROR_DURATION`), а не в тишину.

```ts
const downloadByIndex = useAttachmentDownload(items);

<Preview images={images} onDownload={(_image, index) => downloadByIndex(index)} ... />
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

## useBgImageHost

**Назначение:** Пропсы хоста фонового слоя `bgImage` для корней, которые **не** берут пропсы из **useWrapperSize** — контейнеры, ставящие `id`/`style` руками (`Title`, `Icon`, `Tabs`, `AddFeature`, `ExportPdf`, `Progress`, `RoundedBackground`, `OneColumn`, `TwoColumn`, `DefaultAttributes`, `PagesContainer`, подтипы `Edit*`). Контейнеры на **useContainerRoot** / **useWrapperSize** этот хук не зовут напрямую: `useWrapperSize` вызывает его сам и кладёт результат в `root` — так признаки хоста считаются в одном месте.

**Параметры:**

| Параметр | Тип |
|---|---|
| `elementConfig` | `BgImageHostConfig?` = `Pick<ConfigContainerChild, "children" \| "options">` — узел конфига контейнера |

**Возвращает:** `BgImageHostProps` — `{ $hasBgImage: boolean; $innerPadding: boolean }`

| Проп | Источник | Что включает в `bgImageHostMixin` |
|---|---|---|
| `$hasBgImage` | среди `children` есть узел с `id: "bgImage"` ([[utils\|`hasContainerBgImage`]]) | `position: relative` + `isolation: isolate` |
| `$innerPadding` | `options.innerPadding` | `padding: 1rem` (`CONTAINER_INNER_PADDING`) под селектором `&&` |

Гейты **раздельные**. Для `$hasBgImage` это принципиально: `position: relative` меняет containing block для абсолютно позиционированных потомков (контролы слайдшоу, подписи прогресса), поэтому включается только там, где автор конфига действительно попросил фон. `$innerPadding` фоном не обусловлен — отступ содержимого нужен и без картинки; двойной селектор `&&` поднимает специфичность, чтобы перебить `padding` из `$sizeCss` и внутренних `defaults` контейнера, оставив авторский inline-`style` сильнее.

Третья опция фона, `options.outflow`, сюда **не** входит: вылет за края — свойство самой картинки, его читает слой [[components|`ContainerBackground`]] пропом `$outflow`, а не хост. Механика целиком — в [[concepts#Универсальные слоты и фон контейнера|Основных понятиях]].

```ts
const bgImageHost = useBgImageHost(elementConfig);

return (
  <TitleWrapper id={id} style={style} {...bgImageHost}>
    <ContainerBackground elementConfig={elementConfig} renderElement={renderElement} />
    ...
  </TitleWrapper>
);
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

## useContainerRoot

**Назначение:** Пропсы двух узлов контейнера с заголовком — корня (`ContainerRoot`) и тела. Надстройка над **useWrapperSize**: тот же расчёт размеров, но результат разложен на две части. Используется почти всеми контейнерами, у которых есть `ExpandableTitle`.

**Параметры:**

| Параметр | Тип |
|---|---|
| `elementConfig` | `ConfigContainerChild?` — узел конфига контейнера |
| `defaults` | `CSSObject?` — внутренние дефолты корневой обёртки. Ссылка должна быть стабильной (константа или `useMemo`) |

**Возвращает:** `ContainerRootParts` — `{ root, body }`
- `root` — `WrapperRootProps` из **useWrapperSize**: `id`, `data-id`, `data-templatename`, авторский `style`, `$sizeCss`, `$noMargin` плюс пропсы хоста фона `$hasBgImage` / `$innerPadding` (см. **useBgImageHost**)
- `body` — `{ [CONTAINER_BODY_ATTRIBUTE], $sizeCss }`: маркер `data-container-body` для поиска тела по DOM плюс `flex: 1 1 auto; min-height: 0` (`CONTAINER_BODY_FILL_STYLE`), если у корня получилась определённая высота

> [!info] Почему один корневой узел
> Контейнер обязан рендерить ОДИН корень: фрагмент из заголовка и тела в строке (`options.column: false`) становится **двумя** ячейками родительского flex-row, и доля из `options.width` достаётся только телу. Поэтому `id`, `data-templatename`, `style` и размеры живут на корне, а телу отдаётся лишь остаток высоты под заголовком.
>
> Гейт fill-высоты берётся по **результирующей** высоте (`root.$sizeCss.height`, а при её отсутствии — `minHeight`), а не по `options.height`: высоту задаёт ещё и авторский `style`, и внутренние `defaults` контейнера. Значение `auto` высотой не считается — делить нечего.
>
> `minHeight` учитывается наравне с `height` ради режима [[containers#Рост под содержимое autoHeight|`autoHeight`]]: там корень несёт только минимум, и без этого тело осталось бы без `flex: 1 1 auto` — схлопнулось бы по содержимому, а минимум корня до треков внутри не дошёл бы вовсе.

```ts
const { root, body } = useContainerRoot({ elementConfig });

return (
  <ContainerRoot {...root}>
    <ContainerBackground elementConfig={elementConfig} renderElement={renderElement} />
    <ExpandableTitle elementConfig={elementConfig} type={type} renderElement={renderElement} />
    <Container {...body} isColumn>...</Container>
  </ContainerRoot>
);
```

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
| `type` | `WidgetType?` — виджет, чей контекст читается (по умолчанию Dashboard) |
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

Одинаковые запросы, оказавшиеся в полёте одновременно, схлопываются в один: сигнатура собирается из уже подставленных фильтрами параметров запроса, и второй вызов получает тот же промис вместо нового обращения к серверу. Это дедуп конкурентных дублей, а не кэш — как только промис завершился, запись снимается.

```ts
const { getDataSourcePromises, getUpdatingDataSources } = useDataSources({ type, config: currentPage, filters });
```

---

## useDataSourceLoading

**Назначение:** Признак «данных нет вообще» — единственное условие, при котором допустима полноэкранная заглушка `DashboardLoading`. Как только пришёл первый источник, страницу и модалку наполняют сами контейнеры, каждый со своим `ContainerLoading` / `ChartLoading`.

**Параметры:** `type: WidgetType`

**Возвращает:** `boolean` — `!!currentPage?.dataSources?.length && !dataSources?.length && !!isLoading`

**Где используется:** корневой `Dashboard` и `ElementModal`. Опираться на «сырой» `isLoading` из `useWidgetContext` для гейта целого поддерева нельзя: этот флаг взводится на любой рефетч (смена фильтра, правка источника в редакторе, autoSync-уведомление) и гасит уже отрисованный контент.

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

## useEqualTileWidth

**Назначение:** Уравнивает ширину плиток ряда [[containers#DataSourceContainer|`DataSourceContainer`]] по самой широкой и ограничивает её шириной ячейки grid. Результат контейнер отдаёт в CSS-переменную `--tile-width`, которую читает каждая плитка.

**Параметры:** объект
| Поле | Тип | Описание |
|---|---|---|
| `enabled` | `boolean` | Замер включён. `false` для «Растянуть», «В столбец» и конфигов без `columns` — тогда возвращается `undefined`, и ширину задаёт grid-ячейка |
| `itemsCount` | `number` | Число записей источника: смена состава запускает перезамер |
| `columns` | `number` | Плиток в ряду — делитель доступной ширины |
| `gap` | `number` | Отступ между плитками, px |

**Возвращает:** `[ref: RefObject<HTMLDivElement>, width: number | undefined]` — ref вешается на контейнер ряда.

Натуральная ширина снимается временным inline `width: max-content` (снимает уже применённое ограничение, поэтому замер устойчив к динамике данных и не зацикливает `ResizeObserver`). Итог ограничен `(clientWidth − gap × (columns − 1)) / columns`: плитка задаёт ширину в px и в grid не сжимается вместе с треком, поэтому без ограничения плитки наезжали друг на друга — см. [[containers#DataSourceContainer|примечание про grid-режим]]. Перезамер идёт по `ResizeObserver` на контейнере.

```ts
const [tilesRef, tileWidth] = useEqualTileWidth({
  enabled: !column && !!columns && !stretch,
  itemsCount: dataSource?.features?.length ?? 0,
  columns: columns ?? 1,
  gap: gap ?? DEFAULT_TILE_GAP,
});
```

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

**Назначение:** Готовый `src` для картинки. Свой файл тянется fetch'ем с токеном и отдаётся blob-адресом, чужой возвращается как есть.

**Параметры:** `url: string | null`

**Возвращает:** `string | null` — blob-адрес для своего файла, исходный URL для чужого, `null` при неудаче

Чужой адрес (проверка — [[utils|`isCrossOriginUrl`]]) намеренно не фетчится: тег `img` грузит кросс-доменную картинку без всякого CORS, а `fetch` — только если сторонний сервер отдал `Access-Control-Allow-Origin`. Заодно картинка остаётся в HTTP-кеше браузера, которого blob-адрес лишён. Так же поступает **useAttachmentPreviewImages**: внешней ссылке отдаёт `src: item.link`.

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

Токен (`STORAGE_TOKEN_KEY` из localStorage) уходит **только на свой origin** — гейт [[utils|`isCrossOriginUrl`]]. Чужому серверу он не нужен и вреден: кастомный заголовок переводит кросс-доменный запрос в preflight-режим, а ответ на `OPTIONS` без `Access-Control-Allow-Headers` роняет весь fetch. Параллельные вызовы для одного URL отсекаются флагом загрузки; `cleanup` вызывается при смене значения и на размонтировании (для blob-адресов — `URL.revokeObjectURL`).

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

## Хуки сетки (`grid/hooks`)

Внутренние хуки [[containers#Режим сетки grid|сетки контейнеров]]. Наружу из пакета не экспортируются — используются только компонентами `grid/`.

| Хук | Назначение |
|---|---|
| `useGridDraft({ node, type, onChange })` | Локальный черновик раскладки. Возвращает `draft`, `applyAction`, `commitResize`, `commitHeight`. Правки применяются сразу и уходят в `onChange`; конфиг, вернувшийся сверху, узнаётся по ссылке и не сбрасывает черновик |
| `useGridSelection()` | Выделение ячеек: `selectedIds`, `selectCell(id, additive)`, `clearSelection`. Shift добавляет и убирает, повторный клик по единственной выделенной ячейке снимает выделение, `Escape` снимает всё |
| `useGridResize({ axis, index, sizes, getGrid, onCommit })` | Перетаскивание границы пары треков. Во время жеста раскладка меняется инлайн-стилем без ре-рендера; `onCommit` вызывается один раз на отпускание мыши. Сам жест — общий **useResizeDrag**, поэтому возвращается его `ResizeDrag`: callback-ref `setHandle` и флаг `dragging` |
| `useGridHeightResize({ sizes, autoHeight, getGrid, onCommit })` | Перетаскивание **нижней** границы сетки, за которой соседнего трека уже нет: последняя строка растёт вместе с самой сеткой. В `onCommit` уходит и новая `options.height` корня (в пикселях — исходную единицу автора при таком жесте не восстановить), и пересчитанные доли всех строк, иначе верхние строки разъехались бы пропорционально новой высоте. С `autoHeight` результат означает минимум: корню пишется `min-height`, трекам — `minmax(auto, Npx)`, иначе под курсором сетка садилась бы на заданные пиксели и снова раздувалась содержимым после отпускания. Тот же `ResizeDrag` (`setHandle`, `dragging`) |
| `useGridCellSwap({ draft, applyAction })` | Перетаскивание ячейки на место другой. Один экземпляр на сессию: жест начинается в одной ячейке, а заканчивается в другой, возможно из другой строки или вложенной сетки. Источник и цель размечаются атрибутами прямо в DOM, без ре-рендера; правка уходит одна — на отпускание над валидной целью. Возвращает `beginDrag(cellId, event)` и `consumeDragClick()` |
| `useGridMenuOptions()` | Пункты контекстного меню (`IOption[]`) с переводами и вычисленными `disabled` |

Состояние сессии раздаёт контекст `GridEditContext` (`useGridEdit()`); его создаёт только внешний grid-узел с `options.editMode`. Значение контекста — `GridEditSessionValue` (см. [[types#Публичная поверхность сетки|Типы]]).

Пиксельные размеры треков читает `readTrackPixels(grid, axis)` — из computed `grid-template-*`, а не из прямоугольников ячеек: у отрисованного грида браузер отдаёт уже разрешённые used values в пикселях и без зазоров.

Рендер содержимого сессия строит сама — через **useRenderElement**, замкнутый на черновик: `renderElement` из пропсов для этого не годится, он замкнут на исходный узел и не найдёт ячейки, созданные `split`/`add`. Хост со своим реестром содержимого подменяет это фабрикой `createRenderElement` — см. [[containers#Интеграционный API для хостов|Интеграционный API для хостов]].

---

## useGlobalContext

**Назначение:** Доступ к `GlobalContext` (api, t, ewktGeometry, ewktExtent, zoomLevel, themeName, language).

**Параметры:** нет

**Возвращает:** `GlobalContextProps` (без `children`)

```ts
const { api, t, ewktGeometry, ewktExtent, zoomLevel } = useGlobalContext();
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

Дочернему элементу со slot-id `icon` тип и значение подставляются из настроек атрибута в слое (`attributesConfiguration.attributes[].icon`) через [[utils|`getAttributeIconElement`]]: `Icon` → [[elements#ElementIcon|ElementIcon]] с `iconName`, `PNG` → [[elements#ElementImage|ElementImage]], `SVG` → [[elements#ElementSvg|ElementSvg]] с `resourceId || url`. Собственный `attributeName` у такого элемента сбрасывается.

---

## useRenderElement

**Назначение:** Хук, возвращающий `renderElement` функцию для рендеринга дочерних элементов по `id`. Собирает контекст (config, layerInfo, attributes, табы, страница) из **useWidgetContext** / **useWidgetConfig** / **useWidgetPage** и передаёт в `getRenderElement`.

**Параметры:** `type?: WidgetType` (default `Dashboard`), `elementConfig: ConfigContainerChild`

**Возвращает:** `RenderElementFunction`

> [!info] Зачем нужен, если `renderElement` и так приходит пропом
> Проп замкнут на тот узел, который был при его создании. Если компонент рендерит **изменённое** дерево — как сессия редактирования [[containers#Режим сетки grid|сетки]] со своим черновиком, — узлы, созданные split/merge/add, в старом замыкании не найдутся. Тогда `renderElement` пересоздают от актуального узла этим хуком; вся вложенность ниже подхватывается сама, потому что `getRenderElement` рекурсивно строит новый `renderElement` от каждого найденного узла.

---

## useResizeBox

**Назначение:** Измеряет фактические ширину и высоту DOM-элемента (`ResizeObserver`, content-box) и обновляет их на любой ресайз. Нужен «резиновым» графикам: d3 требует пиксельные числа, а не CSS `100%`, а для вписывания (`fill`) — обе оси сразу. См. [[containers#Как работает fill|`fill` контейнера Chart]].

**Параметры:** `enabled: boolean` — `false` полностью отключает наблюдение (обычные, не-fill графики не измеряются и не вызывают лишних ре-рендеров)

**Возвращает:** `[ref: RefObject<HTMLDivElement>, box: ResizeBox]`, где `ResizeBox` — `{ width?: number, height?: number }`; пустой объект, пока элемент не измерен (ссылка на пустой бокс стабильна)

```ts
const [chartRef, { width, height }] = useResizeBox(fill);

return <ChartFillMeasure ref={chartRef}>{chartBody}</ChartFillMeasure>;
```

---

## useResizeDrag

**Назначение:** Общий жест перетаскивания границы: подписка на ручку, снимок на нажатии, предпросмотр прямо в DOM на движении, запись результата на отпускании. Один и тот же для границ ячеек [[containers#Режим сетки grid|сетки]] и для ширины колонок таблицы [[elements|ElementTable]] — различаются они только тем, что меряют и куда пишут. Клик, которым закончился жест, хук съедает: мышь отпускают уже за пределами ручки, и без этого перетаскивание границы заканчивалось бы сортировкой колонки или сменой выделения в сетке.

**Параметры:** объект
| Параметр | Тип | Описание |
|---|---|---|
| `onStart` | `(event: MouseEvent) => TState \| null` | Снимок на нажатии: всё, от чего считается жест. `null` — жест не начинается (тянуть нечего). Снимок берётся один раз именно здесь: пересчёт на каждом шаге уже по применённым размерам копит округление и уводит границу от курсора |
| `onMove` | `(state: TState, event: MouseEvent) => void` | Шаг жеста. Рисует предпросмотр прямо в DOM — состояние на каждое движение не трогается, ре-рендера нет |
| `onEnd` | `(state: TState, event: MouseEvent) => void` | Конец жеста: снять предпросмотр и записать результат один раз |

**Возвращает:** `ResizeDrag`
| Поле | Описание |
|---|---|
| `setHandle` | Callback-ref для ручки: `useDragAndDropEffect` из `@evergis/uilib-gl` принимает элемент, а не ref |
| `dragging` | Жест идёт. Ручка отдаёт флаг в `data-dragging`, по которому подсвечивается ползунок |

Мышь ведут по документу, а не по самой ручке, поэтому отпускание за пределами окна тоже завершает жест. Ручка — styled-компонент [[components|ResizeHandle]]; на хуке построены **useGridResize**, **useGridHeightResize** (см. «Хуки сетки») и `useColumnResize` таблицы.

```ts
const { setHandle, dragging } = useResizeDrag<DragState>({ onStart, onMove, onEnd });

return <ResizeHandle ref={setHandle} $axis="column" data-dragging={dragging} />;
```

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

## useWrapperSize

**Назначение:** Собирает пропсы корневой обёртки контейнера: идентификаторы для внешних селекторов, авторский `style` из конфига и css-объект размеров из `options.width` / `options.height` / `options.overflow` (см. [[containers#Размерная модель обёртки ContainerBoxOptions|размерную модель]]). Вызывается почти всеми контейнерами вместо ручной сборки стилей.

**Параметры:**

| Параметр | Тип |
|---|---|
| `elementConfig` | `ConfigContainerChild?` — узел конфига контейнера |
| `defaults` | `CSSObject?` — внутренние дефолты обёртки (отступы, собственная высота). Ссылка должна быть стабильной — константа или `useMemo` |

**Возвращает:** `WrapperRootProps` — `{ id, "data-id", "data-templatename", style, $sizeCss, $noMargin, $hasBgImage, $innerPadding }`

Размеры уходят styled-пропом `$sizeCss` (класс), а не inline-стилем, поэтому перебиваются снаружи обычной специфичностью (`#id`, `[data-templatename]`) — без `!important`. Авторский `style` остаётся inline: у него приоритет по замыслу автора конфига. Ширина по умолчанию — `FILL_SIZE` (`"100%"`), поэтому контейнер занимает всю ширину ячейки, пока `options.width` не задан явно. Вычисление делегируется [[utils|утилите]] `getWrapperSizeStyle`.

`data-id` и `$noMargin` раньше жили на внешней обёртке `ElementValueWrapper`; она вешала их на второй узел поверх этого корня, дублируя стили, и для контейнеров из `ROOT_OWNING_TEMPLATES` больше не создаётся.

Пропсы хоста фонового слоя (`$hasBgImage`, `$innerPadding`) подмешиваются здесь вызовом **useBgImageHost**, а не в каждом контейнере: корень и так получает пропсы одним спредом, поэтому фон и внутренний отступ включаются без единой дополнительной строки на месте вызова. Корни, собираемые вручную, зовут **useBgImageHost** сами.

`heightAsMin` приходит из `options.autoHeight` и переводит высоту в `min-height` — узел не опускается ниже неё, но перерастает под содержимое.

```ts
const root = useWrapperSize({ elementConfig });

return <ChartContainerWrapper {...root}>...</ChartContainerWrapper>;
```

---

## Связанные разделы

[[components|Компоненты]] | [[utils|Утилиты]] | [[concepts|Основные понятия]] | [[setup|Подключение]] | [[types|Типы]]
