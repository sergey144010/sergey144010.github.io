# Небольшой рефакторинг и разбор вариантов логирования на рабочем примере

### Что мы будем делать и зачем ?
Мы будем избавляться от жесткой зависимости, в данном случае от логера, в некотором классе.
Мы это будем делать для того, чтобы иметь возможность тестировать данный класс,
иметь возможность подменять/убирать зависимости исходя из определенных условий,
что автоматически подразумевает работу с логикой класса, разделение её на некоторые слои, избавление от жестких зависимостей, соблюдение
принципа единой ответственности.

### Поехали

И так мы имеем следующий код

```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    private string $value;
    private string $date;

    public function process(int $customerAttributeId): void
    {
        $customerAttributeFlow = (new CustomerAttributeFlowRepository())->
        findByCustomerAttributeIdActual($customerAttributeId);

        $this->value = $customerAttributeFlow->value;

        if ($customerAttributeFlow->date === null) {
            Log::info("Attribute's date equality null");
            $customerAttributeFlow->date = '1970-01-01 00:00:01';
        }

        $this->date = $customerAttributeFlow->date;
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function getDate(): string
    {
        return $this->date;
    }
}
```

Где интерфейс выглядит следующим образом

```php
interface FreshValueInterface
{
    public function process(int $customerAttributeId): void;
    public function getValue(): string;
    public function getDate(): string;
}
```

Если посмотреть на класс CustomerAttributeFlowFreshValue то можно обратить внимание на то, что он не отвечает Single Responsibility principle
как минимум по тому, что в методе process() имеется логер, т.е. рабочий объект этого класса помимо своей основной рабочей логики еще и берёт дополнительную обязанность
логировать. Плюс к тому логер напрямую вшит в наш код, что максимально осложняет его какую либо подмену. Соответственно хотелось бы максимально приблизиться к Single Responsibility principle и избавиться от логера, но логирование оставить. Как этого можно добиться ?
Попробуем отрефакторить код и приблизиться к данной цели.

Для начала рассмотрим какие у нас есть варианты:
1. прокинуть логер в класс или сделать его инъекцию через конструктор или какой-либо метод
2. каким-то образом задекорировать данный класс
3. кинуть некое событие и вынести логер в него

Рассмотрим первый вариант.

Попытаемся прокинуть логер в класс через метод process()

```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    private string $value;
    private string $date;

    public function process(int $customerAttributeId, Psr\Log\LoggerInterface $logger): void
    {
        $customerAttributeFlow = (new CustomerAttributeFlowRepository())->
        findByCustomerAttributeIdActual($customerAttributeId);

        $this->value = $customerAttributeFlow->value;

        if ($customerAttributeFlow->date === null) {
            $logger->info("Attribute's date equality null");
            $customerAttributeFlow->date = '1970-01-01 00:00:01';
        }

        $this->date = $customerAttributeFlow->date;
    }
}
```

Но тогда надо править и интерфейс

```php
interface FreshValueInterface
{
    public function process(int $customerAttributeId, Psr\Log\LoggerInterface $logger): void;
    public function getValue(): string;
    public function getDate(): string;
}
```

и что получилось? Композицию заменили на агрегацию, вшитый логер убрали, теперь можем передавать всё что работает с Psr\Log\LoggerInterface, это хорошо.
Но к Single Responsibility principle не приблизились и еще и интерфейс пришлось править, это не очень хорошо.

А если прокидывать логер не через метод process(), а через другой какой-то метод например setLogger(Psr\Log\LoggerInterface $logger),
но его нет в интерфейсе, значит опять править интерфейс, это нам опять не подходит.

А если через конструктор передать ? Давайте посмотрим что получится

```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    private string $value;
    private string $date;
    private Psr\Log\LoggerInterface $logger;
    
    public function __construct(Psr\Log\LoggerInterface $logger = new NullLogger())
    {
        $this->logger = $logger;
    }

    public function process(int $customerAttributeId): void
    {
        $customerAttributeFlow = (new CustomerAttributeFlowRepository())->
        findByCustomerAttributeIdActual($customerAttributeId);

        $this->value = $customerAttributeFlow->value;

        if ($customerAttributeFlow->date === null) {
            $logger->info("Attribute's date equality null");
            $customerAttributeFlow->date = '1970-01-01 00:00:01';
        }

        $this->date = $customerAttributeFlow->date;
    }
}
```

