# GitLab CI/CD: перенос объектов в YAML

Объекты платформы (сущности, БП, страницы, REST, запросы, luas и т. д.) выгружаются с тестового стенда в репозиторий GitLab в виде YAML-файлов. После слияния в боевую ветку GitLab CI загружает изменённые файлы на боевой стенд.

Связанные объекты: `patch_out`, `pkg/object`, `pkg/yaml` (см. `damubpm_pkg_yaml.md`), БП `patch_out_to_gitlab_yaml`, REST `gitlab_runner_accept_object`, БП `import_object_from_file`, `patch_in`, `.gitlab-ci.yml`, `scripts/deploy_changes.py`.

## 1. Как это устроено

1. **Патч на тестовом стенде.** Доработанные объекты собираются в исходящий патч `patch_out`: вручную, автопатчем или MCP-методом `patch_out_add_objects`.
2. **БП `patch_out_to_gitlab_yaml`.** Каждый объект патча превращается в YAML через `pkg/object.toObject` + `pkg/yaml.marshal` и коммитится одним коммитом в ветку `test-helpdesk.damubpm.com`.
3. **Job `promote_to_prod` (ветка теста).** Создаёт merge request `test-helpdesk.damubpm.com` → `helpdesk.damubpm.com` и вливает его, когда GitLab отметит MR как готовый к слиянию.
4. **Job `deploy-objects` (боевая ветка).** `scripts/deploy_changes.py` берёт файлы, изменённые с прошлого состояния ветки, и по одному отправляет их на REST стенда.
5. **REST `gitlab_runner_accept_object` → БП `import_object_from_file`.** Проверка токена, разбор YAML/JSON, Upsert объектов по ключам, удаление лишних дочерних строк, публикация сущностей и БП. Каждый файл — отдельная запись в «Принятых патчах» (`patch_in`).

Ветка определяет стенд: имя боевой ветки совпадает с хостом стенда, и пайплайн сам строит адрес `https://<ветка>/restapi/services/run/gitlab_runner_accept_object`. Тестовая ветка — только источник, на неё деплоя нет.

## 2. Формат YAML-объекта

Один файл — один корневой объект со всеми дочерними записями (для БП это шаги, стрелки, переменные, роли). Путь в репозитории:

```
yaml/<код модуля сущности>/<код сущности>/[<родитель>/]<code объекта>.yaml

yaml/core/rest_services/gitlab_runner_accept_log.yaml
yaml/bpms/bp_processes/import_object_from_file.yaml
yaml/core/detail_queries/its_d_payord_in/its_coa_pst.yaml   # родитель — details
yaml/core/dashboards_page/id_17.yaml                        # пустой code → id_<id>
```

Родительская папка добавляется для `detail_queries` (код `details`), `entity_attrs` (код сущности) и `bp_process_vars` (код БП). Символы `/ \ ? * :` в `code` заменяются на `-`.

Содержимое — то же, что в JSON-выгрузке `pkg/object`, только в YAML:

```yaml
objects:
  - entity: rest_services
    is_root: true
    values:
      - code: code
        is_object: "1"              # ключ для поиска записи (Upsert)
        value: gitlab_runner_accept_log
      - code: body
        is_object: "0"
        value: |-
          output = {}
          output.ok = true
      - code: module_id             # ссылка — не id, а ключ связанной записи
        entity: modules
        is_object: "0"
        pk:
          - code: code
            is_object: "1"
            value: core
```

| Поле | Смысл |
|---|---|
| `entity` | код сущности (таблицы) записи |
| `is_root` | `true` у корневого объекта файла, `false` у дочерних (табличные части с `is_child_obj = 1`) |
| `values[].is_object` | `"1"` — атрибут входит в ключ записи: по нему ищется существующая запись на стенде-приёмнике |
| `values[].pk` | ссылка на другую запись по её ключу. На приёмнике ключ превращается в id; если запись не найдена, ссылка становится пустой |
| `files$base64` | содержимое файла для сущности `files` |
| `bp_points$wf_status_attr_value_code` | статус точки workflow по коду, а не по id |

