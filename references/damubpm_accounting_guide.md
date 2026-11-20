# Бухгалтерский учёт в DamuBPM (its_coa): справочник разработчика

Справочник по подсистеме бухучёта модуля `its`: план счетов, бухгалтерские модели, движок проведения `pkg/its_coa`, процессы «Провести / Отменить проведение», массовое проведение, закрытие месяца, сторно, регистры и отчёты, печатные формы, тестирование и перенос. Состояние на 20.11.2026, стенд test-helpdesk.

Читать перед любой задачей, где документ должен давать проводки, где нужен отчёт по проводкам или правка `pkg/its_coa`.

---

## 1. Главное за минуту

1. Учёт хранится в БД (TigerBeetle выключен с 13.11.2026). Документ → операция `its_coa_oper` → проводки `its_coa_pst` → движения `its_coa_trn` (2 на проводку) → история остатков `its_coa_bal` → текущие остатки лицевых счетов `its_coa_acc`.
2. Проводки строит **бухгалтерская модель** (`its_r_coa_mdl` + строки `its_r_coa_mdl_n` + условия `its_r_coa_mdl_cond`). Код проводок для документа не пишется: создаются модель и, при необходимости, PREPARE-хук в `pkg/its_coa`.
3. Проведение только через `pkg/its_coa`: `postDocument` / `doMdl` (провести), `cancelMdl` (отменить — сторно той же датой), `previewMdl` (пробный расчёт), `reversePst` (сторно одной проводки).
4. Сальдо хранится как Дт−Кт. «Натуральное» сальдо пассивного счёта (`its_r_coa.is_passive = 1`) = Кт−Дт.
5. Лицевой счёт `its_coa_acc` = счёт плана + подразделение + до трёх субконто (`uuid1..3` = `sys$uuid` записей справочников).
6. Пара Дт/Кт обязана быть в `its_r_coa_correct_dtcr`, иначе 422.
7. На документ — одна проведённая операция (уникальный индекс `ux_its_coa_oper_posted`). Повторное проведение → 409 «Документ уже проведен».
8. Закрытый месяц (`its_d_coa_close`) запрещает проведение и отмену с датой ≤ даты закрытия (409).
9. Кнопки документа — клоны готовых процессов (`<entity>_posting`, `_cancel_posting`, `_mass_post`, `_mass_cancel`). `create_bp_process` из шаблона ломает шаблон — клонировать SQL-запросом (раздел 8).
10. По умолчанию отчёты скрывают отменённые проведения и сторно (флажки «Показывать сторно / отмены»): пары гасят друг друга в ту же дату, сальдо от флажков не зависит, меняются только обороты.

---

## 2. Модель данных

| Таблица | Назначение | Ключевые поля |
|---|---|---|
| `its_r_coa` | План счетов (351 счёт, типовой план РК) | `code` (сравнивать через `trim`), `title`, `parent_id`, `is_passive`, `is_contr_active`, `is_cont_passive` |
| `its_r_coa_n` | Виды субконто счёта (664 строки из 1С) | счёт, `nn` 1–3, `title`, `entity_id` (сущность справочника), `code` = `md5('its_r_coa_n|'||trim(coa.code)||'|'||nn)::uuid` (одинаков на всех стендах) |
| `its_r_coa_correct_dtcr` | Допустимые корреспонденции (335) | `dt_coa_id`, `cr_coa_id`, `code` = `<Дт>_<Кт>` (забаланс: `9021_off`) |
| `its_r_coa_mdl` | Бухгалтерская модель | `entity_id`, `code`, `priority`, `is_active`, `match_group`, `condition_mode` (all), `stop_on_match`, `check_amount`, `cashflow_item_id`, `dscr` |
| `its_r_coa_mdl_cond` | Условия выбора модели | `mdl_id`, `condition_type` (attr / tp_rows), `attr_id`, `operator_code`, `value_text`, `compare_by` (code / id / пусто), `nn`, `code` = `<модель>_cond_<nn>` |
| `its_r_coa_mdl_n` | Строки модели (проводки) | `nn`, `dr_coa_id`, `dr_coa_attr_id`, `cr_acc_id` (это счёт Кт плана!), источник суммы, источники субконто, `code` = `<модель>_<nn>` |
| `its_r_coa_mdl_n_by` | Источник субконто/суммы | 1 `attr`, 2 `table_part_attr`, 3 `none`, 4 `attr_opt` |
| `its_coa_oper` | Операция проведения документа | `entity_id`, `entity_pk`, `entity_uuid`, `mdl_id`, `status` (posted / reversed), `posting_date`, `dep_id`, `amount`, `posted_at`, `reversed_at` |
| `its_coa_pst` | Проводка Дт/Кт | `oper_id`, `nn`, `mdl_n_id`, `source_uuid` (документ или строка ТЧ), `dr_acc_id`, `cr_acc_id`, `amount`, `posting_date`, `is_reverse`, `reverse_pst_id`, `entity_id/pk`, `document_uuid` |
| `its_coa_trn` | Движение по лицевому счёту | `acc_id`, `corr_acc_id`, `ui_is_dr`, `amount`, `posting_date`, `pst_id`, `oper_id`, `is_reverse` |
| `its_coa_bal` | Остаток после каждого движения | `acc_id`, `nn` (= `its_coa_acc.bal_nn`), `balance`, `dr_total`, `cr_total`, `trn_id`, `posting_date`; уникально (acc_id, nn) |
| `its_coa_acc` | Лицевой счёт и текущий остаток | `coa_id`, `dep_id`, `uuid1..3`, `balance` (Дт−Кт), `dr_total`, `cr_total`, `bal_nn`, `last_trn_at` |
| `its_coa_mass`, `its_coa_mass_n` | Задания массового проведения (REST its_coa_mass) | |
| `its_d_coa_close` | Закрытие месяца | |
| `its_coa_tiger_trn` | Архив TigerBeetle (не используется) | |

Индексы `ux_its_coa_oper_posted` и `ux_its_coa_bal_acc_nn` созданы вручную — патчем не переносятся.

