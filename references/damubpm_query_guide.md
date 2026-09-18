# Query в DamuBPM: SQL, параметры, фильтры и 60 примеров

[← Главная](index.html)

Материал составлен по серверному обработчику Query и штатной CRUD/JIT-логике DamuBPM через `Models.QueryOptions` и `dbQueryService.getQuery()`.

> **Важно:** Важно.
Алиасы фильтров должны существовать в конкретном query и его метаданных. В примерах
title
,
status_id
,
created_at
,
amount
и другие имена используются как демонстрационные.

## 1. Как работает Query

```text
Query code
  ↓
запись из queries
  ↓
SQL + entity_id + need_filter + is_public + extdb
  ↓
QueryFilterBuild(...)
  ↓
%filter% / %order% / form-параметры
  ↓
ReplaceDbTypeSql2(...)
  ↓
COUNT + LIMIT/OFFSET при пагинации
  ↓
items + allCount + pageCount + metadata
```

Обычный Query Использует подключение default .

External DB Query При наличии extdb_id сервер переключает ORM на внешнюю БД.

Public Query При is_public = 1 обязательная проверка авторизации не выполняется.

Protected Query Для остальных query выполняется проверка текущего пользователя.

## 2. Серверная логика

| Шаг | Что происходит |
| --- | --- |
| 1 | Читаются perpage , page , code . |
| 2 | Код query валидируется. |
| 3 | Из queries загружаются is_auto , id , need_filter , entity_id , sql_text , title , is_public , настройки External DB. |
| 4 | Для непубличного query проверяется авторизация. |
| 5 | При необходимости подключается External DB. |
| 6 | QueryFilterBuild формирует SQL, count SQL, значения фильтров и form-параметров. |
| 7 | При need_filter = 1 пустой запрос блокируется и возвращается error = 1 . |
| 8 | :domain заменяется текущим HTTP Host. |
| 9 | При perpage = 0 запрос выполняется без пагинации. |
| 10 | При пагинации вычисляется общее количество записей. |
| 11 | PostgreSQL/MySQL/ClickHouse используют limit ? offset ? , Oracle — OFFSET ? ROWS FETCH NEXT ? ROWS ONLY . |
| 12 | Колонка total_count$ может использоваться как источник allCount . |

### Типовой SQL query

```javascript
select
    main.id,
    main.title,
    main.created_at
from public.my_entity main
%filter%
%order%
```

В реальных query также встречаются `:user_id`, `:lang`, `:domain`, `total_count$`, а также коды с суффиксами `$all`, `$archive`, `_select`.

## 3. QueryOptions на frontend

```javascript
const { QueryOptions } = Models;
const { first } = rxjs;

let q = new QueryOptions('ai_prompt');
q.page = 1;
q.perpage = 15;

this.dbQueryService.getQuery(q)
    .pipe(first())
    .subscribe(resp => {
        console.log(resp.items);
    });
```

| Параметр | Назначение | Пример |
| --- | --- | --- |
| code | Код query | new QueryOptions('ai_prompt') |
| page | Номер страницы | q.page = 2 |
| perpage | Размер страницы; 0 = без пагинации | q.perpage = 50 |
| flt | Строковый поиск | q.flt.title = 'Test' |
| flteq | Точное равенство | q.flteq.status_id = '1' |
| fltin | Список значений | q.fltin.status_id = ['1','2'] |
| func22 | Диапазоны, даты, специальные условия | q.func22.amount = '100\|500' |
| param1 | Первый параметр конкретного query | q.param1 = '3352' |
| param2 | Второй параметр | q.param2 = 'active' |
| orderBy | Алиас сортировки | q.orderBy = 'title' |
| orderAsc | 0 = ASC, 1 = DESC | q.orderAsc = 0 |

## 4. Виды фильтрации

### 4.1 Строковый поиск — flt

```javascript
q.flt.title = 'договор';
```

### 4.2 Точное значение — flteq

```javascript
q.flteq.status_id = '10';
q.flteq.is_active = true;
```

### 4.3 Список значений — fltin

```javascript
q.fltin.status_id = ['1', '2', '5'];
```

### 4.4 Диапазоны — func22

```javascript
q.func22.amount = '100|500';
```

### 4.5 Комбинирование

```javascript
q.flt.title = 'счёт';
q.flteq.is_active = true;
q.fltin.status_id = ['10', '20'];
q.func22.amount = '100000|500000';
```

> **Примечание:** Подбирайте тип фильтра по типу поля:
строки —
flt
, boolean/точное значение —
flteq
, reference/multi-select —
fltin
, range/date —
func22
.

## 5. Специальные параметры backend

| Параметр | Поведение |
| --- | --- |
| getTitleById | Читает title записи основной entity по ID и возвращает getSelectedTitle . |
| getRowById | Использует исходный SQL, подменяет %filter% на where main.id=? , очищает %order% , подставляет пользователя/язык и возвращает getSelectedRow . |
| format=csv | Ответ отдаётся как tab-separated CSV. |
| ids | Учитывается защитой need_filter ; дальнейшую обработку выполняет QueryFilterBuild . |
| :domain | SQL placeholder, заменяется текущим host. |

## 6. Ответ сервера

