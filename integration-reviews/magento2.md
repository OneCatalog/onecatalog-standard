# Интеграционный план: Magento 2 (Adobe Commerce)

> Реализация стандарта `INTEGRATION-STANDARD.md` (OneCatalog Import) средствами
> Magento 2.4.x / Adobe Commerce. Эталон — WooCommerce. Ниже — перенос каждого
> раздела стандарта на реальные механизмы Magento 2 (классы, интерфейсы, таблицы,
> события), затем список открытых вопросов и блокеры платформы.
>
> Целевой модуль: `Vendor_OnecatalogImport` (`app/code/Vendor/OnecatalogImport`
> или composer-пакет `vendor/onecatalog-import`).
> Минимальная версия: Magento OS 2.4.4+ / Adobe Commerce 2.4.4+ (PHP 8.1+).

---

## Маппинг сущностей и реализация

### 0. Базовый скелет модуля (соответствует §4 «слои»)

Слои стандарта ложатся на стандартную структуру Magento-модуля:

| Слой стандарта (§4) | Реализация в Magento 2 |
|---|---|
| `Plugin` (сборка/ЖЦ/миграции) | `registration.php`, `etc/module.xml`, `Setup/Patch/Data/*`, `Setup/Patch/Schema/*` (declarative schema `etc/db_schema.xml`) |
| `Api` (клиент Wiki API) | `Model/Api/Client.php` на `Magento\Framework\HTTP\ClientInterface` (Curl) или Guzzle; конфиг через `Model/Config.php` (обёртка над `ScopeConfigInterface`) |
| `Units` (конвертер единиц) | `Model/Units/Converter.php` (чистый сервис, без зависимостей от Magento) |
| `Media` | `Model/Media/Importer.php` на `Magento\Catalog\Model\Product\Gallery\Processor` + `Magento\MediaStorage\Model\File\Uploader` / `Filesystem\DirectoryList::MEDIA` |
| `Taxonomies` | `Model/Eav/AttributeResolver.php` + `Model/Catalog/CategoryImporter.php` |
| `CollectionImporter` / `BrandImporter` / `CountryImporter` | отдельные сервисы; цель — EAV-атрибут или категория (см. ниже) |
| `ProductImporter` (оркестратор) | `Model/Import/ProductImporter.php` на `ProductRepositoryInterface` |
| `Queue` | Message Queue (RabbitMQ) **или** cron-consumer на DB-queue (`etc/queue*.xml` + `etc/db_schema.xml`) |
| `Rest` | `etc/webapi.xml` + `Api/ImportManagementInterface` (REST), либо adminhtml-контроллер + ACL |
| `Settings` | `etc/adminhtml/system.xml` + `etc/config.xml` (default values) |
| `AdminUI` | UI Component на странице товаров (`view/adminhtml/ui_component/product_listing.xml` — mass action) + кнопка/модалка с iframe-пикером |
| `assets/picker-loader`, `admin-import` | RequireJS-модули в `view/adminhtml/web/js/*`, объявленные через `requirejs-config.js` + `view/adminhtml/layout` |

DI и зависимости «вниз» (§4) обеспечиваются конструкторным DI Magento
(`etc/di.xml`). Циклов нет — как в эталоне.

---

### 1. Товар (`product`) — §3, §5.1, §5.2

- **Сущность:** `Magento\Catalog\Api\Data\ProductInterface`, тип `simple`
  (по умолчанию). Создание/обновление — через
  `Magento\Catalog\Api\ProductRepositoryInterface::save()`, НЕ через прямые
  ресурс-модели (чтобы отрабатывали индексаторы, плагины и события).
- **Атрибут-сет:** товары пишутся в выделенный attribute set (например
  `OneCatalog`), создаваемый data-патчем через
  `Magento\Eav\Setup\EavSetup` / `Magento\Catalog\Setup\CategorySetup`.
  Это даёт изоляцию EAV-атрибутов характеристик от Default-сета.
- **`public_id` (внешний ключ, §5.1):** отдельный текстовый атрибут продукта
  `onecatalog_public_id` (scope = `global`, не в SKU). Создаётся data-патчем
  через `EavSetup::addAttribute()`. Индексируется для поиска
  (`is_searchable`/`used_in_product_listing` — нет; `is_filterable` — нет;
  но **обязательно** добавить индекс: атрибут типа `varchar`, поиск идёт по
  `catalog_product_entity_varchar` — для производительности можно завести
  собственную flat-таблицу соответствия `onecatalog_product_link`
  (`public_id` PK/unique, `entity_id`) через declarative schema).
- **Идемпотентность (§5.1):** `findByExternalId(public_id)`:
  1) при наличии собственной таблицы — прямой `SELECT entity_id ...`;
  2) иначе — `ProductRepositoryInterface::getList()` с
     `SearchCriteriaBuilder->addFilter('onecatalog_public_id', $id)`.
  Найден → загрузить и обновить; не найден → создать.
