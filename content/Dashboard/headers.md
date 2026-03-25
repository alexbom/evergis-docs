# Шапки

## Обзор

Шапки рендерятся через [[utils|утилиты]] `getDashboardHeader` / `getFeatureCardHeader` по `templateName` из конфига. Точка входа — [[components|компонент]] `DashboardHeader` (для Dashboard) и `FeatureCardHeader` (для FeatureCard).

Доступные типы для Dashboard: `HeaderTemplate.Default`.
Доступные типы для FeatureCard: `HeaderTemplate.Default`, `HeaderTemplate.Gradient`, `HeaderTemplate.Icon`, `HeaderTemplate.Slideshow`.

---

## DashboardDefaultHeader

**Назначение:** Шапка дашборда — логотип, кнопки панели, заголовок текущей страницы, пагинация, меню страниц.

**Props:** нет (данные из контекста)

**Зависимости:**
- [[hooks|хук]] `useDashboardHeader` — `title`, `pageId`, `image`, `icon`, `tooltip`, `themeName`, `onClickLogo`
- `useWidgetContext` — `toggleLayersVisibility`, `components.ProjectPanelMenu`, `components.ProjectPagesMenu`

**Опции конфига шапки:**

| Опция | Тип | Описание |
|---|---|---|
| `image` | `string` | URL логотипа (поддерживает `/sp/resources/file/...`) |
| `icon` | `IconTypesKeys` | Иконка логотипа (если нет изображения) |
| `tooltip` | `string` | Тултип при наведении на логотип |
| `title` | `string \| attributeName` | Заголовок страницы (из конфига или атрибута) |

**Структура:**
```
DefaultHeaderContainer (image, isDark)
  ├── LinearProgress (если !pageId — страница загружается)
  ├── TopContainer: [LogoContainer, ProjectPanelMenu]
  └── PageTitleContainer: [Tooltip(PageTitle), ProjectPagesMenu, Pagination]
```

```tsx
// Регистрируется автоматически как HeaderTemplate.Default
{ header: { templateName: "Default", options: { image: "/logo.svg", tooltip: "На главную" } } }
```

---

## FeatureCardDefaultHeader

**Назначение:** Шапка карточки объекта по умолчанию — иконка слоя, название объекта, кнопки действий (редактирование, закрытие).

**Props:** `noFeature?: boolean`

**Зависимости:** `useWidgetContext(WidgetType.FeatureCard)` — `layerInfo`

**Опции конфига шапки:**

Конфига нет — все данные автоматически берутся из `FeatureCardContext` (`layerInfo`, `attributes`). Иконка слоя — из `layerInfo.configuration.icon`. Заголовок — значение `idAttribute` объекта.

**Структура:**
```
DefaultHeaderWrapper
  └── Header ($isRow)
      └── HeaderFrontView (isDefault)
          ├── HeaderContainer: [LayerIcon, HeaderTitle(noFeature)]
          └── FeatureCardButtons (кнопки: Edit, Save, Cancel, Close)
```

```tsx
{ header: { templateName: "Default" } }
```

---

## FeatureCardGradientHeader

**Назначение:** Шапка с градиентным фоном. Поддерживает кастомный цвет фона и текста. Подходит для эмоционального выделения карточки.

**Props:** `isRow?: boolean`

**Зависимости:** `useWidgetConfig(WidgetType.FeatureCard)`, `useHeaderRender(header)`, `useWidgetContext`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `bgColor` | `string` | Цвет фона шапки (CSS-значение, например `"#2c3e50"`) |
| `fontColor` | `string` | Цвет текста и иконок шапки (например `"#ffffff"`) |

**Дочерние элементы шапки (по `id`):**

| id | Описание |
|---|---|
| `title` | Заголовок (из атрибута или статически) |
| `description` | Подзаголовок / краткое описание |

```tsx
{
  header: {
    templateName: "Gradient",
    options: { bgColor: "#2c3e50", fontColor: "#fff" },
    children: [
      { id: "title", attributeName: "name" },
      { id: "description", attributeName: "address" }
    ]
  }
}
```

---

## FeatureCardIconHeader

**Назначение:** Шапка с большой иконкой рядом с заголовком. Поддерживает режим `bigIcon` для акцента на иконке типа объекта.

**Props:** `isRow?: boolean`

**Зависимости:** `useWidgetConfig(WidgetType.FeatureCard)`, `useHeaderRender(header)`, `useWidgetContext`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `bgColor` | `string` | Цвет фона шапки |
| `fontColor` | `string` | Цвет текста |
| `bigIcon` | `boolean` | Увеличенная иконка (занимает половину шапки, подходит для иконок типа объекта) |

**Дочерние элементы шапки (по `id`):**

| id | Описание |
|---|---|
| `icon` | SVG или иконка типа объекта (`type: "svg"` или `type: "icon"`) |
| `title` | Заголовок карточки |
| `description` | Подзаголовок |

```tsx
{
  header: {
    templateName: "Icon",
    options: { bigIcon: true, bgColor: "#f0f4f8" },
    children: [
      { id: "icon", type: "svg", attributeName: "iconUrl" },
      { id: "title", attributeName: "name" },
      { id: "description", attributeName: "category" }
    ]
  }
}
```

---

## FeatureCardSlideshowHeader

**Назначение:** Шапка со слайдшоу фоновых изображений. Заголовок и описание отображаются поверх изображений. Используется для карточек объектов с фотографиями.

**Props:** `isRow?: boolean`

**Зависимости:** `useWidgetConfig(WidgetType.FeatureCard)`, `useHeaderRender(header)`, `useWidgetContext`

**Опции:**

| Опция | Тип | Описание |
|---|---|---|
| `height` | `number \| string` | Высота блока шапки в px или CSS-значение |
| `withPadding` | `boolean` | Добавить внутренние отступы вокруг заголовка |

**Дочерние элементы шапки (по `id`):**

| id | Описание |
|---|---|
| `slideshow` | Источник изображений — `attributeName` с URL фото через разделитель, или `relatedDataSource` |
| `bgImage` | Статическое фоновое изображение (если нет слайдшоу) |
| `title` | Заголовок поверх изображений |
| `description` | Подзаголовок поверх изображений |

```tsx
{
  header: {
    templateName: "Slideshow",
    options: { height: 200, withPadding: true },
    children: [
      { id: "slideshow", attributeName: "photos" },
      { id: "title", attributeName: "name" },
      { id: "description", attributeName: "address" }
    ]
  }
}
```

---

## Связанные разделы

[[components|Компоненты]] | [[utils|Утилиты]] | [[setup|Подключение]]
