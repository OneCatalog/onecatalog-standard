# OneCatalog Integration Standard

Платформо-независимый **стандарт интеграции** для плагинов импорта каталога из
[OneCatalog](https://docs.onecatalog.net/) (Wiki API) в товарные системы разных
CMS/CRM. Это **канонический источник истины** для всех платформенных портов.

- 📄 **[INTEGRATION-STANDARD.md](INTEGRATION-STANDARD.md)** — сам стандарт. Два
  сценария: **импорт каталога** из Wiki API (§1–§12: контракты, маппинг, слои,
  инварианты, очередь, настройки, точки расширения, i18n, жизненный цикл, чек-лист) и
  **синхронизация цен и остатков** из B2B-фида (§13: отдельный API/авторизация,
  стратегии цены × регионы, остатки по складам, change-detection против нагрузки).
- 🗂 **[integration-reviews/](integration-reviews/)** — планы портирования по
  платформам (`<платформа>.md` — план, `<платформа>-answers.md` — ответы на вопросы
  плана: факты живого API + рекомендации).
- 🧪 **fixtures/** — общие тестовые payload'ы с живого API (для всех портов).

Документация OneCatalog: <https://docs.onecatalog.net/>

## Эталонная реализация

Стандарт абстрагирован от **эталонной реализации** на WordPress + WooCommerce:
**[OneCatalog/onecatalog-woocommerce](https://github.com/OneCatalog/onecatalog-woocommerce)**
(namespace `OneCatalog\Import`). На любой новой платформе строится по этому стандарту
(внешние контракты §2 и инварианты §5 неизменны; меняется только слой «как это лечь
в конкретную CMS»).

## Статус платформ

| Платформа | План | Ответы | Репозиторий |
|---|---|---|---|
| WooCommerce (эталон) | — | — | ✅ [onecatalog-woocommerce](https://github.com/OneCatalog/onecatalog-woocommerce) |
| 1С:Управление Торговлей (УТ 11.5) | — | — | ✅ [onecatalog-1c](https://github.com/OneCatalog/onecatalog-1c) (расширение в ERP, `v1.1.3`) |
| 1С-Битрикс | 📦 [в репо](https://github.com/OneCatalog/onecatalog-bitrix/blob/dev/docs/integration-plan.md) | 📦 [в репо](https://github.com/OneCatalog/onecatalog-bitrix/blob/dev/docs/integration-answers.md) | 🚧 [onecatalog-bitrix](https://github.com/OneCatalog/onecatalog-bitrix) (в разработке, ветка `dev`) |
| Bitrix24 (CRM, REST-приложение) | ✅ | ✅ | 🔜 planned (отдельная цель от 1С-Битрикс: `crm.product` + REST/OAuth, не модуль `/bitrix/modules/`) |
| PrestaShop | ✅ | ✅ | 🔜 planned |
| OpenCart (3.0.3.x) | 📦 [в репо](https://github.com/OneCatalog/onecatalog-opencart/blob/dev/docs/integration-plan.md) | 📦 [в репо](https://github.com/OneCatalog/onecatalog-opencart/blob/dev/docs/integration-answers.md) | 🚧 [onecatalog-opencart](https://github.com/OneCatalog/onecatalog-opencart) (в разработке, ветка `dev`) |
| Magento 2 | ✅ | ✅ | 🔜 planned |
| Shopify | ✅ | ✅ | 🔜 planned |
| BigCommerce | ✅ | ✅ | 🔜 planned |
| Drupal Commerce | ✅ | ✅ | 🔜 planned |
| Joomla | ✅ | ✅ | 🔜 planned |
| МойСклад | ✅ | ✅ | 🔜 planned |
| RetailCRM | ✅ | ✅ | 🔜 planned |
| Megaplan | ✅ | ✅ | 🔜 planned |
| Moguta.CMS | ✅ | ✅ | 🔜 planned |
| NetCat | ✅ | ✅ | 🔜 planned |
| Shop-Script | ✅ | ✅ | 🔜 planned |
| PHPShop | ✅ | ✅ | 🔜 planned |
| UMI.CMS | ✅ | ✅ | 🔜 planned |
| SAP | ✅ | ✅ | 🔜 planned |

При реализации платформы её план (`<платформа>.md` + `-answers.md`) переезжает в
репозиторий платформы как `docs/integration-plan.md`, а здесь остаётся ссылка.

## Версионирование

Стандарт версионируется отдельно (см. [CHANGELOG.md](CHANGELOG.md)). Каждый
платформенный порт указывает, какой версии стандарта он соответствует.

## Лицензия

[MIT](LICENSE).
