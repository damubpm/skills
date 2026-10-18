# DamuBPM: мобильные приложения Flutter (модуль flt)

Практическое руководство по созданию мобильных приложений Android/iOS из DamuBPM: структура проекта в модуле `flt`, серверная часть (OAuth2 PKCE, REST, бизнес-процессы), клиент Flutter, настройки платформ, выгрузка в GitLab и сборка iOS в App Store. Основано на рабочих проектах `map-bapps`, `grc-mobile`, `dlv-mobile`, `contact-center-mobile` и ошибках, найденных при их разработке.

Связанный документ: `rest_api_damubpm_flutter_mobile.md` (общие REST-контракты DamuBPM).

---

## 1. Модуль flt: где что хранится

| Таблица | Назначение |
|---|---|
| `flt_projects` | проект: `code`, `title`, `description`, `pubspec`, настройки платформ, настройки GitLab |
| `flt_pages` | Dart-файлы: `project_id`, `code`, `title`, `description`, `dart_code`, `nn` (порядок), `route` |
| `flt_files` | любые другие файлы репозитория: `path`, `content`, `is_base64` (1 — бинарный, например PNG), `nn` |
| `flt_assets` | ассеты из файлового хранилища: `asset_path`, `file_id` |
| `flt_permissions` | разрешения: `platform` (`android` / `ios`), `perm_key`, `perm_value`, `title`, `nn` |
| `flt_cicd` | настройки CI/CD iOS (см. раздел 9) |

Настройки платформ в `flt_projects`:

| Поле | Пример | Назначение |
|---|---|---|
| `app_name` | `Map BApps` | имя под иконкой (Android label, iOS CFBundleDisplayName) |
| `org` | `kz.bapps` | `flutter create --org` |
| `url_scheme` | `damubpmmap` | схема OAuth-callback: iOS `CFBundleURLSchemes`, Android `appAuthRedirectScheme` (если в pubspec есть `flutter_appauth`) |
| `deep_linking_off` | `1` | `FlutterDeepLinkingEnabled = false` — обязательно для OAuth через `flutter_appauth` |
| `android_min_sdk` | `23` | minSdk |
| `ios_min_version` | `15.0` | `platform :ios` в Podfile |
| `android_generate` | `1` | генерировать gradle/manifest/kotlin (иначе — только `flutter create`) |
| `android_application_id`, `android_target_sdk`, `android_compile_sdk`, `android_agp_version`, `android_kotlin_version`, `android_gradle_version`, `android_default_config`, `android_manifest_application` | | тонкая настройка Android; пусто — значения по умолчанию (AGP 8.11.1, Kotlin 2.2.20, Gradle 8.14) |
| `gitlab_host`, `gitlab_project`, `gitlab_project_token`, `gitlab_branch`, `gitlab_last_md5` | | выгрузка в GitLab |

Бизнес-процессы модуля: `flt_projects_import_zip`, `flt_projects_publish`, `flt_projects_export_zip`, `flt_projects_export_gitlab`, `flt_cicd_apply`, `flt_cicd_run`.

MCP-методы: `flt_projects_list/get/create_update`, `flt_pages_list/get/create_update`, `flt_assets_list/get/create_update`, `flt_projects_publish_gitlab`, `flt_cicd_run`.

### Правила для страниц (`flt_pages`)

- Путь файла берётся из `description` в формате `Файл: lib/core/api.dart`. Без него: `code = main` → `lib/main.dart`, `code` вида `lib/...dart` → как есть, иначе `lib/pages/<code>.dart`.
- Путь обязан соответствовать `^lib/.*\.dart$`. `lib/main.dart` обязателен.
- Пустой `dart_code` — ошибка выгрузки.
- Порядок — по `nn`, затем `id`.
- Путь любого файла (страница, `flt_files`, ассет): только `[A-Za-z0-9_.@/-]`, без `..`, `//`, ведущего `/`, до 240 символов. Кириллица и пробелы в путях недопустимы.
- Повтор пути (без учёта регистра) — ошибка. Нельзя положить в `flt_files` файл с тем же путём, что и сгенерированный.
- Лимиты: до 1000 Dart-файлов, до 20 МБ исходников.

### Известные проблемы инструментов

- MCP `flt_projects_create_update` (2026-10-18) падал с ошибкой платформы `attempt to call a nil value` в ScriptTask. Обход: вставить проект SQL в `flt_projects`, а страницы загружать обычным `flt_pages_create_update`.
- Перед правкой страницы читать её через `flt_pages_get` и передавать полный `dart_code` — частичных правок нет.
- Бинарные файлы (иконки) класть в `flt_files` с `is_base64 = 1` (переводы строк в base64 допустимы — удаляются при выгрузке) или в `flt_assets`. Для `.png` в `flt_assets` проверяется сигнатура PNG.

---

## 2. Что генерирует выгрузка (ZIP / GitLab)

`flt_projects_export_gitlab` собирает:

1. `pubspec.yaml` из `flt_projects.pubspec`.
2. Все `flt_pages` как `lib/...dart`.
3. Все `flt_files` и `flt_assets`.
4. Если заданы настройки платформ или разрешения:
   - `tool/platform_config.json` — имя, org, схема, minSdk, iOS-версия, `deep_linking_off`, `appauth` (есть ли `flutter_appauth` в pubspec), разрешения;
   - `tool/configure_platforms.py` — применяет конфиг к `android/` и `ios/` после `flutter create` (идемпотентно);
   - `tool/setup.sh` — `flutter create . --org <org> --project-name <name> --platforms android,ios` → `configure_platforms.py` → `flutter pub get` → на macOS `pod install --repo-update`.
5. При `android_generate = 1` — `android/settings.gradle.kts`, `build.gradle.kts`, `app/build.gradle.kts` (с `manifestPlaceholders["appAuthRedirectScheme"]`), `MainActivity.kt`, `AndroidManifest.xml` с разрешениями и блоком `<queries>` (https, CustomTabsService — нужен AppAuth и url_launcher), стили запуска. Файл генерируется, только если такого пути нет в `flt_files` — свой файл в `flt_files` имеет приоритет.
6. `README_EXPORT.md`.

Особенности:

- iOS-значения `"true"` / `"false"` в `flt_permissions.perm_value` превращаются в булевы значения plist. Остальное пишется строкой.
- Выгрузка в GitLab инкрементальная: в `gitlab_last_md5` хранятся md5 файлов. Файл отправляется, только если его нет в ветке или md5 изменился. Ручные правки в GitLab не перетираются, пока не изменится содержимое в DamuBPM. Принудительная полная выгрузка — очистить `gitlab_last_md5`.
- Удаляются из GitLab только файлы, которые раньше выгружались из DamuBPM и пропали из проекта.
- Коммиты идут пачками до 200 действий / 8 МБ.
- Токену GitLab нужны роль Maintainer и scope `api`.

---

## 3. Эталонная структура приложения

```text
lib/main.dart                     MaterialApp, тема, локализация ru/kk/en, переключение Splash / Login / Home по сессии
lib/core/api.dart                 HTTP-клиент + Session (ChangeNotifier): токены, refresh, config
lib/core/mobile_oauth.dart        export 'mobile_oauth_stub.dart' if (dart.library.io) 'mobile_oauth_native.dart';
lib/core/mobile_oauth_native.dart flutter_appauth: signIn / renew
lib/core/mobile_oauth_stub.dart   заглушка для Web (UnsupportedError)
lib/core/ui.dart                  тема, палитра, форматирование, toast/confirm, общие виджеты
lib/pages/login_page.dart
lib/pages/<экран>.dart
```

Базовый `pubspec`:

```yaml
name: map_bapps
publish_to: none
version: 1.0.0+1
environment:
  sdk: '>=3.7.0 <4.0.0'
  flutter: '>=3.29.0'
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  http: ^1.2.2
  flutter_secure_storage: ^9.2.2
  flutter_appauth: ^9.0.1
  url_launcher: ^6.3.1
flutter:
  uses-material-design: true
```

Дополнительно по задаче: `maplibre_gl: ^0.27.1` (требует Flutter 3.29+, iOS 13+), `geolocator: ^14.0.2`, сканер/камера — с разрешениями в `flt_permissions`.

Адрес сервера — только через `--dart-define`, не константой в коде по месту:

```dart
const serverUrl = String.fromEnvironment('MAP_BASE_URL', defaultValue: 'https://cc.cloud.damubpm.kz');
```

Значение для CI задаётся в `flt_cicd.dart_defines` (например `MAP_BASE_URL=https://cc.cloud.damubpm.kz`).

---

## 4. Серверная часть: вход OAuth2 Authorization Code + PKCE

Мобильному приложению нельзя давать пароль, общий API-ключ, MCP-токен или client secret. Используется собственный публичный OAuth-клиент модуля и персональный Bearer-токен пользователя.

### Три REST-сервиса на модуль

| Код | is_public | only_auth | Метод | Назначение |
|---|---|---|---|---|
| `<mod>_mobile_oauth_authorize` | 1 | 0 | GET | HTML-страница: при отсутствии сессии — редирект на `/login?returnUrl=…`, иначе проверка ролей, выдача кода и редирект в приложение |
| `<mod>_mobile_oauth_token` | 1 | 0 | POST form | обмен кода на токены и refresh с ротацией |
| `<mod>_mobile_oauth_logout` | 0 | 1 | POST | отзыв текущей сессии или всех (`{"all": true}`) |

Готовые образцы для копирования: `map_mobile_oauth_*`, `grc_mobile_oauth_*`, `dlv_mobile_oauth_*`, `c_omn_mobile_oauth_*`. Меняются только константы и список ролей:

```lua
local CLIENT, REDIRECT, SCOPE, SYSTEM = "damubpm-map-mobile", "damubpmmap://oauth/callback", "map_mobile", "map_mobile"
-- роли: r.code in ('admin','c_admin','map_manager')
-- операции лимита: "map_code" / "map_refresh"
```

