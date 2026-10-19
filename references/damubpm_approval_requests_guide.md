# DamuBPM: заявки с согласованием и маршруты (этапы согласования)

Инструкция по созданию новой заявки (сущности) с маршрутом согласования и по редактированию этапов. Основана на разборе рабочей реализации `its_d_statements` (Заявления работника) в среде onedoc.

---

## 1. Модель данных

```
entities.task_approve_t_id ──┐                       (шаблон по умолчанию для сущности)
<тип документа>.task_approve_t_id ──┐                (шаблон по виду документа, как its_d_statements_type)
                                    ▼
                         task_approve_t              Шаблон согласования
                                    │ 1:N
                                    ▼
                         task_approve_t_s            Этап (шаг) шаблона
                           ├── task_approve_t_s_roles   роли этапа (role_id, send_by_role)
                           ├── task_approve_t_s_c       собственные процессы (кнопки результата)
                           ├── task_approve_res_t       наименования кнопок
                           ├── task_approve_t_s_term    интервалы оповещения
                           └── task_approve_t_s_deps    руководители департаментов
                                    │ копируется при создании записи
                                    ▼
                         tasks (entity_id, entity_pk = sys$uuid записи, approve_t_s_id)
                                    │
                                    ▼
                         its_req (сквозная регистрационная карточка: stat_id, docnum)
```

### 1.1 Справочники

**task_approve_type** — тип согласования внутри шага (`tasks.approve_type_id`):

| id | code | title |
|---|---|---|
| 1 | parallel | Параллельно |
| 2 | serial | Последовательно |

**task_approve_t_s_rt** — вид ответственного этапа (`task_approve_t_s.rt_id`):

| id | code | title | Откуда берётся исполнитель |
|---|---|---|---|
| 1 | users_companies | Сотрудник | выбор карточки сотрудника |
| 2 | user | Пользователь | `task_approve_t_s.user_id` |
| 3 | task_staff | Формула | SQL из `task_staff.formula` (`:user_id`, `:entity_pk`) |
| 4 | role | Роль | `task_approve_t_s.role_id` / `task_approve_t_s_roles` |
| 5 | staff_position | Штатная должность | штатная должность |

**task_types** (вид задачи этапа, `type_id`) — основные: `approve`(1) Согласование, `sign`(2) Подписано, `sign_eds`(3), `exec`(4) Исполнение, `fyi`(6) Ознакомление, `task_sign_eds`(8) Подписание, `register`(9) Регистрация, `approval`(27) Утверждение.

**task_approve_res** (результат задачи) — основные: `approved`(1), `rejected`(2), `sign`(4, set_docnum=1), `exec`(6), `fiy_ok`(8), `rework`(14) На доработке, `registered`(16), `affirm`(50). Флаг `set_docnum=1` присваивает номер документа при закрытии задачи.

**task_approve_filluser** (`filluser_id`, «Заполнить пользователем»): 1 Руководитель организации, 2 Адм. руководитель, 3 Функц. руководитель, 4 Я, 7 Руководитель отдела, 9 Директор департамента и т.д.

### 1.2 task_approve_t (шаблон)

| Поле | Назначение |
|---|---|
| title, code | Наименование / код шаблона |
| user_id | Владелец личного шаблона (пусто — общий) |
| can_add_task | Инициатор может добавлять согласующих |
| can_block | Может блокировать почту |

### 1.3 task_approve_t_s (этап) — ключевые поля

| Группа | Поля |
|---|---|
| Порядок | `step_nn` — номер шага; `nn` — подпункт внутри шага; `approve_type_id` — параллельно/последовательно внутри шага |
| Исполнитель | `rt_id` (вид ответственного), `user_id`, `role_id`, `staff_id` (формула), `filluser_id`, `dep_grp_type_id`, `query_code`, `use_users_comps` |
| Поведение | `type_id` (вид задачи), `is_rq` обязательный, `can_change_user`, `can_empty_user_id`, `only_one_user`, `max_users`, `can_add`, `can_change_nn`, `cant_refuse_after`, `hide_from_initiator`, `hide_type_res` |
| Условие этапа | `use_ng_if` + `ng_if` (JS-выражение по полям записи) |
| Сроки | `set_due`, `due_d/due_h/due_m`, `is_date`, `noedit_due`, `use_365`, `is_planned` + `plan_d/plan_h/plan_m` |
| Уведомления | `disable_notify`, `is_cus_assign_text`, `cus_assign_subject`, `cus_assign_text` |
| Свои процессы | `has_custom` → строки в `task_approve_t_s_c` |
| Прочее | `title` (описание этапа), `recomm`, `responsible_txt` |

### 1.4 task_approve_t_s_roles