- **SKU (§5.2):** Magento `sku` обязателен и уникален. Заполняем `sku`
  значением `article` (артикул производителя) **если он пришёл**. Иначе SKU
  пустым оставить **нельзя** (NOT NULL + бизнес-ограничение) → нужен фолбэк
  (см. блокер №2): синтезировать SKU из `public_id` (например `OC-OC.IND.1`).
  Это отклонение от стандарта (там SKU можно оставить пустым) — обязательно
  согласовать политику. При коллизии `article` с чужим товаром — не
  перезаписывать чужой SKU; для нового товара применить суффикс или фолбэк.
- **Поля «только при создании» (§5.6):** `status`, `visibility`, `price` задаём
  при создании; при апдейте читаем существующий продукт и НЕ перезаписываем эти
  поля. Реализуется в оркестраторе ветвлением `isNew`.
- **Цена (§5.6):** в источнике цены нет. Magento `price` — обязательное поле для
  типа `simple` (decimal, по умолчанию должно быть задано, иначе товар не
  сохранится / будет 0.00). Ставим `0.00` при создании и **не трогаем** при
  апдейте; реальную цену подставляет сайт через событие `product_imported`
  (см. блокер №1). `f(id)`-цену не синтезируем.
- **`in_stock` / наличие:** через MSI —
  `Magento\InventoryApi\Api\SourceItemsSaveInterface` (запись `source_item`
  для нужного `source_code`, `quantity`, `status`). На «голом» 2.4 без
  кастомных источников — `_default` source. См. блокер про MSI ниже.

### 2. Характеристики (`options[]`) → EAV-атрибуты — §3, §5.4

Это центральная точка несоответствия модели (см. блокеры). Маппинг:

- **`specification_type = text`:** EAV-атрибут типа `select` (или
  `multiselect`), `frontend_input = select`, `source_model = eav/entity_attribute_source_table`.
  - Резолв атрибута (§5.4):
    - **ручной маппинг ВКЛ:** по стабильному `specification_id` → код
      существующего атрибута из настроек (см. §7). Импортируются ТОЛЬКО
      сопоставленные характеристики.
    - **ВЫКЛ:** find-or-create по **label** (`frontend_label`), регистронезависимо
      (§5.1) — не по сгенерированному `attribute_code`. Поиск:
      `eav_attribute` JOIN `catalog_eav_attribute` по
      `frontend_label`/`attribute_label` (store-scoped, см. ниже),
      `entity_type_id = catalog_product`. Создание —
      `EavSetup::addAttribute()` НЕ годится в рантайме (это setup-контекст);
      использовать `Magento\Eav\Api\AttributeRepositoryInterface::save()` с
      `Magento\Catalog\Api\Data\ProductAttributeInterface`.
  - Значение характеристики (`specification_option_name`) → **option** атрибута:
    find-or-create опции через `Magento\Eav\Api\Data\AttributeOptionInterface`
    + `Magento\Eav\Model\Entity\Attribute\OptionManagement::add()`; поиск
    существующей опции по label (регистронезависимо, §5.1) через
    `Attribute::getSource()->getOptionId($label)`.
  - **Привязка к атрибут-сету:** добавленный атрибут должен входить в attribute
    set товара (через `EavSetup::addAttributeToSet` в патче, либо динамически —
    что сложнее в рантайме; см. блокер «динамический EAV»).
- **`specification_type = boolean`:** атрибут `frontend_input = boolean`
  (`source_model = Magento\Eav\Model\Entity\Attribute\Source\Boolean`) ИЛИ
  `select` с двумя опциями. Значение из `bool_option`; отсутствие/`null`/`false`
  → **false** (§5.6). В ручном маппинге задаются значения опций для true/false
  (как в стандарте).
- **`specification_type = numeric`:** атрибут `frontend_input = text`,
  `backend_type = decimal`; значение из `numeric_option`. (В стандарте numeric
  явно отдельно — не плодить опции.)

⚠️ **Важно:** в Magento «значение характеристики» — это **option_id внутри
атрибута**, а не отдельная сущность-термин (как WP-term). Поэтому модель
WooCommerce «глобальный атрибут `pa_*` + термины» ложится на «EAV select-атрибут +
options», но НЕ один-в-один (нет общих «терминов» между атрибутами). Это влияет
на дедуп и фасеты — см. блокеры.

### 3. Категории (`categories[]`) — §3, §5.1

- **Сущность:** `Magento\Catalog\Api\Data\CategoryInterface`,
  `Magento\Catalog\Api\CategoryRepositoryInterface`.
- **Идемпотентность по имени (§5.1):** искать категорию по `name`
  (регистронезависимо) под выбранным родителем (настройка «корневая категория
  импорта»), НЕ по `url_key`/слагу. Поиск — через
  `CategoryListInterface::getList()` с фильтром по `name` + `parent_id`, либо
  collection с `addAttributeToFilter('name', ...)`. Не найдена → создать
  (`CategoryInterfaceFactory` + `setParentId` + `setName` + repository->save).
- **Иерархия:** OneCatalog `categories[]` — плоский список без явного дерева в
  payload товара (есть `/categories/` справочник). Нужно решить, строим ли
  дерево из справочника или вешаем все категории плоско под одним родителем
  (вопрос ниже).
- **Привязка к товару:** `setCategoryIds()` на продукте или
  `Magento\Catalog\Model\CategoryLinkManagement::assignProductToCategories()`.

### 4. Бренд (`brand`) — §3, §7

