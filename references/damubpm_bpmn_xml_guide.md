[← На главную](index.html)

# BPMN XML в DamuBPM: практическая спецификация для автоматического редактирования ИИ

> **Назначение.** Этот документ описывает наблюдаемый XML-контракт BPMN в DamuBPM по двум реальным экспортам: `task_sign_cloud(1).xml` и `entity_publish(1).xml`. Цель — дать ИИ безопасный алгоритм добавления, удаления, соединения и перемещения BPMN-элементов без разрушения схемы и визуального представления.
>
> Это **не полная спецификация BPMN 2.0** и не описание всех внутренних таблиц DamuBPM. Правила ниже основаны на реально присутствующих конструкциях двух файлов и существующем DamuBPM BPM/Lua-контракте. То, чего в XML нет, нельзя автоматически додумывать.

---

## 1. Главная модель: один процесс хранится в двух синхронных слоях

В обоих файлах XML состоит из двух логических частей:

1. **Семантический граф процесса** внутри `<process>`:
   - события;
   - задачи;
   - подпроцессы;
   - шлюзы;
   - `sequenceFlow`;
   - ссылки `<incoming>` / `<outgoing>`.
2. **Графическое представление** внутри `<bpmndi:BPMNDiagram>` / `<bpmndi:BPMNPlane>`:
   - `BPMNShape` для каждого узла;
   - `BPMNEdge` для каждого `sequenceFlow`;
   - `omgdc:Bounds` для координат и размеров;
   - `omgdi:waypoint` для маршрута стрелок.

**Критическое правило для ИИ:** нельзя добавить только BPMN-узел или только картинку. Любое изменение графа должно согласованно менять оба слоя.

### Наблюдаемые показатели файлов

| Файл | Узлы | SequenceFlow | ScriptTask | UserTask | SubProcess | ExclusiveGateway | EndEvent |
|---|---:|---:|---:|---:|---:|---:|---:|
| `task_sign_cloud(1).xml` | 15 | 14 | 5 | 1 | 1 | 3 | 4 |
| `entity_publish(1).xml` | 30 | 29 | 22 | 0 | 4 | 1 | 2 |

В обоих примерах:

- все `sourceRef` / `targetRef` ссылаются на существующие узлы;
- все `incoming` / `outgoing` согласованы с `sequenceFlow`;
- каждому семантическому узлу соответствует ровно один `BPMNShape`;
- каждому `sequenceFlow` соответствует ровно один `BPMNEdge`.

Количество `sequenceFlow = nodes - 1` в обоих файлах — **наблюдение этих конкретных схем, а не универсальное правило BPMN**.

---

## 2. Корневой XML-каркас

В обоих файлах используется один и тот же набор namespace:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
  xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC"
  xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  targetNamespace=""
  xsi:schemaLocation="http://www.omg.org/spec/BPMN/20100524/MODEL http://www.omg.org/spec/BPMN/2.0/20100501/BPMN20.xsd">

  <process ...>
    <!-- семантический граф -->
  </process>

  <bpmndi:BPMNDiagram ...>
    <bpmndi:BPMNPlane ...>
      <!-- формы и линии -->
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</definitions>
```

### Наблюдаемые атрибуты process

```xml
<process
  id="sid-..."
  name="Customer"
  processType="None"
  isClosed="false"
  isExecutable="false">
```

**Правило:** при локальном редактировании не менять `process.id`, `name`, `processType`, `isClosed`, `isExecutable`, `targetNamespace` и namespace без отдельной причины. В частности, не переключать `isExecutable` в `true` только потому, что это кажется логичным по стандартному BPMN.

---

## 3. ID: сохранять стиль исходного файла, а не нормализовать

В предоставленных XML встречаются два стиля идентификаторов.

### UUID-подобный стиль

```text
ScriptTask_6c0312e2-d95b-4968-9862-210ead9a628f
SequenceFlow_11d31b8f-6782-45ad-8060-a4245154509b
ExclusiveGW_10b442de-6beb-4930-9ca7-fb9b09489ab9
```

### Короткий bpmn.js-подобный стиль

```text
ScriptTask_02pgcmn
SequenceFlow_1wwr10z
ExclusiveGateway_0dr9gt3
```

**Правила генерации ID:**

1. ID должен быть уникален во всём документе.
2. Сохранять локальный стиль конкретного XML.
3. Не переименовывать существующие ID без необходимости.
4. Не нормализовать `ExclusiveGW_...` в `ExclusiveGateway_...` или обратно — оба варианта реально встречаются.
5. Для DI использовать стабильное соответствие:
   - shape: `<elementId>_di`;
   - edge: `<sequenceFlowId>_di`.
6. `BPMNPlane/@bpmnElement` должен ссылаться на `process/@id`.

---

## 4. Семантические элементы, реально присутствующие в экспортах

### 4.1 StartEvent

Наблюдаемый шаблон:

```xml
<startEvent id="StartEvent_...">
  <outgoing>SequenceFlow_...</outgoing>
