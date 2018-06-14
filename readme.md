# JSON-RPC Server (Laravel, Lumen)
This repository is a hard fork of tochka-developers/jsonrpc.
## Описание
JsonRpc сервер - реализация сервера по спецификации JsonRpc 2.0.

Убрана поддержка старых версий Laravel, поддерживается только Laravel и Lumen версии >5.6.
Для старых версий Laravel/Lumen используйте 1.* версии пакета.

Поддерживает:
* вызов удаленных методов по нотификации имяКонтроллера_имяМетода
* вызов нескольких удаленных методов в одном запросе
* передача параметров в метод контроллера по имени, либо в порядке очереди
* аутентификация с помощью токена, переданного в заголовке (отключаемо)
* контроль доступа по IP-адресам (отключаемо)
* контроль доступа к методам для разных сервисов - ACL (отключаемо)
* автоматическая генерация SMD-схемы
* возможность настройки нескольких точек входа с разными настройками JsonRpc-сервера
## Установка
### Laravel
1. ``composer require tochka-developers/jsonrpc``
2. Опубликуйте конфигурацию:  
```
php artisan vendor:publish
```
### Lumen
1. ``composer require tochka-developers/jsonrpc``
2. Зарегистрируйте сервис-провайдер `Tochka\JsonRpc\JsonRpcServiceProvider` в `bootstrap/app.php`:
```php
$app->register(Tochka\JsonRpc\JsonRpcServiceProvider::class);
```
3. Скопируйте конфигурацию из пакета (`vendor/tochka-developers/jsonrpc/config/jsonrpc.php`) в проект (`config/jsonrpc.php`)
4. Подключите конфигурацию в `bootstrap/app.php`:
```php
$app->configure('jsonrpc');
```
5. Включите поддержку фасадов в `bootstrap/app.php`:
```php
$app->withFacades();
```

## Ручная настройка точек входа
При ручной найтройке вы сами контролируете процесс роутинга. 
Пропишите в вашем route.php:
### Laravel
```php
Route::post('/api/v1/jsonrpc', function (Illuminate\Http\Request $request, \Tochka\JsonRpc\JsonRpcServer $server) {
    return $server->handle($request);
});
```
### Lumen
```php
$router->post('/api/v1/jsonrpc', function (Illuminate\Http\Request $request, \Tochka\JsonRpc\JsonRpcServer $server) {
    return $server->handle($request);
});
```

Если планируется передавать имя контроллера в адресе, после точки входа, роутинги дожны быть следующего вида:
### Laravel
```php
Route::post('/api/v1/jsonrpc/{endpoint}[/{action}]', function (Illuminate\Http\Request $request, \Tochka\JsonRpc\JsonRpcServer $server, $endpoint, $action = null) {
    return $server->handle($request, ['endpoint' => $endpoint, 'action' => $action]);
});
```
### Lumen
```php
$router->post('/api/v1/jsonrpc/{endpoint}[/{action}]', function (Illuminate\Http\Request $request, \Tochka\JsonRpc\JsonRpcServer $server, $endpoint, $action = null) {
    return $server->handle($request, ['endpoint' => $endpoint, 'action' => $action]);
});
```
Для установки уникальных параметров сервера необходимо передать массив с параметрами в метод `handle`:
```php
return $server->handle($request, $options);
```
### Описание массива $options
* `uri` ('/api/v1/jsonrpc') - точка входа
* `namespace` ('App\\Http\\Controllers\\') - Namespace для контроллеров. 
Если не указан - берется значение `jsonrpc.controllerNamespace`
* `controller` ('Api') - контроллер по умолчанию (для методов без указания контроллера - имяМетода). 
Если не указано - берется значение `jsonrpc.defaultController`
* `postfix` ('Controller') - суффикс контроллеров (для метода foo_bar будет выбран контроллер fooController). 
Если не указано - берется значение `jsonrpc.controllerPostfix`
* `middleware` (array) - список обработчиков запроса. 
В списке обработчиков обязательно должен быть `\Tochka\JsonRpc\Middleware\MethodClosureMiddleware::class`, 
данный обработчик отвечате за выбор контроллера и метода. 
Если список не указан - берется значение `jsonrpc.middleware`
* `description` ('JsonRpc server') - описание сервиса. Возвращается в SMD-схеме.
Если не указано - берется значение `jsonrpc.description`
* `auth` (true) - стандартная авторизация по токены в заголовке.
Если не указано - берется значение `jsonrpc.authValidate`
* `acl` (array) - список контроля доступа к методам. Заполняется в виде `имяМетода => [serviceName1, serviceName2]`.
Игнорируется, если не включен обработчик `AccessControlListMiddleware`.
Если не указано - берется значение `jsonrpc.acl`
* `endpoint` (string) - Пространство имён контроллеров. Добавляется к значению параметра namespace.
* `action` (string) - Имя контроллера. Если не указано и при этом указан endpoint, то endpoint является именем контроллера в пространстве namespace.

