# Интеграционный план: PrestaShop

> Реализация модуля импорта OneCatalog Wiki API на **PrestaShop 1.7 / 8.x** (Symfony-модуль).
> База — стандарт `INTEGRATION-STANDARD.md`; эталон — WooCommerce-плагин `onecatalog-import`.
> Внешние контракты (§2) и инварианты (§5) стандарта неизменны; меняется только слой «как это лечь в PrestaShop».
>
> Ключевые архитектурные конфликты с PrestaShop сразу выношу наверх, чтобы они не потерялись:
> 1. **Характеристики OneCatalog — это `Feature`/`FeatureValue` (описательные), НЕ `Attribute`/`AttributeGroup` (вариации/комбинации).** Эталон кладёт их в глобальные атрибуты `pa_*`, но в WooCommerce атрибут без `variation=true` — это именно описательная характеристика. В PrestaShop прямой аналог «невариативной характеристики» — `Feature`, а не `Attribute`. Использование `Attribute` потянуло бы за собой `Combination`/`StockAvailable` на каждую — это неверно (см. §«Сложно / невозможно»).
> 2. **Мультиязычность.** Почти все человекочитаемые поля PrestaShop (`name`, `description`, `Feature::name`, `FeatureValue::value`, `Category::name`, `Manufacturer` имя через `meta`/`description`, `link_rewrite`, `tag`) — это lang-массивы `[$id_lang => value]`. Любая запись и любой `find-or-create` обязаны нести `id_lang`. Стандарт писался под WP, где язык — один параметр API, а не размерность хранилища.
> 3. **Multistore.** `Product`, `Category`, `Feature`/`FeatureValue` (частично), `Manufacturer`, `Image`, `Configuration`, `tag` имеют shop-контекст. Импортёр должен работать в явном `Shop::setContext` и знать целевые `id_shop`.
> 4. **«Коллекция как post type / таксономия» из стандарта в PrestaShop НЕ существует** — нет произвольных типов сущностей и нет «таксономий». Замена: категория / Feature / своя ObjectModel-таблица.

---

## Маппинг сущностей и реализация

### Сводная таблица соответствий (детализация §3 стандарта под PrestaShop)

| OneCatalog payload | Абстракция | PrestaShop-реализация | Идемпотентность |
|---|---|---|---|
| `product` | Товар | `Product` (ObjectModel) | по `public_id` (см. ниже) |
| `public_id` (`OC.IND.1`) | Внешний стабильный ключ | **отдельное поле в своей таблице** `oc_product_link(public_id, id_product, id_shop)` ИЛИ `Product::reference`; рекомендуется своя таблица (reference не уникален и его трогает витрина) | `find_by_public_id()` |
| `article` (артикул произв., часто пуст) | Артикул/SKU | `Product::reference` (и/или `supplier_reference`), только если пришёл; коллизию не перезаписывать | — |
| `menutitle` | Название | `Product::name` (**lang-массив**) | — |
| `description_text` | Описание | `Product::description` / `description_short` (lang) | — |
| `in_stock` (bool) | Наличие | `StockAvailable` (`out_of_stock`, `quantity`) или `Product::quantity` без advanced stock; только индикатор наличия, не реальные остатки | при создании |
| `status_publish` / настройка | Статус | `Product::active` (1/0) — **только при создании** | при создании |
| `sizes.weight` (граммы) | Вес | `Product::weight` (ед. — глобальная `PS_WEIGHT_UNIT`) через конвертер `Units` | — |
| `sizes.length/width/height` (мм) | Габариты | `Product::depth`/`width`/`height` (ед. — глобальная `PS_DIMENSION_UNIT`) через `Units` | — |
| `options[]` (`text`/`boolean`/`numeric`) | Характеристика | **`Feature` + `FeatureValue`**; привязка через `Product::addFeaturesToDB` / `id_feature_value`. Ручной маппинг — по `specification_id` → `id_feature` | Feature по имени (lang), FeatureValue по `(id_feature, value)` |
| `categories[]` | Категория | `Category` (дерево, lang `name`+`link_rewrite`), `Product->addToCategories()` | по имени в пределах родителя |
| `brand` | Бренд | **`Manufacturer`** (по умолчанию) ИЛИ `Feature`/своя таблица (настройка) | `Manufacturer::name` (не lang) |
| `country` | Страна происх. | `Feature` «Страна» (по умолчанию) ИЛИ своя таблица; **не** `Country` (это справочник доставки/налогов) | Feature/FeatureValue |
| `collections[]` | Коллекция | `Category` (спец-ветка) **или** `Feature` **или** своя ObjectModel-сущность (настройка); полей коллекции (link_3d, link_official_site, обложка) — своя таблица/мета | по имени/slug |
| `tags[]` | Теги | `Tag` (`Tag::addTags`, lang) — вкл/выкл, поиск по имени | по имени (lang) |
| `images_urls` (обложка) + `files[]` (галерея) | Медиа | `Image` + `ImageManager` (cover + галерея, трекинг качества, дедуп по контент-ключу) | по контент-ключу + имени файла |
| `measurement_unit`, `areas_per_package`, `product_shape`, alt картинок, детали бренда, upsells/cross_sells | Прочее (вне ядра) | через хук `actionOneCatalogProductImported` (сайтовый слой) | — |

