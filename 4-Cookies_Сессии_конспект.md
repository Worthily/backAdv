# Cookies и сессии — экзаменационный конспект

> **Цель:** подготовка к экзамену по механизмам сохранения состояния в веб-приложениях.  
> **Контекст:** HTTP, PHP, браузер и сервер.  
> **Как пользоваться:** каждый раздел — отдельный экзаменационный вопрос; переходите по оглавлению.

---

## Оглавление

1. [Что такое cookies и где хранятся](#1-что-такое-cookies-и-где-хранятся)
2. [Откуда и как можно обратиться к cookies](#2-откуда-и-как-можно-обратиться-к-cookies)
3. [Что такое сессии и где хранятся](#3-что-такое-сессии-и-где-хранятся)
4. [Взаимодействие сессий с cookies](#4-взаимодействие-сессий-с-cookies)

---

## Введение: почему cookies и сессии нужны

Протокол **HTTP** по умолчанию **без состояния (stateless)**: каждый запрос независим, сервер не «помнит» предыдущие обращения. Для входа в личный кабинет, корзины, языка интерфейса нужен способ **идентифицировать пользователя** между запросами.

| Механизм | Где «живёт» состояние | Типичное применение |
|----------|----------------------|---------------------|
| **Cookie** | Браузер (клиент) | Идентификатор сессии, «Запомнить меня», настройки UI |
| **Сессия** | Сервер (или хранилище сервера) | Данные авторизации, корзина, flash-сообщения |

```
┌──────────┐   HTTP Request (+ Cookie)    ┌──────────┐
│ Браузер  │ ───────────────────────────► │  Сервер  │
│ cookies  │ ◄─────────────────────────── │ session  │
└──────────┘   HTTP Response (Set-Cookie)  └──────────┘
```

---

## 1. Что такое cookies и где хранятся

### Определение

**Cookie (куки)** — небольшой фрагмент данных, который **сервер отправляет браузеру** в заголовке `Set-Cookie`, а браузер **сохраняет** и **автоматически отправляет обратно** в заголовке `Cookie` при последующих запросах к тому же домену (с учётом правил path, domain, Secure, SameSite).

Cookie — это **не** сессия целиком, а чаще **ключ** или **настройка** на стороне клиента.

### Где хранятся cookies

| Место | Описание |
|-------|----------|
| **Браузер пользователя** | Основное хранилище: внутренняя БД браузера (Chrome — SQLite, Firefox — и т.д.) |
| **Не на сервере** | Сервер не хранит файл cookie; он только **читает** то, что прислал браузер |
| **Память (session cookie)** | Если нет `Expires` / `Max-Age` — cookie удаляется при закрытии браузера |
| **Диск (persistent cookie)** | С датой истечения — сохраняется между перезапусками браузера |

> **На экзамене:** cookies хранятся **на клиенте (в браузере)**. Сервер может **записать** cookie в ответе, но физически файл/запись — у пользователя.

### Структура cookie (логически)

| Поле | Смысл |
|------|-------|
| **Имя** | Например `session_id`, `lang` |
| **Значение** | Строка (часто идентификатор или JSON) |
| **Domain** | Для какого домена действует (`example.com`) |
| **Path** | Для каких URL (`/`, `/admin`) |
| **Expires / Max-Age** | Срок жизни |
| **Secure** | Только по HTTPS |
| **HttpOnly** | Недоступно из JavaScript (защита от XSS) |
| **SameSite** | `Strict`, `Lax`, `None` — защита от CSRF |

### Пример HTTP-заголовков

**Сервер устанавливает cookie:**

```http
HTTP/1.1 200 OK
Set-Cookie: lang=ru; Path=/; Max-Age=31536000; SameSite=Lax
Set-Cookie: PHPSESSID=abc123xyz; Path=/; HttpOnly; Secure; SameSite=Lax
```

**Браузер отправляет cookie обратно:**

```http
GET /catalog HTTP/1.1
Host: example.com
Cookie: lang=ru; PHPSESSID=abc123xyz
```

### Ограничения

| Ограничение | Типичное значение |
|-------------|-------------------|
| Размер одной cookie | ~4 КБ |
| Количество на домен | Зависит от браузера (порядка 50–180+) |
| Видимость | Только тот домен/path, для которого выдана |

### Типы cookies по назначению

| Тип | Пример | Особенность |
|-----|--------|-------------|
| **Сессионная** | Временный `cart_step` | Без срока — до закрытия браузера |
| **Постоянная** | `remember_token` | `Max-Age` / `Expires` |
| **Сторонняя (third-party)** | Реклама, аналитика | Домен ≠ домен страницы; ограничивается браузерами |
| **First-party** | Свой сайт | Домен совпадает с сайтом |

### Установка cookie в PHP

```php
<?php
// Простая cookie на 30 дней
setcookie('lang', 'ru', [
    'expires'  => time() + 60 * 60 * 24 * 30,
    'path'     => '/',
    'domain'   => '',           // текущий хост
    'secure'   => true,         // только HTTPS
    'httponly' => false,        // lang можно читать из JS (если нужно)
    'samesite' => 'Lax',
]);

// Сессионная cookie (до закрытия браузера)
setcookie('visited', '1', [
    'expires'  => 0,
    'path'     => '/',
    'httponly' => true,
    'samesite' => 'Strict',
]);
```

> **Важно:** `setcookie()` нужно вызывать **до** любого вывода в тело ответа (до `echo`, HTML), иначе заголовки уже отправлены.

### Источники

- [MDN: HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [MDN: Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie)
- [PHP: setcookie](https://www.php.net/manual/ru/function.setcookie.php)
- [RFC 6265: HTTP State Management Mechanism](https://datatracker.ietf.org/doc/html/rfc6265)

---

## 2. Откуда и как можно обратиться к cookies

### Общая схема доступа

```
                    ┌─────────────────────────────────────┐
                    │           Браузер (клиент)           │
                    │  document.cookie (JS, если не HttpOnly) │
                    └─────────────────┬───────────────────┘
                                      │ Cookie: header
                                      ▼
                    ┌─────────────────────────────────────┐
                    │              Сервер (PHP)            │
                    │  $_COOKIE, $_SERVER['HTTP_COOKIE']   │
                    └─────────────────────────────────────┘
```

---

### 2.1 Доступ на сервере (PHP)

После того как браузер отправил заголовок `Cookie`, PHP автоматически разбирает его в суперглобальный массив **`$_COOKIE`**.

```php
<?php
// Чтение
$lang = $_COOKIE['lang'] ?? 'en';

// Проверка существования
if (isset($_COOKIE['visited'])) {
    echo 'Вы уже были на сайте';
}

// Удаление cookie — установить прошедшую дату
setcookie('visited', '', [
    'expires'  => time() - 3600,
    'path'     => '/',
    'httponly' => true,
]);
```

| Способ | Когда использовать |
|--------|-------------------|
| `$_COOKIE['name']` | Основной способ в PHP |
| `filter_input(INPUT_COOKIE, 'name', FILTER_SANITIZE_SPECIAL_CHARS)` | Безопасное чтение с фильтрацией |
| `$_SERVER['HTTP_COOKIE']` | Сырой заголовок (редко, для отладки) |

**Безопасность:** данные из cookie **контролирует пользователь** (можно подделать в DevTools). Не храните в cookie секреты без подписи и не доверяйте им для критичной авторизации без проверки на сервере.

```php
// Плохо: доверять cookie «is_admin=1»
// Хорошо: хранить на сервере в сессии, в cookie — только случайный id сессии
```

---

### 2.2 Доступ в браузере (JavaScript)

```javascript
// Чтение (только cookies БЕЗ флага HttpOnly)
console.log(document.cookie);
// "lang=ru; theme=dark"

// Запись (создаёт/перезаписывает cookie)
document.cookie = 'theme=dark; path=/; max-age=31536000; samesite=lax';

// Удаление — установить expires в прошлое
document.cookie = 'theme=; path=/; max-age=0';
```

| API | Особенности |
|-----|-------------|
| `document.cookie` | Строковый API, неудобен для парсинга |
| **Cookie Store API** | Современный async API (не везде поддерживается) |

```javascript
// Cookie Store API (если поддерживается)
if ('cookieStore' in window) {
  const lang = await cookieStore.get('lang');
  console.log(lang?.value);
}
```

> Cookies с **`HttpOnly`** (например `PHPSESSID`) **недоступны** из JavaScript — это защита от кражи сессии через XSS.

---

### 2.3 Доступ через инструменты разработчика

В Chrome / Firefox / Edge: **DevTools → Application (Хранилище) → Cookies → домен**.

Там видно имя, значение, domain, path, expires, HttpOnly, Secure, SameSite. Удобно для отладки, на экзамене можно упомянуть как «способ обращения со стороны разработчика».

---

### 2.4 Доступ из других технологий

| Среда | Как получить cookies |
|-------|----------------------|
| **PHP** | `$_COOKIE` |
| **Node.js (Express)** | `req.cookies` (с middleware `cookie-parser`) |
| **Python (Django)** | `request.COOKIES` |
| **cURL** | `curl -b cookies.txt` / `-H "Cookie: name=value"` |

```bash
# Отправка cookie вручную
curl -H "Cookie: lang=ru; PHPSESSID=abc123" https://example.com/page
```

---

### 2.5 Когда cookie отправляется браузером

Браузер прикрепляет cookie к запросу, если совпадают:

- **домен** (и поддомены, если задано `.example.com`);
- **path** (запрос начинается с path cookie);
- cookie **не истекла**;
- для `Secure` — запрос по **HTTPS**;
- правила **SameSite** (cross-site запросы могут не отправлять cookie).

---

### Источники

- [MDN: Document.cookie](https://developer.mozilla.org/en-US/docs/Web/API/Document/cookie)
- [PHP: $_COOKIE](https://www.php.net/manual/ru/reserved.variables.cookies.php)
- [OWASP: Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)

---

## 3. Что такое сессии и где хранятся

### Определение

**Сессия (session)** — механизм **хранения данных пользователя на сервере** (или в серверном хранилище) между HTTP-запросами. Клиенту выдаётся только **идентификатор сессии** (часто через cookie), а **содержимое** (логин, корзина, права) остаётся на сервере.

```
Браузер                          Сервер
────────                         ──────
Cookie: PHPSESSID=abc123   →     Файл/Redis: abc123 → { user_id: 5, cart: [...] }
```

### Где хранятся данные сессии

| Хранилище | Описание | Когда используют |
|-----------|----------|------------------|
| **Файлы на диске сервера** | По умолчанию в PHP: `session.save_path` | Малые и средние сайты, shared hosting |
| **Redis / Memcached** | Быстро, удобно для кластера | Production, несколько серверов |
| **База данных** | Таблица `sessions` | Когда уже есть БД и нужна персистентность |
| **Память процесса** | Встроенные сессии dev-сервера | Только разработка |

> **На экзамене:** **данные сессии** хранятся **на сервере** (или в хранилище, доступном серверу). **Cookie** с `PHPSESSID` — только **ключ**, не сами данные корзины/профиля.

### Жизненный цикл сессии в PHP

```php
<?php
// 1. Старт сессии (читает cookie, поднимает $_SESSION)
session_start();

// 2. Запись данных
$_SESSION['user_id'] = 42;
$_SESSION['username'] = 'anna';
$_SESSION['cart'] = ['item1', 'item2'];

// 3. Чтение
$userId = $_SESSION['user_id'] ?? null;

// 4. Удаление одного ключа
unset($_SESSION['cart']);

// 5. Уничтожение сессии полностью (выход)
$_SESSION = [];
if (ini_get('session.use_cookies')) {
    $params = session_get_cookie_params();
    setcookie(session_name(), '', time() - 42000,
        $params['path'], $params['domain'],
        $params['secure'], $params['httponly']
    );
}
session_destroy();
```

### Настройки PHP (php.ini / session_start options)

| Параметр | Назначение | Пример |
|----------|------------|--------|
| `session.name` | Имя cookie сессии | `PHPSESSID` |
| `session.save_path` | Папка файлов сессий | `/var/lib/php/sessions` |
| `session.gc_maxlifetime` | Время жизни данных (сек) | `1440` (24 мин) |
| `session.cookie_httponly` | HttpOnly для cookie сессии | `1` |
| `session.cookie_secure` | Только HTTPS | `1` |
| `session.cookie_samesite` | SameSite | `Lax` |

```php
session_start([
    'cookie_lifetime' => 0,
    'cookie_httponly' => true,
    'cookie_secure'   => true,
    'cookie_samesite' => 'Lax',
    'use_strict_mode' => true,  // отклонять неизвестные session id
]);
```

### Пример файла сессии (при handler `files`)

Путь: `session.save_path` + файл вида `sess_abc123xyz`

```
user_id|i:42;username|s:4:"anna";cart|a:2:{i:0;s:5:"item1";i:1;s:5:"item2";}
```

Формат — сериализованные PHP-данные. **Не редактировать вручную** в production.

### Сессия vs cookie (данные на клиенте)

| | Cookie (как хранилище данных) | Сессия |
|---|------------------------------|--------|
| **Где данные** | У пользователя в браузере | На сервере |
| **Размер** | ~4 КБ | Практически больше (ограничен сервером) |
| **Безопасность** | Пользователь видит и меняет | Пользователь видит только ID |
| **Нагрузка на сервер** | Меньше | Больше (хранение + GC) |

### Альтернативы классической PHP-сессии

| Подход | Суть |
|--------|------|
| **JWT в cookie** | Вся «сессия» подписана в токене на клиенте (stateless API) |
| **Database session** | Laravel `SESSION_DRIVER=database` |
| **Redis session** | Общее хранилище для нескольких воркеров |

```php
// Laravel .env (для справки)
// SESSION_DRIVER=file|redis|database|cookie
```

### Источники

- [PHP: Сессии](https://www.php.net/manual/ru/book.session.php)
- [PHP: session_start](https://www.php.net/manual/ru/function.session-start.php)
- [PHP: SessionHandlerInterface](https://www.php.net/manual/ru/class.sessionhandlerinterface.php) — своё хранилище
- [OWASP: Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)

---

## 4. Взаимодействие сессий с cookies

### Главная идея

**Сессия и cookie работают вместе**, но выполняют разные роли:

| Роль | Кто |
|------|-----|
| **Идентификатор** («пропуск») | Cookie `PHPSESSID` (или своё имя) |
| **Данные** (логин, корзина, CSRF-токен) | Массив `$_SESSION` на сервере |

Без cookie (или без передачи session id другим способом) сервер **не узнает**, какую сессию поднять, и создаст **новую** при каждом визите (если id не передан иным способом).

### Схема полного цикла

```
1. Первый визит
   Браузер ──GET /──► Сервер
   Сервер: session_start() → новый id "abc123"
   Сервер ◄──Set-Cookie: PHPSESSID=abc123── Браузер (сохранил)

2. Второй запрос
   Браузер ──GET /  Cookie: PHPSESSID=abc123──► Сервер
   Сервер: находит sess_abc123, загружает $_SESSION

3. Выход
   Сервер: session_destroy(), Set-Cookie: PHPSESSID=; expires в прошлом
   Браузер удаляет cookie, сервер удаляет файл сессии
```

### Пошаговый пример: авторизация

```php
<?php
// login.php
session_start();

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $user = authenticate($_POST['email'], $_POST['password']);

    if ($user) {
        // Данные — только в сессии на сервере
        $_SESSION['user_id'] = $user['id'];
        $_SESSION['role'] = $user['role'];

        // Опционально: отдельная cookie «запомнить меня» (долгоживущий токен)
        if (!empty($_POST['remember'])) {
            setcookie('remember_token', createRememberToken($user['id']), [
                'expires'  => time() + 60 * 60 * 24 * 30,
                'httponly' => true,
                'secure'   => true,
                'samesite' => 'Lax',
            ]);
        }

        header('Location: /dashboard.php');
        exit;
    }
}
```

```php
<?php
// dashboard.php
session_start();

if (empty($_SESSION['user_id'])) {
    header('Location: /login.php');
    exit;
}

echo 'Привет, пользователь #' . $_SESSION['user_id'];
```

```php
<?php
// logout.php
session_start();
$_SESSION = [];
session_destroy();

setcookie(session_name(), '', time() - 3600, '/'); // удалить cookie сессии
header('Location: /login.php');
```

### Способы передачи session id (не только cookie)

| Способ | Механизм | Замечание |
|--------|----------|-----------|
| **Cookie** | `PHPSESSID` в заголовке `Cookie` | Стандарт, по умолчанию в PHP |
| **URL** | `?PHPSESSID=abc123` | **Опасно** (утечка в логи, Referer); отключать |
| **POST / заголовок** | Редко в API | Для мобильных API иногда JWT вместо сессии |

```ini
; php.ini — не передавать id в URL
session.use_only_cookies = 1
session.use_trans_sid = 0
```

### Две cookie в одном приложении

Типичная картина на сайте:

```
Cookie: PHPSESSID=abc123; lang=ru; theme=dark
         │                │         │
         │                │         └── обычные настройки (можно HttpOnly=false)
         │                └── обычная cookie
         └── привязка к серверной сессии (HttpOnly=true)
```

| Cookie | Связь с сессией |
|--------|-----------------|
| `PHPSESSID` | **Связана напрямую** — ключ к `$_SESSION` |
| `lang`, `theme` | **Не связана** — независимые настройки |
| `remember_token` | **Косвенно** — при следующем визите сервер по токену восстанавливает сессию |

### «Запомнить меня» — типичное взаимодействие

```
1. Пользователь входит + галочка «Запомнить»
2. Сервер создаёт сессию (PHPSESSID) + долгую cookie remember_token
3. remember_token хранится в БД на сервере (хеш), не пароль
4. Сессия истекла, но remember_token есть → сервер создаёт новую сессию
```

```php
// Упрощённая логика при каждом запросе
session_start();

if (empty($_SESSION['user_id']) && isset($_COOKIE['remember_token'])) {
    $userId = validateRememberToken($_COOKIE['remember_token']);
    if ($userId) {
        $_SESSION['user_id'] = $userId; // новая/та же сессия «ожила»
    }
}
```

### Безопасность связки session + cookie

| Угроза | Защита |
|--------|--------|
| **XSS** (кража cookie) | `HttpOnly` на session cookie |
| **Перехват по сети** | `Secure`, HTTPS |
| **CSRF** | `SameSite`, CSRF-токен в `$_SESSION` |
| **Фиксация сессии** | `session_regenerate_id(true)` после входа |
| **Подделка cookie** | Не хранить в cookie `is_admin=1`; только server-side session |

```php
// После успешного входа — новый id сессии
session_regenerate_id(true);
$_SESSION['user_id'] = $user['id'];
```

### Сравнение: только cookie vs сессия + cookie

```php
// ❌ Вся «авторизация» в cookie (плохо)
setcookie('user', json_encode(['id' => 5, 'role' => 'admin']));
// Пользователь может изменить role в DevTools

// ✅ Сессия + cookie только с id
session_start();
$_SESSION['user_id'] = 5;
$_SESSION['role'] = 'admin'; // роль только на сервере
// Браузер знает только PHPSESSID
```

### Источники

- [PHP: session_set_cookie_params](https://www.php.net/manual/ru/function.session-set-cookie-params.php)
- [PHP: session_regenerate_id](https://www.php.net/manual/ru/function.session-regenerate-id.php)
- [MDN: Session cookies vs Permanent cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#define_the_lifetime_of_a_cookie)

---

## Шпаргалка перед экзаменом

| Вопрос | Краткий ответ |
|--------|---------------|
| Где хранятся **cookies**? | В **браузере** (клиент) |
| Где хранятся **данные сессии**? | На **сервере** (файлы, Redis, БД) |
| Зачем cookie при сессии? | Передать **session id**, чтобы сервер нашёл нужные данные |
| Как сервер читает cookie? | `$_COOKIE` (PHP), заголовок `Cookie` |
| Как JS читает cookie? | `document.cookie` (кроме HttpOnly) |
| Почему не класть секреты в cookie? | Пользователь может **изменить** значение |
| Что после `session_destroy()`? | Удалить данные на сервере **и** cookie с id |

### Типичные ошибки

| Ошибка | Правильно |
|--------|-----------|
| «Сессия хранится в cookie» | В cookie хранится только **идентификатор** |
| `echo` до `session_start()` | Сначала сессия/cookie, потом вывод |
| Доверять `$_COOKIE['role']` | Роль хранить в `$_SESSION` на сервере |
| Не вызывать `session_regenerate_id` после логина | Риск фиксации сессии |

---

*Конспект подготовлен для экзамена «Cookies, Сессии». Примеры на PHP; принципы HTTP универсальны для любого backend.*
