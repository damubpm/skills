# DamuBPM — детализированные страницы по UI-блокам

[← Главная](index.html)

Практическая справка по построению автогенерируемых detail-страниц DamuBPM через `pages_ui_blocks` и `pages_ui_blocks_n`. Документ основан на фактической странице `pagesdetails` (`pages.id=4478`, `is_auto=1`) и текущем механизме `pkg/page → pages_for_gen_ng → ParseTemplate` в `test-helpdeskv19`.

> Главная идея: **страница задаёт общий шаблон, `pages_ui_blocks` задаёт контейнеры, а `pages_ui_blocks_n` задаёт содержимое и связи между контейнерами.** Сгенерированные `pages.angular_template` и `pages.angular_json` являются результатом генерации, а не основным местом ручного редактирования.

---

## 1. Модель страницы

Автогенерируемая detail-страница строится как дерево.

```text
pages
└── pages_ui_blocks                  ← реальные UI-контейнеры
    └── pages_ui_blocks_n            ← элементы внутри контейнера
        ├── attr                     ← атрибут сущности
        ├── block                    ← ссылка на другой UI-блок
        ├── tp                       ← табличная часть
        ├── html                     ← HTML-фрагмент
        ├── logic                    ← логика / код шаблона
        └── ui_component             ← UI-компонент
```

### Что является чем

| Уровень | Назначение | Пример из `pagesdetails` |
|---|---|---|
| `pages` | Сама страница и её общий шаблон | `pagesdetails` |
| `pages_ui_blocks` | Контейнер/секция/вкладка/TabView | `pages_4c18492d...` — `TabView` |
| `pages_ui_blocks_n` | Узел внутри контейнера | `pages_17981` — элемент типа `block` |
| `ui_block_tpl` | Шаблон отображения блока | `default_page`, `tabview`, `tabview_tab` |

**Важно:** элемент типа `block` не является самим дочерним блоком. Он является **ссылкой из родительского блока на дочерний `pages_ui_blocks`**.

---

## 2. Типы элементов `pages_ui_blocks_n`

В текущей системе подтверждены следующие типы:

| Код | Назначение | Что хранит/связывает |
|---|---|---|
| `attr` | Поле основной сущности | `entity_attrs` |
| `block` | Вложенный UI-блок | другой `pages_ui_blocks` |
| `tp` | Табличная часть | `table_parts` |
| `html` | Произвольный HTML-фрагмент | HTML элемента |
| `logic` | Логический/шаблонный фрагмент | `logic` / Angular-шаблон элемента |
| `ui_component` | Явно заданный компонент | UI component |

На странице `pagesdetails` реально используются `block`, `attr`, `tp` и `logic`.

---

## 3. Как определяется корневой блок

При подготовке данных `pages_for_gen_ng` набор `ui_blocks` содержит блоки страницы, которые **не подключены как дочерние через элемент типа `block`**.

То есть логически:

```text
Корневой блок = pages_ui_blocks страницы,
который не встречается как child block
в pages_ui_blocks_n с типом block.
```

Вложенные связи попадают в `ui_blocks2`.

Это позволяет шаблону страницы начать с корневых блоков, а затем рекурсивно разобрать дочерние контейнеры.

---

## 4. Фактическое дерево `pagesdetails`

