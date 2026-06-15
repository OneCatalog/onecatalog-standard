# Интеграционный план: Joomla (+ HikaShop)

> Реализация модуля импорта OneCatalog Wiki API на **Joomla 4.x / 5.x** поверх **HikaShop** (4.x).
> База — стандарт `INTEGRATION-STANDARD.md`; эталон — WooCommerce-плагин `onecatalog-import`.
> Внешние контракты (§2) и инварианты (§5) стандарта неизменны; меняется только слой «как это лечь в Joomla+HikaShop».

## Какое расширение и почему

Беру **HikaShop**, а не VirtueMart. Обоснование:

1. **Жизнеспособность и активная разработка.** HikaShop регулярно выпускается, имеет официальный bridge-слой `hikashopPlugin` под Joomla 3/4/5/6 и WordPress. VirtueMart жив, но релизы редкие (последний стабильный 4.2.18 — авг. 2024), сообщество и темп заметно меньше; под Joomla 5 порог входа выше.
2. **Программный API уровня сервисов.** У HikaShop есть фабрика `hikashop_get('class.product'|'class.category'|'class.characteristic'|'class.file')` с методами `save()`, `updateCategories()`, `addAlias()`, `updateFiles()` — то есть импорт идёт через бизнес-логику магазина, а не «сырыми» вставками в БД. У VirtueMart есть `models`, но импорт там традиционно болезненнее (структура полей и custom fields сложнее, мультиязычность жёстко зашита в `_lang_`-таблицы — см. блокеры).
3. **Чистая система событий магазина.** HikaShop публикует `onBeforeProductCreate/Update`, `onAfterProductCreate/Update`, `onBeforeCategoryCreate/Update` и т. п. для плагинов группы `hikashop` — это прямой аналог `do_action('product_imported')` эталона (§8 стандарта). У VirtueMart хуки разрознены по `plgVm*`-группам и менее предсказуемы.
4. **Единая «категорийная» модель** упрощает маппинг: товарные категории, бренды (manufacturer), налоговые категории и статусы заказов — всё это строки одной таблицы `hikashop_category`, различаемые полем `category_type`. Это значит, что «категория», «бренд» и потенциально «коллекция» ложатся в один механизм find-or-create.

> ⚠️ **Источник деталей HikaShop — официальная docs/форум HikaShop (WebSearch/WebFetch при реализации), не догадки.** Ряд имён полей (`characteristic_parent_id`, точные сигнатуры `class.file`) подтверждён форумом, но НЕ полной схемой БД; в тексте такие места помечены «⚠️ проверить на инстансе». Версии HikaShop отличаются — фиксировать минимальную поддерживаемую в манифесте.

### Ключевые архитектурные конфликты с Joomla+HikaShop (выношу наверх)

1. **Характеристики OneCatalog `options[]` — описательные, НЕ вариации.** В HikaShop `hikashop_characteristic` + `hikashop_variant` — это механизм **вариантов** (цвет/размер с собственной ценой/складом/SKU). Это НЕ то же, что «характеристика-описание». Прямого нативного аналога описательной характеристики (как WooCommerce `pa_*` без `variation`) у HikaShop нет. Кандидаты: **custom fields таблицы `item`** (`hikashop_field`) ИЛИ переиспользовать characteristics как «просто справочник значений без генерации вариантов». Это центральное расхождение — см. §«Сложно/невозможно» и Вопрос 1.
2. **Мультиязычность.** API OneCatalog отдаёт **один язык за запрос** (`?lang=`). HikaShop нативно мультиязычность товаров НЕ хранит «из коробки» — стандартный путь это **Falang** (таблица `falang_content`) или translation-override (`language/overrides`). Joomla-нативные **Associations** для товаров HikaShop НЕ поддерживает (пришлось бы дублировать товары по языкам, как статьи). Это конфликт со стандартом, написанным под WP (где язык — параметр API, а не размерность хранилища). См. Вопрос 7 и блокеры.
3. **«Коллекция как post type / таксономия» из стандарта в Joomla НЕ существует** в смысле WP CPT/taxonomy. Замены: HikaShop-категория (спец-ветка по `category_type`), Joomla **Custom Fields**, либо собственная JTable-таблица + контроллер. Нет «произвольных типов записей».
4. **Joomla Scheduler (4.1+) ненадёжен в lazy-режиме** (выполняется только при трафике, по одной задаче, зависшая задача блокирует очередь). Для пакетного импорта нужен либо реальный CLI-cron, либо **AJAX-степпер** как основной механизм, а Scheduler — опционально. См. §«Фоновая очередь».
5. **Два «плагинных мира».** Есть Joomla-плагины (группы `system`, `content`, …, событие `onContentAfterSave`, `$app->triggerEvent`/`getDispatcher()->dispatch`) и HikaShop-плагины (группа `hikashop`, класс `hikashopPlugin`, события `onAfterProductCreate`…). Точки расширения стандарта (§8) надо публиковать в **обоих** контурах: внутренние триггеры — через Joomla dispatcher, плюс слушать/эмитить HikaShop-события.