Стандарт требует выбор «тип объекта (атрибут / таксономия / CPT) + цель». В
Magento аналоги «таксономии»/«CPT» — это:

- **EAV-атрибут** (`select`) `manufacturer`/`brand` — самый прямой аналог
  WooCommerce-атрибута. find-or-create опции по label. **По умолчанию** —
  атрибут (в Magento нет «нативной таксономии брендов» как в WP).
- **Категория** (ветка «Бренды») — аналог «таксономии/CPT-страницы»: бренд как
  категория-контейнер. Реализуется тем же `CategoryImporter`.
- **Adobe Commerce only:** есть модуль «Content Staging»/PageBuilder, но
  отдельной нативной сущности «Brand» в Open Source НЕТ (в отличие от WC, где
  есть `product_brand`). Сторонние модули (Amasty/Mageplaza Brands) — внешняя
  зависимость, в ядро не закладываем.

Настройка `brand_store_as` = `attribute | category` + цель (код атрибута или
parent-категория). Тип «CPT» из стандарта на Magento маппится на «категорию» или
кастомную сущность (см. вопрос). После импорта — событие `brand_imported`.

### 5. Страна (`country`) — §3, §7

- **EAV-атрибут** `select` (по умолчанию, как в стандарте) — `country_of_origin`.
  ⚠️ В Magento УЖЕ есть системный атрибут `country_of_manufacture` с
  `source_model = Magento\Catalog\Model\Product\Attribute\Source\Countryofmanufacture`
  (ISO-коды стран, не свободные опции). Можно либо:
  - использовать `country_of_manufacture` и маппить `country.country_code` →
    ISO-2 (тогда опции не создаём — список фиксирован Magento); либо
  - завести собственный `select`-атрибут `onecatalog_country` со свободными
    опциями (find-or-create по label).
  Решение — настройка/вопрос.
- Альтернатива «таксономия» → категория (как у бренда). После — `country_imported`.

### 6. Коллекции (`collections[]`) — §3, §7

- Стандарт: «CPT или таксономия + цель». В Magento:
  - **Категория** (ветка «Коллекции») — основной вариант (есть поля
    `link_3d`, `link_official_site`, изображения → кастомные атрибуты категории
    или категорийные поля). find-or-create по имени.
  - **EAV-атрибут** `select` на товаре — если коллекция нужна как фасет.
  - «Своя сущность/CPT» — потребует собственную таблицу + admin grid
    (трудоёмко; вынести в опцию, по умолчанию категория).
- Доп. поля коллекции (`link_3d` и т.д.) — это «проектные» поля → их пишет сайт
  на событии `collection_imported` (граница «универсальное ↔ проектное», §8),
  либо кастомные атрибуты категории, создаваемые патчем.
- ⚠️ `/collections/{slug}/` отдаёт 500 (§2.1) — берём всё из эмбеда
  `collections[]` товара. Учтено.

### 7. Теги (`tags[]`) — §3, §7

- В Magento **нет** нативной «таксономии тегов товара» (Product Tags убраны
  ещё в M1→M2). Аналоги:
  - EAV-атрибут `multiselect` `onecatalog_tags` (find-or-create опции по имени);
  - либо категория-ветка «Теги».
- По умолчанию импорт тегов ВЫКЛ (как в стандарте). При ВКЛ — `multiselect`.

### 8. Медиа (`images_urls`, `files[]`) — §3, §5.3

- **Загрузчик:** URL без расширения (§2.3) → стандартный
  `media_sideload` отсутствует в Magento; качаем вручную:
  `Magento\Framework\HTTP\ClientInterface` → временный файл в
  `var/import/onecatalog/`, MIME определяем по содержимому
  (`finfo` / `Magento\Framework\File\Mime::getMimeType()`), задаём корректное
  расширение, затем `Magento\Catalog\Model\Product\Gallery\Processor::addImage()`
  / `Magento\Catalog\Model\Product\Media\Config` для записи в
  `pub/media/catalog/product`.
- **Обложка vs галерея:** `addImage($product, $file, ['image','small_image','thumbnail'], false, true)` для обложки; остальные — галерея с
  `['media_image']`.
- **Трекинг качества (§5.3):** размер `min|middle|max` хранить рядом с файлом.
  В Magento у gallery-entry есть поля `label`, но нет произвольного meta. Решение:
  собственная таблица `onecatalog_media_quality`
  (`value_id`/`media_path` → `quality`, `content_key`) через declarative schema.
  На повторном импорте: если доступен размер лучше записанного (появился токен) —
  перекачать, заменить файл, обновить запись; иначе не трогать.
- **Идемпотентность галереи по имени файла (§5.3):** сравнивать по имени файла,
  не по URL (подписи в URL меняются). Magento хранит относительные пути в
  `catalog_product_entity_media_gallery.value` — сверяем по basename.
- **Дедуп по контент-ключу (§5.3):** ключ = `hash(path # size)` из base64-префикса
  `media_files`. Хранить в `onecatalog_media_quality.content_key`. Перед загрузкой
  искать вложение по ключу: нашли → переиспользовать тот же файл (Magento галерея
  допускает шаринг физического файла между товарами через одну запись
  `media_gallery` + связи `..._to_entity`), не качать. ⚠️ Общие файлы НЕ удалять
  при апгрейде качества (сломает другие товары). Это требует аккуратной работы с
  таблицами `catalog_product_entity_media_gallery` /
  `catalog_product_entity_media_gallery_value_to_entity` — см. блокер «дедуп
  медиа в Magento».

