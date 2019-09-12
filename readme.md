# JSON-RPC Client (Laravel 5.6, Lumen 5.6)
## Описание
JsonRpc клиент - реализация клиента для JsonRpc-сервера.
Работает по спецификации JsonRpc 2.0. Протестирован и работает с оригинальным сервером JsonRpc от Tochka.
## Установка
### Laravel
1. ``composer require tochka-developers/jsonrpc-client``
2. Опубликуйте конфигурацию:  
```
php artisan vendor:publish --provider="Tochka\JsonRpcClient\JsonRpcClientServiceProvider"
```

### Lumen
1. ``composer require tochka-developers/jsonrpc-client``
2. Скопируйте конфигурацию из пакета (`vendor/tochka-developers/jsonrpc/config/jsonrpc-client.php`) в проект 
(`config/jsonrpc-client.php`)
3. Подключите конфигурацию в `bootstrap/app.php`:
```php
$app->configure('jsonrpc-client');
```
4. Включите поддержку фасадов в `bootstrap/app.php`:
```php
$app->withFacades();
```
5. Если планируете использовать автоматическую генерацию прокси-клиента - зарегистрируйте сервис-провайдер 
`Tochka\JsonRpcClient\JsonRpcClientServiceProvider` в `bootstrap/app.php`:
```php
$app->register(Tochka\JsonRpcClient\JsonRpcClientServiceProvider::class);
```
## Использование
### Настройка
Конфигурация находится в файле `app/jsonrpc-client.php`. 
В данном файле прописываются настройки для всех JsonRpc-подключений.
* `clientName` - Имя клиента. Данное имя будет подставляться в ID-всех запросов в виде префикса.
Позволяет идентифицировать сервис.
* `default` - подключение по умолчанию. Должно содержать имя подключения.
* `connections` - массив подключений. Каждое подключение должно иметь уникальный ключ (название подключения).

