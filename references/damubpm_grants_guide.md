# Проверка полномочий на сущности в DamuBPM (entity_grants / entity_view_limits)

Справка для ИИ: как платформа решает, может ли пользователь видеть список, открыть карточку, создать, изменить или удалить запись сущности. Основано на Go-коде `CheckGrantOfEntity`, `CheckFullGrantOfEntity`, `QueryGetViewQueryLimit` и данных стенда contact-center (19.10.2026).

## 1. Модель данных

```
users ──< user_roles >── roles ──< entity_grants >── entities
                                        │
                                        └── view_limit_id → entity_view_limits (subquery, SQL-условие на main)
entity_grants ──< entity_grants_edit_attr >── entity_attrs   (список атрибутов для edit_only_attr)
roles ──< role_params >── params        (is_admin)
users ──< user_params >── params        (readonly_mode)
```

### entity_grants — «роль × сущность × ограничение × действия»
| Колонка | Смысл |
|---|---|
| `role_id` → roles | кому выдано |
| `entity_id` → entities | на какую сущность |
| `view_limit_id` → entity_view_limits | **на какие строки** (обязательно; JOIN по нему, без него грант не работает) |
| `is_list_view` | видеть в списках (query / таблицы) |
| `is_detail_view` | открывать карточку записи |
| `is_insert` | создавать |
| `is_update` | изменять |
| `is_delete` | удалять |
| `edit_only_attr` | грант разрешает менять **только** атрибуты из `entity_grants_edit_attr` |
| `title` | подпись |

Уникальность на практике — `(view_limit_id, entity_id, role_id)` (так делает Upsert в БП `role_grant_all_view_access_mod`). У одной роли на одну сущность может быть несколько грантов с разными ограничениями (на стенде 294 такие пары).

### entity_view_limits — «какие строки»
| Колонка | Смысл |
|---|---|
| `entity_id` → entities | сущность |
| `code`, `title` | код и название («Мои», «Моего подразделения»…) |
| `subquery` | SQL-условие, подставляется в `WHERE`; таблица сущности доступна под алиасом **`main`** |
| `is_auto` | 1 — создано платформой при создании сущности: `code = код сущности`, `subquery = '1 = 1'` (все строки) |
| `for_filter` | ограничение используется как пользовательский фильтр списка (встречается редко) |
| `warning_text` | ошибка проверки SQL, пишет entityEvent-БП `entity_view_limits_upsert` |

Плейсхолдеры в `subquery`:
- `:user_id` — id текущего пользователя (заменяется строкой во всех трёх функциях).
- `:lang` — язык, заменяется на `'ru'`/`'kk'`/… **только** в `QueryGetViewQueryLimit` (списки). В проверках записи (`CheckGrantOfEntity`) `:lang` не заменяется — не использовать его в ограничениях, по которым проверяется карточка/изменение.
- `:detail_id`, `:detail_query_id` — учитывает только валидатор `entity_view_limits_upsert` (подставляет 0). Go-функции ниже их не заменяют.

### Параметры
- `role_params` с `params.code = 'is_admin'` и значением `1` — роль-администратор (на стенде одна: `admin`). Администратор обходит ограничения (см. ниже).
- `user_params` с `params.code = 'readonly_mode'` = `1` — пользователь только для чтения.

## 2. Алгоритм проверки одной записи

`CheckGrantOfEntity(o, userId, entityCode, grant, pk, skipAdmin) error`

