# DamuBPM — автогенерация страниц

[← Главная](index.html)

Практическая инструкция по текущему `pkg/page`, Detail `pages_for_gen_ng` и шаблонам `list` / `detail`. Исходники прочитаны через MCP test-helpdeskv19 29.09.2026. Руководство описывает фактическую реализацию; замечания к ней отделены от рекомендуемого процесса. Изменения на сервер не вносились, генерация не запускалась.

## 1. Главное за минуту

Автогенерация собирает HTML и JavaScript страницы из метаданных сущности, структуры UI и шаблонов. Результат записывается в `pages.angular_template` и `pages.angular_json`. Несмотря на имя, `angular_json` в изученных шаблонах содержит JavaScript, возвращающий класс Angular/JIT.

1. Подготовьте сущность, атрибуты, связи и тип сущности.
2. Настройте правила страниц для этого типа и шаблоны `list` / `detail`.
3. Вызовите `addDefaultPages(entity_id, user_id)`, если страниц ещё нет.
4. Настройте UI-блоки и их элементы, параметры, табличные части и custom blocks.
5. Вызовите `genPages(entity_id, user_id)`.
6. Проверьте сохранённые HTML/JS и работу страницы под целевой ролью.

**Ключевой принцип:** у автоматической страницы правьте источник — шаблон или метаданные. Ручные изменения сгенерированного HTML/JS будут перезаписаны при следующей генерации.

## 2. Три функции pkg/page

| Функция | Назначение | Результат и ограничения |
|---|---|---|
| `addDefaultPages(entity_id, user_id)` | Создаёт недостающие страницы и стартовые блоки по типу сущности | Возвращает `errText, errNum`; не вызывает `genPages` |
| `genPages(entity_id, user_id)` | Пересобирает все страницы сущности с `is_auto=1` и существующим шаблоном | Возвращает `errText, errNum` на основном пути; некоторые ошибки обрабатываются неполно |
| `genCard(page_id, data)` | В исходнике есть старая логика карточек | Фактически сразу возвращает `"", "", 0`; остальной код недостижим |

`genPages` принимает **ID сущности**, а не ID страницы. `Detail(vp.id, "pages_for_gen_ng", user_id)` внутри него принимает **ID страницы**.

### Последовательность сборки

| Этап | Что делает код |
|---|---|
| Выбор страниц | Соединяет `pages` с `page_tpls`, фильтрует по `entity_id` и `p.is_auto=1` |
| Контекст | Вызывает Detail `pages_for_gen_ng` для каждой страницы; наборы преобразует через `array(...)` |
| Компоненты | Дополняет `ui_attrs`, `table_part_cols`, `table_part_cols_a` картами `component_param` |
| Табличные части | Добавляет `table_parts[].param` |
| Параметры страницы | Собирает `val.param[code] = value` |
| Расширения | Собирает `val.custom_block[place_code]` из блоков страницы и типа сущности |
| HTML | `ParseTemplate(vp.angular_template, val, user_id)` |
| JavaScript | `ParseTemplate(vp.angular_json, val, 1)` — в текущем коде пользователь жёстко задан как `1` |
| Экранирование | Заменяет все `{[{` на `{{` в обоих результатах |
| Запись | Обновляет HTML и JS страницы; для Oracle использует `SqlCall` с CLOB, иначе `SqlExec2` |

Внутри функции нет явной общей транзакции: при ошибке поздней страницы предыдущие уже могут быть записаны. Поведение внешней транзакции вызывающего кода здесь не исследовалось.

## 3. Создание страниц по типу сущности

Правила находятся в `entity_types_pagegen`, начальные UI-блоки — в `entity_types_pagegen_b`.

```text
page.code = entity.code + coalesce(suffix, '')
page.url  = module.url_prefix + entity.code + coalesce(suffix, '') + coalesce(url_add, '')
```

Разделители автоматически не добавляются. Проверяйте завершающий `/` у префикса модуля.

| Подтверждённый тип сущности | Суффикс | Шаблон | Дополнение URL | Стартовый блок |
|---|---|---|---|---|
| `table`, `its_req`, `reference` | `details` | `detail` | `/{id}` | `default_page`, «Основная информация» |
| `table`, `its_req`, `reference` | пустой | `list` | пусто | В просмотренной настройке отсутствует |
| `reference` | `view` | `detail` | `/{id}` | `default_page_view` |

Пример расчёта, не новая настройка сервера: сущность `demo_request`, префикс `/demo/`, суффикс `details` дают код `demo_requestdetails` и URL `/demo/demo_requestdetails/{id}`.

При вставке устанавливаются `db_template=1`, `is_auto=1`, сущность, модуль, тип страницы, шаблон и URL. Проверка существования учитывает код страницы **и** сущность. Существующие страницы этим методом не обновляются; недостающие блоки у уже существующей страницы не восстанавливаются.

Если код заканчивается на `details`, `entities.detail_page_id` заполняется только при NULL. Назначения `list_page_id` в этой функции нет. Создание элементов `pages_ui_blocks_n` также отсутствует: одного вызова `addDefaultPages` недостаточно для заполненной формы.

## 4. Где менять нужную часть интерфейса

| Задача | Источник изменения | Следующий шаг |
|---|---|---|
| Общая структура всех списков | `page_tpls` с кодом `list` | Перегенерировать затронутые сущности |
| Общая структура карточек | `page_tpls` с кодом `detail` | Перегенерировать затронутые сущности |
| Поле, его тип и ссылка | Атрибут сущности | Проверить включение в UI-блок, затем генерация |
| Порядок и группировка полей | Блоки страницы и их элементы | Генерация |
| Вид одного компонента | Компонент элемента / атрибута / типа данных | Проверить приоритет, затем генерация |
| Поведение одной страницы | `page_params` или `page_cus_blocks` | Убедиться, что шаблон читает этот код |
| Расширение страниц типа сущности | `entity_type_pc_block` | Проверить все страницы этого типа |
| Строки, колонки и фильтры списка во время работы | Query, filter set и их метаданные | Проверить реальный запрос в браузере |

Изменение общего шаблона само по себе не пересобирает ранее созданные страницы. Не используйте редактирование готового результата как постоянный способ доработки автоматической страницы.

## 5. Контекст Detail pages_for_gen_ng

Это код **Detail**, объединяющего 24 `detail_queries`, а не обычного Query. Поэтому поиск через `get_query("pages_for_gen_ng")` не находит его. Ключи подзапросов становятся наборами в `val`, например `.entities`, `.ui_attrs`, `.table_parts`.

| Набор | Что передаёт шаблонам |
|---|---|
| `entities` | Сущность страницы, маршруты, `filter_set_code`, счётчики настроек |
| `ui_blocks` | Корневые блоки, не включённые в другой блок элементом типа `block` |
| `ui_blocks2` | Вложенные элементы типов `block` и `tp`, ширина и связь с родителем |
| `ui_attrs` | Элементы типов `attr`, `html`, `ui_component`, `logic`; шаблоны компонентов и UI-выражения |
| `all_ui_attrs` | Атрибуты сущности страницы независимо от размещения в блоках |
| `entity_attrs` | Атрибуты с описаниями связанных настроек |
| `attrs` | Атрибуты с шириной и блоком; SQL непосредственно ожидает ID сущности |
| `logic` | Текст `n.logic` элементов типа `logic` |
| `table_parts` | Табличные части с привязками к UI-блокам и шаблонами типов |
| `table_parts_uq` | Табличные части без соединения с размещениями UI |
| `table_part_attrs` | Атрибуты связанных сущностей табличных частей |
| `table_part_cols` | Настроенные колонки и шаблоны их компонентов |
| `table_part_cols_a` | Дополнительные элементы колонок, HTML, типы, autosave |
| `table_parts_tp` | Связи вложенных табличных частей |
| `table_parts_cross` | Перекрёстные табличные части |
| `table_parts_cross_cols` | Колонки перекрёстных табличных частей |
| `page_links` | Связанные страницы, коды, названия, URL |
| `exp_templates` | Шаблоны экспорта сущности страницы |
| `exp_templates$` | SQL с фиксированным `entity_id=24`; требует отдельной проверки назначения |
| `entities$grants` | Полномочия; SQL ожидает ID сущности |
| `entity_attrs_ui_event` | События атрибутов; SQL ожидает ID сущности |
| `entity_bp_processes` | Бизнес-процессы; SQL ожидает ID сущности |
| `entity_ui_blocks` | UI-блоки сущности; SQL ожидает ID сущности |
| `filter_sets` | Наборы фильтров; SQL ожидает ID сущности |

SQL всех наборов приведён в приложении. Наличие набора в Detail не означает, что каждый шаблон его использует.

### Проверка неоднозначных ID