```text
pagesdetails — «Страницы»
│
└── BLOCK pagesdetails_udsdsad
    │  template: default_page
    │
    ├── [logic] pages_18153
    │      logic: //
    │      nn: 100
    │
    └── [block] pages_17973
        │  nn: 200
        │
        └── BLOCK pages_4c18492d_f81b_4952_9890_083acfa4d486
            │  template: tabview
            │  title: TabView
            │
            ├── [block] pages_17981, nn=50
            │   └── BLOCK ...c956
            │       │  template: tabview_tab
            │       │  header: Информация
            │       │
            │       └── [block] pages_17982
            │           └── BLOCK pages_17982
            │               │  template: default_page
            │               ├── [attr] code            — Код
            │               ├── [attr] entity_id       — Entity
            │               ├── [attr] module_id       — Модуль
            │               ├── [attr] db_template     — Шаблон в базе
            │               ├── [attr] title           — Наименование
            │               ├── [attr] page_type_id    — Вид страницы
            │               ├── [attr] url             — Ссылка
            │               └── [attr] is_auto         — Автогенерация
            │
            ├── [block] pages_17975, nn=100
            │   └── BLOCK ...e8431
            │       │  template: tabview_tab
            │       │  header: Логика
            │       └── [block] pages_17974
            │           └── BLOCK ...eaed
            │               │  template: default_page
            │               └── [attr] angular_json — Логика
            │
            ├── [block] pages_17978, nn=200
            │   └── BLOCK ...5687c
            │       │  template: tabview_tab
            │       │  header: HTML
            │       └── [block] pages_17979
            │           └── BLOCK pages_17979
            │               │  template: default_page
            │               └── [attr] angular_template — HTML
            │
            └── [block] pages_18126, nn=300
                └── BLOCK ...1dc1
                    │  template: tabview_tab
                    │  header: Custom Blocks
                    └── [tp] pages_18127
                        └── page_cus_blocks — Кастомные блоки
```

### Визуальное представление

```text
Главный default_page
└── TabView
    ├── Информация
    │   └── default_page
    │       ├── Код
    │       ├── Entity
    │       ├── Модуль
    │       ├── Шаблон в базе
    │       ├── Наименование
    │       ├── Вид страницы
    │       ├── Ссылка
    │       └── Автогенерация
    ├── Логика
    │   └── default_page
    │       └── angular_json
    ├── HTML
    │   └── default_page
    │       └── angular_template
    └── Custom Blocks
        └── page_cus_blocks
```

---

## 5. Зачем внутри вкладки ещё один `default_page`

`tabview_tab` отвечает за вкладку и её заголовок, но поля удобно размещать не прямо в ней, а во вложенном `default_page`.

Паттерн:

```text
TabView
└── tabview_tab «Информация»
    └── default_page
        ├── attr
        ├── attr
        └── attr
```

Так разделяются ответственности:

- `tabview` — переключатель вкладок;
- `tabview_tab` — одна вкладка и её заголовок;
- `default_page` — сетка/контейнер обычных полей;
- `attr` — конкретное поле.

Этот же паттерн позволяет вместо полей положить внутрь вкладки другой блок или табличную часть.

---

## 6. Как `pkg/page` собирает страницу

Для автогенерируемых страниц основная последовательность выглядит так:

```text
genPages(entity_id, user_id)
│
├── выбирает pages по entity_id и is_auto=1
│
├── для каждой страницы:
│   └── Detail(page_id, "pages_for_gen_ng", user_id)
│
├── подготавливает наборы данных
│   ├── ui_blocks
│   ├── ui_blocks2
│   ├── ui_attrs
│   ├── table_parts
│   ├── table_part_cols
│   └── ...
│
├── добавляет component_param
├── добавляет параметры table parts
├── формирует val.param страницы
├── формирует val.custom_block
│
├── ParseTemplate(page template HTML, val, user_id)
├── ParseTemplate(page template JS,   val, 1)
│
├── заменяет {[{ → {{
│
└── сохраняет результат в:
    ├── pages.angular_template
    └── pages.angular_json
```

`genPages` принимает **ID сущности**, а не ID страницы.

---

## 7. Какие данные получает шаблон страницы

Для блочной генерации особенно важны четыре набора.

### `ui_blocks`

Корневые блоки страницы.

### `ui_blocks2`

Вложенные связи типов `block` и `tp`.

Используются для построения дерева блоков.

### `ui_attrs`

Элементы типов:

- `attr`;
- `html`;
- `ui_component`;
- `logic`.

Для атрибутов сюда уже попадает рассчитанный Angular template компонента.

### `table_parts`

Табличные части, размещённые в блоках страницы.

---

## 8. Как блок получает свой HTML

У каждого `pages_ui_blocks` есть собственный `is_auto`.