### Слои модуля (перенос §4)

Symfony-модуль PrestaShop. Имя, напр., `onecatalogimport`. Структура:

```
onecatalogimport/
  oneocatalogimport.php           — Module: install/uninstall/upgrade, регистрация хуков, tabs, Configuration-дефолты
  src/
    Api/Client.php                — base URL, token, lang, GET, обёртка {data,success}, обработка ошибок → null
    Service/Units.php             — конвертер вес(г)/размеры(мм) → PS_WEIGHT_UNIT/PS_DIMENSION_UNIT (порт class-units.php 1:1)
    Service/Media.php             — выбор размера, скачивание+MIME, cover/галерея, трекинг качества, дедуп по контент-ключу
    Service/Taxonomies.php        — Feature/FeatureValue (поиск по имени + ручной маппинг), Category, Tag
    Importer/CollectionImporter.php
    Importer/BrandImporter.php
    Importer/CountryImporter.php
    Importer/ProductImporter.php  — оркестратор (порядок §5.4)
    Queue/ImportQueue.php         — Messenger ИЛИ cron-контроллер ИЛИ AJAX-степпер (фолбэк)
    Repository/ProductLinkRepository.php  — oc_product_link, дедуп-реестр медиа
  controllers/admin/AdminOneCatalogImportController.php   — приём списка ID + статус (CSRF-токен PS)
  controllers/front/queue.php     — (опц.) фронт-контроллер cron-эндпоинта по секрету
  views/                          — picker-loader.js, admin-import.js, twig настроек
  translations/ или langs         — trans() домен Modules.Oneocatalogimport.*
  sql/install.php, upgrade/       — миграции таблиц
```

Зависимости направлены «вниз» (как в эталоне): `ProductImporter` оркестрирует `Api/Media/Taxonomies/*Importer`; `Queue` → `ProductImporter`; контроллер → `Queue`. Сервисы регистрируются в `config/services.yml` модуля (autowiring), либо собираются вручную в legacy-стиле, если модуль таргетит 1.7.6+ без полной DI.

### Идемпотентность и внешний ключ (§5.1, §5.2)

- **`public_id` → своя таблица** `oc_product_link (id_product INT, public_id VARCHAR(64), id_shop INT, PRIMARY KEY(public_id, id_shop))`. Причина не класть в `reference`: (а) `reference` — это «артикул витрины», им управляет вебмастер и стандарт требует туда писать `article`; (б) `reference` не обязан быть уникальным, поиск по нему ненадёжен; (в) multishop — один товар может присутствовать в нескольких магазинах.
  `find_by_public_id($public_id, $id_shop)` → `id_product` или 0.
  Легаси-фолбэк (как в эталоне SKU→meta): если в `oc_product_link` пусто — искать `Product` по `reference == public_id` и мигрировать запись в таблицу.
- **SKU-политика:** `Product::reference = article` только при непустом `article` и отсутствии коллизии (`Product::getIdByReference`-аналог через `DbQuery`); `public_id` в `reference` не писать.
- **Справочники find-or-create по ИМЕНИ, не по слагу** (§5.1): критично, т.к. `link_rewrite`/slug PrestaShop генерит `Tools::link_rewrite()` с собственной транслитерацией → разные слаги для одного имени → дубли. Матчить:
  - `Feature` — по `feature_lang.name` (без учёта регистра) для нужного `id_lang`;
  - `FeatureValue` — по `(id_feature, feature_value_lang.value)`;
  - `Category` — по `category_lang.name` в пределах `id_parent` (имя категории не глобально-уникально);
  - `Manufacturer` — по `manufacturer.name` (не мультиязычное, проще);
  - `Tag` — `Tag::getMainTags`/прямой SELECT по `(name, id_lang)`.