### 9. Единицы измерения / габариты (`sizes`, `measurement_unit`, `areas_per_package`) — §5.6

- **Конвертер (`Units`, §5.6):** OneCatalog отдаёт вес в **граммах**, размеры в
  **миллиметрах** (подписи локализованы: «мм», «г»). Magento `weight` — единый
  атрибут (без единицы в ядре; единица — это просто настройка отображения,
  фактически «как договорились», часто фунты/кг). Конвертируем источник →
  целевую единицу магазина (настройка модуля, т.к. в Magento нет
  `woocommerce_weight_unit`-аналога в ядре). Универсальный конвертер: таблица
  коэффициентов к базовой единице + нормализация локализованных подписей.
- `length/width/height` — в Magento ядре **нет** полей габаритов товара (есть
  только `weight`). Нужно завести EAV-атрибуты `ts_dimensions_length/width/height`
  (так делает Temando/MSI Shipping в Commerce) или собственные decimal-атрибуты.
  → это проектные поля; в ядре создаём атрибуты патчем, заполняем в импортёре
  ИЛИ отдаём сайту через `product_imported`. См. вопрос.
- `measurement_unit`, `areas_per_package`, `product_shape` — прямого аналога в
  товаре Magento нет → отдельные кастомные атрибуты (патч) или сайт через
  событие `product_imported` (граница «универсальное ↔ проектное», §2.2/§8).

### 10. Порядок импорта одного товара (§5.4)

В оркестраторе `ProductImporter::import(payload)` строго:

```
payload → resolve existing by public_id (idempotency)
        → base fields (name=menutitle, status[create only], weight via Units)
          (price НЕ задаётся при апдейте; при создании 0.00)
        → categories (find-or-create по имени, assign)
        → характеристики (resolve attribute: manual map by specification_id
          OR by label/create; boolean→true/false value; numeric→decimal;
          set option ids on product)
        → ProductRepository::save()  (получаем entity_id; срабатывают индексаторы)
        → public_id в onecatalog_public_id + запись в link-таблицу
        → brand   (attribute|category по настройке) → event brand_imported
        → country (attribute|category по настройке) → event country_imported
        → tags    (если вкл → multiselect/категория)
        → media (обложка + галерея; трекинг качества + дедуп) 
        → collections (find-or-create цель, привязать) → event collection_imported
        → dispatch event 'onecatalog_product_imported' (полный payload)
```

В Magento каждый `ProductRepository::save()` дорогой (EAV + индексаторы) —
делать ОДИН save после набора всех атрибутов, медиа добавлять отдельно (gallery
processor сохраняет сам). См. блокер «производительность save».

---

## Фоновая очередь, медиа, настройки, i18n, жизненный цикл

### Фоновая очередь (§6)

Два варианта (рекомендуется поддержать оба, выбор фолбэка как в стандарте):

- **Message Queue (RabbitMQ)** — нативный механизм Adobe Commerce / Magento OS
  при наличии AMQP:
  - `etc/communication.xml` — топик `onecatalog.import.batch`.
  - `etc/queue_topology.xml`, `etc/queue_publisher.xml`, `etc/queue_consumer.xml`,
    `etc/queue.xml` — биндинги и consumer `OnecatalogImportConsumer`.
  - Publisher режет список public_id на **порции по «шагу импорта» (≥10)** и
    публикует по сообщению на порцию (`queue_enqueued`-аналог).
  - Consumer (`Model/Queue/Consumer.php`) импортирует элементы порции, пишет
    статус/лог; деградация без падений (§5.5) — try/catch на товар.
  - Запуск: `bin/magento queue:consumers:start OnecatalogImportConsumer` (через
    cron `consumers_runner` / supervisor).
- **DB-queue + cron** (фолбэк без RabbitMQ): собственная таблица
  `onecatalog_import_job` (batch, status, log) + cron-job (`etc/crontab.xml`),
  который тянет pending-порции. **Важно:** AMQP в Magento OS не входит «из
  коробки» как обязательный — на многих инсталляциях очередь крутится на
  `db`-коннекторе (`queue.xml` connection=`db` + cron `MessageQueue`), это
  валидный фолбэк.
- **Синхронный фолбэк (§6):** если ни AMQP, ни cron-consumer не настроены —
  REST-эндпоинт импортирует синхронно с тем же отчётом (риск таймаута на больших
  пакетах — предупредить в UI).
- **Статус/прогресс (§6):** UI ставит в очередь и опрашивает статус. Нужен
  REST-эндпоинт статуса, читающий `onecatalog_import_job` + последние N
  результатов (лог, §5.5).

### Контроллер приёма идентификаторов (§6, чек-лист)

- **REST:** `etc/webapi.xml` → `POST /V1/onecatalog/import` (тело: массив
  `productPublicIds`), `Api/ImportManagementInterface::enqueue(string[] $publicIds)`.
  ACL-ресурс `Vendor_OnecatalogImport::import`. Аутентификация — admin token
  (`Magento\Webapi`).
