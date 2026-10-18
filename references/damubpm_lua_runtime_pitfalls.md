# DamuBPM Lua: неочевидное поведение рантайма

Проверено на практике (contact-center, модуль beti). Читать перед написанием REST-сервисов, пакетов `pkg/*`, скриптов БП и крон-задач.

## 1. Стандартная библиотека урезана

| Что | Факт | Как писать |
|---|---|---|
| `unpack` | В REST-сервисах **nil** → `attempt to call a nil value` (сообщение может ошибочно указывать на соседнюю локальную переменную) | В начале скрипта: `local unpack = unpack or table.unpack` |
| `loadstring` | nil | `load(function() ... end)` — `load` с функцией-итератором работает |
| `string.gsub` | запрещён правилами проекта | `StrReplace(s, old, new, -1)`, `RegexpReplace*`, `RegexpFindAllStringsAndJoin` |

## 2. JSON

- `JsonToString` сериализует **обычную Lua-таблицу-массив как объект**: `{"1":…,"2":…}`. Пустая таблица `{}` → `{}` (объект).
  - Для массивов всегда `array({...})`. Если данные пришли извне (StringToJson, decode) и уходят обратно в JSON — пропускать через рекурсивный конвертер:
    ```lua
    local function arr(v)
      if type(v) ~= "table" then return v end
      local n = 0
      for _ in pairs(v) do n = n + 1 end
      if n > 0 and #v == n then
        local a = array({})
        for i = 1, n do a[i] = arr(v[i]) end
        return a
      end
      local o = {}
      for k, x in pairs(v) do o[k] = arr(x) end
      return o
    end
    ```
  - Уже испорченный jsonb в БД чинится SQL-функцией, которая рекурсивно превращает объекты с ключами `"1".."n"` в массивы.
- `request.input` в REST уже распарсен в таблицу; вложенные массивы приходят как Lua-массивы — перед `JsonToString` снова нужен `arr()`.

## 3. Сигнатуры, которые легко перепутать

| Функция | Правильно |
|---|---|
| `FileContent(fileId, uid)` | возвращает `filename, contentType, data, err, code` (не `data` первым) |
| `RegexpFindAllStringsAndJoin(expr, str, count, sep)` | сначала регулярка, потом строка; `count = -1` — все |
| `RegexpFindStringSubmatch(text, pattern, group)` | текст первым |
| `UploadRawData(dir, fileName, data, userId)` | возвращает `uuid, err, code`; `dir` — код из таблицы `dirs`, каталог должен существовать |
| `SqlQueryRow2` | `n == 2` — «строка не найдена» (не ошибка) |
| `SqlExec2(q, ...)` | `nil` в аргументах → `NULL`, это нормально |
| `pkg/http.post(url, body, headers, timeout)` | возвращает `body, err, code` |
| `EncodeToICAO` | не транслитерирует казахские буквы (ә, ғ, қ, ң, ө, ұ, ү, һ, і) — для slug нужна своя таблица |

## 4. Транзакции

- `Commit()` может вернуть `pq: unexpected transaction status idle`, если транзакцию уже закрыла платформа (например, после `UploadRawData`). Данные при этом записаны. В обёртке commit игнорировать именно этот текст:
  ```lua
  local function commit()
    local e, n = Commit()
    if n ~= 0 and not StrContains(tostring(e), "transaction status idle") then error("Не удалось сохранить изменения: " .. tostring(e)) end
  end
  ```
- После `Commit()` дальнейшие запросы фактически автокоммитятся: **`Rollback()` их не откатит**. Временные записи (превью, «песочницы») удалять/восстанавливать явно, в `pcall`, а не через Rollback.
- В крон-задачах семантика транзакций отличается от REST: тест в кроне, прошедший с Rollback, не доказывает, что Rollback сработает в REST.

## 5. Ошибки и отладка

- Если обёртка ошибок срезает `файл:строка:` (cleanErr), при отладке временно отдавать `tostring(err)` целиком — номер строки решает 90% случаев.
- В тексте, который пишется в `warning_text`/varchar, вырезать `string.char(0)`.
- `pcall(dbrequire, "pkg/x")` — пакет может не загрузиться; проверять `type(B) == "table"`.

## 6. Файлы и окружение сервера

- Процесс DamuBPM работает от пользователя `freeswitch`. `/opt/cloud` принадлежит root, писать можно только в `/opt/cloud/dirs` и `TempDir()`. Для собственной папки нужен `sudo mkdir … && sudo chown -R freeswitch …` — сообщать пользователю заранее.
- Атомарная публикация: собирать во временной папке, затем `mv` (через `Command`).

## 7. ParseTemplate (Go text/template)

- Работают `range`, `index`, `if`, `with`, `printf`, `lt`, `HasPrefix`, `QueryEscape`; отсутствующий ключ = false.
- Вложенные массивы допустимы, но массивы должны быть `array()`, иначе `range` идёт по объекту.
- Проверять шаблон до сохранения: `local _, e, n = ParseTemplate(tpl, sample, tostring(uid))`, `n ~= 0` — ошибка.