### Характеристики → Feature/FeatureValue (§5.3, §5.4) — главное расхождение с эталоном

OneCatalog `options[]` — это **описательные характеристики**, не вариации. Маппинг:

- `text` (`specification_option_name`): `Feature(label)` + `FeatureValue(specification_option_name)`.
- `boolean` (значение в `bool_option`, имя `null`): по настроенному в маппинге значению терма для true/false (порт `Settings::bool_value($spec_id, $bool)`); пусто → характеристика пропускается. По умолчанию `null/false → false` (§5.6).
- `numeric` (значение в `numeric_option`): `FeatureValue` со строковым представлением числа (PS хранит значение строкой; «кастомные» feature value по товару тоже есть, но создавать общий FeatureValue — ближе к эталону и переиспользуемо).

Реализация привязки: собрать `[id_feature => [id_feature_value, ...]]`, затем `Product::addFeaturesToDB($id_feature, $id_feature_value)` на сохранённый товар (или прямые вставки в `product_feature`). **Один товар × один FeatureValue на feature** — стандартное ограничение PrestaShop UI; если у товара по одной характеристике несколько значений (множественный `options[]` с одним `specification_id`) — придётся либо склеивать в одно значение, либо использовать несколько кастомных feature-value (см. блокеры).

Ручной маппинг (строгий режим, §5.4): таблица `oc_feature_map (specification_id INT, id_feature INT)`. Резолв (порт `Taxonomies::resolve_attribute`):
- маппинг ВЫКЛ → Feature по имени/создать;
- маппинг ВКЛ → `specification_id → id_feature` из карты; нет привязки или Feature удалён → характеристика **пропускается**;
- для `boolean` в карте задаются значения термов true/false (как у эталона).

### Категории, бренд, страна, коллекции, теги

- **Категории:** `Category` иерархичны. OneCatalog отдаёт `categories[]` плоско (`menutitle`, `slug`), родителя нет. Решение: создавать как детей корневой «OneCatalog» категории (или корня магазина) — find-or-create по имени в пределах этого родителя. `link_rewrite` (lang, обязателен и должен быть валиден) генерить `Tools::link_rewrite($name, $id_lang)`. `Product->addToCategories([...])`, `id_category_default` — первая.
- **Бренд → `Manufacturer`** (по умолчанию; настройка «тип объекта + цель» позволяет вместо этого Feature/свою таблицу). `Manufacturer::name` не мультиязычен — это упрощает поиск. Поля бренда (страна, год, сайт, описание) — `Manufacturer::description`/`meta` (lang) или своя таблица; их заполняет сайт через хук, ядро ставит только связь `Product::id_manufacturer`.
- **Страна → `Feature` «Страна происхождения»** (по умолчанию) или своя таблица. **Не использовать `Country`** — это таблица справочника зон доставки/налогов, у неё ISO-коды и иной смысл; писать туда «страну производителя» сломает чекаут.
- **Коллекции → настройка «тип объекта + цель»** (порт `CollectionImporter`):
  - `Category` (спец-ветка под корнем «Коллекции») — товар добавляется в категорию;
  - `Feature` «Коллекция» + FeatureValue;
  - своя ObjectModel-сущность `oc_collection` (рекомендуется, если коллекции — отдельные страницы витрины). Поля `link_3d`, `link_official_site`, обложку хранить в этой таблице/в `oc_collection_lang`.
  «post type» из стандарта = эта своя сущность; «taxonomy» = Category/Feature.
- **Теги → `Tag`** (вкл/выкл). `Tag::addTags($id_lang, $id_product, [names])` или ручной find-or-create по `(name, id_lang)`; поиск по имени.

### Конвертер единиц (§5.6) — переносится почти 1:1

`Service/Units.php` — порт `class-units.php`: таблицы коэффициентов к базовой (грамм / миллиметр), нормализация локализованных подписей (`мм`,`г`,…). Отличие только в «единице магазина»:
- вес: `Configuration::get('PS_WEIGHT_UNIT')` (kg/g/lb/oz…);
- размеры: `Configuration::get('PS_DIMENSION_UNIT')` (cm/m/mm/in…).
Источник — `sizes.*_unit` из payload (нормализуется), иначе мм/г; переопределяемо хуком-фильтром. **Важно:** в PrestaShop единицы веса и размеров — глобальные на магазин (нет «единицы на товар»), как и в Woo — так что инвариант выполним.