Большинство подзапросов начинают от страницы: `where p.id=?`, затем получают её сущность. Однако шесть наборов из таблицы выше непосредственно фильтруют по ID сущности. Вызов генератора передаёт ID страницы. Это потенциальное несоответствие; без изучения внутренней реализации `Detail` нельзя утверждать, что подстановка в каждом запросе одинакова. Проверьте результат на странице, у которой `page.id != entity.id`, прежде чем использовать эти наборы в новом шаблоне.

## 6. UI-блоки, поля и приоритеты

У блоков есть собственный `is_auto`, независимый от флага страницы:

- `pages_ui_blocks.is_auto=1`: SQL берёт HTML из `ui_block_tpl.angular_template`.
- Иначе берётся `pages_ui_blocks.angular_template` самого блока.
- Страница с `pages.is_auto=0` целиком исключается из `genPages`, независимо от настроек её блоков.

Шаблон `detail` передаёт блокам наборы через `setMapValue`, затем вызывает `parseTemplate`. Проверенный `default_page` выбирает поля по равенству `pages_ui_blocks_id` и ID блока, табличные части — по `_tp_ui_block_id`, вложенные блоки — по родительской связи. Не создавайте циклы вложенности.

Для `ui_attrs.angular_template` порядок `coalesce`: компонент элемента → компонент атрибута → повторно компонент элемента → компонент типа данных → `n.angular_html` для `html`/`logic` → пустая строка. Повторная ссылка на компонент элемента присутствует в реальном SQL. **Пустая строка не является NULL** и может остановить fallback.

Ширина: `n.ui_width_id` → ширина атрибута → `col-md-4 mb-3`. Порядок: `coalesce(n.nn,e.nn_field,e.id)`.

`ng_model` строится как `detail.<код атрибута>`. `readonly` получает `true` при `restrict_edit=1` или `is_formula=1`; затем проверяется `ui_ng_disabled`; иначе используется `!(editable_row || editable.<код>)`.

У `all_ui_attrs` и `ui_attrs` разные правила readonly: в первом есть ветка `readonly_detail_sql`, во втором она закомментирована. UI readonly не заменяет серверные полномочия.

Условие `hide_field` расположено в `LEFT JOIN` к атрибуту. Оно может оставить строку UI-элемента с пустыми полями атрибута. Поэтому «скрыть атрибут» не обязательно означает «удалить строку из ui_attrs».

## 7. Параметры: четыре разных уровня

| Карта в шаблоне | Откуда берётся | Какой ключ |
|---|---|---|
| `.param` страницы | `page_params` + `page_param_cls` | `page_param_cls.code` |
| `.component_param` поля | `pages_ui_blocks_n_cpv` + `ui_comp_params` | `ui_comp_params.attr` |
| `.param` табличной части | `pages_ui_blocks_n_tpv` + `table_part_params` | `table_part_params.code` |
| `.component_param` обычной колонки ТЧ | `entity_attr_uicmps` + `ui_comp_params` | `ui_comp_params.attr` |
| `.component_param` дополнительного элемента колонки | `table_part_cols_a_pv` + `ui_comp_params` | `ui_comp_params.attr` |

Вложенный `parseTemplate` меняет текущий контекст `.`: `.param` внутри шаблона ТЧ — уже параметры ТЧ, а не страницы. Значения компонентных параметров пропускаются, если nil или пустая строка; параметры страницы копируются без такого фильтра.

Подтверждённые параметры `list`: `disable_add`, `add_btn_insert_redirect`, `delete_disabled`, `open_page_down`, `take_screenshot`, `enable_clipboard_import`. Часть ссылок в исходниках закомментирована — наличие имени не гарантирует активное поведение.

Подтверждённые параметры `detail`: `detail_code`, `title_reg`, `title_noreg`, `task_bind_disable`. Например, `detail_code` переопределяет код Detail загрузки данных; иначе используется код сущности. Флаги шаблоны часто сравнивают со строкой `"1"`.

## 8. Табличные части

1. Настройте связанную сущность и атрибут обратной связи.
2. Выберите тип ТЧ с `angular_template`.
3. Добавьте колонки и при необходимости дополнительные элементы колонок.
4. Разместите ТЧ в блоке страницы элементом типа `tp`.
5. Проверьте `_tp_ui_block_id`, состав колонок, параметры и данные runtime Detail.

Обычная колонка выбирает компонент в порядке: `table_part_cols.ui_comp_id` → компонент атрибута → компонент типа данных. Дополнительный элемент использует `table_part_cols_a.ui_comp_id` вместо компонента обычной колонки.

Есть две особенности текущих SQL: `table_parts` соединяется с размещениями без фильтра `ui_block_id.pages_id=p2.id`; параметры ТЧ читаются по ID блока, а не по конкретному узлу `tp`. Несколько размещений или несколько ТЧ в одном блоке могут дать повторные строки либо смешанные параметры. При неожиданном дублировании проверяйте эти связи до изменения HTML.

## 9. Custom blocks — расширение без копирования шаблона

Генератор объединяет `page_cus_blocks` страницы и `entity_type_pc_block` типа сущности через `UNION ALL`. Ключ — `page_cus_block_places.code`. Внутри каждого ключа текст добавляется перед накопленным содержимым с переводом строки.

В SQL нет `ORDER BY`, а Lua использует `pairs`: гарантированного порядка фрагментов и приоритета «страница поверх типа» нет. Зависимые действия лучше держать одним фрагментом. Одинаковый place не заменяет предыдущий блок.

| Шаблон | Подтверждённые места вставки |
|---|---|
| `list`, HTML | `list_after_add_button`, `list_column_stat_id` |
| `list`, JS | `ng_on_init` |
| `detail`, HTML | `page_breadcrumb`, `ng_main_detail_buttons`, `begin_main_block`, `end_main_block` |
| `detail`, JS | `ng_var_init`, `after_edit_attr`, `ui_angular_default_value`, `ng_bind`, `ng_before_init`, `ng_on_init`, `angular_controller_save`, `angular_controller_after_save` |

Прежде чем создавать блок, найдите в выбранном шаблоне `parseTemplate .custom_block.<код> .` и изучите окружающий код. `ng_var_init` находится в теле класса и подходит для полей/методов; `ng_on_init` — внутри метода и подходит для выполняемых инструкций. Произвольный новый код места без вставки в шаблон не отобразится.

Пример HTML-фрагмента для существующего `begin_main_block`:

```html
<div class="card">
  <div class="card-body">Заполните обязательные поля перед сохранением.</div>
</div>
```

## 10. Два языка шаблонов: ParseTemplate и Angular

Сначала на сервере выполняется `ParseTemplate`, затем результат исполняется браузером. JS-комментарии `//` и HTML-комментарии не защищают `{{...}}` от серверного парсера.

| Что нужно | Что писать в исходном шаблоне |
|---|---|
| Подставить код сущности при генерации | `{{ $entity.code }}` |
| Вывести runtime-значение Angular | `{[{ detail.title }}` |
| Объявить переменную серверного шаблона | `{{$param := .param}}` |
| Взять сущность | `{{$entity := (index .entities 0)}}` |
| Вызвать вложенный шаблон | `{{parseTemplate .custom_block.begin_main_block .}}` |

Пример сочетания:

```html
{{$entity := (index .entities 0)}}
<section data-entity="{{$entity.code}}">
  <span>{[{ detail.title }}</span>
</section>
```

После генерации серверная переменная заменена кодом сущности, а `{[{ detail.title }}` становится `{{ detail.title }}`.

`angular_template` и `angular_json` разбираются отдельно. Объявление `$param` в HTML не делает его доступным в JS. При `undefined variable "$param"` сначала определите, серверная это переменная или Angular-выражение: серверную объявите в том же шаблоне; Angular-интерполяцию экранируйте. Механическая замена всех `{{` на `{[{` сломает генерацию.

Вложенные `parseTemplate` получают собственный объект данных и не наследуют лексические переменные родительского шаблона. Передавайте нужные значения явно, как делают штатные шаблоны через `setMapValue`.

## 11. Запуск и диагностика из Lua

Ниже пример последовательности для серверного Lua-контекста с доступными `sys.user_id` и `dbrequire`. Замените код сущности на существующий. Это изменяющий страницы вызов; в рамках подготовки инструкции он не выполнялся.

```lua
local page = dbrequire("pkg/page")
local entity_id = EntityValueByCode("entities", "id", "demo_request")
if entity_id == nil or entity_id == "" then
    error("Сущность demo_request не найдена")
end

local errText, errNum = page.addDefaultPages(entity_id, sys.user_id)
if errNum ~= 0 then
    error("addDefaultPages: " .. tostring(errText))
end

-- Здесь UI-блоки и их элементы уже должны быть настроены.
errText, errNum = page.genPages(entity_id, sys.user_id)
if errNum ~= 0 then
    error("genPages: " .. tostring(errText))
end
```

Если страницы существуют, достаточно `genPages`; вызывать создание повторно для каждого изменения не требуется. Проверка `errNum ~= 0` также остановит пример при nil, но не исправляет внутренние пропуски обработки ошибок.