| Поле | Назначение |
|---|---|
| task_approve_t_s_id | Этап |
| role_id | Роль |
| send_by_role | 1 — задача назначается на роль (группу), а не на конкретных пользователей |

### 1.5 Атрибут `entities.task_approve_t_id`

Шаблон согласования по умолчанию для сущности. В onedoc заполнен для ~150 сущностей (например `its_d_statements` → 215 «Заявления», `its_memo` → 12, `its_in` → 17, `its_out` → 11). Один шаблон может использоваться несколькими сущностями (`its_prikaz` и `its_d_prikaz` → 16).

Если маршрут зависит от вида документа, шаблон хранится в справочнике вида (`its_d_statements_type.task_approve_t_id`), а `entities.task_approve_t_id` служит запасным.

---

## 2. Жизненный цикл заявки (на примере its_d_statements)

```
[Создать заявление] its_d_statements_wizard (entityTool)
   1. DMLI insert its_d_statements
   2. шаблон = its_d_statements_type.task_approve_t_id
   3. для каждого task_approve_t_s → insert tasks (managers по формуле)
   4. redirect → its_d_statementsdetails/<id>
        │
[Сохранение] its_d_statements_insert / its_d_statements_upsert (entityEvent)
   pkg/tasks.upsertEvent — пересчёт исполнителей этапов с rt=task_staff
        │
[Запуск] its_d_statements (workFlow, статус → stat_id)
   Старт (stat 1 «Создан»): pkg/tasks.managersToStep — проверка «Маршрута»
   Нумерация документа → doc_num, its_req.docnum
   Согласование = subprocess tasks_start_approve
        ├─ approved=1        → Создание замещения → E-mail → Успех (stat 3 «Закрыт»)
        ├─ res=rework        → сохранить результаты в *2_id, очистить → E-mail → На доработке (stat 12)
        └─ иначе             → E-mail об отказе → Отказано (stat 4 «Отклонен»)
        │
[Кнопки в задаче] task_approve_or_reject (по каждой задаче)
   Default Need Reason → (комментарий) → Согласовать: BPMSRunManualTask
   → task_hst_add (история)
```

### 2.1 Создание задач маршрута (wizard, скрипт «Создание документа»)

Логика, которую нужно повторить для новой сущности:

```lua
local entity_pk = EntityValueById("<entity>","sys$uuid",i)
-- шаблон
local tpl,errText,errNum = SqlQueryRow2([[select task_approve_t_id from <type_table> where id=? and task_approve_t_id is not null]],var.type_id)
-- этапы
local arr = SqlQueryRows([[select s.approve_type_id, s.use_users_comps, coalesce(s.due_d,0) as "_due_d", s.use_365,
    case when rt.code='task_staff' then st.formula end formula, s.use_ng_if, s.ng_if,
    s.title, s.step_nn, s.nn, s.id, s.type_id
  from task_approve_t_s s
  join task_approve_t_s_rt rt on rt.id=s.rt_id
  left join task_staff st on st.id=s.staff_id
  where s.task_approve_t_id = ?]], tpl.task_approve_t_id)
-- формула: StrReplace(:user_id / :entity_pk) → SqlQueryRow2 → managers = JsonToString(array({...}))
-- insert into tasks (created_by,title,entity_id,autoadd=0,step_nn,nn,entity_pk,approve_t_s_id,type_id,
--                    managers,manager_id,approve_type_id,use_ng_if,ng_if,due_at)
```

Сущностям без визарда маршрут предлагает сама detail-страница: REST `tasks_approves` при `isNew` или пустом `tasks` вызывает `pkg/edms_lua_tasks.loadApproveNew`, который строит черновик маршрута из `entities.task_approve_t_id` (строки с `id=0`, `_inserted=1`). Пользователь видит его на вкладке «Маршрут», а в `tasks` строки попадают при сохранении документа. Подробно — раздел 5.

Ключевые поля `tasks` для маршрута: `entity_id`, `entity_pk` (= `sys$uuid` записи), `approve_t_s_id`, `step_nn`, `nn`, `type_id`, `approve_type_id`, `managers` (JSON-массив пользователей из `Query("?code=users...")`), `is_rq`, `autoadd` (0 — строка маршрута, 1 — развернутая задача на исполнителя).

### 2.2 pkg/tasks

| Функция | Что делает |
|---|---|
| `upsertEvent(entity_code, id, user_id)` | Для этапов с `rt=task_staff` (с учётом `ng_if`) пересчитывает `tasks.managers` по формуле. Вызывается из события upsert |
| `managersToStep(entity_id, pk, user_id)` | Удаляет `autoadd=1`, перенумеровывает шаги, проверяет обязательные (`is_rq=1` без managers → «Не указан ответственный в разделе Маршрут»), удаляет необязательные пустые, раскладывает `managers` в отдельные задачи (`autoadd=1`); для роли без пользователей — одна задача на роль |