- **Adminhtml-контроллер** (для UI-кнопки/пикера): `Controller/Adminhtml/Import/Enqueue.php`
  extends `Magento\Backend\App\Action`, защита — `_isAllowed()` (ACL) +
  Magento form key (аналог nonce/CSRF). area = `adminhtml`.

### Настройки (§7) — `system.xml` / `config.xml`

`etc/adminhtml/system.xml` секция `onecatalog_import`:

| Настройка стандарта | Поле system.xml | Тип/source | Default (config.xml) |
|---|---|---|---|
| Токен Wiki API | `general/api_token` | `obscure` (зашифровать, `backend_model=Encrypted`) | пусто; **env приоритетнее** |
| Язык товаров | `general/lang` | `select`, source из API-списка (`en/ru/ar/zh/kk`) | `en` |
| Шаг импорта | `queue/step` | `text` + backend-валидация **≥10** (clamp) | 10 |
| Статус новых товаров | `product/default_status` | `select` (Enabled/Disabled; + visibility) — **только при создании** | Enabled |
| Импорт коллекций + хранить как | `collections/enabled`, `collections/store_as` (attribute\|category) + цель | `select` + зависимые поля | вкл, category |
| Импорт бренда + хранить как | `brand/enabled`, `brand/store_as`, `brand/target` | как выше | вкл, attribute |
| Импорт страны + хранить как | `country/enabled`, `country/store_as`, `country/target` | как выше | выкл, attribute |
| Импорт тегов | `tags/enabled` | `select` | выкл |
| Ручной маппинг характеристик | `specs/manual_enabled` + отдельная страница маппинга | custom admin grid | выкл |

- **Env-приоритет токена (§7):** в `Model/Config::getToken()` сначала
  `getenv('ONECATALOG_API_TOKEN')` / `Magento\Framework\App\DeploymentConfig`
  (`env.php`), затем `scopeConfig`. Шифрование поля — `Encrypted` backend model.
- **Зависимые поля цели** (показывать «код атрибута» при `store_as=attribute`,
  «parent-категорию» при `category`) — `depends` в system.xml.
- **Страница ручного маппинга характеристик** (`specification_id` → атрибут,
  для boolean — значения true/false из опций атрибута) — отдельный adminhtml
  controller + UI-grid (источник характеристик — `/specifications/` API,
  одним запросом с высоким `limit`, §2.1).
- **Scope:** настройки на уровне `default`/`website`/`store` — но язык товаров
  логичнее привязать к **store view** (мультиязычность, см. ниже).

### i18n (§9)

- Исходные строки EN, единый перевод-механизм Magento: файлы
  `i18n/en_US.csv` (шаблон) и `i18n/ru_RU.csv`. Magento НЕ использует POT/PO/MO —
  это CSV (`"source","translation"`). Эталонный POT/PO → конвертируем в CSV.
- Строки для JS (пикер, прогресс) — через `Magento\Framework\Translate\Js`
  (`view/adminhtml/web/js`/`$.mage.__()`), не хардкодить в JS (§9). Передача
  локализованных строк из бэкенда — через `jsLayout`/`$block->getJsLayout()` или
  `Magento\Framework\Locale`.
- ⚠️ В памяти проекта зафиксировано требование «русский UI» (см.
  `feedback-onecatalog-i18n-russian-ui`): для Magento это `i18n/ru_RU.csv`,
  активный при store view `ru_RU`. POT/PO эталона неприменим напрямую.

### Жизненный цикл, миграции, права (§10, §5.1 легаси)

- **Манифест:** `etc/module.xml` (`setup_version` устарел в declarative; версия
  модуля — в `composer.json`). Требования (Magento/PHP) — `composer.json`
  `require`. SemVer + CHANGELOG (Keep a Changelog) — как в стандарте/памяти
  проекта.
- **Схема:** `etc/db_schema.xml` (declarative schema) для таблиц
  `onecatalog_product_link`, `onecatalog_media_quality`, `onecatalog_import_job`.
  EAV-атрибуты, attribute set, root-категории — `Setup/Patch/Data/*Patch.php`
  (идемпотентные, с `getDependencies()`/`getAliases()`).
- **Миграция ключа (§5.1 легаси):** если раньше `public_id` лежал в SKU —
  data-патч переносит в `onecatalog_public_id` + заполняет link-таблицу; в
  рантайме фолбэк-поиск по SKU. Флаг «миграция выполнена» — сам факт
  применённого патча (`patch_list`).
- **Права/area (§6, чек-лист):** `etc/acl.xml` ресурс
  `Vendor_OnecatalogImport::import` (+ `::config`); admin-меню `etc/adminhtml/menu.xml`.
  Импорт-операции — только area `adminhtml`/`webapi_rest` с ACL; CSRF — form key.
- **Активация без «движка магазина»:** в Magento каталог — часть ядра, отдельной
  зависимости-движка нет (в отличие от WP+WooCommerce). Зависимости объявляем в
  `module.xml` `<sequence>` (`Magento_Catalog`, `Magento_Eav`,
  `Magento_CatalogInventory`/`Magento_InventoryApi`, `Magento_MediaStorage`).
  Падать при `setup:upgrade` без них Magento не даст — проверка через sequence.

