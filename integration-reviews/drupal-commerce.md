# Интеграционный план: Drupal Commerce

Реализация стандарта `INTEGRATION-STANDARD.md` (эталон — WooCommerce) на **Drupal 10/11 + Commerce 2/3**.
Целевой формат поставки — **кастомный модуль** `onecatalog_import` (machine name; Drupal не допускает дефис в имени модуля → не `onecatalog-import`).

Базовые зависимости модуля (`onecatalog_import.info.yml`):

```yaml
name: 'OneCatalog Import'
type: module
description: 'Импорт каталога OneCatalog в Drupal Commerce.'
core_version_requirement: ^10 || ^11
package: Commerce
dependencies:
  - commerce:commerce_product
  - commerce:commerce_price        # обязателен транзитивно для product variation
  - drupal:taxonomy
  - drupal:media
  - drupal:file
  - drupal:options
  # опционально, но рекомендуется для §5.6 (вес/размеры):
  # - physical:physical            # contrib commerceguys/physical
  # - drupal:content_translation   # для i18n сущностей
```

Слои стандарта (§4) ложатся на сервисы Drupal (DI-контейнер, `*.services.yml`), а не на классы-синглтоны:

| Слой стандарта | Реализация Drupal |
|---|---|
| `Plugin` | `.module` + `.install` (hook_install/update_N) + `.info.yml` |
| `Api` | сервис `onecatalog_import.api` на базе `\GuzzleHttp\ClientInterface` (`http_client`) |
| `Units` | сервис `onecatalog_import.units` (чистый конвертер, без зависимостей от БД) |
| `Media` | сервис `onecatalog_import.media` (использует `file.repository`, `entity_type.manager`, `http_client`) |
| `Taxonomies` | сервис `onecatalog_import.taxonomy` (резолв термов/словарей) |
| `CollectionImporter`/`BrandImporter`/`CountryImporter` | отдельные сервисы-резолверы справочных сущностей |
| `ProductImporter` | сервис-оркестратор `onecatalog_import.product_importer` |
| `Queue` | `QueueWorker` плагин (`@QueueWorker`) + cron, либо Batch API |
| `Rest` | контроллер маршрута (`*.routing.yml`) или REST-ресурс плагин |
| `Settings` | Config API: `ConfigFormBase` + `config/install/onecatalog_import.settings.yml` + schema |
| `AdminUI` | `hook_form_alter` на форме товара/коллекции + library с picker-loader |
| `assets/picker-loader`, `admin-import` | Drupal libraries (`*.libraries.yml`) + `drupalSettings` для передачи токена/origin |

---

## Маппинг сущностей и реализация

### Сводная таблица соответствий (§3 → Drupal)

| OneCatalog | Абстракция | Drupal Commerce | Тип объекта / механизм |
|---|---|---|---|
| `product` | Товар | `commerce_product` + минимум 1 `commerce_product_variation` | content entity, bundle (product type) задаётся настройкой |
| `public_id` | Внешний ключ | поле `field_oc_public_id` на `commerce_product` | base/bundle field, индексируется; идемпотентность через `entityQuery` |
| `article` | Артикул произв. | `sku` у `commerce_product_variation` | родное поле variation |
| `options[]` | Характеристика | **выбор**: `commerce_product_attribute` (для вариаций) ИЛИ taxonomy `field_oc_spec_*` ИЛИ обычное поле | по умолчанию — taxonomy-термины через entity_reference поле (см. ниже) |
| `categories[]` | Категория | taxonomy vocabulary (напр. `product_categories`) | entity_reference (multi) с `field_oc_category` |
| `brand` | Бренд | **выбор**: taxonomy `brands` \| node \| attribute | «тип объекта + цель» (см. ниже) |
| `country` | Страна происх. | **выбор**: taxonomy `countries` \| поле | «тип объекта + цель» |
| `collections[]` | Коллекция | **выбор**: node \| taxonomy \| custom entity | «тип объекта + цель» |
| `tags[]` | Теги | taxonomy `tags` | entity_reference (вкл/выкл) |
| `images_urls`/`files` | Медиа | `media` (bundle image) + `file` | дедуп по контент-ключу |
| `measurement_unit`, `sizes`, `areas_per_package`, `product_shape`, alt | прочее | через событие `OneCatalogEvents::PRODUCT_IMPORTED` | сайтовый слой (отдельный модуль) |

### 1. Товар → Commerce Product / Variation

Принципиальное расхождение с WooCommerce: в WooCommerce «товар» — одна сущность (CPT `product`) с SKU. В Commerce товар разделён на **две** сущности:

- `commerce_product` — карточка (заголовок, описание, привязки к таксономиям, статус публикации `status`, langcode).
- `commerce_product_variation` — покупаемая единица: **держит `sku`, `price`, вес/размеры**.