### 2.3 Статусы и its_req — bp_task_start_events id=2 (`set_status`)

Глобальное событие старта задачи БП (nn=200), срабатывает на каждой точке:

1. Берёт `bp_points.wf_status_attr_value` текущей точки. Если пусто — выходит.
2. По процессу получает `action_entity_id`, атрибут статуса (`wf_status_attr_id`) и переменную id записи (`wf_id_var_id`).
3. **Сущность не типа `its_req`** — `update <entity> set <status_attr> = <value> where id = var[<id_var>]`.
4. **Сущность типа `its_req`** (`entity_types.code='its_req'`):
   - если у записи пустой `req_id` — `insert into its_req (entity_id, created_by, entity_pk, entity_uuid, initial_req_uuid)` и `update <entity> set req_id`;
   - номер: если параметр `its_disable_docnum_auto` ≠ 1, `entities.docnum_after_approve` ≠ 1 и `its_req.docnum` пуст — `get_number_by_entity_prefix(entity_id, entities.docnum_prefix)`;
   - `its_req.title` собирается из краткого содержания (жёстко прописанный список сущностей);
   - `update its_req set <status_attr> = <value>`.

Если `docnum_after_approve = 1`, номер присваивается в `tasks_start_approve` после согласования.

Для its_d_statements статусы задаются на точках workflow:

| Точка | wf_status_attr_value | its_req_stat |
|---|---|---|
| Старт | 1 | draft — Создан |
| На доработке | 12 | rework — На доработке |
| Успех | 3 | closed — Закрыт |
| Отказано | 4 | refused — Отклонен |

Следствие: для новой сущности не надо писать код смены статусов и создания its_req — достаточно правильно настроить процесс (тип сущности, атрибут статуса, переменную id) и проставить `wf_status_attr_value` на точках.

---

## 3. Создание новой заявки с согласованием

Пример: `its_d_newreq`, модуль `its`. Работа через MCP onedoc.

### Шаг 1. Сущность

1. `create_entity` — тип `its_req` (иначе `set_status` не создаст its_req и не даст номер).
2. Обязательные атрибуты (`add_entity_attr`):

| code | тип | Назначение |
|---|---|---|
| title | varchar | Наименование |
| stat_id | reference → its_req_stat | Статус (атрибут workflow) |
| req_id | reference → its_req | Регистрационная карточка |
| doc_num | varchar | Номер (если нужен на самой записи) |
| created_by / created_at | системные | |
| type_id | reference → `<справочник видов>` | Только если маршрут зависит от вида |

3. Настроить сущность: `task_approve_t_id` (шаблон по умолчанию), при необходимости `docnum_prefix`, `docnum_after_approve`.

```sql
update entities set task_approve_t_id = <id шаблона>, docnum_prefix = 'НЗ' where code = 'its_d_newreq';
```

### Шаг 2. Шаблон согласования и этапы

```sql
insert into task_approve_t (code, title, can_add_task) values ('its_d_newreq', 'Новая заявка', 1) returning id;

insert into task_approve_t_s (task_approve_t_id, step_nn, nn, title, type_id, rt_id, staff_id, approve_type_id, is_rq)
values
 (<t_id>, 1, 1, 'Руководитель', (select id from task_types where code='approve'),
  (select id from task_approve_t_s_rt where code='task_staff'), <task_staff.id>,
  (select id from task_approve_type where code='serial'), 1),
 (<t_id>, 2, 1, 'Подписание', (select id from task_types where code='sign'),
  (select id from task_approve_t_s_rt where code='user'), null,
  (select id from task_approve_type where code='serial'), 1);
-- для этапа «Пользователь» дополнительно user_id
```

Этап по роли:

```sql
insert into task_approve_t_s (task_approve_t_id, step_nn, nn, title, type_id, rt_id, role_id, is_rq)
values (<t_id>, 3, 1, 'Исполнение', (select id from task_types where code='exec'),
        (select id from task_approve_t_s_rt where code='role'), <role_id>, 1) returning id;
insert into task_approve_t_s_roles (task_approve_t_s_id, role_id, send_by_role) values (<s_id>, <role_id>, 1);
```

Ограничение `db_execute_sql`: возвращается только первый результат, а `?` в литералах ломается — делать по одному оператору.

### Шаг 3. Создание записи с маршрутом (entityTool-визард)

Скопировать `its_d_statements_wizard` (`create_bp_process` по шаблону) и поменять:

- `DMLI("insert","its_d_newreq",...)`;
- источник шаблона: `select task_approve_t_id from entities where code='its_d_newreq'` (или из справочника вида);
- `redirect_url = "its_d_newreqdetails/"..i`.