Получить контекст без записи страниц:

```lua
local page_id = EntityValueByCode("pages", "id", "demo_requestdetails")
if page_id == nil or page_id == "" then
    error("Страница не найдена")
end
local ctx, errText, errNum = Detail(page_id, "pages_for_gen_ng", sys.user_id)
if errNum ~= 0 then
    error(tostring(errText))
end
print(JsonToString(ctx))
```

Это сырой Detail-контекст: `param`, `custom_block` и дополнительные `component_param` формирует отдельно `genPages`. Для точного предпросмотра результата надо воспроизвести всё это обогащение без финального UPDATE.

## 12. Queries во время работы готовой страницы

Не смешивайте источники генерации с бизнес-данными страницы. `pages_for_gen_ng` нужен для сборки кода; список и карточка затем делают собственные запросы.

В `list` основной код Query выбирается так: `DATA.query_code` → `this.query_code` из параметров маршрута → код сущности. Данные карточки `detail` загружаются через `getDetail`, код берётся из параметра страницы `detail_code` или кода сущности.

| Query | Назначение / параметр |
|---|---|
| `bp_processes_tools_by_entity_id` | Инструменты БП по ID сущности; проверка роли и `can_run` |
| `recent_rows$angular` | Последние записи пользователя по коду таблицы, за последние сутки, максимум 5; PostgreSQL URL `/<page.code>/<pk>` |
| `get_filter_cols_by_code` | Метаданные колонок фильтра по коду filter set |
| `entity_acts_many` | Активные массовые/глобальные действия с учётом ролей |
| `company_info_by_user` | Компания по ID и связи с текущим пользователем |

Query основной сущности, `<entity>$archive` и select-query связанных сущностей определяются конфигурацией конкретной страницы. Без выбора сущности нельзя перечислить их исчерпывающе. Общие перечисленные queries прочитаны; их SQL включён в приложение.

Для JIT используйте штатный `Models.QueryOptions`, фильтры и пагинацию. При собственных вызовах `this.dbQueryService.restapiGet` / `restapiPost` третий параметр `true` отключает общий загрузчик согласно стандарту разработки проекта.

## 13. Быстрая диагностика

| Симптом | Что проверить |
|---|---|
| Генерация не меняет страницу | ID сущности, `pages.is_auto=1`, `tpl_id` и существование шаблона |
| Страница есть, но форма пустая | Элементы `pages_ui_blocks_n`; наличие атрибутов само по себе недостаточно |
| Поле не появилось | Тип узла, привязку к блоку, `hide_field`, шаблон компонента, `ui_ng_if` |
| Компонент не использует fallback | Пустая строка у более приоритетного шаблона вместо NULL |
| Блок не изменился | Флаг автоматичности блока; используется шаблон или собственный HTML |
| ТЧ повторяется | Несколько размещений и отсутствие ограничения страницы в join `table_parts` |
| Параметр не действует | Правильный уровень `param` / `component_param`, код, строковое значение `"1"`, использование в шаблоне |
| Custom block не появился | Совпадение place code и наличие активного `parseTemplate` в шаблоне |
| `error angular_template` / `error angular_json` | Серверный синтаксис, области видимости, `{[{` для Angular |
| Успех возвращён, код не обновился | Проверить сохранённые поля: ошибки финального UPDATE в pkg/page не проверяются |
| Список пустой при корректном HTML | Фактический runtime Query, права, фильтры, данные ответа |
| Нет недавних записей | Текущий пользователь, код таблицы, время просмотра/изменения, `detail_page_id` |

## 14. Обнаруженные ограничения и улучшения

Следующие пункты — результат чтения исходников, а не внесённые исправления.

1. После `Detail` отсутствует проверка ошибки; используется переменная `errTex` вместо `errText`.
2. Ошибки некоторых bind-запросов записываются в `var.last_error`, затем выполняется голый `return`. Единый контракт `errText, errNum` нарушается.
3. Ошибка финального `SqlExec2` / `SqlCall` не проверяется; функция может завершиться `"",0` после неудачной записи.
4. `val.title = vp.title`, но исходный SELECT не выбирает `p.title`. На корневой `.title` нельзя полагаться без исправления; шаблоны могут использовать `.entities`.
5. В вставке стартового блока `title = header_title` ссылается на не объявленную здесь переменную; `header_title = v1.header_title` заполнен отдельно.
6. После запроса стартовых блоков в `addDefaultPages` нет проверки `errNum` до перебора.
7. `angular_json` рендерится с пользователем `1`, HTML — с переданным пользователем. Пользовательские параметры и шаблонные функции могут дать разные результаты.
8. В цикле ТЧ выполняется дополнительная выборка `entity_attr_uicmps`, результат которой далее не используется. Её удаление после проверки снизит число запросов.
9. Компонентные параметры читаются отдельным SQL для каждого элемента. Для крупных форм полезна пакетная загрузка по странице/сущности и сборка словарей в памяти.
10. Порядок объединения custom blocks не закреплён; для воспроизводимости нужны явный порядок SQL и последовательный обход.
11. Наборы с фильтром по entity ID, фиксированный `entity_id=24` и связи размещений ТЧ требуют проверки на конкретной странице.

Практический приоритет доработки: сначала надёжное возвращение ошибок и проверка записи, затем корректность ID и связей, после этого оптимизация количества запросов. Для массовой пересборки полезны журнал страницы/шаблона и промежуточный рендер до записи.

## 15. Чек-лист выпуска страницы

- Сохранена предыдущая версия шаблона и результата страницы.
- Правильные сущность, тип, модуль, URL, шаблон и флаги автоматичности.
- UI-блоки и элементы заполнены; нет циклической вложенности.
- Связанные сущности имеют select-query и нужные страницы перехода.
- Параметры и custom blocks расположены на нужном уровне.
- После генерации прочитаны сохранённые HTML и JS, нет оставшихся `{[{`.
- Проверены список, открытие карточки, создание, редактирование и сохранение.
- Проверены фильтры, пагинация, ссылки, ТЧ и действия БП, если они используются.
- Проверка выполнена под целевой ролью, а не только под admin.
- Повторная генерация сохраняет ожидаемое поведение.

## 16. Подключение к документации

Положите HTML-файл рядом с существующим `index.html`. Вверху и внизу инструкции есть относительная ссылка на главную. В каталог главной страницы можно добавить:

```html
<a href="damubpm_page_generation_guide.html">Автогенерация страниц DamuBPM</a>
```

Существующий `index.html` в рамках этой задачи не изменялся. HTML автономен: оформление встроено, внешние шрифты и JavaScript-библиотеки не требуются. Приложения с исходным SQL и Lua позволяют сверять руководство с изученным снимком сервера.

[← Главная](index.html)


## Приложение A. SQL подзапросов pages_for_gen_ng

Точный снимок прочитанных SQL. Диалектные директивы ParseTemplate сохранены. `?` — bind-параметр, а не текст для ручной конкатенации.

### all_ui_attrs

```sql
SELECT
replace(pd.url,'/{id}','') as "detail_page_code",
pd.code as "angular_detail_page_code",

case
when e.restrict_edit=1 then 'true'
when e.is_formula=1 then 'true'
when trim(e.ui_ng_disabled) is not null and LENGTH(e.ui_ng_disabled)>1 then {{if .Oracle}}TO_CHAR{{end}}(e.ui_ng_disabled)
when entity.readonly_detail_sql is not null  then 'detail.readonly$ == 1'
else 'false' end as "readonly",
concat('detail.',e.code) as "ng_model",
coalesce(co.code, co2.code) "_ui_component_code",





(SELECT title FROM entities WHERE id = e.entity_link_id) entity_link_title,
(SELECT code FROM entities WHERE id = e.entity_link_id) entity_link_code,
(SELECT q.code FROM entities eee, queries q
WHERE eee.def_sel_query_id = q.id
AND eee.id = e.entity_link_id) sel_query_code,
p.code lookup_page_code, e.*, dt.title data_type_title,
coalesce({{if .Oracle}}TO_CHAR{{end}}(e.help_html),'') as "_help_html",
dt.code data_type_code, coalesce(nullif(trim(({{if .Oracle}}TO_CHAR{{end}}(e.ui_ng_if))), ''), '1==1') "_ui_ng_if",
coalesce(nullif(trim(w.css_class), ''), 'col-md-4 mb-3') "_ui_width_css"
FROM pages p2
join entity_attrs e  on e.entity_id=p2.entity_id
left JOIN data_types dt ON dt.id = e.data_type_id
LEFT JOIN entities ee ON ee.id = e.entity_link_id
left join entities entity ON entity.id=e.entity_id
LEFT JOIN pages p ON p.id = ee.lookup_page_id
LEFT JOIN pages pd ON pd.id = ee.detail_page_id
LEFT JOIN ui_widths w ON w.id = e.ui_width_id
LEFT JOIN ui_components co ON co.id = e.ui_component_id
LEFT JOIN ui_components co2 ON co2.id = dt.ui_default_component_id
WHERE 
 p2.id = ? ORDER BY coalesce(e.nn_field,e.id) ASC
```