---

## Фоновая очередь, медиа, настройки, i18n, жизненный цикл

### Медиа (§2.3, §5.3)

- **Скачивание вручную + MIME по содержимому:** URL `media_files/<hash>` без расширения. `cURL`/`Tools::file_get_contents` во временный файл, MIME через `finfo`/`getimagesize`, расширение — по MIME. `ImageManager::isRealImage`/`isCorrectImageFileExt` штатно валидируют по расширению/MIME — подставлять корректное имя `<basename>.<ext>`.
- **Cover + галерея через `Image`:** `Image` (ObjectModel) с `cover=1` для обложки; `ImageManager::resize` генерит форматы (`image_type`). Привязка к товару — `Image::id_product`, ассоциация магазина `image_shop`. Watermark-хук по желанию.
- **Трекинг качества (`min|middle|max`):** своя таблица `oc_image (id_image, id_product, content_key, size_rank, file_name)`. При переимпорте: если доступен размер лучше записанного — перекачать, заменить файл, удалить старый `Image` (кроме «общих» — см. дедуп). Реализация рангов — порт `Media::size_rank`.
- **Дедуп по контент-ключу:** контент-ключ из base64-префикса `media_files` (`sha1(path#size)`) — порт `Media::file_key`. Реестр `oc_image.content_key`: перед скачиванием искать существующий `Image` по ключу → переиспользовать (привязать тот же файл к товару второй ассоциацией, **не** копируя). ⚠️ Тонкость PrestaShop: `Image` физически принадлежит одному `id_product` (структура каталога `img/p/<разбивка id_image>`). «Общий файл на много товаров» так напрямую не выразить — варианты: (а) дедуп только в пределах товара/коллекции; (б) копировать файл, но не качать повторно (экономим сеть, не диск); (в) своя таблица многие-ко-многим + кастомный рендер. Рекомендую (б): ключ предотвращает повторное **скачивание**, физический файл копируется. Общие файлы при апгрейде качества не удалять.
- Галерея идемпотентна **по имени файла** (`files[].name`), не по URL (подписи в URL меняются).

### Фоновая очередь (§6)

Три варианта (стандарт допускает любой; фолбэк — синхронно):

1. **AJAX-степпер (рекомендация по умолчанию, аналог нативного импорта PS).** Список ID режется на порции по «шагу» (≥10); JS из админ-контроллера шлёт порции последовательно, контроллер импортирует порцию и возвращает лог. Не требует cron/Messenger, работает на shared-хостинге. Прогресс — естественный (поллинг = сами шаги).
2. **Cron-контроллер.** Фронт-контроллер `controllers/front/queue.php?token=<secret>&task=run`, дергается системным cron или модулем **PS Cron Tasks** (`cronjobs`). Очередь порций — в таблице `oc_queue (id, payload JSON, status)`. Подходит для больших объёмов без участия браузера.
3. **Symfony Messenger** (PS 1.7.6+/8 имеет Messenger). `ImportBatchMessage(ids[])` → handler → `ProductImporter`. Транспорт по умолчанию — Doctrine/`sync`; для реального async нужен запущенный воркер (`bin/console messenger:consume`), что на типичном хостинге не гарантировано → Messenger как опция для «правильных» серверов, не как дефолт.

Во всех вариантах: лог последних N результатов (своя таблица/`Configuration` JSON), статус pending/running, синхронный фолбэк когда нет планировщика. Шаг — кламп [10..100], порт `Queue::step`.

### Настройки (§7) → `Configuration`

| Стандарт | Ключ Configuration | Тип/валидация |
|---|---|---|
| Токен API | `ONECATALOG_TOKEN` | строка; **env приоритетнее** (getenv) |
| Язык товаров | `ONECATALOG_LANG` | enum из API (`en/ru/ar/zh/kk`) |
| Шаг импорта | `ONECATALOG_STEP` | int, кламп ≥10 |
| Статус новых | `ONECATALOG_NEW_ACTIVE` | 1/0 → `Product::active` (только при создании) |
| Импорт коллекций + тип+цель | `ONECATALOG_COLL_*` | вкл; `category\|feature\|object` + id цели |
| Импорт бренда + тип+цель | `ONECATALOG_BRAND_*` | вкл; `manufacturer\|feature\|object` + цель |
| Импорт страны + тип+цель | `ONECATALOG_COUNTRY_*` | вкл; `feature\|object` + цель |
| Импорт тегов | `ONECATALOG_TAGS` | вкл/выкл |
| Ручной маппинг характеристик | `ONECATALOG_ATTR_MAP` + таблица `oc_feature_map` | вкл/выкл; страница маппинга spec→Feature, true/false-значения |