OneCatalog payload описывает **одну** единицу (один `article`, один `public_id`, один набор `sizes`). Значит:

- На каждый импортируемый товар создаём **1 product + 1 variation** (single-variation product).
- `public_id` пишем в поле на уровне **product** (карточка идемпотентна).
- `article` → `sku` variation (§5.2). Если `article` пуст — sku обязателен и **уникален** в Commerce, поэтому нужен фолбэк: сгенерировать sku из `public_id` (напр. `OC-<public_id>`), но это противоречит §5.2 («оставлять пустым»). → **вопрос №1** ниже. Рабочее решение: если `article` пуст — sku = `public_id` (помечен как авто), при этом «родной артикул для покупателя» хранить отдельно нельзя оставить пустым, т.к. Commerce запрещает пустой SKU на стандартной variation.
- Тип товара (product type bundle) и тип вариации (variation type) — **настройка** (`product_type`, `variation_type`), т.к. в Drupal их может быть несколько.
- **Цена НЕ задаётся** (§5.6). Но `commerce_product_variation` по умолчанию имеет **обязательное** поле `price`. → см. блок «🚫 Сложно/невозможно» — это главный конфликт со стандартом.

Создание:

```php
$variation = ProductVariation::create([
  'type' => $config->get('variation_type'),
  'sku' => $sku,
  // price НЕ задаём → нужно либо снять required, либо ставить плейсхолдер
]);
$variation->save();

$product = Product::create([
  'type' => $config->get('product_type'),
  'title' => $payload['menutitle'],
  'stores' => $defaultStores,           // Commerce требует привязку к store(ам)
  'variations' => [$variation],
  'field_oc_public_id' => $payload['public_id'],
]);
```

⚠️ **Commerce требует привязку product к одному или нескольким `commerce_store`.** WooCommerce этого не знает. При импорте без явного выбора store товар не будет покупаемым/видимым. → настройка «целевой магазин(ы)» обязательна (по умолчанию — default store).

### 2. Идемпотентность по `public_id` (§5.1)

- Поле `field_oc_public_id` — `string`, добавляется через `BaseFieldDefinition` (предпочтительно, чтобы было индексируемым и не зависело от bundle) либо config-field. Рекомендую base field в `hook_entity_base_field_info()` для `commerce_product` — переносимо между bundle и быстрее в `entityQuery`.
- Поиск перед импортом:

```php
$ids = $storage->getQuery()
  ->accessCheck(FALSE)
  ->condition('field_oc_public_id', $publicId)
  ->range(0, 1)
  ->execute();
$product = $ids ? $storage->load(reset($ids)) : NULL;
```

- **Справочники ищем по имени (label), не по слагу** (§5.1, критично). В Drupal:
  - taxonomy term: `entityQuery` по `name` + `vid`, БЕЗ учёта регистра. `name` — обычное string-поле; для регистронезависимости либо `->condition('name', $name)` (зависит от collation БД — MySQL utf8mb4 по умолчанию case-insensitive ci, что нам и нужно) либо нормализовать вручную. ⚠️ collation БД влияет на корректность — нельзя полагаться слепо (PostgreSQL — case-sensitive по умолчанию). → **вопрос про СУБД**.
  - slug/`alias`/machine-name в Drupal у термов нет вообще (есть только tid + name; URL-alias — отдельная подсистема). Это устраняет «проблему транслитерации» эталона на уровне термов, НО проблема всплывает в machine-name **словарей/полей/атрибутов** при автосоздании — там нужна транслитерация (`\Drupal::transliteration()`), и она своя. То есть инвариант §5.1 для термов решается сам собой (ищем по name), а для машинных имён vocabulary/attribute создаваемых на лету — остаётся.

- Легаси-фолбэк (§5.1): если ранее ключ лежал в sku — добавить фолбэк-поиск по `sku` variation и миграцию в update hook.

### 3. Характеристики `options[]` (§5.4) — ключевая развилка

Это самое неоднозначное место маппинга. В Commerce есть ДВА разных механизма, и стандарт («глобальный атрибут + термин») напрямую ложится только на один из них с оговорками:

**Вариант A — `commerce_product_attribute` (родной механизм вариаций Commerce).**
- Атрибут (`commerce_product_attribute`, напр. «Цвет») → значения `commerce_product_attribute_value` (entity). Это ближайший аналог WooCommerce «глобальный атрибут `pa_*` + термины».
- НО: атрибуты в Commerce существуют **ради генерации вариаций** (выбор покупателем цвет/размер → разные SKU/цены). У нас **одна** variation на товар. Использовать attribute для чисто описательной характеристики («толщина 9 мм») — это abuse механизма: каждое значение требует, чтобы variation type имел включённый этот атрибут, и плодит пустые комбинации. Технически работает (атрибут можно сделать описательным), но семантически кривовато.
- attribute value привязывается к **variation**, не к product.

