# Печатные формы DamuBPM: exp_templates и БП exp_template

[← На главную](index.html)

Справочник по устройству печатных форм (шаблонов экспорта) DamuBPM: таблица `exp_templates`, связанные справочники, виды шаблонов, БП `exp_template` (код «Экспорт»), процессы обслуживания шаблонов и типовые ошибки. Материал собран по живой базе onedoc (12.11.2026): 264 шаблона по 125 сущностям, скрипты БП и Lua-функции генерации.

Синтаксис `{{ }}` и функции шаблонизатора подробно описаны в [parse_template.md](parse_template.md); здесь — только то, что относится к печатным формам.

## Содержание

- [1. Как это работает в двух словах](#1-как-это-работает-в-двух-словах)
- [2. Таблица exp_templates](#2-таблица-exp_templates)
- [3. Справочники и связанные таблицы](#3-справочники-и-связанные-таблицы)
- [4. Источник данных: query и detail](#4-источник-данных-query-и-detail)
- [5. Виды шаблонов и образцы](#5-виды-шаблонов-и-образцы)
- [6. Функции в шаблонах](#6-функции-в-шаблонах)
- [7. БП exp_template](#7-бп-exp_template)
- [8. Как запускать печать](#8-как-запускать-печать)
- [9. FODT «bottom line»: импорт docx и разметка полей](#9-fodt-bottom-line-импорт-docx-и-разметка-полей)
- [10. Процессы обслуживания шаблонов](#10-процессы-обслуживания-шаблонов)
- [11. Ловушки](#11-ловушки)
- [12. Чек-лист новой печатной формы](#12-чек-лист-новой-печатной-формы)

---

## 1. Как это работает в двух словах

1. Шаблон — строка в `exp_templates.template` (Go text/template + HTML, FODT XML или CSV).
2. Пользователь жмёт «Печать» в списке или карточке → фронт запускает БП `exp_template` с `url` (какие записи) и `entity_id`.
3. БП выбирает шаблон (по коду, по `autoprint` или через выбор пользователя), собирает данные (`Query` или `Detail`) в массив `items`, рендерит `ParseTemplate`.
4. Результат конвертируется LibreOffice (`soffice --headless --convert-to pdf|xls|docx`) и загружается в `files` через `UploadRawData`.
5. PDF показывается в модальном окне (`<embed>`), остальные форматы скачиваются сразу.

```text
UI print() / кнопка карточки / BPMSStartProcess2
        │  url="?code=<query|entity>&ids=1,2", entity_id, [template_id|template_code]
        ▼
   БП exp_template ──► Query(url) / Detail(id, detail_code) ──► items
        ▼
   ParseTemplate(template, {user_id, items, last_file_uuid})
        ▼
   soffice → pdf/xls/docx  ──► UploadRawData ──► files.code
        ▼
   PDF: предпросмотр  |  прочее: скачивание
```

---

## 2. Таблица exp_templates

Сущность `exp_templates` (модуль `core`, id 697).

| Поле | Тип | Назначение |
|---|---|---|
| `id` | bigserial | ID шаблона |
| `code` | varchar | Код. Используется в `template_code`, в `EntityValueByCode("exp_templates",...)` |
| `title` | varchar | Название в списке выбора шаблонов |
| `entity_id` | → `entities` | Сущность, к которой относится шаблон (по ней строится список доступных) |
| `src_id` | → `exp_temp_srcs` | Источник данных: `1 query` (из запроса) / `2 detail` (из детали) |
| `detail_id` | → `details` | Деталь для источника `detail`. Обычно авто-деталь с кодом сущности |
| `ftype_id` | → `exp_tmp_ftypes` | Тип выходного файла (pdf, xls, docx, csv…) |
| `filename` | varchar | Имя файла без расширения; расширение берётся из `exp_tmp_ftypes.ext` |
| `template` | text | Сам шаблон |
| `autoprint` | boolean | «Автоматический предпросмотр без выбора»: если у сущности ровно один такой шаблон, он выбирается без диалога |
| `autopreview` | boolean | Отдаётся в датасете `exp_templates$` детали; БП `exp_template` его не читает |
| `only_users` | boolean | Доступен только пользователям из `exp_templates_user` |
| `is_sql_condition` | boolean | Включить фильтр доступности по SQL |
| `sql_condition` | text | Условие по таблице сущности с алиасом `main`; `:user_id` подставляется |
| `signed` | boolean | «Подписывать?» — флаг, в БП `exp_template` не используется |
| `docx_file_id` | → `files` | Рабочая копия docx при онлайн-редактировании (WebDAV) |
| `dev_notes` | text | Заметки разработчика (что правили руками после импорта и т.п.) |
| `bl_fodt_orig` | text | FODT после конвертации загруженного docx (bottom line) |
| `bl_fodt_with_ids` | text | FODT с маркерами `_FIELD_n_`, `_ROW_…`, `_COL_…`, `_BLOCK_…` |
| `bl_html` | text | HTML-представление для визуальной разметки полей |
| `bl_max_field` | bigint | Количество найденных полей `_FIELD_n_` |
| `bl_map` | jsonb | Карта полей `f<n>` |
| `bl_map_r` | jsonb | Карта табличных строк `r<k>` (какой датасет повторять) |
| `bl_map_c` | jsonb | Карта колонок `c<row>_<col>` |
| `bl_map_b` | jsonb | Карта условных блоков `b<k>` |

Действия на карточке шаблона (bp_processes сущности): «Мастер создания шаблона экспорта», «Импорт шаблона (docx)», «Применить изменения», «Экспорт шаблона в Docx», «Онлайн редактирование», «Прекратить онлайн-редактирование», «Скопировать», «Экспорт шаблона в файл» — см. [раздел 10](#10-процессы-обслуживания-шаблонов).

---

## 3. Справочники и связанные таблицы

### exp_tmp_ftypes — типы файлов

| id | code | ext | Как генерируется в БП |
|---:|---|---|---|
| 1 | `pdf` | `.pdf` | `generatePdfbyHTML` → `soffice --writer --convert-to pdf` |
| 2 | `xml` | `.xml` | `ParseTemplate` + `UploadRawData` (сырой текст) |
| 3 | `xls` | `.xls` | `generateXlsbyHTML` → `soffice --calc --convert-to xls` |
| 4 | `html` | `.html` | сырой текст |
| 5 | `csv` | `.csv` | сырой текст |
| 6 | `docx` | `.docx` | только источник `detail`: `generateDocxbyHTML` → `soffice --writer --convert-to docx` |
| 7 | `xlshtml` | `.xls` | сырой HTML с расширением `.xls` (Excel открывает HTML) |

Ветка `xlsx` (`generateXlsxbyHTML`) есть в скрипте Query, но типа `xlsx` в справочнике нет.

### exp_temp_srcs — источники данных

| id | code | title |
|---:|---|---|
| 1 | `query` | Из запроса |
| 2 | `detail` | Из детали |

### Прочие таблицы

| Таблица | Назначение |
|---|---|
| `exp_templates_user` | `exp_templates_id`, `user_id` — кому доступен шаблон при `only_users=1` |
| `exp_result` | `entity_pk` (sys$uuid записи), `file_id` — «эталонный» файл записи; используется для наложения штампа ЭЦП |
| `template_funcs` | `code → script`: Lua-функции, вызываемые в шаблоне через `{{lua "code" ...}}` |
| `exp_design` | Дизайны для мастера (`simple` — «Простой дизайн», PDF) |
| `luas` | `generatePdfbyHTML`, `generateXlsbyHTML`, `generateXlsxbyHTML`, `generateDocxbyHTML`, `generatePdfbyHTML2`, `convertHTML2Docx` |
| `its_r_statements_onedoc_t`, `its_r_prikaz_pers2_onedoc_t` | Прикладные справочники «тип файла → `exp_template_id`» для пакетной генерации документов |

### Генераторы (luas)

Все `generate*byHTML(dir, filename, html, arr, user_id)` устроены одинаково:

```lua
res, errText, errNum = ParseTemplate(html, arr, user_id)
res = StrReplace(res, "<no value>", "", -1)       -- только pdf и docx
WriteFile(TempDir().."/"..uuid..".html", res)
Command("soffice", "--headless", "--writer", "--convert-to", "pdf", tempFile, "--outdir", TempDir())
s = ReadFile(TempDir().."/"..uuid..".pdf")
uuid = UploadRawData(dir, filename, s, user_id)  -- возвращает files.code
```

Временный файл всегда пишется с расширением `.html`, но LibreOffice определяет формат по содержимому, поэтому FODT-шаблоны (`<?xml …><office:document …>`) тоже конвертируются корректно.

### Статистика onedoc (12.11.2026)

| Показатель | Кол-во |
|---|---:|
| Всего шаблонов | 264 |
| Сущностей с шаблонами | 125 |
| `detail` → PDF: HTML / FODT | 85 / 28 |
| `detail` → DOCX: HTML / FODT | 26 / 10 |
| `query` → XLS (реестры), из них `<entity>$xls$page` | 97 / 65 |
| `query` → CSV | 2 |
| С `autoprint=1` | 56 |
| С `is_sql_condition=1` | 66 |
| С `only_users=1` | 7 |
| Импортированы из docx (bottom line, `bl_html` заполнен) | 39 |
| Созданы мастером (`…$from_wizard$N`) | 8 |

---

## 4. Источник данных: query и detail

Корень данных шаблона всегда один и тот же:

```lua
{ user_id = sys.user_id, items = items, last_file_uuid = "<uuid>" }  -- last_file_uuid только в detail
```

### src = query

`items` — строки запроса, полученные `Query(var.url, sys.user_id)`. Одна строка = одна запись. Ссылочные поля приходят парами: `dep_id` (id) и `dep_id$` (title), вложенные — `created_by$title_rp` и т.п.

```text
{{ range $item_key, $item_value := .items }}
  {{index $item_value "title"}} — {{index $item_value "stat_id$"}}
{{ end }}
```

Значения **не экранируются** — HTML в данных попадёт в документ как есть.

### src = detail

Для каждой строки запроса вызывается `Detail(v.id, <details.code>, sys.user_id)`. `items[i]` — это **объект датасетов детали**: ключ — код `detail_queries`, значение — массив строк.

```text
.items[i]
  ├─ its_hr_premiya        ← основной датасет (код = код сущности), 1 строка
  ├─ its_hr_premiya_user   ← табличная часть
  ├─ its_hr_premiya_file
  ├─ task_approve_list$    ← служебные датасеты авто-детали (с $ на конце)
  ├─ exp_templates$, workflows$, workflow_stat$, error_rows$ …
```

Канонический каркас detail-шаблона:

```text
{{ range $item_key, $item_value := .items }}
  {{ range $key, $value := $item_value.its_hr_premiya }}
    Номер {{index $value "docnum"}} от {{ TimeParseFormat (index $value "created_at") "2006-01-02 15:04:05" "02.01.2006" }}
    Инициатор: {{index $value "created_by$"}}
  {{ end }}

  {{ range $key, $value := $item_value.its_hr_premiya_user }}
    {{inc $key 1}}. {{index $value "user_id$"}} — {{index $value "amount"}}
  {{ end }}

  {{ range $key, $value := index $item_value "task_approve_list$" }}
    {{index $value "approve_res_id$title"}} {{index $value "manager_id$"}}
  {{ end }}
{{ end }}
```

Правила detail-источника:

- Датасеты с `$` в коде читаются только через `index $item_value "код$"` — точечная запись не работает.
- Все строковые значения датасетов проходят `HTMLEscapeString`, **кроме ключей с префиксом `fodt`** (`fodt_text`, `fodt_styles` …). Чтобы вывести HTML из поля, используйте поле `fodt*` или функцию `{{lua "StrReplace" …}}`, которая возвращает `&lt;`/`&gt;` обратно.
- Связь «строка ТЧ → строки вложенной ТЧ» делается фильтром внутри `range`: `{{if eq $value.id $value_inner.parent_id}}` (см. `its_protocol$pdf`, датасет `…$cross$`).
- Датасет `exp_templates$` авто-детали отвечает за показ кнопки печати в карточке; в нём жёстко зашит `entity_id` сущности.

### Формат url

| Вызов | url |
|---|---|
| Карточка записи | `?code=<код сущности>&ids=<id>` |
| Выбранные строки списка | `?code=<код query списка>&ids=1,2,3` |
| Весь список с фильтрами | `?` + `query_options.url` без `page`, `perpage`, `loader` |

---

## 5. Виды шаблонов и образцы

### 5.1 Реестр списка в Excel (`<entity>$xls$page`, query → xls)

Генерируется автоматически для списков; 65 шаблонов почти идентичны (≈1 622 символа).

```html
<html>
{{$items := .items}}
<head>
  <meta http-equiv="content-type" content="text/html; charset=utf-8" />
  <style type="text/css">
    @page { size: landscape; margin-left: 1cm; margin-right: 1cm }
    p { font-family: "Arial"; font-size: 7pt }
  </style>
</head>
<body>
  <table cellpadding=0 cellspacing=0 style="border:solid 1px">
    <tr>
      <th style="border:solid 1px">Должность</th>
      <th style="border:solid 1px">Статус</th>
      <th style="border:solid 1px">Сумма</th>
    </tr>
    {{ range $item_key, $item_value := .items }}
    <tr>
      <td>{{index $item_value "hr_position_id$" }}</td>
      <td>{{index $item_value "stat_id$" }}</td>
      <td>{{index $item_value "amount" }}</td>
    </tr>
    {{ end}}
  </table>
</body>
</html>
```

### 5.2 CSV (query → csv)

Без HTML; разделитель — `\t` или `;`, строки — `\r\n`.

```text
{{$items := .items}}id	title	email
{{ range $item_key, $item_value := .items }}{{index $item_value "id"}}	{{index $item_value "title"}}	{{index $item_value "email"}}
{{ end}}
```

### 5.3 Карточка в PDF/DOCX из HTML (detail → pdf/docx)

Самый массовый вид (111 шаблонов). HTML обычно получен сохранением Word/LibreOffice в HTML, затем вставлены `{{range}}` и поля. `@page` задаёт формат листа и поля:

```html
<style type="text/css">
  @page { size: 21cm 29.7cm; margin: 2cm }
  p { margin-bottom: 0.25cm; line-height: 115% }
</style>
```

Альбомная ориентация — `@page { size: landscape }`. Картинки — абсолютный URL `…/restapi/getfile?code=…` или base64 через `{{lua "file_base64_by_file_id" …}}`.

### 5.4 FODT (detail → pdf/docx)

38 шаблонов — Flat ODT (`<?xml …?><office:document …>`), почти все получены импортом docx ([раздел 9](#9-fodt-bottom-line-импорт-docx-и-разметка-полей)). Каркас, который ставит процесс «Применить изменения»:

```text
{{$last_file_uuid:=.last_file_uuid}}{{$items:= .items}}{{range $item_key, $item_value := .items}}{{range $key, $value := $item_value.<entity_code>}}<?xml version="1.0" encoding="UTF-8"?>
<office:document …>
  … {{index $value "docnum"}} … {{ lua "task_list_approve" (index $value "sys$uuid") }} …
</office:document>{{end}}{{end}}
```

Внутри FODT разрыв строки — `<text:line-break/>`, поэтому Lua-функции для FODT склеивают значения через него (`task_hst_users_info_fodt`, `task_user_approve_list`). QR-код — `{{lua "fodt_qr" <текст> <width> <height> <x> <y> <style>}}`, возвращает `<draw:frame>` с base64 PNG.

### 5.5 Шаблон-обёртка (HTML хранится в записи)

Когда текст документа формируется в самой записи (редактор CKEditor, генерация по шаблону), печатная форма лишь выводит поле:

```text
{{ range $item_key, $item_value := .items }}
{{ range $key, $value := $item_value.its_memo }}
    {{index $value "template"}}
{{end}}{{end}}
```

Учитывайте экранирование detail-источника (см. [раздел 4](#src--detail)).

### 5.6 Мастер (`<entity>$from_wizard$N`)

Процесс `exp_template_wizard` берёт дизайн из `exp_design`, рендерит его по метаданным сущности (`Detail(entity_id, "entities_for_gen")`) и создаёт шаблон с `src=detail`, `detail_id` = деталь с кодом сущности. Результат — готовая заготовка для правки.

### 5.7 Особые случаи

| Шаблон | Что делает |
|---|---|
| `bp_processes_doc` (ftype html) | Выводит `svg` процесса: `{{(index $item_value.bp_processes 0).svg}}` |
| `its_hr_test_req06_lua` | Весь шаблон — `{{ lua "hello_world" "177755" }}`; документ целиком собирается Lua-функцией |
| `its_budget`, `its_time_sheet` | Динамические колонки: `Iterate 1 12`, `index $value (print "day_r_" $val "$")` |
| `its_protocols_v*` | Несколько вариантов одной формы, выбор по `sql_condition` (`main.collegial in (…)`) |
| `its_memo$`, `its_in$xlsx`, `its_out$pdf` | `sql_condition = 1 = 0`: скрыты из выбора, вызываются из Lua по `template_id` |

---

## 6. Функции в шаблонах

Частота по 264 шаблонам onedoc:

| Функция | Вызовов | Пример |
|---|---:|---|
| `index` | 4 320 | `{{index $value "created_by$"}}` |
| `TimeParseFormat` | 326 | `{{ TimeParseFormat (index $value "created_at") "2006-01-02 15:04:05" "02.01.2006" }}` |
| `lua` | 227 | `{{lua "rus_date" (TimeParseFormat (index $value "act") "02.01.2006" "02 January 2006")}}` |
| `eq` / `ne` | 205 | `{{if eq $value2.step_nn "1"}}` |
| `isnil` | 117 | `{{if isnil $value.docnum}}ПР-{{$value.id}}{{else}}{{$value.docnum}}{{end}}` |
| `inc` | 28 | `{{inc $key 1}}` — нумерация строк |
| `Iterate` | 16 | `{{range $val := Iterate 1 (index $value "day_count")}}` |
| `print` | 9 | построение имени ключа |
| `len` | 9 | `{{len $item_value.its_protocols_users}}` |
| `EntityValueById` | 3 | `{{ EntityValueById "users" "title" (index $value_inner "user_id") }}` |
| `DBCurrentDateTime`, `formatNumber`, `GetParamValue` | по 2–3 | |

Самые используемые `template_funcs`:

| Код | Назначение |
|---|---|
| `format_date` | `YYYY-MM-DD…` → `ДД.ММ.ГГГГ` |
| `rus_date` / `kaz_date` | Замена английских месяцев на русские/казахские (после `TimeParseFormat … "02 January 2006"`) |
| `StrReplace` | Замена + возврат `&lt;`/`&gt;` в `<`/`>` (вывод HTML из экранированных полей) |
| `file_base64_by_file_id` | Base64 файла (картинки в документе) |
| `ckeditorTableBorder` | Добавляет `border='1'` таблицам из CKEditor |
| `ruNum2Word` / `kzNum2Word` / `num_to_text_kz` | Сумма прописью |
| `priceSpace` | Разделители разрядов |
| `dayOfWeek` | День недели (табели) |
| `task_list_approve`, `task_user_approve_list`, `task_hst_users_info_fodt/_br/_fyi_fodt*` | Лист согласования / ознакомления из `tasks`/`task_hst` (HTML `<br />` или FODT `<text:line-break/>`) |
| `task_user_*_by_step_nn`, `task_hst_users_*bytitlefodt` | ФИО/должность согласующего по шагу (`argv1` = sys$uuid записи, `argv2` = step_nn или название шага) |
| `fodt_qr` | QR-код в FODT |
| `root_*_prikaz`, `root_head_*` | Прикладные: реквизиты руководителя для приказов |

В `lua` аргументы приходят как `argv1…`, функция обязана вернуть значение. Новую функцию добавляйте в `template_funcs`, а не в шаблон.

---

## 7. БП exp_template

Код `exp_template`, название «Экспорт», тип `businessprocess`, модуль `core`, id 313.

### Переменные

| Переменная | Тип | Вход/выход | Назначение |
|---|---|---|---|
| `url` | longtext | вход, обяз. | Строка `Query` — какие записи печатать |
| `entity_id` | reference entities | вход | Сущность (список шаблонов, таблица для `sql_condition`, деталь) |
| `template_id` | integer | вход | Шаблон задан заранее — без выбора |
| `template_code` | varchar | вход | То же по коду |
| `nopreview` | boolean | вход | `1` — не показывать PDF, завершить на «Без предпросмотра» |
| `add_to_exp_result` | boolean | вход | `1` — записать файл в `exp_result` как эталон записи |
| `avaiable_templates` | struct | внутр. | Список доступных шаблонов для выбора |
| `cnt` | integer | внутр. | Количество документов |
| `last_error` | varchar | выход | Текст ошибки |
| `file_uuid` | varchar | выход | `files.code` результата (**только источник detail**) |
| `file_url` | varchar | выход | `/restapi/getfile?code=…&attachment=true` |
| `file_url_woa` | varchar | выход | Ссылка без attachment (для `<embed>`) |

### Маршрут

```text
Start
 └► Calc TemplateId
     └► Общее количество документов
         └► Рассчитать доступные шаблоны
             └► ◇ template_id задан?
                 ├─ нет ► [UserTask] Выберите шаблон ─┐
                 └─ да ───────────────────────────────┤
                                                      ▼
                                   ◇ exp_temp_srcs.code шаблона
                     ┌──────── query ─────────┴──────── detail ────────┐
                     ▼                                                 ▼
                   Query                                            Detail
                     │                                                 ▼
                     │                                       Add to exp_result
                     │                                                 ▼
                     │                              Наложим штамп из ЭЦП exp_result
                     │                                                 ▼
                     │                                              Ссылки
                     └───────────────────────┬─────────────────────────┘
                                             ▼
                                ◇ last_error ~= ""  ──► [End] Ошибка
                                             ▼ без ошибки
                                ◇ ftype == pdf ?
                       ├─ нет ► [End] Close (скачивание location.href = file_url)
                       └─ да ► ◇ nopreview == "1" ?
                                   ├─ нет ► [End] PDF (embed file_url_woa)
                                   └─ да  ► [End] Без предпросмотра
```

Условия стрелок (Lua):

| Стрелка | Условие |
|---|---|
| Шаблон ещё не выбран | `var.template_id == nil or var.template_id=="" or var.template_id==0` |
| Шаблон уже выбран | `var.template_id~=nil and var.template_id~=0` |
| Query | `EntityValueById("exp_temp_srcs","code", EntityValueById("exp_templates","src_id",var.template_id)) == "query"` |
| Detail | то же `== "detail"` |
| С ошибкой / Без ошибки | `var.last_error ~= ""` / `var.last_error == ""` |
| PDF / не PDF | `EntityValueById("exp_tmp_ftypes","code", EntityValueById("exp_templates","ftype_id",var.template_id)) == "pdf"` / `~= "pdf"` |
| Предпросмотр | `var.nopreview ~= "1"` / `== "1"` |

### Шаги

**Calc TemplateId.** Если передан `template_code` — `select id from exp_templates where code=?`. Иначе ищет шаблоны сущности с `autoprint = 1`; если такой ровно один — выбирает его (`template_id`, `template_code`). Если передан `template_id`, шаг его не трогает.

**Общее количество документов.** `Query(var.url)` → `var.cnt` (выводится в диалоге выбора).

**Рассчитать доступные шаблоны.**

```sql
select coalesce(e.sql_condition,'') sql_condition, coalesce(e.is_sql_condition,0) is_sql_condition, e.id, e.title
from exp_templates e
where e.entity_id = ?
  and (coalesce(e.only_users,0) = 0
       or exists (select 1 from exp_templates_user ee where ee.exp_templates_id = e.id and ee.user_id = ?))
```

Для шаблонов с `is_sql_condition` по **каждой** записи из `url` выполняется
`select count(1) cnt from <entity_code> main where (<sql_condition>) and main.id=?` (`:user_id` заменяется на `'<id>'`). Если хоть одна запись не прошла — шаблон скрывается.

**Выберите шаблон (UserTask).** Таблица ссылок по `vars.avaiable_templates`; клик ставит `vars.template_id` и вызывает `runTask`.

**Query** (src = query). `Query(url)` → `items`; для `pdf`/`xls`/`xlsx` — соответствующий `generate*byHTML`, иначе `ParseTemplate` + замена `&lt;!`→`<!`, `&lt;?`→`<?` + `UploadRawData("CUST", …)`. Ставит `file_url`, `file_url_woa`; `file_uuid` **не ставит**.

**Detail** (src = detail).
1. `last_file_uuid = UUID()`.
2. По каждой строке `Query(url)`: `Detail(v.id, details.code, user)`, все датасеты → `array()`, строковые значения → `HTMLEscapeString` (кроме ключей `fodt*`); `items[i] = detail`.
3. Типографские кавычки `“ ”` в шаблоне заменяются на `"`.
4. `Commit()`.
5. Кеш: `detail_md5 = Md5(JsonToString({template, template_id, items}))`; если в `files` есть файл с таким `detail_md5` — берётся он, генерация пропускается.
6. Генерация по `ftype`: `pdf` → `generatePdfbyHTML`, `xls` → `generateXlsbyHTML`, `docx` → `generateDocxbyHTML`, прочее — сырой текст. Каталог `EXP_TEMPLATE`.
7. `update files set code = last_file_uuid` — код файла заранее известен шаблону (`{{$last_file_uuid}}`), поэтому в документ можно вшить QR-ссылку на сам себя.
8. `update files set detail_md5 = …`.
9. Прикладные хуки: для шаблона `its_memo_ncste$pdf` перед `Detail` вызывается `pkg/its_memo_ncste.prepareFODT(id)`; для сущности `its_decision` PDF склеивается с приложениями (`pdftk … cat output`).

**Add to exp_result.** Только при `add_to_exp_result = "1"`: для каждой записи, у которой ещё нет строки в `exp_result` (`entity_pk` = sys$uuid), вставляет `(entity_pk, file_id)`.

**Наложим штамп из ЭЦП exp_result.** Если у записи есть `exp_result.file_id` — `dbrequire("pkg/eds").addStamp(<id нового файла>, <file_id из exp_result>, entity_pk)`; результат становится новым `file_uuid`.

**Ссылки.**

```lua
var.file_url     = "/restapi/getfile?code=" .. var.file_uuid .. "&attachment=true"
var.file_url_woa = "/restapi/getfile2/" .. EntityValueByCode("files","title",var.file_uuid) .. "?code=" .. var.file_uuid
```

**Конечные события.** `PDF` — `<embed [src]="vars.file_url_woa | safe:'resourceUrl'" type="application/pdf">`, заголовок «Предварительный просмотр». `Close` — `location.href = file_url; appComponent.opened_modal = false`. `Ошибка` — поле `last_error`. `Без предпросмотра` — пустая форма.

Копия процесса `exp_template_without_watch` («Экспорт без просмотра») содержит те же шаги без PDF-просмотра.

---

## 8. Как запускать печать

### Из списка (JIT page_tpls `list`, `list_v2`, `list_basic`, `list_tree`)

```javascript
print() {
    const options = Object.assign({}, this.query_options);

    if (this.checked_rows.length > 0) {
        this.appComponent.bpRun('exp_template', {
            url: `?code=${options.code}&ids=${this.checked_rows.join()}`,
            entity_id: this.entity_id
        });
        return;
    }

    const url = '?' + this.query_options.url.replace(/&((perpage|page|loader)=[^&]*)/gi, '');
    this.appComponent.bpRun('exp_template', { url, entity_id: this.entity_id });
}
```

### Из карточки (page_tpls `detail`, `detail_designer`)

```html
<p-button type="button"
    *ngIf="!form_changed && detail_id != 0 && details && details['exp_templates$'] && details['exp_templates$'].length > 0"
    (click)="appComponent.bpRun('exp_template', { url: '?code=' + entity_code + '&ids=' + detail_id, entity_id })"
    icon="pi pi-print"
    [label]="'Печатная версия документа' | translate"
    styleClass="ms-2">
</p-button>
```

Конкретную форму без диалога: добавьте `template_code: 'its_prikaz$pdf'` (или `template_id`). Скачать без предпросмотра PDF: `nopreview: 1`.

### Из Lua (генерация файла в процессе)

```lua
local found, errText, errNum = SqlQueryRow2(
    [[select id from exp_templates where entity_id = (select id from entities where code = ?) and autoprint = 1 limit 1]],
    entity_code)
if errNum ~= 0 then
    var.last_error = "Не найден шаблон: " .. errText
    return
end

local vars, instanceId, taskId, errText, errNum = BPMSStartProcess2("exp_template", sys.user_id, {
    entity_id   = EntityValueByCode("entities", "id", entity_code),
    template_id = found.id,
    url         = "?code=" .. entity_code .. "&ids=" .. var.id
}, sys.instance_id)

if errNum ~= 0 or (vars.last_error or "") ~= "" then
    var.last_error = "Ошибка печати: " .. tostring(errText) .. " " .. tostring(vars.last_error)
    return
end

local fileId = EntityValueByCode("files", "id", vars.file_uuid)
```

Так генерируются вложения писем, файлы для ЭЦП и ЕСЭДО (`its_sd_application`, `its_send_attach_to_cli_copy`, `its_d_req_item_send_to_prv`). Для такого вызова шаблон должен иметь источник `detail` — только он возвращает `file_uuid`. Скрытые из UI шаблоны (`sql_condition = 1 = 0`) вызываются этим способом.

---

## 9. FODT «bottom line»: импорт docx и разметка полей

Позволяет сделать печатную форму из готового Word-документа без ручного написания `{{ }}`.

### Разметка в docx

| Что пишут в Word | Чем становится |
|---|---|
| Подчёркивания `___` (от 3 до 80 подряд) | `_FIELD_n_` → поле `f<n>` |
| `_ROW_BEGIN_k_` / `_ROW_END_k_` в отдельных строках таблицы | `{{range $key_<код>, $value_<код> := $item_value.<код>}}` … `{{end}}`; строки-маркеры удаляются целиком |
| `_COL_k_c_` в ячейке строки между маркерами | значение колонки `c<k>_<c>` |
| `_BLOCK_BEGIN_k_` / `_BLOCK_END_k_` в отдельных абзацах | `{{ if <условие> }}` … `{{end}}`; абзацы-маркеры удаляются |

### Процессы

1. **Импорт шаблона (docx)** `exp_templates_imp_fodt_bl` — docx → `soffice --convert-to fodt` → `bl_fodt_orig`; подчёркивания нумеруются в `_FIELD_n_` (сначала самые длинные) → `bl_fodt_with_ids`; HTML-превью с кликабельными плашками → `bl_html`; `bl_max_field`.
2. **Разметка** — в карточке `exp_templates` (блок `pages_ui_blocks_n` id 17535): клик по плашке открывает выбор атрибута или выражения.
3. **Применить изменения** `exp_templates_fodt_bl_apply` — собирает `template` из `bl_fodt_with_ids` и карт.
4. **Экспорт шаблона в Docx** `exp_templates_fodt_bl_exp_docx` — скачать размеченный FODT как docx для правки в Word.

### Карты

```json
"bl_map": {
  "f1": { "isExpr": false, "value": { "code": "docnum", "title": "Номер документа" } },
  "f5": { "isExpr": true,  "expr": "{{ lua \"task_list_approve\" ( index $value \"sys$uuid\" ) }}" },
  "f11": { "isExpr": false, "value": { "code": "created_at" }, "template_funcs": { "code": "format_date" } }
},
"bl_map_r": { "r1": { "value": { "id": "27369", "code": "its_d_statements_hc", "title": "Налоговые вычеты" } } },
"bl_map_c": { "c1_2": { "info": { "row": 1, "col": 2 }, "isExpr": false, "value": { "code": "hiring_calc_id$" } } },
"bl_map_b": { "b1": { "value": { "condition": "eq $value.type_id \"2\"" } } }
```

Подстановка при «Применить изменения»:

| Маркер | Не выражение | Выражение (`isExpr`) |
|---|---|---|
| `_FIELD_n_` | `{{index $value "<code>"}}` | `expr` как есть |
| `_COL_k_c_` | `{{index $value_<код r_k> "<code>"}}` | `expr` как есть |
| `_BLOCK_BEGIN_k_` | `{{if eq 1 1}}` если условие пустое | `{{ if <condition> }}` |

Внутри строк ТЧ основная запись по-прежнему доступна как `$value`, строка ТЧ — как `$value_<код>`. Ошибки «Строка k оформлена неверно» / «Блок k оформлен неверно» — нет парного `_END_`.

После «Применить изменения» `template` **перезаписывается**: ручные правки шаблона теряются. Повторяемые ручные вставки записывайте в `dev_notes` (так сделано в `its_memo_ncste$pdf`).

---

## 10. Процессы обслуживания шаблонов

| Код | Тип | Что делает |
|---|---|---|
| `exp_template_wizard` | entityTool | Выбор сущности и дизайна `exp_design` → `Upsert` шаблона `<entity>$from_wizard$N` (src detail) |
| `exp_templates_imp_fodt_bl` | entityActionByID | Импорт docx в bottom line |
| `exp_templates_fodt_bl_apply` | entityActionByID | Сборка `template` из разметки |
| `exp_templates_fodt_bl_exp_docx` | entityActionByID | Выгрузка `bl_fodt_with_ids` в docx |
| `export_template_online` | entityActionByID | HTML-шаблон → docx (`convertHTML2Docx`, каталог `dav`), `docx_file_id`, ссылка `ms-word:ofe|u|<cloud_url>restapi/dav/dav/<code>/<file>` для правки в Word по WebDAV |
| `exp_templates_online` | entityActionByID | «Прекратить онлайн-редактирование»: `docx_file_id = null` |
| `exp_template_copy` | entityActionByID | Копия записи с новым `code`/`title` (по умолчанию `_v1`) через `DMLI("insert", …)` |
| `exp_template_export_to_file` | entityAction | JSON-выгрузка выбранных шаблонов через деталь `exp_templates_export` для переноса |
| `exp_template_without_watch` | businessprocess | Вариант `exp_template` без PDF-просмотра |

---

## 11. Ловушки

1. **`no_autoprint` не существует.** В БП нет такой переменной — параметр игнорируется. Автовыбор управляется только `exp_templates.autoprint`: если у сущности ровно один шаблон с `autoprint=1`, список и карточка печатают его без диалога. Отключить — снять флаг или передать `template_code`.
2. **Заданный шаблон не проверяется на доступ.** При `template_id`/`template_code`/autoprint шаг выбора пропускается, `only_users` и `sql_condition` не применяются. Это штатный способ печатать скрытые формы, но не способ ограничивать права.
3. **`sql_condition` проверяется по всем записям `url`.** Если при печати списка хоть одна запись не проходит условие, шаблон пропадает для всего списка. Алиас таблицы — `main`, пользователь — `:user_id`.
4. **`file_uuid` есть только у detail.** Ветка Query отдаёт лишь `file_url`. Для `BPMSStartProcess2` используйте detail-шаблоны.
5. **DOCX только из detail.** В ветке Query тип `docx` уходит в «сырой текст» — получится HTML с расширением `.docx`.
6. **Экранирование в detail.** Все строки проходят `HTMLEscapeString` (кроме `fodt*`). HTML-поля выводите через `{{lua "StrReplace" (index $value "body") "" "" -1}}` или храните в `fodt_*`. В FODT неэкранированный `&`/`<` ломает XML.
7. **`<no value>`.** Отсутствующий ключ печатается как `<no value>`. PDF и DOCX его вычищают, XLS и текстовые типы — нет: используйте `{{if isnil …}}`.
8. **Кеш по `detail_md5`.** Те же данные + тот же шаблон → возвращается старый файл. После правки Lua-функции (`template_funcs`) или параметров результат не обновится, пока не изменятся данные или текст шаблона.
9. **`Commit()` в шаге Detail.** Всё, что вызывающий процесс записал до запуска `exp_template`, фиксируется.
10. **Ключи с `$`.** `$item_value.task_approve_list$` не парсится — только `index $item_value "task_approve_list$"`. Аналогично поля `created_by$`.
11. **Форматы дат.** `TimeParseFormat` требует точного входного формата: timestamp из detail — `"2006-01-02 15:04:05"`, date — `"2006-01-02"`, иногда `"2006-01-02T15:04:05Z"`. Несовпадение даёт пустую строку.
12. **Кавычки Word.** `“ ”` в шаблоне заменяются автоматически только в ветке Detail; в Query-шаблонах вставка из Word ломает `{{ }}`.
13. **`exp_templates$` в авто-детали** содержит жёсткий `entity_id` и не учитывает `sql_condition`: кнопка видна, если у сущности есть хоть один шаблон.
14. **Лишний `{{end}}`** — частая ошибка ручной правки FODT после «Применить изменения»: проверяйте парность `range`/`if`/`end`.
15. **Временные файлы.** Свой генератор по образцу `generate*byHTML` обязан удалять файлы в `TempDir()` во всех ветках ошибок.

---

## 12. Чек-лист новой печатной формы

1. Определить источник: реестр/выгрузка списка → `query`; документ по записи, нужен файл в Lua, DOCX → `detail`.
2. Для `detail` — убедиться, что есть деталь с кодом сущности и нужные `detail_queries` (ТЧ, `task_approve_list$`).
3. Создать запись `exp_templates`: `code` (`<entity>$pdf`, `<entity>$docx`, `<entity>$xls$page`), `title`, `entity_id`, `src_id`, `detail_id`, `ftype_id`, `filename`.
4. Шаблон: из Word — через «Импорт шаблона (docx)» + разметка + «Применить изменения»; вручную — HTML с `@page` и каркасом из [раздела 4](#4-источник-данных-query-и-detail).
5. Повторяющуюся логику (даты прописью, листы согласования, QR) брать из `template_funcs`, новую — добавлять туда.
6. Настроить доступ: `autoprint` (единственная форма по умолчанию), `only_users` + `exp_templates_user`, `is_sql_condition` + `sql_condition`.
7. Проверить из карточки и из списка (одна и несколько записей), под пользователем без роли admin.
8. Если форма нужна в процессе — вызвать `BPMSStartProcess2("exp_template", …)` и проверить `vars.file_uuid` и `vars.last_error`.