</startEvent>
```

В обоих файлах стартовое событие одно, без имени и с одним исходящим потоком.

### 4.2 EndEvent

```xml
<endEvent id="EndEvent_..." name="Завершено">
  <incoming>SequenceFlow_...</incoming>
</endEvent>
```

Имена используются как человекочитаемый результат: `Завершено`, `Ошибка`, `Не прошла проверка ЭЦП на сервере`, `Success`, `Error Found`.

### 4.3 ScriptTask

```xml
<scriptTask id="ScriptTask_..." name="Сформировать подписываемые данные">
  <incoming>SequenceFlow_IN</incoming>
  <outgoing>SequenceFlow_OUT</outgoing>
</scriptTask>
```

В одном примере встречается ScriptTask без `name`. Поэтому `name` фактически опционален в наблюдаемом формате.

**Важно:** внутри XML нет Lua-кода или `<script>`. Из этих файлов нельзя определить, где и как хранится тело ScriptTask. ИИ не должен вставлять `<script>` или `extensionElements`, пока такой формат не подтверждён отдельным рабочим примером DamuBPM.

### 4.4 UserTask

```xml
<userTask id="UserTask_..." name="Подписывание">
  <incoming>SequenceFlow_IN</incoming>
  <outgoing>SequenceFlow_OUT</outgoing>
</userTask>
```

XML-пример содержит только графовую часть UserTask. Настройки формы, акторов и обязательных переменных в этих XML не представлены.

По существующему DamuBPM BPM/Lua-контракту UserTask в рантайме имеет `task` UUID и продолжение выполняется через UserTask API/механизм задачи. Поэтому добавление `<userTask>` в XML **не означает**, что форма и права уже настроены.

### 4.5 SubProcess

```xml
<subProcess id="SubProcess_..." name="Согласовать">
  <incoming>SequenceFlow_IN</incoming>
  <outgoing>SequenceFlow_OUT</outgoing>
</subProcess>
```

В предоставленных XML `subProcess` не содержит вложенного BPMN-графа. Следовательно, нельзя автоматически предполагать стандартный embedded subprocess или добавлять дочерние элементы внутрь него без отдельного образца DamuBPM.

### 4.6 ExclusiveGateway

```xml
<exclusiveGateway id="ExclusiveGateway_...">
  <incoming>SequenceFlow_IN</incoming>
  <outgoing>SequenceFlow_OK</outgoing>
  <outgoing>SequenceFlow_ERROR</outgoing>
</exclusiveGateway>
```

В примерах gateway имеет один вход и два выхода.

В `entity_publish(1).xml` выходы названы:

```xml
<sequenceFlow id="SequenceFlow_1d0fmly"
              name="No errors"
              sourceRef="ExclusiveGateway_0dr9gt3"
              targetRef="EndEvent_00pun4z" />

<sequenceFlow id="SequenceFlow_046r2np"
              name="If Error"
              sourceRef="ExclusiveGateway_0dr9gt3"
              targetRef="EndEvent_1ajzp6r" />
```

Но `conditionExpression` в XML отсутствует. Поэтому `name` ветки — подпись, а не доказательство того, что условие исполнения хранится в этом XML.

---

## 5. SequenceFlow: единый источник связности должен быть согласован в трёх местах

Каждая стрелка представлена одновременно:

1. `<outgoing>` у source-узла;
2. `<incoming>` у target-узла;
3. `<sequenceFlow sourceRef="..." targetRef="..."/>`.

Пример:

```xml
<scriptTask id="ScriptTask_A" name="A">
  <outgoing>SequenceFlow_AB</outgoing>
</scriptTask>

<scriptTask id="ScriptTask_B" name="B">
  <incoming>SequenceFlow_AB</incoming>
</scriptTask>

<sequenceFlow id="SequenceFlow_AB"
              sourceRef="ScriptTask_A"
              targetRef="ScriptTask_B" />
