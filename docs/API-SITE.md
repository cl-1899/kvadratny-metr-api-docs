Подключение (JavaScript):

Ключи — в `docs/.env.site` (передаются отдельно, не в git). Шаблон имён переменных: `docs/.env.example`.

```js
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_ANON_KEY
)
```

---

Картинки лежат в **публичных** бакетах Storage (`product-images`, `category-images`). В ответах уже приходят полные URL в `image_url` — отдельно Storage API для показа не нужен.

---

## Четыре «эндпоинта» (таблицы для select)

| View | Зачем |
|------|--------|
| `public_categories_view` | Меню каталога, дерево категорий |
| `public_products_view` | Списки товаров, плитки, цена «от» |
| `public_product_variants_view` | Карточка товара: размер, поверхность, цена, скидка |
| `public_product_images_view` | Галерея фото товара |

Во всех view уже отобрано только то, что можно показывать на сайте:

- товар `is_active = true`;
- категория товара активна (если привязана);
- вариант `is_active = true` (в view вариантов);
- категории: `is_active = true`, подкатегория не показывается, если выключен родитель.

Колонки `is_active` в view **нет** — фильтр уже внутри.

---

## 1. Категории — `public_categories_view`

**Поля:**

| Поле | Тип | Смысл |
|------|-----|--------|
| `id` | uuid | ID |
| `name` | text | Название |
| `slug` | text | Для URL, уникальный |
| `parent_id` | uuid \| null | `null` = корневая категория |
| `image_url` | text \| null | Картинка раздела |
| `parent_slug`, `parent_name` | text \| null | Родитель (для подкатегорий) |
| `created_at`, `updated_at` | timestamptz | Даты |

**Пример — все категории:**

```js
const { data, error } = await supabase
  .from('public_categories_view')
  .select('*')
  .order('name')
```

**Дерево на клиенте:**  
`parent_id === null` → корни; остальные — дети с `parent_id = id` корня.

**Одна категория по slug:**

```js
const { data } = await supabase
  .from('public_categories_view')
  .select('*')
  .eq('slug', 'plitka')
  .maybeSingle()
```

**Подкатегории раздела:**

```js
const { data } = await supabase
  .from('public_categories_view')
  .select('*')
  .eq('parent_id', rootCategoryId)
```

Товары вешаются на **любую** категорию (редко на подкатегорию) — поле `category_id` в товаре.

P.S. **Подкатегории пока не используются вообще тут, они были сделаны, но потом Саша сказал что не нужны, я уговорил его оставить на всякий случай, чтоб уже не удалять, мало ли потом понадобятся**

---

## 2. Товары — `public_products_view`

**Поля:**

| Поле | Смысл |
|------|--------|
| `id`, `name`, `sku` | Товар |
| `category_id`, `category_name`, `category_slug` | Категория |
| `youtube_url` | Ссылка на видео (может быть null) |
| `brand_name`, `country_name`, `collection_name`, `sort_name` | Подписи характеристик (текст, не id) |
| `brand_option_id`, … | uuid опций — если нужны фильтры «по id» |
| `price_from`, `price_to` | Min/max цены по **активным** вариантам, где цена **заполнена** |
| `variants_count` | Число активных вариантов |
| `created_at`, `updated_at` | Даты |

Цена на уровне товара **не хранится** — только у вариантов. Если у всех вариантов цена пустая → `price_from` / `price_to` будут `null`.

**Список в категории (включая подкатегории):**  
сначала собрать id категории + детей из `public_categories_view`, потом:

```js
const categoryIds = [rootId, ...childIds]

const { data } = await supabase
  .from('public_products_view')
  .select('*')
  .in('category_id', categoryIds)
  .order('name')
```

**Только одна подкатегория:**

```js
.eq('category_id', subcategoryId)
```

**Поиск по названию / артикулу:**

```js
.or(`name.ilike.%${q}%,sku.ilike.%${q}%`)
```