Сохранить savepoint/rollback из оригинала: при ошибке вставки задач запись и задачи откатываются.

### Шаг 4. События сущности

`entityEvent` `its_d_newreq_upsert` с одной script-точкой:

```lua
local pkgTasks = dbrequire("pkg/tasks")
local errText,errNum = pkgTasks.upsertEvent("its_d_newreq",var.pk,sys.user_id)
if errNum~=0 then
    var.last_error = errText
    return
end
```

Нужно, только если есть этапы с формулой, зависящей от полей записи.

### Шаг 5. Workflow

Создать процесс `its_d_newreq` типа `workFlow` (копия `its_d_statements` или `task_approve_template` как основа):

1. `action_entity_id` = новая сущность, `wf_status_attr_id` = `stat_id`, `wf_id_var_id` = переменная `id`.
2. Переменные: `id`, `last_error`, `approved`, `approve_res_id`.
3. Точки:
   - **Старт** (script, `wf_status_attr_value=1`):
     ```lua
     local pkg_tasks = dbrequire("pkg/tasks")
     local errText,errNum = pkg_tasks.managersToStep(sys.action_entity_id, var.id, sys.user_id)
     if errNum~=0 then var.last_error = errText return end
     ```
   - шлюз `var.last_error==""` / `var.last_error~=""` → EndEvent «Ошибка»;
   - **Согласование** — subprocess `tasks_start_approve`;
   - шлюз `var.approved=="1"` → «Успех» (`wf_status_attr_value=3`);
   - шлюз `EntityValueById("task_approve_res","code",var.approve_res_id) == "rework"` → script сохранения результатов → «На доработке» (12);
   - иначе → «Отказано» (4);
   - при необходимости уведомления — subprocess `workflow_notify_close`.
4. Script для доработки (позволяет перезапустить маршрут):
   ```lua
   local uuid = EntityValueById(EntityValueById("entities","code",sys.action_entity_id),"sys$uuid",var.id)
   local errText,errNum = SqlExec2([[update tasks set approve_res2_id=approve_res_id, closed2_dt=closed_dt, status2_id=status_id where entity_pk=?]],uuid)
   if errNum~=0 then var.last_error = errText return end
   errText,errNum = SqlExec2([[update tasks set approve_res_id=null, closed_dt=null, status_id=null where entity_pk=?]],uuid)
   if errNum~=0 then var.last_error = errText return end
   ```

Нумерацию отдельной точкой (как «Нумерация документа» в its_d_statements) делать не нужно, если работает `set_status` + `docnum_prefix`; собственная точка нужна только при особом формате номера.

В скриптах БП не вызывать `Commit()/Rollback()`.

### Шаг 5а. Доступ к процессам — роль its_user

Каждому созданному процессу, который запускает пользователь, дать роль `its_user` с правом запуска (`bp_process_roles.can_run = 1`). Без этой записи у пользователя нет кнопки/инструмента, а запуск workflow завершается отказом в доступе.

| Тип процесса | Пример | Роль its_user |
|---|---|---|
| `workFlow` | `its_d_company_check`, `its_d_statements` | нужна |
| `entityTool` (визард создания) | `its_d_statements_wizard` | нужна |
| `entityActionByID` (кнопка на записи) | `its_bp_statements_link` | нужна, если кнопка для пользователей |
| `businessprocess`, вызываемый пользователем | `its_bp_statements_prikaz` | нужна |
| `entityEvent` (insert/upsert) | `its_d_company_check_insert` | не нужна — запускается платформой при сохранении |

Так настроено в onedoc: `its_d_company_check`, `its_d_statements`, `its_d_statements_wizard` имеют `its_user:1` (у последних двух ещё `admin:1`), события `*_insert`/`*_upsert` — без ролей.

Выдать доступ (повторный запуск не создаёт дубль):

```sql
insert into bp_process_roles (process_id, role_id, can_run)
select p.id, r.id, 1
from bp_processes p, roles r
where p.code = 'its_d_company_check' and r.code = 'its_user'
  and not exists (select 1 from bp_process_roles x where x.process_id = p.id and x.role_id = r.id);
```

Сразу для всех пользовательских процессов сущности:

```sql
insert into bp_process_roles (process_id, role_id, can_run)
select p.id, r.id, 1
from bp_processes p
join bp_process_types pt on pt.id = p.type_id
join entities e on e.id = p.action_entity_id
cross join roles r
where e.code = 'its_d_company_check' and r.code = 'its_user'
  and pt.code in ('workFlow','entityTool','entityActionByID','businessprocess')
  and not exists (select 1 from bp_process_roles x where x.process_id = p.id and x.role_id = r.id);
```