```

### Инвариант

Для каждого `sequenceFlow F`:

```text
F.sourceRef = A  => A содержит <outgoing>F</outgoing>
F.targetRef = B  => B содержит <incoming>F</incoming>
```

И наоборот, каждый ID, упомянутый в `incoming/outgoing`, должен существовать как `sequenceFlow` с правильным направлением.

**Запрещено:** менять только `sourceRef/targetRef` и забывать про `incoming/outgoing`.

---

## 6. Порядок XML-элементов не определяет порядок выполнения

В `process` элементы расположены не строго топологически: `sequenceFlow` может стоять раньше или позже связанных task. Следовательно:

- нельзя определять предыдущий/следующий шаг по соседству XML-тегов;
- нельзя вставлять новый узел «между строками» и считать этого достаточным;
- граф восстанавливается только по `sourceRef`, `targetRef`, `incoming`, `outgoing`.

При редактировании допустимо сохранять существующий порядок XML и добавлять новые элементы рядом с логически связанными блоками, но семантика от этого не меняется.

---

## 7. BPMN-DI: графическое представление

### 7.1 BPMNDiagram и BPMNPlane

```xml
<bpmndi:BPMNDiagram id="sid-...">
  <bpmndi:BPMNPlane id="sid-..." bpmnElement="PROCESS_ID">
    ...
  </bpmndi:BPMNPlane>
</bpmndi:BPMNDiagram>
```

`bpmnElement` у plane должен совпадать с `process/@id`.

### 7.2 BPMNShape

```xml
<bpmndi:BPMNShape id="ScriptTask_X_di" bpmnElement="ScriptTask_X">
  <omgdc:Bounds x="100" y="220" width="100" height="80" />
</bpmndi:BPMNShape>
```

Наблюдаемые размеры:

| Тип | width | height | Дополнительно |
|---|---:|---:|---|
| `startEvent` | 36 | 36 | внешний `BPMNLabel` присутствует |
| `endEvent` | 36 | 36 | внешний `BPMNLabel` присутствует |
| `scriptTask` | 100 | 80 | отдельного `BPMNLabel` нет |
| `userTask` | 100 | 80 | отдельного `BPMNLabel` нет |
| `subProcess` | 100 | 80 | отдельного `BPMNLabel` нет |
| `exclusiveGateway` | 50 | 50 | `isMarkerVisible="true"`, внешний label |

Для нового элемента лучше использовать эти размеры, пока конкретный исходный файл не показывает иной локальный стиль.

### 7.3 Gateway shape

```xml
<bpmndi:BPMNShape
  id="ExclusiveGateway_X_di"
  bpmnElement="ExclusiveGateway_X"
  isMarkerVisible="true">
  <omgdc:Bounds x="660" y="235" width="50" height="50" />
  <bpmndi:BPMNLabel>
    <omgdc:Bounds x="640" y="285" width="90" height="20" />
  </bpmndi:BPMNLabel>
</bpmndi:BPMNShape>
```

### 7.4 Event label

Для start/end event в обоих примерах используется внешний label 90×20. Обычно он расположен под событием:

```xml
<bpmndi:BPMNLabel>
  <omgdc:Bounds x="CENTER_X_MINUS_45"
                y="EVENT_Y_PLUS_36"
                width="90"
                height="20" />
</bpmndi:BPMNLabel>
```

### 7.5 BPMNEdge

```xml
<bpmndi:BPMNEdge id="SequenceFlow_X_di" bpmnElement="SequenceFlow_X">
  <omgdi:waypoint xsi:type="omgdc:Point" x="SOURCE_X" y="SOURCE_Y" />
  <omgdi:waypoint xsi:type="omgdc:Point" x="TARGET_X" y="TARGET_Y" />
  <bpmndi:BPMNLabel>
    <omgdc:Bounds x="..." y="..." width="90" height="20" />
  </bpmndi:BPMNLabel>