```javascript
{
  "isMobile": false,
  "lang": "",
  "entityCode": "my_entity",
  "pageCount": 4,
  "allCount": 57,
  "title": "Мой запрос",
  "error": "0",
  "getSelectedTitle": "",
  "getSelectedRow": null,
  "items": [],
  "needFilter": 0,
  "license": "..."
}
```

## 7. Как писать SQL для Query в DamuBPM

В `queries.sql_text` хранится SQL-шаблон. DamuBPM не просто выполняет его как строку: сервер строит фильтры, передает параметры, подставляет служебные значения, считает общее количество и добавляет пагинацию.

> **Ключевое правило:** основную таблицу алиасьте как `main`. Обработчик `getRowById` сам заменяет `%filter%` на `where main.id=?`, поэтому другой алиас сломает этот режим.

### 7.1 Базовый шаблон

```sql
select
    main.id,
    main.title
from public.my_entity main
%filter%
%order%
```

### 7.2 Служебные конструкции

| Конструкция | Назначение |
| --- | --- |
| `main` | Рекомендуемый алиас основной сущности. Нужен, в частности, для штатного `getRowById`. |
| `%filter%` | Место, куда `QueryFilterBuild` вставляет динамические условия из `flt`, `flteq`, `fltin`, `func22` и других метаданных query. |
| `%order%` | Место для динамической сортировки `orderBy/orderAsc`. |
| `:user_id` | ID текущего пользователя. Используется в реальных DamuBPM query, например для признака прочтения. |
| `:domain` | В серверном обработчике заменяется на текущий HTTP Host. |
| `:lang` | Язык запроса. В ветке `getRowById` сервер явно подставляет текущее значение языка. |
| `total_count$` | Если колонка присутствует, сервер может взять из нее `allCount`. Типовой вариант: `count(*) over() as total_count$`. |
| `"sys$uuid"` | Системный UUID записи, часто возвращается auto-query. |
| `"field_id$"` | Типовой алиас отображаемого значения reference. |
| `"field_id$code"` | Дополнительный атрибут связанной reference-записи. |

### 7.3 Где ставить постоянные условия

В реальных auto-query DamuBPM используется паттерн, где `%filter%` стоит до постоянных ограничений:

```sql
from public.my_entity main
%filter%
    and coalesce(main.in$trash, 0) = 0
%order%
```

Такой стиль позволяет `QueryFilterBuild` сформировать базовый блок фильтрации, после чего query добавляет собственные ограничения.

### 7.4 Пагинация: LIMIT писать не нужно

Для обычного Query не добавляйте свой `limit/offset`: сервер делает это сам по `page/perpage`. PostgreSQL и MySQL получают `limit ? offset ?`, Oracle — `OFFSET ? ROWS FETCH NEXT ? ROWS ONLY`, ClickHouse — свой вариант `limit/offset`. При `perpage=0` сервер выполняет SQL без пагинации.

### 7.5 COUNT и total_count$

Для auto-query сервер умеет считать `allCount` отдельным count SQL. Если запрос уже возвращает `total_count$`, можно использовать оконную функцию:

```sql
select
    count(*) over() as total_count$,
    main.id,
    main.title
from public.my_entity main
%filter%
%order%
```

### 7.6 Reference-поля

Практичный паттерн — оставить ID в исходном поле и отдельно вернуть отображаемое значение:

```sql
select
    main.status_id,
    status_id.title as "status_id$",
    status_id.code as "status_id$code"
from public.my_entity main
left join statuses status_id on status_id.id = main.status_id
%filter%
%order%
```

### 7.7 Параметры param1/param2

> **Не придумывайте привязку параметров.** Из обработчика видно, что `QueryFilterBuild` формирует `formArray`, который передается в prepared query. Но точное соответствие `param1/param2` SQL-placeholder'ам задается логикой/метаданными конкретной версии `QueryFilterBuild`. Для существующего query сохраняйте уже используемый порядок placeholder'ов.

### 7.8 PostgreSQL и External DB

Основная DamuBPM БД в этой инструкции — PostgreSQL. Для query на External DB учитывайте его `dbtype`: PostgreSQL-специфичные функции вроде `to_char`, `date_trunc`, JSONB и `distinct on` могут не работать в Oracle/MySQL/ClickHouse.

### 7.9 Что обязательно проверить

1. Основная таблица имеет алиас `main`.
2. Есть `%filter%`, если query должен поддерживать штатные фильтры.
3. Есть `%order%`, если нужна динамическая сортировка.
4. Вы не добавили ручной `LIMIT/OFFSET` без необходимости.
5. Алиасы выбранных полей совпадают с метаданными фильтрации/таблицы.
6. Reference-поля возвращают ID и нужные отображаемые алиасы.
7. Для больших выборок используется пагинация или `total_count$`.
8. PostgreSQL-специфичный SQL не переносится вслепую на External DB другого типа.

## 7.10 30 примеров SQL для DamuBPM Query

### SQL 1. Минимальный entity query

Базовый шаблон с обязательным алиасом main и местами для динамического фильтра и сортировки.

```sql
select
    main.id,
    main.title
from public.my_entity main
%filter%
%order%
```

### SQL 2. Query с sys$uuid

Типовой DamuBPM query часто возвращает системный UUID вместе с id.

```sql
select
    main."sys$uuid" as "sys$uuid",
    main.id,
    main.title
from public.my_entity main
%filter%
%order%
```