Проверка:

```sql
select p.code, pt.code ptype,
       (select string_agg(r.code||':'||coalesce(pr.can_run::text,''), ',')
          from bp_process_roles pr join roles r on r.id = pr.role_id where pr.process_id = p.id) roles
from bp_processes p
left join bp_process_types pt on pt.id = p.type_id
join entities e on e.id = p.action_entity_id
where e.code = 'its_d_company_check'
order by 1;
```

`can_run` — smallint: в конкатенации приводить к тексту (`can_run::text`), иначе запрос падает, а `db_execute_sql` после ошибки отвечает «текущая транзакция прервана» и на следующие запросы.

Права на саму сущность (чтение/изменение записей и `is_detail_view`, который проверяет `task_approve_or_reject`) настраиваются отдельно — в правах роли на сущность.

### Шаг 6. Кнопки согласования

Ничего создавать не нужно: задачи обрабатываются общим `task_approve_or_reject` (входы: `entity_code`, `pk_uuid`, `step_nn`, `approve_res_id`, `approve_t_s_c_id`, `reason`). Он:

- проверяет право `is_detail_view` на запись;
- находит открытую задачу пользователя (или его замещаемого по `users_alternate`) на шаге;
- запрашивает комментарий: `need_reason=1` — необязательный, `2` — обязательный (для `rework` и вида `exec`);
- выполняет `BPMSRunManualTask`, пишет историю `task_hst_add`, ставит номер при `set_docnum` / `its_r_nomen_docnum` / `deps_entities_counter`;
- поддерживает массовое согласование (`mass=1`, `mass_data`).

Специфичные для сущности ветки (как `its_decision`, `its_vacation`) добавлять в этот процесс не стоит — лучше реализовать в своём workflow.

### Шаг 7. Страница

Автостраница `<entity>details` уже показывает «Маршрут» по `tasks.entity_pk`. Свои страницы не называть кодами сущностей (`create_page` перезапишет автостраницу).

### Шаг 8. Проверка

1. Создать запись через визард → проверить `tasks` (`where entity_pk = <uuid> order by step_nn, nn`).
2. Запустить workflow → `stat_id=1`, создан `its_req`, присвоен `docnum`.
3. Пройти согласование всеми ролями, включая «На доработке» и «Отказать».
4. Проверить итоговые `stat_id` / `its_req.stat_id` и историю.

---

## 4. Редактирование этапов согласования

### 4.1 Где менять

| Что | Действие | Влияет на |
|---|---|---|
| Состав и порядок этапов | `task_approve_t_s` (step_nn, nn) | только новые записи |
| Исполнитель | rt_id + user_id / role_id / staff_id | новые записи; для формулы — также существующие при сохранении (upsertEvent) |
| Параллельно / последовательно | `approve_type_id` | новые записи |
| Вид задачи | `type_id` | новые записи |
| Обязательность | `is_rq` | проверка в `managersToStep` при старте |
| Условие этапа | `use_ng_if`, `ng_if` | новые записи; формулы — при upsert |
| Шаблон сущности целиком | `entities.task_approve_t_id` или справочник вида | новые записи |

Шаблон копируется в `tasks` в момент создания записи. Изменения шаблона не попадают в уже созданные записи, кроме пересчёта формул.

### 4.2 Типовые операции

Добавить этап между шагами 1 и 2:

```sql
update task_approve_t_s set step_nn = step_nn + 1 where task_approve_t_id = <t_id> and step_nn >= 2;
insert into task_approve_t_s (task_approve_t_id, step_nn, nn, title, type_id, rt_id, user_id, is_rq)
values (<t_id>, 2, 1, 'Юрист', (select id from task_types where code='approve'),
        (select id from task_approve_t_s_rt where code='user'), <user_id>, 1);
```

Параллельное согласование в одном шаге — несколько этапов с одним `step_nn` и разными `nn`, `approve_type_id = parallel`.

Сменить исполнителя:

```sql
update task_approve_t_s set rt_id = (select id from task_approve_t_s_rt where code='user'), user_id = <user_id>, staff_id = null
where id = <s_id>;
```

Сделать этап условным:

```sql
update task_approve_t_s set use_ng_if = 1, ng_if = 'detail.type_id == 5' where id = <s_id>;
```

Удалить этап: сначала проверить, что нет открытых задач.

```sql
select count(1) from tasks where approve_t_s_id = <s_id> and closed_dt is null;
delete from task_approve_t_s_roles where task_approve_t_s_id = <s_id>;
delete from task_approve_t_s where id = <s_id>;
```

Если на этап ссылаются старые задачи, лучше не удалять, а сделать необязательным (`is_rq=0`) или выключить условием `ng_if='false'`.

