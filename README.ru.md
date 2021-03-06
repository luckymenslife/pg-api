# PG-API

## Что это?

PG-API это универсальный настраиваемый конструктор REST API для PostgreSQL. Позволяет строить сложные API к БД Postgres и реализовывать бизнес-логику на хранимых процедурах (функциях).

Особенности:  
 - запросы типа GET / POST / PUT / PATCH / DELETE 
 - авторизация по ключу либо по cookie
 - версионирование на уровне отдельных методов 
 - возможность передавать заголовки HTTP в функции
 - вызов внешних REST API сервисов, фоновая обработка
 - метрики Prometheus
 - возможность запуска в Kubernetes (readiness/liveness probes, graceful shutdown)
 - поддержка работы с файлами через MinIO
 - поддержка CORS

## Быстрый старт (5 простых шагов)

#### 1. Установка

```bash
$ go get github.com/bhmj/pg-api
$ cd cmd/pg-api
$ go build .
```

#### 2. Настройка

создаём настроечный файл `dummy.json`:
```json
{
    "Service": {
        "Version": "1.0.0",
        "Name": "dummy"
    },
    "HTTP": {
        "Port": 8080,
        "Endpoint": "api"
    },
    "DBGroup": {
        "Read": {
            "ConnString": "host=localhost port=5432 dbname=postgres user=postgres password=postgres sslmode=disable",
            "Schema": "api"
        }
    }
}
```

#### 3. Пишем бизнес-логику на PL/pgSQL

```SQL
create or replace function api.hello_get(int, _data json)
returns json
language plpgsql
as $$
declare
    _str text;
begin
    _str := 'Hello there, '||coalesce(_data->>'name', 'stranger')||'!';
    return json_build_object('greeting', _str);
end
$$;
```

#### 4. Запускаем PG-API

```bash
$ ./pg-api dummy.json
```

#### 5. Наш API работает

```bash
$ curl http://localhost:8080/api/v1/hello?name=Mike

{"greeting" : "Hello there, Mike!"}
```
--------------------------------------------------------------------
## Настройка

Для успешного запуска PG-API нужно настроить следующие параметры:
- Название и версию сервиса
- базовый путь к API
- параметры подключения к БД
- Методы и их свойства:
  - соглашение о вызове
  - Content type (*)
  - передачу заголовков HTTP (*)
  - правила вызова внешних сервисов (*)
  - финализирующую функцию для фоновой обработки (*)
- параметры аутентификации (*)
- параметры системы работы с файлами (*)

(*) -- опционально

Для указания файла настройки можно:  
a) задать путь к файлу в переменной окружения `PG_API_CONFIG`  
b) указать путь к файлу как параметр в командной строке

## Основные методы

| Метод | Описание |
| --- | --- |
| `/metrics` | метрики Prometheus |
| `/ready` | метод Readiness для k8s.<br/>Возвращает HTTP 200, если сервис готов, 500 если нет |
| `/alive` | метод Liveness для k8s.<br/>Возвращает HTTP 200, если сервис жив, 500 если завершает работу |
| `/{endpoint}/files/*` | метод файлового хранилища (см. ниже) |
| `/{endpoint}/v1/*` | базовый путь к API. Версия может отличаться от 1 |

## Обработка запроса

Порядок обработки запроса в PG-API.

I. Если финализирующая функция **не** указана 

Здесь выполняется простой линейный сценарий: [предобработка] -> функция -> возврат -> [постобработка]. Используется для коротких быстрых запросов.