### SQL 3. Query с total_count$

Позволяет серверу взять allCount из первой строки результата и не выполнять отдельный COUNT.

```sql
select
    count(*) over() as total_count$,
    main."sys$uuid" as "sys$uuid",
    main.id,
    main.title
from public.my_entity main
%filter%
%order%
```

### SQL 4. Select-query для справочника

Для prime-select/справочника удобно возвращать id и name. Поле two встречается в штатных select-query DamuBPM.

```sql
select
    main.title as name,
    main."sys$uuid" as "sys$uuid",
    main.id,
    2 as two
from public.my_entity main
%filter%
%order%
```

### SQL 5. Исключение записей в корзине

Типовой auto-query DamuBPM размещает постоянное условие после %filter%.

```sql
select
    count(*) over() as total_count$,
    main.id,
    main.title
from public.my_entity main
%filter%
    and coalesce(main.in$trash, 0) = 0
%order%
```

### SQL 6. Только записи из корзины

Отдельный query, например с кодом my_entity$archive, может выбирать архив/корзину.

```sql
select
    count(*) over() as total_count$,
    main.id,
    main.title,
    main.updated_at
from public.my_entity main
%filter%
    and coalesce(main.in$trash, 0) = 1
%order%
```

### SQL 7. Reference через LEFT JOIN

Возвращаем ID ссылки и отображаемое название отдельным DamuBPM-алиасом.

```sql
select
    main.id,
    main.status_id,
    status_id.title as "status_id$"
from public.my_entity main
left join statuses status_id on status_id.id = main.status_id
%filter%
%order%
```

### SQL 8. Reference с дополнительными атрибутами

Можно отдавать код и другие свойства reference-записи через алиасы с $.

```sql
select
    main.id,
    main.status_id,
    status_id.title as "status_id$",
    status_id.code as "status_id$code",
    status_id.start_workflow as "status_id$start_workflow"
from public.my_entity main
left join statuses status_id on status_id.id = main.status_id
%filter%
%order%
```

### SQL 9. Пользователь created_by

Типовой join к users для отображения автора и связанных атрибутов.

```sql
select
    main.id,
    main.created_by,
    created_by.title as "created_by$",
    created_by.email as "created_by$email",
    created_by.login as "created_by$login"
from public.my_entity main
left join users created_by on created_by.id = main.created_by
%filter%
%order%
```

### SQL 10. Признак прочтения текущим пользователем

Реальный DamuBPM-паттерн с :user_id и таблицей прочтений.

```sql
select
    main.id,
    main.title,
    (
        select rr.read_at
        from read_rows rr
        where rr.entity_pk = main."sys$uuid"
          and rr.user_id = :user_id
        limit 1
    ) as "read_at$"
from public.my_entity main
%filter%
%order%
```

### SQL 11. Только записи текущего пользователя

Условие по :user_id добавлено как постоянная часть query.

```sql
select
    main.id,
    main.title,
    main.created_by
from public.my_entity main
%filter%
    and main.created_by = :user_id
%order%
```

### SQL 12. Фильтрация по текущему host через :domain

В QueryRestApiGet :domain заменяется на текущий HTTP Host.

```sql
select
    main.id,
    main.title,
    main.host
from public.my_entity main
%filter%
    and main.host = :domain
%order%
```

### SQL 13. COALESCE для nullable значений

Удобно нормализовать NULL прямо в результирующем наборе.

```sql
select
    main.id,
    main.title,
    coalesce(main.amount, 0) as amount,
    coalesce(main.comment, '') as comment
from public.my_entity main
%filter%
%order%
```

### SQL 14. CASE для вычисляемого статуса

Вычисляемые колонки можно фильтровать/сортировать только если соответствующая мета-логика query это допускает.

```sql
select
    main.id,
    main.title,
    case
        when main.is_active = 1 then 'Активен'
        else 'Неактивен'
    end as state_title
from public.my_entity main
%filter%
%order%
```

### SQL 15. Склейка нескольких полей

Формируем удобное отображаемое поле для таблицы или select-query.

```sql
select
    main.id,
    concat(main.code, ' — ', main.title) as display_name
from public.my_entity main
%filter%
%order%
```

### SQL 16. Форматирование даты PostgreSQL

Подходит для основной PostgreSQL БД. Для External DB учитывайте диалект.

```sql
select
    main.id,
    main.title,
    to_char(main.created_at, 'YYYY-MM-DD HH24:MI:SS') as created_at
from public.my_entity main
%filter%
%order%
```

### SQL 17. Год и месяц через EXTRACT

Полезно для аналитических колонок и группировок.

```sql
select
    main.id,
    main.title,
    extract(year from main.created_at) as created_year,
    extract(month from main.created_at) as created_month
from public.my_entity main
%filter%
%order%
```

### SQL 18. Группировка по дню

PostgreSQL date_trunc удобно применять в отчетных query.

```sql
select
    date_trunc('day', main.created_at) as day,
    count(*) as cnt
from public.my_entity main
%filter%
group by date_trunc('day', main.created_at)
%order%
```

### SQL 19. COUNT по reference

Пример агрегатного query по статусам.

```sql
select
    main.status_id,
    count(*) as cnt
from public.my_entity main
%filter%
group by main.status_id
%order%
```