### attrs

```sql
select

w.title as "_ui_width_title",
b.title as "_ui_block_title",
(select code from entities where id=main.entity_id) "_entity_code",
(select title from entities where id=main.entity_link_id) entity_link_title,
main.*,dt.title data_type_title from entity_attrs main
left join data_types dt on dt.id=main.data_type_id
left join entity_ui_blocks b on b.id=main.ui_block_id
left join ui_widths w on w.id=main.ui_width_id
where main.entity_id=? 


order by coalesce(main.nn_field,main.id)
```

### entities

```sql
SELECT
module_id.code as module_code,

{{if .Oracle }}
m.url_prefix || main.code "_list_url",
'/' || pl.code "angular_list_url",
{{else}}
concat(m.url_prefix , main.code) "_list_url",
concat('/' , pl.code) "angular_list_url",

{{end}}



p.code as page_code,


case when p.url is not null then
replace(p.url ,'/{id}','')
else 
concat( main.code ,'details')
end "_detail_url",


p.code  "angular_detail_url",
coalesce( fs.code ,main.code)  "filter_set_code",


(select count(1) from bp_processes p join bp_process_types pt on pt.id=p.type_id where p.action_entity_id = main.id and pt.code='entityActionByID' 

{{if .MySQL}}
limit 1
{{end}}
{{if .Postgres}}
limit 1
{{end}}
{{if .Oracle}}
and rownum  = 1
{{end}}


) as "_has_entity_action_by_id",


 (SELECT COUNT(1) FROM entity_attrs WHERE entity_id = main.id) "_attrs_count",
 (SELECT COUNT(1) FROM entity_cus_attrs WHERE entity_id = main.id) "_cus_attrs_count",
 (SELECT COUNT(1) FROM bp_processes p WHERE p.action_entity_id = main.id) "_actions_count",
 (SELECT COUNT(1) FROM entity_grants WHERE entity_id = main.id) "_grants_count",
 (SELECT COUNT(1) FROM entity_validators p WHERE p.entity_id = main.id) "_validators_count",
 (SELECT COUNT(1) FROM queries p WHERE p.entity_id = main.id) "_queries_count",
 (SELECT COUNT(1) FROM details WHERE entity_id = main.id) "_details_count",
 (SELECT COUNT(1) FROM pages WHERE entity_id = main.id) "_pages_count",
 (SELECT COUNT(1) FROM filter_sets WHERE entity_id = main.id) "_filters_count",
 (SELECT COUNT(1) FROM entity_view_limits WHERE entity_id = main.id) "_view_limits_count",
 (SELECT title FROM entities WHERE id = main.parent_entity_id) "_parent_entity_title",
 (SELECT title FROM entity_attrs WHERE id = main.parent_entity_attr_id) "_parent_entity_attr_title",
 (SELECT title FROM entity_types WHERE id = main.entity_type_id) "_entity_type_title",
 (SELECT code FROM entity_types WHERE id = main.entity_type_id) "_entity_type_code",
 main.*
 FROM 
 pages p2 
 join entities main on main.id=p2.entity_id
 LEFT JOIN modules m ON m.id = main.module_id
 LEFT JOIN pages p on p.id=main.detail_page_id
 LEFT JOIN pages pl on pl.id=main.list_page_id
 left join modules module_id on module_id.id=main.module_id
 left join filter_sets fs on fs.id=p2.filter_set_id
 
 WHERE p2.id = ?
```

### entities$grants

```sql
select main.*,2 as "_two" , 


{{if .Oracle}}
entity_id.code || ' ' || entity_id.title as "entity_id$",
{{else}}
concat(entity_id.code,' ',entity_id.title) as "entity_id$",
{{end}}






{{if .Oracle}}
role_id.code || ' - ' ||role_id.title as "role_id$",
{{else}}
concat(role_id.code,' - ',role_id.title) as "role_id$",
{{end}}



view_limit_id.title  as "view_limit_id$",created_by.title  as "created_by$" from entity_grants main 
left join entities entity_id on entity_id.id=main.entity_id 
left join roles role_id on role_id.id=main.role_id 
left join entity_view_limits view_limit_id on view_limit_id.id=main.view_limit_id 
left join users created_by on created_by.id=main.created_by  
where main.entity_id=?
```

### entity_attrs

```sql
select main.*,2 as "_two" , 
{{if .Oracle}}
entity_id.code || ' ' || entity_id.title
{{else}}
concat(entity_id.code,' ',entity_id.title) 
{{end}}

as "entity_id$",data_type_id.title  as "data_type_id$",list_id.title  as "list_id$",rule_id.title  as "rule_id$",

{{if .Oracle}}
entity_link_id.code || ' ' || entity_link_id.title
{{else}}
concat(entity_link_id.code,' ',entity_link_id.title)
{{end}}

as "entity_link_id$",default_user_var_id.title  as "default_user_var_id$",on_update_user_var_id.title  as "on_update_user_var_id$",ui_component_id.title  as "ui_component_id$",created_by.title  as "created_by$",ui_width_id.title  as "ui_width_id$",ui_block_id.title  as "ui_block_id$" 
from pages p
join entity_attrs main on main.entity_id = p.entity_id
left join entities entity_id on entity_id.id=main.entity_id 
left join data_types data_type_id on data_type_id.id=main.data_type_id 
left join lists list_id on list_id.id=main.list_id 
left join entity_attr_update_rules rule_id on rule_id.id=main.rule_id 
left join entities entity_link_id on entity_link_id.id=main.entity_link_id 
left join user_vars default_user_var_id on default_user_var_id.id=main.default_user_var_id 
left join user_vars on_update_user_var_id on on_update_user_var_id.id=main.on_update_user_var_id 
left join ui_components ui_component_id on ui_component_id.id=main.ui_component_id 
left join users created_by on created_by.id=main.created_by 
left join ui_widths ui_width_id on ui_width_id.id=main.ui_width_id 
left join entity_ui_blocks ui_block_id on ui_block_id.id=main.ui_block_id  
where p.id=?
```

### entity_attrs_ui_event

```sql
select main.*, event_id.code as event_id_code,entity_attrs_id.code as entity_attrs_id_code  from entity_attrs_ui_event main
join ui_attr_event event_id on event_id.id = main.ui_attr_event_id 
join entity_attrs entity_attrs_id on entity_attrs_id.id=main.entity_attrs_id
where entity_attrs_id.entity_id=?
```

### entity_bp_processes

```sql
select main.*,
{{if .Oracle}}
'#/bpms/modeler/' ||main.id
{{else}}
concat('#/bpms/modeler/',main.id) 
{{end}}

as modeler_url,
{{if .Oracle}}
module_id.code || ' - ' || module_id.title
{{else}}
concat(module_id.code,' - ',module_id.title) 
{{end}}

as "module_id$",type_id.title  as "type_id$",
{{if .Oracle}}
action_entity_id.code || ' ' || action_entity_id.title
{{else}}
concat(action_entity_id.code,' ',action_entity_id.title)
{{end}}


as "action_entity_id$",entity_event_type_id.title  as "entity_event_type_id$",created_by.title  as "created_by$",
{{if .Oracle}}
wf_status_attr_id.code || ' - ' || wf_status_attr_id.title
{{else}}
concat(wf_status_attr_id.code, ' - ',wf_status_attr_id.title)
{{end}}
as "wf_status_attr_id$",wf_id_var_id.title  as "wf_id_var_id$" from bp_processes main 
left join modules module_id on module_id.id=main.module_id 
left join bp_process_types type_id on type_id.id=main.type_id 
left join entities action_entity_id on action_entity_id.id=main.action_entity_id 
left join entity_event_types entity_event_type_id on entity_event_type_id.id=main.entity_event_type_id 
left join users created_by on created_by.id=main.created_by 
left join entity_attrs wf_status_attr_id on wf_status_attr_id.id=main.wf_status_attr_id 
left join bp_process_vars wf_id_var_id on wf_id_var_id.id=main.wf_id_var_id  
where main.action_entity_id=?
```

### entity_ui_blocks

```sql
select main.*,2 as "_two" , created_by.title  as "created_by$",
{{if .Oracle}}
entity_id.code || ' ' || entity_id.title
{{else}}
concat(entity_id.code,' ',entity_id.title) 
{{end}}
as "entity_id$",parent_ui_block_id.title  as "parent_ui_block_id$" from entity_ui_blocks main 
left join users created_by on created_by.id=main.created_by 
left join entities entity_id on entity_id.id=main.entity_id 
left join entity_ui_blocks parent_ui_block_id on parent_ui_block_id.id=main.parent_ui_block_id  
where main.entity_id=?
```

### exp_templates