Через конструктор как-то уже поинтереснее, править интерфейс не нужно, но нужно не забыть при инициализации класса CustomerAttributeFlowFreshValue
передать туда нужный логер. Это еще хорошо что сейчас можно в php передать некий new NullLogger() по дефолту в конструктор.

А скорее всего данный класс находится глубоко в других классах, что заставляет тянуть логер через все эти родительские классы
даже если этот логер им не нужен совсем.

```php
$object = new CustomerAttributeFlowFreshValue( new Logger() );
```

Ну и опять к Single Responsibility principle мы так и не приблизились, т.к. класс так и остался делать что-то еще помимо своих дел.

Получается что первый вариант нам не подходит.

Рассмотрим второй вариант.

Чтобы сделать декоратор для CustomerAttributeFlowFreshValue для логирования данной записи, нужно каким то образом разбить основной метод process()
на несколько методов. Попробуем.


```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    public const DATE_DEFAULT = '1970-01-01 00:00:01';

    private string $value;
    private string $date;

    public function process(int $customerAttributeId): void
    {
        $customerAttributeFlow = $this->getCustomerAttributeFlow($customerAttributeId);

        $this->value = $customerAttributeFlow->value;
        $this->date = $this->getCustomerAttributeFlowDate($customerAttributeFlow);
    }
    
    private function getCustomerAttributeFlow(int $customerAttributeId): CustomerAttributeFlow
    {
        return (new CustomerAttributeFlowRepository())->
            findByCustomerAttributeIdActual($customerAttributeId);
    }
    
    private function getCustomerAttributeFlowDate(CustomerAttributeFlow $customerAttributeFlow): string
    {
        if ($customerAttributeFlow->date === null) {
            return self::DATE_DEFAULT;
        }
        
        return $customerAttributeFlow->date;
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function getDate(): string
    {
        return $this->date;
    }
}
```

Здесь мы разбили метод process() на три логические части, теперь попробуем сделать из этих частей цепочку методов

```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    public const DATE_DEFAULT = '1970-01-01 00:00:01';

    private string $value;
    private string $date;
    
    private CustomerAttributeFlow $customerAttributeFlow;

    public function process(int $customerAttributeId): void
    {
        $this->getCustomerAttributeFlow($customerAttributeId);
        $this->getCustomerAttributeFlowValue();
        $this->getCustomerAttributeFlowDate();
    }
    
    protected function getCustomerAttributeFlow(int $customerAttributeId): void
    {
        $this->customerAttributeFlow =
            (new CustomerAttributeFlowRepository())->findByCustomerAttributeIdActual($customerAttributeId);
    }
    
    protected function getCustomerAttributeFlowValue(): void
    {
        $this->value = $this->customerAttributeFlow->value
    }
    
    protected function getCustomerAttributeFlowDate(): void
    {
        if ($this->customerAttributeFlow->date === null) {
            $this->date = self::DATE_DEFAULT;
            
            return;
        }
        
        $this->date = $this->customerAttributeFlow->date;
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function getDate(): string
    {
        return $this->date;
    }
}
```

И так метод process() теперь разбит на 3 подметода, попробуем сделать декоратор класса CustomerAttributeFlowFreshValue
и самый простой способ это сделать это отнаследоваться от CustomerAttributeFlowFreshValue и задекорировать метод getCustomerAttributeFlowDate()

```php
class CustomerAttributeFlowFreshValueDecorator extends CustomerAttributeFlowFreshValue
{
    protected function getCustomerAttributeFlowDate(): void
    {
        parent::getCustomerAttributeFlowDate();
        
        if ($this->date == self::DATE_DEFAULT) {
            $logger = $this->loggerInit();
            $logger->info("Attribute's date equality null");
        }
    }
    
    /**
    * Здесь у нас логика создания хитроумного логера
    */
    private function loggerInit(): Psr\Log\LoggerInterface
    {
        return (new LoggerBuilder())->buildLogger();
    }
}
```

Теперь у нас появилась возможность вызывать или декоратор класса или сам класс в зависимости от ситуации
и не прокидывая логер вообще никуда. Но тем не менее нужно в коде всё-равно вызвать либо-то либо-то или сделать некий if()
который будет это решение принимать в зависимости от чего-то. Либо просто в тестах использовать незадекорированный класс, а
в какой-то среде, например, при dev окружении использовать задекорированный логером класс.