## Автоматический роутинг
Данный метод более удобен. Для роутинга достаточно перечислить точки входа в параметре `jsonrpc.routes`.
```php
[
    '/api/v1/jsonrpc',                  // для этой точки входа будут использованы глобальные настройки
    'v2' => [                          // для этой точки входа задаются свои настройки. Если какой-то из параметров не указан - используется глобальный
        'uri' => '/api/v1/jsonrpc',                       // URI (обязательный)
        'namespace' => 'App\\Http\\Controllers\\V2\\',   // Namespace для контроллеров
        'controller' => 'Api',                           // контроллер по умолчанию
        'postfix' => 'Controller',                       // суффикс для имен контроллеров
        'middleware' => [],                              // список обработчиков запросов
        'auth' => true,                                  // аутентификация сервиса
        'acl' => [],                                     // Список контроля доступа
        'description' => 'JsonRpc server V2'             // описание для SMD схемы
    ]
]
```

Для использования передачи имени контроллера в адресе
```php
[
    'v3' => [                          // для этой точки входа задаются свои настройки. Если какой-то из параметров не указан - используется глобальный
        'uri' => '/api/v3/jsonrpc/{endpoint}[/{action}]',// URI (обязательный)
        'namespace' => 'App\\Http\\Controllers\\V3\\',   // Namespace для контроллеров
        'controller' => 'Api',                           // контроллер по умолчанию
        'postfix' => 'Controller',                       // суффикс для имен контроллеров
        'middleware' => [],                              // список обработчиков запросов
        'auth' => false,                                  // аутентификация сервиса
        'acl' => [],                                     // Список контроля доступа
        'description' => 'JsonRpc server V3',             // описание для SMD схемы
    ]
]
```

Каждая точка входа может быть либо строкой с указанием адреса, либо массивом, аналогичном $options.

## Обработчики (Middleware)
Обработчики позволяют подготовить запрос, прежде чем вызывать указанный метод. Список обработчиков задается в параметре 
`jsonrpc.middleware`. Это массив, в котором необходимо перечислить в порядке очереди классы обработчиков.
По умолчанию доступны следующие обработчики:
`ValidateJsonRpcMiddleware`
Валидация запроса на соответствие спецификации JsonRpc2.0. Рекомендуется использовать.
`AccessControlListMiddleware`
Контроль доступа к методам.
`ServiceValidationMiddleware`
Контроль доступа к сервису на основе списка IP-адресов
`MethodClosureMiddleware`
Обработчик разбирает запрос и находит необходимый контроллер и метод в нем. Данный обработчик обязательно необходимо 
включать в список для работы сервера.
`AssociateParamsMiddleware`
Передача параметров из запроса в метод на основе имен.

Кроме того, вы можете использовать свои обработчики. 
Для этого просто реализуйте интерфейс `\Tochka\JsonRpc\Middleware\BaseMiddleware` и укажите обработчик в списке.

## Аутентификация
Если включена аутентификация (`jqonrpc.authValidate`), то в каждом запросе должен присутствовать заголовок (указанный в `jsonrpc.accessHeaderName`).
Значение заголовка - токен. Список токенов необходимо указать в параметре `jsonrpc.keys`:
```php
[
    'systemName1' => 'secretToken1',
    'systemName2' => 'secretToken2'
]
```
Если запрос был осуществлен без данного заголовка, либо с токеном, которого нет в списке - клиенту вернется ошибка.
Если аутентификация прошла успешно - клиент будет опознан как `systemName1` (`systemName2`), что позволит контролировать доступ к методам.
Если аутентификация отклбючена - клиент будет опознан как `guest`.

## Контроль доступа к сервису
Если включен обработчик `ServiceValidationMiddleware`, то будет осуществлен контроль доступа к сервису на основе списка IP-адресов.
Для описания доступа необходимо заполнить параметр `jsonrpc.servers`:
```php
[
    'systemName1' => ['192.168.0.1', '192.168.1.5'],
    'systemName2' => '*',
]
```
В указанном примере авторизоваться с ключом доступа сервиса `systemName1` могут только клиенты с IP адресами 
`192.168.0.1` и `192.168.1.5`. Сервис `systemName2` может авторизовываться с любых IP адресов.