### SQL 20. SUM по компаниям

Агрегируем сумму и одновременно выводим название reference.

```sql
select
    main.company_id,
    company_id.title as "company_id$",
    count(*) as cnt,
    sum(coalesce(main.amount, 0)) as total_amount
from public.my_entity main
left join companies company_id on company_id.id = main.company_id
%filter%
group by main.company_id, company_id.title
%order%
```

### SQL 21. ROW_NUMBER для нумерации

Window-функции можно использовать без разрушения исходного набора строк.

```sql
select
    main.id,
    main.title,
    row_number() over(order by main.created_at desc) as row_no
from public.my_entity main
%filter%
%order%
```

### SQL 22. Сумма по группе через window

Показываем строку и одновременно общую сумму внутри компании.

```sql
select
    main.id,
    main.company_id,
    main.amount,
    sum(coalesce(main.amount, 0)) over(partition by main.company_id) as company_total
from public.my_entity main
%filter%
%order%
```

### SQL 23. Последняя запись в каждой группе

PostgreSQL DISTINCT ON удобен для query «последнее состояние объекта».

```sql
select distinct on (main.parent_id)
    main.id,
    main.parent_id,
    main.title,
    main.created_at
from public.my_entity main
%filter%
order by main.parent_id, main.created_at desc
```

### SQL 24. JSONB: чтение поля

PostgreSQL JSONB можно раскрывать в отдельные колонки.

```sql
select
    main.id,
    main.title,
    main.payload ->> 'type' as payload_type,
    main.payload ->> 'source' as payload_source
from public.my_entity main
%filter%
%order%
```

### SQL 25. JSONB: вложенное поле

Оператор #>> возвращает текст по пути внутри JSONB.

```sql
select
    main.id,
    main.payload #>> '{customer,name}' as customer_name,
    main.payload #>> '{customer,bin}' as customer_bin
from public.my_entity main
%filter%
%order%
```

### SQL 26. STRING_AGG для дочерних записей

Собираем названия дочерних строк в одно текстовое поле.

```sql
select
    main.id,
    main.title,
    string_agg(child.title, ', ' order by child.title) as child_titles
from public.my_entity main
left join my_child child on child.parent_id = main.id
%filter%
group by main.id, main.title
%order%
```

### SQL 27. EXISTS вместо JOIN

EXISTS удобен, когда нужно проверить наличие дочерней записи без размножения строк.

```sql
select
    main.id,
    main.title
from public.my_entity main
%filter%
    and exists (
        select 1
        from my_child child
        where child.parent_id = main.id
          and child.is_active = 1
    )
%order%
```

### SQL 28. NOT EXISTS

Выбираем объекты, у которых нет активных дочерних записей.

```sql
select
    main.id,
    main.title
from public.my_entity main
%filter%
    and not exists (
        select 1
        from my_child child
        where child.parent_id = main.id
          and child.is_active = 1
    )
%order%
```

### SQL 29. CTE для сложного отчета

CTE делает сложный sql_text читаемее. Алиас main сохраняем в основной части.

```sql
with paid as (
    select
        p.parent_id,
        sum(coalesce(p.amount, 0)) as paid_amount
    from payments p
    group by p.parent_id
)
select
    main.id,
    main.title,
    coalesce(paid.paid_amount, 0) as paid_amount
from public.my_entity main
left join paid on paid.parent_id = main.id
%filter%
%order%
```

### SQL 30. Расширенный DamuBPM auto-query

Комбинация total_count$, sys$uuid, read_at$, reference, created_by и защиты от корзины.

```sql
select
    count(*) over() as total_count$,
    main."sys$uuid" as "sys$uuid",
    (
        select rr.read_at
        from read_rows rr
        where rr.entity_pk = main."sys$uuid"
          and rr.user_id = :user_id
        limit 1
    ) as "read_at$",
    status_id.code as "status_id$code",
    status_id.title as "status_id$",
    created_by.title as "created_by$",
    main.title,
    main.status_id,
    main.created_by,
    main.id
from public.my_entity main
left join statuses status_id on status_id.id = main.status_id
left join users created_by on created_by.id = main.created_by
%filter%
    and coalesce(main.in$trash, 0) = 0
%order%
```

## 8. 30 примеров QueryOptions

### 1 Минимальный запрос

```javascript
let q = new Models.QueryOptions('ai_prompt');
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 2 Первая страница по 15 записей

```javascript
let q = new Models.QueryOptions('ai_prompt');
q.page = 1;
q.perpage = 15;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => this.filtered = r.items || []);
```

### 3 Третья страница

```javascript
let q = new Models.QueryOptions('ai_prompt');
q.page = 3;
q.perpage = 25;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r));
```

### 4 100 записей

```javascript
let q = new Models.QueryOptions('ai_prompt');
q.page = 1;
q.perpage = 100;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 5 Без пагинации

```javascript
let q = new Models.QueryOptions('ai_prompt');
q.perpage = 0;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

Используйте осторожно на больших выборках.

### 6 Строковый поиск по title

```javascript
let q = new Models.QueryOptions('ai_prompt');
q.flt.title = 'GPT';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 7 Два строковых фильтра