### Точки расширения (§8) — events/plugins

Magento-аналог `do_action`/`apply_filters`:

- **События (реагируют) → `Magento\Framework\Event\ManagerInterface::dispatch()`**
  + наблюдатели в `etc/events.xml` (area-scoped):
  - `onecatalog_before_import_product` (payload, existing_id)
  - `onecatalog_product_imported` (product, payload, report) ← **сайт дозаполняет
    проектные поля здесь** (ед. изм., `areas_per_package`, `product_shape`,
    alt картинок, габариты, upsells/cross_sells, цена — всё вне таблицы маппинга).
    В Magento `upsells`/`cross_sells`/`related` — это
    `Magento\Catalog\Api\ProductLinkRepositoryInterface`; их по `public_id`
    проставит сайт (порядок импорта не гарантирует наличие связанных товаров).
  - `onecatalog_collection_imported`, `onecatalog_brand_imported`,
    `onecatalog_country_imported`, `onecatalog_queue_enqueued`.
- **«Фильтры» (меняют значение) — у Magento нет `apply_filters`.** Аналоги:
  - **Plugins (interceptors)** на публичных методах сервисов (base URL, token,
    request args, lang, step, payload-before-import, SKU, price, image sizes,
    media dedup, gallery files, specs-source, флаги/карта маппинга, picker
    origin) — `etc/di.xml` `<plugin>` (around/before/after).
  - Для этого ВСЕ перечисленные «фильтруемые» значения должны проходить через
    публичные методы-сервисы с интерфейсами (а не приватные хелперы), иначе
    plugin не навесить. Это требование к архитектуре модуля.
  - Где «фильтр» меняет данные внутри потока — можно дополнительно
    `dispatch` с объектом-обёрткой (`DataObject`), наблюдатель меняет поле.
    Но идиоматичнее — interceptor.
- **Граница «универсальное ↔ проектное» (§8):** проектные поля витрины пишет
  отдельный сайтовый модуль (`Vendor_OnecatalogSite`) на
  `onecatalog_product_imported` — ядро о витрине не знает (точно как mu-плагин
  в эталоне).

---

## ❓ Вопросы и сомнения (ответить ДО старта)

1. **SKU при отсутствии `article`.** Стандарт (§5.2) разрешает оставить SKU
   пустым. Magento этого не допускает (SKU обязателен и уникален). Какую политику
   фолбэка принять: SKU = `public_id` (`OC.IND.1`), SKU = префикс+public_id, или
   запрет импорта товара без article? Это влияет на идемпотентность и на UX.
2. **`public_id`: EAV-атрибут vs собственная link-таблица.** Поиск по
   EAV-`varchar` медленный на больших каталогах. Закладывать ли собственную
   таблицу `onecatalog_product_link` (быстрее, но дублирует данные) или хватит
   индексированного EAV-атрибута? Ожидаемый объём каталога?
3. **Тип товара.** Все импортируемые товары — `simple`? Есть ли вариации
   (цвет/размер как configurable)? В payload `options[]` — это характеристики, а
   не варианты; подтвердить, что configurable НЕ требуется (иначе вся модель
   характеристик/атрибутов меняется — нужны attribute с `is_configurable`).
4. **Attribute set.** Заводить выделенный set `OneCatalog` или писать в Default?
   От этого зависит, какие EAV-атрибуты видны в каких товарах и как чистить.
5. **Характеристики: один глобальный атрибут на `specification_label` для всех
   товаров?** В Magento атрибут — глобальная сущность EAV. Если у разных
   `specification_id` совпадает label («Цвет») — это один атрибут или разные? В
   ручном маппинге ключ — `specification_id`, но в авто-режиме (§5.1) поиск по
   label склеит разные spec с одним именем. Это ожидаемо?
6. **`numeric`-характеристики как фасет.** Делать их `decimal`-атрибутами
   (фильтр-диапазон) или текстовыми? Нужны ли они в layered navigation
   (`is_filterable`)?
7. **Категории: дерево или плоско?** Payload товара даёт плоский `categories[]`.
   Строить иерархию из `/categories/` справочника (есть ли там `parent`?) или
   вешать все под одной root-категорией импорта? Какая root-категория (store
   root)?
8. **Бренд/страна/коллекция — целевая модель по умолчанию.** Стандарт даёт выбор
   «атрибут/таксономия/CPT». В Magento «таксономия/CPT» = категория или кастомная
   сущность. Согласовать дефолты: бренд=атрибут? страна=`country_of_manufacture`
   (ISO) или свой атрибут? коллекция=категория? Нужна ли вообще кастомная
   сущность (своя таблица+grid) для коллекций или достаточно категории?
9. **`country_code`.** Использовать системный `country_of_manufacture` (фикс.
   ISO-список Magento, маппинг по `country_code`) или свободный select по label?
   Если страны OneCatalog не покрываются ISO-списком Magento — что делать?
10. **Габариты (`length/width/height`).** В ядре Magento их нет. Заводить
    EAV-атрибуты `ts_dimensions_*` (MSI Shipping), свои decimal-атрибуты, или
    отдать целиком сайту через `product_imported`? Нужны ли они для расчёта
    доставки?