## Контроль доступа к методам
Если включен обработчик `AccessControlListMiddleware`, то будет осуществлен контроль доступа к методам.
Для описания доступа необходимо заполнить параметр `jsonrpc.acl`:
```php
[
        //'App\\Http\\TestController1@method' => ['systemName1', 'systemName2'],
        //'App\\Http\\TestController2' => '*',
],
```
В указанном примере к методу `method` контроллера `App\Http\TestController1` будут иметь доступ только клиенты 
`systemName1` и `systemName2`, и ко всем методам контроллера `App\Http\TestController2` - все клиенты.

## Валидация параметров
Для валидации входных параметров внутри контроллера можно использовать готовый trait: `Tochka\JsonRpc\Traits\JsonRpcController`
Подключив данный trait в своем контроллере вы сможете проверить данные с помощью средств валидации Laravel:
```php
public function store($title, $body)
{
    $validatedData = $this->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ]);

    // The blog post is valid...
}
```
Кроме того, в указанном trait доступны следующие методы:
```php
/**
 * Возвращает массив с переданными в запросе параметрами
 */
protected function getArrayRequest(): array;

/**
 * Возвращает экземпляр класса с текущим запросом
 */
protected function getRequest(): JsonRpcRequest;

/**
 * Валидация переданных в контроллер параметров
 *
 * @param array $rules Правила валидации
 * @param array $messages Сообщения об ошибках
 * @param bool  $noException Если true - Exception генерироваться не будет
 *
 * @return bool|MessageBag Прошла валидация или нет
 */
protected function validate($rules, array $messages = [], $noException = false);

 /**
 * Валидирует и фильтрует переданные в контроллер параметры. Возвращает отфильтрованный массив с параметрами
 *
 * @param array $rules Правила валидации
 * @param array $messages Сообщения об ошибках
 * @param bool  $noException Если true - Exception генерироваться не будет
 */
protected function validateAndFilter($rules, array $messages = [], $noException = false): array;
```

## Логирование
Вы можете указать любой канал логов, который зарегистрирован у вас в системе (либо предварительно создать его) 
с помощью параметра `jsonrpc.log.channel`:
```php
/**
 * Канал лога, в который будут записываться все логи
 */
'channel' => 'default',
```

## Скрытие конфиденциальной информации в логах системы
Для того, чтобы убрать конфиденциальную информацию (логины, пароли, токены и пр.) из логов системы нужно в 
указать,что именно скрывать, в конфигурации `jsonrpc.log.hideParams`:
```php
[
    'App\\Http\\TestController1@method' => ['password', 'data.phone_number'],
    'App\\Http\\TestController2' => '*',
]
```
В указанном примере из логов метода `method` контроллера `App\Http\TestController1` будут скрываться занчения параметров
`password` и `data.phone_number` (поддерживается вложенность параметров в запросе при помощи разделителя `.`). 
В контроллере `App\Http\TestController2` во всех методах будут скрываться значения абсолютно всех параметров.

## Как это работает
Клиент послает валидный JsonRpc2.0-запрос:
```json
{
  "jsonrpc": "2.0", 
  "method": "client_getInfoById",
  "params": {
    "clientCode": "100500",
    "fromAgent" : true
  },
  "id": 15
 }
```
JsonRpc сервер пытается найти указанный метод `client_getInfoById`.
Имя метода разбивается на части: `имяКонтроллера_имяМетода`.
Класс контроллера ищется по указанному пространству имен (параметр `jsonrpc.controllerNamespace`) с указанным суффиксом (по умолчанию `Controller`).
Для нашего примера сервер попытается подключить класс 'App\Http\Controller\ClientController'.
Если контроллер не существует - клиенту вернется ошибка `Method not found`.
В найденном контроллере вызывается метод `getInfoById`.
Далее возможно два варианта. 

Если подключен обработчик `AssociateParamsMiddleware`, то все переданные параметры будут переданы в метод по именам.
То есть в контроллере должен быть метод `getInfoById($clientCode, $fromAgent)`. 
Все параметры будут отвалидированы по типам (если типы указаны). Кроме того, таким способом можно указывать необязательные 
параметры в методе - в таком случае их необязательно передавать в запросе, вместо непереданных параметров будут 
использованы значения по умолчанию из метода.
Если же не будет передан один из обязательных параметров - клиенту вернется ошибка.

