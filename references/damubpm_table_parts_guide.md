# Табличные части DamuBPM

Руководство по устройству табличных частей (ТЧ): модель данных, колонки и их элементы, параметры, генерация страниц, загрузка данных, проверки и типичные ошибки. Описание сверено с метаданными, шаблонами и скриптами стенда test-helpdesk.

## Содержание

1. [Что такое табличная часть](#1)
2. [Модель данных](#2)
3. [Как создаётся табличная часть](#3)
4. [Загрузка строк: detail_queries](#4)
5. [Колонки и элементы колонок](#5)
6. [Параметры: четыре уровня](#6)
7. [Параметры табличной части](#7)
8. [Виды табличных частей](#8)
9. [Размещение на странице и генерация](#9)
10. [Работа на фронтенде](#10)
11. [Обязательность и проверки](#11)
12. [Вложенные и кросс-табличные части](#12)
13. [Поле code и перенос патчем](#13)
14. [Частые ошибки](#14)
15. [Полезные SQL](#15)

---

<a id="1"></a>
## 1. Что такое табличная часть

Табличная часть — это список строк подчинённой сущности, показанный внутри карточки родительской записи. Строки хранятся в своей таблице и ссылаются на родителя атрибутом-ссылкой.

```text
Родительская сущность (table_parts.entity_id)
  └── табличная часть table_parts.code
        └── строки связанной сущности (table_parts.entity_link_id)
              где <link_attr> = id родительской записи (table_parts.link_attr_id)
```

Пример: сущность `its_kuka_test_lvl1`, ТЧ `its_kuka_test_lvl2`. Строки лежат в таблице `its_kuka_test_lvl2`, связь — по полю, указанному в `link_attr_id`.

---

<a id="2"></a>
## 2. Модель данных

| Таблица | Назначение | Ключевые поля |
|---|---|---|
| `table_parts` | Табличная часть | `entity_id`, `code`, `title`, `entity_link_id`, `link_attr_id`, `type_id`, `nn`, `ng_if`, `rq`, `rq_tb`, `gen_show_cols`, `is_child_obj`, `sort_expr`, `help_text`, `th_style`, `td_style`, `ui_block_id`, `tab_nn` |
| `table_part_types` | Вид ТЧ (шаблон таблицы) | `code`, `angular_template`, `ng_template`, `ng_template3` |
| `table_part_cols` | Колонка | `table_part_id`, `attr_id`, `title`, `code`, `nn`, `ui_comp_id`, `ui_width`, `ui_ng_if`, `ui_ng_readonly`, `ui_ng_title`, `rq_mark`, `sort_nn` |
| `table_part_cols_a` | Элемент внутри колонки | `table_part_cols_id`, `t_id`, `attr_id`, `html`, `tp_id`, `code`, `ui_comp_id`, `ui_ng_if`, `ui_ng_readonly`, `nn`, `sort_nn`, `attr_autosave` |
| `table_part_cols_a_t` | Вид элемента | `attr` (1), `html` (2), `tp` (3) |
| `table_part_cols_a_pv` | Параметры UI-компонента элемента | `table_part_cols_a_id`, `param_id` → `ui_comp_params`, `value`, `code` |
| `table_part_params` | Справочник параметров ТЧ | `code`, `title`, `data_type_id` |
| `pages_ui_blocks_n_tpv` | Значения параметров ТЧ в размещении на странице | `pages_ui_blocks_n_id`, `param_id` → `table_part_params`, `value` |
| `table_part_param_values` | Значения параметров ТЧ на уровне самой ТЧ (старые шаблоны) | `table_part_id`, `param_id`, `value` |
| `entity_attr_uicmps` | Параметры компонента атрибута | `attr_id`, `ui_comp_param_id`, `value` |
| `table_parts_tp` | Вложенные ТЧ | `table_parts_id`, `tp_id`, `code`, `nn` |
| `table_parts_cross` | Кросс-табличные части | `table_part_id`, `cross_table_part_id` |

Смысл флагов `table_parts`:

| Поле | Подпись | Эффект |
|---|---|---|
| `gen_show_cols` | Генерировать видимые колонки | При публикации сущности создаёт колонки по атрибутам с `show_col = 1` |
| `rq` | Обязательно к заполнению | Сохранение без строк даёт ошибку |
| `rq_tb` | Не включать в проверку | Исключает ТЧ из проверки строк и обязательности |
| `ng_if` | Angular if | Условие показа всей таблицы (`_ng_if`) |
| `sort_expr` | Выражение сортировки | `order by` запроса строк, если нет полей с `sort_nn` |
| `th_style`, `td_style` | Стили | Шаблоны стилей заголовка и ячейки; разбираются через `ParseTemplate` в контексте колонки |
| `is_child_obj` | Дочерний объект | Признак дочерней сущности |
| `ui_block_id`, `tab_nn` | UI-блок, номер вкладки | Привязка в старой схеме `entity_ui_blocks` |

---

<a id="3"></a>
## 3. Как создаётся табличная часть

Штатный путь — процесс `entities_add_tp` («Добавить табличную часть») из карточки сущности. Он:

1. Создаёт сущность строк с кодом `<entity>_tp_<n>` и типом `table`.
2. Добавляет в неё ссылку `<entity>_id` на родителя (`data_type_id = 7`, правило `cascade`).
3. Создаёт `table_parts`: `entity_link_id` = новая сущность, `link_attr_id` = ссылка, `type_id = 1` (`default`), `gen_show_cols = 1`.
4. Добавляет переданные поля как атрибуты строк (`rq = 1`, `show_col = 1`).

Табличную часть можно построить и на существующей сущности: заполнить `entity_link_id` и `link_attr_id` вручную. Атрибут связи должен принадлежать связанной сущности и ссылаться на родителя.

После создания сущность нужно **опубликовать** (`entity_publish`). Публикация:

- при `gen_show_cols = 1` создаёт колонки для атрибутов связанной сущности с `show_col = 1`. Для каждого атрибута это `UpsertI("table_part_id;attr_id", "table_part_cols")` и `UpsertI("table_part_cols_id;attr_id", "table_part_cols_a")`, то есть колонка вместе с основным элементом;
- генерирует запрос загрузки строк (раздел 4);
- перегенерирует автостраницы сущности.

---

<a id="4"></a>
## 4. Загрузка строк: detail_queries

При публикации у Detail родительской сущности создаётся (`Upsert` по `code;is_auto;detail_id`) запрос `detail_queries` с кодом, равным `table_parts.code`:

```sql
select <поля связанной сущности, формулы, подписи ссылок>
from <schema>.<entity_link_code> main <joins>
where main.<link_attr_code> = ? and coalesce(main.in$trash,0) = 0
order by <sort_fields>
```

- `sort_fields` — список `main.<attr>` элементов колонок с `table_part_cols_a.sort_nn > 0`. Если таких нет, используется `table_parts.sort_expr`. Порядок полей внутри списка не задаётся.
- В выборку попадают атрибуты без `is_formula` и `data_hide_list`, отдельно — формулы и подписи ссылок.
- В runtime строки приходят в `details[<table_part_code>]`, в форме — `form_tb.get('<table_part_code>.values')`.

Если ТЧ переименована или изменена связь, запрос обновится только после повторной публикации.

---

<a id="5"></a>
## 5. Колонки и элементы колонок

### Колонка `table_part_cols`

Колонка — это заголовок и ячейка таблицы. Она привязана к атрибуту связанной сущности (`attr_id`) и несёт свойства отображения: `title`, `nn` (порядок), `ui_width` (минимальная ширина, px), `ui_ng_if`, `ui_ng_title`, `ui_ng_readonly`, `rq_mark` (красная звёздочка).

Порядок колонок: `coalesce(table_part_cols.nn, entity_attrs.nn_field, entity_attrs.id)`.

### Элемент `table_part_cols_a`

Содержимое ячейки задают элементы колонки. В одной колонке их может быть несколько (порядок — `nn`).

| `t_id` (вид) | Что выводит | Поля |
|---|---|---|
| `attr` (1) или пусто | UI-компонент атрибута | `attr_id`, `ui_comp_id`, параметры в `table_part_cols_a_pv` |
| `html` (2) | Произвольный HTML/Angular | `html` |
| `tp` (3) | Ссылка на другую ТЧ | `tp_id` |

На стенде у большинства элементов `t_id` пустой — такой элемент работает как `attr`.

**Важно:** в шаблоне вида `default` ячейка строится **только** из элементов `table_part_cols_a`. Колонка без элементов выводит пустую ячейку. Колонки, созданные публикацией, получают основной элемент автоматически; добавленные вручную — нет.

Компонент элемента вычисляется так:

```text
coalesce(table_part_cols_a.ui_comp_id,
         entity_attrs.ui_component_id,
         data_types.ui_default_component_id)
```

В ячейку подставляется `angular_tp_template` этого компонента (`_ui_component_angular_tp_template`). Для колонки (заголовок, старые шаблоны) действует тот же порядок, но с `table_part_cols.ui_comp_id` на первом месте.

Элемент вида `html` выводится как есть (`table_part_cols_a_html`). Внутри доступен Angular-контекст строки: `field`, `index`, `form_tb` и методы страницы.

---

<a id="6"></a>
## 6. Параметры: четыре уровня

| Карта в шаблоне | Источник | Ключ | Где применяется |
|---|---|---|---|
| `.param` ТЧ | `pages_ui_blocks_n_tpv` + `table_part_params` | `table_part_params.code` | Angular-страницы (генерация `pkg/page`) |
| `.param` ТЧ (старое) | `table_part_param_values` + `table_part_params` | `table_part_params.code` | Старые ng-шаблоны при публикации сущности |
| `.component_param` элемента | `table_part_cols_a_pv` + `ui_comp_params` | `ui_comp_params.attr` | Ячейки Angular-страниц |
| `.component_param` колонки | `entity_attr_uicmps` + `ui_comp_params` | `ui_comp_params.attr` | Датасет колонок (заголовки, старые шаблоны) |

Правила:

- Пустые и `null` значения в карты не попадают.
- Параметры ТЧ на странице читаются **по блоку**: `pages_ui_blocks_n.pages_ui_blocks_id = _tp_ui_block_id`, а не по конкретному узлу `tp`. Две ТЧ в одном блоке получат общий набор параметров.
- Параметры элемента берутся по условию `ca.id = ? and ca.attr_id = ?`. У HTML-элемента нет `attr_id`, поэтому параметров компонента у него не бывает.
- `table_part_cols_a_pv.param_id` должен ссылаться на параметр текущего компонента элемента. При смене компонента ищите старую строку по `ui_comp_params.attr`, а не по `param_id`, иначе появятся дубли одного атрибута.

Пример: `query_code` для select-ячейки с фильтром по полю карточки:

```text
table_part_cols_a_pv.value = entity_attrs_select&flt$entity_id$eq='+detail.entity_id+'
param_id → ui_comp_params (attr = 'query_code') компонента элемента
```

---

<a id="7"></a>
## 7. Параметры табличной части

Справочник `table_part_params` и то, как их использует шаблон `default`:

| code | Название | Эффект в `default` |
|---|---|---|
| `no_title` | Скрыть наименование | `"1"` — не выводить заголовок ТЧ |
| `hide_add_button` | Скрыть кнопку добавить | `"1"` — нет кнопки «Добавить» и «Excel» |
| `add_page_code` | Страница добавления | Добавление и редактирование строки в модальной странице `openPagesAsModal`; ячейки только для чтения |
| `add_page_detail` | Страница ввода для строки ТЧ | Строка раскрывается в `jit-page` под ней; ячейки только для чтения |
| `ng_readonly` | Условие только-чтения | Скрывает добавление и удаление, если условие истинно |
| `max_row_length` | Максимальное число строк | Блокирует «Добавить» при достижении |
| `autosave` | Автосохранение | `"1"` — `save()` после добавления и удаления строки |
| `hide_id_href` | Скрывать ссылку ID | Проверяется, но в текущем `default` ни на что не влияет |
| `delete` | Кнопка Удалить | В `default` не используется (там `hide_del_button`) |
| `selectall`, `expand`, `no_border`, `ng_class`, `add_page_code_callback` | — | В `default` не используются |

Шаблон `default` читает ещё четыре параметра, которых **нет** в справочнике: `hide_del_button`, `min_row_length`, `row_ng_if`, `modal_page_ok_title`. Чтобы задать их на странице, сначала добавьте строку в `table_part_params`.

Флаги сравниваются со строкой `"1"`.

---

<a id="8"></a>
## 8. Виды табличных частей

| code | Название | angular_template |
|---|---|---|
| `default` | Обычный | есть |
| `readonly` | Только чтение | есть |
| `readonly2` | Только чтение без навигации | только ng |
| `type1` | Вид 1 | только ng |
| `table_part_simple_add` | Простой списочный | только ng |
| `row` | Построчный вывод | только ng |

Для Angular-страниц пригодны только `default` и `readonly`. Вид без `angular_template` на современной странице не отрисуется.

---

<a id="9"></a>
## 9. Размещение на странице и генерация

На Angular-странице ТЧ выводится, только если она размещена в блоке: элемент `pages_ui_blocks_n` с типом `tp` и `tp_id = table_parts.id`. ТЧ без размещения на современной странице не видна. Параметры размещения хранятся в `pages_ui_blocks_n_tpv`.

Генерация (`pkg/page.genPages`) собирает датасеты `table_parts`, `table_part_attrs`, `table_part_cols`, `table_part_cols_a`, `table_parts_tp`, `table_parts_cross`, `table_parts_cross_cols` и «запекает» результат в `pages.angular_template`. Поэтому:

- любые изменения колонок, элементов и параметров видны только после перегенерации страниц (публикация сущности или `genPages`);
- `genPages` нужно запускать от пользователя с правами на `entities` (обычно 1). Иначе датасет `entities` пустой и генерация падает с `index .entities 0`;
- при вызове из крона объявите `var = {}` до `genPages`: она пишет в `var.last_error`.

Как шаблон `default` строит таблицу:

```text
{{range $value.table_part_cols}}         — заголовки и ячейки
  if entity_id == $value.entity_link_id   — фильтр колонок
  {{range $value.table_part_cols_a}}
    if table_part_cols_id == table_part_col_id
      html → вывести html
      иначе → parseTemplate _ui_component_angular_tp_template
```

Колонки фильтруются по `entity_id` атрибута, а не по `table_part_id`. Если у одной сущности две ТЧ на одну и ту же связанную сущность, заголовки обеих могут попасть в каждую таблицу.

---

<a id="10"></a>
## 10. Работа на фронтенде

| Что | Как |
|---|---|
| Строки формы | `form_tb.get('<tp_code>.values')` (FormArray), группа строки — `[formGroupName]="index"` |
| Данные Detail | `details['<tp_code>']` |
| Добавить строку | `tablePartAdd('<entity_link_code>', { <link_attr>: <id родителя> })` |
| Удалить строку | `tablePartDelete('<tp_code>', field.value.sys$uuid)` |
| Модальное редактирование | `openPagesAsModal('<add_page_code>', '<tp_code>', index)` |
| Строка в jit-page | `tp_jit_opened.<tp_code>` (раскрытые id), `getTablePartRowJitComponent` |
| Пагинация | `tb_perpage`, `tb_page`, `onTbPageChanged`, `paginator['<tp_code>']` |
| Родитель вложенной ТЧ | `getParentEntity('<tp_code>')` |
| Выгрузка в Excel | процесс `exp_to_excel_by_tp_code` (кнопка видна при доступе к процессу) |
| Редактируемость | `editable_row` или параметр `ng_readonly` |

Новая строка ещё не имеет `id`: используйте `sys$uuid`. Ссылка на родителя заполняется из `id` или `sys$uuid` родительской записи.

---

<a id="11"></a>
## 11. Обязательность и проверки

Проверка при сохранении (`pkg/error_rows`) берёт ТЧ сущности с `entity_link_id <> entity_id` и `coalesce(rq_tb,0) = 0`:

- `rq = 1` и строк нет → «Табличная часть '…' является обязательной к заполнению»;
- каждая строка проверяется как запись связанной сущности (обязательные поля и правила).

`rq_tb = 1` выключает обе проверки для этой ТЧ. Отметка `table_part_cols.rq_mark` только рисует звёздочку и ничего не проверяет. Звёздочка также выводится для атрибутов с `rq` или `rq_send`, если у ТЧ нет `add_page_code` и `add_page_detail`.

---

<a id="12"></a>
## 12. Вложенные и кросс-табличные части

**Вложенная ТЧ** (`table_parts_tp`) выводится под каждой строкой родительской ТЧ. Связь: `table_parts_id` — внешняя ТЧ, `tp_id` — вложенная, `code` — код запроса. Публикация собирает их в запрос `table_parts_tp$`. В шаблоне вложенная таблица рендерится через `{{ lua "gen_tp_by_tps_tp_angular_id" … }}`, если `_table_parts_tp_count` не равен `"0"`.

**Элемент вида `tp`** (`table_part_cols_a.tp_id`) — второй способ вложить ТЧ. `pkg/detail_queries` строит для него запрос с кодом **`table_part_cols_a.code`**: код такого элемента нельзя менять произвольно.

**Кросс-ТЧ** (`table_parts_cross`) связывает ТЧ с другой ТЧ (`cross_table_part_id`). Колонки задаются датасетом `table_parts_cross_cols`.

---

<a id="13"></a>
## 13. Поле code и перенос патчем

- У `table_part_cols`, `table_part_cols_a` и `table_part_cols_a_pv` есть поле `code`: уникальное без учёта регистра (индексы по `lower(code)`), по умолчанию `gen_random_uuid()::text`. Это устойчивый ключ строки между стендами, в отличие от `id`.
- Индекс и default не переносятся вместе с атрибутом. На целевом стенде их создают отдельно:

```sql
update table_part_cols      set code = gen_random_uuid()::text where coalesce(code,'') = '';
update table_part_cols_a    set code = gen_random_uuid()::text where coalesce(code,'') = '';
update table_part_cols_a_pv set code = gen_random_uuid()::text where coalesce(code,'') = '';
alter table table_part_cols      alter column code set default gen_random_uuid()::text;
alter table table_part_cols_a    alter column code set default gen_random_uuid()::text;
alter table table_part_cols_a_pv alter column code set default gen_random_uuid()::text;
create unique index if not exists table_part_cols_code_uindex      on table_part_cols (lower(code));
create unique index if not exists table_part_cols_a_code_uindex    on table_part_cols_a (lower(code));
create unique index if not exists table_part_cols_a_pv_code_uindex on table_part_cols_a_pv (lower(code));
```

- Экспорт патча не берёт `table_part_cols`, `table_part_param_values` и `table_part_cols_a_pv` как дочерние объекты сущности. Изменённые строки добавляйте в патч поштучно, вместе с перегенерированными страницами.
- Удаления патчем не переносятся: удалённые колонки, элементы и параметры нужно удалить на целевом стенде отдельно.

---

<a id="14"></a>
## 14. Частые ошибки

| Симптом | Причина | Что делать |
|---|---|---|
| Изменение не видно на странице | Страница не перегенерирована | Опубликовать сущность или запустить `genPages` |
| ТЧ не выводится на Angular-странице | Нет узла `tp` в `pages_ui_blocks_n` или вид без `angular_template` | Разместить ТЧ в блоке, вид `default`/`readonly` |
| Пустая ячейка в колонке | У колонки нет элемента `table_part_cols_a` | Добавить элемент `attr` с тем же атрибутом |
| Параметр ТЧ не действует | Записан в `table_part_param_values`, а страница Angular | Писать в `pages_ui_blocks_n_tpv` размещения |
| Параметры двух ТЧ смешались | Обе ТЧ в одном блоке, параметры читаются по блоку | Разнести ТЧ по разным блокам |
| Лишние колонки из другой ТЧ | Две ТЧ сущности на одну связанную сущность | Учитывать фильтр по `entity_id`; разделить связанные сущности |
| `query_code` не применился | `param_id` от другого компонента или значение пустое | Проверить эффективный компонент и `ui_comp_params.attr` |
| Дубли параметров элемента | Поиск по `param_id` после смены компонента | Искать по `ui_comp_params.attr`, лишние строки удалить |
| `index .entities 0` при генерации | Генерация под пользователем без грантов | Запускать от пользователя 1 |
| `hide_del_button` и т.п. нельзя выбрать | Нет строки в `table_part_params` | Добавить параметр в справочник |
| Порядок строк не тот | `sort_nn` элементов имеет приоритет над `sort_expr` | Очистить `sort_nn` или задать нужные поля |

---

<a id="15"></a>
## 15. Полезные SQL

Табличные части сущности:

```sql
select tp.id, tp.code, tp.title, el.code entity_link, la.code link_attr, t.code type_code
from table_parts tp
join entities e on e.id = tp.entity_id
left join entities el on el.id = tp.entity_link_id
left join entity_attrs la on la.id = tp.link_attr_id
left join table_part_types t on t.id = tp.type_id
where e.code = :entity_code
order by tp.nn, tp.id;
```

Колонки, элементы, эффективный компонент и параметры:

```sql
select c.id col_id, c.code col_code, c.title, ea.code col_attr, c.nn,
       ca.id elem_id, ca.code elem_code, coalesce(t.code,'attr') elem_type, eaa.code elem_attr,
       comp.code eff_component, p.attr param, pv.value, pv.code pv_code
from table_part_cols c
left join entity_attrs ea on ea.id = c.attr_id
left join table_part_cols_a ca on ca.table_part_cols_id = c.id
left join table_part_cols_a_t t on t.id = ca.t_id
left join entity_attrs eaa on eaa.id = ca.attr_id
left join data_types dt on dt.id = eaa.data_type_id
left join ui_components comp on comp.id = coalesce(ca.ui_comp_id, eaa.ui_component_id, dt.ui_default_component_id)
left join table_part_cols_a_pv pv on pv.table_part_cols_a_id = ca.id
left join ui_comp_params p on p.id = pv.param_id
where c.table_part_id = (select id from table_parts where code = :tp_code)
order by coalesce(c.nn, ea.nn_field, ea.id), ca.nn, ca.id;
```

Параметры ТЧ в размещениях на страницах:

```sql
select pg.code page, n.id node_id, b.id block_id, tpp.code param, tpv.value
from pages_ui_blocks_n n
join pages_ui_blocks b on b.id = n.pages_ui_blocks_id
join pages pg on pg.id = b.pages_id
left join pages_ui_blocks_n_tpv tpv on tpv.pages_ui_blocks_n_id = n.id
left join table_part_params tpp on tpp.id = tpv.param_id
where n.tp_id = (select id from table_parts where code = :tp_code);
```

Колонки без элементов (пустые ячейки):

```sql
select tp.code, c.id, c.title
from table_part_cols c
join table_parts tp on tp.id = c.table_part_id
where not exists (select 1 from table_part_cols_a a where a.table_part_cols_id = c.id);
```

Допустимые параметры компонента:

```sql
select p.attr, p.title
from ui_comp_params p
where p.comp_id = (select id from ui_components where code = :component_code)
order by p.attr;
```
