# WordPress продвинутый — экзаменационный конспект

> **Цель:** подготовка к экзамену по дочерним темам и разработке плагинов WordPress.  
> **Версия:** ориентир WordPress 6.x, PHP 8.0+.  
> **Как пользоваться:** каждый раздел — отдельный вопрос; переходите по оглавлению.

---

## Оглавление

1. [Создание и работа с дочерними темами](#1-создание-и-работа-с-дочерними-темами)
2. [Создание плагина, жизненный цикл, соответствующие хуки](#2-создание-плагина-жизненный-цикл-соответствующие-хуки)

---

## Введение: тема vs плагин

| | **Тема (Theme)** | **Плагин (Plugin)** |
|---|------------------|---------------------|
| **Назначение** | Внешний вид, шаблоны, вывод | Функциональность |
| **При смене темы** | Код темы отключается | Плагин **остаётся** |
| **Где править вёрстку** | Тема / дочерняя тема | — |
| **Где править логику** | Лучше в плагине | Плагин |

> **Правило:** кастомизацию **родительской** темы — только через **дочернюю**. Бизнес-логику — в **плагин**.

---

## 1. Создание и работа с дочерними темами

### Зачем нужна дочерняя тема (Child Theme)

**Дочерняя тема** — тема, которая **наследует** шаблоны и стили **родительской (parent)**, но позволяет **переопределять** отдельные файлы и добавлять свой CSS/PHP **без правки** файлов родителя.

```
При обновлении родительской темы
────────────────────────────────
Родитель (Twenty Twenty-Four)  →  обновляется с wordpress.org
Дочерняя (my-shop-child)       →  ваши правки СОХРАНЯЮТСЯ
```

| Без child theme | С child theme |
|-----------------|-------------|
| Правки в `functions.php` родителя | Правки в child |
| Обновление темы **затирает** изменения | Родитель обновляется безопасно |

---

### 1.1 Минимальная структура дочерней темы

```
wp-content/themes/my-child/
├── style.css          ← обязательно (заголовок темы)
├── functions.php      ← подключение стилей, хуки
├── screenshot.png     ← превью в админке (опционально)
├── template-parts/    ← переопределённые части шаблонов
├── page-custom.php    ← кастомные шаблоны страниц
└── ...
```

---

### 1.2 Файл style.css — обязательный заголовок

WordPress определяет дочернюю тему по полю **`Template`** — slug папки **родителя**.

```css
/*
 Theme Name:   My Shop Child
 Theme URI:    https://example.com
 Description:  Дочерняя тема для My Shop
 Author:        Ваше имя
 Author URI:   https://example.com
 Template:      my-shop
 Version:      1.0.0
 License:      GNU General Public License v2 or later
 Text Domain:  my-shop-child
*/

/* Стили ниже переопределяют родителя */
.site-header {
    background-color: #1a1a2e;
}
```

| Поле | Обязательно | Описание |
|------|-------------|----------|
| `Theme Name` | Да | Название в админке |
| `Template` | **Да для child** | Имя папки родителя (`my-shop`, не «My Shop») |
| `Version` | Рекомендуется | Версия дочерней темы |

**Активация:** Внешний вид → Темы → активировать **My Shop Child**.

---

### 1.3 Подключение стилей родителя и дочерней темы

Родительские стили **не подключаются автоматически** — их нужно enqueue в `functions.php` дочерней темы.

```php
<?php
/**
 * functions.php дочерней темы
 */

add_action('wp_enqueue_scripts', 'my_child_enqueue_styles');

function my_child_enqueue_styles(): void
{
    $parent = 'my-shop'; // slug родительской темы

    // 1. Стили родителя
    wp_enqueue_style(
        $parent . '-style',
        get_template_directory_uri() . '/style.css',
        [],
        wp_get_theme($parent)->get('Version')
    );

    // 2. Стили дочерней темы (зависят от родителя)
    wp_enqueue_style(
        'my-child-style',
        get_stylesheet_directory_uri() . '/style.css',
        [$parent . '-style'],
        wp_get_theme()->get('Version')
    );
}
```

| Функция | Возвращает |
|---------|------------|
| `get_template_directory()` | Путь к **родителю** (файловая система) |
| `get_template_directory_uri()` | URL родителя |
| `get_stylesheet_directory()` | Путь к **активной** теме (child) |
| `get_stylesheet_directory_uri()` | URL активной темы |

> Если активна **только** родительская тема без child — `get_template_*` и `get_stylesheet_*` совпадают.

---

### 1.4 Переопределение шаблонов

WordPress ищет файлы **сначала в дочерней**, потом в родительской.

| Файл в child | Эффект |
|--------------|--------|
| `header.php` | Используется вместо родительского |
| `single.php` | Свой шаблон записи |
| `woocommerce/checkout/form-checkout.php` | Переопределение WooCommerce (если тема поддерживает) |

**Иерархия шаблонов (пример для записи):**

```
single-{post-type}-{slug}.php
single-{post-type}.php
single-{id}.php
single.php
singular.php
index.php
```

Создайте в child тот файл, который хотите переопределить — скопируйте из родителя и измените.

```php
<?php
/**
 * single.php — дочерняя тема
 */
get_header();
?>
<main id="primary" class="site-main">
    <?php
    while (have_posts()) :
        the_post();
        get_template_part('template-parts/content', 'single-custom');
    endwhile;
    ?>
</main>
<?php
get_footer();
```

---

### 1.5 template-parts и get_template_part()

```php
// Ищет: child/template-parts/content-card.php
//        → parent/template-parts/content-card.php
get_template_part('template-parts/content', 'card');
```

В child достаточно положить только изменённый partial — остальное возьмётся из parent.

---

### 1.6 functions.php дочерней темы

Все хуки WordPress можно вешать из child `functions.php`:

```php
<?php
// Регистрация меню
add_action('after_setup_theme', function () {
    register_nav_menus([
        'footer' => __('Footer Menu', 'my-shop-child'),
    ]);
});

// Поддержка миниатюр, title-tag и т.д.
add_action('after_setup_theme', function () {
    add_theme_support('woocommerce');
    add_theme_support('post-thumbnails');
});

// Фильтр excerpt
add_filter('excerpt_length', function () {
    return 25;
});
```

> **Не дублируйте** большие куски PHP из родителя — только **различия** и переопределения.

---

### 1.7 Переопределение функций родителя

Если в родителе функция обёрнута в `function_exists()`:

```php
// Родитель
if (!function_exists('my_shop_header_cart')) {
    function my_shop_header_cart() { /* ... */ }
}
```

В child **до** загрузки родителя (в `functions.php`) объявите свою версию:

```php
function my_shop_header_cart(): void
{
    echo '<div class="custom-cart">...</div>';
}
```

Если родитель **не** проверяет `function_exists` — переопределить нельзя; используйте **фильтры/хуки** родительской темы или копируйте шаблон.

---

### 1.8 Локализация (переводы) в child

```php
add_action('after_setup_theme', function () {
    load_child_theme_textdomain(
        'my-shop-child',
        get_stylesheet_directory() . '/languages'
    );
});
```

Файлы: `languages/my-shop-child-ru_RU.po` / `.mo`.

---

### 1.9 Block Themes (FSE) и child

Для тем на блоках (Full Site Editing) дочерняя тема содержит:

```
my-child/
├── style.css
├── functions.php
├── theme.json          ← переопределение настроек (цвета, шрифты)
└── templates/          ← кастомные HTML-шаблоны
    └── single.html
```

```json
{
  "$schema": "https://schemas.wp.org/trunk/theme.json",
  "version": 3,
  "settings": {
    "color": {
      "palette": [
        { "slug": "primary", "color": "#1a1a2e", "name": "Primary" }
      ]
    }
  }
}
```

Родитель задаётся тем же полем `Template` в `style.css`.

---

### 1.10 Чеклист best practices

| ✅ Делать | ❌ Не делать |
|----------|-------------|
| Править только child | Редактировать файлы родителя напрямую |
| Версионировать child в Git | Хранить секреты в теме |
| Минимум логики в теме | Плагиновая логика в `functions.php` темы |
| `wp_enqueue_style` с зависимостями | `@import` в CSS для родителя |
| Проверять `Template` slug | Опечатка в имени папки родителя |

### Источники (дочерние темы)

- [Theme Handbook: Child Themes](https://developer.wordpress.org/themes/advanced-topics/child-themes/)
- [Creating a Child Theme](https://developer.wordpress.org/themes/classic-themes/child-themes/)
- [wp_enqueue_style](https://developer.wordpress.org/reference/functions/wp_enqueue_style/)
- [Template Hierarchy](https://developer.wordpress.org/themes/basics/template-hierarchy/)
- [theme.json](https://developer.wordpress.org/themes/advanced-topics/theme-json/)

---

## 2. Создание плагина, жизненный цикл, соответствующие хуки

### Что такое плагин

**Плагин** — PHP-код в `wp-content/plugins/`, который расширяет WordPress **независимо от темы**. При смене темы плагин продолжает работать.

---

### 2.1 Минимальная структура плагина

```
wp-content/plugins/my-awesome-plugin/
├── my-awesome-plugin.php   ← главный файл (обязательные заголовки)
├── includes/
│   ├── class-loader.php
│   ├── class-admin.php
│   └── class-public.php
├── admin/
├── public/
├── assets/
│   ├── css/
│   └── js/
├── languages/
└── uninstall.php           ← очистка при удалении
```

---

### 2.2 Заголовок главного файла плагина

```php
<?php
/**
 * Plugin Name:       My Awesome Plugin
 * Plugin URI:        https://example.com/my-awesome-plugin
 * Description:       Расширенный функционал для магазина
 * Version:           1.0.0
 * Requires at least: 6.0
 * Requires PHP:      8.0
 * Author:            Your Name
 * Author URI:        https://example.com
 * License:           GPL v2 or later
 * License URI:       https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain:       my-awesome-plugin
 * Domain Path:       /languages
 */

defined('ABSPATH') || exit;

// Константы
define('MAP_VERSION', '1.0.0');
define('MAP_PLUGIN_DIR', plugin_dir_path(__FILE__));
define('MAP_PLUGIN_URL', plugin_dir_url(__FILE__));

// Автозагрузка (Composer) или require
require_once MAP_PLUGIN_DIR . 'includes/class-plugin.php';

function map_run_plugin(): void
{
    $plugin = new \MyAwesomePlugin\Plugin();
    $plugin->run();
}
map_run_plugin();
```

| Заголовок | Назначение |
|-----------|------------|
| `Plugin Name` | **Обязателен** — имя в админке |
| `Text Domain` | Для `__()`, `_e()` |
| `Version` | Версия плагина |

**Безопасность:** `defined('ABSPATH') || exit;` — запрет прямого вызова файла.

---

### 2.3 Жизненный цикл WordPress (общая схема)

Понимание порядка загрузки нужно, чтобы вешать код на **правильный хук**.

```
HTTP-запрос
    │
    ▼
wp-load.php → wp-config.php → wp-settings.php
    │
    ├── Загрузка must-use plugins (mu-plugins)
    ├── Загрузка active plugins (алфавит / порядок)
    │       └── plugins_loaded
    ├── pluggable.php
    ├── setup_theme → after_setup_theme
    ├── init  ← регистрация CPT, taxonomies, shortcodes
    ├── wp_loaded
    ├── parse_request, pre_get_posts (для фронта)
    ├── admin_menu (админка)
    ├── wp_enqueue_scripts (фронт) / admin_enqueue_scripts
    ├── template_redirect
    └── get_header → loop → get_footer → wp_footer
            │
            └── shutdown
```

---

### 2.4 Жизненный цикл плагина

| Этап | Когда | Типичные действия |
|------|-------|-------------------|
| **Файл подключён** | `include` главного PHP | Константы, autoload, **не** тяжёлая логика |
| **Активация** | Пользователь нажал «Активировать» | Таблицы БД, опции, cron, flush rewrite |
| **Работа** | Каждый запрос | Хуки `init`, `admin_menu`, … |
| **Деактивация** | «Деактивировать» | Очистка cron, временные опции |
| **Удаление** | «Удалить» плагин | `uninstall.php` — удаление данных |

```
Установлен (файлы на диске)
        │
        ▼ activate_plugin
   АКТИВЕН ─────────────────────► каждый HTTP-запрос
        │                              │
        │ deactivate_plugin            │
        ▼                              ▼
  ДЕАКТИВИРОВАН                    delete plugin
        │                              │
        └──────────────────────────────┴──► uninstall.php
```

---

### 2.5 Хуки активации / деактивации / удаления

```php
<?php
// Главный файл плагина

register_activation_hook(__FILE__, 'map_activate');
register_deactivation_hook(__FILE__, 'map_deactivate');

function map_activate(): void
{
    // Создание таблиц, опций по умолчанию
    global $wpdb;
    $table = $wpdb->prefix . 'map_log';
    $charset = $wpdb->get_charset_collate();

    $sql = "CREATE TABLE IF NOT EXISTS {$table} (
        id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
        message TEXT NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (id)
    ) {$charset};";

    require_once ABSPATH . 'wp-admin/includes/upgrade.php';
    dbDelta($sql);

    add_option('map_version', MAP_VERSION);
    flush_rewrite_rules();
}

function map_deactivate(): void
{
  wp_clear_scheduled_hook('map_daily_cleanup');
  flush_rewrite_rules();
}
```

**uninstall.php** (выполняется только при **удалении**, не при деактивации):

```php
<?php
// uninstall.php
defined('WP_UNINSTALL_PLUGIN') || exit;

delete_option('map_version');
global $wpdb;
$wpdb->query("DROP TABLE IF EXISTS {$wpdb->prefix}map_log");
```

> Данные при деактивации **часто сохраняют**; при uninstall — удаляют (или спрашивают пользователя).

---

### 2.6 Ключевые хуки и соответствие задачам

| Хук | Когда срабатывает | Примеры использования |
|-----|-------------------|----------------------|
| **`plugins_loaded`** | Все плагины загружены | Зависимости между плагинами, `class_exists` |
| **`after_setup_theme`** | Тема инициализирована | `add_theme_support` из плагина (редко) |
| **`init`** | WP полностью загружен, пользователь определён | CPT, taxonomies, `register_shortcode` |
| **`admin_init`** | Админка инициализирована | Регистрация настроек `register_setting` |
| **`admin_menu`** | Построение меню админки | `add_menu_page`, `add_submenu_page` |
| **`wp_enqueue_scripts`** | Фронт: подключение CSS/JS | `wp_enqueue_script` |
| **`admin_enqueue_scripts`** | Админка: CSS/JS | Скрипты только на своей странице |
| **`rest_api_init`** | REST API | `register_rest_route` |
| **`save_post`** | Сохранение записи | Мета-боксы, синхронизация |
| **`template_redirect`** | До вывода шаблона | Редиректы, доступ |
| **`wp_footer`** | Перед `</body>` | Inline-скрипты, аналитика |
| **`shutdown`** | Конец запроса | Логирование, отложенные задачи |

#### Приоритет и аргументы

```php
add_action('init', 'map_register_cpt', 10, 0);
add_filter('the_content', 'map_append_banner', 20, 1);
//                              callback    priority  accepted_args
```

Меньше **priority** → раньше выполнится (по умолчанию `10`).

---

### 2.7 Пример: регистрация CPT на `init`

```php
add_action('init', 'map_register_book_cpt');

function map_register_book_cpt(): void
{
    register_post_type('book', [
        'labels' => [
            'name'          => __('Books', 'my-awesome-plugin'),
            'singular_name' => __('Book', 'my-awesome-plugin'),
        ],
        'public'       => true,
        'has_archive'  => true,
        'rewrite'      => ['slug' => 'books'],
        'supports'     => ['title', 'editor', 'thumbnail'],
        'show_in_rest' => true,
    ]);
}
```

После изменения rewrite — `flush_rewrite_rules()` на активации.

---

### 2.8 Пример: страница настроек в админке

```php
add_action('admin_menu', 'map_add_settings_page');
add_action('admin_init', 'map_register_settings');

function map_add_settings_page(): void
{
    add_options_page(
        __('MAP Settings', 'my-awesome-plugin'),
        __('MAP', 'my-awesome-plugin'),
        'manage_options',
        'map-settings',
        'map_render_settings_page'
    );
}

function map_register_settings(): void
{
    register_setting('map_options_group', 'map_api_key', [
        'type'              => 'string',
        'sanitize_callback' => 'sanitize_text_field',
    ]);
}
```

---

### 2.9 Actions vs Filters в плагине

```php
// Action — выполнить код
add_action('user_register', function (int $user_id) {
    update_user_meta($user_id, 'welcome_sent', 0);
});

// Filter — изменить значение
add_filter('the_title', function (string $title, int $id) {
    if (get_post_type($id) === 'book') {
        return '📖 ' . $title;
    }
    return $title;
}, 10, 2);
```

---

### 2.10 AJAX в плагине

```php
add_action('wp_ajax_map_save_item', 'map_ajax_save_item');
add_action('wp_ajax_nopriv_map_save_item', 'map_ajax_save_item'); // для гостей — осторожно

function map_ajax_save_item(): void
{
    check_ajax_referer('map_nonce', 'nonce');

    if (!current_user_can('edit_posts')) {
        wp_send_json_error(['message' => 'Forbidden'], 403);
    }

    $id = absint($_POST['id'] ?? 0);
    // ...

    wp_send_json_success(['id' => $id]);
}
```

```javascript
// localized script
jQuery.post(map_ajax.url, {
  action: 'map_save_item',
  nonce: map_ajax.nonce,
  id: 123
});
```

---

### 2.11 Структура класса плагина (ООП)

```php
<?php
namespace MyAwesomePlugin;

final class Plugin
{
    public function run(): void
    {
        $this->load_dependencies();
        $this->define_admin_hooks();
        $this->define_public_hooks();
    }

    private function define_admin_hooks(): void
    {
        $admin = new Admin();
        add_action('admin_menu', [$admin, 'add_menu']);
        add_action('admin_enqueue_scripts', [$admin, 'enqueue_assets']);
    }

    private function define_public_hooks(): void
    {
        $public = new PublicFacing();
        add_action('wp_enqueue_scripts', [$public, 'enqueue_assets']);
        add_action('init', [$public, 'register_shortcodes']);
    }
}
```

---

### 2.12 Сводная таблица: этап → хук

| Задача плагина | Хук / API |
|----------------|-----------|
| Таблица при установке | `register_activation_hook` + `dbDelta` |
| Очистка при удалении | `uninstall.php` |
| Тип записи | `init` → `register_post_type` |
| Меню админки | `admin_menu` |
| Настройки | `admin_init` → `register_setting` |
| Скрипты фронта | `wp_enqueue_scripts` |
| REST endpoint | `rest_api_init` |
| После сохранения поста | `save_post` |
| Планировщик | `wp_schedule_event` + custom action |

---

### 2.13 Чеклист best practices для плагинов

| ✅ | ❌ |
|----|-----|
| Префикс функций `map_` | Глобальные имена `save()` |
| Nonce + `current_user_can` | Доверять `$_POST` |
| `sanitize_*`, `esc_*` | Сырой вывод |
| Текстовый домен для переводов | Хардкод без i18n |
| `uninstall.php` для данных | Мусор в БД после удаления |
| Composer PSR-4 | Всё в одном файле на 3000 строк |

### Источники (плагины)

- [Plugin Handbook](https://developer.wordpress.org/plugins/)
- [Creating a Plugin](https://developer.wordpress.org/plugins/plugin-basics/)
- [register_activation_hook](https://developer.wordpress.org/reference/functions/register_activation_hook/)
- [Running a Plugin](https://developer.wordpress.org/plugins/plugin-basics/header-requirements/)
- [Hooks: Actions and Filters](https://developer.wordpress.org/plugins/hooks/)
- [Plugin API / Action Reference](https://developer.wordpress.org/reference/hooks/)
- [Writing a Plugin Uninstall Script](https://developer.wordpress.org/plugins/plugin-basics/uninstall-hooks/)

---

## Шпаргалка перед экзаменом

| Вопрос | Ответ |
|--------|--------|
| Зачем child theme? | Не терять правки при обновлении родителя |
| Поле `Template` | Slug папки **родительской** темы |
| `get_template_*` vs `get_stylesheet_*` | Родитель vs активная (child) тема |
| Главный файл плагина | Заголовок `Plugin Name` |
| Когда создавать таблицы? | `register_activation_hook` |
| Когда удалять данные? | `uninstall.php` |
| CPT регистрировать на | `init` |
| Плагины загружены | `plugins_loaded` |

### Связь темы и плагина на экзамене

```
Дочерняя тема     →  внешний вид, шаблоны, style.css
Плагин            →  функции, CPT, админка, API
Хуки              →  точки расширения и в теме, и в плагине
```

---

*Конспект подготовлен для экзамена «WordPress продвинутый». Официальная документация: [developer.wordpress.org](https://developer.wordpress.org/).*