```php
$object = new CustomerAttributeFlowFreshValue();
$object = new CustomerAttributeFlowFreshValueDecorator();
```

Но что мы получили за счет этого ? Класс CustomerAttributeFlowFreshValue без логера, можно сказать почти чистый
и отвечающий только за свою логику, приблизились к Single Responsibility principle, получили чёткую структуру в process().

Но как говорится предпочитай композицию наследованию поэтому давайте попробуем обойтись без наследования и при помощи композиции
построить декоратор. Но тут придётся пожертвовать открытостью методов.

```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    public const DATE_DEFAULT = '1970-01-01 00:00:01';

    private string $value;
    private string $date;
    
    private CustomerAttributeFlow $customerAttributeFlow;

    public function process(int $customerAttributeId): void
    {
        $customerAttributeFlow = $this->getCustomerAttributeFlow($customerAttributeId);
        $this->value = $this->getCustomerAttributeFlowValue($customerAttributeFlow);
        $this->date = $this->getCustomerAttributeFlowDate($customerAttributeFlow);
    }
    
    public function getCustomerAttributeFlow(int $customerAttributeId): CustomerAttributeFlow
    {
        return (new CustomerAttributeFlowRepository())->findByCustomerAttributeIdActual($customerAttributeId);
    }
    
    public function getCustomerAttributeFlowValue(CustomerAttributeFlow $customerAttributeFlow): string
    {
        return $customerAttributeFlow->value
    }
    
    public function getCustomerAttributeFlowDate(CustomerAttributeFlow $customerAttributeFlow): string
    {
        if ($customerAttributeFlow->date === null) {
            return self::DATE_DEFAULT;
        }
        
        return $customerAttributeFlow->date;
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function getDate(): string
    {
        return $this->date;
    }
}
```

И тогда декоратор получится следующий

```php
class CustomerAttributeFlowFreshValueDecorator implements FreshValueInterface
{
    private string $value;
    private string $date;
    
    private CustomerAttributeFlowFreshValue $originalService
    
    public function __construct()
    {
        $this->originalService = new CustomerAttributeFlowFreshValue();
    }
    
    public function process(int $customerAttributeId): void
    {
        $customerAttributeFlow = $this->originalService->getCustomerAttributeFlow($customerAttributeId);
        $this->value = $this->originalService->getCustomerAttributeFlowValue($customerAttributeFlow);
        $this->date = $this->getCustomerAttributeFlowDate($customerAttributeFlow);
    }

    protected function getCustomerAttributeFlowDate(CustomerAttributeFlow $customerAttributeFlow): string
    {   
        $date = $this->originalService->getCustomerAttributeFlowDate($customerAttributeFlow);
        
        if ($date == CustomerAttributeFlowFreshValue::DATE_DEFAULT) {
            $logger = $this->loggerInit();
            $logger->info("Attribute's date equality null");
        }
        
        return $data;
    }
    
    /**
    * Здесь у нас логика создания хитроумного логера
    */
    private function loggerInit(): Psr\Log\LoggerInterface
    {
        return (new LoggerBuilder())->buildLogger();
    }
    
    public function getValue(): string
    {
        return $this->value;
    }

    public function getDate(): string
    {
        return $this->date;
    }
}
```

При таком подходе и тестирование оригинального сервиса значительно упрощается.
Я здесь опускаю момент того, что можно еще поработать и с (new CustomerAttributeFlowRepository()) и вынести его например в конструктор и вообще упростить себе тестирование
оригинального сервиса, а говорю именно про логер и пути его извлечения.

Ну и осталось рассмотреть третий вариант - событие.

Здесь совсем просто - делаем метод событие и выносим логер в него:

```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    private string $value;
    private string $date;

    public function process(int $customerAttributeId): void
    {
        $customerAttributeFlow = (new CustomerAttributeFlowRepository())->
        findByCustomerAttributeIdActual($customerAttributeId);

        $this->value = $customerAttributeFlow->value;

        if ($customerAttributeFlow->date === null) {
            $this->event();
            $customerAttributeFlow->date = '1970-01-01 00:00:01';
        }

        $this->date = $customerAttributeFlow->date;
    }
    
    private function event(): void
    {
        /*
         * Здесь использую стандартное событие laravel
         * подписываюсь на него
         * и инициализирую логер в подписчике
         */
        try {
            event(new SomeEvent());
        }
        catch(Throwable) {
        }
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function getDate(): string
    {
        return $this->date;
    }
}
```