### Субконто и справочники (`its_r_coa_n.entity_id`)

| Вид субконто | Сущность |
|---|---|
| Контрагенты | `its_cli` |
| Договоры | `its_cli_contract` |
| Работники организации | `its_staff` |
| Подразделения | `deps` |
| Статьи движения денежных средств | `its_r_cashflow_item` |
| Денежные средства | `companies_bank` (в отчётах показывать `iban`) |
| Налоги, сборы, отчисления | `its_r_kbk` |
| Виды платежей в бюджет | `its_r_knp` |

При создании **нового** лицевого счёта `pkg.getAccId` проверяет каждое заполненное субконто: запись с таким `sys$uuid` должна быть в таблице сущности, иначе 422 «Счёт 3350, субконто 1 «Работники организации»: значение … не найдено в справочнике…». Существующие лицевые счета не перепроверяются. Субконто «ОС» на 2410/2420 — сама карточка `its_r_f_asset` (сущности у вида нет, значения пишет модель).

Фактическая аналитика расчётных счетов:

| Счёт | uuid1 | uuid2 | uuid3 |
|---|---|---|---|
| 1030 | банковский счёт `companies_bank` | статья ДДС `its_r_cashflow_item` | — |
| 1210, 3510 | покупатель `its_cli` | договор (основной, по цепочке ДС) | — |
| 1710, 3310 | поставщик `its_cli` | договор | — |
| 3350 | работник `its_staff` | — | — |
| 32xx по реестру | — | работник (3211/3212/3213 — субконто 2; 3220/3231 — субконто 3) | |
| 2410, 2420 | основное средство `its_r_f_asset` | — | — |

### Подразделение документа

Проводки всегда в разрезе подразделения (`dep_id`). `resolveDep`: `dep_id` документа → иначе `companies.main_dep_id` организации (через `companies_bank_id → companies_id`, иначе `company_id`) и запись его в документ → нет ни того, ни другого → 422. Колонки определяются через `information_schema`, поэтому сущность без `companies_bank_id`/`company_id` не падает. У всех организаций заполнено «Главное подразделение» `companies.main_dep_id`.

---

## 3. Бухгалтерская модель

### 3.1. Выбор модели

- Кандидаты — активные модели сущности документа (`is_active = 1`, не в корзине), чьи условия выполняются (`condition_mode = all`).
- Побеждает максимальный `priority`. Две модели с одинаковым максимальным приоритетом → 409.
- Ни одной → «Для документа не найдена подходящая активная бухгалтерская модель (проверьте КНП, КБК и операцию)». Для документов, где выбор идёт по своему реквизиту (например, способ поступления ОС), лучше заранее проверить этот реквизит в скрипте БП и выдать понятный текст.
- Явный выбор: `doMdl({mdl_id | mdl_code, ...})`.

Условия (`its_r_coa_mdl_cond`):
- `condition_type = attr` — значение атрибута документа `attr_id`. `compare_by = code` — сравнивается `code` связанной записи (например, `its_r_knp.code`), `id` — сам id, пусто — значение. Значения `trim`-ятся (в `its_r_knp.code` бывают ведущие пробелы « 332»).
- Операторы: `eq`, `neq`, `in`, `not_in` (список через запятую), `prefix`, `not_prefix`, `is_null`, `is_not_null`, `gt`, `gte`, `lt`, `lte`.
- `condition_type = tp_rows` — есть ли строки табличной части: `attr_id` = атрибут связи строки ТЧ с документом, `eq 1` — строки есть, `eq 0` — нет. Так модели «по реестру» (приоритет 310) выигрывают у базовых (300) только при наличии реестра.

### 3.2. Строка модели (`its_r_coa_mdl_n`)

- Счёт Дт: `dr_coa_id`; если задан `dr_coa_attr_id` («Счёт Дт из реквизита документа») и реквизит заполнен — счёт берётся из документа (например, `coa_id` входящего АВР или ОС). Корреспонденция проверяется по фактическому счёту.
- Счёт Кт: `cr_acc_id` (это id счёта **плана** `its_r_coa`, а не лицевого счёта — историческое имя).
- Сумма: `attr_amnt_by_id` 1 (`attr_amnt_id` — атрибут шапки) или 2 (`tp_amnt_id` — ТЧ, `tp_attr_amnt_id` — атрибут строки). Пустое значение → `0.00`, строка пропускается. Отрицательная → 422.
- Субконто Дт/Кт 1–3: `attr_dr_n1_by_id` … `attr_cr_n3_by_id` (1 attr, 2 table_part_attr, 3 none, 4 attr_opt) + `attr_dr_nX_id` / `attr_cr_nX_id` (атрибут шапки) или `tp_dr_nX_id` + `tp_attr_dr_nX_id` (ТЧ и её атрибут). Значение-ссылка превращается в `sys$uuid` связанной записи; атрибут `id` → `sys$uuid` самого документа (или строки ТЧ).
- `attr` и `table_part_attr` обязательны: пусто → 422 «Не заполнено обязательное субконто». `attr_opt` — пусто допустимо (субконто остаётся пустым).
- В одной строке модели нельзя смешивать разные ТЧ (500). Строка с ТЧ даёт по проводке на каждую строку ТЧ (`select * from <ТЧ> where <link>=? and coalesce("in$trash",0)=0 order by id`); `source_uuid` проводки = `sys$uuid` строки ТЧ.
- Пустая ТЧ не ошибка: строки модели по ТЧ пропускаются; 422 «Не найдены строки табличной части…» только если проводок нет совсем.
- **Дата проводки из строки ТЧ** (с 20.11.2026): если в строке ТЧ есть колонка `pst_date`, проводка пишется этой датой, иначе — датой операции. Сторно берёт даты исходных проводок.
- **Формулы не поддерживаются**: строки ТЧ читаются `select *` (формульных колонок там нет), шапка — `EntityValueById`. Нужное значение заранее записать в обычную колонку (PREPARE-хук или скрипт БП перед проведением, пример — `its_d_hr_payroll.sum_stax`).