id, `created_*`, `updated_*`, `entryid` и формульные атрибуты не выгружаются. Для `bp_points`, `bp_sequence_flows`, `bp_point_sfs` берутся только строки с `is_active = 1`. Объекты связываются между стендами только по ключевым атрибутам (`is_object = 1`), поэтому у ключей должны быть одинаковые значения на всех стендах.

### Табличные части в выгрузке сущности

Выгрузка сущности тянет табличные части целиком, по цепочке дочерних объектов (`table_parts.is_child_obj = 1`):

```
entities
└─ table_parts             ключ: code + entity_id
   ├─ table_parts_tp       ключ: code
   └─ table_part_cols      ключ: table_part_id + attr_id + code
      └─ table_part_cols_a        ключ: code
         └─ table_part_cols_a_pv  ключ: code   (параметры компонента, например query_code)
```

Пример: для ТЧ `its_d_payord_out_contract` в файл попадают 12 объектов — сама ТЧ, 5 колонок, 5 элементов и параметр `query_code` с фильтром `its_cli_contract_select&flt$its_cli_id$eq='+detail.cli_id+'`.

- Ключ каждого уровня — `code` (uuid по умолчанию, уникален). У `table_part_cols_a` и `table_part_cols_a_pv` у атрибута `code` должен стоять `is_object = 1`. Без ключа ссылка дочерней записи на родителя (`table_part_cols_a_id`) не выгружается, и параметр садится на приёмнике с пустой ссылкой.
- Коды колонок, элементов и параметров должны совпадать на стендах. Если ТЧ на приёмнике собрана вручную с другими `code`, первая загрузка создаст строки заново, а старые удалит как лишние дочерние.
- Повторная загрузка того же файла ничего не меняет: id, количество строк и связи остаются прежними (проверено двумя загрузками подряд).
- `table_part_param_values` (параметры старых ng-шаблонов) дочерним объектом не выгружается.
- Колонки и элементы в БД появляются при публикации сущности (`gen_show_cols = 1`), а файл их перезаписывает своими значениями. После загрузки страницы сущности нужно перегенерировать, чтобы изменения ТЧ стали видны.

Особенности модуля YAML (экспонента больших чисел, восьмеричные числа с ведущим нулём, даты, `null`) — в `damubpm_pkg_yaml.md`. Выгрузка сама берёт все строки в кавычки, опасен только YAML, отредактированный вручную.

## 3. Выгрузка со стенда

БП `patch_out_to_gitlab_yaml` (вход `id` — id патча; выход `last_error`, `debug`):

- выгружает **все** объекты патча в порядке `coalesce(nn, id)`;
- получает список файлов ветки через GitLab API (`repository/tree`, постранично) и для каждого файла выбирает `create` или `update`; всё уходит одним коммитом (`repository/commits`) с текстом из `patch_out.title` (пусто → `patch_out <id>`);
- объекты, которых уже нет в базе, пропускает и перечисляет в `debug`; при ошибке `toObject` или `yaml.marshal` останавливается с `last_error`;
- если два объекта патча дают один и тот же путь, в коммит идёт один файл.

| Параметр стенда (`params`) | Значение на тесте |
|---|---|
| `gitlab_yaml_host` | `https://gitlab.com/` |
| `gitlab_yaml_project` | `kz.bapps/sources-yaml` |
| `gitlab_yaml_branch` | `test-helpdesk.damubpm.com` |
| `gitlab_yaml_project_token` | project access token с правом `write_repository` |

Перед выгрузкой проверьте патч: тестовые кроны и строки `patch_out_obj` с пустым `id_object` (бывают после MCP `create_bp_process`) в YAML не нужны.

JSON-вариант той же выгрузки — БП `patch_out_to_gitlab` (папка `objects/`, параметры `gitlab_objects_*`). Он по-прежнему принимается, но основной формат — YAML.

## 4. Ветки и пайплайн