**Вариант B — taxonomy-термины + entity_reference поле (рекомендуемый по умолчанию).**
- На каждую характеристику OneCatalog (`specification_label`) — отдельный vocabulary (напр. `oc_spec_thickness`) ИЛИ один общий vocabulary с термами, и поле `field_oc_spec_<id>` (entity_reference на vocabulary) на product.
- Это ближе к «фасет-термин» и к тому, как WooCommerce-эталон по факту использует глобальные атрибуты (как фасеты, а не как генераторы вариаций).
- Идемпотентность термов — по name (§5.1).

**Вариант C — обычное поле** (`field_oc_spec_*` типа string/decimal/boolean) без таксономии — для numeric/boolean, где фасет не нужен.

Стандарт говорит «глоб. атрибут `pa_*` + термины», что в Drupal честнее всего = **Вариант B** (taxonomy), а не Commerce attribute. Поэтому модуль должен дать **настройку режима хранения характеристик** (как и для бренда/страны): `attribute | taxonomy | field`. По умолчанию — `taxonomy`.

Типы из payload (§2.2):
- `text` → термин (option_name) в vocabulary характеристики.
- `boolean` → значение из `bool_option`; при `null`/отсутствии → **false** (§5.6). Нужен явный «терм для true / терм для false» (или boolean-поле). При ручном маппинге задаются термы true/false.
- `numeric` → `numeric_option`; либо decimal-поле, либо термин «<число> <ед>».

**Ручной маппинг (строгий режим, §5.4):** отдельная форма настроек, карта `specification_id → {field/attribute machine_name, true_term, false_term}`. При ВКЛ импортируем ТОЛЬКО сопоставленные; ключ — `specification_id` (стабильный, не переводимый label). Хранить карту в config (`onecatalog_import.spec_mapping`).

### 4. Категории `categories[]` → Taxonomy

- Vocabulary категорий — настройка (`category_vocabulary`, по умолчанию создаётся `product_categories` в hook_install). Иерархии у categories из payload нет (плоский список id/menutitle/slug), parent не задаём.
- find-or-create term **по name** (§5.1), привязка через multi-value entity_reference поле `field_oc_category` на product.

### 5. Бренд / Страна — «тип объекта + цель» (§3, §7)

Реализуется единым паттерном настройки. Для каждого:
- **тип объекта:** `taxonomy` | `entity_reference на node` | `attribute` | (страна — `taxonomy` | `field`).
- **цель:** конкретный vocabulary / node type / attribute + поле на product, куда писать ссылку.

Бренд богаче категории (`description_text`, `foundation_year`, `website`, `country`, `images`). Если цель — taxonomy term: доп. поля кладём на term (нужны поля на vocabulary) либо игнорируем в ядре и отдаём в событие `brand_imported`. Если цель — node: полноценная страница бренда с полями. По умолчанию: бренд → taxonomy `brands`; страна → выкл (или taxonomy `countries`).

В Drupal **нет «нативной таксономии брендов»** (в отличие от WooCommerce 8+, где `product_brand` встроена). → её создаёт hook_install модуля. **вопрос про дефолт.**

### 6. Коллекции `collections[]` — «тип объекта + цель»

- node (тип `oc_collection` с полями `link_3d`, `link_official_site`, изображения) — даёт страницу коллекции; **по умолчанию** node.
- или taxonomy `collections`.
- или custom content entity (если нужен лёгкий объект без node-оверхеда).
- find-or-create по `public_id`/name коллекции, привязка к product через `field_oc_collection`.
- ⚠️ `/collections/{slug}/` отдаёт 500 (§2.1) — НЕ дёргать, все поля берём из эмбеда `collections[]` товара.

### 7. Теги → Taxonomy `tags`

- вкл/выкл (`import_tags`). Vocabulary `tags`, find-or-create по `title` (у тегов поле `title`, не `menutitle`!), привязка `field_oc_tags`.

### 8. Медиа → Media + File (§2.3, §5.3)

Это самая ёмкая часть. В Drupal правильный таргет — **`media` entity (bundle `image`)**, которая оборачивает `file`. Привязка к product/variation — через entity_reference на media (galleries) + одно «featured».

