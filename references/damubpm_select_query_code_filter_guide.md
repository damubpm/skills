# DamuBPM — фильтр ссылочного поля через `query_code`

[← Главная](index.html)

Как на автогенерируемой detail-странице сузить выпадающий список ссылочного поля (`prime-select`) по значению другого поля карточки. Пример — «Номер банковского счета организации» показывает только счета организации, выбранной в поле «Организация». Проверено на стенде test-helpdesk 05.11.2026 на странице `its_cli_contractdetails`.

> Главная идея: **фильтр задаётся параметром компонента `query_code` у элемента страницы (`pages_ui_blocks_n_cpv`). Значение подставляется в Angular-выражение внутри одинарных кавычек, поэтому в нём можно склеить код query, фильтр `flt$<колонка>$eq=` и значение из `detail`.** После правки страницу нужно перегенерировать.

---

## 1. Где что лежит

```text
pages (is_auto = 1)
└── pages_ui_blocks
    └── pages_ui_blocks_n            ← элемент: attr_id, ui_comp_id
        └── pages_ui_blocks_n_cpv    ← параметры компонента: param_id, value
                └── ui_comp_params   ← attr = 'query_code', comp_id = компонент
```

| Таблица | Нужные колонки |
|---|---|
| `pages_ui_blocks_n` | `id`, `code`, `pages_ui_blocks_id`, `attr_id` (не `entity_attrs_id`), `ui_comp_id` |
| `pages_ui_blocks_n_cpv` | `pages_ui_blocks_n_id`, `param_id`, `value` |
| `ui_comp_params` | `id`, `attr`, `comp_id` |
| `entity_attrs` | `entity_link_id`, `ui_component_id` |
| `data_types` | `ui_default_component_id` (у `reference` — `select`, id 4) |
| `entities` | `def_sel_query_id` — query выбора по умолчанию |

Компонент элемента выбирается так: `pages_ui_blocks_n.ui_comp_id` → `entity_attrs.ui_component_id` → `data_types.ui_default_component_id`.

Параметр `query_code` есть у нескольких компонентов, и у каждого свой `ui_comp_params.id`. Для `select` на test-helpdesk это `id = 2`. На другом стенде id искать по `attr` и `comp_id`:

```sql
select cp.id, cp.attr, uc.code
from ui_comp_params cp join ui_components uc on uc.id = cp.comp_id
where cp.attr = 'query_code' and uc.code = 'select';
```

---

## 2. Как компонент `select` использует `query_code`

Фрагмент `ui_components.angular_template` (id 4):

```html
<prime-select
    formControlName="{{.code}}"
    {{if isnil .component_param.flt }}{{else}}[flt]="{{.component_param.flt}}"{{end}}
    {{if isnil .component_param.flteq }}{{else}}[flteq]="{{.component_param.flteq}}"{{end}}
    {{if isnil .component_param.fltin }}{{else}}[fltin]="{{.component_param.fltin}}"{{end}}
    {{if isnil .component_param.param1 }}{{else}}[param1]="{{.component_param.param1}}"{{end}}
    [query-code]="'{{if isnil .component_param.query_code }}{{.sel_query_code}}{{else}}{{.component_param.query_code}}{{end}}'"
>
```

- Без параметра берётся `sel_query_code` — query по умолчанию связанной сущности (`entities.def_sel_query_id`).
- Значение параметра попадает **внутрь** `'…'`. Значение вида

  ```text
  companies_bank_select&flt$companies_id$eq='+detail.company_id+'
  ```

  превращается в выражение Angular

  ```html
  [query-code]="'companies_bank_select&flt$companies_id$eq='+detail.company_id+''"
  ```

  то есть в строку `companies_bank_select&flt$companies_id$eq=13724`, которая пересчитывается при изменении `detail.company_id`.