| Ветка | Роль | Что запускается |
|---|---|---|
| `test-helpdesk.damubpm.com` | сюда пишет тестовый стенд | `promote_to_prod`: MR в прод и автослияние. `deploy-objects` не создаётся — на ветке нет `DAMU_CICD_TOKEN` |
| `helpdesk.damubpm.com` | боевой стенд, protected | `deploy-objects`: загрузка изменённых файлов на `https://helpdesk.damubpm.com` |

Обе ветки должны быть **protected**: protected-переменные GitLab видны только на protected-ветках.

### Job deploy-objects

```yaml
image: python:3.12-slim

variables:
  GIT_DEPTH: "0"                      # полная история, чтобы работал git diff

deploy-objects:
  stage: deploy
  resource_group: $CI_COMMIT_BRANCH   # не загружать на один стенд параллельно
  environment:
    name: $CI_COMMIT_BRANCH
    url: https://$CI_COMMIT_BRANCH
  rules:
    - if: '$CI_PIPELINE_SOURCE == "push" && $DAMU_CICD_TOKEN'
      changes:
        - yaml/**/*
        - objects/**/*
    - if: '$CI_PIPELINE_SOURCE == "web" && $DAMU_CICD_TOKEN'
  before_script:
    - apt-get update -qq && apt-get install -y -qq git > /dev/null
    - pip install --no-cache-dir -q requests
  script:
    - |
      if [ "$CI_PIPELINE_SOURCE" = "web" ] && [ "$DEPLOY_ALL" = "1" ]; then
        python scripts/deploy_changes.py --all
      else
        python scripts/deploy_changes.py
      fi
```

Job `promote_to_prod` (стадия `promote`, `GIT_STRATEGY: none`) работает только через GitLab API и запускается после `deploy`. Полный текст — в `.gitlab-ci.yml` репозитория.

### Что делает deploy_changes.py

1. Берёт изменения `git diff --name-status $CI_COMMIT_BEFORE_SHA $CI_COMMIT_SHA`. Если базового коммита нет (новая ветка, `0000…`), берёт изменения только текущего коммита.
2. Оставляет файлы `yaml/**/*.yaml|yml` (формат yaml) и `objects/**/*.json` (формат json). Удалённые файлы только пишет в лог.
3. Сортирует по сущности: `modules`, `dirs`, `entities`, `entity_attrs`, `table_parts`, `queries`, `details`, `detail_queries`, `luas`, `rest_services`, прочие, `bp_processes`, `pages` — чтобы ссылки находили уже загруженные записи.
4. Отправляет каждый файл: `POST {path, format, commit, content}` с заголовком `Authorization: CICDToken <токен>`, таймаут 15 минут.
5. Ошибки по одному файлу не останавливают остальные; в конце job падает со списком неудачных файлов. Недоступный стенд останавливает job на первом файле. Если имя ветки не похоже на хост (нет точки) и `DAMU_API_URL` не задан — job сразу останавливается с подсказкой.

```bash
python scripts/deploy_changes.py               # изменения последнего пуша
python scripts/deploy_changes.py --all         # все файлы yaml/ и objects/
python scripts/deploy_changes.py --dry-run     # только показать список и порядок
python scripts/deploy_changes.py --files-file list.txt
```

## 5. Переменные GitLab

Settings → CI/CD → Variables.

| Переменная | Где видна | Флаги | Назначение |
|---|---|---|---|
| `DAMU_CICD_TOKEN` | только боевая ветка (scope / environment) | Masked, Protected | совпадает с параметром стенда `gitlab_objects_cicd_token`. Пока переменной нет, `deploy-objects` не создаётся |
| `DAMU_API_URL` | боевая ветка | Protected | необязательна: нужна, только если имя ветки не равно хосту стенда (например, `main`) |
| `GITLAB_MR_TOKEN` | ветка теста | Masked, Protected | токен с правом `api` для `promote_to_prod` (создать и влить MR) |
| `DEPLOY_ALL` | задаётся при ручном запуске | — | `1` — загрузить все файлы (первая синхронизация) |

Не задавайте `DAMU_CICD_TOKEN` на ветку теста: пайплайн загрузит объекты обратно в тот же стенд, из которого их выгрузили.

## 6. Приём на стенде

### REST gitlab_runner_accept_object