### 4.3 Перенос изменений в существующие записи

Для черновиков (workflow ещё не запускался):

```sql
-- посмотреть
select id, title, step_nn, nn, approve_t_s_id, managers from tasks where entity_pk = '<uuid>' order by step_nn, nn;
```

Точечно — обновить `tasks.managers` / `step_nn` вручную, либо удалить строки маршрута записи и пересоздать их логикой визарда (шаг 2.1) для этого `entity_pk`. Для записей на согласовании правка маршрута меняет текущий процесс — делать только через доработку.

### 4.4 Частые ошибки

- «Не указан ответственный в разделе Маршрут» — обязательный этап без `managers`: пустая формула, у пользователя нет руководителя, или `rt_id`/`staff_id` не заданы.
- Задачи не появились — у вида документа / сущности пустой `task_approve_t_id` (визард выдаст «Ошибка при поиске вида документа»).
- Нет its_req и номера — сущность не типа `its_req`, не задан `wf_status_attr_id` / `wf_id_var_id` у процесса, или на точке нет `wf_status_attr_value`.
- Номер не присваивается — включен параметр `its_disable_docnum_auto` или `docnum_after_approve=1` (номер появится после согласования).
- Массивы в `managers` — только через `JsonToString(array(...))`, иначе получится объект `{"1":…}`.

---

## 5. Вкладки details-страницы: «Маршрут» и «Ход исполнения»

Общая модель блоков (`pages_ui_blocks`, `pages_ui_blocks_n`, корневой блок, `genPages`) описана в `damubpm_detail_page_blocks_guide.md`. Здесь — только то, что касается вкладок согласования, на примере `its_d_statementsdetails` (pages.id=12265).

### 5.1 Как устроена вкладка

Вкладка — это всегда два блока и две связи:

```
TabView (tabview)                              pages_ui_blocks 2710  «tabs»
└─[block nn=300]→ Вкладка (tabview_tab)         pages_ui_blocks 2717  header_title «Маршрут»
                  └─[block nn=310]→ Содержимое  pages_ui_blocks 2718  ui_block_tpl = task_approve_task
```

| Уровень | Таблица | Ключевые поля |
|---|---|---|
| Контейнер вкладок | `pages_ui_blocks`, tpl `tabview` | один на страницу |
| Вкладка | `pages_ui_blocks`, tpl `tabview_tab` | `header_title` — текст ярлыка, `ng_if` — показ вкладки, `nn` |
| Ссылка TabView → вкладка | `pages_ui_blocks_n`, тип `block` | `pages_ui_blocks_id` = TabView, `block_id` = вкладка, `nn` — порядок ярлыков |
| Содержимое | `pages_ui_blocks`, tpl содержимого | `task_approve_task`, `task_approve_hst`, `panel_block`, `default_page` … |
| Ссылка вкладка → содержимое | `pages_ui_blocks_n`, тип `block` | `block_id` = блок содержимого |

Порядок ярлыков задаёт `pages_ui_blocks_n.nn` в TabView (в its_d_statements: 10 Главная, 200 Связанные документы, 300 Маршрут, 500 Ход исполнения, 600 Предпросмотр).

Шаблон `tabview_tab` рендерит `<p-tabPanel header="{{header_title}}" *ngIf="{{ng_if}}">`, внутри — поля (`attr`), табличные части и дочерние блоки (`block`) этой вкладки.

Коды блоков в onedoc — `<page_code>_<6 hex>`, элементов — `<код блока>_<число>`.

### 5.2 Вкладки its_d_statementsdetails

| nn | Вкладка (tabview_tab) | Содержимое (ui_block_tpl) | Примечание |
|---|---|---|---|
| 10 | Главная | `panel_block` / `default_page` | панели по видам с `ng_if` вида `detail['type_id$code'] == 'leave'` |
| 200 | Связанные документы | `default_page` → `its_link` | `details.its_link` |
| 300 | Маршрут | `task_approve_task` | см. 5.3 |
| 500 | Ход исполнения | `task_approve_hst` | см. 5.4 |
| 600 | Предпросмотр | `default_page` + html | скрыта: `ng_if = 1==0` |
| 700 | — | — | элемент 18834 с пустым `block_id` |

Найденная проблема: вкладка «Подписанные файлы» (блок 3034, tpl `tabview_tab`, внутри 3033 `task_eds_files`) ни к чему не привязана — элемент TabView с nn=700 (id 18834) имеет пустой `block_id`, так же как элемент 18835 во вкладке «Главная». Непривязанный блок генератор считает корневым и выводит вне TabView. Исправление:

```sql
update pages_ui_blocks_n set block_id = 3034 where id = 18834;
```

После правки перегенерировать страницу.