```javascript
let q = new Models.QueryOptions('my_query');
q.flt.title = 'договор';
q.flt.code = 'SUP';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 8 Точное равенство по ID

```javascript
let q = new Models.QueryOptions('my_query');
q.flteq.id = '125';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 9 Boolean = true

```javascript
let q = new Models.QueryOptions('my_query');
q.flteq.is_active = true;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 10 Boolean = false

```javascript
let q = new Models.QueryOptions('my_query');
q.flteq.is_active = false;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 11 Reference: несколько статусов

```javascript
let q = new Models.QueryOptions('my_query');
q.fltin.status_id = ['10', '20', '30'];
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 12 Два multi-select фильтра

```javascript
let q = new Models.QueryOptions('my_query');
q.fltin.status_id = ['10', '20'];
q.fltin.company_id = ['1', '5', '9'];
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 13 Числовой диапазон

```javascript
let q = new Models.QueryOptions('my_query');
q.func22.amount = '100000|500000';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 14 Только нижняя граница

```javascript
let q = new Models.QueryOptions('my_query');
let from = 250;
let to = null;
q.func22.amount = (from || to || 0) + '|' + (to || from || 10000000);
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 15 Date range из prime-date

```javascript
let q = new Models.QueryOptions('my_query');
q.func22.created_at = this.createdAtRange;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 16 Строка + точное значение

```javascript
let q = new Models.QueryOptions('my_query');
q.flt.title = 'заявка';
q.flteq.is_active = true;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 17 Строка + список + диапазон

```javascript
let q = new Models.QueryOptions('my_query');
q.flt.title = 'оплата';
q.fltin.status_id = ['2', '3'];
q.func22.amount = '50000|300000';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 18 param1

```javascript
let q = new Models.QueryOptions('get_filter_cols_by_code');
q.param1 = 'ai_prompt';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => this.fields = r.items || []);
```

### 19 param1 для массовых действий

```javascript
let q = new Models.QueryOptions('entity_acts_many');
q.param1 = this.entity_id;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => this.acts_many = r.items || []);
```

### 20 param1 для tools сущности

```javascript
let q = new Models.QueryOptions('bp_processes_tools_by_entity_id');
q.page = 1;
q.perpage = 100;
q.param1 = this.entity_id;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => this.tools = r.items || []);
```

### 21 param1 + param2

```javascript
let q = new Models.QueryOptions('my_query');
q.param1 = '100';
q.param2 = 'active';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 22 Параметры + фильтр

```javascript
let q = new Models.QueryOptions('my_query');
q.param1 = this.entity_id;
q.flt.title = 'тест';
q.flteq.is_active = true;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 23 Сортировка ASC

```javascript
let q = new Models.QueryOptions('ai_prompt');
q.orderBy = 'title';
q.orderAsc = 0;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 24 Сортировка DESC

```javascript
let q = new Models.QueryOptions('ai_prompt');
q.orderBy = 'updated_at';
q.orderAsc = 1;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 25 Фильтр + сортировка + пагинация

```javascript
let q = new Models.QueryOptions('my_query');
q.page = 2;
q.perpage = 20;
q.flt.title = 'банк';
q.orderBy = 'created_at';
q.orderAsc = 1;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items, r.allCount));
```

### 26 JIT: параметры из data

```javascript
let q = new Models.QueryOptions(data.query_code || 'my_query');
if (data.flt) q.flt = data.flt;
if (data.flteq) q.flteq = data.flteq;
if (data.fltin) q.fltin = data.fltin;
if (data.func22) q.func22 = data.func22;
if (data.param1) q.param1 = data.param1;
if (data.param2) q.param2 = data.param2;
if (data.limit) q.perpage = data.limit;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => console.log(r.items));
```

### 27 Архив через $archive

```javascript
let q = new Models.QueryOptions(this.table_code + '$archive');
q.param1 = item.initial_req_uuid;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => this.archives = r.items || []);
```

### 28 Select-query для выпадающего списка

```javascript
let q = new Models.QueryOptions('ai_prompt_select');
q.page = 1;
q.perpage = 35;
q.flt.name = 'sales';
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => this.options = r.items || []);
```

### 29 Query с need_filter

```javascript
let q = new Models.QueryOptions('heavy_query');
q.page = 1;
q.perpage = 50;
q.flteq.company_id = this.companyId;
this.dbQueryService.getQuery(q).pipe(rxjs.first()).subscribe(r => {
    if (r.error == '1') return;
    this.filtered = r.items || [];
});
```

### 30 Универсальный helper

```javascript
loadQuery(code, options = {}) {
    let q = new Models.QueryOptions(code);
    q.page = options.page || 1;
    q.perpage = options.perpage === 0 ? 0 : (options.perpage || 15);

    if (options.flt) q.flt = options.flt;
    if (options.flteq) q.flteq = options.flteq;
    if (options.fltin) q.fltin = options.fltin;
    if (options.func22) q.func22 = options.func22;
    if (options.param1 !== undefined) q.param1 = options.param1;
    if (options.param2 !== undefined) q.param2 = options.param2;
    if (options.orderBy) q.orderBy = options.orderBy;
    if (options.orderAsc !== undefined) q.orderAsc = options.orderAsc;

    return this.dbQueryService.getQuery(q).pipe(rxjs.first());
}