- **Скачивание вручную** (URL без расширения, §2.3): Guzzle GET → тело + MIME из заголовка/контента. НЕ использовать «sideload»-хелперы (их в Drupal-аналоге — `system_retrieve_file()` — тоже могут спотыкаться на URL без расширения). Сохранять через `file.repository` (`writeData()`), имя файла генерировать с расширением по MIME (`image/jpeg → .jpg`).
- Из `file` создаём `media` (bundle image, поле `field_media_image`), `alt`/`title` из `images_data`/`files[].alt`.
- **Трекинг качества (§5.3):** рядом с файлом/media хранить выбранный размер. В Drupal у file/media нет произвольных мет «из коробки» → добавить **поля на media bundle**: `field_oc_size` (min/middle/max), `field_oc_content_key` (контент-ключ для дедупа), `field_oc_source_path`. На повторном импорте: если доступен размер лучше — перекачать, заменить file, старый удалить (если НЕ общий).
- **Дедуп по контент-ключу (§5.3):** ключ = `hash(path # size)` из base64-префикса `media_files`. Перед загрузкой `entityQuery` по media: `condition('field_oc_content_key', $key)` → нашли → переиспользуем media (без скачивания). Общие файлы при апгрейде качества **не удалять** (ломает другие товары).
- **Галерея идемпотентна по имени файла** (§5.3), не по URL (подписи в URL меняются).
- featured = `images_urls`, галерея = `files[category=images]`.

Альтернатива «media»: писать напрямую в `file` + поле `image` на product (без media entity). Проще, но теряем переиспользование/UI медиатеки и поля трекинга кладутся хуже. → рекомендую media. **вопрос: использует ли проект media library?**

### 9. Единицы веса/размеров → Physical/Measurement (§5.6)

OneCatalog отдаёт вес в **граммах**, размеры в **миллиметрах** (наименьшая единица), с локализованной подписью (`sizes.*_unit`: «мм», «г»). Платформа хранит в своих единицах.

В Drupal Commerce есть **два** конкурирующих способа хранить физические величины:

1. **`physical` (contrib, `commerceguys/physical`)** — модуль `physical` + типы полей `physical_measurement` / `physical_dimensions` (на variation: `weight`, `dimensions`). Хранит значение **вместе с единицей** (`{number, unit}`), есть классы `Weight`, `Length`, `->convert()`. Это правильный таргет: пишем `new Weight($grams, 'g')` и при необходимости конвертируем в нужную единицу. Используется `commerce_shipping`.
2. **Без physical** — кастомные decimal-поля `field_weight`/`field_length`/… + хранение в фиксированной единице магазина (как WooCommerce `woocommerce_weight_unit`). Тогда конвертер пишем сами.

**Рекомендация:** если установлен `physical` — использовать его (`Weight`/`Length` уже умеют конвертацию, единица хранится рядом → §5.3-подобный трекинг единицы решается самим типом поля). Если нет — собственный конвертер `Units` (таблицы коэффициентов к базовой единице + нормализация локализованных подписей «мм/mm», «г/g»), как в эталоне, и фиксированная целевая единица из настройки.

⚠️ Подписи единиц **локализованы** на языке запроса (`lang`): при `lang=ru` придёт «мм»/«г», при `lang=en` — «mm»/«g». Конвертер обязан нормализовать оба написания. → настройка/фильтр единицы источника как фолбэк (мм/г).

### 10. Идемпотентность справочников и медиа — сводно

| Сущность | Ключ поиска | Создание если нет |
|---|---|---|
| product | `field_oc_public_id` (+ фолбэк sku) | create product+variation |
| term категории/бренда/страны/тега/характеристики | `name`/`title` (case-insensitive) в нужном vocabulary | create term |
| коллекция (node/term) | `public_id` коллекции, иначе name | create node/term |
| media | `field_oc_content_key` | download → file → media |
| vocabulary/attribute/field (автосоздание) | machine_name (транслит) | create config entity |

---

## Фоновая очередь, медиа, настройки, i18n, жизненный цикл

### Фоновая очередь (§6)

Drupal даёт **два** штатных механизма, и стандарт допускает оба (Queue API/cron ИЛИ Batch):

**A. Queue API + cron + `@QueueWorker` (для фона, рекомендуется для пакетов из UI).**
- Контроллер принимает список `productPublicIds`, режет на порции по «шагу импорта» (≥10, §6) и кладёт items в очередь:

```php
$queue = \Drupal::queue('onecatalog_import_products');   // DatabaseQueue по умолчанию
foreach (array_chunk($publicIds, $step) as $batch) {
  $queue->createItem(['public_ids' => $batch, 'lang' => $lang]);
}
```

- `QueueWorker`-плагин:

```php
#[QueueWorker(
  id: 'onecatalog_import_products',
  title: new TranslatableMarkup('OneCatalog product import'),
  cron: ['time' => 30],
)]
class ProductImportWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface { ... }
```

- ⚠️ **«Гарантированный фон» — слабое место.** Стандартный cron-runner Drupal запускается **при веб-трафике** (или внешним cron). Без настроенного системного cron / `drush queue:run` очередь может не обрабатываться — это ровно тот же изъян, что WP-Cron в эталоне. Для гарантии — рекомендовать внешний cron (`drush cron` / `drush queue:run onecatalog_import_products`) или contrib `ultimate_cron`. **вопрос: есть ли системный cron / drush?**
- Прогресс/лог (§5.5, §6): хранить состояние пакета (статус pending/running, лог последних N результатов) в `State API` (`\Drupal::state()`) или отдельной таблице; UI **опрашивает** через AJAX-эндпоинт.