Конечно я не рассматриваю здесь логику написания событийной модели, а беру просто стандартную логику laravel events.
Мета подписчик будет выглядеть примерно так, который конечно должен быть подписан на SomeEvent в фреймворке, через ваш EventServiceProvider.

```php
class SomeEventListener
{
    protected function handle(): void
    {
        $logger = $this->loggerInit();
        $logger->info("Attribute's date equality null");
    }
    
    /**
    * Здесь у нас логика создания хитроумного логера
    */
    private function loggerInit(): Psr\Log\LoggerInterface
    {
        return (new LoggerBuilder())->buildLogger();
    }
}
```

Но это хорошо если у вас есть глобальная функция event() реализованная вашим фреймворком, а что делать если её нет?
Тут очевидно, что придётся прокинуть некий преднастроенный евент менеджер в ваш класс и слушать уже ваше событие.
В этом вам могут помочь следующие библиотеки:

- https://github.com/yiisoft/event-dispatcher
- https://github.com/doctrine/event-manager
- https://github.com/symfony/event-dispatcher

Словом все библиотеки которые поддерживают [PSR-14](https://www.php-fig.org/psr/psr-14/) стандарт. А найти их можно
просто на [packagist.org](https://packagist.org/providers/psr/event-dispatcher-implementation)

Внедрение диспетчеризации событий уже постепенно отсылает нас к использованию контейнера внедрения зависимостей.

И ёще один способ убрать прямую инициализацию логера в сервисе это использование контейнера внедрения зависимостей.
Начнём разбираться по порядку. И для более ясного понимания процесса вернёмся к начальному коду, но просто заменим
прямой вызов логера, на вызов логера из контейнера внедрения зависимостей:
```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    private string $value;
    private string $date;

    public function process(int $customerAttributeId): void
    {
        $customerAttributeFlow = (new CustomerAttributeFlowRepository())->
        findByCustomerAttributeIdActual($customerAttributeId);

        $this->value = $customerAttributeFlow->value;

        if ($customerAttributeFlow->date === null) {
            app(LoggerInterface)->info("Attribute's date equality null");
            $customerAttributeFlow->date = '1970-01-01 00:00:01';
        }

        $this->date = $customerAttributeFlow->date;
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function getDate(): string
    {
        return $this->date;
    }
}
```
Здесь опять стоит упомянуть о том, что мы используем встроенную в фреймворк laravel функцию app().
Которая из контейнера достаёт нужный нам проинициализированный объект. Если у вас нет контейнера, то можно
использовать любой который поддерживает [PSR-11](https://www.php-fig.org/psr/psr-11/) стандарт. А найти их также можно
 на [packagist.org](https://packagist.org/providers/psr/container-implementation)
Что нам дал такой подход? Такой подход дал нам возможность подменять объект логера через контейнер.
И теперь если мы захотим протестировать данный класс, то для тестов мы можем подменить логер на NullLogger() в контейнере или
непосредственно в тесте обратимся к контенеру и подменим логер, но по хорошему для тестового окружения нужно создать свой контейнер
и описать в нём зависимости которые мы хотим или подменять или использовать.
А теперь вернёмся к принципу единой ответственности и скомбинируем евенты и контейнер как в предыдущем примере
```php
class CustomerAttributeFlowFreshValue implements FreshValueInterface
{
    private string $value;
    private string $date;

    public function process(int $customerAttributeId): void
    {
        ...
        $this->event();
        ...
    }
}
```
А в слушателе уже будем доставать из контейнера логер
```php
class SomeEventListener
{
    protected function handle(): void
    {
        $logger = $this->loggerInit();
        $logger->info("Attribute's date equality null");
    }
    
    /**
    * Логика создания хитроумного логера перенесена в контейнер
    */
    private function loggerInit(): Psr\Log\LoggerInterface
    {
        return app(LoggerInterface);
    }
}
```

Мы рассмотрели несколько вариантов извлечения логера из класса.
Какой из них выбрать нужно решать исходя из ваших текущих условий.
Надеюсь кому-то было полезно это прочитать.

Enjoy !

PS: неплохо былобы добавить тесты и 
собрать мини приложение с контейнером и евент диспетчером и тоже всю эту всязку протестировать