</bpmndi:BPMNEdge>
```

Все flow в обоих примерах имеют `BPMNLabel`, даже когда `sequenceFlow/@name` отсутствует.

### 7.6 Waypoints

Наблюдаются:

- прямые связи — 2 waypoint;
- ортогональные повороты — 3 waypoint.

Типичная горизонтальная связь справа налево/слева направо использует точки на границах фигур, а не в центрах.

Для task 100×80:

```text
left center  = (x, y + 40)
right center = (x + 100, y + 40)
top center   = (x + 50, y)
bottom center= (x + 50, y + 80)
```

Для gateway 50×50:

```text
left center  = (x, y + 25)
right center = (x + 50, y + 25)
top center   = (x + 25, y)
bottom center= (x + 25, y + 50)
```

Для event 36×36:

```text
left center  = (x, y + 18)
right center = (x + 36, y + 18)
top center   = (x + 18, y)
bottom center= (x + 18, y + 36)
```

**Правило:** первый waypoint должен лежать на стороне source-элемента, последний — на стороне target-элемента. Промежуточные точки используются для обхода фигур и ортогональных поворотов.

---

## 8. Что нельзя выводить из этих XML

В предоставленных файлах **нет достаточных данных**, чтобы описать формат хранения следующих вещей:

- Lua-код ScriptTask;
- `conditionExpression` gateway;
- actor/role настройки UserTask;
- HTML/Angular форма UserTask;
- входные/выходные переменные точек;
- содержимое/код вызываемого SubProcess;
- таймеры, message/signal events, boundary events;
- parallel/inclusive gateway;
- serviceTask, manualTask, callActivity;
- DamuBPM-специфические `extensionElements`.

Если задача требует что-либо из этого, ИИ должен:

1. найти отдельный рабочий пример или документацию DamuBPM;
2. использовать существующий DamuBPM API/MCP/сущности;
3. не придумывать XML-атрибуты и расширения на основе общего знания BPMN 2.0.

---

## 9. Безопасный алгоритм вставки нового элемента между A и B

Пусть существует связь:

```text
A --F1--> B
```

Нужно вставить новый элемент `N`:

```text
A --F1--> N --F2--> B
```

### Предпочтительный вариант: минимальная мутация

Сохраняем существующий `F1`, создаём только новый `F2`.

#### Шаг 1. Найти прямой flow

```xml
<sequenceFlow id="F1" sourceRef="A" targetRef="B" />
```

#### Шаг 2. Перенаправить F1 на N

```xml
<sequenceFlow id="F1" sourceRef="A" targetRef="N" />
```

`A/<outgoing>F1</outgoing>` остаётся без изменений.

#### Шаг 3. Создать N

Для ScriptTask:

```xml
<scriptTask id="N" name="Новый шаг">
  <incoming>F1</incoming>
  <outgoing>F2</outgoing>
</scriptTask>
```

#### Шаг 4. Изменить incoming у B

Было:

```xml
<incoming>F1</incoming>
```

Стало:

```xml
<incoming>F2</incoming>
```

#### Шаг 5. Создать F2

```xml
<sequenceFlow id="F2" sourceRef="N" targetRef="B" />
```

#### Шаг 6. Добавить BPMNShape N

```xml
<bpmndi:BPMNShape id="N_di" bpmnElement="N">
  <omgdc:Bounds x="..." y="..." width="100" height="80" />
</bpmndi:BPMNShape>
```

#### Шаг 7. Обновить BPMNEdge F1

Сохранить `F1_di`, но его последний waypoint должен теперь упираться в `N`.

#### Шаг 8. Создать BPMNEdge F2

Новая линия от `N` к `B`, включая `BPMNLabel`.

### Почему этот вариант предпочтителен

- сохраняется существующий ID F1;
- source A вообще не нужно менять;
- меньше вероятность сломать внешние ссылки на существующий flow;
- дифф XML меньше.

### Особый случай: именованная ветка gateway

Если `F1` выходит из gateway и имеет `name="If Error"`, сохранение F1 на сегменте `gateway -> N` обычно сохраняет смысл подписи ветки. Не переносить имя автоматически на `N -> B` без причины.

---

## 10. Добавление нового ScriptTask

### Семантика

```xml
<scriptTask id="ScriptTask_NEW" name="Новая обработка">
  <incoming>SequenceFlow_IN</incoming>
  <outgoing>SequenceFlow_OUT</outgoing>
</scriptTask>
```

### DI

```xml
<bpmndi:BPMNShape id="ScriptTask_NEW_di" bpmnElement="ScriptTask_NEW">
  <omgdc:Bounds x="X" y="Y" width="100" height="80" />
</bpmndi:BPMNShape>
```

**После XML-правки:** отдельно настроить исполняемую логику ScriptTask штатным механизмом DamuBPM. XML из примеров не содержит тело скрипта.

---

## 11. Добавление UserTask

### Семантика

```xml
<userTask id="UserTask_NEW" name="Проверить документ">
  <incoming>SequenceFlow_IN</incoming>
  <outgoing>SequenceFlow_OUT</outgoing>
</userTask>
```

### DI

```xml
<bpmndi:BPMNShape id="UserTask_NEW_di" bpmnElement="UserTask_NEW">
  <omgdc:Bounds x="X" y="Y" width="100" height="80" />
</bpmndi:BPMNShape>
```

**После XML-правки:** отдельно проверить/создать форму, акторов и контракт входных/выходных переменных. В существующей документации DamuBPM UserTask работает через `task` UUID; XML-фрагмент сам по себе не настраивает этот runtime-контракт.

---

## 12. Добавление SubProcess

```xml
<subProcess id="SubProcess_NEW" name="Согласование">
  <incoming>SequenceFlow_IN</incoming>
  <outgoing>SequenceFlow_OUT</outgoing>