### 3.3. Прочее

- `check_amount = 1`: сумма строк с денежным счётом 10xx (если таких строк нет — всех строк) должна равняться `amount` документа.
- `cashflow_item_id`: при проведении `fillCashflow` ставит статью ДДС в документ, если там пусто (21 статья `its_r_cashflow_item`, у 55 моделей).
- `REGISTRIES` (ПП исх. с реестром MT102 `its_d_payord_out_reg`): если строки есть, их сумма = `amount`, суммы > 0, иначе 422.
- Нет ни одной суммы → 400/422 «Бухгалтерская модель не сформировала проводок».
- Коды строк и условий уникальны, заполнять по шаблону `<модель>_<nn>`, `<модель>_cond_<nn>` (при совпадении — `_<id>`), иначе перенос в YAML теряет ключ.

---

## 4. Движок `pkg/its_coa`

`local pkg = dbrequire("pkg/its_coa")`. Состояние на 20.11.2026: md5 `a68096bb2fe8319dbea352880ed1fd99`, 120 557 символов.

### 4.1. API

| Функция | Что делает |
|---|---|
| `pkg.postDocument({entity_code, entity_pk, posting_date?, user_id})` | Дата = `posting_date` или `pkg.docDate` (по `DOC_TYPES`); нет даты → 422 с названием поля; затем `doMdl` |
| `pkg.doMdl({entity_code \| entity_id, entity_pk, posting_date?, mdl_id? \| mdl_code?, user_id})` | Проведение. Возврат `{oper_id, mdl_id, mdl_code, posting_date, dep_id, posting_count, total_amount, …}`, `errText`, `errNum` |
| `pkg.cancelMdl({entity_code \| entity_id, entity_pk, user_id})` | Отмена: обратные проводки (Дт↔Кт, `is_reverse = 1`, `reverse_pst_id`) с датами исходных, `oper.status = reversed`. Уже сторнированные проводки пропускаются. 404 «Документ не проведен» |
| `pkg.previewMdl({entity_code, entity_pk})` | Пробный расчёт без записи: `{mdl_id, mdl_code, mdl_title, lines[{dr, cr, amount}], amount_ok, amount_error}`. PREPARE выполняется (с записью реквизитов!) на дату `coalesce(posting_date, current_date)` |
| `pkg.reversePst({pst_id \| trn_id, user_id})` | Сторно одной проводки датой исходной. 409: сторнирующая, операция отменена, уже сторнирована, закрытый период. Статус операции остаётся `posted` |
| `pkg.registerOper(params, lines)` | Низкоуровневая запись операции и проводок (используется закрытием месяца) |
| `pkg.isPosted(entity_id, entity_pk)` | Есть ли проведённая операция |
| `pkg.getAccId(coa_id, dep_id, uuid1, uuid2, uuid3)` | Найти/создать лицевой счёт (с проверкой субконто) |
| `pkg.getAccInfo(acc_id, on_date?)`, `pkg.getAccBal(acc_id, on_date?)` | Сведения/натуральное сальдо счёта на дату |
| `pkg.DOC_TYPES`, `pkg.docTypes()`, `pkg.docDate(entity_code, pk)` | Реестр документов для массового проведения и даты проведения |
| `pkg.findUnposted(p)`, `pkg.massCreate/massStep/massRetry` | Массовое проведение (REST its_coa_mass) |
| `pkg.closedUntil(dep_id)`, `pkg.closeCalc`, `pkg.closeMonth`, `pkg.closeCancel` | Закрытие месяца |
| `pkg.fillPayordOutRegStaff(payord_out_id)` | Работник в строках реестра ПП исх. по ИИН |
| `pkg.depTree(dep_id, with_children)` | Подразделение с дочерними |

### 4.2. Алгоритм `doMdl`

1. Дата проведения: `info.posting_date` или `current_date`.
2. Модель (`resolveMdl`): явная или по условиям/приоритету.
3. Проверка «уже проведён» (409).
4. Подразделение `resolveDep` (с записью в документ).
5. Закрытый период по дате операции (409), кроме самого `its_d_coa_close`.
6. **PREPARE-хук** сущности (`PREPARE[entity_code](entity_pk, dep_id, posting_date, mdl)`) + `fillCashflow`.
7. План (`buildPlan`): по строкам модели — счета, корреспонденции, субконто, суммы, лицевые счета. Ничего не записывается.
8. `checkPlanAmount`; закрытый период по датам всех строк плана (`pst_date` строк ТЧ).
9. `posting_date` пишется в документ (`update <entity> set posting_date=…` — колонка обязательна!).
10. `registerOper`: операция, проводки, движения; обновления остатков отсортированы по `acc_id` (нет дедлоков). Сбой записи → `error()` → откат всей транзакции.

`cancelMdl` проверяет закрытый период по **минимальной** дате проводок операции.

### 4.3. PREPARE-хуки

Таблица `PREPARE` в пакете, ключ — код сущности. Вызываются в `doMdl` и `previewMdl` до построения плана: проверяют документ (422 с понятным текстом) и дописывают в него вычисляемые суммы, которые потом читает модель.