Настройки подключений:
* `url` - URL-адрес (или IP) для подключения к JsonRpc-серверу. Должен содержать полный путь к точке входа
(например: https://api.jsonrpc.com/v1/jsonrpc).
* `clientClass` - класс, который используется в качестве прокси-класса. Необходимо указывать полное наименование 
(с пространством имен). Используется при автоматической генерации прокси-класса.
* `extendedStubs` - генерация расширенного описания АПИ в виде классов-хелперов для входных и выходных параметров методов
* `middleware` - список классов-middleware, которые подготавливают запрос перед отправкой. Возможно перечисление 
классов-middleware в виде элементов массива, либо, если необходимо передать в класс дополнительные параметры - в качестве
ключей массива указываются классы-middleware, в качестве значения - массив с параметрами.
В пакете доступно две middleware:
* `AuthTokenMiddleware` - класс авторизации по токену в заголовке. Параметры: `name` - имя заголовка, `value` - значение 
токена
* `AuthBasicMiddleware` - класс Basic-авторизации. Параметры: `scheme` - тип авторизации (`basic`, `digest`, `ntlm`), 
`username` и `password` - данные для авторизации
* `AdditionalHeadersMiddleware` - класс для добавления кастомных заголовков. Параметры: `headers` - ассоциативный массив 
с заголовками, где ключ - имя заголовка, а значение - значение заголовка.

### Вызовы без прокси-класса
Вызов метода JsonRpc:
```php
use Tochka\JsonRpcClient\Client;
//....
$result = Client::fooBar('Some text');
```
Если необходимо использовать конкретное подключение, используется метод `get`:
```php
$result = Client::get('api')->fooBar('Some text');
```
Если не указано конкретное подключение - используется подключение по умолчанию.

По умолчанию клиент передает все переданные в метод параметры в виде индексированного массива.
Если JsonRpc-сервер требует передачи именнованных параметров - воспользуйтесь методом `call`:
```php
$result = Client::get('api')->call('fooBar', ['text' => 'Some text']);
```
Клиент поддерживает вызов нескольких удаленных методов через один запрос:
```php
$api = Client::get('api')->batch();
$resultFoo = $api->foo('params');
$resultBar = $api->bar(123);
$resultSome = $api->call('someMethod', ['param1' => 1, 'param2' => true]);
$api->execute();
```
В указанном примере в переменных $resultFoo, $resultBar и $resultSome будет пустой класс `Tochka\JsonRpcClient\Response`, 
пока не будет вызван метод `execute`. После этого будет осуществлен один запрос на JsonRpc-сервер, переменные 
заполнятся вернувшимися результатами с сервера.

Клиент поддерживает кеширование результатов с помощью метода `cache`:
```php
$result = Client::get('api')->cache(10)->fooBar('Some text');
```
При таком вызове результаты будут закешированы на 10 минут, и последующих вызовах этого метода с такими же параметрами - 
запрос на сервер не будет посылаться, результат будет сразу получаться из кеша. Естественно, результаты кешируются 
только для успешных вызовов. 

Также кеширование поддерживается и для нескольких вызовов:
```php
$api = Client::get('api')->batch();
$resultFoo = $api->cache(10)->foo('params');
$resultBar = $api->bar(123);
$resultSome = $api->cache(60)->call('someMethod', ['param1' => 1, 'param2' => true]);
$api->execute();
```
Учтите, что кешироваться будет только тот метод, перед которым был вызван `cache`. 

### Генерация прокси-класса
Прокси-класс - это фасад JsonRpcClient, который содержит информацию обо всех доступных методах
JsonRpc-сервера, а также сам делает маппинг параметров, переданных в метод, в виде ассоциативного массива.
Если сервер умеет возвращать SMD-схему, то такой класс может быть сгенерирован автоматически.

Для генерации класса воспользуйтесь командой:
```
php artisan jsonrpc:generateClient connection
```
Для успешной генерации должно выполняться несколько условий:
1. JsonRpc-сервер должен поддерживать возврат SMD-схемы (при передаче GET-параметра ?smd)
2. Желательно, чтобы в качестве сервера использовался `tochka-developers/jsonrpc`. Данный пакет умеет возвращать
расширенную информацию для более точной генерации прокси-класса
3. Должен быть прописан URL-адрес JsonRpc-сервера
4. Должно быть указано полное имя прокси-класса. Путь к файлу класса будет сгенерирован автоматически исходя из 
пространства имен и настроек `composer`.
5. Папка, в которой будет находиться прокси-класс, должна иметь иметь права на запись.

Если все указанные условия выполняются - то будет создан прокси-класс на указанное соединение.
Для обновления прокси-класса (в случае обновления методов сервера) - повторно вызовите указанную команду.
Если необходимо сгенерировать классы для всех указанных соединений - вызовите указанную команду без указания соединения:
```
php artisan jsonrpc:generateClient
```
### Вызовы методов
Вызов метода JsonRpc:
```php
//....
$result = Api::fooBar('Some text');
```

Клиент поддерживает вызов нескольких удаленных методов через один запрос:
```php
$api = Api::batch();
$api->foo('params');
$api->bar(123);
$api->someMethod(1, true);
[$resultFoo, $resultBar, $resultSome] = $api->execute();
```


Клиент поддерживает кеширование результатов с помощью метода `cache`:
```php
$result = Api::cache(10)->fooBar('Some text');
```
При таком вызове результаты будут закешированы на 10 минут, и последующих вызовах этого метода с такими же параметрами - 
запрос на сервер не будет посылаться, результат будет сразу получаться из кеша. Естественно, результаты кешируются 
только для успешных вызовов. 

Также кеширование поддерживается и для нескольких вызовов:
```php
$api = Api::batch();
$resultFoo = $api->cache(10)->foo('params');
$resultBar = $api->bar(123);
$resultSome = $api->cache(60)->someMethod(1, true);
[$resultFoo, $resultBar, $resultSome] = $api->execute();
```
Учтите, что кешироваться будет только тот метод, перед которым был вызван `cache`. 

### Middleware
Классы-middleware позволяет внести изменения в исходящие запросы, например добавить дополнительные заголовки, включить 
авторизацию, либо внести изменения в само тело запроса. 

Вы можете использовать свои классы, указав их имена в конфигурации необходимого подключения.
В классе middleware должне быть реализован один метод - `handle`. Первые два параметра обязательные: 
```php
public function handle(\Tochka\JsonRpcClient\Request $request, \Closure $next): void
    {
        return $next($request);
    }
```
Чтобы продолжить выполнение цепочки middleware, в методе необходимо обязательно вызвать метод $next, передав туда 
актуальную версию $request.
Кроме того, вы можете в параметрах метода `handle` использовать:
* дополнительные параметры, передаваемые в конфигурации:
```php
// config
'middleware'  => [
    \Tochka\JsonRpcClient\Middleware\AuthTokenMiddleware::class => [
        'name'  => 'X-Access-Key',
        'value' => 'TokenValue',
    ],
]

// middleware
use Tochka\JsonRpcClient\Request;

class AuthTokenMiddleware
{
    public function handle(Request $request, \Closure $next, $value, $name = 'X-Access-Key') 
    {
        // ...

        return $next($request);
    }
}
```
Порядок указание параемтров не важен, указанные в конфигурации значения будут переданы в middleware по имени параметра.
* контекстные классы `Tochka\JsonRpcClient\Contracts\TransportClient` и `Tochka\JsonRpcClient\ClientConfig`. 
Если у параметра указать один из указанных типов, то в метод при вызове будут переданы текущие экземпляры классов,
отвечающих за формирование транспортного запроса (например, сконфигурированный экземпляр класса 
`Tochka\JsonRpcClient\Client\HttpClient`) либо класс с конфигурацией текущего соединения.
* любой другой класс/контракт/фасад, зарегистрированный в DI Laravel