`Configuration` поддерживает multishop (`updateValue(..., $id_shop_group, $id_shop)`) и lang (`updateValue([$id_lang=>...])`) — для глобальных скаляров достаточно обычной записи. UI настроек — Twig + Symfony Form (8.x) или `HelperForm`/`renderForm` (1.7 legacy). UI скрывает нерелевантные поля цели по выбранному типу (как в эталоне).

### i18n (§9)

- Исходные строки — **английские**, домен трансляции модуля `Modules.Oneocatalogimport.Admin` (и `.Picker` для фронт-скриптов).
- В PS 1.7/8 — новая система переводов (`$this->trans('...', [], 'Modules.Oneocatalogimport.Admin')`), переводимая через UI «Перевод → Переводы модуля». Старый стиль (`{l s='' mod=''}` в tpl, `translations/ru.php`) — фолбэк для legacy-шаблонов.
- **Строки JS-скриптов (picker/прогресс) передавать из бэкенда локализованными** (`Media::addJsDef`/`Context::getContext()->smarty->assign`), не хардкодить в JS — прямой аналог требования стандарта.
- Реальный целевой язык витрины — `ru_RU`; учесть, что строки в `.po`/UI-переводах должны быть собраны (аналогично инварианту проекта про пересборку `.mo`).

### Жизненный цикл (§10)

- **Манифест:** в `oneocatalogimport.php` — `$this->version`, `$this->ps_versions_compliancy = ['min' => '1.7.6.0', 'max' => '8.99.99']`, `$this->dependencies` (не зависит от внешних модулей; бренды — нативны). SemVer + `CHANGELOG.md` (Keep a Changelog), версия в `version` и в `config.xml` (PS дублирует версию туда) — поднимать перед сборкой zip (аналог инварианта проекта про 2 места версии).
- **install():** создать таблицы (`oc_product_link`, `oc_collection(+lang)`, `oc_image`, `oc_feature_map`, `oc_queue`, `oc_log`), зарегистрировать хуки (`displayAdminProductsExtra` или кнопка в списке товаров, `actionXxx` для точек расширения), создать `Tab`/контроллер в меню «Каталог», дефолты `Configuration`. **Не падать без движка каталога:** PrestaShop сам и есть магазин (в отличие от WP+Woo), но проверить наличие нужных классов/версии.
- **upgrade-NNN.php:** миграции схемы (`sql/upgrade/`), перенос старых ключей, флаг «миграция выполнена». Легаси-фолбэк поиска по `reference` (§5.1).
- **uninstall():** снять хуки, удалить `Tab`/`Configuration`; **данные таблиц/товары не удалять** по умолчанию (или спросить) — иначе теряется каталог.

### Права и защита (§11)

- Админ-контроллер наследует `ModuleAdminController`/`FrameworkBundleAdminController`; доступ — по профилю/правам `Tab` (PrestaShop ACL), а не «manage_woocommerce».
- **CSRF:** PrestaShop-токен (`$this->getAdminToken`/Symfony CSRF), аналог nonce. AJAX-эндпоинты проверяют токен.
- Cron-фронт-контроллер — по секретному токену в URL (не по сессии).
- Origin виджета (picker) задавать явно, проверять `event.origin` (как в эталоне).

### Точки расширения (§8) → хуки PrestaShop

Эталонные `do_action`/`apply_filters` → PrestaShop `Hook::exec` (события) и собственные «filter»-хуки (PrestaShop хуки могут возвращать/менять значение через передачу по ссылке в `$params`):

| Стандарт | PrestaShop-хук |
|---|---|
| `before_import_product(payload, existing_id)` | `actionOneCatalogBeforeImportProduct` |
| `product_imported(id, payload, report)` ← **сайт дозаполняет** | `actionOneCatalogProductImported` |
| `collection_imported` / `brand_imported` / `country_imported` | `actionOneCatalog{Collection,Brand,Country}Imported` |
| `queue_enqueued(ids, batches, step)` | `actionOneCatalogQueueEnqueued` |
| фильтры: base URL, token, lang, payload, SKU, цена, порядок размеров, дедуп, файлы галереи, справочник характеристик, флаги, origin | `filterOneCatalog*` (значение в `$params` по ссылке) |