**B. Batch API — для синхронного запуска из формы админки** (даёт прогресс-бар «из коробки»), но привязан к HTTP-сессии пользователя (вкладка открыта). Хорош как **синхронный фолбэк** (§6) и для CLI (`drush`).

Рекомендация: основной путь — Queue+cron (фон, переживает закрытие вкладки), Batch — фолбэк/CLI и наглядный прогресс при ручном запуске.

Событие `queue_enqueued` (§8) → диспатч Symfony-события после постановки в очередь.

### Точки расширения → Symfony EventDispatcher + alter-хуки (§8)

WP `do_action`/`apply_filters` → в Drupal раскладывается на:
- **События (реагируют)** — Symfony `EventDispatcher`: класс `OneCatalogEvents` с константами, event-объекты с геттерами/сеттерами:
  - `OneCatalogEvents::BEFORE_IMPORT_PRODUCT` (payload, existing entity)
  - `OneCatalogEvents::PRODUCT_IMPORTED` (product, payload, report) ← **сайт дозаполняет поля тут** (ед.изм., areas_per_package, product_shape, alt, детали бренда, upsells). Полный payload в событии.
  - `COLLECTION_IMPORTED`, `BRAND_IMPORTED`, `COUNTRY_IMPORTED`, `QUEUE_ENQUEUED`.
- **Фильтры (меняют значение)** — в Drupal нет `apply_filters`. Два варианта:
  - **alter-хуки** (`hook_onecatalog_import_request_args_alter()`, `..._payload_alter()`, `..._sku_alter()`, `..._media_dedup_alter()`, `..._languages_alter()` и т.д.) — идиоматичный аналог фильтров.
  - либо **изменяемые события** (event со сеттером значения) — единообразнее с событиями.
  Рекомендую: значения-фильтры (URL, токен, lang, шаг, payload, SKU, цена, порядок размеров, дедуп, флаги, origin) — через **events с сеттерами** (один механизм), плюс при желании alter-хуки для совместимости с привычками Drupal-разработчиков.

Граница «универсальное ↔ проектное» (§8): оформительские поля витрины и неиспользуемый ядром payload — в **отдельном модуле сайта** (subscriber на `PRODUCT_IMPORTED`), ядро о витрине не знает. Это прямой аналог mu-плагина эталона.

### Настройки → Config API (§7)

- `ConfigFormBase` (`onecatalog_import.settings`), schema в `config/schema/onecatalog_import.schema.yml` (обязательно — иначе config не типизируется/не переводится).
- Дефолты в `config/install/onecatalog_import.settings.yml`.
- Поля (§7): токен, язык (из списка `en/ru/ar/zh/kk`), шаг (кламп ≥10), статус новых (publish/unpublished — у Drupal нет pending/draft/private у product «из коробки»; есть только boolean `status` published/unpublished, либо `content_moderation` workflow → см. блокеры), импорт коллекций + «тип объекта+цель», бренд + «тип объекта+цель», страна + «тип+цель», теги вкл/выкл, ручной маппинг характеристик (отдельная форма).
- **Приоритет env над полем токена (§7):** читать сначала `getenv('ONECATALOG_TOKEN')` / `$settings['onecatalog_token']` из `settings.php`, затем config. В Drupal идиоматично — через `Settings::get()` (settings.php) с приоритетом над config.
- Целевые сущности (store, product type, variation type, vocabularies, node types) — тоже настройки, т.к. в Drupal их не предопределить однозначно.

### Мультиязычность / i18n (§9 + сущности)

Две независимые задачи:

1. **UI-строки модуля** (§9): исходник — английский, `t()`/`TranslatableMarkup`, text-domain в Drupal один глобальный. Переводы — `.po`/`.mo` в каталоге `translations/` + указать `interface translation server`/`'project'`, либо положить `onecatalog_import.ru.po`. Drupal сам собирает из `.po` (Locale module). JS-строки — `Drupal.t()` + передавать локализованное из бэка через `drupalSettings` (а не хардкодить).
2. **Мультиязычность импортируемого контента** (важнее и сложнее): OneCatalog отдаёт payload **на языке `lang`**. У Drupal — `content_translation`: одна сущность, переводы по `langcode`.
   - Если нужен один язык — пишем сущности с `langcode` = язык настройки.
   - Если нужно несколько языков товара — надо тянуть payload на каждом `lang` и **добавлять перевод** к той же сущности (`$product->addTranslation($langcode, $fields)`), сохраняя `public_id` как ключ. Это многократно усложняет импорт (N запросов на товар, перевод термов таксономии — у термов свои переводы). Стандарт этот сценарий **не специфицирует** → **вопрос про мультиязычность**.
   - Термины/категории/бренды тоже переводимы — если включить, find-or-create должен матчить по name **в нужном langcode**.