</subProcess>
```

```xml
<bpmndi:BPMNShape id="SubProcess_NEW_di" bpmnElement="SubProcess_NEW">
  <omgdc:Bounds x="X" y="Y" width="100" height="80" />
</bpmndi:BPMNShape>
```

Не добавлять внутрь `<subProcess>` вложенный граф, `calledElement` или DamuBPM extension без отдельного подтверждённого примера.

---

## 13. Добавление ExclusiveGateway с двумя ветками

Целевая структура:

```text
A -> G -> B
     └-> E
```

### Семантика

```xml
<exclusiveGateway id="ExclusiveGateway_NEW">
  <incoming>SequenceFlow_AG</incoming>
  <outgoing>SequenceFlow_GB</outgoing>
  <outgoing>SequenceFlow_GE</outgoing>
</exclusiveGateway>

<sequenceFlow id="SequenceFlow_GB"
              name="Успех"
              sourceRef="ExclusiveGateway_NEW"
              targetRef="B" />

<sequenceFlow id="SequenceFlow_GE"
              name="Ошибка"
              sourceRef="ExclusiveGateway_NEW"
              targetRef="E" />
```

### DI

```xml
<bpmndi:BPMNShape
  id="ExclusiveGateway_NEW_di"
  bpmnElement="ExclusiveGateway_NEW"
  isMarkerVisible="true">
  <omgdc:Bounds x="X" y="Y" width="50" height="50" />
  <bpmndi:BPMNLabel>
    <omgdc:Bounds x="X_MINUS_20" y="Y_PLUS_50" width="90" height="20" />
  </bpmndi:BPMNLabel>
</bpmndi:BPMNShape>
```

**Критично:** названия `Успех`/`Ошибка` — только подписи веток. Условие маршрутизации нельзя выдумывать как `conditionExpression`, потому что такого механизма нет в предоставленных XML.

---

## 14. Добавление EndEvent

```xml
<endEvent id="EndEvent_NEW" name="Ошибка">
  <incoming>SequenceFlow_TO_END</incoming>
</endEvent>
```

```xml
<bpmndi:BPMNShape id="EndEvent_NEW_di" bpmnElement="EndEvent_NEW">
  <omgdc:Bounds x="X" y="Y" width="36" height="36" />
  <bpmndi:BPMNLabel>
    <omgdc:Bounds x="X_MINUS_27" y="Y_PLUS_36" width="90" height="20" />
  </bpmndi:BPMNLabel>