Сайтовый слой (texture/ед.изм./areas_per_package/product_shape/alt/upsells) живёт в **отдельном модуле-наблюдателе** на `actionOneCatalogProductImported` — ядро о витрине не знает (граница «универсальное ↔ проектное»). Цена по умолчанию не задаётся (`Product::price` не трогаем при создании — её нет в источнике; §5.6); подстановка — через `filterOneCatalogProductPrice`.

---

## ❓ Вопросы и сомнения (ответить ДО старта)

1. **Целевая версия PrestaShop и режим модуля.** 1.7.6 (legacy HelperForm/ObjectModel) или 8.x (Symfony Form, Doctrine, autowiring)? Это решает форму контроллеров настроек, систему переводов и доступность Messenger. Какой минимальный `ps_versions_compliancy`?
2. **Характеристики: `Feature` или всё-таки `Attribute`?** Подтвердить, что `options[]` OneCatalog — описательные (Feature), а не вариации товара (Attribute/Combination). Если хоть какая-то характеристика должна порождать варианты/цену/остаток — это `Attribute`, и тогда нужен отдельный маппинг + создание `Combination`. По эталону (`variation=false`) — это Feature. Подтверждаете?
3. **Хранилище `public_id`.** Своя таблица `oc_product_link` (рекомендую) или `Product::reference`? Если `reference` уже используется витриной/1С-обменом — точно своя таблица. Есть ли уже внешняя интеграция (1С), которая претендует на `reference`/`supplier_reference`?
4. **Multistore.** Магазин один или мультимагазин? В какие `id_shop` импортировать (все / выбранный)? Должны ли товар/категории/feature быть shared между магазинами или раздельны? Это влияет на каждый `save()` и на ключ идемпотентности.
5. **Мультиязычность импорта.** API отдаёт один язык за запрос (`lang`). Нужно ли заполнять несколько языков PrestaShop (несколько проходов API по каждому `id_lang`) или только один целевой (`ru`)? Что делать с пустыми lang-полями для прочих языков (PrestaShop часто требует непустой `name`/`link_rewrite` хотя бы для дефолтного языка)? Какой `id_lang` — дефолтный?
6. **Численные характеристики (`numeric_option`).** Класть как обычный `FeatureValue` (строкой) или использовать значение/единицу отдельно? Нужны ли единицы измерения у численных характеристик в витрине?
7. **Множественные значения одной характеристики.** Может ли у товара быть несколько `options[]` с одним `specification_id` (несколько значений одной характеристики)? PrestaShop UI допускает одно `FeatureValue` на `Feature` на товар. Если да — склеивать в одно значение, или использовать кастомные feature-value, или несколько Feature?
8. **Коллекция — какая цель по умолчанию?** Своя сущность-страница (`oc_collection`), `Category`-ветка или `Feature`? Есть ли уже на витрине понятие «коллекция» (страница/блок), к которому надо привязаться? От этого зависит, нужна ли ObjectModel + фронт-контроллер коллекции, или хватит категории.
9. **Страна происхождения.** Подтвердить, что НЕ нужно писать в `Country` (доставка), а в `Feature`/свою сущность. Нужна ли отдельная витринная страница страны?
10. **Бренд.** `Manufacturer` устраивает как цель по умолчанию? Нужны ли расширенные поля бренда (год основания, сайт, страна, описание, лого) в карточке бренда — и куда (Manufacturer-поля vs своя таблица)?
11. **Очередь.** Какой механизм приоритетен: AJAX-степпер (надёжно на любом хостинге), cron (модуль PS Cron Tasks установлен?), или Messenger (есть ли запущенный воркер на сервере)? Есть ли системный cron у хостинга?
12. **Дедуп медиа в PrestaShop.** Принять компромисс «дедуп предотвращает повторное скачивание, но файл физически копируется на товар» (из-за привязки `Image` к одному `id_product`), или нужен настоящий общий файл (своя таблица м-к-м + кастомный рендер галереи)? Насколько критичен объём диска vs трафик?
13. **Лимиты Wiki API.** Есть ли троттлинг/rate-limit на инстансе OneCatalog? При импорте сотен товаров (каждый = ≥1 запрос payload + N запросов media) можно упереться. Нужна ли пауза между запросами / ретраи с backoff? Какой реальный объём каталога?
14. **Picker (iframe).** Нужен ли виджет-пикер или достаточно импорта по списку `public_id` (вставка/CSV)? Если пикер — учесть CSP/`X-Frame-Options` админки PrestaShop и `parentOrigin` (см. блокеры).
15. **Изображения без токена.** Подтвердить, что токен будет (иначе только `min`-качество). Токен один на инстанс или есть env-секрет?
16. **Что считать «обновлением» при переимпорте.** Какие поля «только при создании» (статус, цена) и какие обновляются всегда (название, габариты, характеристики, фото)? Должны ли ручные правки категорий/фото вебмастера сохраняться (т.е. не перезатирать связи)?
17. **Single vs Pack/Virtual.** Все товары — простые (`Product` без комбинаций)? Есть ли виртуальные/наборы? Влияет на `Combination`/`StockAvailable`.
18. **Площадь упаковки / ед. измерения (м²/шт).** `measurement_unit`, `areas_per_package` — куда (в `Product::unity`/`unit_price`? в Feature? в свой модуль)? По стандарту это «сайтовый слой» (хук), но подтвердить, что ядро их не пишет.

