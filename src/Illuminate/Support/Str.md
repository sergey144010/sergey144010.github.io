# Illuminate\Support\Str
Можно найти здесь: [Illuminate\Support\Str](https://github.com/illuminate/support/blob/master/Str.php)

Очень интересный класс помошник для работы со строками. Идёт в связке с классом
[Illuminate\Support\Stringable](./src/Illuminate/Support/Stringable/Stringable.md)
Предоставляет много статических методов для работы со строкой, причём даже может иметь некоторые предварительные настройки.

Пройдёмся по каждому методу, которые имеются на момент написания данного текста.

### of()
Получает новый объект Stringable из переданной строки, как с ним работать смотри здесь [Illuminate\Support\Stringable](./src/Illuminate/Support/Stringable/Stringable.md)
```php
$stringAbleObject = Str::of('hello world');

var_dump($stringAbleObject);
# возвращает:
object(Illuminate\Support\Stringable)#3 (1) {
  ["value":protected]=>
  string(11) "hello world"
}
```

### after()
Возвращает остаток строки после первого вхождения заданного значения.
```php
$result = Str::after('world hello 1 world hello', 'hello');

# возвращает:
string(14) " 1 world hello"
```

### afterLast()
Возвращает остаток строки после последнего вхождения заданного значения.
```php
$result = Str::afterLast('world hello 1 world 2 hello', 'world');

# возвращает:
string(8) " 2 hello"
```

### ascii()
Переводит UTF-8 в ASCII.
```php
$result = Str::ascii('�Düsseldorf�');

# возвращает:
string(10) "Dusseldorf"
```

### transliterate()
Транслитерация строки до ее ближайшего ASCII-представления.
```php
$result = Str::transliterate('déjà σσς iıii');

# возвращает:
string(13) "deja sss iiii"
```

### before()
Получает часть строки перед первым вхождением заданного значения.
```php
$result = Str::before('world 0 hello 1 world hello', 'hello');

# возвращает:
string(8) "world 0 "
```

### beforeLast()
Получает часть строки перед последним вхождением заданного значения.
```php
$result = Str::beforeLast('world 0 hello 1 world hello', 'hello');

# возвращает:
string(22) "world 0 hello 1 world "
```

### between()
Получить часть строки между двумя заданными значениями.
```php
$result = Str::between('world 0 hello 1 my world hello', 'hello', 'world');

# возвращает:
string(6) " 1 my "
```