```text
pages_ui_blocks.is_auto = 1
    → используется ui_block_tpl.angular_template

pages_ui_blocks.is_auto != 1
    → используется pages_ui_blocks.angular_template
```

Это отдельно от `pages.is_auto`.

Если `pages.is_auto=0`, страница целиком исключается из обычной `genPages`.

---

## 9. Как `default_page` собирает содержимое

Концептуально `default_page` делает три вещи:

```text
default_page(block)
│
├── вывести ui_attrs, принадлежащие этому block
├── вывести table_parts, размещённые в этом block
└── разобрать дочерние block-узлы этого block
```

Для полей используется связь `pages_ui_blocks_id`.

Для табличных частей генератор использует `_tp_ui_block_id`.

Для вложенных блоков используется связь родительского `pages_ui_blocks_n` типа `block` с `block_id` дочернего `pages_ui_blocks`.

---

## 10. Рекурсивная логика построения дерева

Для ИИ или генератора полезно мыслить так:

```pseudo
renderBlock(block):
    template = resolveBlockTemplate(block)

    nodes = all nodes where nodes.parent_block_id == block.id
    sort nodes by nn

    for node in nodes:
        if node.type == "attr":
            render attribute component

        if node.type == "tp":
            render table part

        if node.type == "block":
            child = node.block_id
            renderBlock(child)

        if node.type == "logic":
            render logic template

        if node.type == "html":
            render html template

        if node.type == "ui_component":
            render explicit component
```

При обходе нужно хранить `visited block ids`, чтобы не допустить бесконечный цикл при ошибочной конфигурации.

---

## 11. Порядок элементов

Для узлов страницы ключевое поле порядка — `nn`.

В `pagesdetails` порядок вкладок такой:

| `nn` | Вкладка |
|---:|---|
| 50 | Информация |
| 100 | Логика |
| 200 | HTML |
| 300 | Custom Blocks |

Рекомендуется всегда явно заполнять `nn`, если порядок имеет значение.

Не стоит рассчитывать на ID записи как на бизнес-порядок.

---

## 12. Ширина элементов

Для обычного UI-элемента приоритет ширины следующий:

```text
pages_ui_blocks_n.ui_width_id
        ↓
entity_attr.ui_width_id
        ↓
col-md-4 mb-3
```

То есть ширину можно переопределить на уровне конкретного размещения поля, не меняя сам атрибут сущности глобально.

---

## 13. Выбор UI-компонента атрибута

Для `ui_attrs.angular_template` текущая логика делает fallback между компонентами.

Концептуально:

```text
компонент элемента страницы
        ↓
компонент атрибута
        ↓
компонент типа данных
        ↓
HTML/logic самого элемента
```

Поэтому внешний вид одного и того же атрибута можно менять на конкретной странице, не обязательно меняя глобальный компонент атрибута.

**Осторожно:** пустая строка и `NULL` ведут себя по-разному при `coalesce`.

---

## 14. Практический рецепт новой detail-страницы

### Шаг 1. Создать/настроить страницу

Нужны:

- `pages.entity_id`;
- шаблон detail-страницы;
- `pages.is_auto=1`;
- URL страницы;
- код страницы.

### Шаг 2. Создать корневой блок

Типичный вариант:

```text
root
└── ui_block_tpl = default_page
```

### Шаг 3. Если нужны вкладки — создать `TabView`

Создать блок:

```text
ui_block_tpl = tabview
```

И добавить его в корневой блок через `pages_ui_blocks_n` типа `block`.

### Шаг 4. Создать вкладки

Для каждой вкладки создать отдельный `pages_ui_blocks`:

```text
ui_block_tpl = tabview_tab
header_title = "..."
```

Каждую вкладку подключить элементом типа `block` внутрь `TabView`.

### Шаг 5. Создать внутренний `default_page`

Если вкладка содержит обычные поля, создать внутри неё дочерний блок `default_page`.

```text
tabview_tab
└── default_page
```

### Шаг 6. Добавить поля

В `default_page` добавить элементы `pages_ui_blocks_n` типа `attr`.

Каждый узел должен ссылаться на нужный `entity_attr`.