Если обработчик `AssociateParamsMiddleware` не подключен - то все параметры из запроса будут переданы в метод по порядку.
В таком случае указанному запросу аналогичен следующий:
```json
{
  "jsonrpc": "2.0", 
  "method": "client_getInfoById",
  "params": ["100500", true],
  "id": 15
 }
```

Если настроено получение имени контроллера из точки входа то логика следующая:
Клиент посылает валидный JsonRpc2.0-запрос:
```json
{
  "jsonrpc": "2.0", 
  "method": "getInfoById",
  "params": {
    "clientCode": "100500",
    "fromAgent" : true
  },
  "id": 15
 }
```
На адрес `\client`.
JsonRpc сервер пытается найти указанный метод `getInfoById`.
В контроллере: `Сlient`. Если запрос идет на адрес `\client\action` то имя контроллера будет иметь вид `СlientAction`.
Класс контроллера ищется по указанному пространству имен (параметр `jsonrpc.controllerNamespace`) с указанным суффиксом (по умолчанию `Controller`).
Для нашего примера сервер попытается подключить класс 'App\Http\Controller\ClientController' или 'App\Http\Controller\ClientActionController' соответственно.
Если контроллер не существует - клиенту вернется ошибка `Method not found`.
В найденном контроллере вызывается метод `getInfoById`.


## Несколько вызовов в одном запросе
По спецификации JsonRpc разрешено вызывать несколько методов в одном запросе. Для этого необходимо валидные JsonRpc2.0-вызовы 
передать в виде массива. Каждый вызываемый метод будет вызван из соответствующего контроллера, а вернувшиеся результаты
будут возвращены клиенту в том порядке, в котором пришли запросы. 

В ответе клиенту всегда присутствует параметр id, если таковой был передан клиентом.
Данный параметр также позволяет идентифицировать ответы на свои запросы на стороне клиента.

## SMD-схема
Если на настроенную точку входу сделать запрос с параметром `?smd`, то сервер проигнорирует запрос и вернет полную
SMD-схему для указанной точки входа. SMD-схема строится автоматически на основании настроек указанной точки входа.
Список методов формируется исходя из доступных контроллеров в указанном для точки входа пространстве имен.
В SMD-схеме кроме стандартных описаний присутствуют дополнительные параметры, которые позволяют более точно сгенерировать 
JsonRpc-клиента и документацию по серверу.

## Дополнительное описание для SMD-схемы
SMD-схема генерируется на основе доступных публичных методов в доступных в указанном пространстве имен контроллеров.
По умолчанию этой информации достаточно для генерации на основе схемы прокси-клиента. Но для генерации понятной и полной 
документации также можно использовать расширенное описание контроллеров и методов в блоках `PhpDoc`.

### Описание группы методов
По умолчанию в качестве названия группы методов используется описание класса из PhpDoc:
```php
<?php

/**
 * Методы для работы с чем-то
 */
class SomeController {
    // ...
}
```
Также название можно указать с помощью тега `@apiGroupName`. В таком случае само описание класса будет проигнорировано:
```php
<?php

/**
 * Class SomeController
 * @package App\Http\Controllers\Api
 *
 * @apiGroupName
 * Методы для работы с чем-то
 */
class SomeController {
    // ...
}
```

### Доступные методы
По умолчанию генератор SMD-схемы собирает все доступные публичные методы из контроллера. Если необходимо скрыть из 
схемы какие-либо методы, можно воспользоваться тегом `@apiIgnoreMethod`:
```php
/**
 * Class DossierController
 * @package App\Http\Controllers\Api
 *
 * @apiGroupName
 * Методы для работы с чем-то
 * @apiIgnoreMethod someMethod
 * @apiIgnoreMethod otherMethod
 */
class SomeController {
```

### Описание метода
```php
/**
 * Основное описание метода.
 * @apiName Название метода. Если не указан этот тег - название метода формируется автоматически по правилам
 * @apiDescription
 * Описание метода. Если не указан - то берется основное описание метода.
 * @apiNote Замечание к методу
 * @apiWarning Предупреждение к методу
 * @apiParam * date="d.m.Y" $param="01.01.1970" ("23.12.2018") Описание параметра
 * @apiReturn int $someData Описание возвращаемого значения
 *
 * @param string $param1 Описание параметра 1
 * @param int $param2 Описание параметра 2
 * @param string $param3 Описание параметра 3
 * @return bool Описание возвращаемого ответа
 * 
 * @apiRequestExample
 * Пример запроса
 * @apiResponseExample
 * Пример ответа
 */
```
