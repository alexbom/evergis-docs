# Шапки

> [!danger] Дочерние элементы шапки обязаны иметь slot-`id`
> Каждый ребёнок шапки идентифицируется по slot-`id` из фиксированного набора (`title`, `description`, `bgImage`, `icon`, `slideshow`). Без него элемент шапки не отрисуется. Таблица slot-id и чек-лист — [[authoring|Правила генерации]].

## Обзор

Шапки рендерятся через [[utils|утилиты]] `getDashboardHeader` / `getFeatureCardHeader` по `templateName` из конфига. Точка входа — [[components|компонент]] `DashboardHeader` (для Dashboard) и `FeatureCardHeader` (для FeatureCard).

Дочерние элементы шапки **обязательно** идентифицируются по slot-`id` из фиксированного набора (`title`, `description`, `bgImage`, `icon`, `slideshow`) — конкретный набор зависит от типа шапки. Подробности и таблица slot-id — в [[concepts#ID контейнеров и элементов|разделе про id]].

`HeaderTemplate` enum (см. [[types|Типы]]):

| Значение | Шапка | Виджет |
|---|---|---|
| `HeaderTemplate.Default` | **DashboardDefaultHeader** | `WidgetType.Dashboard` |
| `HeaderTemplate.Default` | **FeatureCardDefaultHeader** | `WidgetType.FeatureCard` |
| `HeaderTemplate.Background` | **FeatureCardBackgroundHeader** | `WidgetType.FeatureCard` |
| `HeaderTemplate.Slideshow` | **FeatureCardSlideshowHeader** | `WidgetType.FeatureCard` |

Union типов конфигов шапок: `DashboardHeaderConfig` (см. [[types#Union шапок DashboardHeaderConfig|типы]]). `HeaderTemplate.Default` сразу матчится двумя ветвями (Dashboard vs FeatureCard) — конкретная выбирается на уровне виджета.

Shared-компоненты в `headers/components/`:
- **HeaderLayerIcon** — кликабельная иконка слоя с тултипом «приблизить к объекту»; используется во всех `FeatureCard*Header`.
- **HeaderTitle** — заголовок карточки (значение `titleAttribute` или `feature.id`) + описание слоя; используется в `FeatureCardDefaultHeader`.

---

## DashboardDefaultHeader

**Назначение:** Шапка дашборда — логотип, кнопки панели, заголовок текущей страницы, пагинация, меню страниц.

**Типы:** `templateName = HeaderTemplate.Default` · `DashboardDefaultHeaderOptions` · `DashboardDefaultHeaderConfig`. См. [[types#Шапки|сводную таблицу]].

**Props:** нет (данные из контекста)

**Зависимости:**
- [[hooks|хук]] `useDashboardHeader` — `title`, `pageId`, `image`, `icon`, `tooltip`, `themeName`, `onClickLogo`
- [[hooks|хук]] `useWidgetContext` — `toggleLayersVisibility`, `components.ProjectPanelMenu`, `components.ProjectPagesMenu`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `url` | `string` | URL логотипа (через `image` из `useDashboardHeader`) |

Остальные данные (image, icon, tooltip, themeName) приходят из `useDashboardHeader`, который читает `projectInfo.content.dashboardConfiguration.header.options` напрямую и нормализует.

**Структура:**
```
DefaultHeaderContainer (image, isDark)
  ├── LinearProgress (если !pageId — страница загружается)
  ├── TopContainer: [LogoContainer(icon), ProjectPanelMenu]
  └── PageTitleContainer: [Tooltip(PageTitle onClick=toggleLayersVisibility), ProjectPagesMenu, Pagination]
```

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
- [[hooks|хук]] `useWidgetConfig(WidgetType.FeatureCard)` — конфиг карточки
- [[utils|утилита]] `getThemeByName(themeName)` — применение темы к шапке

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `themeName` | `"light" \| "dark"` | Принудительная тема для шапки |
| `withPadding` | `boolean` | Внутренние отступы вокруг заголовка |
| `column` | `boolean` | Вертикальная раскладка (иначе строкой) |
| `height` | `number` | Высота шапки в px |
| `overlay` | `string` | CSS-наложение поверх содержимого |

**Дочерние элементы:**

Кастомные дети не используются — структура фиксирована.

**Структура:**
```
DefaultHeaderWrapper(withPadding, height)
  └── ThemeProvider(getThemeByName(themeName))
      └── Header($overlay, $isRow=!column)
          └── HeaderFrontView(isDefault=!column)
              ├── HeaderContainer: [HeaderLayerIcon, HeaderTitle(noFeature)]
              └── FeatureCardButtons (Edit, Save, Cancel, Close)
```

```tsx
{ header: { templateName: "Default", options: { themeName: "light", withPadding: true } } }
```

---

## FeatureCardBackgroundHeader

**Назначение:** Шапка карточки с цветной/градиентной заливкой фона и опциональным фоновым изображением. Подходит для эмоционального выделения карточки и визуального обозначения типа объекта (через `bigIcon` или `bgImage`).

**Типы:** `templateName = HeaderTemplate.Background` · `FeatureCardBackgroundHeaderOptions` · `FeatureCardBackgroundHeaderConfig`.

**Props:** нет

**Зависимости:**
- [[hooks|хук]] `useWidgetConfig(WidgetType.FeatureCard)` — конфиг карточки
- [[hooks|хук]] `useHeaderRender(header)` — функция рендера дочерних элементов шапки
- [[utils|утилита]] `getThemeByName(themeName)`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `fontColor` | `string` | Цвет текста шапки |
| `bgColor` | `string` | Цвет фона |
| `height` | `number` | Высота |
| `overlay` | `string` | CSS-наложение поверх фона |
| `bigIcon` | `boolean` | Увеличенная иконка (занимает половину шапки) |
| `withPadding` | `boolean` | Внутренние отступы |
| `bottomBlur` | `boolean` | Эффект размытия снизу |
| `themeName` | `"light" \| "dark"` | Принудительная тема |
| `column` | `boolean` | Вертикальная раскладка |

**Дочерние элементы (по `id`):**

| id | Описание |
|---|---|
| `title` | Заголовок (рендерится через `renderElement({ id: "title", wrap: false })`) |
| `description` | Подзаголовок / краткое описание |
| `bgImage` | Фоновое изображение (в `ImageContainerBg`) |
| `icon` | Иконка объекта (в `HeaderIcon`) — обычно `type: "svg"` или `type: "icon"` |

**Структура:**
```
BackgroundHeaderWrapper($fontColor, $bgColor, $height, $bigIcon, $withPadding, $bottomBlur)
  └── ThemeProvider(getThemeByName(themeName))
      └── Header($overlay, $isRow=!column)
          ├── HeaderFrontView
          │   ├── HeaderContainer(column): [HeaderLayerIcon, FeatureCardTitle(title, description)]
          │   └── FeatureCardButtons
          ├── ImageContainerBg → renderElement(id="bgImage")
          └── HeaderIcon → renderElement(id="icon")
```

```tsx
{
  header: {
    templateName: "Background",
    options: { bgColor: "#2c3e50", fontColor: "#fff", bigIcon: true, height: 120 },
    children: [
      { id: "title", attributeName: "name" },
      { id: "description", attributeName: "category" },
      { id: "icon", type: "svg", attributeName: "iconUrl" },
      { id: "bgImage", type: "image", value: "https://example.com/bg.png" }
    ]
  }
}
```

> ⚠️ `bgImage` — это `type: "image"`, URL лежит в **корневом** `value` (или `attributeName`), а не в `options.value`. В `options` для image допустимы только `width`, `height` и `fit` (см. [[elements#ElementImage]]).

---

## FeatureCardSlideshowHeader

**Назначение:** Шапка со слайдшоу фоновых изображений. Заголовок и описание отображаются поверх изображений. Используется для карточек объектов с фотогалереей.

**Типы:** `templateName = HeaderTemplate.Slideshow` · `FeatureCardSlideshowHeaderOptions` · `FeatureCardSlideshowHeaderConfig`.

**Props:** нет

**Зависимости:**
- [[hooks|хук]] `useWidgetConfig(WidgetType.FeatureCard)`
- [[hooks|хук]] `useHeaderRender(header)`
- [[utils|утилита]] `getThemeByName(themeName)`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `height` | `number` | Высота блока шапки в px |
| `fontColor` | `string` | Цвет текста |
| `withPadding` | `boolean` | Внутренние отступы вокруг заголовка |
| `themeName` | `"light" \| "dark"` | Принудительная тема |
| `column` | `boolean` | Вертикальная раскладка |
| `overlay` | `string` | CSS-наложение |

**Дочерние элементы (по `id`):**

| id | Описание |
|---|---|
| `title` | Заголовок поверх изображений |
| `description` | Подзаголовок |
| `bgImage` | Статическое фоновое изображение (под слайдшоу) |
| `slideshow` | Источник изображений (`type: "slideshow"`) — `attributeName` со списком URL или `relatedDataSource` |

**Структура:**
```
SlideshowHeaderWrapper(fontColor, withPadding, height, big)
  └── ThemeProvider(getThemeByName(themeName))
      └── Header($overlay, $isRow=!column)
          ├── HeaderFrontView
          │   ├── HeaderContainer(column): [HeaderLayerIcon, FeatureCardTitle(title, description)]
          │   └── FeatureCardButtons
          ├── ImageContainerBg → renderElement(id="bgImage")
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
      { id: "title", attributeName: "name" },
      { id: "description", attributeName: "address" }
    ]
  }
}
```

---

## Связанные разделы

[[components|Компоненты]] | [[utils|Утилиты]] | [[setup|Подключение]] | [[options|Опции]] | [[types|Типы]]
