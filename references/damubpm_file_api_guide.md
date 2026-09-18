# DamuBPM File API — загрузка и скачивание файлов

[← Вернуться на Главную](index.html)

Практический справочник по штатным HTTP-методам DamuBPM для загрузки и выдачи файлов. Описание основано на предоставленных реализациях `Upload` и `GetFile` и на маршрутах:

```text
POST /restapi/upload
GET  /restapi/getfile
GET  /restapi/getfile2/:filename
```

> Для клиентской загрузки используется `multipart/form-data`. Основной идентификатор файла после загрузки — поле `guid`; именно его нужно передавать как `code` при скачивании.

---

## Содержание

1. [Быстрый старт](#быстрый-старт)
2. [POST /restapi/upload](#post-restapiupload)
3. [GET /restapi/getfile](#get-restapigetfile)
4. [GET /restapi/getfile2/:filename](#get-restapigetfile2filename)
5. [Примеры для DamuBPM Frontend](#примеры-для-damubpm-frontend)
6. [Локальное и S3-хранилище](#локальное-и-s3-хранилище)
7. [Hooks каталогов](#hooks-каталогов)
8. [Права доступа и заголовки](#права-доступа-и-заголовки)
9. [Миниатюры изображений](#миниатюры-изображений)
10. [Ошибки и диагностика](#ошибки-и-диагностика)
11. [Важные замечания по текущей реализации](#важные-замечания-по-текущей-реализации)
12. [Чек-лист интеграции](#чек-лист-интеграции)
13. [Добавление ссылки в index.html](#добавление-ссылки-в-indexhtml)

---

# Быстрый старт

## 1. Загрузить файл

```bash
curl -X POST \
  "https://example.kz/restapi/upload" \
  -F "dir=documents" \
  -F "file=@./contract.pdf"
```

Типичный успешный ответ:

```json
{
  "guid": "documents-550e8400-e29b-41d4-a716-446655440000",
  "url": "/restapi/getfile?code=documents-550e8400-e29b-41d4-a716-446655440000",
  "id": "12345",
  "result": "ok",
  "filename": "contract.pdf",
  "restServiceOutput": null
}
```

Сохраните `guid`. Это публичный идентификатор файла для последующих вызовов File API.

## 2. Открыть файл в браузере

```text
/restapi/getfile?code=<guid>&inline=true
```

## 3. Скачать как вложение

```text
/restapi/getfile?code=<guid>&attachment=true
```

## 4. Получить Base64

```text
/restapi/getfile?code=<guid>&base64=true
```

## 5. Получить миниатюру

```text
/restapi/getfile?code=<guid>&thumb=120-0
```

---

# POST /restapi/upload

Endpoint принимает файл в `multipart/form-data`, проверяет каталог, тип и размер, сохраняет содержимое в локальное файловое хранилище либо S3-совместимое хранилище, регистрирует запись в таблице `files` и возвращает `guid`.

## Поля запроса

| Поле | Обязательное | Назначение |
|---|---:|---|
| `file` | да | Multipart-файл. Сервер читает его через `FormFile("file")`. |
| `dir` | да | Код каталога из `dirs.code`. Определяет правила, путь и тип хранилища. |
| `idref` | нет | Ссылка на внешний объект. Сохраняется в `files.idref`, если включён `FeatureFilesIdRef`. |
| `id` | нет | Не сохраняется напрямую в `files`, но передаётся в `on_upload_script`. |

Не задавайте `Content-Type: multipart/form-data` вручную в браузере: `FormData` должен сам добавить boundary.

## Пример cURL с idref

```bash
curl -X POST \
  "https://example.kz/restapi/upload" \
  -F "dir=contract_files" \
  -F "idref=98765" \
  -F "id=98765" \
  -F "file=@./contract_signed.pdf"
```

## Что делает Upload по шагам

```text
multipart request
      ↓
проверка server status
      ↓
проверка общего Content-Length (до 2 000 000 000 байт)
      ↓
получение file + обязательного dir
      ↓
проверка имени файла
      ↓
проверка разрешённой группы типов файла для каталога
      ↓
чтение настроек dirs + s3bucket
      ↓
формирование guid и относительного filepath
      ↓
локальное сохранение ИЛИ PutObject в S3
      ↓
генерация миниатюр для изображений при локальном хранении
      ↓
проверка max_size_byte каталога
      ↓
INSERT в files
      ↓
on_upload_script, если настроен
      ↓
JSON-ответ с guid/url/id
```

## Формирование guid

Код формирует идентификатор примерно по схеме:

```text
<dir>-<uuid>
```

Например:

```text
contract_files-a132d462-6690-4ca0-a29c-345b1f38d1ef
```

Именно это значение возвращается как `guid` и затем используется как `code` в `GetFile`.

## Размещение файла

По умолчанию формируется дата:

```text
YYYY/MM/DD/
```

Если у каталога заполнен `path_expr`, значение пути вычисляется SQL-выражением каталога.

При `filename_as_uuid = 0`:

```text
<pathExprValue>/<guid>
```

При `filename_as_uuid = 1`:

```text
<pathExprValue>/<guid>/<original_filename>
```

Клиент не передаёт физический путь напрямую — он определяется настройками `dirs`.

## Проверка имени файла

Отклоняются имена, содержащие:

```text
\ / : * ? " < > |
```

Ответ:

```json
{"result":"INCORRECT FILENAME"}
```

HTTP status: `406 Not Acceptable`.

## Проверка типа файла

Если для каталога задан `allowed_ftg_id`, расширение файла должно входить в разрешённую группу типов `file_type_groups` / `file_type_group_types` / `file_types`.

При запрете:

```json
{"result":"It is forbidden to upload a file with this type"}
```

HTTP status: `406`.

## Ограничение размера

Есть два уровня:

1. общий предел по `r.ContentLength` — `2 000 000 000` байт;
2. `dirs.max_size_byte`, если включён `FeatureFileSize`.

Ответ при превышении общего предела:

```json
{"result":"TOO LARGE FILE"}
```

Ответ при превышении лимита каталога:

```json
{"result":"Error on upload: File max size exceeded"}
```

## Успешный ответ

Структура ответа:

| Поле | Значение |
|---|---|
| `guid` | код файла, который передаётся в `GetFile` как `code` |
| `url` | URL на скачивание, собранный из `ecm_getfile_prefix + guid` |
| `id` | ID записи таблицы `files` в строковом виде |
| `result` | `ok` |
| `filename` | исходное имя файла; при upload-hook может быть пустым в текущей реализации |
| `restServiceOutput` | результат `on_upload_script`, если hook настроен |

---

# GET /restapi/getfile

Endpoint находит файл по `files.code`, проверяет доступ при необходимости и отдаёт содержимое из локального диска либо S3.

## Основной параметр

| Параметр | Обязательный | Назначение |
|---|---:|---|
| `code` | да | `guid`, полученный от `/restapi/upload` |

Пример:

```text
GET /restapi/getfile?code=contract_files-a132d462-6690-4ca0-a29c-345b1f38d1ef
```

## Параметры режима ответа

| Параметр | Значение | Эффект |
|---|---|---|
| `attachment` | `true` | Добавляет `Content-Disposition: attachment; filename="..."` |
| `inline` | `true` | Добавляет `Content-Disposition: inline; filename="..."` |
| `base64` | `true` | Вместо бинарного содержимого отдаёт Base64-текст |
| `thumb` | `120-0`, `240-0`, `360-0`, `480-0`, `1024-0` | Переключает локальный путь на файл миниатюры |
| `id` | произвольное | Передаётся в `on_read_script`; на выбор файла не влияет |

Если одновременно передать `attachment=true` и `inline=true`, в текущем коде приоритет имеет `attachment`.

## Скачать файл

```bash
curl -L \
  "https://example.kz/restapi/getfile?code=<guid>&attachment=true" \
  -o contract.pdf
```

## Показать PDF/изображение inline

```text
/restapi/getfile?code=<guid>&inline=true
```

## Base64

```bash
curl \
  "https://example.kz/restapi/getfile?code=<guid>&base64=true"
```

Сервер сохраняет `Content-Type` исходного файла, поэтому клиенту следует учитывать, что тело при `base64=true` уже является текстовым Base64-представлением.

## Заголовки ответа

`GetFile` формирует:

```text
Content-Type: <mime из file_types>
Etag: "<md5 от file_id>"
Last-Modified: <created_at файла>
```

Дополнительно может выставляться `Content-Disposition`.

`Content-Length` в предоставленном коде не выставляется.

---

# GET /restapi/getfile2/:filename

Маршрут зарегистрирован так:

```text
GET /restapi/getfile2/:filename
```

но передаётся в тот же обработчик `GetFile`.

По предоставленной реализации сам `GetFile` не читает `:filename` из `httprouter.Params` и всё равно ищет файл по query-параметру `code`.

Поэтому безопасный вариант вызова:

```text
/restapi/getfile2/contract.pdf?code=<guid>&inline=true
```

`contract.pdf` здесь полезен для человекочитаемого URL, но идентификация файла в показанном обработчике выполняется через `code`.

---

# Примеры для DamuBPM Frontend

Для `/restapi/upload` нужен настоящий multipart-запрос. Так как предоставленный endpoint не является обычным JSON Lua REST-service, надёжный клиентский вариант — стандартный Browser API `FormData + fetch`.

## HTML

```html
<input
    #fileInput
    type="file"
    (change)="onFileSelected($event)">

<button
    type="button"
    (click)="uploadSelected()"
    [disabled]="!selectedFile || uploading">
    Загрузить
</button>
```

## JIT logic

```javascript
selectedFile = null;
uploading = false;
uploadedFile = null;

onFileSelected(event) {
    const files = event.target.files;
    this.selectedFile = files && files.length ? files[0] : null;
}

async uploadSelected() {
    if (!this.selectedFile || this.uploading) {
        return;
    }

    this.uploading = true;

    try {
        const form = new FormData();
        form.append('dir', 'contract_files');
        form.append('file', this.selectedFile, this.selectedFile.name);

        const response = await fetch('/restapi/upload', {
            method: 'POST',
            body: form,
            credentials: 'same-origin'
        });

        const text = await response.text();

        if (!response.ok) {
            throw new Error(text || ('HTTP ' + response.status));
        }

        this.uploadedFile = JSON.parse(text);
    } finally {
        this.uploading = false;
    }
}
```

## Загрузка с idref

```javascript
const form = new FormData();
form.append('dir', 'contract_files');
form.append('idref', String(this.contractId));
form.append('id', String(this.contractId));
form.append('file', this.selectedFile, this.selectedFile.name);

const response = await fetch('/restapi/upload', {
    method: 'POST',
    body: form,
    credentials: 'same-origin'
});
```

## Построение URL

```javascript
fileUrl(guid) {
    return '/restapi/getfile?code=' + encodeURIComponent(guid);
}

downloadUrl(guid) {
    return this.fileUrl(guid) + '&attachment=true';
}

inlineUrl(guid) {
    return this.fileUrl(guid) + '&inline=true';
}

thumbUrl(guid, size) {
    return this.fileUrl(guid) + '&thumb=' + encodeURIComponent(size);
}
```

## Открыть файл

```javascript
openFile(guid) {
    window.open(this.inlineUrl(guid), '_blank');
}
```

## Скачать файл

```javascript
downloadFile(guid) {
    window.location.href = this.downloadUrl(guid);
}
```

## Показать изображение

```html
<img
    *ngIf="uploadedFile?.guid"
    [src]="thumbUrl(uploadedFile.guid, '120-0')"
    alt="preview">
```

---

# Локальное и S3-хранилище

Тип хранения определяется настройками каталога и записывается в `files`.

## Локальный диск

Физический путь строится из:

```text
dirs.unix_path / dirs.win_path
+
path_expr или дата
+
сформированный filepath
```

Файл создаётся с правами `0600`.

## S3-совместимое хранилище

Используются настройки `s3bucket`:

```text
endpoint
bucket
access_key_id
secret_access_key
usessl
```

Если `access_key_id` или `secret_access_key` начинается с `DAMU_S3`, значение читается из одноимённой переменной окружения.

При обычной бинарной загрузке содержимое передаётся через `PutObject`.

При скачивании используется `GetObject`, после чего данные копируются в HTTP response.

---

# Hooks каталогов

## on_upload_script

Если у `dirs` заполнен `on_upload_script`, после записи файла и INSERT в `files` запускается Lua service script.

В `request.input` передаются:

```text
id
idref
fileName
uuid
origFileName
fileURL
```

Где:

- `uuid` — `guid` файла;
- `fileURL` — URL, собранный через `ecm_getfile_prefix`;
- `origFileName` в текущей реализации содержит временный путь с исходным именем, а не только basename.

Результат hook возвращается клиенту в:

```json
"restServiceOutput": ...
```

## on_read_script

После выдачи файла может запускаться `on_read_script` каталога.

В `request.input` передаются:

```text
id
fileName
```

`id` берётся из query-параметра `id` вызова `GetFile`.

Такой hook хорошо подходит для аудита/логирования факта чтения. В текущей реализации он запускается уже после записи содержимого файла в HTTP response, поэтому использовать его для изменения самого ответа не следует.

---

# Права доступа и заголовки

`GetFile` читает `dirs.access_control`.

Если значение не равно `0`, выполняется:

```text
DetailEntityGrantCheck(..., "files", file_id, current_user_id)
```

При отсутствии прав выдача прекращается с ошибкой `Access Denied To File`.

То есть файл из защищённого каталога нельзя считать только по знанию `guid`, если серверная проверка прав для этой записи не пройдена.

В предоставленном `Upload` аналогичного вызова `DetailEntityGrantCheck` внутри функции нет. Поэтому права на сам endpoint загрузки должны дополнительно обеспечиваться общим механизмом авторизации/маршрутизации DamuBPM и настройками приложения.

---

# Миниатюры изображений

При локальной загрузке для форматов:

```text
JPG
JPEG
JFIF
PNG
```

вызывается генерация JPEG-превью.

Текущий `Upload` создаёт:

```text
120
240
320
480
1024
```

Имена:

```text
<physical_file>-thumb-120-0.jpg
<physical_file>-thumb-240-0.jpg
...
```

Для S3-ветки `thumbImageAll` не вызывается.

---

# Ошибки и диагностика

| Ситуация | Ответ / поведение |
|---|---|
| `dir` не передан | `406`, `{"result":"dir is required field"}` |
| Размер запроса > 2 000 000 000 | `406`, `TOO LARGE FILE` |
| Недопустимое имя | `406`, `INCORRECT FILENAME` |
| Тип запрещён каталогом | `406`, `It is forbidden to upload a file with this type` |
| Превышен `max_size_byte` | `406`, `Error on upload: File max size exceeded` |
| Ошибка S3 клиента/PutObject | `500` с текстом ошибки |
| Нет доступа к защищённому файлу | `Access Denied To File` через общий обработчик ошибок |

Клиенту рекомендуется всегда проверять одновременно:

```text
response.ok
HTTP status
JSON.result
```

---

# Важные замечания по текущей реализации

Ниже — не контракт API, а найденные особенности конкретного предоставленного кода.

## 1. Несоответствие размера 320 / 360

`Upload.thumbImageAll()` создаёт миниатюру `320-0`, а предоставленный `GetFile` распознаёт `thumb=360-0`.

Следствие:

- `thumb=360-0` пытается открыть файл, который стандартная загрузка не создаёт;
- `thumb=320-0` не обрабатывается веткой `GetFile` и, вероятно, вернёт оригинал.

Рекомендуется привести обе стороны к одному размеру.

## 2. SQL-опечатка при FeatureFilesIdRef + FeatureFileSize

В одной ветке INSERT присутствует:

```text
created_by.filesize
```

в списке колонок. По смыслу должно быть две колонки:

```text
created_by,filesize
```

Иначе комбинация `FeatureFilesIdRef=true`, непустой `idref` и `FeatureFileSize=true` может приводить к SQL-ошибке.

## 3. max_size_byte проверяется после физической записи

Сначала файл уже записывается на диск/S3, и только затем проверяется `dirs.max_size_byte`.

При превышении лимита функция возвращает ошибку до INSERT в `files`, но физически созданный объект в показанном коде не удаляется. Возможны orphan-файлы.

## 4. Base64 + S3 требует исправления/проверки

В ветке `Content-Transfer-Encoding: base64` для S3 исходный `file` сначала копируется в буфер, а затем `PutObject` получает decoder, читающий тот же уже использованный stream. Такой путь выглядит некорректным.

Для S3 рекомендуется использовать обычную бинарную multipart-загрузку до исправления этой ветки.

## 5. Проверка TLS S3 отключена

Создаваемый transport использует `InsecureSkipVerify: true`.

Для production лучше валидировать сертификат S3 endpoint либо сделать этот режим явно конфигурируемым.

## 6. Локальное скачивание читает весь файл в память

Локальная ветка использует `ioutil.ReadFile(fullfileName)`, а затем `w.Write(b)`.

Для крупных файлов эффективнее потоковая отдача через `os.Open` + `io.Copy`/`http.ServeContent`, особенно если нужны большие файлы и Range/resume.

## 7. on_read_script запускается после выдачи тела

Hook полезен для логирования, но HTTP response к этому моменту уже начат/отправлен. Ошибка hook не может надёжно заменить ранее выданный ответ.

## 8. Отсутствующий multipart `file` обрабатывается неявно

При ошибке `r.FormFile("file")` код только печатает ошибку и делает `return`, не задавая явный HTTP status/JSON. Клиент может получить неинформативный ответ. Лучше вернуть `400 Bad Request` с единым JSON-форматом.

---

# Чек-лист интеграции

1. Создайте/проверьте запись каталога в `dirs`.
2. Проверьте `unix_path`/`win_path` или S3-настройки.
3. Настройте разрешённые типы через `allowed_ftg_id`, если требуется.
4. Настройте `max_size_byte`, если включён `FeatureFileSize`.
5. Отправляйте `POST /restapi/upload` как `multipart/form-data`.
6. Всегда передавайте `dir` и multipart part `file`.
7. После успеха храните `response.guid` рядом с бизнес-объектом.
8. Для связи с объектом используйте `idref`, если `FeatureFilesIdRef` включён.
9. Для выдачи используйте `/restapi/getfile?code=<guid>`.
10. Для скачивания добавляйте `attachment=true`.
11. Для просмотра добавляйте `inline=true`.
12. Для защищённых каталогов проверьте права на `files`.
13. Если нужен audit, используйте `on_upload_script` / `on_read_script`.
14. До исправления thumbnail mismatch не полагайтесь на `360-0` без проверки.
15. Для S3 используйте обычный бинарный multipart, а не `Content-Transfer-Encoding: base64`.

---

# Добавление ссылки в index.html

Карточка/ссылка на этот справочник:

```html
<a href="damubpm_file_api_guide.html">
    Файлы: загрузка и скачивание
</a>
```

Если главная страница построена карточками:

```html
<a class="guide-card" href="damubpm_file_api_guide.html">
    <strong>File API</strong>
    <span>Загрузка, скачивание, S3, Base64, thumbnails и hooks</span>
</a>
```

На самой странице справочника используйте:

```html
<a href="index.html">← Вернуться на Главную</a>
```

---

# Короткая памятка

```text
UPLOAD
POST /restapi/upload
multipart: dir + file [+ idref] [+ id]
→ response.guid

DOWNLOAD
GET /restapi/getfile?code=<response.guid>

DOWNLOAD AS FILE
GET /restapi/getfile?code=<guid>&attachment=true

INLINE
GET /restapi/getfile?code=<guid>&inline=true

BASE64
GET /restapi/getfile?code=<guid>&base64=true

THUMB
GET /restapi/getfile?code=<guid>&thumb=120-0
```