### Шаг 7. Добавить табличные части

Если нужна ТЧ:

```text
[tp] → table_parts.code
```

Пример `pagesdetails`:

```text
Custom Blocks
└── [tp] page_cus_blocks
```

### Шаг 8. Настроить `nn` и ширины

Задать порядок и при необходимости `ui_width_id`.

### Шаг 9. Перегенерировать страницу

После изменения метаданных вызвать генерацию для **сущности**:

```text
genPages(entity_id, user_id)
```

### Шаг 10. Проверить результат

Проверить:

- `pages.angular_template`;
- `pages.angular_json`;
- наличие нужных блоков;
- порядок вкладок;
- поля;
- табличные части;
- readonly/required;
- работу страницы в браузере.

---

## 15. Что редактировать, а что не редактировать

### Правильно

Редактировать источник генерации:

- `pages_ui_blocks`;
- `pages_ui_blocks_n`;
- `ui_block_tpl`;
- UI component;
- параметры элементов;
- `table_parts`;
- `page_params`;
- `page_cus_blocks`;
- общий `page_tpls.detail`, если изменение должно затронуть все detail-страницы.

### Нежелательно

Ручное редактирование:

```text
pages.angular_template
pages.angular_json
```

для страницы `is_auto=1`.

Следующая генерация может полностью перезаписать изменения.

---

## 16. Custom Blocks

`page_cus_blocks` — это не то же самое, что `pages_ui_blocks`.

- `pages_ui_blocks` определяют **структуру интерфейса**;
- `page_cus_blocks` добавляют фрагменты в предусмотренные шаблоном точки расширения.

В `pagesdetails` табличная часть `page_cus_blocks` вынесена в отдельную вкладку `Custom Blocks`, чтобы можно было редактировать расширения конкретной страницы.

Генератор формирует:

```text
val.custom_block[place_code]
```

из custom blocks страницы и типа сущности.

---

## 17. Быстрый SQL для просмотра дерева страницы

Для диагностики удобно получить связи блоков и элементов одним запросом:

```sql
select
    p.code as page_code,
    b.code as parent_block_code,
    ubt.code as block_tpl_code,
    b.header_title,
    n.code as element_code,
    nt.code as element_type,
    n.nn,
    ea.code as attr_code,
    tp.code as table_part_code,
    cb.code as child_block_code
from pages p
join pages_ui_blocks b on b.pages_id = p.id
left join ui_block_tpl ubt on ubt.id = b.ui_block_tpl_id
left join pages_ui_blocks_n n on n.pages_ui_blocks_id = b.id
left join pages_ui_blocks_n_t nt on nt.id = n.t_id
left join entity_attrs ea on ea.id = n.attr_id
left join table_parts tp on tp.id = n.tp_id
left join pages_ui_blocks cb on cb.id = n.block_id
where p.code = 'pagesdetails'
order by coalesce(b.nn,b.id), coalesce(n.nn,n.id);
```

Дальше дерево строится по паре:

```text
parent_block_code → child_block_code
```

для элементов `element_type='block'`.

---

## 18. Алгоритм анализа страницы для ИИ

Если нужно автоматически понять существующую автогенерируемую страницу:

1. Получить страницу по `code`/UUID.
2. Проверить `is_auto`.
3. Получить все `pages_ui_blocks` страницы.
4. Получить все `pages_ui_blocks_n`.
5. Для каждого узла определить тип.
6. Для `block` определить дочерний `pages_ui_blocks`.
7. Найти блоки, на которые никто не ссылается как на child — это корни.
8. Построить adjacency map:

```text
parent_block_id → [nodes]
```

9. Отсортировать каждый набор по `nn`.
10. Рекурсивно вывести дерево.
11. Для `attr` показывать `entity_attrs.code/title`.
12. Для `tp` показывать `table_parts.code/title`.
13. Для вкладок показывать `header_title`.
14. Контролировать циклы.
15. После изменений запускать генерацию по `entity_id`.

---

## 19. Типовые конструкции

### Обычная карточка

```text
default_page
├── attr
├── attr
├── attr
└── tp
```