Общие таблицы (модуль c): `c_omn_mobile_oauth_code`, `c_omn_mobile_oauth_session`, `c_omn_mobile_auth_rate`. Access-токен — строка `auth_tokens` с профилем `Bearer` и `system = <SYSTEM>`.

### authorize — ключевые проверки

- Только GET. `client_id`, `redirect_uri`, `scope`, `response_type=code`, `code_challenge_method=S256` сравниваются строго; `code_challenge` — `^[A-Za-z0-9_-]{43}$`; `state` 16–1024 символа.
- Адрес возврата на вход строится от `mcp_oauth_settings.issuer` (`where id = 1`), а не от `request.host`.
- Неавторизованный пользователь (`request.user_id < 1`) → HTML-страница с `location.replace(...)` на `/login?returnUrl=<authorize с теми же параметрами>`.
- Роль проверяется до выдачи кода; нет роли → понятная HTML-ошибка 403.
- Не более 12 кодов в минуту на пользователя.
- Код `dmc_` + 32 случайных байта hex; в БД хранится только sha256-хэш, `code_challenge`, срок 5 минут.
- Ответ — HTML с заголовками `Cache-Control: no-store`, `Referrer-Policy: no-referrer`; все значения в HTML через `HTMLEscapeString`, в JS — через `JsonToString`.

### token — ключевые проверки

- Только POST. Тело `application/x-www-form-urlencoded` разбирается вручную из `request.input_raw` (`Split` по `&`, `QueryUnescape`), лимит 8 КБ, повтор ключа — ошибка. Если `input_raw` пустой, а `request.input` — таблица, используется она.
- `client_secret`, `username`, `password` в запросе — отказ `invalid_client`.
- Лимит 60 запросов в минуту на IP+операцию через `insert … on conflict … where attempts < 60 returning attempts` (пустой результат → 429 `slow_down`, `Retry-After: 60`). После вставки — `Commit()`, иначе счётчик откатится вместе с ошибкой.
- Код погашается одним `update … set used_at = now() … returning user_id` с проверкой PKCE в SQL:

```sql
code_challenge = translate(rtrim(encode(digest(?::text,'sha256'),'base64'),'='),'+/','-_')
```

- Роль проверяется заново и при обмене кода, и при каждом refresh (снятие роли отзывает сессию).
- Refresh-токен `dmr_…` хранится хэшем; при ротации старый хэш переносится в `previous_refresh_hash`. Предъявление старого refresh → отзыв всей сессии и её access-токенов (защита от кражи).
- Access-токен живёт 15 минут, сессия — 30 дней.
- Ответ — стандартный OAuth, без обёртки `success/data`:

```json
{"access_token":"…","refresh_token":"dmr_…","token_type":"Bearer","expires_in":900,"scope":"map_mobile"}
```

- Ошибки — стандартные `{"error":"invalid_grant","error_description":"…"}` с HTTP 400/403/405/429/503.

### Ловушки Lua в этих сервисах

- `string.match` в песочнице равен `nil` — шаблоны проверять через `RegexpCheck(value, "^…$")` или SQL `~`.
- `SqlQueryRow2` при отсутствии строки возвращает ненулевой код («No data found»). Для «найдено / не найдено» использовать `SqlQueryRows` и проверять `#rows`.
- В `fail()` вызывать `Rollback()` до формирования ответа; при успехе проверять код `Commit()`.
- `header` задавать таблицей в начале скрипта: `Content-Type`, `Cache-Control: no-store`, `Pragma: no-cache`, `X-Content-Type-Options: nosniff`.
- Токены, коды и PKCE-verifier не писать в журналы и `warning_text`.

---

## 5. Серверная часть: API данных и изменения через бизнес-процессы

### Чтение: один REST на модуль

`<mod>_api` / `<mod>_m_api` / `<mod>_mobile_api`: `only_auth = 1`, `cut_status_info = 1`, `is_public = 0`, роли модуля. Контракт:

```text
POST {action, data}
→ 200 {"success": true, "data": {...}}
→ 4xx {"success": false, "error": {"code": "…", "message": "…", "field": "…"}}
```

Логику держать в `pkg/<mod>_mobile` (luas), REST — тонкий диспетчер по `action`. Первое действие — `config`: справочники, параметры, права, которые клиент кэширует в сессии. Успешный `config` = пользователь авторизован и имеет доступ.

### Изменения: REST `<mod>_bp` → бизнес-процесс

Все изменения данных (продажа, приёмка, этапы заказа) идут через бизнес-процессы — так сохраняются аудит, проверки и единая логика с веб-интерфейсом. Образец — `grc_bp`, `dlv_bp`:

```lua
header = {["Cache-Control"] = "no-store"}
local uid = tonumber(request.user_id)
local input = request.input or {}
local ALLOWED = {grc_sale_create = true, grc_sale_return = true}

local function fail(status, code, message)
  Rollback()
  http_response_code = status
  output = {success = false, error = {code = code, message = message}}
end

if request.method ~= "POST" then header.Allow = "POST"; fail(405, "METHOD_NOT_ALLOWED", "Требуется POST"); return end
if not uid or uid < 1 then fail(401, "UNAUTHORIZED", "Требуется вход"); return end
local process = tostring(input.process or "")
if not ALLOWED[process] then fail(400, "UNKNOWN_PROCESS", "Неизвестный бизнес-процесс: " .. process); return end
local data = type(input.data) == "table" and input.data or {}

local out, instance, task, errText, errNum = BPMSStartProcess(process, uid, {input = JsonToString(data)})
if errNum ~= 0 or out == nil then fail(500, "BP_START_FAILED", "Не удалось запустить процесс: " .. tostring(errText or "")); return end

local result = {}
if type(out.result) == "string" and out.result ~= "" then
  local ok, parsed = pcall(StringToJson, out.result)
  if ok and type(parsed) == "table" then result = parsed end
elseif type(out.result) == "table" then
  result = out.result
end
if (out.last_error or "") ~= "" then fail(422, "BP_ERROR", tostring(out.last_error)); return end
if result.ok == false then fail(422, "BP_ERROR", tostring(result.error or "Ошибка обработки")); return end

local _, ccode = Commit()
if ccode ~= 0 then fail(500, "COMMIT_FAILED", "Не удалось сохранить изменения"); return end
result.instance = instance
output = {success = true, data = result}
```

Правила:

- Белый список процессов обязателен — нельзя запускать произвольный процесс по имени из запроса.
- Процесс принимает одну переменную `input` (JSON-строка) и возвращает `result` (JSON) и `last_error`.
- В скрипте БП нельзя вызывать `Commit()` / `Rollback()` — транзакцией управляет движок («transaction not begin»). Commit делает REST после успешного процесса.
- Бизнес-ошибка → 422, клиент показывает `error.message` пользователю.
- Для идемпотентных операций (оплата, отправка) клиент передаёт `client_request_uuid` (UUIDv4) и при повторе после таймаута использует тот же ключ. Таймаут не означает, что операция не выполнилась.

### Ловушки данных

- `JsonToString` превращает Lua-массив в объект `{"1":…}`. Списки в ответе — только через `array({})`. Пустой список без `array()` придёт как `{}`, и `List` на клиенте не распарсится.
- Значения из `SqlQueryRows` могут приходить строками (`"12"`, `"1"`, `"43.25"`). Клиент обязан разбирать числа и флаги терпимо (раздел 6).
- Тип атрибута `double` создаёт `numeric(20,2)`. Координаты, курсы и другие точные дроби хранить `varchar(32)` (и приводить SQL-функцией) — иначе точность обрежется до 2 знаков.
- Исполнитель SQL заменяет `?` на параметры даже внутри строковых литералов и регулярных выражений — вместо `?` в регулярке писать `{0,1}`.
- Поиск без учёта регистра и «ё/е» делать в SQL (`lower`, `translate`), не на клиенте по всей выборке.

---

## 6. Клиент Flutter: HTTP и сессия

### Обязательные свойства клиента

1. Только HTTPS: если схема не `https` — ошибка до запроса.
2. Тело ответа декодировать из байтов: `jsonDecode(utf8.decode(response.bodyBytes))`. `response.body` берёт кодировку из `Content-Type`; без `charset=utf-8` кириллица превращается в «кракозябры».
3. Таймаут на каждый запрос (25 с) и отдельная ошибка «нет связи» (`code = NETWORK`).
4. Заголовки: `Authorization: Bearer <access>`, `Accept: application/json`, `Content-Type: application/json`.
5. Ошибки читать по порядку: `error.message` (объект) → `error` (строка) → `resultText` (ошибка платформы) → «Ошибка сервера (HTTP N)». 403 — понятный текст «нет роли …».
6. 401 → один раз обновить токен и повторить запрос; второй 401 — выход из сессии.
7. Параллельные запросы не должны запускать несколько refresh одновременно — общий `Future` «в полёте».
8. Обновлять access-токен заранее — за 45 секунд до `expiresAt`.
9. `invalid_grant` / `access_denied` при refresh — очистить хранилище и показать «Сессия завершена. Войдите снова.»
10. Токены хранить только в `flutter_secure_storage`, ключи с версией (`<app>_access_v1`). Не логировать токены, коды, verifier.