---

## Маппинг сущностей и реализация

### Сводная таблица соответствий (детализация §3 стандарта под HikaShop)

| OneCatalog payload | Абстракция | HikaShop-реализация | Идемпотентность |
|---|---|---|---|
| `product` | Товар | строка `hikashop_product` через `class.product->save()` | по `public_id` (см. ниже) |
| `public_id` (`OC.IND.1`) | Внешний стабильный ключ | **своя таблица** `#__onecatalog_product_link(public_id, product_id)` ИЛИ custom field `item`; рекомендуется своя таблица (не зависит от UI магазина) | `find_by_public_id()` |
| `article` (артикул произв., часто пуст) | Артикул/SKU | `product_code` (родное поле кода товара HikaShop) — **только если пришёл**; коллизию `product_code` не перезаписывать | — |
| `menutitle` | Название | `product_name` | — |
| `description_text` | Описание | `product_description` | — |
| `in_stock` (bool) | Наличие | `product_quantity` (`-1` = неограниченно / либо 0/1) — индикатор наличия, не реальные остатки | при создании |
| `status_publish` / настройка | Статус | `product_published` (1/0) — **только при создании** | при создании |
| `sizes.weight` (граммы) | Вес | `product_weight` (+ `product_weight_unit`) через конвертер `Units` | — |
| `sizes.length/width/height` (мм) | Габариты | `product_dimensions` JSON (`length/width/height` + `product_dimension_unit`) через `Units` — ⚠️ формат хранения габаритов проверить на инстансе | — |
| `options[]` (`text`/`boolean`/`numeric`) | Характеристика | **custom fields `item`** (`hikashop_field`) ИЛИ characteristics-справочник (см. конфликт 1) | по имени поля/значению |
| `categories[]` | Категория | `hikashop_category` (`category_type='product'`) + `class.product->updateCategories()` / таблица `hikashop_product_category` | по имени (label), не по alias |
| `brand` | Бренд | **`hikashop_category` (`category_type='manufacturer'`)** + `product_manufacturer_id` (по умолчанию) ИЛИ custom field (настройка) | по имени manufacturer |
| `country` | Страна происх. | **custom field `item`** (по умолчанию) ИЛИ своя категория-ветка (настройка) | по значению |
| `collections[]` | Коллекция | **своя JTable-сущность** ИЛИ `hikashop_category` спец-ветка ИЛИ custom field (настройка) | по имени/slug |
| `tags[]` | Теги | **Joomla `#__tags`** + `#__contentitem_tag_map` ИЛИ custom field (вкл/выкл) | по имени (label) |
| `images_urls` (обложка) + `files[]` (галерея) | Медиа | `hikashop_file` (`file_type='product'`, `file_ref_id=product_id`) через `class.file` (обложка + галерея, трекинг качества, дедуп по контент-ключу) | по контент-ключу + имени файла |
| `measurement_unit`, `areas_per_package`, `product_shape`, alt картинок, детали бренда, upsells/cross_sells | Прочее (вне ядра) | через событие `onOnecatalogProductImported` (сайтовый слой) | — |

### Слои модуля (перенос §4)

Дистрибутив — **Joomla-компонент** `com_onecatalogimport` (несёт админ-UI, контроллер, настройки, JTable-таблицы) **+ один Joomla-плагин** (группа `system` или собственная) для публикации/слушания событий **+ опционально HikaShop-плагин** (группа `hikashop`) для встраивания кнопки «Импорт» в форму товара HikaShop и реакции на его события. Всё в одном `pkg_`-пакете (Joomla package manifest), чтобы ставилось одной кнопкой.