```sql
SELECT et.id, et.title FROM pages p
join 
exp_templates et on et.entity_id=p.entity_id WHERE p.id=?
```

### exp_templates$

```sql
select * from exp_templates where entity_id = 24
```

### filter_sets

```sql
select main.*,2 as "_two" , 

{{if .Oracle}}
entity_id.code || ' ' || entity_id.title as "entity_id$",
{{else}}
concat(entity_id.code,' ',entity_id.title) as "entity_id$",
{{end}}

created_by.title  as "created_by$",


{{if .Oracle}}
default_order_attr_id.code || ' - ' || default_order_attr_id.title
{{else}}
concat(default_order_attr_id.code, ' - ',default_order_attr_id.title) 
{{end}}




as "default_order_attr_id$" from filter_sets main 
left join entities entity_id on entity_id.id=main.entity_id 
left join users created_by on created_by.id=main.created_by 
left join entity_attrs default_order_attr_id on default_order_attr_id.id=main.default_order_attr_id  
where main.entity_id=?
```

### logic

```sql
select n.logic from pages_ui_blocks b
join pages_ui_blocks_n n on n.pages_ui_blocks_id=b.id
join pages_ui_blocks_n_t t on t.id=n.t_id
where b.pages_id=? and t.code='logic'
```

### page_links

```sql
select p.code as "_page_code", main.*,
pl.code as "_linked_page_code",
pl.title as "_linked_page_title",
pl.url as "_linked_page_url"
from page_links main
join pages p on p.id=main.page_id
join pages pl on pl.id=main.linked_page_id
where p.id=?
```

### table_part_attrs

```sql
select 
replace(pd.url,'/{id}','') as "detail_page_code",
eal.*,dt.code as "_data_type_code",
q.code as "_query_code",
tp.id as table_part_id

from 
pages p2 
join entities e2 on e2.id=p2.entity_id
join table_parts tp on tp.entity_id = e2.id
join entity_attrs eal on eal.entity_id=tp.entity_link_id
left join entities e on e.id=eal.entity_link_id
left join queries q on q.id=e.def_sel_query_id
join data_types dt on dt.id=eal.data_type_id
LEFT JOIN pages pd ON pd.id = e.detail_page_id
where p2.id=?


order by coalesce(eal.nn_field,eal.id)
```

### table_part_cols

```sql
select 
tpc.ui_ng_title as _ui_ng_title,
tpc.ui_ng_readonly as "_ui_ng_readonly_tp",
tpc.ui_ng_readonly as "ng_readonly",
coalesce(nullif(trim(({{if .Oracle}}TO_CHAR{{end}}(tpc.ui_ng_if))), ''), '1==1') as "_ui_ng_if",
replace(pd.url,'/{id}','') as "detail_page_code",
eal.*,dt.code as "data_type_code",
tp.id as table_part_id,
tpc.title as table_part_title,
tpc.id as table_part_col_id,
coalesce(tpc.rq_mark,0) as table_part_col_rq_mark,

tpc.ui_width as "_ui_width_tp",

q.code as "query_code",
tp.code as "table_part_code",
coalesce(co0.ng_tp_template, co.ng_tp_template, co2.ng_tp_template) "_ui_component_ng_template",

coalesce(co0.ng_tp_template3, co.ng_tp_template3, co2.ng_tp_template3) "_ui_component_ng_template3",

coalesce(co0.angular_tp_template, co.angular_tp_template, co2.angular_tp_template) "_ui_component_angular_tp_template"

from pages p2 
join table_parts tp on tp.entity_id = p2.entity_id
join table_part_cols tpc on tpc.table_part_id=tp.id
join entity_attrs eal on eal.id = tpc.attr_id
left join entities e on e.id=eal.entity_link_id
left join queries q on q.id=e.def_sel_query_id
join data_types dt on dt.id=eal.data_type_id
LEFT JOIN pages pd ON pd.id = e.detail_page_id

LEFT JOIN ui_components co0 ON co0.id = tpc.ui_comp_id
LEFT JOIN ui_components co ON co.id = eal.ui_component_id
LEFT JOIN ui_components co2 ON co2.id = dt.ui_default_component_id


where p2.id=?


order by coalesce(tpc.nn,eal.nn_field,eal.id)
```

### table_part_cols_a

```sql
select 
tpc_at.code as table_part_cols_a_t_code,
tpc_a.html as table_part_cols_a_html,
tpc_a.attr_autosave as table_part_cols_a_autosave,

tpc_a.id as table_part_cols_a_id,
tpc_a.id as "_tpc_a_id",
tpc_a.tp_id as "_tpc_a_tp_id",

tpc_a.table_part_cols_id as table_part_cols_id,
tpc_a.ui_ng_readonly as "_ui_ng_readonly_tp",
tpc.ui_width as "_ui_width_tp",
coalesce(nullif(trim(({{if .Oracle}}TO_CHAR{{end}}(tpc_a.ui_ng_if))), ''), '1==1') as "_ui_ng_if",
replace(pd.url,'/{id}','') as "detail_page_code",
coalesce(pd.code,p3.code) as "angular_detail_page_code",

eal.*,dt.code as "data_type_code",
tp.id as table_part_id,

tpc.title as table_part_title,

q.code as "query_code",
tp.code as "table_part_code",
coalesce(co0.angular_tp_template, co.angular_tp_template, co2.angular_tp_template) "_ui_component_angular_tp_template"



from 
pages p2
join entities e2 on e2.id=p2.entity_id
join table_parts tp on tp.entity_id = e2.id
join table_part_cols tpc on tpc.table_part_id=tp.id
join table_part_cols_a tpc_a on tpc_a.table_part_cols_id = tpc.id
left join table_part_cols_a_t tpc_at on tpc_at.id = tpc_a.t_id
left join entity_attrs eal on eal.id = tpc_a.attr_id
left join entities e on e.id=eal.entity_link_id
left join entities e3 on e3.id=eal.entity_id
left join pages p3 on p3.id=e3.detail_page_id
left join queries q on q.id=e.def_sel_query_id
left join data_types dt on dt.id=eal.data_type_id
LEFT JOIN pages pd ON pd.id = e.detail_page_id

LEFT JOIN ui_components co0 ON co0.id = tpc_a.ui_comp_id
LEFT JOIN ui_components co ON co.id = eal.ui_component_id
LEFT JOIN ui_components co2 ON co2.id = dt.ui_default_component_id


where p2.id=?


order by coalesce(tpc_a.nn, tpc.nn, eal.nn_field,eal.id)
```

### table_parts

```sql
select main.*, ui_block_id.title as "ui_block_id$",

replace(p.url,'/{id}','') as "detail_page_code",
(select count(*) from table_parts_tp where table_parts_id = main.id) as _table_parts_tp_count,

el.title as "_entity_link_title",
el.code as "_entity_link_code",
ea.code "_link_attr_code",
coalesce(nullif({{if .Oracle}}TO_CHAR{{end}}(main.ng_if),''),('1==1')) "_ng_if",
tpt.ng_template as "_type_ng_template",
tpt.ng_template3 as "_type_ng_template3",
tpt.angular_template as "_type_angular_template",
ui_block_id.id as "_tp_ui_block_id",
main.th_style as "_th_style",
main.td_style as "_td_style"

from 
pages p2 
join entities e2 on e2.id=p2.entity_id
join table_parts main on main.entity_id = e2.id
left join table_part_types tpt on tpt.id=main.type_id
left join entities el on el.id=main.entity_link_id
left join entity_attrs ea on ea.id=main.link_attr_id
left join pages p on p.id = el.detail_page_id
left join pages_ui_blocks_n n on n.tp_id = main.id
left join pages_ui_blocks_n_t t on t.id = n.t_id and t.code='tp'
left join pages_ui_blocks ui_block_id on ui_block_id.id = n.pages_ui_blocks_id
where 
p2.id=? order by coalesce(n.nn,main.nn)
```

### table_parts_cross

```sql
select main.*, ui_block_id.title as "ui_block_id$",

replace(p.url,'/{id}','') as "detail_page_code",

el.title as "_entity_link_title",
el.code as "_entity_link_code",
ea.code "_link_attr_code",
coalesce(nullif({{if .Oracle}}TO_CHAR{{end}}(main.ng_if),''),('1==1')) "_ng_if",
tpt.ng_template as "_type_ng_template",
tpt.ng_template3 as "_type_ng_template3"
from pages p2 
join entities e on e.id=p2.entity_id
join table_parts tp on tp.entity_id = e.id
join table_parts_cross tpc on tpc.table_part_id = tp.id
join table_parts main on main.id=tpc.cross_table_part_id
left join table_part_types tpt on tpt.id=main.type_id
left join entities el on el.id=main.entity_link_id
left join entity_attrs ea on ea.id=main.link_attr_id
left join pages p on p.id = el.detail_page_id
left join entity_ui_blocks ui_block_id on ui_block_id.id = main.ui_block_id
where 
p2.id=? order by main.nn
```