```dart
class ApiError implements Exception {
  const ApiError(this.status, this.message, [this.code = 'API_ERROR', this.field]);
  final int status;
  final String message;
  final String code;
  final String? field;
  @override
  String toString() => message;
}

Future<Map<String, dynamic>> call(String code, Map<String, dynamic> body, {bool retried = false}) async {
  final bearer = await token();
  if (bearer == null || bearer.isEmpty) throw const ApiError(401, 'Необходимо войти', 'UNAUTHORIZED');
  final base = Uri.parse(server);
  if (base.scheme != 'https') throw const ApiError(0, 'Требуется HTTPS', 'TLS_REQUIRED');
  final url = base.replace(path: '/restapi/services/run/$code');
  http.Response response;
  try {
    response = await _http
        .post(url,
            headers: {'Authorization': 'Bearer $bearer', 'Accept': 'application/json', 'Content-Type': 'application/json'},
            body: jsonEncode(body))
        .timeout(const Duration(seconds: 25));
  } catch (_) {
    throw const ApiError(0, 'Нет связи с сервером. Проверьте интернет и повторите.', 'NETWORK');
  }
  if (response.statusCode == 401 && !retried && onUnauthorized != null) {
    if (await onUnauthorized!()) return call(code, body, retried: true);
  }
  dynamic data;
  try {
    data = jsonDecode(utf8.decode(response.bodyBytes));
  } catch (_) {
    throw ApiError(response.statusCode, 'Сервер вернул непонятный ответ (HTTP ${response.statusCode})');
  }
  final object = data is Map ? Map<String, dynamic>.from(data) : <String, dynamic>{};
  if (response.statusCode >= 400 || object['success'] == false) {
    final raw = object['error'];
    if (raw is Map && raw['message'] != null) {
      throw ApiError(response.statusCode, raw['message'].toString(), (raw['code'] ?? 'API_ERROR').toString(), raw['field']?.toString());
    }
    final platform = text(object['resultText']);
    throw ApiError(response.statusCode,
        raw is String && raw.isNotEmpty ? raw : (platform.isNotEmpty ? platform : 'Ошибка сервера (HTTP ${response.statusCode})'));
  }
  final result = object['data'];
  return result is Map ? Map<String, dynamic>.from(result) : <String, dynamic>{};
}
```

Обновление токена без гонок:

```dart
Future<bool>? _refreshInFlight;

Future<bool> _renew() async {
  if (_refreshToken == null || kIsWeb) return false;
  final pending = _refreshInFlight ?? _renewDirect();
  _refreshInFlight = pending;
  try {
    return await pending;
  } finally {
    if (identical(_refreshInFlight, pending)) _refreshInFlight = null;
  }
}

Future<String?> _activeToken() async {
  if (_refreshToken != null &&
      (_token == null || _expiresAt == null || DateTime.now().isAfter(_expiresAt!.subtract(const Duration(seconds: 45))))) {
    await _renew();
  }
  return _token;
}
```

### Терпимый разбор ответов

DamuBPM может вернуть число строкой, флаг как `1`/`"1"`/`true`, пустой список как `{}` или `null`. Все страницы используют общие помощники вместо прямых приведений `as int` / `as List`:

```dart
Map<String, dynamic> asMap(dynamic v) => v is Map ? Map<String, dynamic>.from(v) : <String, dynamic>{};
List<Map<String, dynamic>> rows(dynamic v) =>
    v is List ? v.whereType<Map>().map((e) => Map<String, dynamic>.from(e)).toList() : <Map<String, dynamic>>[];
String text(dynamic v) => v == null ? '' : v.toString();
double numv(dynamic v) => v is num ? v.toDouble() : double.tryParse('$v'.replaceAll(',', '.')) ?? 0;
int intv(dynamic v) => v is num ? v.toInt() : int.tryParse('$v') ?? 0;
bool flag(dynamic v) => v == true || v == 1 || v == '1' || v == 'true';
```

### Вход OAuth на клиенте

```dart
export 'mobile_oauth_stub.dart' if (dart.library.io) 'mobile_oauth_native.dart';
```

```dart
AuthorizationServiceConfiguration get _configuration {
  final base = Uri.parse(server);
  if (base.scheme != 'https' || base.userInfo.isNotEmpty) throw const FormatException('Untrusted OAuth issuer');
  final issuer = base.origin;
  return AuthorizationServiceConfiguration(
    authorizationEndpoint: '$issuer/restapi/services/run/map_mobile_oauth_authorize',
    tokenEndpoint: '$issuer/restapi/services/run/map_mobile_oauth_token',
  );
}

final result = await _appAuth.authorizeAndExchangeCode(AuthorizationTokenRequest(
  clientId, redirectUrl, serviceConfiguration: _configuration, scopes: scopes));
```

- `clientId`, `redirectUrl`, `scopes` должны совпадать с константами сервера символ в символ.
- Схема из `redirectUrl` должна быть зарегистрирована: Android `manifestPlaceholders["appAuthRedirectScheme"]`, iOS `CFBundleURLTypes`. В DamuBPM это делается полем `flt_projects.url_scheme` + `deep_linking_off = 1` — вручную ничего править не нужно.
- Без `FlutterDeepLinkingEnabled = false` на iOS навигатор Flutter перехватывает callback, и пользователь возвращается на экран входа без токенов.
- После изменения Info.plist нужна пересборка и переустановка; hot reload не применяет его.
- Отмену входа распознавать по `PlatformException.code == 'authorize_and_exchange_code_failed'` или коду, содержащему `cancel` → «Вход отменён», без красной ошибки.
- Если токен не пришёл полностью (нет access или refresh) — ошибка, а не «полувход».
- Отладочный вход по личному токену допустим только при `!kReleaseMode`.
- Выход: сначала REST `<mod>_mobile_oauth_logout` (таймаут 5 с, ошибки игнорировать), затем очистка хранилища. Отдельный пункт «Выйти на всех устройствах» → `{'all': true}`.