```
pkg_onecatalogimport/                    — package-манифест (компонент + плагины)
  com_onecatalogimport/
    onecatalogimport.xml                  — манифест компонента: версия, requirements, install/update SQL, langs
    admin/
      src/
        Api/Client.php                    — base URL, token, lang, GET через Joomla\Http, обёртка {data,success}, ошибки→null
        Service/Units.php                  — конвертер вес(г)/размеры(мм) → единицы HikaShop (порт class-units.php 1:1)
        Service/Media.php                  — выбор размера, скачивание+MIME, обложка/галерея, трекинг качества, дедуп
        Service/Taxonomies.php             — custom fields/characteristics (поиск по имени + ручной маппинг), категории, теги
        Importer/CollectionImporter.php
        Importer/BrandImporter.php
        Importer/CountryImporter.php
        Importer/ProductImporter.php       — оркестратор (порядок §5.4), вызывает class.product->save()
        Queue/ImportQueue.php              — AJAX-степпер (основной) + опц. Joomla Scheduler task + синхронный фолбэк
        Table/ProductLinkTable.php         — #__onecatalog_product_link (JTable)
        Table/MediaKeyTable.php            — реестр дедупа медиа по контент-ключу
        Controller/ImportController.php    — приём списка ID + статус (Joomla token/CSRF, ACL-проверка)
        Controller/QueueController.php     — степпер: обработать одну порцию, вернуть прогресс (JSON)
        View/Settings, View/Import         — Joomla MVC views (forms.xml для настроек)
      sql/install.utf8mb4.sql / updates/   — JTable-таблицы и миграции
      language/en-GB, ru-RU                 — .ini + .sys.ini
      forms/config.xml                      — параметры компонента (Joomla JForm) — настройки §7
    media/com_onecatalogimport/js/          — picker-loader.js (iframe+postMessage), admin-import.js (степпер+поллинг)
  plg_system_onecatalogimport/             — Joomla-плагин: триггеры §8 (dispatch), регистрация Scheduler-таски
  plg_hikashop_onecatalogimport/           — HikaShop-плагин: кнопка «Импорт» в форме товара, слушание onAfterProductUpdate
```

Зависимости направлены «вниз» (как в эталоне): `ProductImporter` оркестрирует `Api/Media/Taxonomies/*Importer`; `Queue` → `ProductImporter`; контроллеры → `Queue`. UI-слои зависят от контроллеров/настроек. Циклов нет.

### Идемпотентность и внешний ключ (§5.1, §5.2)

- **`public_id` → своя таблица** `#__onecatalog_product_link (product_id INT, public_id VARCHAR(64) UNIQUE)`. Причина не класть в `product_code`: (а) `product_code` — это «артикул/код витрины», стандарт требует туда писать `article` (§5.2); (б) у вариантов HikaShop `product_code` формируется как `код + id характеристик` — конфликт; (в) поле редактирует вебмастер. `find_by_public_id($public_id)` → `product_id` или 0.
  Легаси-фолбэк (как в эталоне SKU→meta): если в таблице пусто — искать товар по `product_code == public_id` и мигрировать запись.
- **SKU-политика:** `product_code = article` только при непустом `article` и отсутствии коллизии (проверить `SELECT product_id FROM #__hikashop_product WHERE product_code=? AND product_id<>?`); `public_id` в `product_code` не писать.
- **Справочники find-or-create по ИМЕНИ, не по слагу/alias** (§5.1): критично — HikaShop генерит `category_alias` собственной транслитерацией (а Joomla-теги — своей; Falang — третьей). Поиск по alias плодит дубли при разных транслитах («Толщина» → разные alias). Матчить по `category_name` / имени поля / значению без учёта регистра; alias — только при создании нового.
  - категория: `hikashop_category.category_name` в пределах `category_parent_id` и `category_type='product'`;
  - бренд: `hikashop_category.category_name` при `category_type='manufacturer'`;
  - тег: `#__tags.title` (без учёта регистра);
  - custom field значение/опция: по тексту значения.