### Жизненный цикл (§10)

- `hook_install()`: создать словари (categories, brands, countries, tags, спец-словари по необходимости), node type/таксономию для коллекций, поля (`field_oc_public_id` как base field — через `hook_entity_base_field_info`, остальные — config fields/`FieldStorageConfig`+`FieldConfig`), media bundle поля трекинга. Идемпотентно (проверять существование).
- `hook_uninstall()`: по политике — удалять созданные config-сущности/поля (с осторожностью к данным).
- `hook_update_N()`: миграции схемы (перенос ключа из sku в `field_oc_public_id`, добавление полей дедупа/качества медиа, флаг «миграция выполнена» в State). Update hooks **идемпотентны** и нумеруются.
- Манифест `.info.yml`: version, `core_version_requirement`, dependencies (commerce, taxonomy, media, опц. physical). Активация **не должна падать без Commerce** — но в Drupal зависимость от `commerce_product` в `.info.yml` просто **не даст включить** модуль без Commerce (жёстче WP, где «активация не падает без движка» — у нас это гарантируется системой зависимостей; отдельная проверка-деградация не нужна, но функции, дёргающие Commerce-сервисы, всё равно стоит защищать на случай удаления зависимости).
- SemVer + CHANGELOG (Keep a Changelog) — как в стандарте; версия в `.info.yml` (+ composer.json если поставляется через composer).

### Права (§7, чек-лист)

- Объявить permissions в `onecatalog_import.permissions.yml`: `administer onecatalog import` (настройки), `import onecatalog products` (запуск импорта/picker).
- Маршрут контроллера приёма ID — `_permission: 'import onecatalog products'` + **CSRF**: для POST-маршрута добавить `_csrf_token: 'TRUE'` (Drupal-аналог nonce; токен прокидывать в JS через `drupalSettings`/`Url::fromRoute(...,['token'=>...])` или `\Drupal::csrfToken()`).
- Picker/iframe эндпоинты — те же права + проверка `event.origin` на фронте.

### Picker (iframe + postMessage, §2.4)

- Library `picker-loader` (JS), встраивается на форму товара через `hook_form_alter` + `#attached`. Кнопка открывает iframe `https://tools.onecatalog.net/picker.html?token=…&parentOrigin=…`.
- Токен и origin виджета прокидывать через `drupalSettings` (origin задавать **явно**, не `window.location.origin`; проверять `event.origin`).
- ⚠️ Drupal admin-формы могут жить под разными темами (Claro/Gin) и под `/admin/*` с CSP. iframe-sandbox `allow-scripts allow-forms allow-same-origin` + возможный CSP-заголовок ядра/Seckit могут блокировать сторонний iframe → нужно проверить/ослабить CSP для конкретной admin-страницы. **вопрос про CSP.**
- Импорт по `productPublicIds`, не по числовым id.

---

## ❓ Вопросы и сомнения (ответить ДО старта)

1. **Пустой `article` vs обязательный SKU Commerce.** В Commerce `commerce_product_variation.sku` **обязателен и уникален**, пустым быть не может. Стандарт (§5.2) требует «если `article` пуст — оставлять SKU пустым». Это **несовместимо**. Какой фолбэк допустим: sku = `public_id` (тогда «артикул» в карточке будет техническим, не артикулом производителя)? Или генерируемый `OC-<n>`? Нужна явная политика для Drupal.

2. **Цена не задаётся (§5.6) vs обязательное `price` у variation.** Стандартный `commerce_product_variation` имеет required `price` (и без store/price товар не покупаем). Варианты: (а) снять required с поля price на нашем variation type; (б) использовать «безценовой» variation type; (в) ставить плейсхолдер `0`/null. Что выбираем? (От этого зависит схема variation type.)

3. **Привязка к `commerce_store`.** Commerce требует product↔store. Какой store(ы) назначать импортируемым товарам по умолчанию (default store? настройка списком)? Без этого товары не отображаются в каталоге.

4. **Характеристики: attribute vs taxonomy vs field — что дефолт?** Я предлагаю `taxonomy` по умолчанию (ближе к «фасет-термин» эталона; Commerce attribute семантически для вариаций, а у нас одна variation). Подтвердить, что **не** нужны реальные покупательские вариации (разные цвета/размеры → разные SKU). Если в будущем один `public_id` сможет давать несколько вариаций — модель меняется кардинально.