1. парсится запрос и находится соответствующий `Method` в файле настроек
2. выполняется [предобработка](#предобработка--постобработка), если она описана в секции `Enhance`
3. строится SQL запрос для вызова функции на основании параметров из URL/body и с учётом соглашения о вызове
4. выполняется запрос в БД
5. полученный из функции ответ возвращается вызвавшей стороне
6. постобработка выполняется в фоне (если задана)

II. Если финализирующая функция **указана**

Это сценарий для быстрого создания объекта: инициализация -> возврат -> [предобработка] -> финализация -> [постобработка]. Полезен, когда предобработка или функция полного создания объекта занимают значительное время, а результат вызова (как правило, это ID созданного объекта) нужен сразу. Пример использования такого сценария: нужно создать запись в БД для нового отзыва к товару, при этом требуется рассчитать рейтинг клиента на основе его предыдущих покупок и отзывов и прогнать текст отзыва и фото через нейронную сеть (внешний сервис, предобработка), которая классифицирует отзыв, определит язык, автоматически создаст перевод на английский и заблокирует к публикации неприемлемые фотографии. Вся эта обработка требует времени, а принять отзыв на форме мы должны как можно быстрее; при этом для успешного завершения приёма отзыва сайту требуется только ID созданного отзыва.

1. парсится запрос и находится соответствующий `Method` в файле настроек
2. строится **инициализирующий** SQL запрос для вызова функции на основании параметров из URL/body и с учётом соглашения о вызове
3. выполняется **инициализирующий** запрос в БД, **получается ID новой записи**
4. ID новой записи возвращается вызывающей стороне; все дальнейшие шаги выполняются в фоне
5. выполняется [предобработка](#предобработка--постобработка), если она описана в секции `Enhance` **с учётом ID записи**
6. строится **финализирующий** запрос для вызова функции на основании параметров из URL/body и с учётом соглашения о вызове
7. выполняется **финализирующий** запрос в БД
8. выполняется постобработка (если задана)

Применение финализирующей функции может быть полезно в том случае, когда создание объекта происходит быстро, а предобработка (обогащение данных) может потребовать длительного времени. При указании финализирующей функции происходит вызов основной функции без обогащения, после чего полученный ID возвращается вызывающей стороне. Созданный объект сразу же может быть использован вызывающей стороной. Обогащение выполняется в фоне; по готовности вызывается та же функция, что и при создании объекта, но на этот раз с дополнительным параметром -- ID объекта. Это позволяет обновить данные в объекте. После обновления данных происходит постобработка (опциональный вызов внешних сервисов).

## Параметры HTTP запроса

`{method} domain:port / {endpoint} / {version} / {path} ? {params}`

| Параметр | Формат / Источник | Описание |
|---|---|---|
|**{method}** | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` | возможные HTTP методы |
|**{endpoint}** | `$.HTTP.Endpoint` | Произвольное слово. Обычно "api" |
|**{version}** | `v[0-9]+` | Обязательный номер версии |
|**{path}** | `(/blabla/[0-9]*)+` | объекты и их идентификаторы |
|**{params}** | `param=value & ...` | URL параметры |

### Правила построения запроса

* **{endpoint}** это база всех путей к методам API.  

* **{path}** разбирается как массив пар **объектов** и (опционально) их **идентификаторов**, разделённых символом `/`. **объекты** объединяются в строку через `_`, формируя имя функции. **Идентификаторы** передаются в функцию как параметры. Пропущенные идентификаторы заменяются нулями.  

* Для CRUD: **{method}** превращается в суффикс функции:
  | метод | суффикс |
  |---|---|
  |`GET`|`_get`|
  |`POST`|`_ins`|
  |`PUT`|`_upd`|
  |`PATCH`|`_pat`|
  |`DELETE`|`_del`|
* Для POST: **{method}** игнорируется

* **{version}** добавляется после суффикса в виде `_vN` только **в том случае, когда версия больше 1**.

* **{params}** преобразуются в пары "ключ-значение" и передаются последним аргументом в виде объекта JSON.

* **{body}** (кроме методов GET и DELETE) должен представлять собой объект либо массив JSON. Если тело запроса представляет собой объект JSON, то все параметры, переданные через URL, добавляются к нему (с замещением). Если тело запроса представляет собой массив, то параметры ,переданные через URL, игнорируются. Финальный JSON затем передаётся в функцию последним параметром.

### Правила преобразований в примерах

|**`CRUD`**  |  |  |
|:--|--|---|
|`GET /api/v1/foo/7/bar/9`| --> |`foo_bar_get(7,9,'{}')` |
|`GET /api/v1/foo/bar/12` | --> | `foo_bar_get(0,12,'{}')` |
|`GET /api/v1/foo/bar` | --> | `foo_bar_get(0,0,'{}')` |
|`GET /api/v1/foo/bar/3?p=v` | --> | `foo_bar_get(0,3,'{"p":"v"}')` |
|`POST /api/v1/foo/12/bar/` + `{...}` как body | --> | `foo_bar_ins(12,'{...}')` |
|`PUT /api/v3/foo/12/bar/34` + `{...}` как body | --> | `foo_bar_upd_v3(12,34,'{...}')` |
|`DELETE /api/v3/foo/bar/12` | --> | `foo_bar_del_v3(0,12)` |  
|  **`POST`**  |  |  |
|`POST /api/v1/foo/bar` + `{...}` как body | --> |`foo_bar(0,'{...}')` |
|`POST /api/v1/foo/9/bar` + `{...}` как body | --> |`foo_bar(9,'{...}')` |
|`POST /api/v3/profile?entry=FOO` + `{...}` как body | --> | `profile_v3('{"entry":"FOO", ...}')` |
|`GET /api/v1/foo/bar` | --> | `foo_bar(0,0,'{}')` |
| NB: метод GET для соглашения вызова POST не рекомендуется | | |

--------------------------------------------------------------------

## Настроечный файл

Настроечный файл в формате JSON. Поддержка других форматов планируется.  

### Подстановка переменных окружения

Для подстановки значений из переменных окружения в настроечный файл используется `{{ТАКОЙ_СИНТАКСИС}}`.  
Пример:
```json
{ "Password": "{{SECRET}}" }
```
Здесь, если определена переменная окружения `SECRET` со значением `abc123`, то предыдущая строка при выполнении программы примет вид
```json
{ "Password": "abc123" }
```
### Минимальный (обязательный) набор полей

`$.Service.Name` -- для метрик  
`$.Service.Version` -- для версионности  
`$.HTTP.Port` -- порт, на котором слушает сервис  
`$.HTTP.Endpoint` -- база URL  
`$.DBGroup.Read.ConnString` -- подключение к БД.  
`$.DBGroup.Read.Schema` -- схема БД, содержащая функции API  

см. также `examples/minimal.json`

### Значения по умолчанию

Соглашение о вызове : `CRUD`  
Content-Type : `application/json`  
CORS : `выключен`  
Авторизация : `нет`  
Prometheus buckets : `1 мс .. 5 с, логарифмическая шкала`  
Open connections : `не ограничено`  
Idle connections : `нет`  
LogLevel : `0` (нет)  

### Секция HTTP 
```Go
HTTP struct {
    Endpoint    string   // база URL
    Port        int      // порт, на котором слушает сервис  
    UseSSL      bool     // использовать SSL
    SSLCert     string   // путь к файлу SSL сертификата
    SSLKey      string   // путь к файлу приватного ключа SSL
    AccessFiles []string // список файлов, содержащих пару "ключ + имя" для авторизации по ключу
    CORS        bool     // включить CORS
}
```
### Секция БД
```Go
DBGroup struct {
    Read  Database  // настройки БД для чтения
    Write Database  // настройки БД для записи (не указывается, если такие же, как для чтения)
}
```
```Go
Database struct {
    ConnString string  // готовая строка подключения
    // --OR--
    Host       string  // либо
    Port       int     // составные
    Name       string  // части 
    User       string  // строки
    Password   string  // подключения
    //
    Schema     string  // (обязательно) схема, содержащая функции API 
    MaxConn    int     // (не обязательно) ограничение на кол-во открытых соединений
}
```
### Секция методов (и их свойства)

```Go
MethodConfig struct {
    Name         []string     // имя метода
    VersionFrom  int          // версия метода
    FinalizeName []string     // (*) завершающая функция
    Convention   string       // соглашение о вызове: POST, CRUD (по умолчанию CRUD)
    ContentType  string       // тип возвращаемого содержимого (по умолчанию application/json)
    Enhance      []Enhance    // (*) секция обогащения данных через вызов внешнего сервиса
    Postproc     []Enhance    // (*) секция постобработки через вызов внешнего сервиса
    HeadersPass  []HeaderPass // передача HTTP заголовков в функцию обработки
}
```
(*) -- необязательные поля

#### Тип возвращаемого содержимого

По умолчанию тип возвращаемого содержимого `application/json`, но можно указать любой другой, например `application/xml`, `text/html`, `text/plain`. Также при необходимости можно указать локаль: `application/xml; charset="UTF-8"`

#### Передача HTTP заголовков

Есть возможность настроить передачу заголовков (для каждого метода в отдельности или для всех). Для каждого передаваемого заголовка нужно задать имя поля JSON. Поля заголовка перезаписывают поля из тела запроса.

```Go
HeaderPass struct {
    Header    string  // поле заголовка
    FieldName string  // поле в JSON
}
```
#### Типы соглашений о вызове

Имеется два возможных соглашения о вызове: `POST` и `CRUD`  

`CRUD` (по умолчанию):
- метод GET читает; методы POST, PUT, PATCH и DELETE записывают.
- используются суффиксы функций: `get`, `ins`, `upd`, `pat` и `del` соответственно.
- предназначено для классического REST API (операции над объектами) и для интерфейсов, имеющих сильный перекос в сторону чтений (проще масштабировать через k8s: много реплик на чтение, один мастер на запись).

`POST`:
- для заданного URL любой HTTP метод вызывает одну и ту же функцию.
- суффиксы не применяются.
- все вызовы используют подключение к БД на запись.
- предназначено для сложных API, активно использующих json (SPA, микросервисы).

Следует заметить, что граница между "классическим REST API" и SPA/микросервисным режимом сильно размыта. Я рекомендую по возможности придерживаться соглашения `CRUD`, т.к. это облегчит масштабирование в случае непредвиденного увеличения нагрузки.

### Секция внешних сервисов

Необязательная секция `Enhance` в описании метода содержит информацию о внешних сервисах и набор правил для обогащения данных (применимо только для соглашения о вызове `POST`).

Обращения ко внешним сервисам производятся последовательно, что позволяет передавать в запрос к следующему внешнему сервису данные, полученные из предыдущего.

Пример:  
```Go
"Enhance": [ // массив: может содержать несколько обращений ко внешним сервисам
    {
        "URL"            : "http://some.service/api/", // URL внешнего сервиса
        "Method"         : "POST",                     // метод отправки запроса: POST или GET
        "IncomingFields" : ["$.nm_id", "$.chrt_id"],   // поля из входящего запроса (jsonpath)
        "ForwardFields"  : ["nms", "chrts"],           // соответствующие поля для внешнего сервиса
        "TransferFields" : [                           // правила выборки данных, полученных от внешнего сервиса:
            { "From": "$.result.details[0].shk_id",  "To": "shk_id" },
            { "From": "$.result.details[0].brand",   "To": "brand_name" },
            { "From": "$.result.details[0].%2.size", "To": "size_name" }
            // From: путь jsonpath к полю в ответе сервиса
            // To: имя поля, которе будет добавлено в наш json
            // %2: можно использовать %x чтобы сослаться на *значение* из ForwardField прямо в пути jsonpath (по порядковому номеру, от 1)
        ]
    }
]
```

В случае вызова внешнего сервиса методом GET параметры передаются в URL в виде пар `param=value`.  

В случае вызова внешнего сервиса методом POST параметры передаются в теле запроса как объект JSON.  

Ответ от внешнего сервиса ожидается в формате JSON.  

Результатом обогащения будет объект JSON, дополненный данными, которые были получены от всех внешних сервисов. Ошибки при вызове внешних сервисов не прерывают обработку.

#### Предобработка / постобработка

#### Финализирующая функция (опционально)

#### Параметры аутентификации (опционально)

#### Загрузка и выгрузка файлов (опционально)

#### "Общие" параметры

Вы можете указать общие параметры в секции `General`. Поля настроек, которые не указаны явно в каком-либо методе, ищутся в секции `General`; в случае отстутствия их в секции `General`, используются значения по умолчанию. Если метод, к которому происходит обращение, не имеет соответствия в секции `Methods[:]` по имени, то параметры его вызова берутся из секции `General`.  

---------------------------------------------------------------------

## Ещё примеры

В каталоге `examples/` приведены примеры настроечных файлов из реальных продуктовых систем.

NB: Все реальные значения в указанных примерах заменены на вымышленные. Все пароли, имена пользователей, сервера и поля данных полностью обезличены.

---------------------------------------------------------------------

## Changelog

**0.3.0** (2020-05-07) -- Первый выпуск системы в opensource.

## Дорожная карта

- [x] версионирование методов
- [x] вызов внешних сервисов
- [x] финализирующая функция
- [x] универсальные метрики
- [x] поддержка CORS 
- [x] передача заголовков HTTP
- [x] авторизация по ключу или cookie
- [x] поддержка MinIO 
- [x] Enhance[:].InArray
- [x] Enhance[:].HeadersToSend
- [ ] тесты!
- [ ] ещё примеры с комментариями
- [ ] circuit breaker
- [ ] экспорт в CSV / XLSX 

## Как поучаствовать в проекте

1. Fork it!
2. Create your feature branch: `git checkout -b my-new-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin my-new-feature`
5. Submit a pull request :)

## Лицензия

[MIT](http://opensource.org/licenses/MIT)

## Автор

Michael Gurov aka BHMJ