Публичный (`is_public = 1`), защищён токеном из параметра `gitlab_objects_cicd_token`.

```http
POST /restapi/services/run/gitlab_runner_accept_object
Authorization: CICDToken <gitlab_objects_cicd_token>
Content-Type: application/json

{
  "path": "yaml/core/rest_services/gitlab_runner_accept_log.yaml",
  "format": "yaml",
  "commit": "969c4589",
  "content": "objects:\n  - entity: rest_services\n ..."
}
```

`format`: `yaml` | `yml` | `json`; пусто — по расширению `path`. Для совместимости принимается и старый вариант: сырой JSON-объект в теле или сырой YAML с `?format=yaml&path=…`.

| Код | Ответ | Когда |
|---|---|---|
| 200 | `{success: true, path, format, patch_in_id, instance_id}` | объект загружен |
| 401 | `Invalid cicd token` | нет заголовка или токен не совпал |
| 400 | `Пустое содержимое`, `Неизвестный формат` | неверный запрос |
| 422 | `{success: false, error, debug, patch_in_id}` | ошибка разбора или импорта |
| 500 | `{success: false, error}` | не задан токен на стенде, БП не запустился |

Присланный файл сохраняется в `files` (каталог `CUST`) и привязывается к записи `patch_in` с заголовком `GitLab <commit>: <path>` — в «Принятых патчах» видно, что и из какого коммита приехало. БП запускается от пользователя 1 через `BPMSStartProcess`.

Не передавайте тело в БП через `JsonToString(request.input)`: берите строку `request.input.content` или `request.input_raw`.

### БП import_object_from_file

| Переменная | Вход/выход | Смысл |
|---|---|---|
| `data` | вход | содержимое; если пусто — форма выбора файла |
| `format` | вход | `json` / `yaml`; пусто — по первому символу (`{` или `[` → json, иначе yaml) |
| `enc` | вход | `base64` — `data` закодирована |
| `title` | вход | заголовок `patch_in`, если в файле нет `title` |
| `file_id` | вход | файл-источник для `patch_in` |
| `patch_in_id`, `last_error`, `debug` | выход | результат |

- Для YAML числа и `true/false` в `is_object` и `value` приводятся к строкам — файл, поправленный вручную, всё равно загрузится.
- Записи ищутся и обновляются `UpsertI` по ключевым атрибутам; ссылки `pk` переводятся в id (`pkg/object.pkToId`).
- Дочерние строки, которых нет в файле, удаляются (например, удалённые шаги БП).
- После загрузки публикуются затронутые сущности (`entity_publish`) и БП (`bp_process_publish`), сбрасывается кеш.
- YAML-файл можно загрузить и вручную: запустить БП и выбрать файл в форме.
- Шаг импорта вызывает `DMLI` и `Commit()`: из крона без открытой транзакции он падает с `SAVEPOINT can only be used in transaction blocks`. Тестировать импорт через REST (из крона — `http.post` на `http://` .. `GetHTTPListenHostPort()` .. `/restapi/services/run/gitlab_runner_accept_object`).

## 7. Настройка с нуля

**Боевой стенд**

1. Установить патчем REST `gitlab_runner_accept_object` и БП `import_object_from_file` с поддержкой YAML. Старая версия YAML не понимает.
2. Задать параметр `gitlab_objects_cicd_token` — длинный случайный токен, свой для каждого стенда.

**Тестовый стенд**

1. Заполнить параметры `gitlab_yaml_*` (раздел 3).

**Репозиторий**

1. Положить в обе ветки `.gitlab-ci.yml` и `scripts/deploy_changes.py` — одинаковые файлы, чтобы слияния не конфликтовали.
2. Сделать обе ветки protected (Settings → Repository → Protected branches); пушить и вливать в боевую — только мейнтейнерам.
3. Завести переменные из раздела 5.
4. Первая синхронизация: если боевой стенд отстаёт от теста, запустить пайплайн вручную на боевой ветке с `DEPLOY_ALL=1`. Если стенды уже совпадают — пропустить.

