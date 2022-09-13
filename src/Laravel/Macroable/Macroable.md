# Macroable

Начнём обзор с рассмотрения трейта который не имеет внешних зависимостей и сам по себе является
достаточно простым

## Illuminate\Support\Traits\Macroable

и разберёмся как его использовать

Название трейта говорит о том, что в класс в который включен данный трейт, а также в экземпляр этого класса
 можно будет подмешать другие методы из другого класса или подмешать анонимные функции как некий
именованный метод текущего класса.

Данный трейт предполагает хранение в себе некоторых так называемых
макросов в защищенном массиве и имеет следующие публичные основные рабочие методы:

- macro($name, $macro)
- mixin($mixin, $replace = true)
- hasMacro($name)
- flushMacros()

и два метода реализующие доступ как к статическим методам класса так и их экземплярах
- __callStatic($method, $parameters) - если вызываем неизвестный метод статически
- __call($method, $parameters) - если вызываем неизвестный метод через инициализированный объект

Для тестирование создадим два класса:

Включаем Macroable в OneService
```php
class OneService
{
    use Macroable;
}
```

и создаем сторонний второй сервис

```php
class TwoService
{
    public function twoMethod()
    {
        echo 'twoMethod() from ' . self::class . PHP_EOL;
    }
}
```

Начнем с самого простого, зарегистрируем некий макрос и проверим есть он или нет ну и сделаем сброс
всех макросов

```php
        /** Register */
        OneService::macro('two', new TwoService());
        if (OneService::hasMacro('two')) {
            echo '1' . PHP_EOL;
        }
```
Зарегистрировали в OneService под именем two экземпляр класса TwoService и проверили, что действительно что-то там зарегистрировалось.


```php
        /** Flush */
        OneService::flushMacros();
        if (OneService::hasMacro('two')) {
            echo '2' . PHP_EOL;
        }
```
Сбросили всё и действительно нашего two макроса нет.

Идём дальше и пробуем вызвать этот метод two в той реализации класса TwoService которая сейчас есть

```php
        /** Object invoke */
        OneService::macro('two', new TwoService());
        OneService::two();
```
 а вот тут мы получаем ошибку

```php
/** Response */
In Macroable.php line 98:
Object of type App\Functionality\Macroable\TwoService is not callable
```

в которой говорят, что наш TwoService не callable.
Попробуем это исправить и добавляем магический метод __invoke()
```php
class TwoService
{
    public function twoMethod()
    {
        echo 'twoMethod() from ' . self::class . PHP_EOL;
    }

    public function __invoke()
    {
        echo '__invoke() from ' . self::class . PHP_EOL;
    }
}
```

Метод __invoke() будет отрабатывать если наш TwoService будут вызывать как функцию, т.е. например

```php
$twoService = new TwoService();
$twoService();
```

Теперь еще раз проверяем
```php
        /** Object invoke */
        OneService::macro('two', new TwoService());
        OneService::two();
```
Вывод
```txt
/** Response */
__invoke() from App\Functionality\Macroable\TwoService
```

Т.е. получается, что если мы хотим зарегистрировать метод через сторонний объект,
то этот объект должен быть вызываемым, ну и соответственно остальные методы в этом
объекте работать не будут и не сильно нам получаются нужны, т.е.
```php
class TwoService
{
    public function __invoke()
    {
        echo '__invoke() from ' . self::class . PHP_EOL;
    }
}
```

Хорошо. Сигнатура метода macro говорит о том, что $macro может быть либо объектом либо чем-то вызываемым
```php
    /**
     * Register a custom macro.
     *
     * @param  string  $name
     * @param  object|callable  $macro
     * @return void
     */
    public static function macro($name, $macro)
    {
        static::$macros[$name] = $macro;
    }
```

Объект мы уже передавали, теперь попробуем передать анонимную функцию

```php
        /** Static method */
        OneService::macro('three', function() { echo 'Anonym static function' . PHP_EOL; });
        OneService::three();
```
Вывод говорит о том, что всё работает как надо
```txt
/** Response */
Anonym static function
```

Теперь попробуем вызвать некий метод four на объекте
```php
        OneService::macro('four', function() { echo 'Anonym function' . PHP_EOL; });
        $oneService = new OneService();
        $oneService->four();
```
И здесь всё хорошо
```txt
/** Response */
Anonym function
```

