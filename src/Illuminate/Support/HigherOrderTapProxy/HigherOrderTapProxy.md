# HigherOrderTapProxy

Еще один интересный и в тоже время очень простой класс

## Illuminate\Support\HigherOrderTapProxy

Название класса говорит о том, что это класс высшего порядка, который выступает как прокси класс цели которой будет
касаться (tap)

так как класс сам по себе не большой, то просто продублирую его здесь

```php
namespace Illuminate\Support;

class HigherOrderTapProxy
{
/**
* The target being tapped.
*
* @var mixed
*/
public $target;

    /**
     * Create a new tap proxy instance.
     *
     * @param  mixed  $target
     * @return void
     */
    public function __construct($target)
    {
        $this->target = $target;
    }

    /**
     * Dynamically pass method calls to the target.
     *
     * @param  string  $method
     * @param  array  $parameters
     * @return mixed
     */
    public function __call($method, $parameters)
    {
        $this->target->{$method}(...$parameters);

        return $this->target;
    }
}
```

Опять видим, что тип для $target в конструкторе не указан, а в док блоках опять тип значится как `mixed`
хотя совершенно очевидно, что $target должен быть объектом, причем в котором также должен быть и некий $method.
Потому, что если или $target не будет объектом или метода в нём не будет, то код упадёт с ошибкой.

Протестируем
```php
namespace App\Functionality\HigherOrderTapProxy;

class One
{
    public function sum(int $a, int $b): void
    {
        echo $a + $b . PHP_EOL;
    }
}

class Two
{
    public function diff(int $a, int $b): void
    {
        echo $a - $b . PHP_EOL;
    }
}
```

```php
        $proxy = new HigherOrderTapProxy(new One());
        $one = $proxy->sum(1, 2);
        var_dump($one);

        $proxy->target = new Two();
        $two = $proxy->diff(5, 4);
        var_dump($two);
```

```php
/** Response */
3
object(App\Functionality\HigherOrderTapProxy\One)#640 (0) {
}
1
object(App\Functionality\HigherOrderTapProxy\Two)#652 (0) {
}
```

По факту что делает - оборачивает объект класса в тип HigherOrderTapProxy,
т.е. приводит например множество различных объектов к единому типу,
и даёт возможность вызывать методы исходного объекта через прокси объект,
а также даёт возможность получить сам исходный объект.

Например получаем на входе множество объектов разного типа, а хотим в коде работать с одним объектом,
тогда оборачиваем все эти объекты в данный прокси объект и работаем уже с одним HigherOrderTapProxy объектом.