5. **Статусы новых товаров.** Стандарт (§7) перечисляет `publish/pending/draft/private`. У Drupal `commerce_product` «из коробки» — только boolean `status` (published/unpublished). `pending/draft/private` существуют лишь при включённом `content_moderation` (workflow) и не у каждого сайта. Маппим `publish→published`, всё остальное→`unpublished`? Или интегрируемся с `content_moderation` (если есть)?

6. **Physical/Measurement модуль (§5.6).** Установлен ли contrib `physical` (`commerceguys/physical`)? Если да — пишем `Weight/Length` с конвертацией «из коробки»; если нет — кастомные decimal-поля + собственный конвертер и фиксированная целевая единица. Какая целевая единица веса/длины магазина (кг/г, см/мм)?

7. **СУБД проекта (MySQL/MariaDB vs PostgreSQL).** От collation зависит регистронезависимый поиск термов по name (§5.1). MySQL utf8mb4_*_ci — ок; PostgreSQL — case-sensitive по умолчанию → нужен `LOWER()`/функциональный индекс или нормализация. Какая СУБД?

8. **Гарантированность фона.** Настроен ли системный cron / доступен ли `drush` (`drush cron`, `drush queue:run`)? Без внешнего cron штатный Drupal-cron зависит от веб-трафика — та же ненадёжность, что WP-Cron. Это влияет на выбор: Queue+cron как основной путь или Batch/степпер.

9. **Мультиязычность контента.** Нужны ли товары/термины/коллекции **на нескольких языках** (через `content_translation` + перевод по `langcode`), или достаточно одного языка `lang` из настройки? Многоязычный импорт = N запросов на товар + переводы термов — кратно дороже; стандарт это не специфицирует.

10. **Media entity vs прямой file/image.** Использует ли проект Media Library? Кладём изображения как `media` (bundle image, с полями трекинга качества/контент-ключа) или проще — `file` + image-поле на product? Рекомендую media; нужны поля `field_oc_content_key`, `field_oc_size` на bundle.

11. **Бренд/коллекция как node или term?** Дефолты стандарта: бренд→taxonomy (в WooCommerce есть нативная `product_brand` — в Drupal её НЕТ, создаём сами); коллекция→CPT (в Drupal = node). Подтвердить дефолты: бренд → vocabulary `brands`, коллекция → node `oc_collection`? Богатые поля бренда/коллекции (year/website/links/images) на term требуют доп. полей на vocabulary — ок?

12. **CSP / iframe-пикер в Drupal admin.** Есть ли на сайте CSP (ядро Drupal 10.1+ умеет CSP для admin, либо contrib `seckit`/`csp`)? iframe на `tools.onecatalog.net` может быть заблокирован `frame-src`. Нужно ослабить политику для страницы пикера.

13. **Лимиты/пагинация источника (§2.1).** Подтвердить практический «высокий limit» для справочников (`/specifications/` и др.), чтобы не словить молчаливое усечение, и реальный rate-limit OneCatalog (для размера порции очереди и пауз между запросами). Стандарт не даёт чисел.

14. **`public_id` как base field vs config field.** Предлагаю base field (`hook_entity_base_field_info` для `commerce_product`) ради индексации и переносимости между bundle. Возражений нет? (config-field проще удалять, но медленнее в query и привязан к bundle.)

15. **Удаление при апгрейде качества + дедуп.** При замене файла на лучший размер (§5.3) удаляем старый `file`/`media`, НО только если он **не общий** (контент-ключ не используется другими product). Подтвердить: проверяем usage (`file_usage`/обратные ссылки entityQuery) перед удалением. Политика для media: удалять media или только перецеплять file?

16. **Несколько вариаций / upsells / cross_sells.** `upsells[]`/`cross_sells[]` ядро не маппит (отдаём в событие). Подтвердить, что это задача сайтового слоя (как в эталоне), а не ядра.

17. **Уникальность `public_id`.** Гарантируем ли один product на `public_id` (уникальный индекс/проверка в коде)? Drupal не навешивает unique на произвольное поле сам — нужна проверка в импортёре (range(0,1)) + опц. constraint.

---

## 🚫 Сложно / невозможно на Drupal Commerce (как написано в стандарте)

1. **«Товар = одна сущность с SKU» — неверно для Commerce.** Стандарт и эталон (WooCommerce) мыслят товар как ОДНУ сущность, где SKU/вес/цена/идемпотентность на одном объекте. В Commerce это **две** сущности: `commerce_product` (карточка) + `commerce_product_variation` (SKU/цена/вес). 
   - *Последствие:* маппинг раздваивается — `public_id` логично на product, а `article`/sku/sizes — на variation. Каждый «единичный» товар OneCatalog → product+variation.
   - *Обход:* модель «single-variation product»; оркестратор создаёт обе сущности атомарно; `public_id` — на product (идемпотентность), sku/sizes — на variation.