</bpmndi:BPMNShape>
```

Новый входящий `sequenceFlow` должен быть отражён и в `<incoming>`, и в `BPMNEdge`.

---

## 15. Безопасное удаление обычного узла с одним входом и одним выходом

Исходно:

```text
A --F1--> N --F2--> B
```

После удаления:

```text
A --F1--> B
```

Алгоритм:

1. убедиться, что у `N` ровно один incoming и один outgoing;
2. изменить `F1.targetRef` с `N` на `B`;
3. у `B` заменить `<incoming>F2</incoming>` на `<incoming>F1</incoming>`;
4. удалить `F2`;
5. удалить `N`;
6. удалить `N_di`;
7. удалить `F2_di`;
8. обновить последний waypoint `F1_di`, чтобы он заканчивался на `B`;
9. выполнить полный валидатор связности.

**Не применять автоматически** к gateway, узлу с несколькими входами/выходами или любому участку, где удаление меняет ветвление. Там требуется явное перестроение графа.

---

## 16. Перемещение элемента без изменения логики

Если нужно только изменить положение блока:

1. поменять `x/y` его `omgdc:Bounds`;
2. не менять semantic XML;
3. пересчитать первый/последний waypoint всех подключённых edge;
4. при необходимости добавить промежуточные waypoint для обхода;
5. обновить внешний `BPMNLabel` события/gateway;
6. не менять ID.

**Главное:** перемещение — DI-операция, а не бизнес-операция.

---

## 17. Переименование элемента

Для task/subprocess/endEvent:

```xml
name="Новое название"
```

При переименовании:

- ID не менять;
- sequenceFlow не менять;
- DI shape не менять;
- для event при длинной подписи можно скорректировать `BPMNLabel/Bounds`, но это визуальная правка.

Для sequenceFlow изменение `name` меняет только подпись ветки в наблюдаемом XML. Не считать это изменением runtime-условия.

---

## 18. Геометрия автоматической вставки

### Горизонтальная цепочка

Если `A` и `B` расположены на одной горизонтали и между ними достаточно места:

```text
N.x = midpoint(A.right, B.left) - N.width / 2
N.y = common_center_y - N.height / 2
```

Для task/subprocess используйте 100×80.

Если места недостаточно:

1. сдвинуть B и все последующие элементы вправо на безопасный шаг;
2. сохранить существующую вертикальную полосу/ряд;
3. обновить connected edge.

### Ветка вверх/вниз от gateway

Предпочтительно:

- gateway оставить в основной линии;
- альтернативный end/task поставить вертикально выше или ниже;
- исходный waypoint взять из `top center` или `bottom center` gateway;
- последний waypoint — соответствующая ближайшая сторона target;
- при необходимости использовать один ортогональный поворот.

### Не рассчитывать layout по BPMNLabel

В исходных файлах координаты некоторых `BPMNLabel` у flow визуально не совпадают с серединой соответствующей линии. Поэтому label bounds нельзя использовать как источник геометрии графа. Источники геометрии — `BPMNShape/Bounds` и `BPMNEdge/waypoint`.

---

## 19. Алгоритм ИИ перед изменением XML

### Фаза A. Разбор

1. распарсить XML с namespace;
2. найти единственный/целевой `<process>`;
3. собрать `nodesById` по всем узлам кроме `sequenceFlow`;
4. собрать `flowsById`;
5. построить `incomingByNode` и `outgoingByNode` из `sourceRef/targetRef`;
6. найти `BPMNPlane`;
7. собрать `shapesByElement`;
8. собрать `edgesByFlow`;
9. определить локальный стиль ID;
10. определить локальные размеры и типичную сетку координат.

### Фаза B. Проверка до правки

Перед изменением убедиться, что исходник уже согласован. Если нет — не маскировать проблему новой правкой.

### Фаза C. Минимальная мутация

- изменять только нужные semantic элементы;
- сохранять существующие ID;
- создавать минимальное число новых flow;
- отдельно синхронизировать DI.

### Фаза D. Полная валидация

После изменения выполнить проверки из раздела 20.

### Фаза E. Отчёт

ИИ должен сообщить:

- какие узлы добавлены/удалены;
- какие flow изменены/созданы;
- какие runtime-настройки **не были** заданы через XML;
- прошла ли структурная проверка.

---

## 20. Обязательный валидатор после каждой правки

### 20.1 XML

- документ парсится без ошибок;
- исходные namespace сохранены;
- XML остаётся UTF-8.

### 20.2 Уникальность ID

- все `id` уникальны;
- нет конфликта semantic ID и DI ID;
- новые ID не совпадают со старыми.

### 20.3 SequenceFlow

Для каждого flow:

- `sourceRef` существует;
- `targetRef` существует;
- source содержит `<outgoing>flowId</outgoing>`;
- target содержит `<incoming>flowId</incoming>`.

Для каждого `<incoming>/<outgoing>`:

- flow существует;
- направление совпадает.

### 20.4 DI

- каждый узел имеет `BPMNShape`;
- каждый flow имеет `BPMNEdge`;
- каждый `BPMNShape/@bpmnElement` существует;
- каждый `BPMNEdge/@bpmnElement` существует и является `sequenceFlow`;
- `BPMNPlane/@bpmnElement == process/@id`;
- `Bounds` содержит числовые `x/y/width/height`;
- `width > 0`, `height > 0`;
- каждый edge имеет минимум 2 waypoint;
- waypoint имеют числовые `x/y`.

### 20.5 Базовая логика наблюдаемых типов

Для автоматически создаваемого обычного шага:

- ScriptTask/UserTask/SubProcess: обычно 1 incoming + 1 outgoing;
- StartEvent: outgoing, без incoming;
- EndEvent: incoming, без outgoing;
- ExclusiveGateway: в наблюдаемом паттерне 1 incoming + 2 outgoing.

Это шаблоны данных файлов, а не запрет на другие валидные BPMN-конструкции.

---

## 21. Машинный псевдокод проверки ссылок

```text
nodes = all process children except sequenceFlow
flows = all sequenceFlow

for flow in flows:
    assert flow.sourceRef in nodes
    assert flow.targetRef in nodes
    assert flow.id in nodes[flow.sourceRef].outgoing
    assert flow.id in nodes[flow.targetRef].incoming

for node in nodes:
    for flowId in node.incoming:
        assert flowId in flows
        assert flows[flowId].targetRef == node.id

    for flowId in node.outgoing:
        assert flowId in flows
        assert flows[flowId].sourceRef == node.id

for node in nodes:
    assert exactly_one_shape_for(node.id)

for flow in flows:
    assert exactly_one_edge_for(flow.id)
```

---

## 22. XML-фрагмент: новый ScriptTask между двумя существующими задачами

Исходно:

```xml
<scriptTask id="ScriptTask_A" name="A">
  <outgoing>SequenceFlow_AB</outgoing>