---

## 🚫 Сложно / невозможно на PrestaShop (как написано в стандарте)

1. **«Коллекция как post type / таксономия» — таких сущностей в PrestaShop нет.**
   В PrestaShop нет произвольных типов записей (CPT) и нет «таксономий» как абстракции. *Обход:* настройка «тип объекта + цель» переопределяется тремя реальными вариантами — `Category` (ветка), `Feature` (+FeatureValue), своя ObjectModel-сущность `oc_collection` (+ `oc_collection_lang` + опц. фронт-контроллер для страницы). Стандартный текст «post type | taxonomy» в UI заменить на «своя сущность | категория | характеристика».

2. **Характеристики как «глобальный атрибут pa_*» прямого аналога не имеют.**
   Эталон кладёт характеристики в WooCommerce-атрибуты с `variation=false` (= описательные). В PrestaShop `Attribute`/`AttributeGroup` ВСЕГДА предназначены для **вариаций**, и навешивание их на товар тянет `Combination` + `StockAvailable` на каждую — это семантически неверно для характеристик и раздувает данные. *Обход:* использовать `Feature`/`FeatureValue` (описательные, без комбинаций). Расхождение терминологии: «атрибут» стандарта = `Feature` PrestaShop, а не `Attribute`.

3. **Несколько значений одной характеристики у товара.**
   PrestaShop штатно держит **одно `FeatureValue` на `Feature` на товар**. Если OneCatalog отдаёт у товара несколько значений одной характеристики — прямого хранения нет. *Обход:* (а) склеить значения в одно `FeatureValue` («10; 20; 30»); (б) кастомные (custom) feature value на товар; (в) несколько Feature. Требует решения (вопрос 7).

4. **«Общий медиафайл на много товаров» (дедуп §5.3) не выражается напрямую.**
   `Image` в PrestaShop физически принадлежит одному `id_product` (путь `img/p/<id_image разбит>`). Один и тот же файл, переиспользуемый многими товарами (обложка + лого бренда), как в WP-вложении, недостижим без кастомного слоя. *Обход:* дедуп по контент-ключу предотвращает повторное **скачивание**; файл при этом копируется (экономим трафик и битость, не диск). Настоящий shared-файл — только через свою таблицу м-к-м + кастомный рендер галереи (дороже). «Не удалять общие при апгрейде качества» соблюдается через флаг в `oc_image`.

5. **Мультиязычные поля требуют `id_lang` на каждой записи — стандарт это игнорирует.**
   `name`, `description`, `Feature::name`, `FeatureValue::value`, `Category::name`+`link_rewrite`, `Tag` — всё lang-массивы. API отдаёт один язык за запрос. *Обход:* импортировать в дефолтный `id_lang`; для остальных языков либо повторные проходы API по каждому языку (дороже, N× запросов), либо дублирование значения дефолтного языка (чтобы пройти валидацию непустых полей). `link_rewrite` обязателен и валиден для дефолтного языка минимум — генерировать `Tools::link_rewrite`.

