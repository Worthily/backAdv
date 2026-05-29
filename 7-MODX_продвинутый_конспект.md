# MODX продвинутый — экзаменационный конспект

> **Цель:** подготовка к экзамену по MODX Revolution: сниппеты, miniShop2, FormIt, getResources.  
> **Версия:** MODX Revolution 3.x / 2.8.x, PHP 8.0+.  
> **Как пользоваться:** каждый раздел — отдельный вопрос; переходите по оглавлению.

---

## Оглавление

1. [Создание своих сниппетов](#1-создание-своих-сниппетов)
2. [Как работает miniShop2](#2-как-работает-minishop2)
3. [Как работает FormIt](#3-как-работает-formit)
4. [Как работает getResources](#4-как-работает-getresources)

---

## Введение: архитектура MODX Revolution

| Элемент | Назначение |
|---------|------------|
| **Resource** | Страница / документ в дереве сайта |
| **Template** | Обёртка HTML для ресурса |
| **Chunk** | Переиспользуемый HTML-фрагмент |
| **Snippet** | Исполняемый **PHP-код**, возвращающий результат |
| **Plugin** | PHP на **событиях** (OnWebPageInit, …) |
| **TV** (Template Variable) | Доп. поля ресурса |
| **Namespace / Extra** | Пакеты (miniShop2, FormIt, pdoTools) |

```
Запрос → MODX Core → Resource + Template
              │
              └── Парсинг тегов [[snippet]], [[$chunk]], [[*tv]]
                        │
                        └── Snippet (PHP) → HTML на выход
```

---

## 1. Создание своих сниппетов

### Что такое сниппет

**Сниппет (Snippet)** — элемент MODX с PHP-кодом, который вызывается в шаблоне, чанке или содержимом ресурса через тег:

```modx
[[MySnippet? &foo=`bar` &tpl=`itemTpl`]]
```

Сниппет **должен возвращать строку** (`return`), которую MODX подставит вместо тега. Вывод через `echo` без `return` тоже возможен, но **не рекомендуется** (проблемы с кэшем и вложенными вызовами).

---

### 1.1 Способы создания сниппета

| Способ | Где хранится | Когда использовать |
|--------|--------------|-------------------|
| **Менеджер** | БД (`modx_site_snippets`) | Быстрые правки, мало кода |
| **Статический файл** | `core/components/...` или `assets/snippets/` | Версионирование, Git, большие сниппеты |
| **В составе пакета** | Build через `resolvers` / `setup` | Extra для распространения |

**Создание в менеджере:** Элементы → Сниппеты → Создать сниппет.

| Поле | Значение |
|------|----------|
| **Имя** | `MySnippet` (латиница, без пробелов) |
| **Категория** | Для группировки |
| **Сниппет (код)** | PHP |
| **Свойства по умолчанию** | JSON или список `&param` |

---

### 1.2 Минимальный сниппет

```php
<?php
/**
 * MySnippet
 * @var modX $modx
 * @var array $scriptProperties
 */

$name = $modx->getOption('name', $scriptProperties, 'Гость');
$upper = (bool) $modx->getOption('upper', $scriptProperties, false);

$greeting = 'Привет, ' . $name . '!';

return $upper ? mb_strtoupper($greeting) : $greeting;
```

**Вызов:**

```modx
[[MySnippet? &name=`Анна` &upper=`1`]]
```

Результат: `ПРИВЕТ, АННА!`

---

### 1.3 $modx и $scriptProperties

| Переменная | Содержимое |
|------------|------------|
| **`$modx`** | Главный объект MODX API |
| **`$scriptProperties`** | Массив параметров из тега (`&foo` → `foo`) |

**Чтение параметров** — всегда через `getOption` с дефолтом:

```php
$parent = (int) $modx->getOption('parent', $scriptProperties, 0);
$tpl = $modx->getOption('tpl', $scriptProperties, 'itemTpl');
$limit = (int) $modx->getOption('limit', $scriptProperties, 10);
```

---

### 1.4 Работа с ресурсами и TV

```php
<?php
/** @var modX $modx */
$id = (int) $modx->getOption('id', $scriptProperties, $modx->resource->get('id'));

if ($resource = $modx->getObject('modResource', $id)) {
    $title = $resource->get('pagetitle');
    $price = $resource->getTVValue('price'); // TV с именем price

    return $modx->getChunk('priceTpl', [
        'title' => $title,
        'price' => $price,
    ]);
}

return '';
```

| API | Назначение |
|-----|------------|
| `$modx->getObject('modResource', $id)` | Загрузить ресурс |
| `$modx->resource` | Текущий ресурс страницы |
| `getTVValue('name')` | Значение TV |
| `$modx->getChunk('tpl', $placeholders)` | Парсинг чанка |

---

### 1.5 xPDO Query в сниппете

```php
<?php
$c = $modx->newQuery('modResource');
$c->where([
    'parent' => 5,
    'published' => 1,
    'deleted' => 0,
]);
$c->sortby('publishedon', 'DESC');
$c->limit(5);

$resources = $modx->getCollection('modResource', $c);
$output = '';

foreach ($resources as $resource) {
    $output .= $modx->getChunk('rowTpl', [
        'id'    => $resource->get('id'),
        'title' => $resource->get('pagetitle'),
        'url'   => $modx->makeUrl($resource->get('id')),
    ]);
}

return $output;
```

---

### 1.6 Статический сниппет (файл)

1. Файл: `core/components/mysite/snippets/mysnippet.snippet.php`
2. В менеджере: сниппет **MySnippet** → галочка **Статический элемент** → путь к файлу.

Или регистрация при установке пакета через **Vehicle** / XML в build.

**Преимущества:** Git, IDE, автодополнение, code review.

```php
<?php
// mysnippet.snippet.php — подключается MODX автоматически
return 'OK from file';
```

---

### 1.7 Свойства сниппета (Properties)

В менеджере задаются **default properties** — видны в UI при вставке сниппета:

| Имя | Тип | Default | Описание |
|-----|-----|---------|----------|
| parent | number | 0 | ID родителя |
| tpl | text | rowTpl | Чанк строки |
| limit | number | 10 | Лимит |

В коде те же ключи доступны в `$scriptProperties`.

---

### 1.8 Кэширование и производительность

```php
// Кэш результата на 1 час (3600 сек)
$cacheKey = 'mysnippet/' . md5(serialize($scriptProperties));
if ($cached = $modx->cacheManager->get($cacheKey)) {
    return $cached;
}

$output = buildHeavyOutput();
$modx->cacheManager->set($cacheKey, $output, 3600);

return $output;
```

| Правило | Зачем |
|---------|-------|
| `return` вместо `echo` | Корректная подстановка и кэш страницы |
| Не тяжёлая логика без кэша | Скорость страницы |
| `unset($modx->map...) ` | Редко; не загрязнять глобальное состояние |

---

### 1.9 Отладка сниппета

```php
$modx->log(modX::LOG_LEVEL_ERROR, 'MySnippet parent=' . $parent);
// Лог: core/cache/logs/error.log

// Только для менеджеров (осторожно на проде)
if ($modx->user->isMember('Administrator')) {
    return '<pre>' . print_r($scriptProperties, true) . '</pre>';
}
```

---

### 1.10 Чеклист best practices

| ✅ | ❌ |
|----|-----|
| `return` + `getOption` | Голый `$_GET` / `$_POST` |
| Экранирование в чанке `[[+title]]` | SQL без xPDO |
| Префикс имён `mySnippet` | Конфликт имён с core |
| Статический файл для >30 строк | Огромный код в БД |
| DocBlock `@var modX $modx` | Прямой `mysql_*` |

### Источники (сниппеты)

- [MODX Docs: Snippets](https://docs.modx.com/3.x/en/building-sites/elements/snippets)
- [MODX API: modX](https://docs.modx.com/3.x/en/extending-modx/modx-class)
- [xPDO Query](https://docs.modx.com/3.x/en/extending-modx/xpdo/class-reference/xpdoquery)
- [Creating a static element](https://docs.modx.com/3.x/en/building-sites/elements/static-elements)

---

## 2. Как работает miniShop2

### Что такое miniShop2

**miniShop2 (ms2)** — extra для MODX Revolution: **интернет-магазин**. Товары — это **ресурсы** специального класса `msProduct`, заказы хранятся в **таблицах БД**, корзина — в **сессии** или для авторизованных — в БД.

```
Каталог (ресурсы-контейнеры)
    └── msProduct (товар: цена, артикул, вес, …)
            │
            ▼
    msCart (сниппет) → сессия ms2_cart
            │
            ▼
    msOrder → msOrderAddress, msOrderProduct → оплата / статусы
```

---

### 2.1 Основные сущности

| Сущность | Класс / таблица | Описание |
|----------|-----------------|----------|
| **Товар** | `msProduct` extends `modResource` | Ресурс + поля ms2 |
| **Категория** | `msCategory` | Родитель для товаров |
| **Опции** | `msOption`, `msProductOption` | Размер, цвет (не только TV) |
| **Корзина** | Сессия `$_SESSION['minishop2']['cart']` | Массив id => count |
| **Заказ** | `msOrder` | Шапка заказа |
| **Позиции** | `msOrderProduct` | Товары в заказе |
| **Доставка / оплата** | `msDelivery`, `msPayment` | Способы |

---

### 2.2 Структура пакета

```
core/components/minishop2/
├── model/minishop2/       ← схема xPDO, классы msProduct, msOrder
├── processors/mgr/        ← админка (CRUD заказов, настройки)
├── controllers/           ← контроллеры фронта (корзина, заказ)
├── elements/
│   ├── snippets/          ← msProducts, msCart, msOrder, …
│   ├── chunks/            ← tpl корзины, писем
│   └── plugins/           ← события ms2
└── lexicon/
```

**Путь заказа на фронте (упрощённо):**

1. `[[!msProducts]]` — каталог  
2. `[[!msCart]]` — корзина + AJAX `action=cart/add`  
3. `[[!msOrder]]` — оформление, создание `msOrder`  
4. Плагины оплаты (Robokassa, и т.д.) — отдельные extras  

`!` — **некэшируемый** вызов (обязателен для корзины и форм).

---

### 2.3 Сниппет msProducts — вывод каталога

```modx
[[!msProducts?
    &parents=`5,6,7`
    &depth=`10`
    &tpl=`tpl.msProducts.row`
    &limit=`12`
    &sortby=`pagetitle`
    &sortdir=`ASC`
    &includeThumbs=`medium`
]]
```

| Параметр | Смысл |
|----------|-------|
| `parents` | ID категорий |
| `tpl` | Чанк карточки товара |
| `includeThumbs` | Превью из ms2 / Media Sources |
| `where` | JSON-фильтр xPDO |
| `optionFilters` | Фильтр по опциям |

**Чанк строки** (`tpl.msProducts.row`):

```html
<div class="product" itemscope itemtype="https://schema.org/Product">
    <a href="[[+uri]]">
        <img src="[[+thumb.medium]]" alt="[[+pagetitle]]">
        <h3>[[+pagetitle]]</h3>
    </a>
    <span class="price">[[+price]] [[%ms2_frontend_currency]]</span>
    <form method="post" class="ms2_form">
        <input type="hidden" name="id" value="[[+id]]">
        <input type="hidden" name="count" value="1">
        <input type="hidden" name="options" value="{}">
        <button type="submit" name="ms2_action" value="cart/add">В корзину</button>
    </form>
</div>
```

---

### 2.4 Корзина msCart

```modx
[[!msCart?
    &tpl=`tpl.msCart`
    &rowTpl=`tpl.msCart.row`
]]
```

**Как добавление работает внутри:**

1. POST с `ms2_action=cart/add` и `id` товара  
2. Плагин / контроллер ms2 перехватывает запрос  
3. Обновляет `$_SESSION['minishop2']['cart']`  
4. Возвращает JSON (при AJAX) или редирект  

```php
// Упрощённая логика (концептуально)
$cart = $_SESSION['minishop2']['cart'] ?? [];
$cart[$productId] = ($cart[$productId] ?? 0) + $count;
$_SESSION['minishop2']['cart'] = $cart;
```

---

### 2.5 Оформление заказа msOrder

```modx
[[!msOrder?
    &tpl=`tpl.msOrder`
    &userFields=`email,phone,receiver`
    &deliveryFields=`city,street`
]]
```

Этапы:

1. Пользователь заполняет форму (чанк)  
2. Выбирает `delivery` и `payment`  
3. ms2 создаёт запись `msOrder`, переносит позиции из корзины в `msOrderProduct`  
4. Очищает корзину  
5. Плагины отправляют письма (`msOnCreateOrder`, `msOnSubmitOrder`)  

---

### 2.6 События miniShop2 (для кастомизации)

| Событие | Когда |
|---------|-------|
| `msOnBeforeAddToCart` | До добавления в корзину |
| `msOnAddToCart` | После добавления |
| `msOnBeforeCreateOrder` | До создания заказа |
| `msOnCreateOrder` | Заказ создан |
| `msOnChangeOrderStatus` | Смена статуса |

**Плагин-пример:**

```php
<?php
/** @var msOrder $order */
switch ($modx->event->name) {
    case 'msOnCreateOrder':
        $modx->log(1, 'New order #' . $order->get('id'));
        break;
}
```

---

### 2.7 Цены, опции, модификации

| Механизм | Назначение |
|----------|------------|
| **Поле `price`** | Базовая цена msProduct |
| **Опции ms2** | Атрибуты с модификатором цены |
| **TV** | Доп. поля (бренд, badge) — совместимо |
| **Валюта** | Настройки ms2, сниппет `msPrice` |

---

### 2.8 Админка и процессоры

Заказы: **Компоненты → miniShop2 → Заказы**.  
CRUD через **processors** (`mgr/orders/getlist`, `update` и т.д.) — стандартный паттерн MODX extras.

---

### 2.9 Типичные ошибки

| Проблема | Решение |
|----------|---------|
| Корзина не обновляется | Вызов с `[[!msCart]]`, проверить сессии |
| Товар не добавляется | Класс ресурса `msProduct`, форма `ms2_form` |
| Пустой каталог | `parents`, `published`, права доступа |
| Кэш старой цены | Сброс кэша, `[[!...]]` |

### Источники (miniShop2)

- [miniShop2 документация (modstore)](https://docs.modx.pro/components/minishop2/)
- [GitHub: modx-pro/miniShop2](https://github.com/modx-pro/miniShop2)
- [Сниппет msProducts](https://docs.modx.pro/components/minishop2/snippets/msproducts)
- [Разработка дополнений ms2](https://docs.modx.com/3.x/en/extending-modx/developing-add-ons)

---

## 3. Как работает FormIt

### Что такое FormIt

**FormIt** — extra для **обработки HTML-форм** на фронте: валидация, anti-spam, хуки (email, сохранение в БД, интеграции), редиректы, сообщения об ошибках.

```
Страница с [[!FormIt?]]
        │
        ▼
   POST формы → FormIt snippet
        │
        ├── Валидаторы (required, email, …)
        ├── Hooks (email, redirect, save, custom)
        └── Плейсхолдеры fi.* в чанк формы
```

---

### 3.1 Базовый вызов

**Чанк формы** `tpl.contactForm`:

```html
[[!FormIt?
    &hooks=`email,redirect`
    &emailTpl=`contactEmailTpl`
    &emailTo=`manager@site.ru`
    &emailSubject=`Сообщение с сайта`
    &redirectTo=`25`
    &validate=`name:required,email:email:required,message:required`
    &validationErrorMessage=`Проверьте поля формы`
    &successMessage=`Спасибо! Мы ответим вам soon.`
]]

<form action="[[~[[*id]]]]" method="post">
    <input type="text" name="name" value="[[!+fi.name]]">
    [[!+fi.error.name]]

    <input type="email" name="email" value="[[!+fi.email]]">
    [[!+fi.error.email]]

    <textarea name="message">[[!+fi.message]]</textarea>
    [[!+fi.error.message]]

    <input type="hidden" name="nospam" value="">
    [[!+fi.error.nospam]]

    <button type="submit">Отправить</button>
</form>
```

| Параметр | Смысл |
|----------|-------|
| `hooks` | Цепочка обработчиков после успешной валидации |
| `validate` | Правила `поле:валидатор:валидатор` |
| `emailTpl` | Чанк письма |
| `redirectTo` | ID ресурса для редиректа |
| `[[!+fi.name]]` | Значение поля после отправки |
| `[[!+fi.error.name]]` | Текст ошибки валидации |

---

### 3.2 Жизненный цикл FormIt

```
1. GET  → форма пустая (или fi.* из предыдущего failed POST)
2. POST → FormIt получает поля
3. Запуск валидаторов
4a. Ошибка → fi.error.*, fi.validation_error_message
4b. Успех  → по очереди hooks → fi.success, redirect
```

---

### 3.3 Валидаторы (встроенные)

| Валидатор | Пример | Описание |
|-----------|--------|----------|
| `required` | `name:required` | Не пусто |
| `email` | `email:email` | Формат email |
| `minLength` | `pass:minLength=8` | Мин. длина |
| `maxLength` | `text:maxLength=500` | Макс. длина |
| `matches` | `pass_confirm:matches=password` | Совпадение полей |
| `regexp` | `phone:regexp` + `&regexp.phone` | Свой regex |

```modx
&validate=`name:required:minLength=2,email:email:required,phone:regexp`
&regexp.phone=`/^\+7\d{10}$/`
```

---

### 3.4 Hooks (цепочка хуков)

| Hook | Назначение |
|------|------------|
| `email` | Отправка письма |
| `redirect` | Редирект на `redirectTo` |
| `saveForm` | Сохранение (FormIt сохраняет в таблицу при настройке) |
| `spam` | Anti-spam проверки |
| `recaptcha` | reCAPTCHA (отдельный hook) |
| `fiHooks` | Кастомный сниппет-хук |

**Порядок:** хуки выполняются **слева направо** из `&hooks`:

```modx
&hooks=`spam,email,redirect`
```

Если hook вернёт `false` — цепочка прервётся.

---

### 3.5 Кастомный хук (сниппет)

Сниппет `myFormHook`:

```php
<?php
/** @var modX $modx */
/** @var fiHooks $hook */

$email = $hook->getValue('email');
$name = $hook->getValue('name');

// Запись в свою таблицу через xPDO
$modx->log(modX::LOG_LEVEL_INFO, "Form submit: {$name} <{$email}>");

// Можно установить поле для чанка
$hook->setValue('custom_field', 'processed');

// false — остановить цепочку hooks
return true;
```

Вызов:

```modx
&hooks=`myFormHook,email,redirect`
```

---

### 3.6 Чанк письма emailTpl

```html
<p><strong>Имя:</strong> [[+name]]</p>
<p><strong>Email:</strong> [[+email]]</p>
<p><strong>Сообщение:</strong></p>
<p>[[+message]]</p>
```

Плейсхолдеры — **имена полей формы** (`name`, `email`, …).

---

### 3.7 AJAX и FormIt

FormIt рассчитан на классический POST. Для AJAX часто:

- отдельный **коннектор** / сниппет с `return json_encode`, или  
- hook, который при `$_SERVER['HTTP_X_REQUESTED_WITH']` отдаёт JSON.

Для сложного API иногда используют **FormIt + fetch** или кастомный processor.

---

### 3.8 FormIt vs miniShop2

| | FormIt | miniShop2 msOrder |
|---|--------|-------------------|
| Назначение | Универсальные формы | Заказ магазина |
| Корзина | Нет | Да |
| Заказы в ms2 | Нет | Да |

Форму «заказ в один клик» можно собрать на FormIt, но **полноценный магазин** — через ms2.

### Источники (FormIt)

- [FormIt Docs (MODX.com)](https://docs.modx.com/extras/revo/formit)
- [FormIt Hooks](https://docs.modx.com/extras/revo/formit/formit.hooks)
- [Validators](https://docs.modx.com/extras/revo/formit/formit.validators)
- [GitHub: modxcms/FormIt](https://github.com/modxcms/FormIt)

---

## 4. Как работает getResources

### Что такое getResources

**getResources** — сниппет для **выборки и вывода списка ресурсов** (документов) по условиям: родитель, шаблон, TV, даты, поиск. Один из самых используемых сниппетов в MODX наряду с pdoTools.

> Часто на сайтах стоит **pdoTools** и сниппет **`pdoResources`** — быстрая альтернатива с тем же назначением. На экзамене знайте **логику getResources**; синтаксис `pdoResources` очень похож.

---

### 4.1 Задача, которую решает

Без PHP в шаблоне вывести:

- новости из папки `parent=10`;
- 5 последних статей с TV `image`;
- ресурсы с шаблоном `id=5`;
- теги / фильтры через TV.

```
[[getResources? &parents=`10` &tpl=`newsRow` &limit=`5`]]
        │
        ▼
   xPDO query modResource (+ TVs)
        │
        ▼
   Для каждого ресурса → парсинг чанка tpl
        │
        ▼
   Склеенный HTML
```

---

### 4.2 Базовый пример

```modx
[[getResources?
    &parents=`10,11`
    &depth=`2`
    &tpl=`newsRowTpl`
    &limit=`10`
    &offset=`0`
    &sortby=`publishedon`
    &sortdir=`DESC`
    &includeContent=`1`
    &includeTVs=`1`
    &processTVs=`1`
]]
```

**Чанк `newsRowTpl`:**

```html
<article class="news-item">
    <h2><a href="[[+uri]]">[[+pagetitle]]</a></h2>
    <time>[[+publishedon:strtotime:date=`%d.%m.%Y`]]</time>
    <div class="intro">[[+introtext]]</div>
    [[+tv.image:notempty=`<img src="[[+tv.image]]" alt="">`]]
</article>
```

---

### 4.3 Ключевые параметры

| Параметр | Описание |
|----------|----------|
| `parents` | ID родителей (через запятую) |
| `depth` | Глубина вложенности (0 = только parents) |
| `resources` | Конкретные ID (через запятую) |
| `tpl` | Чанк одной строки |
| `tplWrapper` | Обёртка вокруг всех строк (`[[+output]]`) |
| `limit` / `offset` | Пагинация |
| `sortby` / `sortdir` | Сортировка |
| `where` | JSON для xPDO `WHERE` |
| `templates` | Фильтр по ID шаблонов |
| `includeTVs` | Подгрузить TV (`1` или список имён) |
| `tvFilters` | Фильтр по значениям TV |
| `showHidden` | Показывать скрытые в меню |
| `showUnpublished` | Черновики (права!) |
| `context` | Ключ контекста |

---

### 4.4 Фильтр where (JSON)

```modx
[[getResources?
    &parents=`10`
    &tpl=`itemTpl`
    &where=`{"template":5,"published":1}`
]]
```

Комбинация с TV:

```modx
&tvFilters=`tag==новости,tag==акция`
```

или (в зависимости от версии сниппета):

```modx
&tvFilters=`tag:=новости`
```

---

### 4.5 tplWrapper — обёртка списка

```modx
[[getResources?
    &parents=`10`
    &tpl=`rowTpl`
    &tplWrapper=`listWrapperTpl`
]]
```

**listWrapperTpl:**

```html
<ul class="news-list">
[[+output]]
</ul>
```

---

### 4.6 Несколько чанков (tplCondition)

```modx
&tpl=`@INLINE <div>[[+pagetitle]]</div>`
```

или разные tpl по условию (в getResources / pdoResources через `tplCondition`, `conditionalTpl` в pdoTools).

**pdoResources** пример условного tpl:

```modx
[[pdoResources?
    &parents=`10`
    &tpl=`rowTpl`
    &tplCondition=`template`
    &tplOperator=`==`
    &conditionalTpls=`5==videoTpl,6==articleTpl`
]]
```

---

### 4.7 Внутренняя логика (концептуально)

```php
<?php
// Упрощённая схема getResources
$parents = explode(',', $modx->getOption('parents', $scriptProperties, ''));
$limit = (int) $modx->getOption('limit', $scriptProperties, 10);

$c = $modx->newQuery('modResource');
$c->where([
    'parent:IN' => $parents,
    'published' => 1,
    'deleted' => 0,
]);
$c->sortby('publishedon', 'DESC');
$c->limit($limit);

$collection = $modx->getCollection('modResource', $c);
$output = '';

foreach ($collection as $resource) {
    $placeholders = $resource->toArray();
    // + TV, uri, thumbnails
    $output .= $modx->getChunk($tpl, $placeholders);
}

return $output;
```

Реальный сниппет сложнее: кэш, права, контексты, объединение parents/depth.

---

### 4.8 getResources vs pdoResources

| | getResources | pdoResources (pdoTools) |
|---|--------------|-------------------------|
| Пакет | Отдельный / в сборках | pdoTools |
| Скорость | Хорошая | Обычно **быстрее** на больших выборках |
| Синтаксис | `[[getResources?` | `[[pdoResources?` |
| Fenom | Нет | Поддержка `@FILE` шаблонов |

```modx
[[pdoResources?
    &parents=`10`
    &tpl=`@FILE chunks/news/row.tpl`
    &limit=`12`
    &includeTVs=`image,author`
]]
```

---

### 4.9 Кэширование

```modx
[[getResources?
    &parents=`10`
    &tpl=`rowTpl`
    &cache=`3600`
    &cacheKey=`news-list-10`
]]
```

При изменении ресурса сбрасывать кэш или использовать короткий TTL.

---

### 4.10 Связь с miniShop2 и FormIt

| Задача | Инструмент |
|--------|------------|
| Список статей / новостей | **getResources** / pdoResources |
| Список товаров | **msProducts** (своя модель msProduct) |
| Форма обратной связи | **FormIt** |

Иногда каталог «услуг» делают на обычных ресурсах + getResources; товары магазина — только через ms2.

### Источники (getResources)

- [getResources (MODX.com)](https://docs.modx.com/extras/revo/getresources)
- [pdoTools / pdoResources](https://docs.modx.com/extras/revo/pdotools)
- [pdoTools на modx.pro](https://docs.modx.pro/components/pdotools/)
- [TV Filters](https://docs.modx.com/extras/revo/getresources#getresources.tvfilters)

---

## Шпаргалка перед экзаменом

| Вопрос | Ответ |
|--------|--------|
| Сниппет возвращает | `return $string` |
| Параметры в PHP | `$scriptProperties`, `getOption` |
| ms2 товар | Ресурс класса `msProduct` |
| Корзина ms2 | Сессия + `[[!msCart]]` |
| FormIt валидация | `&validate=` поле:правила |
| FormIt после успеха | `&hooks=` |
| getResources | Выборка ресурсов + чанк `tpl` |
| Некэшируемо | `[[!Snippet]]` |

### Сводная схема extras

```
┌─────────────────┐  ┌──────────────┐  ┌─────────────────┐
│  getResources   │  │   FormIt     │  │   miniShop2     │
│  списки страниц │  │  формы, email│  │  магазин, заказы│
└─────────────────┘  └──────────────┘  └─────────────────┘
         │                    │                  │
         └────────────────────┴──────────────────┘
                              │
                    MODX Core + xPDO + Snippets
```

---

*Конспект подготовлен для экзамена «MODX продвинутый». Актуальные версии extras смотрите на [docs.modx.com](https://docs.modx.com/) и [docs.modx.pro](https://docs.modx.pro/).*
