# DamuBPM MCP (contact-center): ловушки инструментов и безопасный деплой

## 1. db_execute_sql

- Символ `?` внутри строковых литералов портится (трактуется как плейсхолдер). Писать `chr(63)` или избегать.
- Несколько операторов в одном вызове выполняются, но возвращается результат **только первого** SELECT; SELECT после UPDATE не покажет данные — проверять отдельным вызовом.
- Нельзя передавать BEGIN/COMMIT/ROLLBACK/SAVEPOINT.
- В `DO $$ … $$`/plpgsql давать уникальные алиасы (`generate_series(...) g(i)` конфликтует с колонкой `i` → «column i is ambiguous»).
- Временная функция для массовых замен: `create or replace function pg_temp.fn(...)` в том же вызове работает.

## 2. Страницы

- `create_page(uuid=<code>)` **перезаписывает существующую страницу с этим кодом**, включая автогенерированные страницы сущностей (`is_auto = 1`): список `<entity_code>` и карточку `<entity_code>details`. Перед созданием всегда проверять:
  ```sql
  select id, code, is_auto, entity_id from pages where code = '<code>';
  ```
  Свои страницы называть так, чтобы не совпадать с кодами сущностей (`beti`, `beti_tpl`, а не `beti_templates`).
- URL страницы = `/<code>`.
- Автостраницы списков сущностей отличаются только кодом сущности, `entity_id = '…'` и `page_id = '…'` в angular_json — испорченную можно восстановить копией соседней с заменой этих трёх значений.
- Большие страницы (50–100 КБ) дешевле ставить крон-установщиком (см. п. 5), чем base64 в create_page.

## 3. Сущности и БП

- `create_entity` не добавляет `title`, `created_at`, `updated_at`, `created_by` — добавлять через `add_entity_attr`.
- Тип `double` создаёт `numeric(20,2)` — для координат и точных чисел использовать `varchar`/`numeric` нужной точности.
- `create_bp_process` по шаблону `mcp_method_create_update_template` может вернуть «no row found», но процесс создаётся. Шаблон добавляет обязательную переменную `input` (jsonb) — удалить её, иначе БП не запустится без input.
- Тип БП (entityTool, entityActionByID), `action_entity_id`, иконку, `button_title` меняют SQL-апдейтом `bp_processes` (type_id из `bp_process_types`).
- `update_bp_points_usertask` работает и для EndEvent (экран завершения).
- Условия на стрелках шлюза: `var.last_error == nil or var.last_error == ""` и обратное.
- Запуск из JIT: `this.appComponent.bpRun('<код БП>', {}, cb)`.

## 4. Меню, CSS, каталоги

- Меню: `angular_menu` (parent New features, `place_id = 3`) + `angular_menu_role`. Новую роль модуля добавлять и в роли родительского пункта, иначе пункт не виден.
- `create_css(code="mod/main.css")` → доступен по `../cms/css/mod/main.css`; подключать из JIT через `<link>` с `document.baseURI`.
- Для загрузки файлов из REST нужен каталог в `dirs` (code, unix_path внутри `/opt/cloud/dirs/...`, module_id).

## 5. Крон-задачи как инструмент деплоя и тестов

- Большой Lua/HTML/JS кладётся в БД скриптом крон-задачи с длинными строками `[=====[ … ]=====]` (проверить, что внутри нет `]=====]`).
- **Крон-задача выполняется повторно по расписанию.** Установщик, оставленный включённым, откатит все последующие правки. После проверки результата сразу: `is_active=false`, скрипт `return`.
- Одноразовость: первой строкой проверять маркер в `crontab.warning_text` и выходить, если задача уже отработала.
- Результат писать в `crontab.warning_text` и читать через db_execute_sql.
- Патчи REST/пакетов делать не полной перезаписью, а заменой фрагмента (`StrContains` → `StrReplace` или вырезка между маркерами `string.find`), с проверкой «уже применено».

## 6. Тестирование REST без браузера

Крон-задача загружает тело сервиса и выполняет его с подставленным `request`:
```lua
local src = SqlQueryRow2([[select body from rest_services where code = 'my_api']]).body
local function run(action, data)
  local done = false
  local chunk, err = load(function() if done then return nil end done = true return src end)
  if not chunk then return {success = false, error = {message = tostring(err)}} end
  request = {user_id = 1, input = {action = action, data = data or {}}, method = "POST", header = {}, get = {}}
  output = nil
  local ok, e = pcall(chunk)
  if not ok then return {success = false, error = {message = tostring(e)}} end
  return output
end
```
- Покрывать тестом **операции записи** (save/insert/update), а не только чтение — ошибки вроде `unpack == nil` проявляются только там.
- Синтаксис Lua до деплоя проверять локально (LuaJIT/`lupa`: `loadstring(src)`).
- Синтаксис JIT-класса проверять `node -e "new Function('rxjs','forms','data', src)"`.
