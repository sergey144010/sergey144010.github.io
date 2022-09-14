# helpers.php

Это очень интересный файл, который устанавливает глобальные функции доступные всему коду проекта

## Illuminate\Support\helpers.php

### tap

Данная функция вызывает колбек, пропускает туда некое значение и возвращает значение.
По факту просто вызов колбека с каким то параметром. И значение может в данном случае быть абсолютно любым типом.
Но если колбек отсутствует, то значение оборачивается в прокси объект HigherOrderTapProxy, и значение уже может быть только
объектом, а дальше из этого прокси можно вызвать методы того объекта который передали как значение.
Данный прокси объект мы рассматривали подробно в разделе [Illuminate\Support\HigherOrderTapProxy](./src/Laravel/HigherOrderTapProxy/HigherOrderTapProxy.md)

```php
if (! function_exists('tap')) {
    /**
     * Call the given Closure with the given value then return the value.
     *
     * @param  mixed  $value
     * @param  callable|null  $callback
     * @return mixed
     */
    function tap($value, $callback = null)
    {
        if (is_null($callback)) {
            return new HigherOrderTapProxy($value);
        }

        $callback($value);

        return $value;
    }
}
```

Тестируем
```php
    $response = tap(4, function($item) {
        echo $item . PHP_EOL;
    });
    var_dump($response);
```

```php
/** Response */
4
int(4)
```

Т.е. можно написать абсолютно тоже самое вот так
```php
    $func = function($item) {
        echo $item . PHP_EOL;
    };
    $in = 4;
    $func($in);
    var_dump($in);
```

а можно элегантненько, как мы тестировали выше. Достойно внимания.

Но чтоже если колбека нет? Тогда у нас возвращается проски объект HigherOrderTapProxy
и значение $value уже должно быть объектом, а не чем то другим, а то всё упадёт.

и выглядеть это будет вот так
```php
class One
{
    public function sum(int $a, int $b): void
    {
        echo $a + $b . PHP_EOL;
    }
}

$proxy = tap(new One());
var_dump($proxy);
$one = $proxy->sum(1,2);
var_dump($one);
```

```php
/** Response */
object(Illuminate\Support\HigherOrderTapProxy)#640 (1) {
  ["target"]=>
  object(App\Functionality\HigherOrderTapProxy\One)#58 (0) {
  }
}

3

object(App\Functionality\HigherOrderTapProxy\One)#58 (0) {
}
```

Вот такая вот интересная функция