Теперь исследуем самый интересный метод mixin(). Сигнатура метода говорит о том,
что мы должны передать в метод некий объект $mixin.

```php
    /**
     * Mix another object into the class.
     *
     * @param  object  $mixin
     * @param  bool  $replace
     * @return void
     *
     * @throws \ReflectionException
     */
    public static function mixin($mixin, $replace = true)
```

Будем использовать второй класс сервиса, как и в самом начале со следующим кодом

```php
class TwoService
{
    public function twoMethod()
    {
        echo 'twoMethod() from ' . self::class . PHP_EOL;
    }
}
```

Теперь пробуем его передать в mixin()

```php
OneService::mixin(new TwoService());
OneService::twoMethod()
```

и что мы получаем

```php
/** Response */
twoMethod() from App\Functionality\Macroable\TwoService

In Macroable.php line 113:                                                                          
Method App\Functionality\Macroable\OneService::twoMethod does not exist.  
```

видим, что метод вызвался, а потом ошибка, что метода нет, странно ...
но если посмотреть на саму реализацию метода mixin(), то можно увидеть, что
ключу в массиве присваивается результат вызова метода

```php
static::macro($method->name, $method->invoke($mixin));
```

где macro()
```php
    public static function macro($name, $macro)
    {
        static::$macros[$name] = $macro;
    }
```

т.е. метод вызвался ничего не вернул, и в массиве static::$macros[$name] не создался ключ.
Проверим
```php
        OneService::macro('two', null);
        if (! OneService::hasMacro('two')) {
            echo 'two not found' . PHP_EOL;
        }
```
возврат
```php
/** Response */
two not found
```

так и есть.
Самое здесь интересное это то, что метод вызвался.

Т.е. если мы напишем всего одну строку
```php
OneService::mixin(new TwoService());
```
то все методы из TwoService будут вызваны.

Это наталкивает на мысль о том, что для
нормальной работы, все методы из TwoService должны возвращать анонимные функции, т.е.

```php
    public function threeMethod()
    {
        return function () {
            echo 'threeMethod() from ' . self::class . PHP_EOL;
        };
    }
```

тогда при вызове mixin() в static::$macros[$name] будет присвоена некая анонимная функция,
т.е. произойдёт некая инициализация без полезной работы,
а уже только дальше она будет обработана, т.е.
```php
class TwoService
{
    public function threeMethod()
    {
        return function () {
            echo 'threeMethod() from ' . self::class . PHP_EOL;
        };
    }
```

```php
OneService::mixin(new TwoService()); // инициализация
$oneService = new OneService();
$oneService->threeMethod(); // и только здесь полезная работа метода
```

## Вывод

Трейт Illuminate\Support\Traits\Macroable так или иначе призван работать только с анонимными функциями,
т.е. или таким образом
```php
        /** Static method */
        OneService::macro('three', function() { echo 'Anonym static function' . PHP_EOL; });
        OneService::three();
```
или если объект является вызываемым, т.е. содержит мотод __invoke(), таким образом
```php
        /** Static method */
        OneService::macro('three', new TwoService());
        OneService::three();
```
```php
class TwoService
{
    public function __invoke()
    {
        echo '__invoke() from ' . self::class . PHP_EOL;
    }
}
```

или если объект содержит набор методов, каждый из которых возвращает анонимную функцию

```php
OneService::mixin(new TwoService());
$oneService = new OneService();
$oneService->threeMethod();
$oneService->twoMethod();
```

```php
class TwoService
{

    public function twoMethod()
    {
        return function () {
            echo 'twoMethod() from ' . self::class . PHP_EOL;
        };
    }
    
    public function threeMethod()
    {
        return function () {
            echo 'threeMethod() from ' . self::class . PHP_EOL;
        };
    }
```

## Где можно применить
 - можно использовать TwoService как фабрику для создания например неких объектов
 - можно использовать Macroable для подмешивания функционала в класс или объект,
функционал который напрямую не относится к данному классу, но иметь в нём былобы не плохо,
как пример функционал расчета некой суммы

```php
class TwoService
{
    protected function sum()
    {
        return function(int $a, int $b) {
            echo $a + $b . PHP_EOL;
        };
    }
}
```

```php
OneService::mixin(new TwoService());
$oneService = new OneService();
$oneService->sum(1, 2);
```