### Характеристики (§5.3, §5.4) — главное расхождение с эталоном

OneCatalog `options[]` — **описательные** характеристики (text/boolean/numeric), НЕ вариации. В HikaShop `characteristic`/`variant` создают именно ВАРИАНТЫ (комбинации цен/складов/кодов) — использовать их «в лоб» неверно: получим лишние варианты товара, дубли product-строк, сломанный чекаут. Рекомендуемый маппинг:

- **По умолчанию → Joomla/HikaShop Custom Fields таблицы `item` (`hikashop_field`).** Создаём поле типа «text»/«radio»/«checkbox» (для boolean) для каждой `specification_label`, привязываем значение к товару. HikaShop умеет custom fields на `item` и показывать их на странице товара.
  - `text` (`specification_option_name`): значение поля = имя опции.
  - `boolean` (значение в `bool_option`, имя `null`): по настроенному значению поля для true/false (порт `Settings::bool_value($spec_id, $bool)`); пусто → характеристика пропускается. По умолчанию `null/false → false` (§5.6).
  - `numeric` (значение в `numeric_option`): строковое представление числа.
- **Ручной маппинг (строгий режим, §5.4):** таблица `#__onecatalog_field_map (specification_id INT, field_id INT)`. Резолв (порт `Taxonomies::resolve_attribute`): маппинг ВЫКЛ → поле по `specification_label`/создать; маппинг ВКЛ → `specification_id → field_id` из карты, нет привязки → характеристика **пропускается**; для `boolean` — значения true/false из карты.

> ⚠️ **Альтернатива (Вопрос 1):** если проекту важна фасетная фильтрация по характеристикам и витрина уже завязана на characteristics — поддержать второй режим «characteristic как справочник» БЕЗ генерации вариантов (записывать значение, но не плодить `hikashop_variant`). Это нетривиально и зависит от версии HikaShop — пометить как риск.

### Категории, бренд, страна, коллекции, теги

- **Категории:** `hikashop_category` (`category_type='product'`) иерархичны (вложенные множества / `category_parent_id`). OneCatalog отдаёт `categories[]` плоско (`menutitle`, `slug`), родителя нет. Решение: создавать детьми корневой «OneCatalog» категории — find-or-create по имени в пределах этого родителя. `category_alias` генерить из имени. Привязка `class.product->updateCategories($product, $product_id)` (через `$product->categories = [ids]`).
- **Бренд → manufacturer** (`hikashop_category` c `category_type='manufacturer'`), связь `product_manufacturer_id` (по умолчанию). Подтверждено: бренд хранится строкой `hikashop_category`, его `category_id` кладётся в `product_manufacturer_id`. Поля бренда (страна, год, сайт, описание, логотип) — `category_description`/доп. поля или своя таблица; заполняет сайт через событие, ядро ставит только связь.
- **Страна → custom field `item`** «Страна происхождения» (по умолчанию) ИЛИ своя категория-ветка (настройка «тип объекта + цель»). **Не использовать HikaShop-зоны/страны доставки** — это справочник адресов/налогов с ISO, другой смысл; запись туда «страны производителя» сломает доставку.
- **Коллекции → настройка «тип объекта + цель»** (порт `CollectionImporter`): (а) **своя JTable-сущность** `#__onecatalog_collection` (рекомендуется — есть поля `link_3d`, `link_official_site`, обложка, которых нет в категории); (б) `hikashop_category` спец-ветка; (в) custom field. Привязка товар↔коллекция — своя link-таблица или категория-членство. Best-effort обогащение полей коллекции из эмбеда `collections[]` товара (отдельный endpoint `/collections/{slug}/` отдаёт 500 — не использовать, §2.1 стандарта).
- **Теги → Joomla core `#__tags`** + `#__contentitem_tag_map` (вкл/выкл). ⚠️ HikaShop-товар — НЕ нативный Joomla content item, поэтому стандартный `com_tags`-маппинг к нему «из коробки» не привяжется (контекст `com_content.article` не подойдёт). Варианты: (1) свой контекст `com_onecatalogimport.product` + своя выборка; (2) custom field-тег. Пометить как уточнение (Вопрос 5).

### Медиа (§5.3) — `hikashop_file` через `class.file`

