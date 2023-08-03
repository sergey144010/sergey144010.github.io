# Illuminate\Support\Str
Можно найти здесь: [Illuminate\Support\Str](https://github.com/illuminate/support/blob/master/Str.php)

Очень интересный класс помошник для работы со строками. Идёт в связке с классом
[Illuminate\Support\Stringable](../../../src/Illuminate/Support/Stringable/Stringable.md)
Предоставляет много статических методов для работы со строкой, причём даже может иметь некоторые предварительные настройки.

Пройдёмся по каждому методу, которые имеются на момент написания данного текста.

### of()
Получает новый объект Stringable из переданной строки, как с ним работать смотри здесь [Illuminate\Support\Stringable](../../../src/Illuminate/Support/Stringable/Stringable.md)
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
$result = Str::between('world 0 hello 1 my world world hello', 'hello', 'world');

# возвращает:
string(12) " 1 my world "

```

### betweenFirst()
Получить наименьшую возможную часть строки между двумя заданными значениями.
```php
$result = Str::betweenFirst('world 0 hello 1 my world world hello', 'hello', 'world');

# возвращает:
string(6) " 1 my "
```

### camel()
Преобразует значение в camelCase. Из строки разделённой пробелами.
```php
$result = Str::camel('hello world my friends');

# возвращает:
string(19) "helloWorldMyFriends"
```

### charAt()
Получить символ по указанному индексу.
```php
$result = Str::charAt('hello world my friends', 4);

# возвращает:
string(1) "o"
```

### contains()
Определяет содержит ли заданная строка заданную подстроку.
```php
$result = Str::contains('hello world my friends', 'my');

# возвращает:
bool(false)

# применяют:
Str::contains($actualListener, '@')
```

### containsAll() 
Определяет содержит ли данная строка все значения массива.
```php
$result = Str::containsAll('hello world my friends', ['my', 'world']);

# возвращает:
bool(true)
```

### endsWith()
Определяет заканчивается ли данная строка одной из заданных подстрок.
```php
$result = Str::endsWith('hello world my friends', ['world', 'friends']);

# возвращает:
bool(true)
```

### excerpt()
Извлекает отрывок из текста, который соответствует первому экземпляру фразы.
```php
$text = 'Отрывок - это небольшой фрагмент произведения, который может содержать центральный элемент его сюжета, ' .
    ' настроения или стиля. В литературе отрывки используются для создания особой атмосферы и эффекта, когда читатель ' .
    'узнает о чем-то большем, чем то, что может предложить отдельный отрывок. Одним из главных навыков писателя является ' .
    'умение писать краткие, но ёмкие отрывки. Они должны быть наполнены эмоциями, описывать события и детали максимально ' .
    'точно и ярко передавать мир, созданный автором. Но как написать отрывок, который заинтересует читателя и не оставит его равнодушным?';

$result = Str::excerpt($text, 'для создания особой атмосферы', ['radius' => 21]);

# возвращает:
string(138) "...отрывки используются для создания особой атмосферы и эффекта, когда чит..."
```

### finish()
Закрыть строку одним экземпляром заданного значения.
```php
$result = Str::finish('hello world ', '!!!');

# возвращает:
string(15) "hello world !!!"
```

### wrap()
Оберните строку заданными строками.
```php
$result = Str::wrap('hello world ', 'before', 'after');

# возвращает:
string(23) "beforehello world after"
```