# Conditionable

Рассмотри ещё один абсолютно независимый трейт

## Illuminate\Support\Traits\Conditionable

Название его говорит о том, что он наделяет класс возможностью задавать делать некую работу по условию.

У него всего два публичный метода, которые абсолютно идентичны, за исключением одного знака

 - when()
 - unless()

Код которого можно даже и продублировать здесь

```php
    /**
     * Apply the callback if the given "value" is truthy.
     *
     * @param  mixed  $value
     * @param  callable  $callback
     * @param  callable|null  $default
     * @return $this|mixed
     */
    public function when($value, $callback, $default = null)
    {
        if ($value) {
            return $callback($this, $value) ?: $this;
        } elseif ($default) {
            return $default($this, $value) ?: $this;
        }

        return $this;
    }
```

Ну выглядит достаточно просто, давайте теперь это протестируем. Создадим класс и подключим в него трейт.

```php
namespace App\Functionality\Conditionable

class One
{
    use Conditionable;
}
```

Как таковой типизации на уровне языка здесь нет и только в док блоках написано, что они бы хоткели увидеть.
Но что такое `mixed $value` ? Пойдём с самого начала и протестируем данный метод передава ему везде null.
```php
        $one = new One();
        $one->when(null, null, null);
        var_dump($one);
```
Как и ожидалось он вернул самого себя
```php
/** Response */
object(App\Functionality\Conditionable\One)#58 (0) {
}
```

Пойдём дальше. Ну очевидно что вторым параметром должна быть функция обратного вызоыв, а то просто вывалится ошибка - Value of type null is not callable
А вот что с первым параметром? Попробуем bool, array, object
```php
        $one = new One();
        $response = $one->when(true, function() { echo '1' . PHP_EOL; }, null);
        var_dump($response);
```
```php
/** Response */
1
object(App\Functionality\Conditionable\One)#58 (0) {
}
```
Коллбек отработал и отдал нам тот же объект.

Теперь массив
```php
        $one = new One();
        $response = $one->when([], function() { echo '1' . PHP_EOL; }, null);
        var_dump($response);
```
```php
/** Response */
object(App\Functionality\Conditionable\One)#58 (0) {
}
```
А здесь колбек не отработал, а просто вернул объект. А если массив не пуст ?
```php
        $one = new One();
        $response = $one->when(['a'], function() { echo '1' . PHP_EOL; }, null);
        var_dump($response);
```
```php
/** Response */
1
object(App\Functionality\Conditionable\One)#58 (0) {
}
```

Здесь уже отработал коллбек и вернулся к нам объект.
Ну а если пустой объект?

```php
        $one = new One();
        $response = $one->when(new \stdClass(), function() { echo '1' . PHP_EOL; }, null);
        var_dump($response);
```
```php
/** Response */
1
object(App\Functionality\Conditionable\One)#58 (0) {
}
```

Тоже отработало.

Т.е. получается что по факту, если $value присутствует, а в случае с массивом и не пустое, то коллбек
будет отрабатывать, иначе попробует отработать второй коллбек по дефолту, т.е.

```php
        $one = new One();
        $response = $one->when(
            [],
            function() { echo 'if true' . PHP_EOL; },
            function() { echo 'else' . PHP_EOL; }
        );
        var_dump($response);
```
```php
/** Response */
else
object(App\Functionality\Conditionable\One)#58 (0) {
}
```

Т.е. получается некая замена конструкции if else с возвратом объекта

А теперь попробуем вернуть что-то из колбека

```php
        $one = new One();
        $response = $one->when(
            true,
            function() { return 1; },
            function() { return 2; }
        );
        var_dump($response);
```
```php
/** Response */
int(1)
```

А вот здесь уже респонс у нас не объект, а то что вернулось из колбека

```php
        $one = new One();
        $response = $one->when(
            false,
            function() { return 1; },
            function() { return new \stdClass(); }
        );
        var_dump($response);
```
```php
/** Response */
object(stdClass)#653 (0) {
}
```

Вот такая вот интересная замена для if else получается.

Метод unless() рассматривать не будем, т.к. он абсолютно одинаковый, но работает наоборот - на false отрабатывает первый коллбек.

Но еще важно упомянуть, что колбек принимает текущий объект и переданное значение, которое используется для сравнения, т.е.

```php
        $one = new One();
        
        $condition = true;
        
        $response = $one->when(
            $condition,
            function($currentOne, $currentCondition) {},
            function($currentOne, $currentCondition) {}
        );
        var_dump($response);
```

что позволяет делать такие вот штуки

```php
       ->when($before, function ($q, $currentBefore) {
           return $q->where('id', '<', $currentBefore);
       })
```