- URL `media_files/<base64hash>` **без расширения** → скачивать вручную (`Joomla\Http`), MIME определять по содержимому (`finfo`), сохранять во временный файл, потом отдавать в `class.file`. HikaShop кладёт файлы в `media/com_hikashop/upload/safe`, в БД — относительный путь.
- Запись: создать объект файла (`file_path`, `file_type='product'`, `file_ref_id=product_id`, `file_name`, alt/описание из `images_data`), `hikashop_get('class.file')->save()`, затем `class.product->updateFiles()`. ⚠️ точные имена полей и сигнатуру `updateFiles` сверить на инстансе.
- **Трекинг качества:** рядом с файлом хранить выбранный размер `min|middle|max` (в своей таблице-реестре или в `file_*` доп. поле). На повторе: доступен лучше — перекачать и заменить, хуже — не трогать. Галерея идемпотентна **по имени файла** (подписи в URL меняются).
- **Дедуп по контент-ключу:** извлечь из base64-префикса `media_files` стабильный `path`, ключ `= hash(path # size)`, держать реестр `#__onecatalog_media_key (content_key, file_id|media_path)`; перед загрузкой искать — нашли переиспользуем без скачивания. **Общие (дедуплицированные) файлы НЕ удалять при апгрейде качества** (сломает другие товары).

---

## Фоновая очередь, медиа, настройки, i18n, жизненный цикл

### Фоновая очередь (§6) — AJAX-степпер как основной механизм

Joomla Scheduler (4.1+) для пакетов **ненадёжен**: lazy-режим выполняется только при наличии трафика, по одной задаче за тик, зависшая задача блокирует очередь; «настоящий» путь — серверный CLI-cron `cli/joomla.php scheduler:run`, который не на всех хостингах настроен. Поэтому:

- **Основной механизм — AJAX-степпер** (как нативный импорт PrestaShop/как «admin-import» эталона): список ID режется на порции по «шагу импорта» (минимум 10); фронт `admin-import.js` дергает `QueueController` по порции, тот импортирует элементы и возвращает прогресс/лог JSON; фронт поллит до готовности. Не зависит от cron, не упирается в один веб-таймаут (каждая порция — отдельный запрос).
- **Опционально — Joomla Scheduler task-плагин** (свой `task`-тип): для автоматических периодических импортов на сайтах с настроенным CLI-cron. В lazy-режиме — пометить «не рекомендуется/не гарантируется».
- **Синхронный фолбэк** (§6): если JS недоступен — обработать всё в одном запросе с тем же отчётом (риск таймаута на больших списках — клампить).
- Состояние очереди/лог — в своей JTable-таблице (`#__onecatalog_queue`) или в `#__session`/кэше; лог последних N результатов для прогресс-бара.

### Настройки (§7) — параметры компонента (JForm `config.xml`)

Хранилище — стандартные **параметры компонента** Joomla (`#__extensions.params`, редактируются в `forms/config.xml` / Options). Минимум:

| Настройка | Реализация | По умолчанию |
|---|---|---|
| Токен Wiki API | text; **приоритет env/константы над полем** | пусто |
| Язык товаров | list из `en/ru/ar/zh/kk` | `en` |
| Шаг импорта | number, кламп ≥10 | 10 |
| Статус новых товаров | list (published/unpublished/…); только при создании | published |
| Импортировать коллекции + хранить как | вкл; JTable \| category \| custom field + цель | вкл, JTable |
| Импортировать бренд + хранить как | вкл; manufacturer \| custom field + цель | вкл, manufacturer |
| Импортировать страну + хранить как | вкл; custom field \| category + цель | выкл, custom field |
| Импортировать теги | вкл/выкл (Joomla tags / custom field) | выкл |
| Ручной маппинг характеристик | вкл/выкл + отдельная страница: spec→field, boolean true/false | выкл |

Требования: валидация/кламп; приоритет env-токена; язык — только из списка; UI скрывает нерелевантные поля цели по выбранному типу (`showon` в JForm).

### Конвертер единиц (§5.6) — порт `class-units.php` 1:1