### 5.3 «Маршрут» — ui_block_tpl `task_approve_task`

Что показывает: маршрут согласования документа — шаги, вид задачи, исполнители, сроки; до запуска workflow позволяет его редактировать.

Источник данных — REST `tasks_approves` (метод `bindTasks()` шаблона страницы `detail`), параметры `{ isNew, entity_code, sys$uuid }`, результат в `this.infos`:

| Ключ | Что это |
|---|---|
| `infos.tasks` | строки маршрута. Нет задач → `loadApproveNew`: черновик из `entities.task_approve_t_id` → `task_approve_t_s` (id=0, `_inserted=1`, исполнители по формулам, сроки по `set_due`/`due_d`/`use_365`, этапы с ложным `ng_if` отброшены). Есть задачи → `loadApproveOld`: строки из `tasks` |
| `infos.task_approve_res` | кнопки результата для текущего пользователя |
| `infos.myTasks`, `myRoleTasks` | открытые задачи пользователя / его ролей |
| `infos.autostart_task(_approve_res)` | задача, которая запускается сразу после старта workflow |
| `infos.fyi_task_cnt`, `fyi_tasks` | задачи на ознакомление (в маршруте скрыты) |
| `details.route_map_tasks_all$` | краткая таблица «Статус / Вид / Инициатор / Даты / Исполнитель» |

Строки маршрута лежат в `form_task.values` (FormArray). Добавление/удаление/правка пишутся в `form_changes` с `table_name: 'tasks'` и сохраняются вместе с документом.

Что можно делать на вкладке и что это разрешает:

| Действие | Где в UI | Разрешено, если |
|---|---|---|
| Изменить исполнителя (`manager_id`) | поле исполнителя | `task_approve_t_s.can_change_user = 1` и `detail.readonly$ != 1` |
| Изменить срок (`due_at`) | дата | `noedit_due != 1` и `readonly$ != 1` |
| Изменить параллельно/последовательно | `approve_type_id` | `can_change_nn = 1` |
| Добавить согласующего на шаг | `addTask()` | `can_add = 1` и `readonly$ != 1`; новая строка — шаг +1, `is_rq=0`, без исполнителя |
| Удалить строку | `delTask()` | `is_rq != 1` и `readonly$ != 1` |
| Выполнить задачу (Согласовать/Отказать…) | `doTask()` → `bpRun(approve_res.process_code, …)` | текущий пользователь = `manager_id`, статус `opened` |
| Переназначить / Восстановить | `bpRun('task_reassign_manual' / 'task_restore_manual')` | роль процесса в `bp_process_roles_list` |
| Войти как исполнитель | `loginAs()` → `users_sign` | роль `users_sign` |

Строка скрыта, если `ng_if` этапа ложен, вид `fyi` или `hide_from_initiator` (`_hide = 1`). Подсказка под строкой — `task_approve_t_s.recomm`.

`doTask()` передаёт в процесс результата: `approve_t_s_c_id`, `approve_res_id`, `step_nn`, `task_id`, `entity_code`, `pk_uuid`, `approve_t_s_id` — это ровно входы `task_approve_or_reject`.

### 5.4 «Ход исполнения» — ui_block_tpl `task_approve_hst`

Что показывает: фактическое исполнение — кто, когда и с каким результатом закрыл задачу, комментарии, файлы, замещения, версии документа. Только чтение (кроме админ-действий).

Источник — тот же `tasks_approves`, ключ `infos.executionProgress`, дерево строит `initExecutionProgress()` → `executionProgressItems` → `p-treeTable`.

| Колонка | Откуда |
|---|---|
| Исполнитель | `users.title` по `tasks.manager_id`, должность; «и.о.» из `users_alternate` на дату задачи |
| Дата | `tasks.created_at`, прочитано — `read_at` |
| Тип задачи | `task_types.title` |
| Статус задачи | `task_approve_res.do_hst_title` (или `approve_res2_id` после доработки), иначе `task_statuses.title` + цвет |
| Действия | `task_hst` по задаче: `action_txt`, `txt` (комментарий), `files`, `file_id` |

Отбор: задачи записи (`entity_pk` или `version_entity_pk`), не групповые (`is_group=0`), с заданным `status_id` или `status2_id` — то есть только запущенные. До старта workflow вкладка пустая. Вложенность — по `parent_id`; версии документа (`its_new_version`) выводятся отдельными узлами «Версия N» и «Текущая версия».

Админ-кнопки строки: `task_set_responsible`, `task_reassign_manual`, `task_restore_manual`, `task_delete_manual` (по ролям и `session_roles.admin`).

Разница двух вкладок кратко: «Маршрут» — план (что будет и кто согласует, редактируется до запуска), «Ход исполнения» — факт (что уже сделано, история `task_hst`).