11. **Единицы веса/размеров магазина.** В Magento нет `woocommerce_weight_unit`.
    В какие единицы конвертировать (кг/г, см/мм)? Брать целевую единицу из
    настройки модуля? Подтвердить таблицу коэффициентов.
12. **MSI / наличие.** На какой source писать `in_stock` (`_default` или
    конкретный source_code)? Количество в источнике нет — ставить
    qty=0/qty=большое при `in_stock=true`? Нужна ли вообще запись stock при
    импорте (или только статус)?
13. **Цена = 0.00.** Magento покажет товар с ценой 0.00 на витрине. Скрывать ли
    такие товары (visibility) до подстановки цены сайтом, или сайт обязан
    проставить цену на `product_imported` синхронно? Политика отображения
    «нулевых» товаров?
14. **Мультиязычность (store views).** OneCatalog `lang` — один язык на запрос.
    Magento хранит переводимые атрибуты per store view. Импортировать каждый язык
    в свой store view (несколько прогонов с разным `lang` → запись store-scoped
    значений) или только в один (default)? Это меняет резолв атрибутов/опций
    (они store-scoped) и идемпотентность.
15. **Очередь: AMQP или DB-queue?** На целевой инсталляции есть RabbitMQ или
    закладываем DB-коннектор + cron? От этого зависит набор `etc/queue*.xml` и
    требования к окружению/деплою.
16. **Статус/прогресс UI.** Достаточно ли REST-поллинга job-таблицы, или нужен
    стриминг (SSE)? Сколько последних результатов в логе (N)?
17. **Лимиты API источника.** Какие rate limits у Wiki API? При импорте 1000+
    товаров и медиа — нужен ли троттлинг/ретраи (`/specifications/` одним
    запросом с высоким limit — какой максимум `limit` безопасен)?
18. **Iframe-пикер и CSP.** Magento adminhtml имеет
    `Content-Security-Policy` (`csp_whitelist.xml`). iframe на
    `tools.onecatalog.net` + `postMessage` потребуют whitelиста `frame-src` и
    проверки `event.origin`. sandbox iframe + `allow-same-origin` —
    согласовать с политикой безопасности. Разрешено ли встраивать внешний iframe
    в админку?
19. **Где кнопка импорта в админке.** Mass action на product grid (по выбранным —
    но это локальные товары, а пикер выбирает внешние) ИЛИ отдельная страница
    «Импорт из OneCatalog» в меню? Логичнее отдельная страница + пикер; уточнить
    UX.
20. **Дедуп медиа между товарами.** Шарить ли один физический файл между
    товарами (Magento позволяет, но усложняет логику и риск «удалил у одного —
    пропало у всех»)? Или дублировать файл на каждый товар (проще, больше места)?
    Стандарт требует дедуп — подтвердить, что идём в шаринг.
21. **Индексация.** После пакетного импорта запускать реиндекс
    (`bin/magento indexer:reindex`) или полагаться на on-save? На больших пакетах
    on-save медленный. Переключать индексаторы в `schedule` и реиндексить после
    батча?
22. **Удаление/снятие товаров.** Стандарт описывает импорт/апдейт, но не
    удаление товаров, исчезнувших из источника. Нужна ли синхронизация удалений
    (disable/delete) или только upsert?

---

## 🚫 Сложно / невозможно на Magento 2

1. **SKU нельзя оставить пустым (расхождение со стандартом §5.2).**
   *Причина:* в Magento `sku` — обязательное уникальное поле, без него
   `ProductRepository::save()` бросит исключение. Стандарт допускает пустой SKU
   при отсутствии `article`.
   *Обход:* фолбэк-SKU из `public_id` (или префикс). Требует явного решения
   (вопрос 1) и аккуратной идемпотентности (искать всё равно по `public_id`, не
   по сгенерированному SKU).

2. **Цена обязательна — «не задавать цену» невозможно буквально (§5.6).**
   *Причина:* для `simple` товара `price` фактически обязателен (иначе товар не
   валиден/не покупаем; на витрине 0.00). В WooCommerce товар без цены —
   нормален.
   *Обход:* ставить `0.00` при создании, не трогать при апдейте, цену
   подставляет сайт на `product_imported`; опционально скрывать нулевые товары
   через visibility до подстановки (вопрос 13). Полностью «без цены» — нельзя.

3. **Динамическое создание EAV-атрибутов в рантайме — дорого и рискованно.**
   *Причина:* идиоматично атрибуты создаются в `Setup/Patch` (setup-контекст), а
   не на лету при импорте. Создание атрибута в рантайме (`AttributeRepository::save`)
   требует добавить его в attribute set/group, очистить EAV-кэш, потенциально
   реиндекс — на каждый новый `specification_label` это тяжело и плохо
   масштабируется; параллельные consumers могут гонками создать дубликаты.
   *Обход:* (а) **рекомендуемый ручной маппинг** (§5.4, строгий режим) — атрибуты
   заведены заранее патчем, импорт только проставляет опции; (б) если auto-create
   нужен — делать его в одной транзакции с локом
   (`Magento\Framework\Lock\LockManagerInterface`), батчить создание атрибутов
   ДО импорта товаров (предсканировать `/specifications/`), затем чистить
   `eav`-кэш один раз. «Создавать атрибут по ходу каждого товара» как в WC — не
   масштабируется.