OneCatalog отдаёт вес в **граммах**, размеры в **миллиметрах** (подпись локализована: «мм»/«г»). HikaShop хранит в своих единицах (`product_weight_unit`, `product_dimension_unit` — по настройке магазина). Перед записью конвертировать: таблицы коэффициентов к базовой единице + нормализация локализованных подписей; единицу источника брать из `sizes.*_unit`, иначе мм/г; переопределяемо событием/фильтром. Без этого 1200 мм → 1200 см.

### i18n (§9) — `.ini` + Associations

- Исходные строки EN (`en-GB/com_onecatalogimport.ini` + `.sys.ini`), перевод `ru-RU`. Строки фронт-скриптов передавать из бэкенда через `Text::script()` → `Joomla.Text._()` в JS (не хардкодить).
- **Язык контента vs язык UI:** настройка «язык товаров» (`?lang=` к API) — это язык ИМПОРТИРУЕМЫХ данных, отдельно от языка админ-UI. API даёт один язык за запрос → мультиязычный каталог = N проходов импорта по языку (см. блокеры/Вопрос 7).
- **Associations**: Joomla-нативные ассоциации к HikaShop-товарам неприменимы. Если нужен мультиязычный каталог — через Falang (`falang_content`) или дублирование товаров; ядро импорта должно уметь писать перевод в выбранный механизм или (минимум) импортировать один язык, остальное — забота сайта.

### Жизненный цикл (§10) — манифест компонента + package

- **Манифест** `onecatalogimport.xml` (`<extension type="component">`): `<version>` (SemVer), `<author>`, `<creationDate>`, `<install>/<update>` SQL, `<languages>`, `<media>`, `<administration>`. Package-манифест `pkg_*.xml` собирает компонент + плагины. Зависимость от HikaShop в манифесте напрямую не выражается — проверять наличие в коде (`class_exists('hikashop')`/`hikashop_get`) и не падать при отсутствии (§10).
- **SemVer + CHANGELOG** (Keep a Changelog), версию поднимать во всех местах манифеста перед сборкой; новые UI-строки — переводить.
- **Миграции** — через `<update><schemas>` (SQL по версиям) + опц. install-скрипт (`script.php` с `postflight`) для переноса ключей/флага «миграция выполнена».
- Активация не должна падать без HikaShop; функции импорта проверяют его наличие и деградируют с сообщением.

### Права (ACL)

- Завести компонентные действия в `access.xml` (`core.admin`, `core.manage`, + кастомное `onecatalog.import`). Контроллеры импорта проверяют `$user->authorise('onecatalog.import', 'com_onecatalogimport')` перед запуском.
- CSRF: все POST/AJAX — с Joomla token (`Session::checkToken()` / `JHtml::_('form.token')`), как `nonce` эталона.
- Картинки/файлы пишутся в `media/com_hikashop/upload/safe` — операции под правами магазина; проверять доступность каталога на запись.

### Точки расширения (§8) — два контура событий

- **Joomla dispatcher** (внутренние триггеры ядра импорта): `$app->getDispatcher()->dispatch(new Event('onOnecatalogProductImported', [...]))` с ПОЛНЫМ payload — сайт дозаполняет поля (ед. изм., `areas_per_package`, `product_shape`, alt картинок, детали бренда, upsells) в своём `plg_system`/`plg_hikashop`-плагине. Аналоги `before_import_product`, `collection_imported`, `brand_imported`, `country_imported`, `queue_enqueued`.
- **Фильтры стандарта** (меняют значение: base URL, токен, аргументы, язык, шаг, payload перед импортом, SKU, цена [по умолчанию не задаётся], порядок размеров, дедуп медиа, справочник характеристик, флаги/карта маппинга, origin виджета) — в Joomla нет `apply_filters`; реализуются как **события с изменяемым результатом** (передавать объект-аргумент по ссылке/через `Event::getArgument/setArgument`, как делают сами HikaShop-события `&$element, &$do`).
- **HikaShop-события**: слушать `onAfterProductUpdate` чтобы не затирать ручные правки; при необходимости эмитить собственные для интеграций.

### Виджет-пикер (§2.4) — iframe + postMessage в админке Joomla