### 5.5 Добавить вкладки на новую details-страницу

Через MCP onedoc (`add_page_ui_blocks`, затем `add_pages_ui_blocks_n`). Предполагается, что TabView уже есть (код `<page>_tabs`), иначе создать и его (`ui_block_tpl_code: tabview`) и привязать к корневому блоку.

1. Блоки:

```json
{
  "page_code": "its_d_newreqdetails",
  "pages_ui_blocks": [
    { "code": "its_d_newreqdetails_route_tab", "title": "Маршрут вкладка", "header_title": "Маршрут", "ui_block_tpl_code": "tabview_tab", "nn": "300" },
    { "code": "its_d_newreqdetails_route",     "title": "Маршрут",         "header_title": "Маршрут", "ui_block_tpl_code": "task_approve_task", "nn": "310" },
    { "code": "its_d_newreqdetails_hst_tab",   "title": "Ход исполнения вкладка", "header_title": "Ход исполнения", "ui_block_tpl_code": "tabview_tab", "nn": "500" },
    { "code": "its_d_newreqdetails_hst",       "title": "Ход исполнения", "header_title": "Ход исполнения", "ui_block_tpl_code": "task_approve_hst", "nn": "510" }
  ]
}
```

2. Связи (все типа `block`):

```json
{
  "page_code": "its_d_newreqdetails",
  "pages_ui_blocks_n": [
    { "code": "its_d_newreqdetails_tabs_route", "pages_ui_blocks_code": "its_d_newreqdetails_tabs",      "pages_ui_blocks_n_t_code": "block", "block_code": "its_d_newreqdetails_route_tab", "nn": "300" },
    { "code": "its_d_newreqdetails_route_in",   "pages_ui_blocks_code": "its_d_newreqdetails_route_tab", "pages_ui_blocks_n_t_code": "block", "block_code": "its_d_newreqdetails_route",     "nn": "310" },
    { "code": "its_d_newreqdetails_tabs_hst",   "pages_ui_blocks_code": "its_d_newreqdetails_tabs",      "pages_ui_blocks_n_t_code": "block", "block_code": "its_d_newreqdetails_hst_tab",   "nn": "500" },
    { "code": "its_d_newreqdetails_hst_in",     "pages_ui_blocks_code": "its_d_newreqdetails_hst_tab",   "pages_ui_blocks_n_t_code": "block", "block_code": "its_d_newreqdetails_hst",       "nn": "510" }
  ]
}
```

3. Перегенерировать страницу (`genPages(entity_id, user_id)`, см. guide по блокам) и проверить дерево запросом:

```sql
select b.id, ubt.code tpl, b.header_title, n.id nid, nt.code et, n.nn, n.block_id
from pages p
join pages_ui_blocks b on b.pages_id = p.id
left join ui_block_tpl ubt on ubt.id = b.ui_block_tpl_id
left join pages_ui_blocks_n n on n.pages_ui_blocks_id = b.id
left join pages_ui_blocks_n_t nt on nt.id = n.t_id
where p.code = 'its_d_newreqdetails' and coalesce(nt.code,'') <> 'attr'
order by b.id, n.nn;
```

Проверки: у всех элементов `et='block'` заполнен `block_id`; каждый `tabview_tab` привязан к TabView; блоки содержимого `task_approve_*` привязаны к своей вкладке, а не к TabView напрямую.

Другие готовые блоки для вкладок документа: `task_eds_files` (Подписанные файлы), `its_link` (Связанные документы), `log_rows` (История), `global_msg` (Обсуждение и файлы), `files_for_preview` (Предпросмотр), `changes` (Изменение документа).

### 5.6 Условия и частые ошибки

- Скрыть вкладку по полю записи — `ng_if` у блока `tabview_tab` (например `detail['type_id$code'] == 'leave'`); скрыть совсем — `1==0`, как у «Предпросмотр».
- Вкладка «Маршрут» пустая на новой записи — у сущности нет `entities.task_approve_t_id` и записи ещё не создан маршрут визардом.
- «Ход исполнения» пустой — workflow не запускался (у задач нет `status_id`), это нормально.
- Блок оказался над вкладками или внизу страницы — у связи `block` пустой `block_id`, блок стал корневым.
- Маршрут нельзя править — документ в `readonly$` (после старта workflow) или у этапа не включены `can_change_user` / `can_add`.
- Правку вносить в `pages_ui_blocks`/`pages_ui_blocks_n`, а не в `pages.angular_template`: при генерации он перезаписывается. Шаблоны `task_approve_task`/`task_approve_hst` общие для всех страниц — менять их только осознанно; для своей разметки — блок с `is_auto != 1` и собственным `angular_template`.