| Сущность | Что делает |
|---|---|
| `its_d_payord_out` | Разбивка оплаты поставщику: по строкам ТЧ `its_d_payord_out_contract` — долг (кредитовое сальдо 3310 по договору) / аванс; без ТЧ — `debt_amount` / `advance_amount` шапки по поставщику; `fillPayordOutRegStaff`; `staff_id` по ИИН получателя для КНП 332/311; 422, если модель берёт реестр, а работник не найден |
| `its_d_payord_in` | `debt_amount` = min(дебетовое сальдо 1210, amount), `advance_amount` = остаток; договор = основной по цепочке ДС; разбивка по строкам ТЧ 866 |
| `its_d_avr` | Проверки (покупатель, amount > 0, НДС ≥ 0, amount_w_tax = amount + НДС); `coa_contract_id` = основной договор; `advance_offset` (зачёт аванса 3510 по договору), `advance_offset_free` (без договора) |
| `its_d_avr_in` | Контрагент обязателен; статусы DECLINED/REVOKED/CANCELED/CANCELLED/DELETED/FAILED → 422; подбор договора по номеру (буквы/цифры без регистра) + дате; `coa_id` = coalesce(АВР, договор, 7210); `advance_offset` = min(дебетовое сальдо 1710, сумма с НДС) |
| `its_d_hr_payroll` | Проверка строк и `staff_id`; `pst_accrual` = calc или salary+ОПВ+ВОСМС+ИПН |
| `its_r_f_asset` | Способ поступления обязателен; поставщик для purchase; стоимость из ТЧ, если пусто; `amo_amount`/`pst_date` строк ТЧ «Амортизация»; `amo_accum`, `residual`; зачёт аванса 1710 для purchase |

Помощники для сальдо расчётов (в SQL-тексте): `cliBalanceSql(coa_code, cli_expr, dep_expr, date_expr)`, `settleBalanceSql(coa_code, cli_expr, ctr_expr, dep_expr, date_expr)` — сумма `its_coa_trn` по счёту и контрагенту (и договору) по всем подразделениям той же организации на дату включительно.

Правила для хуков:
- Обновления — через `SqlExec2` (возвращает `errText, errNum`). `SqlInsert` добавляет `returning id` и падает на UPDATE без строк («no row found») и на `UPDATE … FROM` с колонкой id («ambiguous»).
- `?` в тексте SQL пакета писать как `Q` (`local Q = string.char(63)`), если пакет правится через `db_execute_sql`.
- В Lua нет `string.match`/`gsub` — используйте `pkg/strings`, `RegexpCheck`.

### 4.4. `DOC_TYPES` (реестр документов)

Поля: `code`, `title`, `short`, `ord`, `date` (SQL-выражение даты проведения от `d`), `date_title`, `company`, `num`, `amount`, `cli`, `extra` (доп. условие отбора).

| Документ | Дата проведения |
|---|---|
| `its_d_avr` | `avr_sign_dt` |
| `its_d_avr_in` | `coalesce(performed_date, awp_date)`, статусы отказа исключены |
| `its_d_hr_payroll` | последний рабочий день месяца периода по `its_calendar` (`its_day_type.code = 'work-day'`), иначе последний будний, иначе `oper_date` |
| `its_d_payord_in`, `its_d_payord_out` | `oper_date` |
| `its_r_f_asset` | `date_in`, иначе 1 января первого года ТЧ «Амортизация», иначе `posting_date`; только ОС со способом поступления |

Новый документ для массового проведения и страницы `its_coa_mass_post` — добавить элемент в `DOC_TYPES`.

---

## 5. Каталог моделей (20.11.2026)

### Исходящие платежи `its_d_payord_out` (денежный счёт всегда Кт 1030)

| Приор. | Модели | Условие | Дт |
|---|---|---|---|
| 400 | supplier_payment (4) / supplier_advance (5) | операция 3 / 4 | 3310 долг, 1710 аванс (по ТЧ договоров или по поставщику) |
| 350 | pension/social/osms/vosms_penalty, budget_penalty/fine | КНП 019,098 / 017 / 123 / 124 / 912 / 913 | Дт 7480 Кт 32xx/3190, затем Дт 32xx/3190 Кт 1030 |
| 310 | its_po_reg_* (67–74) | КНП как у базовой + есть строки реестра | по каждой строке реестра с работником: 3212, 3220, 3231, 3211, 3213, 3350 |
| 300 | pension, oppv, opvr / social_contribution, osms, vosms / single_payment, salary | 010, 015, 089 / 012, 121, 122 / 183,185 / 332 | 3220 / 3211, 3213, 3212 / 3231 / 3350 |
| 300 | loan_short, loan_long, loan_issue_short, loan_issue_long, lease | 421,429 / 423 / 411,419 / 413 / 424 | 3010 / 4010 / 1110 / 2010 / 3360 |
| 300 | dividends, deposit, own_transfer | 661 / 311,312,314,319 / 342,343 | 3040 / 1050 / 1021 |
| 300 | customer_refund, bank_fee, charity, guarantee, other_transfer | 780,880 / 842,120 / 119 / 171 / 190 | 3510 / 7210 / 7480 / 1274 / 3387 |
| 250 | ipn, kpn, social_tax, vat, excise, property, land, transport | 911 + КБК 1012 / 1011 / 1031 / 1051 / 106 / 1041 / 1043 / 1044 | 3120 / 3110 / 3150 / 3130 / 3140 / 3180 / 3160 / 3170 |
| 200 | other_tax | 911 без распознанного КБК | 3190 |
| 100 | supplier_default (33) | КНП 7xx, 8xx | как supplier_payment |

### Входящие платежи `its_d_payord_in` (Дт 1030)

| Приор. | Модели | Условие | Кт |
|---|---|---|---|
| 400 | customer_payment (2) / customer_advance (3) | операция 5 / 6 | 1210 долг, 3510 аванс / 3510 |
| 400 | revenue (43) | операция 17 «Оплата с признанием выручки (без АВР)» | Дт 1210 Кт 6010 (с НДС), Дт 6010 Кт 3130 (НДС), Дт 1030 Кт 1210 |
| 300 | supplier_refund, own_transfer, deposit_return, interest | 780,880 / 342,343 / 321,322,324,329 / 315–318,333 | 1710 / 1021 / 1050 / 6110 |
| 300 | loan_short, loan_long, loan_repay_short, loan_repay_long | 411,419 / 413 / 421,429 / 423 | 3010 / 4010 / 1110 / 2010 |
| 300 | dividends, grant, guarantee_return, other | 661 / 119 / 171 / 190 | 6120 / 6220 / 1274 / 3387 |
| 100 | customer_default (59) | КНП 7xx, 8xx без операции | как customer_payment |

