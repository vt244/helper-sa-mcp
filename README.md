# helper-sa-mcp

Подключение ИИ-агентов (Claude Code, Cursor, Codex, VS Code Copilot, Claude Desktop и других клиентов MCP)
к [«Помощнику SA»](https://helper-sa.ru) — платформе системного аналитика: проекты и артефакты DBML, PlantUML, OpenAPI,
AsyncAPI, BPMN, XSD, WSDL, XML, JSON Schema, YAML, JSON, Markdown, AsciiDoc.

Агент ищет, читает, создаёт и меняет артефакты ваших проектов, проверяет синтаксис, смотрит разницу версий и
откатывает изменения. Каждое изменение попадает в историю версий с источником «MCP»; правка поверх чужих
изменений не затирает их (`VERSION_CONFLICT`).

| Файл | Назначение |
|---|---|
| [`helper-sa-mcp-setup.md`](helper-sa-mcp-setup.md) | Инструкция для агента: он сам подключит MCP-сервер и установит навык |
| [`skills/helper-sa/SKILL.md`](skills/helper-sa/SKILL.md) | Навык (skill): порядок работы с проектами, правила изменений, проверки и отката |

## Что нужно

1. Аккаунт на https://helper-sa.ru с тарифом «Про» или выше.
2. Ключ API (начинается с `sah_`): профиль → «Ключи API» — https://helper-sa.ru/account/api-keys.
   Ключ даёт агенту ваши права; храните его только в личной конфигурации клиента или в переменной окружения,
   не коммитьте в репозитории.

Параметры сервера:

- адрес: `https://helper-sa.ru/api/mcp` (транспорт Streamable HTTP);
- заголовок: `Authorization: Bearer sah_…`.

## Способ 1. Доверить установку агенту

1. Скачайте [`helper-sa-mcp-setup.md`](https://raw.githubusercontent.com/vt244/helper-sa-mcp/main/helper-sa-mcp-setup.md)
   (или на странице https://helper-sa.ru/agents — кнопка «helper-sa-mcp-setup.md»).
2. Передайте файл агенту со словами «Выполни инструкцию из файла» и приложите ключ в сообщении.
3. Агент определит свой клиент, пропишет MCP-сервер, установит навык и проверит подключение (`whoami`,
   `list_projects`). Если клиент попросит перезапуск — перезапустите его.

## Обновление

Навык меняется вместе с платформой; его версия — в строке «Версия навыка» в начале `SKILL.md` (текущая —
`237cab87`). Чтобы обновить, передайте агенту свежий
[`helper-sa-mcp-setup.md`](https://raw.githubusercontent.com/vt244/helper-sa-mcp/main/helper-sa-mcp-setup.md) со словами «Обнови подключение по инструкции»: агент
проверит подключение (ключ повторно не нужен), заменит навык и сообщит версии. Вручную — замените файл навыка
содержимым [`skills/helper-sa/SKILL.md`](skills/helper-sa/SKILL.md).

## Способ 2. Установить вручную

### MCP-сервер

Подставьте свой ключ вместо `sah_ВАШ_КЛЮЧ`.

**Claude Code** — команда в терминале (сервер будет доступен во всех проектах):

```bash
claude mcp add --transport http --scope user helper-sa https://helper-sa.ru/api/mcp \
  --header "Authorization: Bearer sah_ВАШ_КЛЮЧ"
```

**Cursor** — файл `~/.cursor/mcp.json` (или `.cursor/mcp.json` в проекте):

```json
{
  "mcpServers": {
    "helper-sa": {
      "url": "https://helper-sa.ru/api/mcp",
      "headers": {
        "Authorization": "Bearer sah_ВАШ_КЛЮЧ"
      }
    }
  }
}
```

**VS Code (GitHub Copilot)** — пользовательский `mcp.json` (команда «MCP: Open User Configuration»):

```json
{
  "servers": {
    "helper-sa": {
      "type": "http",
      "url": "https://helper-sa.ru/api/mcp",
      "headers": {
        "Authorization": "Bearer sah_ВАШ_КЛЮЧ"
      }
    }
  }
}
```

**Codex CLI** — файл `~/.codex/config.toml`, ключ — в переменной окружения:

```toml
# ~/.codex/config.toml
[mcp_servers.helper-sa]
url = "https://helper-sa.ru/api/mcp"
bearer_token_env_var = "HELPER_SA_API_KEY"

# и в окружении: export HELPER_SA_API_KEY=sah_ВАШ_КЛЮЧ
```

**Claude Desktop и другие клиенты без HTTP** — через мост `mcp-remote` (нужен Node.js):

```json
{
  "mcpServers": {
    "helper-sa": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://helper-sa.ru/api/mcp",
        "--header",
        "Authorization:${HELPER_SA_AUTH}"
      ],
      "env": {
        "HELPER_SA_AUTH": "Bearer sah_ВАШ_КЛЮЧ"
      }
    }
  }
}
```

### Навык

**Claude Code** — навык пользователя:

```bash
mkdir -p ~/.claude/skills/helper-sa
curl -fsSL https://raw.githubusercontent.com/vt244/helper-sa-mcp/main/skills/helper-sa/SKILL.md -o ~/.claude/skills/helper-sa/SKILL.md
```

(или скопируйте `skills/helper-sa` из клонированного репозитория в `~/.claude/skills/`).

**Cursor** — правило `.cursor/rules/helper-sa.mdc` с содержимым `SKILL.md` (во frontmatter добавьте
`alwaysApply: false`) или пользовательские правила (Settings → Rules).

**Codex** — добавьте содержимое `SKILL.md` отдельным разделом в `~/.codex/AGENTS.md`.

**Другие клиенты** — в их файл пользовательских инструкций.

### Проверка

Попросите агента: «Вызови whoami и list_projects helper-sa». Ответ `API_KEY_INVALID` — ключ неверный или
истёк; `API_TARIFF_REQUIRED` — нужен тариф «Про» или выше.

## Что умеет агент

MCP-сервер предоставляет 28 инструментов.

**Аккаунт**

| Инструмент | Назначение |
|---|---|
| `whoami` | Под каким аккаунтом и тарифом подключён агент и до какой даты действует тариф |
| `list_tools` | Коды инструментов: dbml, openapi, xsd… |

**Проекты**

| Инструмент | Назначение |
|---|---|
| `list_projects` | Все проекты пользователя |
| `search_projects` | Поиск проекта по названию, роли, дате |
| `get_project` | Один проект |
| `create_project` | Создать проект |
| `update_project` | Переименовать, изменить описание |
| `delete_project` | Удалить проект (только по явной просьбе) |

**Папки и структура**

| Инструмент | Назначение |
|---|---|
| `get_project_tree` | Дерево папок и файлов с путями |
| `search_folders` | Поиск папок |
| `get_folder` | Папка с содержимым |
| `create_folder` | Создать папку |
| `update_folder` | Переименовать или переместить папку |
| `delete_folder` | Удалить папку |
| `place_artifact` | Положить файл в папку |

**Артефакты**

| Инструмент | Назначение |
|---|---|
| `list_artifacts` | Артефакты проекта или личные |
| `search_artifacts` | Поиск по названию, пути и тексту |
| `get_artifact` | Прочитать артефакт с номером версии |
| `create_artifact` | Создать артефакт в проекте и папке |
| `update_artifact` | Изменить целиком — с защитой от затирания чужих правок |
| `edit_artifact` | Точечная правка без передачи файла целиком |
| `validate_artifact` | Проверить синтаксис — как в редакторе |
| `move_artifact` | Перенести в проект или в личное |
| `delete_artifact` | Удалить артефакт |

**Версии**

| Инструмент | Назначение |
|---|---|
| `list_versions` | История версий с авторами |
| `get_version` | Прочитать версию |
| `diff_versions` | Что изменилось между версиями |
| `restore_version` | Откатить к версии |

Те же операции доступны через REST API `https://helper-sa.ru/api/ext/v1` — документация: https://helper-sa.ru/api-docs.

## Другой сервер

Файлы в репозитории настроены на https://helper-sa.ru. Для другого стенда «Помощника SA» скачайте инструкцию и навык на его
странице `/agents` — в них будет его адрес.

## Об этом репозитории

Файлы выгружаются из исходного кода платформы (страница https://helper-sa.ru/agents) и совпадают с тем, что она
скачивает; вопросы и предложения — через техподдержку https://helper-sa.ru.