### Устаревшие ответы и состояние

- Для запросов, которые могут обгонять друг друга (видимая область карты, поиск по вводу, маршрут), — счётчик последовательности: `final seq = ++_seq; … if (seq != _seq || !mounted) return;`.
- Ввод в поиске и движение камеры — с задержкой (debounce) 250–350 мс через `Timer`.
- После каждого `await` в `State` проверять `mounted` перед `setState`, `Navigator`, `ScaffoldMessenger`.
- Таймеры и `http.Client` закрывать в `dispose()`.

---

## 7. Карта MapLibre во Flutter (maplibre_gl)

- `maplibre_gl ^0.27.1` требует Flutter 3.29+ и iOS 13+ (в проектах задан iOS 15.0).
- URL стиля не зашивать в код: сервер отдаёт его в `config` (`mobile_style_url`, запасной `style_url`). Источник `pmtiles://` поддерживается MapLibre Native, но при проблемах указывать стиль с векторными тайлами по HTTP.
- Источники и слои добавлять только в `onStyleLoadedCallback`. До этого `setGeoJsonSource` падает. Держать флаг `_styleReady` и проверять его во всех методах отрисовки.
- Каждый `setGeoJsonSource`, `setLayerVisibility`, `queryRenderedFeatures` оборачивать в `try` — при пересоздании стиля они бросают исключения.
- `queryRenderedFeatures` на разных платформах возвращает `Map` или JSON-строку — нормализовать:

```dart
Map<String, dynamic> featureMap(dynamic f) {
  if (f is Map) return Map<String, dynamic>.from(f);
  if (f is String) {
    try { return Map<String, dynamic>.from(jsonDecode(f) as Map); } catch (_) {}
  }
  return {};
}
```

- Для своих слоёв `enableInteraction: false`, а клики обрабатывать в `onMapClick` с `featureTapsTriggersMapClick: true` и `queryRenderedFeatures(point, layerIds, null)`.
- `textFont` должен существовать в glyphs стиля (`['Noto Sans Regular']` для Liberty/OpenFreeMap), иначе подписи не появятся.
- Координаты в GeoJSON — `[lon, lat]`, в `LatLng` — `(lat, lon)`.
- Загрузка точек — по `onCameraIdle` + `getVisibleRegion()` с запасом 15% и серверной кластеризацией (`bbox` → `points` / `grid`). Тысячи точек без кластеров тормозят телефон.
- `trackCameraPosition: true`, если нужен `controller.cameraPosition`.
- `resizeToAvoidBottomInset: false` у `Scaffold` с картой — иначе клавиатура перерисовывает карту.
- Отступы контролов учитывать с `MediaQuery.of(context).padding.top/bottom` (вырез, жесты).
- Атрибуция OSM/OpenMapTiles должна оставаться видимой (`attributionButtonPosition`).

### Геолокация (geolocator)

Порядок: `isLocationServiceEnabled()` → `checkPermission()` → при `denied` `requestPermission()` → при `denied`/`deniedForever` понятное сообщение → `getCurrentPosition(locationSettings: LocationSettings(accuracy: high, timeLimit: 15 s))`. Включать `myLocationEnabled` на карте только после выданного разрешения.

### Внешние приложения

`launchUrl(uri, mode: LaunchMode.externalApplication)` и проверка результата. Для Android 11+ нужен блок `<queries>` — генерируемый манифест его уже содержит.

---

## 8. Разрешения, иконки, идентификаторы

### flt_permissions

| platform | perm_key | perm_value |
|---|---|---|
| android | `android.permission.INTERNET` | |
| android | `android.permission.ACCESS_FINE_LOCATION` | |
| android | `android.permission.ACCESS_COARSE_LOCATION` | |
| android | `android.permission.CAMERA` | |
| ios | `NSLocationWhenInUseUsageDescription` | Геолокация нужна, чтобы показать вас на карте и строить маршрут |
| ios | `NSCameraUsageDescription` | Камера нужна для сканирования штрихкодов и фото накладных |
| ios | `NSPhotoLibraryUsageDescription` | Выбор фото из галереи |
| ios | `ITSAppUsesNonExemptEncryption` | `false` |

- Для каждого iOS-разрешения, которое запрашивает код или плагин, нужен ключ `NS…UsageDescription` с понятным текстом на языке пользователя. Без него приложение падает при запросе, а App Store отклоняет сборку.
- `ITSAppUsesNonExemptEncryption = false` (HTTPS не считается собственным шифрованием) убирает ручной вопрос об экспортном контроле при каждой загрузке.
- `android.permission.INTERNET` добавлять явно — в release-манифесте Flutter его нет.
- `android_min_sdk = 23` — минимум для `flutter_secure_storage` 9.x и AppAuth.