2. **Обязательный SKU + обязательная цена + обязательный store — против §5.2/§5.6.** 
   - SKU не может быть пустым (§5.2 требует «пусто, если нет article»). Цена required по умолчанию (§5.6 требует «не задавать»). Product без store не покупаем/не виден.
   - *Обход:* отдельный variation type с **необязательной ценой** (снять required на field price) и фолбэк-SKU из `public_id`; назначать default store настройкой. Это компромисс — «чистое незадание цены» как в стандарте не выходит без правки схемы поля.

3. **«Глобальный атрибут `pa_*` + термин» не имеет прямого 1:1 аналога.** В Drupal два разных механизма: Commerce **attribute** (для генерации покупательских вариаций) и **taxonomy** (для фасетов/описаний). Стандарт подразумевает «атрибут как фасет» (WooCommerce так и используется), что в Drupal честнее = taxonomy, а **не** Commerce attribute. Использовать Commerce attribute для описательной «толщины 9 мм» при одной вариации — abuse механизма (плодит пустые комбинации, привязка к variation, а не product).
   - *Обход:* по умолчанию характеристики → taxonomy + entity_reference поле; режим attribute оставить опцией для случаев реальных вариаций. Дать настройку режима (`attribute|taxonomy|field`).

4. **Статусы `pending/draft/private` (§7) у Commerce product отсутствуют.** Есть только boolean published/unpublished. `pending/draft/private` — это WordPress-понятия; в Drupal аналог только через `content_moderation` workflow, которого может не быть.
   - *Обход:* маппинг `publish→published`, остальное→unpublished; опциональная интеграция с content_moderation, если на сайте есть workflow для product.

5. **Гарантированный фон не гарантирован (как и WP-Cron).** Drupal cron-runner по умолчанию пинается веб-трафиком; без системного cron/`drush` очередь может простаивать. Это **тот же** изъян, что в эталоне, а не преимущество.
   - *Обход:* рекомендовать внешний cron (`drush cron`/`drush queue:run`) или `ultimate_cron`; Batch API как синхронный фолбэк (но он держит HTTP-сессию пользователя — вкладку нельзя закрывать).

6. **`apply_filters` нет.** «Фильтры» (§8) — чисто WP-механизм синхронной модификации значения. В Drupal нет прямого аналога одного вызова `apply_filters` где угодно.
   - *Обход:* фильтры → **изменяемые Symfony-события** (event со сеттером) и/или `hook_*_alter()`. Полная функциональная эквивалентность есть, но это другой код-стиль и иной порядок вызова подписчиков (приоритеты подписчиков вместо приоритетов фильтров).

7. **«Нативная таксономия брендов» (дефолт §7) в Drupal отсутствует.** WooCommerce 8+ имеет встроенный `product_brand`; Drupal Commerce — нет.
   - *Обход:* модуль сам создаёт vocabulary `brands` в hook_install. Не блокер, но «по умолчанию — нативная таксономия брендов» из стандарта в Drupal неприменимо дословно.

8. **Иерархия категорий / порядок не передаётся, а Drupal term без slug.** У payload `categories[]` плоские (нет parent). У термов Drupal нет slug/machine-name (только tid+name) — это **снимает** проблему транслитерации §5.1 на уровне термов (ищем по name), НО переносит её на автосоздаваемые **machine-name словарей/полей/атрибутов** (там нужен `transliteration` сервис и свои коллизии). Не блокер, но место расхождения с эталоном.

9. **Локализованные единицы измерения + мультиязычные термы.** §5.6 требует нормализовать локализованные подписи («мм/mm», «г/g»), а §9/мультиязычность сущностей требует переводы термов по `langcode`. Если включить многоязычный импорт, find-or-create термов должен матчить по name **в конкретном langcode**, а единицы приходят на языке `lang` — двойная локализационная сложность, которую стандарт не разводит явно. 
   - *Обход:* конвертер нормализует обе раскладки подписей; многоязычный импорт термов — отдельный режим (по умолчанию один язык), иначе резко растёт стоимость.

10. **Picker-iframe под Drupal CSP.** Drupal 10.1+/contrib CSP может блокировать сторонний `frame-src tools.onecatalog.net`. На admin-страницах под Gin/Claro + строгий CSP iframe не откроется.
    - *Обход:* добавить `tools.onecatalog.net` в `frame-src` для конкретной admin-страницы (alter заголовков/настройка seckit/csp). Не блокер, но требует конфигурации окружения.

---

### Итог по приоритетам реализации

Сначала закрыть конфликты модели (вопросы 1–3, 6): variation type без required-цены, политика SKU, default store, наличие physical. Это фундамент — без решения не построить даже «импорт одного товара» (§порядок прототипирования, шаг 1). Далее — идемпотентность по base field `public_id`, find-or-create термов по name, медиа с дедупом, очередь Queue+cron с Batch-фолбэком, события вместо фильтров.