this.loadQuery('ai_prompt', {
    page: 1,
    perpage: 25,
    flt: { title: 'GPT' },
    orderBy: 'updated_at',
    orderAsc: 1
}).subscribe(r => this.filtered = r.items || []);
```

## 9. Типовые ошибки

| Ошибка | Как правильно |
| --- | --- |
| Reference фильтруется через flt | Использовать fltin с массивом ID. |
| Boolean передаётся как строковый поиск | Использовать flteq . |
| Диапазон собирается двумя независимыми фильтрами | Использовать func22 . |
| Нет фильтра при need_filter=1 | Передать фильтр/form-параметр. |
| perpage=0 на огромной таблице | Использовать пагинацию. |
| Сортировка по несуществующему alias | orderBy должен соответствовать полю query. |
| items.length используется как общее количество | Для пагинации использовать allCount . |

DamuBPM Query — практическое руководство.

[← Главная](index.html)

## SQL с `?` placeholders — 30 примеров

В DamuBPM `?` используется как **позиционный bind-плейсхолдер значения**. Сервер готовит SQL через `DbBindReplace(...)`, а значения передаёт ORM отдельно. Поэтому значение не нужно склеивать со строкой SQL.

> **Важно:** порядок `?` должен совпадать с порядком параметров, которые Query передаёт в `formArray`. На frontend обычно используются `param1`, `param2` и далее. Точное сопоставление определяется метаданными Query и `QueryFilterBuild`.

### Базовый принцип

```sql
select main.id, main.title
from public.my_entity main
where main.company_id = ?
  and main.status_id = ?
```

```javascript
let q = new Models.QueryOptions('my_query');
q.param1 = 10;
q.param2 = 2;
```

Первый `?` получает первое bind-значение, второй `?` — второе и т.д.

### Что нельзя параметризовать через `?`

- имя таблицы: `from ?` — нельзя;
- имя колонки: `select ?` — это будет значение, а не идентификатор;
- направление сортировки: `order by title ?` — нельзя;
- динамический список неизвестной длины одним `IN (?)` — не рассчитывайте на автоматическое раскрытие; для UI-списков используйте штатный `fltin`.

Для сортировки DamuBPM используйте `%order%` / `orderBy` / `orderAsc`, а для динамической фильтрации — `%filter%` вместе с `flt`, `flteq`, `fltin`, `func22`.

### 1. Поиск по ID

```sql
select main.id, main.title
from public.my_entity main
where main.id = ?
```

Параметры:
- `param1 = 125`

### 2. Поиск по коду

```sql
select main.id, main.code, main.title
from public.my_entity main
where main.code = ?
```

Параметры:
- `param1 = 'DOC-001'`

### 3. Компания + статус

```sql
select main.id, main.title, main.company_id, main.status_id
from public.my_entity main
where main.company_id = ?
  and main.status_id = ?
```

Параметры:
- `param1 = 10`
- `param2 = 2`

### 4. Параметр + фиксированное условие

```sql
select main.id, main.title
from public.my_entity main
where main.company_id = ?
  and coalesce(main.in$trash, 0) = 0
```

Параметры:
- `param1 = 10`

### 5. ILIKE по названию

```sql
select main.id, main.title
from public.my_entity main
where main.title ilike concat('%', ?, '%')
```

Параметры:
- `param1 = 'договор'`

### 6. Дата от указанной

```sql
select main.id, main.title, main.created_at
from public.my_entity main
where main.created_at >= ?
```

Параметры:
- `param1 = '2026-01-01'`

### 7. Диапазон дат

```sql
select main.id, main.title, main.created_at
from public.my_entity main
where main.created_at >= ?
  and main.created_at < ?
```

Параметры:
- `param1 = '2026-09-01'`
- `param2 = '2026-10-01'`

### 8. Сумма от значения

```sql
select main.id, main.title, main.amount
from public.my_entity main
where main.amount >= ?
```

Параметры:
- `param1 = 100000`

### 9. Диапазон суммы

```sql
select main.id, main.title, main.amount
from public.my_entity main
where main.amount between ? and ?
```

Параметры:
- `param1 = 100000`
- `param2 = 500000`

### 10. Boolean / флаг

```sql
select main.id, main.title, main.is_active
from public.my_entity main
where main.is_active = ?
```

Параметры:
- `param1 = true`

### 11. JOIN + параметр основной таблицы

```sql
select main.id, main.title, c.title as company_title
from public.my_entity main
left join companies c on c.id = main.company_id
where main.company_id = ?
```

Параметры:
- `param1 = 10`

### 12. JOIN + два параметра

```sql
select main.id, main.title, c.title as company_title
from public.my_entity main
left join companies c on c.id = main.company_id
where main.company_id = ?
  and main.status_id = ?
