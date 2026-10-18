# DamuBPM JIT (Angular): неочевидное

## 1. Шаблоны

- `[innerHTML]` прогоняется через санитайзер Angular: **вырезаются `style="…"` и `<svg>`**. Мини-превью строить обычной разметкой с `[style.background]`, `[style.font-family]` и т. п.; SVG показывать через `<img [src]="dataUri">`:
  ```js
  svgData(svg) {
    let x = String(svg).replace(/currentColor/g, '#5b4ee6');
    if (x.indexOf('xmlns=') < 0) x = x.replace('<svg', '<svg xmlns="http://www.w3.org/2000/svg"');
    return 'data:image/svg+xml;charset=utf-8,' + encodeURIComponent(x);
  }
  ```
- Одиночная `{` в тексте шаблона воспринимается как ICU-выражение → писать `{{ '{' }}` / `{{ '}' }}`.
- Текст с `{{ … }}` (примеры Go-шаблонов, Handlebars) — только внутри элемента с `ngNonBindable`.
- CSS-переменные биндятся как `[style.--v]="value"`.
- Рекурсивные формы (вложенные списки) — `<ng-template #tpl let-fields="fields" let-obj="obj">` + `*ngTemplateOutlet="tpl; context: {...}"`.

## 2. iframe-превью

- `[srcdoc]` требует bypass санитайзера — проще присвоить через DOM после рендера:
  ```js
  setFrame(id, html, n = 0) {
    setTimeout(() => {
      const el = document.getElementById(id);
      if (el) { el.srcdoc = html; return; }
      if (n < 20) this.setFrame(id, html, n + 1);
    }, n ? 60 : 0);
  }
  ```
- Связь превью ↔ редактор — `postMessage` в обе стороны; слушатель `message` вешать в `ngOnInit`, снимать в `ngOnDestroy`.
- Корневые ссылки (`/restapi/getfile?code=…`) внутри srcdoc работают (базовый URL наследуется от родителя).

## 3. Жизненный цикл и навигация

- Хуки (`ngOnDestroy` и др.) берутся из класса при компиляции: **переопределять их на экземпляре в рантайме бесполезно** — вся очистка внутри метода класса.
- Параметры URL: `new URLSearchParams(location.search)` (плюс запасной разбор `location.hash`).
- Переход на тот же маршрут с другим `?id=` не пересоздаёт компонент — после возврата из БП обновлять данные явно (колбэк `bpRun`).
- Каждая страница самостоятельна: общий хелпер (`api`, `notify`, загрузка CSS) копируется в каждую.

## 4. Работа с API

- `this.dbQueryService.restapiPost(service, body, true).pipe(first())`; при `cut_status_info = 1` ответ `{success, data}` или `{success:false, error:{message}}`. Обрабатывать оба варианта плюс HTTP-ошибку:
  ```js
  api(action, data, ok, fail) {
    this.dbQueryService.restapiPost('my_api', { action, data: data || {} }, true).pipe(first()).subscribe({
      next: r => { if (r && r.success === false) { const m = (r.error && r.error.message) || 'Ошибка'; this.error = m; fail && fail(m); return; } ok && ok(r && r.data ? r.data : (r || {})); },
      error: e => { const x = e && (e.error || e); const m = (x && x.error && x.error.message) || (x && x.message) || 'Ошибка'; this.error = m; fail && fail(m); }
    });
  }
  ```
- jsonb-поля из `SqlQueryRows` могут прийти строкой — на фронте `parse(v, def)`: объект вернуть как есть, строку `JSON.parse` в `try`.
- Загрузка файлов без возни с multipart и авторизацией: `FileReader.readAsDataURL` → base64 в REST → `UploadRawData`. Ограничивать размер (~10 МБ).

## 5. Оформление

- Свой CSS-префикс модуля (`be-`, `mp-`) и обёртка `.<prefix>`; цвета через `var(--damu-v1-*)` с фолбэком и отдельный блок `.damu-theme-dark .<prefix>{…}` для тёмной темы.
- Не использовать префикс, совпадающий с классами генерируемого контента (например, сайтов в превью).
