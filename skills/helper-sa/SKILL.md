---
name: helper-sa
description: Работа с проектами и артефактами «Помощника SA» (https://helper-sa.ru) через MCP-сервер helper-sa — схемы БД DBML, UML PlantUML, спецификации OpenAPI и AsyncAPI, BPMN, XSD, WSDL, XML, JSON Schema, YAML, JSON, Markdown, AsciiDoc. Используй, когда пользователь просит найти, прочитать, создать или изменить артефакты своих проектов «Помощника SA», разложить их по папкам или откатить изменения.
---

# Помощник SA: проекты и артефакты через MCP

> Версия навыка: `237cab87`. Обновить навык и подключение: попроси агента выполнить в режиме обновления
> инструкцию https://raw.githubusercontent.com/vt244/helper-sa-mcp/main/helper-sa-mcp-setup.md

«Помощник SA» — платформа системного аналитика. Работа с ней идёт через MCP-сервер `helper-sa`
(https://helper-sa.ru/api/mcp). Права — как у владельца API-ключа; каждое изменение содержимого создаёт версию
артефакта с источником **MCP**.

## Инструменты сервера

| Инструмент | Назначение |
|---|---|
| `whoami`, `list_tools` | Владелец ключа, тариф и срок его действия; коды инструментов |
| `list_projects`, `search_projects`, `get_project` | Проекты: все, поиск (название/описание, роль, период, сортировка), один |
| `create_project`, `update_project`, `delete_project` | Создание, изменение, удаление проекта (только владелец) |
| `get_project_tree` | Папки и артефакты проекта с путями «папка/файл» |
| `search_folders`, `get_folder` | Поиск папок; папка с содержимым (`recursive` — на любую глубину) |
| `create_folder`, `update_folder`, `delete_folder`, `place_artifact` | Папки и размещение артефактов |
| `list_artifacts`, `search_artifacts`, `get_artifact` | Артефакты проекта или личные; поиск по названию, пути и тексту (`inContent`); чтение с исходным текстом |
| `edit_artifact` | Точечные правки: замена фрагментов `oldText` → `newText` без передачи файла целиком |
| `create_artifact`, `update_artifact`, `move_artifact`, `delete_artifact` | Создание, изменение целиком, перенос, удаление |
| `validate_artifact` | Проверка синтаксиса без ИИ — те же правила, что в редакторе (по `id` или по тексту `source`) |
| `list_versions`, `get_version`, `diff_versions`, `restore_version` | История версий (номер, автор, источник), разница между версиями, откат |

Коды инструментов: `dbml`, `plantuml`, `openapi`, `asyncapi`, `bpmn`, `xsd`, `wsdl`, `xml`,
`jsonschema`, `yaml`, `json`, `markdown`, `asciidoc`.

Операции с существующим артефактом (`get_artifact`, `edit_artifact`, `update_artifact`, `validate_artifact`,
`move_artifact`, `place_artifact`, `delete_artifact`, версии) принимают `id`; `tool` можно не передавать — сервер
определит инструмент по `id`. `tool` обязателен только в `create_artifact` и в `validate_artifact` с текстом `source`.

## Порядок работы

1. Найди проект: `search_projects` (по названию) или `list_projects`. Если вариантов несколько — уточни.
2. Посмотри структуру: `get_project_tree` (или `get_folder` для нужной папки). Нужный файл или место в тексте
   ищи через `search_artifacts` (`inContent: true` вернёт строки с номерами). Пути «папка/файл» — это адреса для ссылок между файлами
   (`$ref` в OpenAPI/AsyncAPI/JSON Schema, `xs:include`/`xs:import` в XSD, `include::` в AsciiDoc,
   ссылки в Markdown). Ссылки указывай относительно папки файла.
3. Прочитай артефакт (`get_artifact`) **перед** изменением и меняй на основе актуального текста; запомни
   номер версии `version`.
4. Изменяй точечно через `edit_artifact` (`oldText` — фрагмент из текста точно, с отступами и соседними
   строками для однозначности), а если меняется почти весь файл — `update_artifact` с текстом **целиком**.
   Всегда передавай `baseVersion` = `version` из `get_artifact`. Сохраняй стиль, порядок, комментарии
   и всё, что не относится к задаче.
5. Ответ на изменение содержит проверку синтаксиса (`validation`): если `valid: false` — исправь ошибки
   (строка и сообщение указаны) следующей правкой. Перед сохранением большого текста можно проверить его
   `validate_artifact` с `source` (для связанных файлов — с `projectId` и `path`). В Markdown и AsciiDoc
   проверяются ссылки на файлы проекта (путь — от папки документа, `/` — от корня проекта) и на разделы `#якорь`.
6. `VERSION_CONFLICT` — пока ты работал, файл изменил пользователь или другой агент: перечитай артефакт,
   посмотри, что изменилось (`diff_versions`), и повтори правку поверх актуального текста. Не перезаписывай
   чужие изменения без `baseVersion`.
7. Новые артефакты создавай в нужном проекте и папке: `create_artifact` с `projectId` и `folderId`; без
   `folderId` — в корне проекта, без `projectId` — в личном пространстве. Новая папка — `create_folder`
   (`parentId` — вложенная): в ответе её `id`, он и есть `folderId` для `create_artifact`. Название
   артефакта — имя файла с расширением (`orders.yaml`, `common.xsd`, `schema.dbml`).
8. Если изменение оказалось ошибочным — `list_versions`, `diff_versions` и `restore_version` (создаётся новая
   версия, история не теряется).
9. В конце кратко перечисли, что создано и изменено (проект, путь, номер новой версии); по просьбе покажи
   `diff_versions`.

## Правила

- Удаляй (артефакты, папки, проекты) только по явной просьбе пользователя и после подтверждения.
- Не меняй артефакты, о которых пользователь не просил; большие правки согласуй заранее.
- Соблюдай синтаксис инструмента: корректный YAML/JSON для спецификаций, корректный XML для XSD/WSDL/BPMN,
  блок `@startuml … @enduml` для PlantUML. Не оставляй артефакт с ошибками `validation`.
- **BPMN** — это не только процесс, но и его схема: без раздела `bpmndi` редактор ничего не покажет.
  - `bpmndi:BPMNDiagram` → `bpmndi:BPMNPlane` (`bpmnElement` — collaboration, если есть пулы, иначе process);
    в нём `BPMNShape` с `dc:Bounds` для каждого события, задачи, шлюза, пула и дорожки и `BPMNEdge` с `di:waypoint`
    для каждого `sequenceFlow` и `messageFlow`. Объяви пространства имён `bpmndi`, `dc`, `di`.
  - Пулы — `participant` в `collaboration`; потоки сообщений между пулами — `messageFlow` **внутри
    `collaboration`**, не в `definitions` и не в `process`.
  - `incoming`/`outgoing` элемента ссылаются только на `sequenceFlow` его процесса; `messageFlow` в них не указывай.
  - Перенос строки в `name` — `&#10;`, а не `\n`.
  - Для большой диаграммы сначала проверь текст `validate_artifact` с `source`, затем сохраняй.
- Ошибка `…_DIAGRAM_LIMIT` — достигнут лимит тарифа: сообщи пользователю, не пытайся обойти.
- Ошибка `API_KEY_INVALID` — ключ неверный или истёк: попроси пользователя создать новый в профиле
  (https://helper-sa.ru/account/api-keys).
- Для проверки качества артефактов в веб-интерфейсе есть кнопка «ИИ-ревью» (тарифы «Макс» и выше).