```

Параметры:
- `param1 = 10`
- `param2 = 2`

### 13. EXISTS

```sql
select main.id, main.title
from public.my_entity main
where exists (
    select 1
    from public.my_entity_items i
    where i.parent_id = main.id
      and i.type_id = ?
)
```

Параметры:
- `param1 = 5`

### 14. NOT EXISTS

```sql
select main.id, main.title
from public.my_entity main
where not exists (
    select 1
    from public.my_entity_items i
    where i.parent_id = main.id
      and i.status_id = ?
)
```

Параметры:
- `param1 = 9`

### 15. IN с фиксированным количеством placeholders

```sql
select main.id, main.title, main.status_id
from public.my_entity main
where main.status_id in (?, ?, ?)
```

Параметры:
- `param1 = 1`
- `param2 = 2`
- `param3 = 5`

### 16. CTE + параметр

```sql
with selected as (
    select id
    from public.companies
    where parent_id = ?
)
select main.id, main.title
from public.my_entity main
where main.company_id in (select id from selected)
```

Параметры:
- `param1 = 100`

### 17. Подзапрос с агрегатом

```sql
select main.id, main.title,
       (select count(*)
        from public.my_entity_items i
        where i.parent_id = main.id
          and i.type_id = ?) as item_count
from public.my_entity main
```

Параметры:
- `param1 = 5`

### 18. GROUP BY + HAVING

```sql
select main.company_id, count(*) as cnt
from public.my_entity main
group by main.company_id
having count(*) >= ?
```

Параметры:
- `param1 = 10`

### 19. JSONB: значение ключа

```sql
select main.id, main.payload
from public.my_entity main
where main.payload ->> 'status' = ?
```

Параметры:
- `param1 = 'approved'`

### 20. JSONB: вложенное поле

```sql
select main.id, main.payload
from public.my_entity main
where jsonb_extract_path_text(main.payload, 'client', 'bin') = ?
```

Параметры:
- `param1 = '123456789012'`

### 21. COALESCE для nullable значения

```sql
select main.id, main.title, main.status_id
from public.my_entity main
where coalesce(main.status_id, 0) = ?
```

Параметры:
- `param1 = 0`

### 22. CASE + параметр

```sql
select main.id,
       case when main.amount >= ? then 'large' else 'small' end as amount_group
from public.my_entity main
```

Параметры:
- `param1 = 1000000`

### 23. %filter% + свой параметр + %order%

```sql
select main.id, main.title, main.company_id
from public.my_entity main
%filter%
  and main.company_id = ?
%order%
```

Параметры:
- `param1 = 10`
- `дополнительные UI-фильтры идут через flt/flteq/fltin/func22`

### 24. total_count$ + placeholder

```sql
select count(*) over() as total_count$,
       main.id,
       main.title
from public.my_entity main
%filter%
  and main.company_id = ?
%order%
```

Параметры:
- `param1 = 10`

### 25. :user_id + placeholder

```sql
select main.id, main.title
from public.my_entity main
where main.created_by = :user_id
  and main.company_id = ?
```

Параметры:
- `param1 = 10`
- `:user_id подставляет сервер DamuBPM`

### 26. :lang + placeholder

```sql
select main.id,
       case when :lang = 'kk' then main.title_kk else main.title end as title
from public.my_entity main
where main.status_id = ?
```

Параметры:
- `param1 = 2`
- `:lang подставляет сервер DamuBPM`

### 27. :domain + placeholder

```sql
select main.id, main.title
from public.my_entity main
where main.domain = :domain
  and main.company_id = ?
```

Параметры:
- `param1 = 10`
- `:domain заменяется текущим HTTP Host`

### 28. Select-query: id + name

```sql
select main.id as id,
       main.title as name
from public.my_entity main
where main.company_id = ?
order by main.title
```

Параметры:
- `param1 = 10`

### 29. Archive query по UUID

```sql
select main.id, main.title, main.initial_req_uuid
from public.my_entity_archive main
where main.initial_req_uuid = ?
order by main.id desc
```

Параметры:
- `param1 = '2a5ec7c0-0000-0000-0000-111111111111'`

### 30. Два bind-параметра + динамические фильтры

```sql
select count(*) over() as total_count$,
       main.id,
       main.title,
       main.created_at
from public.my_entity main
%filter%
  and main.company_id = ?
  and main.created_at >= ?
%order%
```

Параметры:
- `param1 = 10`
- `param2 = '2026-01-01'`
- `flt/flteq/fltin/func22 могут добавлять дополнительные фильтры`

## preparam1, preparam2 … preparamN — параметры до `%filter%`

> **Главное правило:** если `?` расположен в SQL **до** `%filter%`, используйте `preparam1`, `preparam2` … `preparamN`. Для `?` после `%filter%` используйте `param1`, `param2` … `paramN`.

Нумерации независимы: первый `?` до `%filter%` — `preparam1`; первый `?` после `%filter%` — `param1`, даже если оба типа присутствуют в одном SQL.

### Почему нужен preparam

Query endpoint передаёт весь `req.Form` в `QueryFilterBuild(...)`. Этот слой формирует итоговый SQL, `filterArray` и `formArray`. Затем ORM получает сначала `filterArray`, потом `formArray`; при пагинации после них добавляются `limit` и `offset`.

```text
SQL template
  ?  <- preparam1
  ?  <- preparam2
%filter%
  ?  <- param1
  ?  <- param2
%order%