1. **Белый список прав**: `grant` ∈ `is_list_view | is_detail_view | is_update | is_delete | is_insert` (+ `edit_only_attr`, если включён `FeatureGrantEditAttr`). Иначе ошибка `Unknown grant type`. Имя права вставляется в SQL как имя колонки, поэтому список закрыт — это защита от SQL-инъекции.
2. **Админ**: если `skipAdmin` и у одной из ролей пользователя `is_admin = 1` → доступ без дальнейших проверок.
3. **Только чтение**: для `is_update | is_delete | is_insert` при `readonly_mode = 1` → `E-BPM-00030. Readonly Mode`. Проверка идёт **после** админа: админ со `skipAdmin` её не получает.
4. **Сбор ограничений**:
   ```sql
   select evl.subquery
   from entity_grants eg, entities e, entity_view_limits evl, user_roles ur
   where eg.<grant> = 1 and e.code = :entityCode and e.id = eg.entity_id
     and eg.view_limit_id = evl.id and ur.role_id = eg.role_id and ur.user_id = :userId
   ```
   Пусто → `Access Denied` (ни одна роль не даёт это право).
5. **Объединение через OR**: `( sq1 or sq2 or … )`, `:user_id` → число. Права ролей **складываются**: достаточно, чтобы запись подходила под любое ограничение любой роли с этим правом.
6. **Проверка строки**:
   ```sql
   select 1 from <entityCode> main where (main.id = :pk) and ( (sq1 or sq2 …) )
   ```
   Нет строки (или ошибка SQL в subquery) → `Access Denied … Error <текст>`. Есть → `nil`.

Важные следствия:
- Проверяется по `main.id` (bigint). Нужен числовой id, не `sys$uuid`.
- Для `is_insert` проверяется уже существующая строка `pk` — то есть «может ли эта запись существовать у пользователя». Для проверки до вставки этой функции недостаточно.
- Ошибка SQL в любом `subquery` ломает проверку целиком для всех ролей пользователя (OR склеивается в один запрос). Смотреть `entity_view_limits.warning_text`.
- Подзапросы `subquery` подставляются как текст; их пишут администраторы, пользовательский ввод туда не попадает. Значения из запроса в `subquery` не клеить.

### CheckFullGrantOfEntity — отличие
То же самое, но в шаге 4 добавлено `coalesce(eg.edit_only_attr,0) = 0`: гранты «только отдельные атрибуты» не учитываются. Использовать, когда нужна полная правка записи (сохранение всей формы, удаление). `CheckGrantOfEntity` с `is_update` вернёт успех и пользователю с `edit_only_attr`-грантом — тогда разрешённые поля берутся из `entity_grants_edit_attr`.

## 3. Ограничение списков

`QueryGetViewQueryLimit(o, entityId, queryId, userId, lang) string` — возвращает кусок SQL, который платформа дописывает к query списка.

1. Админ (`is_admin = 1`) → `""` (без ограничений; параметра skipAdmin нет).
2. Query с `queries.is_public = 1` → `""` (ограничения сущности не применяются — публичный query видит всё, что в нём выбрано).
3. Ограничения всех ролей пользователя с `is_list_view = 1` на `entity_id`.
4. Есть → `and ( (sq1) or (sq2) … ) ` с заменой `:user_id` и `:lang`.
5. Нет ни одного → ` and 1 = 0` (пустой список, не ошибка).

Поэтому: у роли нет `is_list_view` на сущность — список пустой, даже если страница и меню доступны.

## 4. Как это использовать в Lua

Lua-обёртки (без `skipAdmin` и без `o`):
```lua
local errText, errNum = CheckGrantOfEntity(user_id, "entity_code", "is_detail_view", id)
if errNum ~= 0 then
  output.error_code = 403
  output.error_text = "Нет доступа"
  return
end
local errText, errNum = CheckFullGrantOfEntity(user_id, "entity_code", "is_update", id)
```
- Порядок результатов: текст, код (`errNum ~= 0` — отказ).
- В REST брать `request.user_id`, в БП — `sys.user_id`. Реальные примеры: REST `auditinfo`, `its_req_reassign`.
- Пропускает ли Lua-обёртка админа (skipAdmin=true внутри) — на стенде не проверено. Если важно, проверять отдельно под admin и под обычным пользователем.
- REST с `only_auth`, роли REST и `bp_process_roles` — **другой** уровень доступа (кто может вызвать сервис/процесс). Они не проверяют строки сущности. Если REST читает или меняет запись по id из запроса — вызывать `CheckGrantOfEntity` самому.
- SQL в Lua (`SqlQueryRows`, `SqlExec2`) ограничения не применяет вообще. Чтение «как пользователь» — либо свой WHERE, либо проверка гранта по каждой записи.