Новый стенд (например, предпрод) — ещё одна protected-ветка с именем по хосту и своя переменная токена со scope на неё.

## 8. Как выпускать доработку

1. Собрать доработанные объекты в патч на тесте.
2. Запустить БП «Выгрузка в gitlab (yaml)» (`patch_out_to_gitlab_yaml`) для патча. Проверить `debug`: там пропущенные объекты.
3. В GitLab появится коммит в ветке теста, `promote_to_prod` создаст и вольёт MR.
4. На боевой ветке отработает `deploy-objects`: в логе строка на каждый файл — `OK … patch_in_id=…` или `FAIL … HTTP 422: …`.
5. На боевом стенде проверить «Принятые патчи» и саму доработку.

## 9. Ошибки и диагностика

| Сообщение в логе job | Причина | Что делать |
|---|---|---|
| `Failed to resolve 'main'`, `Ветка 'main' не похожа на адрес стенда` | имя ветки не является хостом, адрес построен из имени ветки | задать `DAMU_API_URL` или назвать ветку по хосту |
| `Стенд … недоступен` | DNS, сеть, стенд выключен | проверить адрес и доступность стенда из GitLab |
| `HTTP 401: Invalid cicd token` | токен в GitLab ≠ `gitlab_objects_cicd_token` | выровнять значения |
| `HTTP 500 … gitlab_objects_cicd_token не задан` | на стенде нет параметра | создать параметр |
| `HTTP 422: Ошибка разбора YAML: yaml: line N …` | файл поправлен вручную с ошибкой (табы в отступах, незакрытые скобки) | исправить файл |
| `HTTP 422: В файле нет списка objects` | файл не в формате выгрузки или на стенде старая версия БП | проверить файл и версию БП |
| Параметр колонки ТЧ (`query_code` и т. п.) приехал без привязки к элементу | у `code` в `table_part_cols_a` / `table_part_cols_a_pv` нет `is_object = 1` на стенде-источнике | выставить ключ, выгрузить сущность заново |
| `HTTP 422: … Error on Upsert on <entity>` | не прошла запись: нет колонки, нарушен уникальный индекс, неверный тип | смотреть `debug` в логе и `patch_in` |
| `Нет изменённых объектов в yaml/ или objects/` | в пуше нет файлов объектов | норма; для полной загрузки — ручной запуск с `DEPLOY_ALL=1` |
| `Удалён в git, на стенде не удаляется` | файл удалили из репозитория | удалить объект на стенде вручную |
| job `deploy-objects` не появился | на ветке не видна `DAMU_CICD_TOKEN` или нет изменений в `yaml/`/`objects/` | проверить protected-ветку и scope переменной |

Ручная проверка REST без GitLab:

```bash
curl -sS -X POST "https://<стенд>/restapi/services/run/gitlab_runner_accept_object" \
  -H "Authorization: CICDToken $TOKEN" -H "Content-Type: application/json" \
  --data "$(python3 -c 'import json,sys;print(json.dumps({"path":sys.argv[1],"content":open(sys.argv[1]).read()}))' yaml/core/rest_services/x.yaml)"
```

## 10. Ограничения

- **Удаление объектов не переносится.** Удалённый в git файл на стенде ничего не удаляет (кроме лишних дочерних строк внутри загружаемого объекта).
- **Переводы строк.** После YAML в многострочных значениях (скрипты, шаблоны) `\r\n` становится `\n`, хвостовые пробелы строк удаляются.
- **Ссылки по ключу.** Если связанной записи на стенде-приёмнике нет, ссылка молча становится пустой. Порядок загрузки это частично снимает; новые справочники выгружайте вместе с объектами.
- **Длинные загрузки.** Крупная сущность или БП с публикацией могут идти дольше таймаута прокси перед стендом — тогда job получит 504, хотя импорт мог завершиться. Проверяйте `patch_in`.
- **Табличные части** переносятся вместе с сущностью, но страницы после загрузки нужно перегенерировать. `table_part_param_values` (старые ng-шаблоны) не переносится.
- **MCP-методы** (`mcp_method*`, `mcp_object*`) не попадают в патчи автоматически.
