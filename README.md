# Глобальное подключение скиллов к opencode

В репозитории скиллы лежат в директории `skills/`:

- `skills/coding-style/SKILL.md`
- `skills/tests/SKILL.md`
- `skills/ddd/SKILL.md`

Чтобы эти скиллы были доступны из любой папки, я подключил их через глобальную конфигурацию opencode.

Глобальный файл конфигурации находится здесь:

```text
~/.config/opencode/opencode.jsonc
```

В него добавлен абсолютный путь к директории `skills/` из этого репозитория:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/home/execut/Projects/opencode/skills"]
  }
}
```

Поле `skills.paths` указывает opencode сканировать директорию со скиллами. Для глобального подключения нужен абсолютный путь, потому что opencode может запускаться из любой рабочей директории.

Внутри указанной директории opencode рекурсивно ищет файлы `SKILL.md`, поэтому каждый скилл должен находиться в отдельной папке и иметь такую структуру:

```text
skills/<skill-name>/SKILL.md
```

Файл `SKILL.md` должен содержать frontmatter с `name` и `description`, например:

```markdown
---
name: example-skill
description: "Use when example instructions are relevant"
---

# Example Skill
```

Если репозиторий будет перемещён, нужно обновить путь в `~/.config/opencode/opencode.jsonc`.

После изменения глобальной конфигурации или файлов скиллов нужно перезапустить opencode, потому что конфигурация загружается только при старте.