С 14.11.2026 оплата по КНП 7/8 **не признаёт выручку** (её признаёт АВР), иначе выручка задваивается. Нет моделей для КНП 021 и 172.

### Реализация `its_d_avr` — модель its_d_avr_sale (58)
Дт 1210 Кт 6010 `amount` · Дт 1210 Кт 3130 `tax_amount` · Дт 3510 Кт 1210 `advance_offset` (по договору) · Дт 3510 Кт 1210 `advance_offset_free` (без договора). 1210: покупатель + договор (`coa_contract_id`, attr_opt).

### Входящий АВР `its_d_avr_in` — its_d_avr_in_purchase (60)
Дт [`coa_id`, по умолчанию 7210] Кт 3310 без НДС · Дт 1421 Кт 3310 НДС · Дт 3310 Кт 1710 зачёт аванса. 3310/1710: поставщик + договор.

### Начисление ЗП `its_d_hr_payroll` — its_d_hr_payroll_accrual (61), суммы из ТЧ 804
Дт 7210 (`coa_id`) Кт 3350 начислено (работник) · Дт 3350 Кт 3220 ОПВ · Дт 3350 Кт 3212 ВОСМС · Дт 3350 Кт 3120 ИПН · Дт 7210 Кт 3220/3211/3213 ОПВР, СО, ООСМС · Дт 7210 Кт 3150 СН одной проводкой из `sum_stax` шапки (пересчитывает БП перед проведением) · Дт 5610 Кт 3040 дивиденды · Дт 3040 Кт 3120 ИПН с дивидендов.

### Основные средства `its_r_f_asset` — модели 75–81 (условие `in_type_id` по коду)
| Способ поступления | Проводки |
|---|---|
| purchase | Дт 2410 Кт 3310 стоимость, Дт 1421 Кт 3310 НДС, Дт 3310 Кт 1710 зачёт аванса |
| opening | Дт 2410 Кт 5620; амортизация Дт 5620 Кт 2420 |
| construction / contribution / gratis / surplus | Дт 2410 Кт 2931 / 5030 / 6220 / 6290 |
| doc | только амортизация (стоимость уже на 2410 через входящий АВР) |
Амортизация (все, кроме opening): Дт 7210 (`coa_exp_id`) Кт 2420 по строкам ТЧ `its_r_f_asset_amo`: `amo_amount` = (остаток на начало, иначе стоимость для первой строки, иначе остаток прошлого года) − остаток на конец; незавершённый год = 0; дата `pst_date` = greatest(дата проведения, least(31.12 года, сегодня)).

### Закрытие месяца (без модели, через `registerOper`)
Дт 3130 Кт 1421 — зачёт НДС (меньшая из сумм); закрытие 6xxx и 7xxx на 5610.

---

## 6. Атрибуты документа для учёта

Минимум для проводимой сущности:
- `dep_id` (ref `deps`) и организация (`company_id` или `companies_bank_id`) — для подразделения;
- `posting_date` (date) — **обязательно**, `doMdl` пишет в неё дату;
- суммы, которые читает модель, — обычными колонками (не формулами);
- счета «из реквизита» — ref `its_r_coa` (`coa_id`, `coa_exp_id` …);
- аналитика — ref на справочники из таблицы субконто (`its_cli`, `its_cli_contract`, `its_staff` …);
- служебные расчётные поля PREPARE (`advance_offset`, `debt_amount`, `advance_amount`, `amo_accum` …);
- `cashflow_item_id`, если документ денежный.

Для ТЧ, по которой строятся проводки: колонки сумм и аналитики; опционально `pst_date` — своя дата проводки.

После добавления атрибутов страницы карточки нужно перегенерировать (`dbrequire("pkg/page").genPages(entity_id, 1)` или «Опубликовать» на сущности из интерфейса). Публикация из крона (`BPMSStartProcess2("entity_publish", …)`) падает на шаге колонок фильтра (`SAVEPOINT can only be used in transaction blocks`), хотя страницы при этом пересоздаются.

---

## 7. Сторно и отмена

- **Отмена проведения** (`cancelMdl`): операция `status = reversed`, на каждую проводку — зеркальная с `is_reverse = 1`, `reverse_pst_id`, той же датой. Документ можно провести снова (новая операция).
- **Сторно проводки** (`reversePst`, БП `its_coa_trn_reverse`, кнопка на `its_coa_trn`, роль its_accountant): одна зеркальная проводка, операция остаётся `posted`; последующая отмена документа сторнирует только оставшиеся.
- Группы движения в отчётах:
  - `rev` — операция `reversed` (прямые и обратные движения отменённого проведения);
  - `sto` — `is_reverse = 1` или проводка сторнирована (есть `its_coa_pst` с `reverse_pst_id` = pst и `is_reverse = 1`);
  - `act` — остальные.
- По умолчанию отчёты показывают только `act` (баланс, выписка, карточка счёта, вкладка «Проводки» договора). Пары `rev`/`sto` дают 0 в ту же дату — сальдо на начало/конец не меняется, только обороты. Если включён только один вид — обороты и сальдо искажаются (страница предупреждает).
- Проверка «сторно существует» — `exists` по индексу `its_coa_pst.reverse_pst_id`; join с `distinct`-CTE работал минуты.

---

## 8. Процессы проведения (кнопки)

### 8.1. Набор на документ

| Процесс | Тип | Вход/выход | Назначение |
|---|---|---|---|
| `<entity>_posting` «Провести» | entityActionByID (12) | `id` (in, rq), `last_error` (**out**) | `pkg.postDocument({entity_code, entity_pk = var.id, user_id = tonumber(sys.user_id)})` |
| `<entity>_posting_nonoperational` «Провести неоперативно» | entityActionByID | то же | `doMdl` с заполненной `posting_date` (ПП, АВР; у ЗП отключён) |
| `<entity>_cancel_posting` «Отменить проведение» | entityActionByID | то же | `pkg.cancelMdl` |
| `<entity>_mass_post` «Провести (массово)» | массовое действие (3), `is_action_single_only = 0` | `url` (in, rq), `last_error` (out) | `Query(var.url, sys.user_id)` → по каждой записи одиночный БП |
| `<entity>_mass_cancel` «Отменить проведение (массово)» | то же | то же | то же для отмены |