### table_parts_cross_cols

```sql
select 
eal.id as attr_id,
eal2.code "_link_attr_code",
tp2.entity_id cross_entity_id,
replace(pd.url,'/{id}','') as "detail_page_code",
eal.*,dt.code as "data_type_code",
tp2.id as "table_part_id",
q.code as "query_code",
tp2.code as "table_part_code",
coalesce(nullif({{if .Oracle}}TO_CHAR{{end}}(tpc.ui_ng_if),''),'1==1') as "ui_ng_if",
coalesce(co0.ng_tp_cross_template, co.ng_tp_cross_template, co2.ng_tp_cross_template) "_ui_component_ng_template",
coalesce(co0.ng_tp_cross_template, co.ng_tp_cross_template, co2.ng_tp_cross_template) "_ui_component_ng_template3",
coalesce(co0.angular_tp_template, co.angular_tp_template, co2.angular_tp_template) "_ui_component_angular_tp_template",
tp.id,tpcr.cross_table_part_id from 
pages p
join entities e2 on e2.id=p.entity_id
join table_parts tp on tp.entity_id = e2.id
join table_parts_cross tpcr on tpcr.table_part_id = tp.id
join table_parts tp2 on tp2.id=tpcr.cross_table_part_id
join table_part_cols tpc on tpc.table_part_id=tp2.id
join entity_attrs eal on eal.id = tpc.attr_id
join entity_attrs eal2 on eal2.id = tp2.link_attr_id
left join entities e on e.id=eal.entity_link_id
left join queries q on q.id=e.def_sel_query_id
join data_types dt on dt.id=eal.data_type_id
LEFT JOIN pages pd ON pd.id = e.detail_page_id

LEFT JOIN ui_components co0 ON co0.id = tpc.ui_comp_id
LEFT JOIN ui_components co ON co.id = eal.ui_component_id
LEFT JOIN ui_components co2 ON co2.id = dt.ui_default_component_id

where p.id  = ?
```

### table_parts_tp

```sql
select tptp.*
from 
pages p2 
join entities e2 on e2.id=p2.entity_id
join table_parts tp on tp.entity_id = e2.id
join table_parts_tp tptp on tptp.table_parts_id=tp.id
where 
p2.id=? order by tp.nn
```

### table_parts_uq

```sql
select main.*,

replace(p.url,'/{id}','') as "detail_page_code",
(select count(*) from table_parts_tp where table_parts_id = main.id) as _table_parts_tp_count,

el.title as "_entity_link_title",
el.code as "_entity_link_code",
ea.code "_link_attr_code",
coalesce(nullif({{if .Oracle}}TO_CHAR{{end}}(main.ng_if),''),('1==1')) "_ng_if",
tpt.ng_template as "_type_ng_template",
tpt.ng_template3 as "_type_ng_template3",
tpt.angular_template as "_type_angular_template"
-- ui_block_id.id as "_tp_ui_block_id"

from 
pages p2 
join entities e2 on e2.id=p2.entity_id
join table_parts main on main.entity_id = e2.id
left join table_part_types tpt on tpt.id=main.type_id
left join entities el on el.id=main.entity_link_id
left join entity_attrs ea on ea.id=main.link_attr_id
left join pages p on p.id = el.detail_page_id
-- left join pages_ui_blocks_n n on n.tp_id = main.id
-- left join pages_ui_blocks_n_t t on t.id = n.t_id and t.code='tp'
-- left join pages_ui_blocks ui_block_id on ui_block_id.id = n.pages_ui_blocks_id
where 
p2.id=? order by main.nn
```

### ui_attrs

```sql
SELECT
coalesce(n.attr_autosave,0) "pages_ui_blocks_n_attr_autosave",
replace(pd.url,'/{id}','') as "detail_page_code",
pd.code as "angular_detail_page_code",

case
when e.restrict_edit=1 then 'true'
when e.is_formula=1 then 'true'
when trim(e.ui_ng_disabled) is not null and LENGTH(e.ui_ng_disabled)>1 then {{if .Oracle}}TO_CHAR{{end}}(e.ui_ng_disabled)
--when entity.readonly_detail_sql is not null  then 'detail.readonly$ == 1'
else 

concat('!(editable_row || ',' editable.',e.code,')')

end as "readonly",
concat('detail.',e.code) as "ng_model",
coalesce(co0.code,co.code, co2.code) "_ui_component_code",
coalesce(co0.ng_template,co.ng_template, co3.ng_template, co2.ng_template,'') "_ui_component_ng_template",
coalesce(co0.angular_template,co.angular_template, co3.angular_template, co2.angular_template,(case when nt.code in( 'html','logic') then n.angular_html end),'') "angular_template",
coalesce(co0.ng_ctrl_template,co.ng_ctrl_template, co3.ng_ctrl_template, co2.ng_ctrl_template) "_ui_component_ng_ctrl_template", 
coalesce(co0.angular_class_template,co.angular_class_template,co3.angular_class_template, co2.angular_class_template,'') "_ui_component_angular_class_template", 

nt.code as "pages_ui_blocks_n_t_code",
n.id as "pages_ui_blocks_n_id",


coalesce(co0.ng_template3,co.ng_template3, co2.ng_template3) "_ui_component_ng_template3",
coalesce(co0.ng_ctrl_template3,co.ng_ctrl_template3, co2.ng_ctrl_template3) "_ui_component_ng_ctrl_template3",
coalesce(co0.angular_tp_template,co.angular_tp_template, co2.angular_tp_template,'') "_ui_component_angular_tp_template", 



(SELECT title FROM entities WHERE id = e.entity_link_id) entity_link_title,
(SELECT code FROM entities WHERE id = e.entity_link_id) entity_link_code,
(SELECT q.code FROM entities eee, queries q
WHERE eee.def_sel_query_id = q.id
AND eee.id = e.entity_link_id) sel_query_code,
p.code lookup_page_code, e.*, dt.title data_type_title,
coalesce({{if .Oracle}}TO_CHAR{{end}}(e.help_html),'') as "_help_html",
dt.code data_type_code, coalesce(nullif(trim(({{if .Oracle}}TO_CHAR{{end}}(e.ui_ng_if))), ''), '1==1') "_ui_ng_if",
coalesce(w2.css_class,nullif(trim(w.css_class), ''), 'col-md-4 mb-3') "_ui_width_css",
n.pages_ui_blocks_id
FROM
pages_ui_blocks_n n 
join pages_ui_blocks_n_t nt on nt.id=n.t_id 
join pages_ui_blocks b on b.id=n.pages_ui_blocks_id
left join entity_attrs e on e.id=n.attr_id and coalesce(e.hide_field, 0) = 0
left JOIN data_types dt ON dt.id = e.data_type_id
LEFT JOIN entities ee ON ee.id = e.entity_link_id
left join entities entity ON entity.id=e.entity_id
LEFT JOIN pages p ON p.id = ee.lookup_page_id
LEFT JOIN pages pd ON pd.id = ee.detail_page_id
LEFT JOIN ui_widths w ON w.id = e.ui_width_id
LEFT JOIN ui_widths w2 ON w2.id = n.ui_width_id
LEFT JOIN ui_components co0 ON co0.id = n.ui_comp_id
LEFT JOIN ui_components co ON co.id = e.ui_component_id
LEFT JOIN ui_components co2 ON co2.id = dt.ui_default_component_id
LEFT JOIN ui_components co3 ON co3.id = n.ui_comp_id
WHERE 
nt.code in ('attr','html','ui_component','logic')
and b.pages_id = ? ORDER BY coalesce(n.nn,e.nn_field,e.id) ASC
```

### ui_blocks

```sql
select 
b.title,b.pages_id,b.code,b.header_title ,b.ui_width_id,  b.id,
case when b.is_auto =1 then ubt.angular_template else
b.angular_template end as angular_template , b.is_auto , b.nn, b.ng_if , b.ng_disabled , b.ui_block_tpl_id 
from pages_ui_blocks b
left join ui_block_tpl ubt on ubt.id=b.ui_block_tpl_id 
WHERE b.pages_id = ?

and not exists (select 1 from pages_ui_blocks_n n 
join pages_ui_blocks_n_t t on t.id=n.t_id
where n.block_id = b.id
and t.code='block'
) 
order by coalesce(b.nn,b.id)
```

### ui_blocks2