### Идентификаторы

- Android `applicationId` = `android_application_id` или `<org>.<name из pubspec>` (подчёркивания допустимы: `kz.bapps.map_bapps`).
- iOS Bundle ID задаётся в `flt_cicd.ios_bundle_id` и применяется скриптом подписи. В Bundle ID подчёркивания недопустимы — использовать camelCase (`kz.bapps.mapBapps`). Bundle ID должен совпадать с приложением, созданным в App Store Connect.
- Код проекта (`flt_projects.code`) — с дефисами, `name` в pubspec — snake_case.

### Иконки

- Android: `android/app/src/main/res/mipmap-{m,h,xh,xxh,xxxh}dpi/ic_launcher.png` в `flt_files` (`is_base64 = 1`); при наличии `mipmap-hdpi` манифест получает `android:icon`.
- iOS: `ios/Runner/Assets.xcassets/AppIcon.appiconset/AppIcon-1024.png` + `Contents.json`. Иконка 1024×1024 для App Store должна быть без прозрачности (альфа-канала).
- Свои файлы в `flt_files` не перезаписываются `flutter create` из `tool/setup.sh` (существующие файлы сохраняются).

---

## 9. CI/CD: сборка iOS и публикация в App Store

Схема: DamuBPM → GitLab (`flt_projects_export_gitlab`) → GitLab Runner на собственном MacBook (executor `shell`, тег `macos`) → `xcodebuild` → App Store Connect / TestFlight.

### Запись flt_cicd (вкладка «Flutter: CI/CD (iOS, App Store)» в карточке проекта)

| Поле | Назначение |
|---|---|
| `runner_tag` | тег раннера, по умолчанию `macos` |
| `trigger_mode` | `manual` / `tag` / `push` |
| `auto_publish` | загружать IPA в App Store Connect автоматически |
| `ios_bundle_id`, `apple_team_id` | Bundle ID и Team ID (10 символов `[A-Z0-9]`) |
| `asc_key_id`, `asc_issuer_id`, `asc_key_p8` | ключ App Store Connect API; `.p8` после отправки в GitLab очищается, `asc_key_sent = 1` |
| `keychain_password` | пароль связки ключей login на маке |
| `app_version`, `build_number_offset` | версия (пусто — из pubspec) и сдвиг номера сборки |
| `flutter_bin_dir` | путь к `flutter/bin`, если его нет в PATH раннера |
| `dart_defines` | строки `KEY=value` для `--dart-define` |
| `last_pipeline_url`, `last_result`, `last_apply_at` | результат последнего действия |

БП `flt_cicd_apply` («Применить CI/CD») пишет в `flt_files` `.gitlab-ci.yml`, `tool/ci/ios.env`, `dart_defines.txt`, `ios_build.sh`, `ios_publish.sh`, `ios_signing.py`, `README.md`, записывает переменные GitLab (`ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_KEY_P8_B64` masked, `KEYCHAIN_PASSWORD`) и запускает выгрузку. БП `flt_cicd_run` («Собрать iOS») запускает pipeline с `IOS_RELEASE=1`.

Эти файлы CI не правят вручную ни в `flt_files`, ни в GitLab — они перезаписываются при следующем «Применить CI/CD». Меняются поля `flt_cicd`.

### Pipeline

`ios_build`: проверка macOS и инструментов → ключ `.p8` в `~/.appstoreconnect/private_keys/AuthKey_<id>.p8` → разблокировка keychain → `tool/setup.sh` → `ios_signing.py` (Bundle ID, Team, `CODE_SIGN_STYLE = Automatic` для Runner и RunnerTests) → `flutter build ios --release --config-only --build-name --build-number --dart-define…` → `xcodebuild archive` и `-exportArchive` с `-allowProvisioningUpdates` и API-ключом → IPA в артефактах (14 дней).
`ios_publish`: `xcrun altool --upload-app --apiKey --apiIssuer`. Сборка появляется в TestFlight через 5–30 минут.

### Ошибки и правила

- Раннер запускать в сессии пользователя: `brew services start gitlab-runner`. Как системный демон через `sudo` он не видит связку ключей — подпись падает.
- Xcode: `sudo xcode-select -s /Applications/Xcode.app`, `sudo xcodebuild -license accept`, `xcodebuild -runFirstLaunch`. Нужны `cocoapods`, Flutter 3.29+, `python3`.
- Ключ App Store Connect API — с ролью Admin: автоматическая облачная подпись (`-allowProvisioningUpdates`) создаёт сертификаты и профили.
- Метод экспорта зависит от Xcode: с 15.3 — `app-store-connect`, раньше — `app-store`. Скрипт выбирает его сам.
- Номер сборки = `CI_PIPELINE_IID + build_number_offset` и должен расти. Если репозиторий пересоздан или номер уже был загружен — увеличить `build_number_offset`, иначе App Store Connect отклонит загрузку как дубликат.
- В режиме `push` каждая выгрузка в ветку `main` (в том числе «Выгрузить в GitLab» из DamuBPM) запускает сборку на маке. Для частых правок использовать `manual` и запускать «Собрать iOS» осознанно.
- Токену проекта GitLab нужны Maintainer и scope `api` — для записи переменных и запуска pipeline.
- Для `LANG`/`LC_ALL` задан `en_US.UTF-8` — без этого CocoaPods падает на кириллице в путях и выводе.
- Приложение в App Store Connect с тем же Bundle ID создаётся вручную до первой загрузки.