`condition_action`:
```sql
-- «Провести»
not exists (select 1 from its_coa_oper o where o.entity_id=(select id from entities where code='<entity>') and o.entity_pk=main.id and o.status='posted' and coalesce(o."in$trash",0)=0)
-- «Отменить проведение» — exists (...) с тем же условием
```

Массовый скрипт: для каждой записи проверяет `its_coa_oper.status = 'posted'`, пропускает уже проведённые (при отмене — непроведённые), вызывает `BPMSStartProcess2(PROCESS, sys.user_id, {id = id}, sys.instance_id)`, перепроверяет состояние, собирает итог «Проведено: N, пропущено: M, ошибок: K» и до 30 строк «ID …: текст» в `last_error`. Причину он берёт из `rv.last_error` дочернего процесса — **`last_error` одиночного БП должна быть выходной** (`is_output = 1`), иначе в итоге будет только «документ не проведён». Все документы выделения обрабатываются в одной транзакции БП, ошибка одного не откатывает остальные.

Роли: its_user, its_accountant, helpdesk_admin (`bp_process_roles`, `can_run = 1`; вставлять с `not exists`). Ранние массовые процессы (ПП, ЗП, АВР) — только its_accountant.

Существующие (20.11.2026): its_d_payord_out (2687, 2691, 2829, 2830 + nonoperational), its_d_payord_in (2804, 2806, 2831, 2832 + nonoperational), its_d_avr (2808–2810, 2835, 2836), its_d_avr_in (2821–2823), its_d_hr_payroll (2824, 2826, 2833, 2834), its_r_f_asset (2844–2847), its_coa_trn_reverse (2837), its_coa_accdetails_showbal (2688).

Перед проведением в скрипте БП можно подготовить данные, которые модель читает (пример — `its_d_hr_payroll_posting` пересчитывает `sum_stax`). REST `its_coa_mass` вызывает `pkg.postDocument` напрямую, минуя БП, — такая подготовка там не выполнится; её место — PREPARE-хук.

### 8.2. Клонирование процессов (рецепт)

`create_bp_process` из `tpl_entity_action_by_id` «забирает» точки шаблона себе (коды точек и стрелок глобально уникальны) и иногда возвращает «no row found» без ролей и переменных. Надёжный путь — клон готового процесса одним `DO`-блоком:

1. `insert into bp_processes` копией всех колонок, кроме `id, code, title, action_entity_id, module_id, condition_action, sys$uuid, created_at, updated_at, wf_id_var_id, wf_status_attr_id`; `condition_action` — `replace` кода сущности.
2. `bp_points` и `bp_sequence_flows`: новые коды `split_part(code,'_',1) || '_' || gen_random_uuid()`; в `diagram` и `svg` заменить каждый старый код новым.
3. `bp_process_vars` — копия (колонки без префикса таблицы дают «column reference "code" is ambiguous» при join — выбирать через `where process_id = (select id …)`).
4. Роли — `bp_process_roles`.
5. Таблица переменных: `create table "i$<новый>" (like "i$<старый>" including all)`.
6. Скрипты — `update bp_points set script_txt = …`.
7. **`update_bp_process_diagram`** с диаграммой нового процесса — без этого процесс не запускается (экземпляр висит). `name` у `scriptTask` в диаграмме становится `title` точки.
8. `patch_out_add_objects` с `bp_processes` (клон через SQL сам в патч не попадает).

---

## 9. Массовое проведение и закрытие месяца

- REST `its_coa_mass` (страница `its_coa_mass_post`): GET `types`, `jobs`, `preview`, `job`; POST `create` (mode `check` | `post`), `step`, `retry`. Задание в `its_coa_mass` / `its_coa_mass_n`, каждый документ — отдельная транзакция, ошибка не останавливает задание. Список типов — `pkg.DOC_TYPES`.
- Закрытие месяца: сущность `its_d_coa_close`, REST `its_coa_close` (GET `calc`, `history`, `postings`; POST `close`, `cancel`; роли its_accountant, helpdesk_admin), страница `its_coa_month_close` (`?dep_id=&m=YYYY-MM`). Проверки: месяц уже закрыт (ошибка), непроведённые документы за месяц (предупреждение), сходимость оборотов, сальдо не на своей стороне. Отменить можно только последнее закрытие.

---

## 10. Регистры и отчёты

Все отчёты — query + JIT-страница; Excel — FODS-шаблон (`exp_tmp_ftypes` `fods`, «Excel (FODS → xlsx)»). Центр отчётов `its_hr_reports`, группа «Финансы» (`report_page`, роли its_user / its_accountant).