- Кнопка на странице товаров (через HikaShop-плагин в форме товара ИЛИ отдельная админ-вью компонента) открывает iframe `https://tools.onecatalog.net/picker.html?token=…&parentOrigin=…`.
- `postMessage`: `ONECATALOG_SELECTED {productIds, productPublicIds}` / `ONECATALOG_CLOSE`. Импортировать по `productPublicIds`. Проверять `event.origin`; origin виджета задавать явно (не `window.location.origin`).
- ⚠️ Joomla-админка отдаёт строгие заголовки/CSP в свежих версиях — **frame-src/`Content-Security-Policy`** может блокировать iframe внешнего домена. Нужно либо ослабить CSP для своей страницы, либо документировать настройку (Вопрос 6). На WP такой проблемы нет — это специфика Joomla-админки + плагинов безопасности.

---

## ❓ Вопросы и сомнения (ответить ДО старта)

1. **Куда класть описательные характеристики `options[]`?** Рекомендация — **custom fields `item`** (`hikashop_field`), НЕ characteristics/variants (последние генерят товарные варианты — неверно). Подтвердить: (а) витрина проекта фильтрует по характеристикам? если да — custom fields дают фасеты HikaShop хуже, чем нужно; (б) допустим ли режим «characteristic как справочник без вариантов» (нетривиально, зависит от версии). Без решения — центральный риск маппинга.
2. **`public_id` — своя таблица или custom field?** Рекомендую свою JTable `#__onecatalog_product_link` (не зависит от UI/обновлений магазина, уникальность на уровне БД). Альтернатива — custom field, но тогда поиск/уникальность слабее. Согласовать.
3. **Бренд: manufacturer достаточно или нужен «бренд как страница/CPT»?** По умолчанию manufacturer (нативно, связь `product_manufacturer_id`). Если проекту нужны лендинги брендов с произвольными полями — мануфактур-описания может не хватить → своя сущность. Уточнить целевое хранилище (паттерн «тип объекта + цель»).
4. **Коллекции: своя JTable, категория или custom field?** Рекомендую свою JTable (есть `link_3d`, `link_official_site`, обложка). Подтвердить, нужны ли эти поля и как коллекция показывается на витрине (отдельная страница? фасет?).
5. **Теги: Joomla `#__tags` или custom field?** HikaShop-товар не нативный content item → стандартный `com_tags`-маппинг к нему не привязывается из коробки. Решить: свой контекст в tag-map vs custom field-тег. Влияет на фронт-вывод.
6. **CSP/iframe в админке Joomla.** Разрешено ли встраивать iframe `tools.onecatalog.net` (CSP сайта, плагины безопасности Admin Tools/RSFirewall)? Если нет — fallback на импорт по списку `productPublicIds` без пикера. Нужна позиция до релиза.
7. **Мультиязычность каталога.** Нужен ли мультиязычный товарный каталог? Если да — какой механизм: **Falang** (`falang_content`) или дублирование товаров по языкам? API даёт один язык за запрос → нужно N проходов. Joomla-Associations к товарам HikaShop неприменимы. От ответа зависит вся стратегия импорта переводов (см. блокеры).
8. **Фоновая очередь: есть ли CLI-cron на хостинге?** Если да — можно задействовать Joomla Scheduler task для авто-импорта. Если нет (типичный shared-хостинг) — только AJAX-степпер; lazy-Scheduler не гарантирует выполнение. Подтвердить инфраструктуру.
9. **Цена.** Стандарт: цену НЕ синтезировать (в API её нет). HikaShop-товар без цены валиден? (обычно да, но проверить настройки отображения/чекаута). Подстановка цены из своего источника — только через событие.
10. **Версии Joomla и HikaShop (минимально поддерживаемые).** Зафиксировать (Joomla 4.4+/5.x; HikaShop 4.x — какая минорная?). От этого зависят: API событий, формат `product_dimensions`, наличие `class.file->updateFiles`, Scheduler. ⚠️ часть имён полей подтверждена форумом, не полной схемой — провести разведку схемы на целевом инстансе (шаг 0 стандарта).
11. **Наличие/остатки (`in_stock`).** Писать в `product_quantity` (`-1`=неограниченно) только индикатор наличия или не трогать вовсе (склад ведёт магазин)? Стандарт — только индикатор, при создании.
12. **Единицы HikaShop.** Какие `product_weight_unit`/`product_dimension_unit` целевые (кг/см? фунты/дюймы?) — нужно для таблиц конвертера. Брать из настроек магазина в рантайме, но подтвердить, что они заданы.

