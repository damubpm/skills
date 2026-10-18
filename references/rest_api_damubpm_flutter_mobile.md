# REST API DamuBPM для Flutter и Mobile

[← Главная](index.html)

Практическая документация по REST API DamuBPM для мобильных приложений, включая Flutter/Dart.

Документ охватывает:

- пользовательские Lua REST-сервисы;
- BPM REST API;
- UserTask;
- формы BPM-задач и экземпляров;
- `/update_v_1_1`;
- JSON request/response;
- HTTP status codes;
- async;
- `no_parallel`;
- API Key;
- Flutter/Dart client;
- примеры интеграции.

> Важно: здесь перечислены REST-контракты, подтверждённые предоставленными материалами. Это не означает, что в полном серверном коде DamuBPM нет дополнительных endpoint’ов.

---

## Содержание

1. [Общая схема REST API](#1-общая-схема-rest-api)
2. [Lua REST Services](#2-lua-rest-services)
3. [Объект request](#3-объект-request)
4. [Формирование ответа](#4-формирование-ответа)
5. [BPM REST API](#5-bpm-rest-api)
6. [Формы UserTask и Instance](#6-формы-usertask-и-instance)
7. [update_v_1_1](#7-update_v_1_1)
8. [Flutter API Client](#8-flutter-api-client)
9. [Flutter примеры](#9-flutter-примеры)
10. [HTTP status и ошибки](#10-http-status-и-ошибки)
11. [Async и no_parallel](#11-async-и-no_parallel)
12. [Авторизация и headers](#12-авторизация-и-headers)
13. [Mobile BPM Flow](#13-mobile-bpm-flow)
14. [Checklist](#14-checklist)
15. [Проверенные практики мобильных проектов](#15-проверенные-практики-мобильных-проектов)

> Полное руководство по модулю `flt` (проект, OAuth2 PKCE, MapLibre, разрешения, CI/CD iOS, известные ошибки): `damubpm_flutter_mobile_guide.md`.

---

# 1. Общая схема REST API

Основное семейство пользовательских REST API DamuBPM:

```text
/restapi/services/run/<code>
```

Например, если код сервиса:

```text
api/v1/customer/create
```

то endpoint:

```text
/restapi/services/run/api/v1/customer/create
```

Поддерживаются методы:

```text
GET
POST
PUT
OPTIONS
```

---

# 2. Lua REST Services

## Универсальный endpoint

```text
GET /restapi/services/run/<code>
POST /restapi/services/run/<code>
PUT /restapi/services/run/<code>
OPTIONS /restapi/services/run/<code>
```

`<code>` — код пользовательского Lua REST-сервиса.

Пример:

```text
/restapi/services/run/api/v1/customer/create
```

## GET

Запрос:

```http
GET /restapi/services/run/customer/get?id=123
```

Lua:

```lua
local id = request.get.id

output = {
    success = true,
    id = id
}
```

### Пример JSON ответа

```json
{
  "success": true,
  "id": "123"
}
```

---

## POST

Запрос:

```http
POST /restapi/services/run/customer/create
Content-Type: application/json
```

Body:

```json
{
  "title": "ТОО Test",
  "email": "info@test.kz"
}
```

Lua:

```lua
local input = request.input

output = {
    success = true,
    title = input.title,
    email = input.email
}
```

### Пример JSON ответа

```json
{
  "success": true,
  "title": "ТОО Test",
  "email": "info@test.kz"
}
```

---

## PUT

Запрос:

```http
PUT /restapi/services/run/customer/update?id=123
Content-Type: application/json
```

Body:

```json
{
  "title": "Новое название"
}
```

Lua:

```lua
local id = tonumber(request.get.id)
local input = request.input

output = {
    success = true,
    id = id,
    title = input.title
}
```

### Пример JSON ответа

```json
{
  "success": true,
  "id": 123,
  "title": "Новое название"
}
```

---

## Универсальный JSON response

Lua-сервис сам формирует структуру `output`.

Например:

```lua
http_response_code = 200

output = {
    success = true,
    data = {
        id = 123,
        title = "Test"
    }
}
```

Ответ:

```json
{
  "success": true,
  "data": {
    "id": 123,
    "title": "Test"
  }
}
```

> Точная JSON-структура `/restapi/services/run/<code>` зависит от Lua-кода конкретного сервиса.

---

# 3. Объект request

В Lua REST-сервисе доступен глобальный объект:

```lua
request
```

| Поле | Назначение |
|---|---|
| `request.input` | JSON body как Lua-таблица |
| `request.input_raw` | исходное тело запроса |
| `request.get` | query/form параметры |
| `request.header` | HTTP headers |
| `request.user_id` | ID текущего пользователя |
| `request.last_user_id` | предыдущий user ID |
| `request.lang` | язык |
| `request.anonymous_session_id` | ID анонимной сессии |
| `request.host` | HTTP Host |
| `request.method` | HTTP method |
| `request.RemoteAddr` | адрес клиента |
| `request.NTLM_DOMAIN` | NTLM domain |
| `request.NTLM_USER` | NTLM user |
| `request.NTLM_HOST` | NTLM host |

Текущий пользователь:

```lua
local userId = tonumber(request.user_id)
```

GET:

```lua
local id = request.get.id
```

POST JSON:

```lua
local input = request.input
```

Header:

```lua
local apiKey = request.header["X-Api-Key"]
```

---

# 4. Формирование ответа

Основные глобальные переменные:

```lua
output
http_response_code
header
```

Пример:

```lua
http_response_code = 200

header = {
    ["X-Service"] = "mobile-api",
    ["Cache-Control"] = "no-store"
}

output = {
    success = true,
    data = {
        id = 123
    }
}
```

JSON:

```json
{
  "success": true,
  "data": {
    "id": 123
  }
}
```

---

# 5. BPM REST API

## POST /restapi/bpms/start

Запускает BPM-процесс.

```http
POST /restapi/bpms/start
Content-Type: application/json
```

Request:

```json
{
  "processCode": "process_code",
  "input": {
    "id": 125,
    "title": "Заявка №125"
  }
}
```

### JSON response

```json
{
  "ok": true,
  "instance": "INSTANCE-UUID",
  "task": "TASK-UUID",
  "errorText": "",
  "output": {},
  "instanceIsFinished": false
}
```

### Поля

| Поле | Назначение |
|---|---|
| `ok` | результат выполнения |
| `instance` | UUID экземпляра BPM |
| `task` | UUID текущей ожидающей задачи |
| `errorText` | текст ошибки |
| `output` | выходные переменные процесса |
| `instanceIsFinished` | завершён ли процесс |

Если:

```json
{
  "instanceIsFinished": true
}
```

процесс завершён.

---

## POST /restapi/bpms/runUserTaskByTask/

Рекомендуемый endpoint для завершения UserTask.

```http
POST /restapi/bpms/runUserTaskByTask/
Content-Type: application/json
```

Request:

```json
{
  "task": "TASK-UUID",
  "input": {
    "decision": "approve",
    "comment": "Согласовано"
  }
}
```

### Пример JSON ответа

```json
{
  "ok": true,
  "errorVars": {},
  "errorText": "",
  "task": "NEXT-TASK-UUID",
  "output": {},
  "instance": "INSTANCE-UUID",
  "instanceIsFinished": false
}
```

Основной принцип:

```text
task + input
```

Для нового кода рекомендуется продолжать BPM именно по UUID `task`.

---

# 6. Формы UserTask и Instance

## GET /restapi/bpms/ShowUserTaskForm/:task

```http
GET /restapi/bpms/ShowUserTaskForm/TASK-UUID
```

Назначение:

- получить форму UserTask;
- переменные;
- информацию по шагам.

### Ориентировочный JSON

```json
{
  "task": "TASK-UUID",
  "form": "...",
  "variables": {},
  "steps": []
}
```

> Точные имена полей ответа в предоставленном справочнике не зафиксированы.

---

## GET /restapi/bpms/ShowUserTaskFormAngular/:task

```http
GET /restapi/bpms/ShowUserTaskFormAngular/TASK-UUID
```

### Ориентировочный JSON

```json
{
  "task": "TASK-UUID",
  "angularForm": "...",
  "data": {}
}
```

> Точная структура ответа в предоставленных знаниях не описана.

---

## GET /restapi/bpms/ShowInstanceForm/:instance

```http
GET /restapi/bpms/ShowInstanceForm/INSTANCE-UUID
```

### Ориентировочный JSON

```json
{
  "instance": "INSTANCE-UUID",
  "form": "...",
  "data": {}
}
```

---

## GET /restapi/bpms/ShowInstanceAngularClass/:instance

```http
GET /restapi/bpms/ShowInstanceAngularClass/INSTANCE-UUID
```

### Ориентировочный JSON

```json
{
  "instance": "INSTANCE-UUID",
  "angularClass": "..."
}
```

---

## GET /restapi/bpms/ShowInstanceAngularForm/:instance

```http
GET /restapi/bpms/ShowInstanceAngularForm/INSTANCE-UUID
```

### Ориентировочный JSON

```json
{
  "instance": "INSTANCE-UUID",
  "angularForm": "..."
}
```

---

# 7. update_v_1_1

Это связанный data API DamuBPM.

Endpoint:

```text
/update_v_1_1
```

## Insert

```http
POST /update_v_1_1
Content-Type: application/json
```

Request:

```json
{
  "items": [
    {
      "table_name": "entity_code",
      "action": "insert",
      "values": [
        {
          "sys$uuid": "UUID",
          "id": 0,
          "title": "Новая запись"
        }
      ]
    }
  ]
}
```

### Пример JSON ответа

```json
{
  "error": 0,
  "error_text": "",
  "items": [],
  "last_insert_id": 123
}
```

---

## Delete

Request:

```json
{
  "items": [
    {
      "table_name": "entity_code",
      "action": "delete",
      "values": [
        {
          "id": 10
        },
        {
          "id": 11
        }
      ]
    }
  ]
}
```

### Пример JSON ответа

```json
{
  "error": 0,
  "error_text": "",
  "items": []
}
```

---

# 8. Flutter API Client

Добавьте пакет:

```yaml
dependencies:
  http: ^1.0.0
```

Client:

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class DamuApiClient {
  DamuApiClient({
    required this.baseUrl,
    this.defaultHeaders = const {},
  });

  final String baseUrl;
  final Map<String, String> defaultHeaders;

  Uri uri(
    String path, [
    Map<String, dynamic>? query,
  ]) {
    final base = Uri.parse('$baseUrl$path');

    if (query == null || query.isEmpty) {
      return base;
    }

    return base.replace(
      queryParameters: query.map(
        (key, value) => MapEntry(
          key,
          value?.toString() ?? '',
        ),
      ),
    );
  }

  Future<Map<String, dynamic>> getJson(
    String path, {
    Map<String, dynamic>? query,
    Map<String, String>? headers,
  }) async {
    final response = await http.get(
      uri(path, query),
      headers: {
        ...defaultHeaders,
        ...?headers,
      },
    ).timeout(const Duration(seconds: 25));

    return _decode(response);
  }

  Future<Map<String, dynamic>> postJson(
    String path,
    Object body, {
    Map<String, dynamic>? query,
    Map<String, String>? headers,
  }) async {
    final response = await http.post(
      uri(path, query),
      headers: {
        'Content-Type': 'application/json',
        ...defaultHeaders,
        ...?headers,
      },
      body: jsonEncode(body),
    ).timeout(const Duration(seconds: 25));

    return _decode(response);
  }

  Future<Map<String, dynamic>> putJson(
    String path,
    Object body, {
    Map<String, dynamic>? query,
    Map<String, String>? headers,
  }) async {
    final response = await http.put(
      uri(path, query),
      headers: {
        'Content-Type': 'application/json',
        ...defaultHeaders,
        ...?headers,
      },
      body: jsonEncode(body),
    ).timeout(const Duration(seconds: 25));

    return _decode(response);
  }

  Map<String, dynamic> _decode(
    http.Response response,
  ) {
    // Не response.body: без charset в Content-Type кириллица ломается.
    final dynamic decoded =
        response.bodyBytes.isEmpty
            ? <String, dynamic>{}
            : jsonDecode(utf8.decode(response.bodyBytes));

    if (response.statusCode < 200 ||
        response.statusCode >= 300) {
      throw DamuApiException(
        statusCode: response.statusCode,
        body: decoded,
      );
    }

    if (decoded is Map<String, dynamic>) {
      return decoded;
    }

    return {
      'data': decoded,
    };
  }
}

class DamuApiException implements Exception {
  DamuApiException({
    required this.statusCode,
    required this.body,
  });

  final int statusCode;
  final Object body;

  @override
  String toString() {
    return 'DamuApiException($statusCode): $body';
  }
}
```

---

# 9. Flutter примеры

## Создание клиента

```dart
final api = DamuApiClient(
  baseUrl: 'https://example.kz',
  defaultHeaders: {
    // Заголовки авторизации проекта.
  },
);
```

---

## GET Lua REST

```dart
final result = await api.getJson(
  '/restapi/services/run/api/v1/customer/get',
  query: {
    'id': 123,
  },
);

print(result);
```

---

## POST Lua REST

```dart
final result = await api.postJson(
  '/restapi/services/run/api/v1/customer/create',
  {
    'title': 'ТОО Test',
    'email': 'info@test.kz',
  },
);

print(result);
```

---

## PUT Lua REST

```dart
final result = await api.putJson(
  '/restapi/services/run/api/v1/customer/update',
  {
    'title': 'Новое название',
  },
  query: {
    'id': 123,
  },
);
```

---

## Запуск BPM

```dart
final result = await api.postJson(
  '/restapi/bpms/start',
  {
    'processCode': 'process_code',
    'input': {
      'id': 125,
      'title': 'Заявка №125',
    },
  },
);

final instance = result['instance'];
final task = result['task'];
final finished = result['instanceIsFinished'] == true;
```

---

## Завершение UserTask

```dart
final result = await api.postJson(
  '/restapi/bpms/runUserTaskByTask/',
  {
    'task': task,
    'input': {
      'decision': 'approve',
      'comment': 'Согласовано из Flutter',
    },
  },
);

final nextTask = result['task'];
final finished =
    result['instanceIsFinished'] == true;
```

---

## Получение формы UserTask

```dart
final form = await api.getJson(
  '/restapi/bpms/ShowUserTaskForm/$task',
);
```

---

## Insert через update_v_1_1

```dart
final result = await api.postJson(
  '/update_v_1_1',
  {
    'items': [
      {
        'table_name': 'entity_code',
        'action': 'insert',
        'values': [
          {
            r'sys$uuid': 'UUID',
            'id': 0,
            'title': 'Новая запись',
          },
        ],
      },
    ],
  },
);
```

---

# 10. HTTP status и ошибки

В Lua REST можно задавать:

```lua
http_response_code = 200
```

Поддерживаемые примеры:

| HTTP | Назначение |
|---:|---|
| `200` | OK |
| `201` | Created |
| `400` | Bad Request |
| `401` | Unauthorized |
| `404` | Not Found |
| `405` | Method Not Allowed |
| `409` | Conflict |

Рекомендуемый helper:

```lua
local function fail(status, code, message)
    http_response_code = status

    output = {
        success = false,
        error = {
            code = code,
            message = message
        }
    }
end
```

Пример:

```lua
local input = request.input

if not input then
    fail(
        400,
        "INVALID_BODY",
        "JSON body required"
    )
    return
end
```

JSON:

```json
{
  "success": false,
  "error": {
    "code": "INVALID_BODY",
    "message": "JSON body required"
  }
}
```

---

# 11. Async и no_parallel

## Async

Для POST и PUT:

```text
?async=1
```

Пример:

```text
/restapi/services/run/import/run?async=1
```

Ответ:

```text
ASYNC OK
```

---

## no_parallel

Настройка:

```text
no_parallel = 1
```

Одновременно может выполняться только один экземпляр REST-сервиса.

Полезно для:

- импорта;
- синхронизации;
- массовых операций;
- перерасчётов;
- больших отчётов;
- интеграционных процедур.

Если сервис уже выполняется:

```text
Service already started
```

Служебный код:

```text
7
```

---

# 12. Авторизация и headers

## Текущий пользователь

```lua
local userId = tonumber(request.user_id)
```

---

## API Key

```lua
local apiKey =
    request.header["X-Api-Key"]

if apiKey ~= "replace-with-secret" then

    http_response_code = 401

    output = {
        success = false,
        error = "Invalid API key"
    }

    return
end
```

Flutter:

```dart
final result = await api.postJson(
  '/restapi/services/run/api/v1/private/check',
  {
    'action': 'check',
  },
  headers: {
    'X-Api-Key': 'demo-secret',
  },
);
```

> Не рекомендуется хранить привилегированный API Key прямо в исходниках мобильного приложения.
>
> Для приложений сотрудников API Key не используется: вход — OAuth2 Authorization Code + PKCE через REST `<mod>_mobile_oauth_authorize` / `_token`, далее каждый запрос с `Authorization: Bearer <access_token>`. Сервер видит пользователя в `request.user_id` и проверяет его роли. См. раздел 15 и `damubpm_flutter_mobile_guide.md`.

## Bearer-токен мобильного пользователя

```dart
final result = await api.postJson(
  '/restapi/services/run/map_api',
  {'action': 'config', 'data': {}},
  headers: {'Authorization': 'Bearer $accessToken'},
);
```

Access-токен живёт 15 минут. На 401 клиент один раз обновляет его через `grant_type=refresh_token` и повторяет запрос.

---

## Response headers

Lua:

```lua
header = {
    ["X-Service"] = "customer",
    ["Cache-Control"] = "no-store"
}
```

---

## CORS

В настройках REST-сервиса можно использовать:

```text
Access-Control-Allow-Origin: {{Origin}}
```

Для нативного Flutter mobile CORS обычно не является browser-ограничением, но становится важен для Flutter Web.

---

# 13. Mobile BPM Flow

Рекомендуемая последовательность:

```text
Flutter
   |
   v
POST /restapi/bpms/start
   |
   +--> instance
   |
   +--> task
   |
   +--> output
   |
   v
task существует?
   |
   +-- нет --> instanceIsFinished
   |
   +-- да
         |
         v
GET ShowUserTaskForm/:task
         |
         v
Flutter UI
         |
         v
POST runUserTaskByTask
         |
         +--> next task
         |
         +--> output
         |
         +--> instanceIsFinished
```

Главный идентификатор продолжения процесса:

```text
task
```

Не следует строить новый мобильный API вокруг `pointCode`, если доступен UUID `task`.

---

# 14. Checklist

Перед подключением Flutter к DamuBPM REST API:

- [ ] используется HTTPS;
- [ ] определён `baseUrl`;
- [ ] используется `/restapi/services/run/<code>`;
- [ ] JSON отправляется с `Content-Type: application/json`;
- [ ] проверяется HTTP status;
- [ ] ошибки сервера обрабатываются отдельно;
- [ ] обязательные `request.input` поля валидируются;
- [ ] GET параметры валидируются;
- [ ] `request.user_id` преобразуется через `tonumber`;
- [ ] секреты не хранятся открыто в Flutter;
- [ ] для BPM сохраняется `task`;
- [ ] проверяется `instanceIsFinished`;
- [ ] UserTask завершается через `runUserTaskByTask`;
- [ ] async используется только при необходимости;
- [ ] `no_parallel` включается для тяжёлых процедур;
- [ ] response JSON имеет единый формат;
- [ ] обработаны 400/401/404/409;
- [ ] Flutter client имеет timeout/retry стратегию проекта;
- [ ] проверены ошибочные сценарии;
- [ ] ответ декодируется через `utf8.decode(response.bodyBytes)`;
- [ ] списки в Lua-ответах собраны через `array({})`;
- [ ] числа и флаги разбираются терпимо (могут прийти строками);
- [ ] изменения данных идут через `<mod>_bp` → бизнес-процесс с белым списком;
- [ ] вход — OAuth2 PKCE, токены в `flutter_secure_storage`;
- [ ] проверены операции записи, а не только чтение.

---

# 15. Проверенные практики мобильных проектов

Практики из рабочих приложений `map-bapps`, `grc-mobile`, `dlv-mobile`, `contact-center-mobile`.

## Настройки REST-сервисов

| Сервис | is_public | only_auth | cut_status_info | Метод |
|---|---|---|---|---|
| `<mod>_mobile_oauth_authorize` | 1 | 0 | 1 | GET, HTML |
| `<mod>_mobile_oauth_token` | 1 | 0 | 1 | POST form |
| `<mod>_mobile_oauth_logout` | 0 | 1 | 1 | POST |
| `<mod>_api` / `<mod>_m_api` | 0 | 1 | 1 | POST JSON |
| `<mod>_bp` | 0 | 1 | 1 | POST JSON |

`cut_status_info = 1` убирает служебную обёртку платформы, и клиент получает ровно `output`.

## Единый конверт ответа

```json
{"success": true, "data": {}}
{"success": false, "error": {"code": "BP_ERROR", "message": "Текст для пользователя", "field": "phone"}}
```

Исключение — OAuth `token`: стандартный ответ `{"access_token", "refresh_token", "token_type", "expires_in", "scope"}` и ошибки `{"error", "error_description"}`.

Если сервис упал на уровне платформы, текст ошибки приходит в `resultText` — клиент должен показывать его, когда `error` пуст.

## Коды HTTP, которые реально используются

| HTTP | Когда |
|---:|---|
| 400 | неверные параметры, неизвестный процесс (`UNKNOWN_PROCESS`), `invalid_grant` |
| 401 | нет или истёк Bearer — клиент обновляет токен один раз |
| 403 | нет роли модуля — показать «попросите администратора выдать роль …» |
| 405 | неверный метод, заголовок `Allow` |
| 422 | бизнес-ошибка процесса (`BP_ERROR`) — текст показать пользователю |
| 429 | лимит, заголовок `Retry-After` |
| 500 | процесс не запустился, ошибка commit |
| 503 | временная недоступность хранилища OAuth |

## Чтение и запись

- Чтение — один диспетчер `POST {action, data}` в `<mod>_api`, логика в `pkg/<mod>_mobile`.
- Запись — `POST {process, data}` в `<mod>_bp` → `BPMSStartProcess(process, uid, {input = JsonToString(data)})`, белый список процессов, `last_error` → 422, `Commit()` только в REST и с проверкой кода. В скрипте БП `Commit()`/`Rollback()` не вызывать.

## Типовые ошибки

- Кириллица «кракозябрами» → `response.body` вместо `utf8.decode(response.bodyBytes)`.
- `type '_Map<String, dynamic>' is not a subtype of 'List'` → Lua-массив сериализован без `array()`, пустой список пришёл как `{}`.
- `type 'String' is not a subtype of 'int'` → числа из SQL пришли строками; использовать `intv()` / `numv()`.
- Координаты округлены до 0,01° → атрибут типа `double` (`numeric(20,2)`); хранить `varchar(32)`.
- После входа на iOS возврат на экран входа → не зарегистрирована схема или не выключен `FlutterDeepLinkingEnabled`.
- Повторное списание/отправка после таймаута → нет `client_request_uuid` для идемпотентности.

---

## Быстрая шпаргалка

```text
Lua REST:
GET/POST/PUT/OPTIONS
/restapi/services/run/<code>

BPM start:
POST /restapi/bpms/start

UserTask:
POST /restapi/bpms/runUserTaskByTask/

UserTask form:
GET /restapi/bpms/ShowUserTaskForm/:task

UserTask Angular form:
GET /restapi/bpms/ShowUserTaskFormAngular/:task

Instance form:
GET /restapi/bpms/ShowInstanceForm/:instance

Instance Angular class:
GET /restapi/bpms/ShowInstanceAngularClass/:instance

Instance Angular form:
GET /restapi/bpms/ShowInstanceAngularForm/:instance

CRUD:
POST /update_v_1_1
```

---

[← Главная](index.html)

`DamuBPM REST API · Flutter · Mobile`
