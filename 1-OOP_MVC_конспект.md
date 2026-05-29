# Понимание ООП и MVC — экзаменационный конспект

> **Цель:** подготовка к экзамену по объектно-ориентированному программированию и архитектуре MVC.  
> **Язык примеров:** PHP (актуален для большинства учебных курсов и веб-разработки).  
> **Как пользоваться:** каждый раздел — отдельный вопрос; читайте сверху вниз или переходите по оглавлению.

---

## Оглавление

1. [Суть ООП и парадигмы](#1-суть-ооп-и-парадигмы)
2. [Классы и методы](#2-классы-и-методы)
3. [Интерфейсы и абстрактные классы](#3-интерфейсы-и-абстрактные-классы)
4. [Область видимости](#4-область-видимости)
5. [Наследование и реализация интерфейсов](#5-наследование-и-реализация-интерфейсов)
6. [Пространства имён](#6-пространства-имён)
7. [Геттеры и сеттеры](#7-геттеры-и-сеттеры)
8. [Передача параметров по ссылке](#8-передача-параметров-по-ссылке)
9. [Исключения (try / catch)](#9-исключения-try--catch)
10. [Ключевые слова](#10-ключевые-слова)
11. [MVC-модель](#11-mvc-модель)

---

## 1. Суть ООП и парадигмы

### Что такое ООП

**Объектно-ориентированное программирование (ООП)** — парадигма, в которой программа строится из **объектов**: сущностей, объединяющих **данные (состояние)** и **поведение (методы)**. Вместо «глобальных функций + разрозненных переменных» код организуется вокруг сущностей предметной области: `User`, `Order`, `Product`.

### Четыре столпа ООП

| Принцип | Суть | Зачем |
|--------|------|-------|
| **Инкапсуляция** | Скрытие внутренней реализации, доступ через публичный API | Защита данных, упрощение изменений |
| **Наследование** | Класс-потомок расширяет или переопределяет родителя | Повторное использование кода |
| **Полиморфизм** | Один интерфейс — разные реализации | Гибкость, подмена реализаций |
| **Абстракция** | Выделение существенного, отбрасывание деталей | Упрощение модели предметной области |

### Парадигмы программирования (кратко)

| Парадигма | Идея | Пример |
|-----------|------|--------|
| **Процедурная** | Программа = последовательность процедур/функций | C, ранний PHP |
| **Объектно-ориентированная** | Программа = взаимодействие объектов | PHP, Java, C# |
| **Функциональная** | Вычисления через чистые функции, иммутабельность | Haskell, частично JS |
| **Декларативная** | Описываем *что* нужно, а не *как* | SQL, HTML |

ООП не исключает другие подходы: в PHP можно сочетать ООП, процедурный стиль и функциональные приёмы (замыкания, `array_map`).

### Пример: объект vs процедурный стиль

```php
// Процедурный подход
$balance = 1000;
function withdraw(array &$account, float $amount): void {
    $account['balance'] -= $amount;
}

// ООП-подход
class BankAccount
{
    public function __construct(private float $balance = 0) {}

    public function withdraw(float $amount): void
    {
        if ($amount > $this->balance) {
            throw new RuntimeException('Недостаточно средств');
        }
        $this->balance -= $amount;
    }

    public function getBalance(): float
    {
        return $this->balance;
    }
}

$account = new BankAccount(1000);
$account->withdraw(200);
```

### Источники

- [PHP: Введение в классы и объекты](https://www.php.net/manual/ru/language.oop5.php)
- [MDN: Object-oriented programming](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects/Object-oriented_programming) — концепции универсальны
- [Refactoring Guru: Основы ООП](https://refactoring.guru/ru/design-patterns/what-is-pattern) — наглядные объяснения принципов

---

## 2. Классы и методы

### Класс и объект

- **Класс** — шаблон (описание структуры и поведения).
- **Объект (экземпляр)** — конкретная сущность, созданная из класса через `new`.

```php
class Car
{
    public string $brand;

    public function __construct(string $brand)
    {
        $this->brand = $brand;
    }

    public function drive(): string
    {
        return "{$this->brand} едет";
    }
}

$car = new Car('Toyota');
echo $car->drive(); // Toyota едет
```

### Обычные (экземплярные) методы

Вызываются у **объекта** и работают с его состоянием (`$this`).

```php
class Counter
{
    private int $value = 0;

    public function increment(): void
    {
        $this->value++;
    }

    public function getValue(): int
    {
        return $this->value;
    }
}
```

### Статические методы и свойства

Принадлежат **классу**, а не объекту. Вызываются через `ClassName::method()` без `new`.

```php
class MathHelper
{
    public static function sum(int $a, int $b): int
    {
        return $a + $b;
    }

    private static int $calls = 0;

    public static function countCall(): int
    {
        return ++self::$calls;
    }
}

echo MathHelper::sum(2, 3);      // 5
echo MathHelper::countCall();    // 1
```

> **Важно:** внутри статического метода `$this` недоступен — нет текущего объекта.

### Магические методы

Специальные методы с именами вида `__имя`, которые PHP вызывает автоматически в определённых ситуациях.

| Метод | Когда вызывается |
|-------|------------------|
| `__construct()` | При создании объекта |
| `__destruct()` | Перед уничтожением объекта |
| `__get($name)` | Чтение недоступного свойства |
| `__set($name, $value)` | Запись недоступного свойства |
| `__call($name, $args)` | Вызов недоступного метода |
| `__toString()` | Преобразование объекта в строку |
| `__invoke()` | Вызов объекта как функции |

```php
class User
{
    public function __construct(
        private string $name,
        private string $email
    ) {}

    public function __get(string $name): mixed
    {
        return $this->$name ?? null;
    }

    public function __toString(): string
    {
        return $this->name;
    }
}

$user = new User('Анна', 'anna@mail.ru');
echo $user;           // Анна (через __toString)
echo $user->name;     // Анна (через __get)
```

### Источники

- [PHP: Классы и объекты](https://www.php.net/manual/ru/language.oop5.php)
- [PHP: Статические методы](https://www.php.net/manual/ru/language.oop5.static.php)
- [PHP: Магические методы](https://www.php.net/manual/ru/language.oop5.magic.php)

---

## 3. Интерфейсы и абстрактные классы

### Интерфейс

**Интерфейс** — контракт: описывает **какие методы** должен иметь класс, но **не содержит реализации** (в PHP до 8.0; с PHP 8+ допускаются default-методы в traits, но интерфейс по сути остаётся контрактом).

```php
interface Notifiable
{
    public function send(string $message): bool;
}

class EmailNotifier implements Notifiable
{
    public function send(string $message): bool
    {
        // отправка email
        return true;
    }
}

class SmsNotifier implements Notifiable
{
    public function send(string $message): bool
    {
        // отправка SMS
        return true;
    }
}

function notifyUser(Notifiable $notifier, string $text): void
{
    $notifier->send($text); // полиморфизм
}
```

Класс может **реализовать несколько интерфейсов**: `class A implements B, C`.

### Абстрактный класс

Содержит **общую реализацию** и **абстрактные методы** без тела. Нельзя создать объект напрямую: `new AbstractClass()` — ошибка.

```php
abstract class Animal
{
    public function __construct(protected string $name) {}

    // Общая реализация
    public function sleep(): string
    {
        return "{$this->name} спит";
    }

    // Обязательно переопределить в потомке
    abstract public function makeSound(): string;
}

class Dog extends Animal
{
    public function makeSound(): string
    {
        return 'Гав!';
    }
}

$dog = new Dog('Шарик');
echo $dog->makeSound(); // Гав!
```

### Интерфейс vs абстрактный класс

| Критерий | Интерфейс | Абстрактный класс |
|----------|-----------|-------------------|
| Множественное наследование | Да (несколько интерфейсов) | Нет (один родитель) |
| Состояние (свойства) | Только константы | Может иметь свойства |
| Реализация методов | Нет (кроме PHP 8+ нюансов) | Частичная реализация |
| Назначение | Контракт «умеет делать» | Общая база для родственных классов |

### Источники

- [PHP: Интерфейсы](https://www.php.net/manual/ru/language.oop5.interfaces.php)
- [PHP: Абстрактные классы](https://www.php.net/manual/ru/language.oop5.abstract.php)

---

## 4. Область видимости

Модификаторы доступа управляют тем, **откуда** можно читать и изменять свойства и методы.

| Модификатор | Класс | Наследник | Вне класса |
|-------------|-------|-----------|------------|
| `public` | ✓ | ✓ | ✓ |
| `protected` | ✓ | ✓ | ✗ |
| `private` | ✓ | ✗ | ✗ |

```php
class ParentClass
{
    public string $publicProp = 'всем видно';
    protected string $protectedProp = 'наследникам';
    private string $privateProp = 'только этому классу';

    public function show(): void
    {
        echo $this->privateProp; // OK внутри класса
    }
}

class ChildClass extends ParentClass
{
    public function test(): void
    {
        echo $this->protectedProp; // OK
        // echo $this->privateProp; // Ошибка!
    }
}

$obj = new ChildClass();
echo $obj->publicProp;
// echo $obj->protectedProp; // Ошибка снаружи
```

### Инкапсуляция на практике

Внутренние поля делают `private` / `protected`, наружу отдают методы — так проще менять реализацию без поломки внешнего кода.

### Источники

- [PHP: Видимость](https://www.php.net/manual/ru/language.oop5.visibility.php)

---

## 5. Наследование и реализация интерфейсов

### Наследование классов

Потомок **получает** свойства и методы родителя и может **переопределять** их.

```php
class Vehicle
{
    public function move(): string
    {
        return 'Движение';
    }
}

class Bicycle extends Vehicle
{
    public function move(): string
    {
        return 'Еду на велосипеде';
    }
}

$bike = new Bicycle();
echo $bike->move(); // Еду на велосипеде
```

Ключевое слово `parent::` вызывает метод родителя:

```php
class ElectricBicycle extends Bicycle
{
    public function move(): string
    {
        return parent::move() . ' с мотором';
    }
}
```

### Реализация интерфейсов

```php
interface Payable
{
    public function pay(float $amount): bool;
}

interface Loggable
{
    public function log(string $message): void;
}

class PaymentService implements Payable, Loggable
{
    public function pay(float $amount): bool
    {
        $this->log("Оплата: {$amount}");
        return true;
    }

    public function log(string $message): void
    {
        echo $message;
    }
}
```

### Источники

- [PHP: Наследование](https://www.php.net/manual/ru/language.oop5.inheritance.php)
- [PHP: implements](https://www.php.net/manual/ru/language.oop5.interfaces.php)

---

## 6. Пространства имён

**Пространство имён (namespace)** — способ организовать код и **избежать конфликтов имён** классов и функций. Аналог «папок» для имён.

```php
namespace App\Models;

class User
{
    public function __construct(public string $name) {}
}
```

```php
namespace App\Controllers;

use App\Models\User;

class UserController
{
    public function show(): void
    {
        $user = new User('Иван');
        echo $user->name;
    }
}
```

### Псевдонимы и групповой import

```php
use App\Models\User as ModelUser;
use App\Models\{Post, Comment};
```

### Глобальные классы

```php
$date = new \DateTime(); // класс из глобального пространства
```

### PSR-4 и автозагрузка

Структура папок обычно соответствует namespace: `App\Models\User` → `src/Models/User.php`. Стандарт: [PSR-4](https://www.php-fig.org/psr/psr-4/).

### Источники

- [PHP: Пространства имён](https://www.php.net/manual/ru/language.namespaces.php)
- [PSR-4: Autoloading Standard](https://www.php-fig.org/psr/psr-4/)

---

## 7. Геттеры и сеттеры

**Геттер** — метод для **чтения** приватного свойства.  
**Сеттер** — метод для **записи** с проверкой и валидацией.

```php
class Product
{
    public function __construct(
        private string $name,
        private float $price
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function getPrice(): float
    {
        return $this->price;
    }

    public function setPrice(float $price): void
    {
        if ($price < 0) {
            throw new InvalidArgumentException('Цена не может быть отрицательной');
        }
        $this->price = $price;
    }
}

$product = new Product('Книга', 500);
$product->setPrice(450);
echo $product->getPrice(); // 450
```

### Зачем не public-свойства

- Валидация при записи
- Контроль формата при чтении
- Возможность менять внутреннее представление (например, хранить цену в копейках)

В PHP 8+ часть геттеров заменяют **promoted properties** и **readonly**, но для бизнес-логики сеттеры остаются актуальны.

### Источники

- [PHP: Свойства](https://www.php.net/manual/ru/language.oop5.properties.php)
- [Refactoring Guru: Encapsulate Field](https://refactoring.guru/ru/extract-method) — идея инкапсуляции полей

---

## 8. Передача параметров по ссылке

В PHP переменные по умолчанию передаются **по значению** (копия). С **`&`** — **по ссылке** (изменения видны снаружи).

### В функциях

```php
function addTen(int &$value): void
{
    $value += 10;
}

$number = 5;
addTen($number);
echo $number; // 15
```

### В методах

```php
class ArrayHelper
{
    public static function append(array &$items, mixed $item): void
    {
        $items[] = $item;
    }
}

$data = [1, 2];
ArrayHelper::append($data, 3);
// $data = [1, 2, 3]
```

### Объекты — особый случай

Объекты передаются как **идентификатор (handle)**; менять свойства объекта можно без `&`, но **переприсвоить** сам параметр — только со ссылкой:

```php
function replaceObject(?object &$obj): void
{
    $obj = new stdClass();
}

$a = new stdClass();
replaceObject($a); // $a теперь другой объект
```

### Источники

- [PHP: Передача аргументов по ссылке](https://www.php.net/manual/ru/language.functions.arguments.php#language.functions.arguments.pass-by-reference)

---

## 9. Исключения (try / catch)

**Исключение (Exception)** — механизм обработки ошибок без `return false` на каждом шаге. При ошибке выбрасывается объект исключения, выполнение переходит в `catch`.

### Базовый синтаксис

```php
try {
  $result = riskyOperation();
} catch (InvalidArgumentException $e) {
  echo 'Неверный аргумент: ' . $e->getMessage();
} catch (RuntimeException $e) {
  echo 'Ошибка выполнения: ' . $e->getMessage();
} finally {
  echo 'Выполнится всегда';
}
```

### Своё исключение

```php
class InsufficientFundsException extends Exception {}

class Wallet
{
    public function __construct(private float $balance) {}

    public function withdraw(float $amount): void
    {
        if ($amount > $this->balance) {
            throw new InsufficientFundsException('Недостаточно средств');
        }
        $this->balance -= $amount;
    }
}

try {
    $wallet = new Wallet(100);
    $wallet->withdraw(500);
} catch (InsufficientFundsException $e) {
    echo $e->getMessage();
}
```

### Иерархия

Ловите **сначала конкретные**, потом **общие** исключения. В PHP 7+ есть `Throwable` — базовый тип для `Exception` и `Error`.

### Источники

- [PHP: Исключения](https://www.php.net/manual/ru/language.exceptions.php)

---

## 10. Ключевые слова

### `final`

Запрещает **переопределение** метода или **наследование** класса.

```php
final class Config
{
    final public function getVersion(): string
    {
        return '1.0';
    }
}

// class ExtendedConfig extends Config {} // Ошибка
```

### `static`

Свойство или метод принадлежит **классу**, не экземпляру (см. [раздел 2](#2-классы-и-методы)).

### `use`

1. **Импорт** классов из namespace  
2. **Подключение trait** в классе

```php
namespace App;

use App\Traits\Loggable;

class Service
{
    use Loggable;
}
```

### `trait`

Механизм **повторного использования кода** без наследования. Решает проблему «один класс — один родитель».

```php
trait Timestampable
{
    public function touch(): void
    {
        $this->updatedAt = new DateTime();
    }
}

class Article
{
    use Timestampable;

    public DateTime $updatedAt;
}
```

### `clone`

Поверхностное копирование объекта. Для глубокого копирования переопределяют `__clone()`.

```php
$a = new stdClass();
$a->nested = new stdClass();

$b = clone $a;           // копия $a
$b->nested === $a->nested; // true — та же вложенная ссылка!

class DeepCopy implements Cloneable
{
    public function __clone()
    {
        $this->nested = clone $this->nested;
    }
}
```

### `$this`

Ссылка на **текущий объект** внутри экземплярного метода.

### `self`, `parent`, `static`

| Ключевое слово | Значение |
|----------------|----------|
| `self::` | Текущий класс (раннее связывание) |
| `parent::` | Родительский класс |
| `static::` | Позднее статическое связывание — класс, через который вызвали |

```php
class Base
{
    public static function who(): string { return 'Base'; }
}

class Child extends Base
{
    public static function who(): string { return 'Child'; }

    public static function test(): void
    {
        echo self::who();    // Child
        echo parent::who();  // Base
        echo static::who(); // Child (при вызове Child::test())
    }
}
```

### Источники

- [PHP: final](https://www.php.net/manual/ru/language.oop5.final.php)
- [PHP: Traits](https://www.php.net/manual/ru/language.oop5.traits.php)
- [PHP: Object cloning](https://www.php.net/manual/ru/language.oop5.cloning.php)
- [PHP: Late Static Bindings](https://www.php.net/manual/ru/language.oop5.late-static-bindings.php)

---

## 11. MVC-модель

### Что такое MVC

**Model–View–Controller** — архитектурный паттерн разделения приложения на три слоя:

```
┌─────────────┐     запрос      ┌──────────────┐
│   Browser   │ ──────────────► │  Controller  │
└─────────────┘                 └──────┬───────┘
       ▲                               │
       │                               ▼
       │                        ┌──────────────┐
       │      данные            │    Model     │
       │ ◄────────────────────  │ (бизнес-     │
       │                        │  логика, БД) │
       │                        └──────────────┘
       │                               │
       │         view data             ▼
       └──────────────────────  ┌──────────────┐
                                │     View     │
                                │ (HTML, UI)   │
                                └──────────────┘
```

| Слой | Ответственность | Не должен |
|------|-----------------|-----------|
| **Model** | Данные, бизнес-правила, работа с БД | Знать про HTML и HTTP-детали |
| **View** | Отображение (шаблоны, форма) | Содержать сложную бизнес-логику |
| **Controller** | Принять запрос, вызвать Model, передать данные во View | Быть «богом» со всей логикой |

### Поток запроса (пример)

1. Пользователь открывает `/products/5`
2. **Router** направляет на `ProductController::show(5)`
3. **Controller** вызывает `ProductModel::find(5)`
4. **Model** возвращает данные товара
5. **Controller** передаёт `$product` во **View** `products/show.php`
6. **View** рендерит HTML

### Упрощённый пример на PHP

```php
// Model
class ProductModel
{
    public function find(int $id): ?array
    {
        // Условно: запрос к БД
        return ['id' => $id, 'name' => 'Ноутбук', 'price' => 50000];
    }
}

// Controller
class ProductController
{
    public function __construct(
        private ProductModel $model
    ) {}

    public function show(int $id): void
    {
        $product = $this->model->find($id);
        if ($product === null) {
            http_response_code(404);
            return;
        }
        require __DIR__ . '/../views/product_show.php';
    }
}

// View (product_show.php)
?>
<h1><?= htmlspecialchars($product['name']) ?></h1>
<p>Цена: <?= $product['price'] ?> ₽</p>
```

### MVC во фреймворках

| Фреймворк | Controller | Model | View |
|-----------|------------|-------|------|
| Laravel | `app/Http/Controllers` | Eloquent Models | Blade templates |
| Symfony | Controller classes | Doctrine entities | Twig |
| Yii2 | `controllers/` | `models/` | `views/` |

### Плюсы MVC

- Разделение ответственности
- Проще тестировать Model без UI
- Параллельная работа frontend/backend

### Минусы и нюансы

- «Толстые» контроллеры при плохой дисциплине
- На практике часто используют **MVVM**, **MVP**, слоистую архитектуру поверх MVC

### Источники

- [MDN: MVC architecture](https://developer.mozilla.org/en-US/docs/Glossary/MVC)
- [Microsoft: Model-View-Controller (MVC)](https://learn.microsoft.com/en-us/aspnet/core/mvc/overview)
- [Laravel: Architecture Concepts](https://laravel.com/docs/architecture)
- [Symfony: Controller](https://symfony.com/doc/current/controller.html)

---

## Шпаргалка перед экзаменом

| Тема | Одна фраза |
|------|------------|
| ООП | Объекты = данные + поведение; 4 столпа: инкапсуляция, наследование, полиморфизм, абстракция |
| Методы | Обычные — у объекта; `static` — у класса; `__magic__` — хуки PHP |
| Интерфейс / abstract | Контракт vs база с частичной реализацией |
| Видимость | `public` / `protected` / `private` |
| Namespace | Изоляция имён, `use`, PSR-4 |
| MVC | Model — данные, View — UI, Controller — связь |

---

*Конспект подготовлен для экзамена «Понимание ООП, MVC». Примеры проверяйте на PHP 8.1+.*