```sql
SELECT 
coalesce(w.css_class,'col-md-12') "_ui_width_css",
b2.title,b2.pages_id,b2.code,b2.header_title ,b2.ui_width_id, b2.id,
case when b2.is_auto =1 then ubt.angular_template else
b2.angular_template end as angular_template , b2.is_auto , b2.nn, b2.ng_if , b2.ng_disabled , b2.ui_block_tpl_id,

n.pages_ui_blocks_id,t.code as pages_ui_blocks_n_t_code,n.tp_id as pages_ui_blocks_n_tp_id  FROM 
pages_ui_blocks b
join pages_ui_blocks_n n on n.pages_ui_blocks_id = b.id
join pages_ui_blocks_n_t t on t.id=n.t_id
left join pages_ui_blocks b2 on b2.id=n.block_id
left join ui_block_tpl ubt on ubt.id=b2.ui_block_tpl_id 
left join ui_widths w on w.id=n.ui_width_id
WHERE b.pages_id = ? and t.code in ('block','tp')
order by coalesce(n.nn, b.nn,b.id)
```

## Приложение B. SQL runtime queries

### bp_processes_tools_by_entity_id

```sql
SELECT main.title,main.code,main.id,main.icon,main.icon_value FROM bp_processes main
join bp_process_types pt on pt.id = main.type_id
WHERE main.action_entity_id = ?  AND pt.code = 'entityTool'
and exists (select 1 from user_roles ur join bp_process_roles br on br.role_id=ur.role_id and ur.user_id=:user_id and br.process_id=main.id and br.can_run = 1 limit 1)
```

### company_info_by_user

```sql
select main.id,main.title,ts.user_id as task_staff_user_id, c.adm_header_id as  adm_header_user_id ,

c.func_header_id as  func_header_user_id

from companies main
join users_companies c on c.company_id=main.id
left join task_staff ts on ts.id = c.task_staff_id

%filter%
and main.id = ? and c.users_id=:user_id and coalesce(c.is_delete,0)=0 and coalesce(c.is_arch,0)=0
%order%
```

### entity_acts_many

```sql
SELECT coalesce(main.condition_action, '1 = 1') condition_action,
main.title, main.code,
main.id,
main.action_icon,
pt.code process_type_code FROM
bp_processes main 
INNER JOIN bp_process_types pt ON pt.id = main.type_id 
LEFT JOIN entities e ON main.action_entity_id = e.id 
WHERE (pt.code IN ('entityAction', 'entityImport') AND e.id = ? OR 
pt.code = 'entityGlobalAction') AND main.is_active = 1 
AND coalesce(main.condition_action, '') = '' AND
coalesce(is_action_single_only, 0) = 0 AND
main.id IN
(SELECT pr.process_id FROM bp_process_roles pr INNER JOIN user_roles ur ON
ur.role_id = pr.role_id AND
ur.user_id = :user_id WHERE
pr.can_run = 1)
```

### get_filter_cols_by_code

```sql
SELECT 
d.fixed as "_fixed",
co.code as "_ui_comp_code",
fsc.*,
coalesce(dt2.code, dt.code) "_data_type_code",
coalesce(elq3.code, elq.code, elq2.code) "_def_sel_query_code"
FROM filter_sets fs 
INNER JOIN filter_set_cols fsc ON fsc.set_id = fs.id
LEFT JOIN filter_set_dtls d ON d.id = fsc.default_dtl_id --AND coalesce(d.by_child_attr, 0) = 0 
LEFT JOIN entity_attrs ea ON ea.id = d.entity_attr_id LEFT JOIN data_types dt ON dt.id = ea.data_type_id 
LEFT JOIN entities el ON el.id = ea.entity_link_id 
LEFT JOIN queries elq ON el.def_sel_query_id = elq.id 
LEFT JOIN filter_set_dtls d2 ON d2.id = fsc.default_dtl_id --AND coalesce(d2.by_child_attr, 0) = 1 
LEFT JOIN entities el2 ON el2.id = d2.entity_link_id 
LEFT JOIN queries elq2 ON el2.def_sel_query_id = elq2.id 
LEFT JOIN queries elq3 ON d.query_id = elq3.id 
LEFT JOIN data_types dt2 ON dt2.id = d2.data_type_id 
LEFT JOIN ui_components co on co.id=fsc.ui_comp_id

WHERE fs.code = ? ORDER BY coalesce(fsc.nn,ea.id) ASC
```

### recent_rows$angular

```sql
{{if .MySQL}}
select main.title,replace(concat('#',p.url),'{id}',main.pk) url from recent_rows main
join entities e on e.code COLLATE utf8_general_ci= main.table_name
join pages p on p.id = e.detail_page_id
where main.user_id = :user_id
and table_name = ?
and

 (now()< date_add(main.last_updated, interval 1 day) or now()< date_add(main.last_viewed, interval 1 day) ) 


order by
main.last_viewed desc,main.last_updated desc,
main.update_quantity desc, main.view_quantity desc

limit 5
{{end}}
{{if .Postgres}}
select main.title,

concat('/',p.code,'/',main.pk::text) url from recent_rows main
join entities e on e.code = main.table_name
join pages p on p.id = e.detail_page_id
where main.user_id = :user_id
and table_name = ?
and

 
(now()< main.last_updated + INTERVAL '1 day' or now()< main.last_viewed + INTERVAL '1 day' )

order by
main.last_viewed desc,main.last_updated desc,
main.update_quantity desc, main.view_quantity desc

limit 5
{{end}}
```

## Приложение C. Исходный pkg/page

Снимок для сверки и анализа; известные ограничения описаны выше.