### Карточка с вкладками

```text
default_page
└── tabview
    ├── tabview_tab
    │   └── default_page
    │       ├── attr
    │       └── attr
    └── tabview_tab
        └── default_page
            └── tp
```

### Вложенные секции

```text
default_page
├── default_page
│   ├── attr
│   └── attr
└── default_page
    ├── attr
    └── tp
```

### Вкладка с несколькими секциями

```text
tabview_tab
├── default_page «Основное»
│   ├── attr
│   └── attr
└── default_page «Дополнительно»
    ├── attr
    └── tp
```

---

## 20. Типовые ошибки

### Блок создан, но не отображается

Проверить:

- принадлежит ли он нужной `pages`;
- подключён ли через `pages_ui_blocks_n` типа `block`;
- является ли он корневым;
- заполнен ли шаблон блока;
- `is_auto` блока;
- `ng_if`.

### Вкладка есть, но пустая

Проверить, есть ли внутри `tabview_tab`:

- `attr`;
- `tp`;
- либо дочерний `default_page`.

### Поле существует, но не видно

Проверить:

- есть ли узел типа `attr`;
- правильный ли `attr_id`;
- `ng_if`;
- `hide_field`;
- компонент;
- результат генерации.

### Порядок нестабильный

Заполнить `nn` у элементов и блоков.

### После ручной правки HTML всё исчезло

Если страница `is_auto=1`, изменение было внесено в результат генерации. Нужно исправлять источник и повторно выполнять `genPages`.

### Генерация уходит в рекурсию

Проверить циклы:

```text
Block A → Block B → Block A
```

Такие связи создавать нельзя.

---

## 21. Проверочный чек-лист

Перед генерацией:

- [ ] страница привязана к правильной сущности;
- [ ] `pages.is_auto=1`;
- [ ] есть корневой блок;
- [ ] каждый дочерний блок подключён узлом `block`;
- [ ] нет циклов;
- [ ] у вкладок есть `header_title`;
- [ ] поля ссылаются на правильные `entity_attrs`;
- [ ] табличные части ссылаются на правильные `table_parts`;
- [ ] задан `nn`;
- [ ] при необходимости задан `ui_width_id`;
- [ ] блоки используют правильный `ui_block_tpl`.

После генерации:

- [ ] сгенерирован `angular_template`;
- [ ] сгенерирован `angular_json`;
- [ ] дерево визуально соответствует метаданным;
- [ ] вкладки идут в правильном порядке;
- [ ] поля редактируются/блокируются корректно;
- [ ] required/readonly работают;
- [ ] табличные части загружаются;
- [ ] custom blocks вставились в нужные места;
- [ ] повторная генерация не ломает страницу.

---

## 22. Короткая памятка

```text
PAGE
└── ROOT BLOCK
    └── BLOCK NODE
        └── TABVIEW
            ├── BLOCK NODE → TAB
            │   └── BLOCK NODE → DEFAULT_PAGE
            │       ├── ATTR
            │       ├── ATTR
            │       └── TP
            └── BLOCK NODE → TAB
                └── BLOCK NODE → DEFAULT_PAGE
                    └── ATTR
```

Если нужно изменить **структуру страницы** — работайте с `pages_ui_blocks` и `pages_ui_blocks_n`.

Если нужно изменить **общий внешний вид блока** — работайте с `ui_block_tpl`.

Если нужно изменить **вид одного поля** — работайте с UI component конкретного элемента/атрибута.

Если нужно добавить **список связанных записей** — размещайте `tp`.

Если нужно изменить **все detail-страницы** — меняйте общий шаблон `page_tpls.detail` и затем перегенерируйте затронутые сущности.

---

## 23. Источники истины

Для автогенерируемой страницы приоритетно анализировать:

```text
pages
pages_ui_blocks
pages_ui_blocks_n
ui_block_tpl
entity_attrs
table_parts
page_params
page_cus_blocks
```

А `pages.angular_template` и `pages.angular_json` рассматривать как **скомпилированный результат** этих настроек.

[← Главная](index.html)