| Отчёт | Объекты | Параметры query |
|---|---|---|
| Бухгалтерский баланс | query `its_coa_balance` (24602), `its_coa_balance_form` (24648), REST `its_coa_balance_drilldown`, страница `its_coa_balance` (15200), шаблон `its_coa_balance$fods` (1028) | dep_id, с, по, показывать сторно, показывать отмены (все 5 передавать) |
| Выписка по счёту | REST `its_coa_acc_statement`, страница `its_coa_acc_statement` (15197) | `id` лицевого счёта, `date_from/date_to` (пусто — весь период), `show_storno`, `show_cancel` |
| ОСВ | query `its_coa_osv`, страница `its_hr_report_osv` | dep, с, по, дочерние, уровень 0/1/2, маска счёта; CSV |
| Журнал хозяйственных операций | query `its_coa_journal` (24626), REST `its_coa_journal_xlsx`, страница `its_hr_report_journal`, шаблон `its_coa_journal$fods` | dep, с, по, дочерние, маска счёта, скрыть сторнированные (6) |
| Карточка счёта | query `its_coa_acc_card` (24625), REST `its_coa_acc_card_xlsx`, страница `its_hr_report_acc_card`, шаблон `its_coa_acc_card$fods` | dep, coa_id (с дочерними счетами), с, по, дочерние подр., отменённые, сторно (7) |
| ОПиУ | query `its_coa_pnl`, `its_coa_pnl_form`, REST `its_coa_pnl_xlsx`, страница `its_hr_report_pnl`, шаблон `its_coa_pnl$fods` | dep, с, по, дочерние; предыдущий период = −1 год |
| ОДДС (прямой метод) | query `its_coa_cf_form`, REST `its_coa_cf_xlsx`, страница `its_hr_report_cf`, шаблон `its_coa_cf$fods` | dep, с, по, дочерние |
| Движение денег по статьям ДДС | query `its_coa_cf_items`, страница `its_hr_report_cf_items` | |
| Акт сверки | query `its_coa_recon_act` (24634), `its_coa_recon_act_lookup`, страница `its_hr_report_recon_act`, шаблоны `its_coa_recon_act$fods`, `$pdf`, template_funcs `fods_image_by_file_id` | организация, контрагент, договоры через запятую, с, по, зеркально, печать (7) |
| Незакрытые договоры / без оплат | query `its_cli_contract_unclosed`, страница `its_hr_report_contracts_unclosed` | dep, на дату, дочерние |
| Проводки договора | detail_query `its_coa_pst_contract`, ui_block_tpl 19, вкладка на `its_cli_contractdetails` | |
| Проводки документа | ui_block_tpl `its_coa_pst` (18) + detail_query `its_coa_pst` в details документа | |
| Реестр ЭСФ, ФНО 300 | `its_d_esf_report(_form)`, `its_d_esf_report$fods`; БП `its_fno_gen_300_json` | |

Методические карты (в SQL отчётов):
- **ОПиУ**: 010 — 60xx (6010 − 6020/6030, НДС уменьшает), 011 — 70xx, 013/014 — 71xx/72xx, 015 — 74xx, 016 — 62xx, 021/022 — 61xx/73xx, 023 — 64xx/76xx, 101 — 77xx, 201 — 63xx/75xx, 400 — 5520/5530/5540/5560. Обороты с 56xx/57xx (закрытие) исключаются; счета вне карты — предупреждение (line_code 999).
- **ОДДС**: денежные — 10xx, кроме 1000 и 1090; строка по корр. счёту (самый длинный префикс), не найдено → 016/027 с пометкой «не сопоставлен»; переводы между своими денежными счетами исключены; сторно уменьшает исходный поток.
- **Баланс**: сторона по `is_passive`, раздел — первая цифра кода, группа — XX00; расшифровка по субконто — строки `kind = sub`.
- **Аналитика по uuid**: имя ищется по `its_r_coa_n.entity_id` (`query_to_xml` по `sys$uuid`), в старых запросах — перечнем таблиц (its_cli, its_cli_contract, companies_bank, deps, its_r_cashflow_item, its_staff, its_d_payord_out/in, its_d_avr, its_d_avr_in). Новый вид аналитики — дописать ветку.

Документ-основание проводки: `its_coa_pst.entity_id/entity_pk` → `entities.detail_page_id` → ссылка `/<detail_page>/<pk>`; номер/дата — из полей документа (ПП `document_number`/`oper_date`, АВР `avr_num_ext`/`avr_dt`, вх. АВР `num`/`awp_date`, иначе `its_req.docnum/doc_at`).

---

## 11. Печатные формы FODS

- Тип `fods`: Go-шаблон по `.items` → Flat ODS → `generateXlsxbyFODS` (soffice) → xlsx с настоящими формулами (`of:=SUBTOTAL(9;[.J8:.J20])`), outline-группами, повтором шапки при печати.
- Query формы возвращает строки уже с `row_no` (номер строки листа), `kind`/`st` (стиль), формулами `f_*` и флагами групп `*_open/*_close`.
- Вызов: REST (`Query(...)` → `generateXlsxbyFODS('EXP_TEMPLATE', '<имя>.xlsx', template, {user_id, items = array(rows)}, user_id)` → `Commit()` → `{success, file_uuid, file_url}`) или БП `exp_template` (`bpRun('exp_template', {url, entity_id, template_code})`, ветка fods есть в шагах Query и Detail). PDF — тот же текст шаблона с ftype `pdf` (soffice распознаёт Flat ODS).
- **Шаблон проверяется при сохранении/импорте на пустом `.items`**: `index .items 0` вне `range` → «slice index out of range» → Upsert падает. Шапку брать внутри `{{if .items}}…{{end}}` или `{{range $i, $h := .items}}{{if eq $i 0}}…{{end}}{{end}}`.
- `ParseTemplate(s, arr, userId)` — третий аргумент обязателен.
- Два `generateXlsxbyFODS` подряд (≈2 с) — второй иногда «exit status 1»; повтор проходит.
- Закрепление областей при headless-конвертации не сохраняется; длинный текст — объединять ячейки на все колонки с переносом.
- Шаблоны с `{{`, `}}`, `?` в `db_execute_sql` не передавать (MCP-мост отвечает «Информация не найдена» и ничего не делает): `convert_from(decode('<b64>','base64'),'UTF8')`, частями с охраной `length(...)`, сверять md5; либо кроном (Lua long string / gzip + `GunZipString`).

---

## 12. Как добавить проведение новому документу (чек-лист)