**Один товар:**

```js
.eq('id', productId).maybeSingle()
// или по sku:
.eq('sku', 'ABC123').maybeSingle()
```

**Фильтр по бренду (текст):**

```js
.eq('brand_name', 'ESTIMA')
```

**Пагинация:**

```js
.range(0, 23)   // первые 24
// count: 'exact' в select options supabase v2:
.select('*', { count: 'exact' })
```

---

## 3. Варианты — `public_product_variants_view`

Один товар = несколько строк (размер × поверхность × цена).

| Поле | Смысл |
|------|--------|
| `id` | ID варианта |
| `product_id` | Товар |
| `size_name`, `surface_name` | Подписи |
| `size_option_id`, `surface_option_id` | uuid (если нужны ключи) |
| `price` | number \| null — базовая цена, **может быть пустой** |
| `discount_percent` | number \| null — скидка в % (0–100) |
| `is_on_sale` | boolean — на распродаже товар или нет |
| `is_recommended` | boolean — рекомендуемый вариант или нет |
| `sort_order` | порядок в админке |
| `product_name`, `product_sku`, `brand_name` | дубли с товара для удобства |

**Все варианты товара:**

```js
const { data } = await supabase
  .from('public_product_variants_view')
  .select('*')
  .eq('product_id', productId)
  .order('sort_order')
  .order('id')
```

**Цена со скидкой на фронте** (как в админке):

```js
function priceWithDiscount(price, discountPercent) {
  if (price == null) return null
  const p = Number(price)
  const d = Number(discountPercent)
  if (!Number.isFinite(p)) return null
  if (!Number.isFinite(d) || d <= 0 || d > 100) return p
  return Math.round(p * (1 - d / 100) * 100) / 100
}
```

Показывать зачёркнутую старую и новую цену, если `discount_percent > 0`.  
`is_on_sale` — для бейджа «Акция», логику отображения можно связать с ним.

**Рекомендуемый вариант на карточке:**

```js
const recommended = variants.find(v => v.is_recommended) ?? variants[0]
```

---

## 4. Картинки — `public_product_images_view`

| Поле | Смысл |
|------|--------|
| `id` | ID записи |
| `product_id` | Товар |
| `image_url` | **Готовый URL** для `<img src>` |
| `image_path` | путь в Storage (обычно не нужен на сайте) |
| `sort_order` | порядок: 1 — главное фото |
| `product_name`, `product_sku`, `category_id` | справочно |

```js
const { data } = await supabase
  .from('public_product_images_view')
  .select('id, product_id, image_url, sort_order')
  .eq('product_id', productId)
  .order('sort_order')
  .order('id')
```

Превью в списке товаров: первое фото по `sort_order` (запросить пачкой по списку `product_id` через `.in('product_id', ids)`).

---

## Типовые сценарии

### Главная / каталог

1. `public_categories_view` → меню.  
2. По выбранному разделу → `public_products_view` с фильтром по `category_id`.  
3. Для превью → `public_product_images_view` с `.in('product_id', …)`, взять с минимальным `sort_order`.

### Страница товара

1. `public_products_view` — `.eq('id', id).maybeSingle()`.  
2. `public_product_variants_view` — `.eq('product_id', id)`.  
3. `public_product_images_view` — галерея.  
4. `youtube_url` — встроить плеер, если есть.

### URL

Обычно: `/catalog/:category_slug` и `/product/:product_id` или slug товара, если заведёте на фронте (в БД у товара slug нет, есть `sku` и `id`).

---

## Ошибки

| Симптом | Что проверить |
|---------|----------------|
| 401 / Invalid API key | URL и anon key |
| Пусто, хотя в админке есть | Товар/категория/вариант выключены (`is_active`) |
| permission denied for view | Не выполнен grant для `anon` (миграция выше) |
| `price_from` null | У вариантов не заполнена цена |

---
