# Шапки

> [!danger] Дочерние элементы шапки обязаны иметь slot-`id`
> Каждый ребёнок шапки идентифицируется по slot-`id` из фиксированного набора (`title`, `description`, `featureIcon`, `bgImage`, `icon`, `slideshow`). Без него элемент шапки не отрисуется. Таблица slot-id и чек-лист — [[authoring|Правила генерации]].

## Обзор

Шапки рендерятся через [[utils|утилиты]] `getDashboardHeader` / `getFeatureCardHeader` по `templateName` из конфига. Точка входа — [[components|компонент]] `DashboardHeader` (для Dashboard) и `FeatureCardHeader` (для FeatureCard).

Дочерние элементы шапки **обязательно** идентифицируются по slot-`id` из фиксированного набора (`title`, `description`, `featureIcon`, `bgImage`, `icon`, `slideshow`) — конкретный набор зависит от типа шапки. Подробности и таблица slot-id — в [[concepts#ID контейнеров и элементов|разделе про id]]. Слот `bgImage` у шапок — тот же универсальный слот фона, что и у контейнеров: рендерится общим [[components|`ContainerBackground`]].

`HeaderTemplate` enum (см. [[types|Типы]]):

| Значение | Шапка | Виджет |
|---|---|---|
| `HeaderTemplate.Default` | **DashboardDefaultHeader** | `WidgetType.Dashboard` |
| `HeaderTemplate.Default` | **FeatureCardDefaultHeader** | `WidgetType.FeatureCard` |
| `HeaderTemplate.Background` | **FeatureCardBackgroundHeader** | `WidgetType.FeatureCard` |
| `HeaderTemplate.Slideshow` | **FeatureCardSlideshowHeader** | `WidgetType.FeatureCard` |

Union типов конфигов шапок: `DashboardHeaderConfig` (см. [[types#Union шапок: DashboardHeaderConfig|типы]]). `HeaderTemplate.Default` сразу матчится двумя ветвями (Dashboard vs FeatureCard) — конкретная выбирается на уровне виджета.

Shared-компоненты в `headers/components/`:
- **HeaderContent** — общая раскладка всех `FeatureCard*Header`: иконка, текст и кнопка закрытия — строкой или колонкой (`column`). См. [[#Иконка объекта, раскладка и строки текста]].
- **HeaderLayerIcon** — кликабельная иконка с тултипом «приблизить к объекту»: иконка объекта из слота `featureIcon` (хук `useHeaderFeatureIcon`), а без неё — значок слоя по типу геометрии; используется во всех `FeatureCard*Header`.
- **HeaderTitle** — заголовок и описание `FeatureCardDefaultHeader`: авторские слоты `title`/`description`, а при отсутствии соответствующего слота — значение `titleAttribute` или `feature.id` и описание слоя. Число строк — `multilineTitle` / `multilineDescription`.

> [!warning] Текстовый слот шапки задаётся с `type`
> `{ id: "title", type: "attributeValue", attributeName: "name" }` выводит значение атрибута, `{ id: "title", type: "text", value: "..." }` — текст. Узел **без** `type` (только `attributeName`) ничего не рендерит: `Background` и `Slideshow` молча показывают штатный текст (`titleAttribute` / `feature.id` для заголовка, имя слоя для подзаголовка), а у `Default` строка остаётся пустой — слот задан, и штатный текст не подставляется.

---

## DashboardDefaultHeader

**Назначение:** Шапка дашборда — логотип, кнопки панели, заголовок текущей страницы, пагинация, меню страниц.

**Типы:** `templateName = HeaderTemplate.Default` · `DashboardDefaultHeaderOptions` · `DashboardDefaultHeaderConfig`. См. [[types#Шапки|сводную таблицу]].

**Props:** нет (данные из контекста)

**Зависимости:**
- [[hooks|хук]] `useDashboardHeader` — `title`, `pageId`, `image`, `icon`, `tooltip`, `themeName`, `isPaginationVisible`, `onClickLogo`
- [[hooks|хук]] `useWidgetContext` — `toggleLayersVisibility`, `components.ProjectPanelMenu`, `components.ProjectPagesMenu`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `url` | `string` | URL логотипа (через `image` из `useDashboardHeader`) |

Остальные данные (image, icon, tooltip, themeName) приходят из `useDashboardHeader`, который читает `projectInfo.content.dashboardConfiguration.header.options` напрямую и нормализует.

**Дочерние элементы (по `id`):**

| id | Описание |
|---|---|
| `title` | Заголовок страницы. Нет такого ребёнка — берётся `currentPage.options.title` |
| `icon` | Логотип в левом верхнем углу. Нет — рисуется штатная иконка `logo` (`2rem`) |
| `image` | Фоновое изображение шапки: значение прогоняется через `getResourceUrl` и грузится с авторизацией ([[hooks\|`useFetchImageWithAuth`]]). Нет — дефолтный градиент |

Слот `description` эта шапка **не рендерит** — он есть только у `FeatureCard*`-шапок.

**Фон шапки:** заданное `image` кадрируется по `cover` (пропорции фотографии сохраняются), а дефолтный градиент `header-gradient.png` растягивается на всю шапку (`background-size: 100% 100%`). Картинка нарисована ровно под ширину панели (360×134), и без растяжения на шапке пошире она легла бы пятном по центру.

**Структура:**
```
DefaultHeaderContainer (image, isDark)
  ├── LinearProgress (если !pageId — страница загружается)
  ├── TopContainer: [LogoContainer(icon), ProjectPanelMenu]
  └── PageTitleContainer: [Tooltip(PageTitle onClick=toggleLayersVisibility), ProjectPagesMenu, Pagination]
```

**Пагинация:** `capabilities.pagination: false` в корне конфига (`isPaginationVisible` из `useDashboardHeader`) скрывает `ProjectPagesMenu` и `Pagination` — см. [[concepts#Отключение функций проекта — `capabilities`|capabilities]].

```tsx
// HeaderTemplate.Default в контексте WidgetType.Dashboard
{ header: { templateName: "Default", options: { url: "/logo.svg" } } }
```

---

## FeatureCardDefaultHeader

**Назначение:** Шапка карточки объекта по умолчанию — иконка слоя, название объекта, кнопки действий (редактирование, закрытие).

**Типы:** `templateName = HeaderTemplate.Default` · `FeatureCardDefaultHeaderOptions` · `FeatureCardDefaultHeaderConfig`.

**Props:** `noFeature?: boolean`

**Зависимости:**
- [[hooks|хук]] `useConditionalHeader(WidgetType.FeatureCard)` — шапка текущей страницы с разрешёнными условиями
- [[hooks|хук]] `useHeaderRender(header, WidgetType.FeatureCard)` — рендер авторских слотов
- [[utils|утилита]] `getThemeByName(themeName)` — применение темы к шапке

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `themeName` | `"light" \| "dark"` | Принудительная тема для шапки. Не задана — берётся глобальная тема из [[setup\|GlobalContext]] (`themeName ?? pageThemeName`) |
| `withPadding` | `boolean` | Внутренние отступы вокруг заголовка |
| `column` | `boolean` | Иконка и кнопка закрытия в верхней строке, текст под ними на всю ширину (иначе текст справа от иконки). См. [[#Иконка объекта, раскладка и строки текста]] |
| `height` | `number` | Высота шапки в px |
| `overlay` | `string` | CSS-наложение поверх содержимого |
| `bigFeatureIcon` | `boolean` | Иконка объекта (слот `featureIcon`) 72px вместо 42px |
| `multilineTitle` | `boolean` | Заголовок до двух строк (без опции — одна), дальше многоточие |
| `multilineDescription` | `boolean` | Подзаголовок до двух строк (без опции — одна), дальше многоточие |

**Дочерние элементы:**

| id | Описание |
|---|---|
| `title` | Авторский заголовок, рендерится через `useHeaderRender` с `wrap: false`. При отсутствии слота сохраняется штатный `titleAttribute` с форматированием или `feature.id` |
| `description` | Авторское описание. При отсутствии слота сохраняется `layerInfo.configuration.alias`, затем `name` |
| `featureIcon` | Иконка объекта на месте значка слоя. Пустая — остаётся значок слоя. См. [[#Иконка объекта, раскладка и строки текста]] |

Заданность слота определяется его наличием, а не непустым результатом: скрытый или пустой авторский `title`/`description` не заменяется штатным текстом. У скалярного содержимого `style` дочернего узла применяется к существующей обёртке слота; у React-элемента стиль уже находится на его собственном корне. `visibility: false` скрывает обёртку слота в обоих случаях; правила `conditions` разрешаются тем же механизмом [[conditional-formatting|условного форматирования]].

При `noFeature` авторские слоты не используются: заголовок остаётся «Объектов не найдено», описание — «Все слои». Для выбранного объекта клик по заголовку по-прежнему приближает карту к нему; tooltip приближения скрывается вместе с заголовком или его предком. Иконка слоя и кнопки карточки сохраняют штатное поведение.

`header.style` и условная `header.visibility` применяются к `DefaultHeaderWrapper`. Скрытая шапка остаётся смонтированной.

**Структура:**
```
ThemeProvider(getThemeByName(themeName ?? pageThemeName))
  └── DefaultHeaderWrapper(withPadding, height, $noFeature)
      └── Header($overlay, $isRow=!column, $noFeature)
          └── HeaderContent(column, isDefault, noFeature)
              ├── строкой: HeaderFrontView(isDefault)
              │   ├── HeaderContainer: [HeaderLayerIcon, HeaderTitle(noFeature)]
              │   └── FeatureCardButtons (Close)
              └── column: HeaderColumnView
                  ├── HeaderIconRow: [HeaderLayerIcon, FeatureCardButtons]
                  └── HeaderTitle(noFeature)
```

```tsx
{
  header: {
    templateName: "Default",
    options: { themeName: "light", withPadding: true },
    children: [
      { id: "title", type: "text", value: "{name}" },
      { id: "description", type: "text", value: "{category}" }
    ]
  }
}
```

---

## FeatureCardBackgroundHeader

**Назначение:** Шапка карточки с цветной/градиентной заливкой фона и опциональным фоновым изображением. Подходит для эмоционального выделения карточки и визуального обозначения типа объекта (через `bigIcon` или `bgImage`).

**Типы:** `templateName = HeaderTemplate.Background` · `FeatureCardBackgroundHeaderOptions` · `FeatureCardBackgroundHeaderConfig`.

**Props:** нет

**Зависимости:**
- [[hooks|хук]] `useConditionalHeader(WidgetType.FeatureCard)` — шапка текущей страницы с разрешёнными условиями
- [[hooks|хук]] `useHeaderRender(header, WidgetType.FeatureCard)` — функция рендера дочерних элементов шапки
- [[hooks|хук]] `useGlobalContext` — глобальная тема `themeName` как запасная
- [[utils|утилита]] `getThemeByName(themeName ?? pageThemeName)`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `fontColor` | `string` | Цвет текста шапки |
| `bgColor` | `string` | Цвет фона |
| `height` | `number` | Высота |
| `overlay` | `string` | CSS-наложение поверх фона |
| `bigIcon` | `boolean` | Увеличенная декоративная иконка справа (слот `icon`, занимает половину шапки) |
| `withPadding` | `boolean` | Внутренние отступы |
| `bottomBlur` | `boolean` | Эффект размытия снизу |
| `themeName` | `"light" \| "dark"` | Принудительная тема. Не задана — глобальная тема из [[setup\|GlobalContext]] |
| `column` | `boolean` | Иконка и кнопка закрытия в верхней строке, текст под ними на всю ширину. См. [[#Иконка объекта, раскладка и строки текста]] |
| `bigFeatureIcon` | `boolean` | Иконка объекта (слот `featureIcon`) 72px вместо 42px |
| `multilineTitle` | `boolean` | Заголовок до двух строк (без опции — одна), дальше многоточие |
| `multilineDescription` | `boolean` | Подзаголовок до двух строк (без опции — одна), дальше многоточие |

**Дочерние элементы (по `id`):**

| id | Описание |
|---|---|
| `title` | Заголовок (рендерится через `renderElement({ id: "title", wrap: false })`) |
| `description` | Подзаголовок / краткое описание |
| `featureIcon` | Иконка объекта слева от заголовка, на месте значка слоя. См. [[#Иконка объекта, раскладка и строки текста]] |
| `bgImage` | Фоновое изображение — универсальный слот, рендерится общим [[components\|`ContainerBackground`]] в слое `ContainerBackgroundLayer` |
| `icon` | Декоративная иконка справа (в `HeaderIcon`) — обычно `type: "svg"` или `type: "icon"` |

**Структура:**
```
ThemeProvider(getThemeByName(themeName ?? pageThemeName))
  └── BackgroundHeaderWrapper($fontColor, $bgColor, $height, $bigIcon, $withPadding, $bottomBlur)
      └── Header($overlay, $isRow=!column)
          ├── HeaderContent(column) → [HeaderLayerIcon, FeatureCardTitle(title, description), FeatureCardButtons]
          ├── ContainerBackground → ContainerBackgroundLayer → renderElement(id="bgImage")
          └── HeaderIcon → renderElement(id="icon")
```

```tsx
{
  header: {
    templateName: "Background",
    options: { bgColor: "#2c3e50", fontColor: "#fff", bigIcon: true, height: 120 },
    children: [
      { id: "title", type: "attributeValue", attributeName: "name" },
      { id: "description", type: "attributeValue", attributeName: "category" },
      { id: "icon", type: "svg", attributeName: "iconUrl" },
      { id: "bgImage", type: "image", value: "https://example.com/bg.png" }
    ]
  }
}
```

> ⚠️ `bgImage` — это `type: "image"`, URL лежит в **корневом** `value` (или `attributeName`, или `options.resourceId`), а не в `options.value`. В `options` для image допустимы только `width`, `height` и `fit` (см. [[elements#ElementImage]]).

> [!info] `bgImage` — общий слот, а не приватная деталь шапки
> Фон шапки рисует тот же компонент [[components|`ContainerBackground`]], что и фон обычного контейнера: прежний локальный `ImageContainerBg` убран. Слот `bgImage` универсален — допустим у любого контейнера, кроме `Divider` (см. [[concepts|Основные понятия]], «Универсальные слоты и фон контейнера»). Маска `bottomBlur` у `FeatureCardBackgroundHeader` теперь целится в `ContainerBackgroundLayer`.
>
> По умолчанию картинка растягивается `object-fit: cover`; авторский `options.fit` у элемента перебивает дефолт слоя.

---

## FeatureCardSlideshowHeader

**Назначение:** Шапка со слайдшоу фоновых изображений. Заголовок и описание отображаются поверх изображений. Используется для карточек объектов с фотогалереей.

**Типы:** `templateName = HeaderTemplate.Slideshow` · `FeatureCardSlideshowHeaderOptions` · `FeatureCardSlideshowHeaderConfig`.

**Props:** нет

**Зависимости:**
- [[hooks|хук]] `useConditionalHeader(WidgetType.FeatureCard)` — шапка текущей страницы с разрешёнными условиями
- [[hooks|хук]] `useHeaderRender(header, WidgetType.FeatureCard)`
- [[hooks|хук]] `useGlobalContext` — глобальная тема `themeName` как запасная
- [[utils|утилита]] `getThemeByName(themeName ?? pageThemeName)`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `height` | `number` | Высота блока шапки в px |
| `fontColor` | `string` | Цвет текста |
| `withPadding` | `boolean` | Внутренние отступы вокруг заголовка |
| `themeName` | `"light" \| "dark"` | Принудительная тема. Не задана — глобальная тема из [[setup\|GlobalContext]] (`themeName ?? pageThemeName`) |
| `column` | `boolean` | Иконка и кнопка закрытия в верхней строке, текст под ними на всю ширину. См. [[#Иконка объекта, раскладка и строки текста]] |
| `overlay` | `string` | CSS-наложение |
| `bigFeatureIcon` | `boolean` | Иконка объекта (слот `featureIcon`) 72px вместо 42px |
| `multilineTitle` | `boolean` | Заголовок до двух строк (без опции — одна), дальше многоточие |
| `multilineDescription` | `boolean` | Подзаголовок до двух строк (без опции — одна), дальше многоточие |

**Дочерние элементы (по `id`):**

| id | Описание |
|---|---|
| `title` | Заголовок поверх изображений |
| `description` | Подзаголовок |
| `featureIcon` | Иконка объекта слева от заголовка, на месте значка слоя. См. [[#Иконка объекта, раскладка и строки текста]] |
| `bgImage` | Статическое фоновое изображение (под слайдшоу) — общий [[components\|`ContainerBackground`]] |
| `slideshow` | Источник изображений (`type: "slideshow"`) — `attributeName` со списком URL или `relatedDataSource` |

**Структура:**
```
ThemeProvider(getThemeByName(themeName ?? pageThemeName))
  └── SlideshowHeaderWrapper($fontColor, withPadding, height, big)
      └── Header($overlay, $isRow=!column)
          ├── HeaderContent(column) → [HeaderLayerIcon, FeatureCardTitle(title, description), FeatureCardButtons]
          ├── ContainerBackground → ContainerBackgroundLayer → renderElement(id="bgImage")
          └── HeaderSlideshow(height) → renderElement(id="slideshow")
```

```tsx
{
  header: {
    templateName: "Slideshow",
    options: { height: 200, withPadding: true },
    children: [
      { id: "slideshow", type: "slideshow", attributeName: "photos" },
      { id: "bgImage", type: "image", value: "https://example.com/fallback-bg.png" },
      { id: "title", type: "attributeValue", attributeName: "name" },
      { id: "description", type: "attributeValue", attributeName: "address" }
    ]
  }
}
```

---

## Иконка объекта, раскладка и строки текста

Общее для `FeatureCardDefaultHeader`, `FeatureCardBackgroundHeader` и `FeatureCardSlideshowHeader`: все три собирают иконку, текст и кнопку закрытия общим `HeaderContent`.

```tsx
{
  header: {
    templateName: "Background",
    options: { column: true, bigFeatureIcon: true, multilineTitle: true, multilineDescription: true },
    children: [
      { id: "featureIcon", type: "icon", attributeName: "coat_of_arms" },
      { id: "title", type: "attributeValue", attributeName: "name" },
      { id: "description", type: "attributeValue", attributeName: "category" }
    ]
  }
}
```

### Слот `featureIcon`

Иконка объекта на месте значка слоя слева от заголовка. Это обычный элемент: `HeaderLayerIcon` рендерит его через `renderElement({ id: "featureIcon", wrap: false })` (хук `useHeaderFeatureIcon`), поэтому работают `type`, `attributeName`, `value`, `style` и `conditions`. Основной вариант — `type: "icon"` ([[elements#ElementIcon|ElementIcon]]):
- атрибут-вложение (`subType: Attachments`) — первое **изображение** из списка: SVG рисуется инлайном и перекрашивается `options.fontColor`, растр — картинкой как есть (`object-fit: contain`);
- строковый атрибут или `value` — глиф иконочного шрифта по имени.

**Размер** — 42px, при `bigFeatureIcon` — 72px; `options.fontSize` слота (число px) перебивает оба. Шапка передаёт размер иконке как её `fontSize`, а рамке `FeatureIconContainer` — как ширину и высоту, поэтому место под картинку не схлопывается, пока та грузится. Отступ до текста — `0.5rem`.

Клик по иконке и тултип «Приблизить к объекту» — те же, что у значка слоя.

**Когда остаётся значок слоя.** Слот уступает место штатному значку геометрии, если его нет, он скрыт условием (`visibility: false`) или ему нечего показать: пустой атрибут или `value`, а у вложения — ни одного изображения. Решение принимается по значению до рендера (`hasFeatureIconContent`). При `noFeature` слот не используется: рисуется иконка `no_geom`.

> [!warning] Не путать с `icon` / `bigIcon` у `Background`
> `icon` и `bigIcon` управляют декоративной иконкой справа (`HeaderIcon`). Иконку объекта слева от заголовка задают `featureIcon` и `bigFeatureIcon`.

### Раскладка `column`

Без опции текст стоит справа от иконки, а кнопка закрытия — отдельной колонкой справа (`HeaderFrontView` → `HeaderContainer`). С `column` иконка и кнопка делят верхнюю строку (`HeaderIconRow`, `space-between`), а заголовок и подзаголовок идут под ними на всю ширину (`HeaderColumnView`; подзаголовок без резерва под кнопку). Содержимое прижато к верху шапки и при заданной `height`.

### Строки текста `multilineTitle` / `multilineDescription`

По умолчанию заголовок и подзаголовок занимают одну строку и обрезаются многоточием. С опцией — **до двух** строк, дальше многоточие; длинные слова переносятся (`word-break: break-word`). Опции действуют и на авторские слоты `title`/`description`, и на штатный текст (`titleAttribute`, `feature.id`, имя слоя).

Число строк шапка передаёт пропом `$lines` в `FeatureTitleContainer` и `LayerDescription` (`HeaderTextLinesMixin`). Без пропа эти styled-компоненты обрезают текст на второй строке — так их используют вне шапок карточки (например, заголовок статистики слоя в client-new).

---

## Условное оформление шапок

Шапка и её авторские слоты поддерживают `conditions` с override `options`/`style`/`visibility`. Стиль и скрытие шапки применяются к её корневой обёртке. Стиль скалярного содержимого назначается существующей обёртке слота, а стиль React-элемента — его собственному корню; скрытие применяется к обёртке слота. Дополнительная DOM-обёртка для форматирования не создаётся. Сама шапка остаётся смонтированной. Подробнее — [[conditional-formatting|Условное форматирование]].

У `FeatureCardDefaultHeader` появились авторские `title` и `description`, при этом сохраняются описанные выше fallback и `noFeature`. У `FeatureCardBackgroundHeader` и `FeatureCardSlideshowHeader` прежняя семантика их слотов и fallback не меняется. `DashboardDefaultHeader` по-прежнему не рендерит слот `description`.

---

## Связанные разделы

[[components|Компоненты]] | [[utils|Утилиты]] | [[setup|Подключение]] | [[options|Опции]] | [[types|Типы]]