---

## 10. Ловушки DamuBPM, встреченные при мобильной разработке

| Проблема | Решение |
|---|---|
| `string.match` = nil в Lua | `RegexpCheck`, `RegexpFindStringSubmatch` или SQL `~` |
| `SqlQueryRow2` без строки возвращает ошибку | проверка существования через `SqlQueryRows` |
| `unpack` = nil в REST | `local unpack = unpack or table.unpack` |
| `JsonToString` делает из массива объект | `array({})`, рекурсивный `arr()` для внешних данных |
| `?` внутри литерала SQL заменяется параметром | в регулярках `{0,1}`, в `db_execute_sql` — `chr(63)` |
| `Commit()` → `unexpected transaction status idle` | транзакцию уже закрыла платформа, данные записаны |
| `Commit()`/`Rollback()` в скрипте БП → «transaction not begin» | в БП не вызывать, коммит делает REST |
| `Rollback()` после `Commit()` ничего не откатывает | временные записи удалять явно |
| тип `double` → `numeric(20,2)` | точные дроби и координаты — `varchar(32)` |
| logic-элемент страницы с одним HTML падает `SyntaxError: Unexpected token '.'` | поле `logic` оставлять `NULL` |
| `create_page` перезаписывает автостраницы сущностей | не называть свои страницы кодами сущностей |
| правка `crontab` через SQL не подхватывается | только через `create_crontab` |
| MCP `flt_projects_create_update` падает | вставка SQL в `flt_projects`, страницы — `flt_pages_create_update` |

---

## 11. Порядок создания нового мобильного приложения

1. Сервер: модуль, роли, `pkg/<mod>_mobile`, REST `<mod>_api` (`config` + чтение), REST `<mod>_bp` с белым списком, бизнес-процессы с `input`/`result`/`last_error`.
2. OAuth: скопировать `map_mobile_oauth_*` → `<mod>_mobile_oauth_*`, заменить `CLIENT`, `REDIRECT`, `SCOPE`, `SYSTEM`, роли, имена операций лимита. `authorize` и `token` — `is_public = 1`, `only_auth = 0`; `logout` — `only_auth = 1`.
3. Проверить сервер без телефона: `authorize` без сессии отдаёт редирект на `/login`; `token` с неверным PKCE — `invalid_grant`; повтор refresh — отзыв сессии; `api` без Bearer — 401; `config` с ролью — 200; операции записи через `<mod>_bp` действительно меняют данные.
4. Проект `flt_projects`: `code`, `pubspec`, `app_name`, `org`, `url_scheme`, `deep_linking_off = 1`, `android_min_sdk = 23`, `ios_min_version = 15.0`, `android_generate = 1`, GitLab.
5. `flt_permissions`: INTERNET, нужные разрешения, iOS-тексты, `ITSAppUsesNonExemptEncryption = false`.
6. `flt_pages` по эталонной структуре (раздел 3), у каждой страницы `description = "Файл: lib/…"`.
7. Иконки в `flt_files` (`is_base64 = 1`).
8. Выгрузить в GitLab, локально `bash tool/setup.sh && flutter analyze && flutter run`.
9. `flt_cicd`: Bundle ID, Team ID, ключ ASC, `dart_defines`, режим → «Применить CI/CD» → «Собрать iOS».
10. Обновить описание проекта (`flt_projects.description`): REST-сервисы, схема, разрешения, параметры сервера — это документация для следующих правок.

## 12. Чек-лист перед релизом

- [ ] `flutter analyze` без ошибок.
- [ ] Вход, отмена входа, повторный запуск с сохранённой сессией, истечение access-токена (15 мин), refresh, выход, выход на всех устройствах.
- [ ] Снятие роли у пользователя → следующий refresh даёт «Сессия завершена».
- [ ] Нет сети / таймаут / 500 — понятные сообщения, приложение не падает.
- [ ] Кириллица в ответах отображается корректно (utf8 из байтов).
- [ ] Пустые списки и числа-строки не ломают экраны.
- [ ] Все операции записи проверены на сервере, а не только чтение.
- [ ] iOS: URL-схема в Info.plist, `FlutterDeepLinkingEnabled = false`, тексты разрешений.
- [ ] Android: `appAuthRedirectScheme`, INTERNET, `<queries>`.
- [ ] Иконки на обеих платформах, AppIcon 1024 без прозрачности.
- [ ] В исходниках нет токенов, паролей, ключей; адрес сервера — через `--dart-define`.
- [ ] Номер сборки больше уже загруженного в App Store Connect.