- Фильтр `flt$<колонка>$eq=<значение>` применяется к `main.<колонка>` таблицы query. Колонку не обязательно выводить в select: в `its_cli_tp_bank_select` нет `its_cli_id`, а фильтр `flt$its_cli_id$eq` работает.
- Пустые значения параметров генератор пропускает (`nullif(value,'') is not null`).
- Параметры `flt`, `flteq`, `fltin`, `fltnotin`, `param1`, `param2`, `limit` в шаблоне есть, но как они работают, на стенде не проверялось.

---

## 3. Рабочие примеры на стенде

| Страница | Поле | `query_code` |
|---|---|---|
| `its_cli_contractdetails` | `companies_bank_id` | `companies_bank_select&flt$companies_id$eq='+detail.company_id+'` |
| `its_d_invoicedetails` | `its_cli_tp_bank_id` | `its_cli_tp_bank_select&flt$its_cli_id$eq='+detail.its_cli_id+'` |
| `its_d_invoicedetails` | `its_cli_contract_id` | `its_cli_contract_select&flt$its_cli_id$eq='+detail.its_cli_id+'` |
| `its_stmt_in_enrolldetails` | `its_cli_contract_item_id` | `its_cli_contract_item_select&flt$its_cli_contract_id$eq='+detail.its_cli_contract_id+'` |
| `its_d_statementsdetails` | `staff_position_id` | `staff_position_select&flt$deps_id$eq='+detail.new_dep_id+'` |
| `its_indetails` | `receiver_cli_id` | `its_cli_select_my_branch` — фильтр внутри SQL query (`:user_id`) |

Если условие нельзя выразить фильтром по колонке, делают отдельный query с условием в SQL после `%filter%` (как `its_cli_select_my_branch`) и указывают его код без `flt$`.

---

## 4. Порядок работы

### 4.1. Найти элемент

```sql
select n.id n_id, n.code, n.ui_comp_id, ea.code attr, ea.title, el.code link_ent,
       (select json_agg(json_build_object('attr', cp.attr, 'v', v.value))
          from pages_ui_blocks_n_cpv v join ui_comp_params cp on cp.id = v.param_id
         where v.pages_ui_blocks_n_id = n.id) cpv
from pages p
join pages_ui_blocks b on b.pages_id = p.id
join pages_ui_blocks_n n on n.pages_ui_blocks_id = b.id
left join entity_attrs ea on ea.id = n.attr_id
left join entities el on el.id = ea.entity_link_id
where p.code = 'its_cli_contractdetails' and ea.code = 'companies_bank_id';
```

### 4.2. Проверить query связанной сущности

```sql
select e.code, q.code qcode, q.sql_text
from entities e left join queries q on q.id = e.def_sel_query_id
where e.code = 'companies_bank';
```

Убедиться, что в таблице есть колонка для фильтра (`companies_bank.companies_id`) и в SQL есть `%filter%`.

### 4.3. Записать параметр

Кавычки собирать через `chr(39)` — так не ошибиться с экранированием:

```sql
with u as (
  update pages_ui_blocks_n set ui_comp_id = 4
  where id = 18949 and ui_comp_id is null returning id
), i as (
  insert into pages_ui_blocks_n_cpv (pages_ui_blocks_n_id, param_id, value)
  select 18949, 2,
         'companies_bank_select&flt$companies_id$eq=' || chr(39) || '+detail.company_id+' || chr(39)
  where not exists (select 1 from pages_ui_blocks_n_cpv
                    where pages_ui_blocks_n_id = 18949 and param_id = 2)
  returning id, value
)
select (select count(*) from u) upd, (select json_agg(i) from i) ins;
```

`ui_comp_id = 4` ставится явно, чтобы параметр относился к тому компоненту, который реально выводится. Для reference-атрибута это тот же компонент по умолчанию.

### 4.4. Перегенерировать страницу

Запись в `pages_ui_blocks_n_cpv` страницу **не** перегенерирует: `pages.updated_at` и `angular_template` не меняются. Генерация — `pkg/page.genPages(entity_id, user_id)` (передаётся id **сущности**, генерируются все её автостраницы). Одноразовый крон, который сам себя выключает:

```lua
SqlExec2("update crontab set is_active = 0, warning_text = 'running' where code = 'its_cli_contract_genpages_once'")
Commit()
local res = ""
local ok, e = pcall(function()
  local page = dbrequire("pkg/page")
  local errText, errNum = page.genPages(3190, 1)
  res = "genPages: " .. tostring(errNum) .. " " .. tostring(errText)
end)
if not ok then res = "error: " .. tostring(e) end
SqlExec2("update crontab set is_active = 0, warning_text = ? where code = 'its_cli_contract_genpages_once'", res)
Commit()
```

Создать через MCP `create_crontab` (`croninterval` `*/10 * * * * *`, `is_active = true`, модуль сущности). Через 10–20 секунд в `warning_text` должно быть `genPages: 0`, `is_active = 0`.

### 4.5. Проверить результат

```sql
select (select is_active::text || ' | ' || coalesce(warning_text, '')
          from crontab where code = 'its_cli_contract_genpages_once') cron,
       p.updated_at,
       substr(p.angular_template,
              position('[query-code]="''companies_bank_select' in p.angular_template), 120) frag
from pages p where p.id = 12053;
```

Ожидается `[query-code]="'companies_bank_select&flt$companies_id$eq='+detail.company_id+''"`.

---

## 5. Подводные камни

1. Колонка ссылки на атрибут в `pages_ui_blocks_n` — `attr_id`. Запрос с `entity_attrs_id` падает с `column does not exist`.
2. У `data_types` нет `ui_component_id` — колонка называется `ui_default_component_id`.
3. Тройные кавычки (`eq='''+detail…`) дают `unterminated quoted string`. Использовать `chr(39)`.
4. `$` внутри строкового литерала (`flt$companies_id$eq`) безопасен. Нельзя только `$1`, `$2`… — MCP `db_execute_sql` примет их за параметры. Символ `?` в литерале тоже ломается (`chr(63)`).
5. `db_execute_sql` принимает SQL в Base64; из нескольких операторов возвращает только первый результат, поэтому изменения удобно собирать в один `with … returning`.
6. Без перегенерации правка не видна. `genPages` принимает id сущности, а не страницы.
7. Пока поле-источник пустое, выражение даёт `…$eq=null` или `…$eq=undefined`, и список пустой. Это ожидаемо, но пользователю стоит подсказать, что сначала выбирается организация.
8. При смене поля-источника уже выбранное значение зависимого поля не очищается. Если нужно, это отдельная логика на странице.
9. Значение, которое само в кавычках (`'its_staff_bank_select'`, встречается на `its_d_hr_contract_staff_ndetails`), после подстановки даёт `''its_staff_bank_select''`. Такое выражение подозрительно; в браузере не проверено. Писать код query без кавычек.
10. Фильтр в `query_code` — только удобство интерфейса. Права на строки определяются `entity_grants` и `entity_view_limits`; серверную проверку, что счёт принадлежит организации договора, фильтр не заменяет.

---

## 6. Короткая памятка

```text
ЗАВИСИМЫЙ СПИСОК НА DETAIL-СТРАНИЦЕ
1. pages_ui_blocks_n (attr_id) → id элемента
2. entities.def_sel_query_id → код query, есть ли колонка и %filter%
3. ui_comp_params: attr='query_code' для своего компонента → param_id
4. pages_ui_blocks_n_cpv.value = <query>&flt$<колонка>$eq='+detail.<поле>+'
   (кавычки через chr(39))
5. pkg/page.genPages(<entity_id>, 1) одноразовым кроном → genPages: 0
6. pages.angular_template содержит [query-code]="'<query>&flt$…'+detail.<поле>+''"
7. Проверить в браузере: пустой источник, смена источника, сохранение
```

[← Главная](index.html)