6. **Multistore-контекст.**
   Запись `Product`/`Category`/`Image`/`Configuration`/`Feature` зависит от активного `Shop::getContext`. Импорт вне веб-запроса (cron/Messenger) **не имеет** магазинного контекста по умолчанию → записи уйдут «в никуда» или в дефолтный магазин. *Обход:* в очереди явно `Shop::setContext(Shop::CONTEXT_SHOP, $id_shop)` перед импортом, хранить целевой `id_shop` в задаче, ассоциации `*_shop` заполнять руками (`Image::associateTo`, `Product::id_shop_default`).

7. **Единицы веса/размеров — глобальные, отдельной единицы на товар нет.**
   `PS_WEIGHT_UNIT`/`PS_DIMENSION_UNIT` — на магазин. Это **совпадает** с инвариантом (конвертируем источник → единицы магазина), но значит: нельзя сохранить «исходную единицу товара» в самом товаре — только сконвертированное число. *Обход:* это и не требуется; конвертер `Units` решает задачу. Размеры — ровно в `width/height/depth` (одна единица на все три); «length» OneCatalog → `depth` (глубина) PrestaShop (у `Product` нет «length», есть `depth`).

8. **Фоновые задачи без планировщика.**
   У PrestaShop нет встроенного аналога WP-Cron/Action Scheduler, гарантированно работающего. Системный cron не всегда есть; Messenger-воркер на shared-хостинге обычно не запущен. *Обход:* дефолт — **AJAX-степпер** (браузер ведёт шаги, не нужен серверный планировщик), синхронный фолбэк для малых партий; cron/Messenger — опции для серверов, где это есть. Минус AJAX-степпера: импорт идёт, пока открыта вкладка.

9. **Picker-iframe и CSP/`X-Frame-Options` админки.**
   Бэк-офис PrestaShop отдаёт защитные заголовки; встраивание стороннего iframe (`tools.onecatalog.net/picker.html`) и обмен `postMessage` могут блокироваться CSP/`frame-src`. *Обход:* ослабить CSP только на странице импорта (или вынести пикер в отдельную «чистую» страницу контроллера без жёсткого CSP), origin задавать явно, проверять `event.origin`. Фолбэк без пикера — импорт по списку `public_id` (вставка/CSV) — обязателен как план Б.

10. **`/collections/{slug}/` отдаёт 500; лимиты пагинации справочников.**
    Контракт §2.1: не дёргать `/collections/{slug}/` (берём данные коллекции из эмбеда `collections[]` товара); справочники (`/specifications/`) запрашивать одним запросом с высоким `limit`, сверяясь с `meta.counts`. *Обход:* порт логики `Api::specifications` (limit=1000 + дочитывание страницами); кэш справочника (вместо WP-transient — `Cache`/`Configuration` с TTL или своя таблица на язык).

11. **Цена отсутствует в источнике — `Product::price` обязателен и не может быть null.**
    Стандарт требует «не синтезировать цену». Но `Product` в PrestaShop требует числовую `price` (по умолчанию 0). *Обход:* при создании ставить `0` (не «правдоподобную формульную»), статус/цену — только при создании, при переимпорте не перезатирать (сохраняем ручную правку вебмастера). Цену из своего источника — через `filterOneCatalogProductPrice`. То есть «не задавать» = «оставить 0 и не трогать на апдейте», а не «null».

12. **`reference`/SKU-уникальность и коллизии.**
    `Product::reference` в PrestaShop не уникален на уровне БД (в отличие от Woo SKU), но логически артикул должен быть уникален. *Обход:* перед записью `article` в `reference` проверять коллизию запросом (`SELECT id_product ... WHERE reference=...`), не перезаписывать чужой товар; `public_id` держать отдельно в `oc_product_link` (см. вопрос 3).

---

### Итоговая оценка переносимости

Инварианты ядра (§5: идемпотентность, трекинг качества, дедуп, не-синтез данных, конверсия единиц, порядок зависимостей) переносятся. Расхождения — в слое хранения PrestaShop: **Feature вместо Attribute** для характеристик, **своя таблица для public_id**, **своя сущность/категория/feature для коллекций** (нет CPT/таксономий), **обязательный `id_lang`** на всех человекочитаемых полях, **явный multishop-контекст** в фоновых задачах, **AJAX-степпер** вместо встроенного планировщика, и **дедуп медиа без настоящего shared-файла**. Все они имеют рабочий обход; ни один инвариант стандарта не нарушается, меняется реализация.