---

## 🚫 Сложно / невозможно на Joomla+HikaShop

1. **Нет нативной «таксономии»/«CPT» как в WP.** «Коллекция как post type / taxonomy» (§3/§7 стандарта) буквально невозможна — у Joomla нет произвольных типов записей и произвольных таксономий, привязываемых к товару. **Обход:** своя JTable-сущность + контроллер (для коллекций), HikaShop-категория спец-ветки (`category_type`) или Joomla Custom Fields. Паттерн «тип объекта + цель» сохраняется, но набор «типов» иной (JTable/category/custom field вместо CPT/taxonomy/attribute).

2. **Описательные характеристики ≠ варианты HikaShop.** HikaShop-characteristics неизбежно тянут механизм вариантов (комбинации цены/склада/кода, доп. строки `hikashop_product` для вариантов). Прямого «глобального атрибута-описания» как `pa_*` в WooCommerce нет. **Обход:** custom fields `item` (рекомендуется) — но они слабее для фасетной фильтрации и не дают «термов» с переиспользованием как WP-таксономия. Если нужна именно фасетная фильтрация по характеристикам — это архитектурный компромисс, обсудить (Вопрос 1).

3. **Мультиязычность — самый тяжёлый конфликт.** API OneCatalog отдаёт ОДИН язык за запрос; Joomla-Associations к HikaShop-товарам неприменимы (нет нативной поддержки — пришлось бы дублировать товары по языкам). HikaShop сам по себе мультиязычность товаров не хранит — нужен **Falang** (отдельная БД-таблица `falang_content`) или translation-override. **Обход:** (а) импортировать один язык (MVP); (б) для мультиязычности — N проходов импорта с разным `?lang=` и запись переводов в Falang через его API (зависимость от стороннего расширения; идемпотентность переводов — отдельная задача). Это принципиально сложнее, чем на WP, где язык — просто параметр.

4. **Joomla Scheduler ненадёжен для пакетов.** Lazy-режим (без CLI-cron) выполняет задачи только при трафике, по одной, зависшая блокирует очередь — на низкотрафиковых/shared-сайтах пакет может не доехать. **Обход:** AJAX-степпер как основной механизм (не зависит от cron), Scheduler — опционально для сайтов с реальным CLI-cron. Синхронный фолбэк для совсем простых случаев.

5. **CSP/security-плагины могут блокировать iframe-пикер в админке.** Joomla-админка (особенно с Admin Tools/RSFirewall и строгим CSP) может не дать встроить iframe внешнего `tools.onecatalog.net`. **Обход:** документировать необходимую CSP-настройку для страницы импорта ИЛИ fallback — импорт по списку `productPublicIds` без виджета (контракт §2.4 допускает работу без пикера).

6. **Теги к HikaShop-товару из core `com_tags` — не из коробки.** HikaShop-товар не является нативным Joomla content item, поэтому стандартный `#__contentitem_tag_map` не привязывается автоматически. **Обход:** собственный контекст в tag-map + своя выборка ИЛИ custom field-тег (проще, но без UI Joomla-тегов).

7. **Два плагинных контура усложняют точки расширения (§8).** Часть событий логично слушать в Joomla-плагине (`onContentAfterSave`/`$app->triggerEvent`), часть — в HikaShop-плагине (`onAfterProductUpdate`, класс `hikashopPlugin`). Нет единого «do_action», и «фильтры» (изменяемые значения) приходится эмулировать событиями с аргументами по ссылке. **Обход:** публиковать собственные триггеры через Joomla dispatcher (единая точка для сайта), а HikaShop-события использовать только для интеграции с формой товара/реакции магазина. Документировать оба набора.

8. **Габариты/единицы хранятся неоднородно по версиям HikaShop.** Формат `product_dimensions` (JSON vs отдельные поля) и наличие методов `class.file->updateFiles` зависят от версии — нельзя жёстко кодировать. **Обход:** разведка схемы на целевом инстансе (шаг 0), фиксация минимальной версии в манифесте, абстракция записи габаритов/файлов за методом сервиса.