## 5. Как выдать доступ (рецепты)

Найти автоограничение «все строки» сущности и выдать роли список + карточку:
```sql
insert into entity_grants (role_id, entity_id, view_limit_id, is_list_view, is_detail_view, is_insert, is_update, is_delete, title)
select r.id, e.id, v.id, 1, 1, 0, 0, 0, e.code
from roles r, entities e
join entity_view_limits v on v.entity_id = e.id and v.is_auto = 1
where r.code = 'map_manager' and e.code = 'map_orgs'
  and not exists (select 1 from entity_grants g where g.role_id = r.id and g.entity_id = e.id and g.view_limit_id = v.id);
```
Своё ограничение «только мои записи»:
```sql
insert into entity_view_limits (entity_id, code, title, subquery, is_auto)
select id, 'cln_orders_my', 'Мои заказы', 'main.created_by = :user_id', 0
from entities where code = 'cln_orders';
```
и грант с `view_limit_id` этого ограничения. После вставки — проверить `warning_text` и вызов под не-admin пользователем.

Типовые `subquery` на стенде:
| Назначение | subquery |
|---|---|
| все строки (автоматическое) | `1 = 1` |
| созданные мной | `main.created_by = :user_id` |
| мои по полю | `main.user_id = :user_id` |
| моего подразделения | `main.dep_id = (select dep_id from users where id=:user_id)` |
| где я исполнитель задачи (с замещением) | `exists (select 1 from tasks where entity_pk = main.sys$uuid and (manager_id=:user_id or manager_id in (select ua.users_id from users_alternate ua where ua.user_id = :user_id)))` |
| где я участвовал в ходе исполнения | `exists (select 1 from task_hst where entity_pk = main.sys$uuid and created_by=:user_id)` |
| моей организации | `main.company_id = (select company_id from users where id=:user_id)` |
| только активные | `main.is_active = 1` |

Служебные БП (модуль core):
- `role_grant_all_view_access_mod` (entityAction на roles) — выдаёт роли `is_list_view + is_detail_view` на все справочники (`entity_types.code='reference'`) модуля роли через их ограничение `1 = 1`.
- `entity_view_limits_upsert` (entityEvent) — подставляет `:user_id`, `:detail_id`, `:detail_query_id` = 0, проверяет SQL через `CheckQuery`, ошибку пишет в `warning_text`.
- `roles_copy_grants` «Скопировать доступы в другую роль» — несмотря на название, копирует только `role_menus` (пункты меню), не `entity_grants`.

## 6. Частые ошибки

1. Создали сущность, страницы и меню, но не выдали `entity_grants` — у обычных пользователей пустые списки (` and 1 = 0`) и «Access Denied» на карточке; у admin всё работает, поэтому ошибку не видно. Проверять под не-admin.
2. Грант без `view_limit_id` не работает (JOIN). У 45 сущностей на стенде нет автоограничения — для них сначала создать `entity_view_limits` с `1 = 1`.
3. В `subquery` писать условия на алиас `main`. Без `main.` колонка может разрешиться в другую таблицу подзапроса.
4. Ограничения разных ролей объединяются через OR — «закрыть» доступ ролью нельзя, можно только не выдавать.
5. Для карточки нужно `is_detail_view`, для списка — `is_list_view`; это разные флаги.
6. `queries.is_public = 1` снимает ограничения сущности в списке — не делать публичными query по чувствительным сущностям.
7. `readonly_mode` проверяется только в функциях записи; списки и карточку он не ограничивает.