</scriptTask>

<scriptTask id="ScriptTask_B" name="B">
  <incoming>SequenceFlow_AB</incoming>
</scriptTask>

<sequenceFlow id="SequenceFlow_AB"
              sourceRef="ScriptTask_A"
              targetRef="ScriptTask_B" />
```

После вставки:

```xml
<scriptTask id="ScriptTask_A" name="A">
  <outgoing>SequenceFlow_AB</outgoing>
</scriptTask>

<scriptTask id="ScriptTask_NEW" name="Новый шаг">
  <incoming>SequenceFlow_AB</incoming>
  <outgoing>SequenceFlow_NEW_B</outgoing>
</scriptTask>

<scriptTask id="ScriptTask_B" name="B">
  <incoming>SequenceFlow_NEW_B</incoming>
</scriptTask>

<sequenceFlow id="SequenceFlow_AB"
              sourceRef="ScriptTask_A"
              targetRef="ScriptTask_NEW" />

<sequenceFlow id="SequenceFlow_NEW_B"
              sourceRef="ScriptTask_NEW"
              targetRef="ScriptTask_B" />
```

DI:

```xml
<bpmndi:BPMNShape id="ScriptTask_NEW_di" bpmnElement="ScriptTask_NEW">
  <omgdc:Bounds x="500" y="220" width="100" height="80" />
</bpmndi:BPMNShape>

<bpmndi:BPMNEdge id="SequenceFlow_AB_di" bpmnElement="SequenceFlow_AB">
  <omgdi:waypoint xsi:type="omgdc:Point" x="450" y="260" />
  <omgdi:waypoint xsi:type="omgdc:Point" x="500" y="260" />
  <bpmndi:BPMNLabel>
    <omgdc:Bounds x="430" y="250" width="90" height="20" />
  </bpmndi:BPMNLabel>
</bpmndi:BPMNEdge>

<bpmndi:BPMNEdge id="SequenceFlow_NEW_B_di" bpmnElement="SequenceFlow_NEW_B">
  <omgdi:waypoint xsi:type="omgdc:Point" x="600" y="260" />
  <omgdi:waypoint xsi:type="omgdc:Point" x="650" y="260" />
  <bpmndi:BPMNLabel>
    <omgdc:Bounds x="580" y="250" width="90" height="20" />
  </bpmndi:BPMNLabel>
</bpmndi:BPMNEdge>
```

---

## 23. Разбор `task_sign_cloud(1).xml` как эталона ветвления

Высокоуровневая цепочка:

```text
Start
  -> Поиск Export Templates
  -> Поиск файлов в сущности
  -> Сформировать подписываемые данные
  -> Gateway
       -> Ошибка
       -> Подписывание (UserTask)
          -> Проверка ЭЦП на сервере
          -> ScriptTask без name
          -> Gateway
               -> Не прошла проверка ЭЦП на сервере
               -> Согласовать (SubProcess)
                  -> Gateway
                       -> Ошибка
                       -> Завершено
```

Этот файл полезен как образец для:

- UserTask;
- нескольких ExclusiveGateway;
- вертикальных error-веток;
- SubProcess;
- русского `name`;
- UUID-подобных ID;
- прямых edge из 2 waypoint.

---

## 24. Разбор `entity_publish(1).xml` как эталона длинного процесса

Файл показывает длинную цепочку из ScriptTask/SubProcess, разложенную в два горизонтальных ряда. В конце:

```text
... -> ClearCache -> ExclusiveGateway
                     -> "No errors" -> Success
                     -> "If Error"  -> Error Found