1. Изучить сущность: атрибуты, ТЧ (`table_parts`), формулы (модель их не читает), данные.
2. Добавить атрибуты учёта (раздел 6): `dep_id`, организация, `posting_date`, суммы, счета, аналитика, служебные поля.
3. Проверить корреспонденции `its_r_coa_correct_dtcr`, недостающие добавить с `code = <Дт>_<Кт>`.
4. Создать модели (`its_r_coa_mdl` + `_cond` + `_n`) SQL-блоком: коды строк/условий по шаблону, `check_amount`, приоритеты без равных конкурентов, `cashflow_item_id` для денежных.
5. Если нужны вычисления/проверки — PREPARE-хук в `pkg/its_coa` (бэкап `create table bak_luas_its_coa_<дата> as select * from luas where code='pkg/its_coa'`; правка точечными `replace` по якорям с проверкой числа вхождений).
6. Добавить тип в `pkg.DOC_TYPES` (дата проведения, номер, сумма, контрагент, `extra`).
7. Клонировать 4 процесса (раздел 8.2), `last_error` — выходная, роли, `update_bp_process_diagram`, таблицы `i$`.
8. Перегенерировать страницы сущности (новые поля, колонки ТЧ).
9. При желании — вкладка «Проводки» (ui_block_tpl `its_coa_pst` + detail_query `its_coa_pst` с кодом сущности).
10. Протестировать кроном (раздел 13), убрать следы, собрать патч, записать заметку в проект.

---

## 13. Тестирование без браузера

- Временный крон с guard-строкой во временной таблице (`tmp_<x>_test`): первая строка скрипта — «если есть строка `start` — return», затем запись `start`, затем шаги с логом `insert into tmp(step,res) values (Q,Q)`.
- Запуск процессов: `pcall(BPMSStartProcess2, code, 1, vars, nil)` → `rv.last_error`. Нельзя присваивать `sys.user_id` и вызывать `Commit()` до старта БП — откатывается без сообщения. `BPMSStartProcess` из крона с `DMLI` внутри падает «SAVEPOINT can only be used in transaction blocks» (проведение через pkg работает).
- **Планировщик держит крон в памяти**: `update crontab set is_active=0` не останавливает его. Выключать через `create_crontab` с тем же кодом и `is_active = false`, затем удалить строку `crontab` и её запись в `patch_out_obj` (крон, созданный через MCP, попадает в открытый патч), удалить tmp-таблицу.
- `Rollback()` в кроне не откатывает записи pkg — тестовые проведения отменять `cancelMdl`, данные документов возвращать из бэкапа.
- Что проверять: проводки (`its_coa_pst` с кодами счетов и датами), операция posted/reversed, суммы = документу, Дт = Кт, чистое сальдо по тестовым операциям после отмены = 0, тексты ошибок (нет обязательного реквизита, повторное проведение 409, отмена непроведённого 404), массовые — пропуски и причины.
- Query с `?` из `db_execute_sql`: `query_to_xml(regexp_replace(sql_text, chr(92)||chr(63), quote_literal(p)), …)` по одному параметру; `EXPLAIN` — только во временной volatile-функции.
- Тело REST можно выполнить в кроне через `load(function)` с подменой `request`.

---

## 14. Ловушки

- `db_execute_sql`: SQL в base64; `?` в литералах — `chr(63)`; `{{`/`}}` — `chr(123)/chr(125)` или base64; из нескольких операторов возвращается только первый результат (сводить в один `select` или `DO`-блок); `limit` в конце `union all` действует на весь union.
- Атрибут, добавленный `add_entity_attr`, появляется колонкой сразу, но на форме — только после перегенерации страниц.
- `EntityValueById` на сущностях типа `its_req` из временного БП под user 1 может вернуть пусто даже для обычных полей — суммы для модели держать в колонках и проверять предпросмотром.
- `its_r_knp.code` с ведущим пробелом — всегда `trim`.
- `x OR NULL = NULL`: условия «закрыт по дате» оборачивать `coalesce(…, false)`.
- Патч, уже выгруженный в файл, не принимает объекты — `patch_out_create` новый. Объекты, изменённые SQL-запросом (модели, строки, условия, корреспонденции, справочники, `luas`, `bp_processes`), добавлять `patch_out_add_objects` явно.
- `_mcp_method_create_query`: title > 100 символов откатывает всё; срезает завершающий перевод строки.
- `previewMdl` запускает PREPARE и пишет вычисляемые поля в документ; дата — текущая, если `posting_date` пуст.
- Названия видов субконто плана счетов (из 1С) могут не совпадать с фактической аналитикой моделей (у 6010 в плане «Доходы…», по факту контрагент) — в шапки отчётов не выводить.

---

## 15. Перенос на прод

Порядок патчей учёта: 81 (`coa-db-ledger-20261113`) → 83 → 110 (поставщики) → 115 (договор в субконто АВР) → 123 (ЗП, ДДС, массовое, закрытие, ОСВ, реестр) → 130/132/133/134/136/139 (выписка, баланс, карточка) → 135 (ОС). Отчёты: 98 (тип fods и ОПиУ — раньше любых FODS-шаблонов) → 112, 113, 114/116, 122, 124, 126, 131.

Не переносится патчем (выполнить вручную): уникальные индексы `ux_its_coa_oper_posted`, `ux_its_coa_bal_acc_nn`; `its_d_stmt_dtnumcompanyidx` (по `document_number, oper_date, company_id, dt`); заполнение `companies.main_dep_id`, `dep_id` документов; скрипт `its_r_coa_n` (колонка `code`, `entity_id` по названию вида); `ANALYZE its_coa_acc` (устаревшая статистика замедляла баланс в 15 раз).

---

## 16. Не покрыто / открытые вопросы (20.11.2026)

- Нет моделей для КНП 021 и 172 во входящих платежах.
- Модели its_d_payord_in пишут на 1210/3510 договор только при указанном договоре; платежи без договора остаются на контрагенте (встречные сальдо по договорам). Вариант — автозаполнение ТЧ договоров по долгам (FIFO).
- Начисление ЗП кредитует 3220/3211/3212/3213 без работника, выплаты по реестру дебетуют по работнику — по субконто остатки не сворачиваются.
- Нет ежемесячного документа амортизации ОС и выбытия ОС; амортизация идёт годовыми суммами из ТЧ при проведении карточки.
- Валютные платежи, КНП 322 в исходящих — не реализовано.
- Ручных сторно на стенде нет — группа `sto` на реальных данных не проверена.
- Страницы отчётов в браузере в основном не проверялись (проверены query, REST и файлы xlsx).
