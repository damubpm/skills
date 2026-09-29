# DamuBPM — экранные формы для User Task

[← Главная](index.html)

Практическое руководство: JIT JavaScript, Angular HTML, переменные процесса, валидация, табличные части, облачная ЭЦП и автогенерация форм.

## Содержание

- [01. Назначение и подключение](#start)
- [02. Контекст и переменные](#contract)
- [03. Жизненный цикл формы](#lifecycle)
- [04. Базовая форма: JavaScript](#base)
- [05. Базовая форма: HTML](#html)
- [06. Расширение: облачная ЭЦП](#sign)
- [07. Табличные части через JIT-PAGE](#tables)
- [08. Что исправить в исходном примере](#issues)
- [09. Проверка перед использованием](#check)
- [10. Автогенерация User Task](#generation)
- [11. Поля и HTML автогенерируемой формы](#generated-fields)
- [12. Проверки сверх автогенерации](#generated-validation)
- [13. UI-блоки и проверка результата генерации](#generation-review)
- [14. Ссылка на главную](#home)

<a id="start"></a>

## 01. Назначение и подключение

Экранная форма User Task показывает переменные текущей задачи, принимает ввод пользователя и передаёт результат в бизнес-процесс через `appComponent.runTask(data.task, payload)`. Форма состоит из JavaScript-логики JIT и Angular HTML-шаблона.

1. В модели БП выберите нужный User Task и настройте исполнителей.
2. Определите переменные точки: что поступает на экран и что форма возвращает.
3. В настройках пользовательской формы разместите JavaScript и HTML в соответствующих полях. Названия вкладок зависят от версии DamuBPM.
4. Сохраните форму и опубликуйте изменения процесса по принятому в вашей системе порядку.
5. Запустите тестовый экземпляр и откройте задачу от имени назначенного исполнителя.

> Основа руководства — предоставленный код формы облачной ЭЦП и справочники DamuBPM. Примеры требуют проверки в вашей версии runtime; работающая система при подготовке инструкции не изменялась.

<a id="contract"></a>

## 02. Контекст и переменные

| Объект | Назначение |
| --- | --- |
| `data.processTitle` | Заголовок процесса. В примере записывается в `appComponent.user_task_title`. |
| `data.vars` | Массив объектов `{ name, value }`. Значения сохраняются в `detail`. |
| `data.task` | Идентификатор текущей задачи. Не подменяйте его ID процесса или кодом точки. |
| `detail` | Локальные данные для отображения. Изменение объекта само по себе не завершает задачу. |
| `form / f` | Reactive Form и её контролы. `f` — сокращение для `form.controls`. |
| `runTask(task, payload)` | Передача явно перечисленных выходных значений и продолжение текущей задачи. |
| `resp.task` | Следующая задача, если она возвращена сервером. |

| Переменная примера | Вход на экран | Возврат из формы |
| --- | --- | --- |
| `txt` | Текст для отображения или подписания | `txt: this.f.txt.value` |
| `cms` | Может быть пустой | Подпись после успешного вызова сервиса |
| `filenames` | Необязательная подпись со списком файлов | В исходном payload не передаётся |

Имена контролов, ключи payload и переменные БП должны совпадать. Обязательность и типы настраиваются и в модели процесса. `Validators.required` в браузере не заменяет проверку сервера.

<a id="lifecycle"></a>

## 03. Жизненный цикл формы

1. **Создание:** `GenClass extends vm.constructor` получает сервисы runtime; отдельный `@Component` не нужен.
2. **Инициализация:** `ngOnInit()` читает `data.vars`, создаёт контролы и подписки.
3. **Редактирование:** HTML связан с контролами через `[formGroup]` и `formControlName`.
4. **Отправка:** проверка полей, подготовка табличных частей, сбор payload, вызов `runTask`.
5. **Продолжение:** при включённом шаблоне задач и наличии `resp.task` обновляется query-параметр `task`.
6. **Уничтожение:** `ngOnDestroy()` освобождает подписки.

Если маршрут переиспользует компонент, проверьте, что runtime пересоздаёт форму при смене `task`. Иначе останутся прежние `data`, контролы и флаг `completed`.

<a id="base"></a>

## 04. Базовая форма: JavaScript

Самостоятельный пример для обычного User Task с полями `txt` и `cms`. Здесь `txt` обязателен, а подпись ещё не требуется. Для формы ЭЦП примените изменения из раздела 06.

```javascript
const vm = this;
const { first } = rxjs;
const { Validators } = forms;

return class GenClass extends vm.constructor {
    detail = {};
    submitted = false;
    busy = false;
    completed = false;
    loadingText = '';
    error_text = '';
    subscriptions = [];
    form = this.formBuilder.group({});

    get f() { return this.form.controls; }

    ngOnInit() {
        this.appComponent.user_task_title = data.processTitle;
        for (const v of (data.vars || [])) {
            this.detail[v.name] = v.value;
        }
        this.form.addControl('txt', this.formBuilder.control(
            this.detail.txt == null ? '' : this.detail.txt,
            Validators.required
        ));
        this.form.addControl('cms', this.formBuilder.control(
            this.detail.cms == null ? '' : this.detail.cms
        ));
        for (const name of ['txt', 'cms']) {
            this.subscriptions.push(this.f[name].valueChanges.subscribe(value => {
                this.detail[name] = value;
            }));
        }
    }

    next() {
        if (this.busy || this.completed) return;
        this.submitted = true;
        this.error_text = '';
        for (const control of Object.values(this.f)) {
            control.markAsTouched();
            control.markAsDirty();
        }
        if (this.form.invalid || this.form.pending) return;
        if (!data.task) {
            this.error_text = 'Не указан идентификатор задачи';
            return;
        }
        this.busy = true;
        this.loadingText = 'Отправка данных…';
        this.subscriptions.push(
            this.appComponent.runTask(data.task, {
                txt: this.f.txt.value,
                cms: this.f.cms.value
            }).pipe(first()).subscribe({
                next: resp => {
                    this.busy = false;
                    this.loadingText = '';
                    // Контракт бизнес-ошибок уточните для вашей версии runTask.
                    if (!resp || resp.ok === false || resp.ok === 0 || resp.ok === '0') {
                        this.error_text = 'Задача не выполнена. Проверьте ответ сервера.';
                        return;
                    }
                    this.completed = true;
                    if (this.appComponent.use_user_task_template && resp.task) {
                        this.router.navigate([], {
                            relativeTo: this.route,
                            queryParams: { task: resp.task },
                            queryParamsHandling: 'merge'
                        });
                    }
                },
                error: () => {
                    this.busy = false;
                    this.loadingText = '';
                    this.error_text = 'Не удалось подтвердить выполнение. Проверьте состояние задачи перед повтором.';
                }
            })
        );
    }

    ngOnDestroy() {
        for (const sub of this.subscriptions) sub.unsubscribe();
    }
}
```

> В исходном примере показан `resp.task`, но не полный контракт ошибок `runTask`. Проверка `resp.ok` ниже учитывает распространённый отрицательный ответ; перед внедрением согласуйте её с фактическим ответом вашей версии. HTTP 200 сам по себе не доказывает успешное выполнение БП.

<a id="html"></a>

## 05. Базовая форма: HTML

```html
<form [formGroup]="form" (ngSubmit)="next()">
    <label for="task-txt">Текст</label>
    <textarea id="task-txt" class="form-control" rows="5"
        formControlName="txt" [readOnly]="busy || completed"></textarea>
    <div class="text-danger"
        *ngIf="f.txt.invalid && (f.txt.touched || submitted)">
        Заполните текст.
    </div>

    <div role="status" aria-live="polite" *ngIf="loadingText">
        {{ loadingText }}
    </div>
    <div class="text-danger" role="alert" *ngIf="error_text">
        {{ error_text }}
    </div>
    <div *ngIf="completed">Данные задачи отправлены.</div>

    <button type="submit" class="btn btn-primary"
        [disabled]="busy || completed || form.pending">
        Дальше
    </button>
</form>
```

`type="submit"` вызывает `next()` через `ngSubmit`. Кнопкам вспомогательных действий задавайте `type="button"`. Не используйте одновременно `ngModel` и `formControlName` для одного поля.

В JIT вставляйте только шаблон формы, без `<html>`, `<head>` и `<body>`. Полный HTML-документ нужен для этой инструкции.

<a id="sign"></a>

## 06. Расширение: облачная ЭЦП

Предоставленный сервис `users_set_eds_profile_cloud_rawsign` принимает `{ raw, key, pwd }`. Успех в исходном контракте: `error_code = 0`, `response.status = 0` и непустой `response.result.cms`.

1. Добавьте поля и метод из фрагмента ниже в базовый класс.
2. Блок инициализации вставьте в конец `ngOnInit()`, после создания контролов.
3. В первой проверке `next()` добавьте `this.signing`: `if (this.signing || this.busy || this.completed) return;`.
4. Замените базовый HTML шаблоном подписи ниже.

В этом варианте пользователь нажимает «Подписать и продолжить», затем полученный CMS передаётся в `next()`. Автоподписание при открытии из исходного `ngOnInit()` намеренно не включено: выбирайте этот режим явно по сценарию процесса.

```javascript
// Добавьте поля в GenClass.
cloud_sign_saved_key = '';
cloud_sign_saved_password = '';
signing = false;

// В конце ngOnInit(), после создания контролов:
this.cloud_sign_saved_key = localStorage.getItem('cloud_sign_saved_key') || '';
this.cloud_sign_saved_password = localStorage.getItem('cloud_sign_saved_password') || '';
this.f.cms.setValidators(Validators.required);
this.f.cms.updateValueAndValidity();
this.subscriptions.push(this.f.txt.valueChanges.subscribe(() => {
    this.f.cms.setValue(''); // Изменённый текст требует новой подписи.
}));

// Добавьте метод в GenClass.
signCloud() {
    if (this.signing || this.busy || this.completed) return;
    this.error_text = '';
    this.f.txt.markAsTouched();
    if (this.f.txt.invalid) return;
    if (!this.cloud_sign_saved_key || !this.cloud_sign_saved_password) {
        this.error_text = 'Настройте профиль ЭЦП в разделе «Мои настройки».';
        return;
    }
    const raw = this.f.txt.value;
    this.f.cms.setValue('');
    this.signing = true;
    this.loadingText = 'Получение подписи…';
    this.subscriptions.push(this.dbQueryService.restapiPost(
        'users_set_eds_profile_cloud_rawsign',
        { raw, key: this.cloud_sign_saved_key, pwd: this.cloud_sign_saved_password },
        true
    ).pipe(first()).subscribe({
        next: sign => {
            this.signing = false;
            this.loadingText = '';
            const response = sign && sign.response;
            const cms = response && response.result && response.result.cms;
            if (sign && String(sign.error_code) === '0' && response &&
                String(response.status) === '0' && cms) {
                if (this.f.txt.value !== raw) {
                    this.error_text = 'Текст изменился. Подпишите его повторно.';
                    return;
                }
                this.f.cms.setValue(cms);
                this.next();
            } else {
                this.error_text = response && typeof response.message === 'string'
                    ? response.message : 'Не удалось получить облачную ЭЦП';
            }
        },
        error: () => {
            this.signing = false;
            this.loadingText = '';
            this.error_text = 'Ошибка обращения к сервису подписи';
        }
    }));
}
```

```html
<form [formGroup]="form">
    <p class="text-danger"
       *ngIf="!cloud_sign_saved_key || !cloud_sign_saved_password">
        Настройте профиль ЭЦП: Мои настройки → Профиль ЭЦП.
    </p>
    <label for="sign-txt">Подписываемый текст</label>
    <textarea id="sign-txt" class="form-control" rows="5"
        formControlName="txt" readonly></textarea>
    <div class="text-danger" *ngIf="f.txt.invalid && f.txt.touched">
        Отсутствует текст для подписания.
    </div>
    <p *ngIf="detail.filenames">{{ detail.filenames }}</p>
    <p role="status" aria-live="polite" *ngIf="loadingText">{{ loadingText }}</p>
    <p class="text-danger" role="alert" *ngIf="error_text">{{ error_text }}</p>
    <button type="button" class="btn btn-primary" (click)="signCloud()"
        [disabled]="signing || busy || completed || !cloud_sign_saved_key || !cloud_sign_saved_password">
        Подписать и продолжить
    </button>
    <p *ngIf="f.cms.value && !completed">Подпись получена.</p>
    <p *ngIf="completed">Данные задачи отправлены.</p>
</form>
```

> Чтение ключа и пароля из `localStorage` показано для совместимости с предоставленным примером. Эти значения доступны скриптам страницы. Для рабочего решения используйте согласованный механизм защищённого профиля ЭЦП; не выводите ключ, пароль и содержимое подписи в консоль. Сервер должен проверять подпись, подписанный текст и полномочия подписанта.

Если подпись получена, но отправка задачи завершилась сетевой ошибкой, сначала проверьте состояние задачи. Повторный вызов не должен считаться безопасным автоматически: сервер мог уже выполнить первый запрос.

<a id="tables"></a>

## 07. Табличные части через JIT-PAGE

Исходный обработчик получает экземпляр дочерней страницы через `$event`, код табличной части и имя переменной процесса. Конкретное имя компонента и его события зависит от JIT-PAGE вашей системы; в предоставленном HTML этой привязки нет.

```javascript
table_parts = [];

startTablePartEdit(jit_page, table_part_code, variable) {
    const entry = { jit_page, table_part_code, variable };
    const index = this.table_parts.findIndex(x => x.variable === variable);
    if (index >= 0) this.table_parts[index] = entry;
    else this.table_parts.push(entry);
}

// В ngOnInit: переменная items_json должна быть объявлена в БП.
this.form.addControl('items_json', this.formBuilder.control('[]'));

// В next(), ДО проверки this.form.invalid:
for (const part of this.table_parts) {
    const control = this.f[part.variable];
    const details = part.jit_page && part.jit_page.details;
    const rows = details && details[part.table_part_code];
    if (!control || !Array.isArray(rows)) {
        this.error_text = 'Табличная часть ещё не готова';
        return;
    }
    control.setValue(JSON.stringify(rows));
}

// Добавьте в объект payload вызова runTask:
items_json: this.f.items_json.value
```

Для JSON-строки в БП используйте согласованный строковый тип. Если процесс ожидает структуру, формат передачи нужно изменить по его контракту. Валидация обязательных ячеек дочерней таблицы выполняется отдельно: непустая JSON-строка не подтверждает корректность строк. Убедитесь, что все ожидаемые табличные части зарегистрированы до отправки.

<a id="issues"></a>

## 08. Что исправить в исходном примере

| Наблюдение | Решение |
| --- | --- |
| `this.f.cms.setValue(...)`, но в HTML `*ngIf="cms"` | Использовать `f.cms.value` либо явно синхронизировать поле `cms`. |
| `*ngIf="subjectDn && loadingText"` | `subjectDn` в данном коде не задаётся. Показывать индикатор по `loadingText`. |
| Обращение к `sign.response.message` без проверки | Сначала проверить наличие `sign` и `response`. |
| Нет обработчиков HTTP-ошибок | Добавить `error` в подписки и сбрасывать занятость/индикатор. |
| Нет блокировки повторных нажатий | Использовать `signing`, `busy`, `completed` и `disabled`. |
| `Validators` импортирован, но не применяется | Указать валидаторы при создании контролов; для ЭЦП требовать `cms`. |
| Таблицы записываются после проверки формы | Синхронизировать таблицы до итоговой проверки и включать их переменные в payload. |
| Подписка `eds_sub` не освобождается | Отписаться при уничтожении; для варианта только с REST-подписью эта подписка не нужна. |
| Подпись запускается в `ngOnInit()` | Учитывать повторное открытие формы. Для ручного подтверждения оставить запуск только по кнопке. |
| Неиспользуемые поля и импорты | `moment`, `take`, `distinctUntilChanged`, `FormControl`, `all_cms` и прочее удалять только после проверки подключаемых UI-блоков. |
| `fileChange()` читает первый файл без проверки | Проверять наличие файла и обрабатывать ошибку FileReader; в приведённой REST-форме метод не используется. |

<a id="check"></a>

## 09. Проверка перед использованием

- Входные `data.vars` отображаются корректно, включая пустые значения.
- Имена и типы выходных переменных совпадают с настройками User Task.
- Пустые обязательные поля не отправляются; пользователь видит причину.
- Без профиля ЭЦП кнопка подписи недоступна, сообщение видно.
- Неуспешный ответ сервиса, отсутствующий `response` и HTTP-ошибка не скрывают ошибку и не оставляют бесконечную загрузку.
- Изменение текста после подписи сбрасывает CMS; сервер проверяет соответствие текста и подписи.
- Двойной щелчок не создаёт два одновременных запроса.
- Бизнес-ошибка `runTask` обработана по фактическому контракту runtime.
- При `resp.task` открывается новая форма, сохраняются остальные query-параметры.
- Ответ без следующей задачи не интерпретируется автоматически как завершение всего БП.
- Права исполнителя проверяются сервером; чужая или завершённая задача корректно отклоняется.
- Уход со страницы освобождает подписки. Отписка не отменяет уже начатую операцию на сервере.

<a id="generation"></a>

## 10. Автогенерация User Task

Второй предоставленный пример — автоматически созданная форма **«Регистрация по приглашению»**. Генератор уже создаёт каркас JIT-класса, контролы Reactive Forms, HTML полей, валидацию обязательности и передачу переменных в `runTask`. Этот результат можно использовать как основу пользовательской формы.

> Ниже разобран фактически предоставленный результат генерации. Названия кнопок генератора, структура его метаданных и точные правила выбора виджетов в материалах не показаны: их необходимо сверить с вашей версией DamuBPM.

| Элемент результата | Что создано в примере |
| --- | --- |
| Каркас | `const vm = this`, импорт из `forms`, `GenClass extends vm.constructor`. |
| Данные | Перенос `data.vars` в `detail` и установка заголовка. |
| Контролы | Для каждой переменной — `addControl` и подписка `valueChanges` для синхронизации с `detail`. |
| Обязательность | `Validators.required` для ИИН, логина и обоих паролей. |
| Разметка | Карточка, сетка, подписи, PrimeNG-поля, варианты мобильного отображения и сообщения ошибок. |
| Завершение | Проверка формы, сериализация табличных частей, явный payload и переход к `resp.task`. |
| Расширения | Маркеры `logic by bp_points_ui_blocks_n begin/end` для связанной логики UI-блоков. |

### Порядок работы

1. Определите переменные User Task, их назначение, обязательность и представление на экране.
2. В доступном редакторе вашей системы настройте форму и выполните штатную генерацию JavaScript/HTML.
3. Проверьте результат: каждому используемому `formControlName` должен соответствовать контрол, а каждой возвращаемой переменной — ключ payload.
4. Добавьте проверки, которых нет в генерации: межполевая валидация, обработка ошибок запросов, блокировка повторной отправки.
5. Протестируйте форму в контексте реальной задачи и назначенного исполнителя.

Перед повторной генерацией сохраните ручные доработки и сравните новую версию со старой. По предоставленному коду нельзя установить, сохраняет ли генератор ручные изменения. Не рассчитывайте на это без проверки.

<a id="generated-fields"></a>

## 11. Поля и HTML автогенерируемой формы

| Переменная | Контрол / обязательность | HTML и поведение |
| --- | --- | --- |
| `last_error` | Без `Validators.required` | Блок ошибки виден при непустом значении. Значение также включено в payload. |
| `iin` | `Validators.required` | `p-inputMask`, маска `999999999999`. |
| `login` | `Validators.required` | Текстовое поле с `pInputText`. |
| `password1` | `Validators.required` | `input type="password"`, десктопная и мобильная разметка. |
| `password2` | `Validators.required` | Подтверждение пароля. Совпадение с первым паролем в исходном JS не проверяется. |

`last_error` — переменная процесса в этой конкретной форме. `error_text` из базового примера инструкции — локальное состояние ошибки. Это разные механизмы; автоматически заменять одно другим не нужно.

### Связь переменной, контрола и шаблона

```javascript
// В ngOnInit():
this.form.addControl('login', this.formBuilder.control(
    this.detail.login == null ? '' : this.detail.login,
    Validators.required
));
// При использовании базового шаблона — сохраняем подписку для очистки.
this.subscriptions.push(this.f.login.valueChanges.subscribe(value => {
    this.detail.login = value;
}));

// В payload runTask():
login: this.f.login.value
```

```html
<label for="login">Логин <span class="text-danger">*</span></label>
<input id="login" pInputText type="text" formControlName="login"
       autocomplete="username" />
<div *ngIf="submitted && f.login.errors?.required" class="invalid-feedback">
    Заполните логин.
</div>
<div *ngIf="f.login.errors?.isExternalError" class="invalid-feedback">
    <span [translate]="f.login.errors?.isExternalErrorText"></span>
</div>
```

### Условия и зависимости шаблона

| Конструкция | Как читать |
| --- | --- |
| `[ngClass]="{ 'd-none': !(1==1) }"` | В этом примере класс скрытия не применяется. При настройке динамического условия учитывайте, что CSS скрывает поле, но не отключает его контрол и валидаторы. |
| `*ngIf="!(1==0)"` | Ветка видима; альтернативная ветка `*ngIf="1==0"` не создаётся. Это выражения данного результата, а не документированный формат метаданных генератора. |
| `[disabled]="1==0"` | В примере поле не отключено. Для Reactive Forms согласуйте disabled-состояние с самим контролом. |
| `d-none d-md-block` / `d-md-none` | CSS-варианты для разных ширин. Оба элемента могут оставаться в DOM; при упрощении используйте одно адаптивное поле. |
| `translate`, `[translate]` | Директивы перевода из runtime. |
| `p-inputMask`, `pInputText`, `ngbTooltip` | Требуют соответствующих компонентов и директив в JIT runtime. Самостоятельный статический HTML не исполняет Angular-шаблон. |

<a id="generated-validation"></a>

## 12. Проверки сверх автогенерации

**Маска ИИН и обязательность не равны полной проверке ИИН.** В предоставленном коде установлен только `Validators.required`. Сообщение для `errors.pattern` есть в HTML, но валидатора `pattern` в JS нет.

```javascript
// Добавьте после создания iin в ngOnInit():
this.f.iin.setValidators([
    Validators.required,
    Validators.pattern(/^[0-9]{12}$/)
]);
this.f.iin.updateValueAndValidity();

// После создания password1 и password2 добавьте валидатор группы.
// Если валидаторы группы уже есть — объедините их с этим правилом.
this.form.setValidators(group => {
    const first = group.get('password1').value;
    const second = group.get('password2').value;
    if (!first || !second) return null; // Пустые значения проверяет required.
    return first === second ? null : { passwordsMismatch: true };
});
this.form.updateValueAndValidity();
```

```html
<div *ngIf="submitted && f.iin.errors?.pattern" class="invalid-feedback">
    ИИН должен состоять из 12 цифр.
</div>
<div *ngIf="submitted && form.errors?.passwordsMismatch"
     class="invalid-feedback" role="alert">
    Пароли не совпадают.
</div>
```

Проверка 12 цифр проверяет только формат; проверку достоверности ИИН выполняйте отдельно на сервере. Передавайте ИИН строкой, чтобы не терять ведущие нули. Правила пароля и повторную проверку совпадения задайте на сервере согласно требованиям процесса.

### Внешние ошибки

HTML ожидает ошибки контролов с ключами `isExternalError` и `isExternalErrorText`. В приложенном JS нет кода их установки. Он может находиться в runtime или подключённых UI-блоках — это нужно проверить.

```javascript
// Пример ручного назначения ошибки, если это не делает runtime.
this.f.login.setErrors({
    ...(this.f.login.errors || {}),
    isExternalError: true,
    isExternalErrorText: 'Логин уже используется'
});
```

Текст следует брать из проверенного ответа сервера по известному контракту. Не очищайте все ошибки через `setErrors(null)` без учёта других валидаторов. Если реализация вручную хранит внешние ошибки, отдельно определите их сброс при изменении поля.

При групповой ошибке `form.invalid` будет истинным, хотя все отдельные контролы могут быть корректны. Исходный цикл диагностики только по `this.f` не покажет `passwordsMismatch`: учитывайте также `this.form.errors`.

<a id="generation-review"></a>

## 13. UI-блоки и проверка результата генерации

### Маркеры UI-блоков

```javascript
//logic by bp_points_ui_blocks_n begin

//logic by bp_points_ui_blocks_n end
```

В приложенном примере этот участок пуст. Сохраняйте маркеры, если форма обслуживается генератором. Они указывают на область связанной логики UI-блоков, но не подтверждают, что произвольный код внутри будет сохранён при следующей генерации. Способ подключения блока и правила перегенерации уточните в редакторе вашей версии.

### Замечания именно к приложенному результату

| Место | Что проверить / изменить |
| --- | --- |
| Искажённое условие строки | В Markdown присутствует `&#x30;*<*(1+1+1+1+1+0)`. Оно не является пригодным Angular-выражением. Сверьте с исходным HTML в редакторе; не восстанавливайте условие по догадке. Если строка безусловно видима, удалите только этот `*ngIf`. |
| Экранирование Markdown | `\<`, `\*` и `&#x20;` могут быть следствием копирования. В редактор JIT вставляйте обычные теги и `*ngIf`, а не Markdown-экранирование. |
| Заголовок карточки | `bg-white` вместе с `text-white` даёт белый текст на белом фоне без переопределения CSS. Выберите контрастный цвет. Заголовок также продублирован над формой. |
| Tooltip логина | Есть `[ngbTooltip]="tipContentlogin"`, но `ng-template #tipContentlogin` закомментирован. Восстановите шаблон либо удалите привязку. |
| `service_error` | Используется в HTML, но не объявлен в данном классе. Проверьте наследование runtime либо объявите локальное состояние. |
| Сообщения обязательности | Логин и пароли обязательны в JS, но HTML в основном показывает внешние ошибки. Добавьте сообщения `required`. |
| Повторная отправка и HTTP-ошибки | В исходной генерации нет блокировки запроса и обработчика `error`. Перенесите схему из базового примера инструкции. |
| Подписки | Сохраняйте `valueChanges` и подписку отправки, освобождайте их в `ngOnDestroy`. |
| Начальные значения | Проверка `this.detail ? this.detail['login'] : ''` не гарантирует пустую строку при отсутствующем свойстве: объект `{}` уже truthy. Проверяйте само значение на `null/undefined`. |
| Поля паролей | Не логируйте payload с паролями и не возвращайте заполненные пароли на повторно открытую форму. `type="password"` лишь скрывает символы на экране. |
| Отображение на телефоне | Проверьте единственное видимое поле для каждой переменной, доступность подписей и отсутствие повторяющихся HTML id. |
| Табличные части | В JS поддержка есть, но в приложенном HTML таблиц нет. Добавлять их в форму без настройки переменных и событий JIT-PAGE не требуется. |

### Минимальный прогон

- Пустые поля: отправка блокируется, ошибки видны.
- ИИН с неполной длиной: отклоняется после добавления валидатора формата.
- Разные пароли: видна групповая ошибка, `runTask` не вызывается.
- Серверная ошибка логина: отображается у поля либо через `last_error` согласно контракту процесса.
- Корректные значения: уходят все пять ключей исходного payload, если контракт процесса не изменён.
- Повторная генерация: ручные изменения проверены сравнением; мобильный и десктопный режимы работают.

Дополнительный источник: предоставленный файл «Вставленный Markdown(2).md», автогенерируемая форма «Регистрация по приглашению». Примеры новых валидаторов и обработки ошибок являются доработками, а не описанием уже реализованного поведения генератора.

<a id="home"></a>

## 14. Ссылка на главную

```html
<a href="index.html">← Главная</a>
```

Ссылка уже размещена в шапке и внизу инструкции. Положите этот файл рядом с существующим `index.html`. Главная страница в комплект не входит.

Источники: предоставленные JavaScript и HTML; DamuBPM JIT Widget Developer Guide; DamuBPM BPM / Lua Guide. Все доработки в примерах обозначены отдельно от исходного поведения.

[← Вернуться на Главную](index.html)