```

Полезные особенности:

- короткие ID (`ScriptTask_...`, `SequenceFlow_...`);
- 22 ScriptTask;
- несколько SubProcess;
- именованные ветки sequenceFlow;
- 3-точечные ортогональные edge на финальном gateway;
- смешанная геометрия слева-направо и справа-налево;
- отсутствие runtime condition в самом XML.

---

## 25. Что ИИ должен сохранять при локальной правке

По умолчанию сохранять неизменными:

- XML declaration;
- namespace и `schemaLocation`;
- process ID и BPMNPlane link;
- все не затронутые IDs;
- все не затронутые `name`;
- существующую ID-конвенцию;
- существующие координаты не затронутых фигур;
- существующие flow labels;
- порядок веток gateway, если пользователь не просил его менять;
- неизвестные атрибуты/узлы, если они появятся в других файлах.

**Не выполнять глобальное форматирование или переименование всей схемы ради красоты.** Для автоматизированной разработки важнее минимальный дифф.

---

## 26. Что ИИ не должен делать

1. Не добавлять элемент только в `<process>` без DI.
2. Не добавлять DI без semantic элемента.
3. Не менять `sourceRef/targetRef` без `incoming/outgoing`.
4. Не генерировать `conditionExpression`, `extensionElements`, `<script>` по догадке.
5. Не считать `sequenceFlow/@name` условием выполнения.
6. Не менять все ID на UUID или короткий формат.
7. Не менять `isExecutable` автоматически.
8. Не считать порядок XML-тегов порядком процесса.
9. Не использовать координаты `BPMNLabel` для восстановления топологии.
10. Не удалять gateway как обычный узел с одним входом/выходом.
11. Не перезаписывать весь XML, если нужна локальная вставка.
12. Не считать, что добавленный UserTask уже имеет форму и акторов.
13. Не считать, что добавленный ScriptTask уже имеет Lua-код.

---

## 27. Рекомендованный контракт команды для будущего ИИ

Для автоматической операции удобно мыслить структурой:

```text
operation: insert_node
node_type: scriptTask | userTask | subProcess | exclusiveGateway | endEvent
name: "..."
insert_after: EXISTING_ELEMENT_ID
insert_before: EXISTING_ELEMENT_ID
branch_name: optional
placement: auto | {x, y}
```

Алгоритм ИИ:

```text
1. parse
2. validate source
3. locate source/target/flow
4. generate IDs in local style
5. mutate semantic graph minimally
6. mutate DI
7. validate all references
8. serialize UTF-8
9. report runtime metadata still required
```

Это не формат API DamuBPM, а внутренний безопасный алгоритм для агента/навыка.

---

## 28. Связь с runtime DamuBPM

XML отвечает за структуру процесса, но выполнение DamuBPM дополнительно связано с runtime-механизмом процесса.

Из существующей документации DamuBPM:

- процесс запускается через `BPMSStartProcess` / `BPMSStartProcess2`;
- UserTask/ожидающая задача идентифицируется `task` UUID;
- продолжение задачи выполняется через `BPMSRunManualTask` или UserTask REST-контракт;
- процесс использует `var` для глобальных переменных и `sys` для системного контекста.

**Практический вывод:** BPMN XML-правка должна отвечать только за подтверждённую часть модели. ScriptTask-код, условия, формы, акторы и переменные настраиваются тем механизмом DamuBPM, который реально хранит эти данные. Если такого механизма/примера нет в контексте — остановиться на корректной структурной правке и явно отметить недостающую runtime-настройку.

---

## 29. Финальный чек-лист для автономной работы ИИ

Перед сохранением изменённого BPMN XML убедиться:

- [ ] XML парсится.
- [ ] Namespace не потеряны.
- [ ] Все ID уникальны.
- [ ] Существующие ID без необходимости не изменены.
- [ ] Каждый `sequenceFlow` ссылается на существующие source/target.
- [ ] Каждый source содержит правильный `outgoing`.
- [ ] Каждый target содержит правильный `incoming`.
- [ ] Каждый узел имеет `BPMNShape`.
- [ ] Каждый flow имеет `BPMNEdge`.
- [ ] `BPMNPlane/@bpmnElement` совпадает с `process/@id`.
- [ ] Размеры новых элементов соответствуют локальному паттерну.
- [ ] Edge начинаются/заканчиваются на границах соответствующих фигур.
- [ ] Gateway имеет `isMarkerVisible="true"`.
- [ ] Для event/gateway добавлены внешние labels по локальному стилю.
- [ ] Для flow добавлен `BPMNLabel` по локальному стилю.
- [ ] Не добавлены неподтверждённые DamuBPM extension/script/condition поля.
- [ ] Пользователю/следующему агенту сообщено, какие runtime-настройки ещё нужны.

---

## 30. Короткая памятка для навыка `damubpm-development`

```text
DamuBPM BPMN XML = semantic graph + BPMN-DI.

При любой правке:
- semantic node <-> BPMNShape
- sequenceFlow <-> BPMNEdge
- sourceRef <-> source/outgoing
- targetRef <-> target/incoming

Наблюдаемые размеры:
- event 36x36
- task/userTask/subProcess 100x80
- exclusiveGateway 50x50 + isMarkerVisible=true

Не придумывать:
- Lua script в XML
- conditionExpression
- UserTask actors/forms
- SubProcess internals
- extensionElements

Вставка A->B:
- сохранить F1 и retarget A->N
- создать F2 N->B
- заменить incoming B: F1 -> F2
- добавить N incoming=F1/outgoing=F2
- добавить N_di, обновить F1_di, создать F2_di
- прогнать полный валидатор ссылок и DI
```

---

[← На главную](index.html)