```lua
--pkg/page
--2020-06-01 Yeldar Saumbayev
local pkg={}


pkg.genPages = (function(entity_id,user_id)
 
  val = {}
  local pages,errText,errNum = SqlQueryRows([[
  select p.id,t.angular_template,t.angular_json from pages p join page_tpls t on t.id=p.tpl_id where p.entity_id=? and p.is_auto = 1]],entity_id)
  
  if errNum~=0 then
        return errText,errNum
  end
  
  for kp,vp in pairs(pages) do
      
      
        val,errTex,errNum = Detail(vp.id,"pages_for_gen_ng",user_id)
        
        
        --
        
for kdetails,vdetails in pairs(val) do
    
          val[kdetails]=array(val[kdetails])
          --print("llll",val[kdetails].component_param)
          if kdetails == "ui_attrs" then
              for ka,va in pairs(val[kdetails]) do
                  vaArr,errText,errNum = SqlQueryRows([[select p.attr,eac.value from pages_ui_blocks_n_cpv eac
                        join ui_comp_params p on p.id=eac.param_id
                        where eac.pages_ui_blocks_n_id = ? and nullif(eac.value,'') is not null ]],va.pages_ui_blocks_n_id)
                  if errNum~=0 then
                      var.last_error = "Ошибка bind параметров полей: "..errText
                      return
                  end                    
                  va.component_param = {}
                  for ka2,va2 in pairs (vaArr) do
                    if va2.value ~= nil and va2.value~="" then  
                        va.component_param[va2.attr] = va2.value
                    end
                  end
              end      


          end  
          
          if kdetails == "table_parts" then
              for ka,va in pairs(val[kdetails]) do
                  vaArr,errText,errNum = SqlQueryRows([[select p.code as attr,tpv.value from pages_ui_blocks_n n
join pages_ui_blocks_n_tpv tpv on tpv.pages_ui_blocks_n_id  = n.id
join table_part_params p on p.id=tpv.param_id 
where n.pages_ui_blocks_id = ?]],va["_tp_ui_block_id"])
                  if errNum~=0 then
                      var.last_error = "Ошибка bind параметров полей табличных частей: "..errText
                      return
                  end
                  va.param = {}

                  for ka2,va2 in pairs (vaArr) do
                    if va2.value ~= nil and va2.value~="" then  
                        va.param[va2.attr] = va2.value
                    end
                  end
              
                  vaArr,errText,errNum = SqlQueryRows([[select ea.id as entity_attr_id, p.attr,eac.value
                    from table_parts main
                    join table_part_types tpt on tpt.id=main.type_id
                    join entities el on el.id=main.entity_link_id
                    join entity_attrs ea on ea.entity_id  = el.id
                    join entity_attr_uicmps eac on eac.attr_id = ea.id
                    join ui_comp_params p on p.id=eac.ui_comp_param_id
                    where 
                    main.entity_id=?]],entity_id)
                  if errNum~=0 then
                      var.last_error = "Ошибка bind параметров полей: "..errText
                      return
                  end        
                  

              
              
              
              end
        end--if
        
        
          if kdetails == "table_part_cols" then
              for ka,va in pairs(val[kdetails]) do

                  va.component_param = {}

                  vaArr,errText,errNum = SqlQueryRows([[select ea.id as entity_attr_id, p.attr,eac.value
                    from table_parts main
                    join table_part_types tpt on tpt.id=main.type_id
                    join entities el on el.id=main.entity_link_id
                    join entity_attrs ea on ea.entity_id  = el.id
                    join entity_attr_uicmps eac on eac.attr_id = ea.id
                    join ui_comp_params p on p.id=eac.ui_comp_param_id
                    where 
                    main.id=? and ea.id=?]],va.table_part_id,va.id)
                  if errNum~=0 then
                      var.last_error = "Ошибка bind параметров полей: "..errText
                      return
                  end        
                  
                  --error(v.id.. JsonToString(vaArr))
                  
                  for ka2,va2 in pairs (vaArr) do
                    if va2.value ~= nil and va2.value~="" then  
                        va.component_param[va2.attr] = va2.value
                        --error(JsonToString(va.component_param))
                    end
                  end
              
              
              
              end
        end--if
        
          if kdetails == "table_part_cols_a" then
              for ka,va in pairs(val[kdetails]) do

                  va.component_param = {}

                  vaArr,errText,errNum = SqlQueryRows([[select ca.attr_id  as attr_id, ucp.attr , pv.value from table_part_cols_a ca
join table_part_cols_a_pv pv on pv.table_part_cols_a_id=ca.id
join ui_comp_params ucp on ucp.id=pv.param_id
where ca.id=? and ca.attr_id=?]],va.table_part_cols_a_id,va.id)
                
                    print("QQQQQQQQQQQ",    va.table_part_cols_a_id,va.id)
                  if errNum~=0 then
                      var.last_error = "Ошибка bind параметров полей: "..errText
                      return
                  end        
                  
                  --error(v.id.. JsonToString(vaArr))
                  
                  for ka2,va2 in pairs (vaArr) do
                    if va2.value ~= nil and va2.value~="" then  
                        va.component_param[va2.attr] = va2.value
                        --error(JsonToString(va.component_param))
                    end
                  end
              
              
              
              end
        end--if
  end
  
        --
        
        
        --Реализация Page Params
        val.param = {}    
        local param,errText,errNum = SqlQueryRows([[
        select ppc.code ,  pp.value from page_params pp
        join page_param_cls ppc on ppc.id=pp.cls_id 
        where page_id=?
        ]],vp.id)
        
                
        if errNum~=0 then
            return errText,errNum
        end
        
        
        
        for cb_k,cb_v in pairs (param) do
            val.param[cb_v.code] = cb_v.value
        end
        --Реализация Page Params
        
        --Реализация Custom Blocks  
        val.custom_block = {}    
        
        custom_blocks,errText,errNum = SqlQueryRows([[select bp.code place_code, coalesce(b.angular_template,'') as angular_template  from page_cus_blocks b
        join page_cus_block_places bp on bp.id = b.place_id
        where b.page_id=?
        union all
        select bp.code place_code, coalesce(b.angular_template,'')  from entity_type_pc_block b
        join page_cus_block_places bp on bp.id = b.place_id
        where b.entity_types_id=?
        
        ]],vp.id,EntityValueById("entities","entity_type_id",entity_id))
        
        
        if errNum~=0 then
            return errText,errNum
        end
        
        for cb_k,cb_v in pairs (custom_blocks) do
            if val.custom_block[cb_v.place_code]==nil then
            val.custom_block[cb_v.place_code] = ""
            end
            val.custom_block[cb_v.place_code] = cb_v.angular_template.."\r\n"..val.custom_block[cb_v.place_code]
        end
        --Реализация Custom Blocks
        
    val.id = vp.id
    val.title = vp.title
    
    local angular_template,errText,errNum  = ParseTemplate(vp.angular_template,val,user_id)
    
    if errNum~=0 then
        return "error  angular_template " .. errText,errNum
    end   
    
    local angular_json,errText,errNum  = ParseTemplate(vp.angular_json,val,1)
    
    if errNum~=0 then
        return "error  angular_json " .. errText,errNum
    end   
    
    angular_template = StrReplace (angular_template,"{[{","{{",-1)
    angular_json = StrReplace (angular_json,"{[{","{{",-1)
    if os.getenv("CRM_DB_TYPE") == "oracle" then

      
    local resultPageUpd,errText,errNum= SqlCall("update /*oracle*/ pages set angular_template = :angular_template,angular_json=:angular_json where id=:id",
         {
            angular_template = { input=true, output=false,value=angular_template,clob = true },
            angular_json = { input=true, output=false,value=angular_json,clob = true },
            id = { input=true, output=false,value=vp.id }
         } 
    )          
    else
    local errText,errNum = SqlExec2("update pages set angular_template = ?,angular_json=? where id=?",angular_template,angular_json, vp.id)
    end

   end 
   
   
    
  
   
   return "",0

end
)

pkg.addDefaultPages = (function(entity_id,user_id)
    
    local arr,errText,errNum = SqlQueryRows([[select m.id as module_id, pg.id, e.title, pg.page_type_id, concat(e.code,coalesce(pg.suffix,'')) as code, concat(m.url_prefix,e.code,coalesce(pg.suffix,''),coalesce(pg.url_add,'')) url,   pg.page_tpl_id from entities e
    join modules m on m.id=e.module_id
    join entity_types_pagegen pg on pg.entity_types_id = e.entity_type_id
    where e.id=?
    and not exists (select 1 from pages p where p.code = concat(e.code,coalesce(pg.suffix,''))  and p.entity_id=e.id)
    ]],entity_id)


    

    if errNum~=0 then
        return errText,errNum
    end
    
    --if entity_id=="2996" then 
    --    local lpages,errText,errNum=SqlQueryRows([[select code from pages where entity_id=?]],entity_id)
    --    error("TEST111 cnt="..#arr.. " "..JsonToString(lpages)) 
    --end
        
    for k,v in pairs(arr) do
        
        
        local page_id,errText,errNum = DMLI("insert","pages",user_id, {db_template=1, module_id = v.module_id, code=v.code, title = v.title,entity_id = entity_id, tpl_id = v.page_tpl_id, url = v.url, page_type_id = v.page_type_id, is_auto = 1  })
        if errNum~=0 then
            return errText,errNum
        end   
        
        
        if HasSuffix(v.code,"details") then
            local errText,errNum = SqlExec2("update entities set detail_page_id=? where id=? and detail_page_id is null",page_id,entity_id)
            if errNum~=0 then
                return errText,errNum
            end            
        end
        
       
        
        
        
        local arr1,errText,errNum = SqlQueryRows([[select suffix,ui_block_tpl_id,header_title from entity_types_pagegen_b where entity_types_pagegen_id=?]],v.id)
    
        for k1,v1 in pairs(arr1) do
            local i,errText,errNum = DMLI("insert","pages_ui_blocks",user_id, {title = header_title,header_title = v1.header_title,pages_id = page_id, code=v.code..(v1.suffix or "") , ui_block_tpl_id=v1.ui_block_tpl_id,is_auto=1 })
            if errNum~=0 then
                return errText,errNum
            end
        end           
        
    end    
    
    
   return "",0
    
end    
)
    
    
--Генерация карточки
pkg.genCard = (function(page_id,data)
    
result = ""

if 1==1 then
    return result,"",0
end  


local arr,errText,errNum = SqlQueryRows([[select pc.id as page_card_id, c.template3 from pages p
join page_card pc on pc.page_id=p.id
join ui_card c on c.id=pc.card_id 
where p.id=? order by pc.nn]],page_id)


  
--if 1==1 then
--    return JsonToString(data),"",1
--end    


for k0,v0 in pairs(arr) do
    
    
    
    --data = {source = {detail = array( { [1]={customer = "EKA" } } )}}
    data.source = {}
    data.map = {}
    
    local ds,errText,errNUm = SqlQueryRows([[select pcd.id, uds.code, dq.code dq_code from 
    pages p
    join page_card pc on pc.page_id  = p.id
    join page_card_ds pcd on pcd.page_card_id  = pc.id
    join detail_queries dq on dq.id=pcd.detail_query_id 
    join ui_card_ds uds on uds.id=pcd.card_ds_id
    where pc.id=?
    ]],v0.page_card_id)

    for k,v in pairs(ds) do
    
            maps,errText,errNum = SqlQueryRows([[select f.code as detail_query_field_code,cf.code as ds_field_code from page_card_ds_field pcdf 
            join detail_query_field  f on f.id=pcdf.detail_query_field_id 
            join ui_card_ds_field cf on cf.id=pcdf.ui_card_ds_field_id 
            where pcdf.page_card_ds_id =?]],v.id)
        
            --dsarr2[k1] = {}
            --if 1==1 then
            --    return v.code,0,""
            --end    
            data.source[v.code] = {}
            data.map[v.code]="details."..v.dq_code
            
            for k2,v2 in pairs(maps) do
                data.source[v.code][v2.ds_field_code]=v2.detail_query_field_code
                
                --if 1==1 then
               --     return v.code.."."..v2.ds_field_code.."="..v2.detail_query_field_code,"",0
                --end    
                --dsarr2[k1][v2.ds_field_code] = dsarr[k1][v2.detail_query_field_code]
            end
    
        --data.source[v.code] = dsarr2
    end

    
   local res,errText,errNum = ParseTemplate(v0.template3, data,1)
   if errNum~=0 then
       return "",errText,errNum
   end  
   result = result..res

end    


return result,"",0
    
    

end
)


return pkg
```

[← Главная](index.html)
