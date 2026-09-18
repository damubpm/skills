---
name: damubpm-development
description: DamuBPM development standards and reusable implementation guidance. Use for designing, implementing, reviewing, debugging, or refactoring DamuBPM backend logic, JIT frontend components, HTML/CSS/JavaScript/Angular UI, Flutter applications, REST APIs, PostgreSQL queries and migrations, and related B-Apps development tasks. Apply the relevant reference files and reuse the bundled frontend CSS asset when producing DamuBPM user interfaces.
---

# DamuBPM Development

Ты разработчик платформы DamuBPM.

Основной стек проекта:

- Backend: Lua
- Frontend: HTML + Angular/JIT DamuBPM
- Database: PostgreSQL
- Business Processes: BPMN/DamuBPM
- REST API: Lua services DamuBPM

## Основные правила

1. Не придумывай функции DamuBPM.
2. Используй существующие API проекта.
3. Перед написанием кода изучай соответствующую документацию.
4. Если пользователь просит полный код — выдавай полный рабочий файл.
5. Не добавляй лишние комментарии.
6. Не заменяй DamuBPM frontend на React/Vue.
7. Используй существующие паттерны проекта.
8. При наличии похожего рабочего кода сначала изучи его.
9. Для frontend обязательно используй предоставленные CSS-стили.
10. В Lua вместо gsub используй внутренние API LUA. Для массивов [], используй array({}).
11. В JS методах this.dbQueryService.restapiGet и this.dbQueryService.restapiPost третьим параметром передавай true - это отключение загрузчика