4. **Нет общих «терминов» между атрибутами (EAV vs WP-term).**
   *Причина:* в WooCommerce термин таксономии может переиспользоваться; в Magento
   значение характеристики — это `option` ВНУТРИ конкретного атрибута, не
   разделяемая сущность. Модель «атрибут + термин» стандарта (§3) ложится не
   один-в-один.
   *Обход:* трактовать «термин» как option атрибута; дедуп опций — в пределах
   одного атрибута по label. Кросс-атрибутного шаринга значений не будет (и не
   нужно).

5. **Нет нативных «таксономии брендов» и «тегов товара» (CPT-аналогов).**
   *Причина:* в отличие от WooCommerce (`product_brand`, теги) и WP CPT, Magento
   OS не имеет нативной сущности «бренд» или «тег товара».
   *Обход:* маппить на EAV-атрибут (`select`/`multiselect`) или на категорию;
   «CPT/таксономия» стандарта → «категория» или кастомная сущность (своя
   таблица + admin grid — трудоёмко). Дефолты — атрибут (бренд/страна/теги),
   категория (коллекции).

6. **Нет ядровых полей габаритов товара (`length/width/height`).**
   *Причина:* Magento товар имеет только `weight`; габариты появляются лишь с
   MSI/Temando shipping (атрибуты `ts_dimensions_*`) или кастомные.
   *Обход:* завести decimal-атрибуты патчем и заполнять в импортёре, либо отдать
   сайту через `product_imported`. Прямого аналога нет.

7. **MSI усложняет «in_stock» (нет простого булева наличия).**
   *Причина:* в 2.4 наличие — это `source_item` (source_code + qty + status) +
   reservations; простого «in stock yes/no» на товаре нет (legacy
   `cataloginventory_stock_item` — через MSI-адаптер). В источнике нет
   количества.
   *Обход:* писать `SourceItem` на `_default` source со `status = in/out` и
   условным qty (вопрос 12). На отключённом MSI — legacy stock item. Требует
   зависимости `Magento_InventoryApi` и решения по qty.

8. **Нет `apply_filters` — «фильтры» стандарта (§8) делаются плагинами, что
   требует публичного API.**
   *Причина:* у Magento нет глобального фильтр-хука; изменить значение можно
   только interceptor'ом на публичном методе.
   *Обход:* спроектировать ВСЕ «фильтруемые» точки (URL, token, lang, step,
   payload, SKU, price, image sizes, media dedup, gallery, specs source, флаги,
   picker origin) как публичные методы интерфейсов-сервисов, на которые можно
   навесить plugin. Если метод приватный — фильтр невозможен. Это ограничение
   на архитектуру, не на платформу, но забыв про него — потеряем переносимость.

9. **i18n не POT/PO/MO, а CSV.**
   *Причина:* Magento использует `i18n/*.csv`, а не gettext PO/MO эталона.
   *Обход:* конвертировать строки в `i18n/en_US.csv` + `i18n/ru_RU.csv`; JS-строки
   через `Magento\Framework\Translate\Js`. Эталонный POT/PO неприменим напрямую —
   нужен отдельный пайплайн строк (но это перенос, не блокер).

10. **Iframe-пикер в админке упирается в CSP/`frame-src` и form key.**
    *Причина:* adminhtml применяет CSP (`csp_whitelist.xml`); внешний iframe и
    `postMessage` требуют whitelист `frame-src https://tools.onecatalog.net` и
    проверки `event.origin`. Жёсткие CSP-инсталляции могут запрещать внешние
    фреймы.
    *Обход:* добавить домен в `csp_whitelist.xml` (`frame-src`,
    при необходимости `connect-src`), sandbox iframe, строгая проверка
    `event.origin`. Если безопасность запрещает внешний iframe — фолбэк «импорт
    по списку public_id» (стандарт это допускает, §2.4 опционален).

11. **Производительность `ProductRepository::save()` на больших пакетах.**
    *Причина:* каждый save затрагивает EAV (десятки строк), плагины, индексаторы;
    1000+ товаров с медиа в синхронном/мелко-порционном режиме медленны.
    *Обход:* индексаторы в `schedule`-режим + реиндекс после батча (вопрос 21);
    один `save` на товар (не несколько); медиа-дедуп ради экономии IO; очередь с
    разумным шагом (стандарт ≥10, для Magento — выше, 20–50). Bulk-API
    (`Magento\AsynchronousOperations`) — альтернатива MQ для массовых операций.

12. **Store-view мультиязычность не покрывается «одним lang за импорт».**
    *Причина:* Magento хранит переводимые значения атрибутов/опций per store
    view; стандарт мыслит «один язык на запрос». Импорт в один прогон заполнит
    только один scope.
    *Обход:* отдельные прогоны на каждый `lang` с записью store-scoped значений
    (имя, опции, категории); резолв опций/атрибутов вести по `admin`-scope label,
    переводы — на store view. Это многократно усложняет идемпотентность опций
    (вопрос 14) — без явного решения легко наплодить дубли опций в разных store.