Conceptual bind order:
preparam1, preparam2, [dynamic %filter% values], param1, param2, limit, offset
```

### Правила

| Ситуация | Что использовать |
| --- | --- |
| `?` до `%filter%` | `preparam1..N` |
| `?` после `%filter%` | `param1..N` |
| Нет `?` до `%filter%` | `preparam*` не нужен |
| Несколько `?` до фильтра | `preparam1`, `preparam2`, ... |
| Несколько `?` после фильтра | `param1`, `param2`, ... |
| Динамические UI-фильтры | `flt`, `flteq`, `fltin`, `func22` |

> **Важно для `getRowById`:** текущий Go-обработчик берёт `origSql`, заменяет `%filter%` на `where main.id=?` и выполняет SQL с аргументами `rowId, formArray`. Если исходный SQL содержит дополнительные `?` до `%filter%`, detail-path нужно обязательно тестировать: bind-порядок может отличаться от обычного Query. До подтверждения реализации `QueryFilterBuild` безопаснее не использовать preparam в query, который должен работать через `getRowById`.

### 12 примеров preparam

#### 1. Один preparam до %filter%

```sql
select main.id, main.title
from (
    select *
    from public.my_entity
    where company_id = ?
) main
%filter%
%order%
```

```javascript
let q = new Models.QueryOptions('my_query');
q.preparam1 = 10;
q.flt.title = 'банк';
```

Первый ? расположен до %filter%, поэтому ему соответствует preparam1.

#### 2. Два preparam до %filter%

```sql
select main.id, main.title
from (
    select *
    from public.my_entity
    where company_id = ?
      and type_id = ?
) main
%filter%
%order%
```

```javascript
q.preparam1 = 10;
q.preparam2 = 7;
```

Нумерация preparam идёт слева направо только среди ? до %filter%.

#### 3. preparam до фильтра и param после фильтра

```sql
select main.id, main.title, main.status_id
from (
    select *
    from public.my_entity
    where company_id = ?
) main
%filter%
  and main.status_id = ?
%order%
```

```javascript
q.preparam1 = 10;
q.flt.title = 'договор';
q.param1 = 2;
```

preparam1 обслуживает ? до %filter%, param1 — первый ? после %filter%. Нумерации независимы.

#### 4. Два preparam + два param

```sql
select main.id, main.title, main.created_at
from (
    select *
    from public.my_entity
    where company_id = ?
      and type_id = ?
) main
%filter%
  and main.status_id = ?
  and main.created_at >= ?
%order%
```

```javascript
q.preparam1 = 10;
q.preparam2 = 7;
q.flteq.is_active = true;
q.param1 = 2;
q.param2 = '2026-01-01';
```

Концептуальный порядок bind: preparam1, preparam2, значения динамического фильтра, param1, param2.

#### 5. preparam в CTE

```sql
with allowed_rows as (
    select id
    from public.my_entity
    where company_id = ?
)
select main.id, main.title
from public.my_entity main
join allowed_rows x on x.id = main.id
%filter%
%order%
```

```javascript
q.preparam1 = 10;
```

CTE находится до %filter%, поэтому его placeholder относится к preparam.

#### 6. preparam в JOIN

```sql
select main.id, main.title, c.title as company_title
from public.my_entity main
join companies c
  on c.id = main.company_id
 and c.parent_id = ?
%filter%
%order%
```

```javascript
q.preparam1 = 100;
```

Placeholder в JOIN текстуально стоит до %filter%.

#### 7. preparam в SELECT-подзапросе

```sql
select main.id,
       main.title,
       (select count(*)
          from public.my_entity_items i
         where i.parent_id = main.id
           and i.type_id = ?) as item_count
from public.my_entity main
%filter%
%order%
```

```javascript
q.preparam1 = 5;
```

Даже если ? находится в SELECT-списке, решает его позиция относительно %filter%.

#### 8. preparam для даты внутри CTE

```sql
with recent as (
    select *
    from public.my_entity
    where created_at >= ?
)
select main.id, main.title, main.created_at
from recent main
%filter%
%order%
```

```javascript
q.preparam1 = '2026-01-01';
```

Дата передаётся bind-параметром, без конкатенации SQL.

#### 9. preparam + total_count$

```sql
select count(*) over() as total_count$,
       main.id,
       main.title
from (
    select *
    from public.my_entity
    where company_id = ?
) main
%filter%
%order%
```

```javascript
q.preparam1 = 10;
q.page = 1;
q.perpage = 50;
```

total_count$ не меняет правило preparam; LIMIT/OFFSET сервер добавит в самом конце.

#### 10. Только param после %filter%

```sql
select main.id, main.title
from public.my_entity main
%filter%
  and main.company_id = ?
%order%
```

```javascript
q.param1 = 10;
```

Здесь preparam не нужен: до %filter% нет ни одного bind-placeholder.

#### 11. Только preparam, без param после фильтра

```sql
select main.id, main.title
from (
    select *
    from public.my_entity
    where company_id = ?
      and created_at >= ?
) main
%filter%
%order%
```

```javascript
q.preparam1 = 10;
q.preparam2 = '2026-01-01';
```

После %filter% пользовательских ? нет, поэтому param1/param2 не нужны.

#### 12. preparamN: много параметров до %filter%

```sql
select main.*
from (
    select *
    from public.my_entity
    where company_id = ?
      and type_id = ?
      and category_id = ?
      and created_at >= ?
) main
%filter%
%order%
```

```javascript
q.preparam1 = 10;
q.preparam2 = 7;
q.preparam3 = 15;
q.preparam4 = '2026-01-01';
```

Смысл preparamN: N — порядковый номер placeholder до %filter%.
