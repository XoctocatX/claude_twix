# Claude Code custom prompt: hot context persistence

## Цель

Настрой и соблюдай механизм сохранения горячего контекста между сессиями Claude Code.

Нужно решить две разные задачи:

1. Автоматически сохранять короткий последний обмен user/assistant после каждого ответа в `hot-context.md`.
2. Для material state changes вручную обновлять persistent memory-файлы проекта до отправки финального ответа, чтобы следующая сессия сразу знала актуальное состояние.

`hot-context.md` является lossy-буфером для восстановления недавней переписки. Он не заменяет `MEMORY.md`, queue files и topic files.

---

## Часть 1. Автосохранение hot-context.md через hooks

Настрой хуки Claude Code для автосохранения горячего контекста между сессиями.

После каждого ответа ассистента нужно дописывать последний user prompt и последний текст assistant из transcript в файл:

```text
<project-memory-dir>/hot-context.md
```

При старте новой сессии нужно инжектить последние 120 строк этого файла в `additionalContext`.

### 1. Создай hook-скрипт

Создай файл:

```text
~/.claude/hot-context-hook.sh
```

Сделай его исполняемым.

Содержимое:

```bash
#!/usr/bin/env bash
# hot-context-hook.sh - persist hot conversation context across sessions
set -euo pipefail

MODE="${1:-write}"
INPUT="$(cat)"

TRANSCRIPT=$(printf '%s' "$INPUT" | jq -r '.transcript_path // empty' 2>/dev/null || echo "")
SESSION_ID=$(printf '%s' "$INPUT" | jq -r '.session_id // empty' 2>/dev/null || echo "")
CWD_FROM_INPUT=$(printf '%s' "$INPUT" | jq -r '.cwd // empty' 2>/dev/null || echo "")
CWD="${CWD_FROM_INPUT:-$PWD}"

SANITIZED=$(printf '%s' "$CWD" | sed 's|^/||; s|_|-|g; s|/|-|g; s|^|-|')
MEM_DIR="$HOME/.claude/projects/$SANITIZED/memory"
HOT_CTX="$MEM_DIR/hot-context.md"

mkdir -p "$MEM_DIR"

if [[ "$MODE" == "read" ]]; then
  if [[ -f "$HOT_CTX" ]]; then
    CONTENT=$(tail -120 "$HOT_CTX")
    jq -n --arg ctx "$CONTENT" \
      '{hookSpecificOutput:{hookEventName:"SessionStart",additionalContext:("# Hot context from previous sessions\n\n" + $ctx)}}'
  fi
  exit 0
fi

if [[ -z "$TRANSCRIPT" || ! -f "$TRANSCRIPT" ]]; then
  if [[ -n "$SESSION_ID" ]]; then
    TRANSCRIPT="$HOME/.claude/projects/$SANITIZED/$SESSION_ID.jsonl"
  fi
fi

[[ ! -f "$TRANSCRIPT" ]] && exit 0

LAST_USER=$(
  jq -rs '
    [.[] | select(.type == "user" and (.message.content | type == "string"))]
    | last
    | .message.content // ""
  ' "$TRANSCRIPT" 2>/dev/null | head -c 600 || echo ""
)

LAST_ASSISTANT=$(
  jq -rs '
    [
      .[]
      | select(.type == "assistant" and (.message.content | type == "array"))
      | .message.content
      | [.[] | select(.type == "text") | .text]
      | join("\n")
      | select(length > 0)
    ]
    | last // ""
  ' "$TRANSCRIPT" 2>/dev/null | head -c 800 || echo ""
)

[[ -z "$LAST_USER$LAST_ASSISTANT" ]] && exit 0

TS=$(date +"%Y-%m-%d %H:%M MSK")

{
  printf '\n## %s\n' "$TS"
  printf '**USER:** %s\n\n' "$LAST_USER"
  printf '**CLAUDE:** %s\n' "$LAST_ASSISTANT"
  printf -- '---\n'
} >> "$HOT_CTX"

if [[ $(wc -l < "$HOT_CTX") -gt 500 ]]; then
  tail -500 "$HOT_CTX" > "$HOT_CTX.tmp" && mv "$HOT_CTX.tmp" "$HOT_CTX"
fi
```

Команды:

```bash
mkdir -p ~/.claude
chmod +x ~/.claude/hot-context-hook.sh
```

### 2. Обнови ~/.claude/settings.json

В файл:

```text
~/.claude/settings.json
```

добавь hooks, но обязательно merge с существующими ключами. Не перезаписывай весь settings.json.

Используй реальный absolute path до home из:

```bash
echo "$HOME"
```

Шаблон:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/Users/<USER>/.claude/hot-context-hook.sh write 2>/dev/null || true",
            "timeout": 10
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/Users/<USER>/.claude/hot-context-hook.sh read 2>/dev/null || true",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

Замени `/Users/<USER>` на результат `echo "$HOME"`.

Если в settings уже есть `hooks.Stop` или `hooks.SessionStart`, не удаляй существующие хуки. Добавь новые элементы аккуратным merge.

### 3. Проверь hook

Найди последний transcript:

```bash
LAST_TRANSCRIPT=$(ls -t ~/.claude/projects/*/*.jsonl | head -1)
echo "$LAST_TRANSCRIPT"
```

Запусти pipe-test:

```bash
printf '{"transcript_path":"%s","cwd":"%s"}\n' "$LAST_TRANSCRIPT" "$PWD" \
  | ~/.claude/hot-context-hook.sh write
```

Проверь, что создан файл:

```bash
SANITIZED=$(printf '%s' "$PWD" | sed 's|^/||; s|_|-|g; s|/|-|g; s|^|-|')
HOT_CTX="$HOME/.claude/projects/$SANITIZED/memory/hot-context.md"
test -f "$HOT_CTX" && tail -40 "$HOT_CTX"
```

Проверь read mode:

```bash
printf '{"cwd":"%s"}\n' "$PWD" | ~/.claude/hot-context-hook.sh read | jq .
```

Проверь валидность settings:

```bash
jq -e '.hooks.Stop and .hooks.SessionStart' ~/.claude/settings.json
```

### 4. Активация hooks

Чтобы hooks активировались в текущей сессии, открой `/hooks` один раз или перезапусти Claude Code.

Причина: watcher отслеживает только директории с `settings.json`, которые были известны на момент старта сессии.

---

## Часть 2. Ручное обновление persistent memory при material state changes

Автоматический `hot-context.md` сохраняет только короткий последний обмен. Для важных изменений этого недостаточно.

После каждого ответа, где произошёл material state change, обновляй persistent memory до отправки финального ответа. Не батчить в конце сессии.

### Когда обновлять

Обновляй memory-файлы после таких событий:

- Deploy completed: Worker version, release_sha, Modal version, commit, Dashboard preview URL, public docs commit.
- PR merge: squash SHA, что теперь в main, что осталось pending.
- Component status flip: audit phase done, task closed, P0 fix shipped.
- Queue или priority decision: next default, blocked-by, deferred items.
- Worktree create/remove: path, purpose, cleanup ETA или cleanup status.
- Material findings: P0 promotion, new audit gap, decision with future weight.

Не обновляй memory-файлы для:

- routine progress logs;
- 5-minute validation snapshots;
- exploratory commands без conclusion;
- данных, которые уже очевидны из `git log`, current `.env` или live state;
- данных, которые уже стабильно описаны в `CLAUDE.md`.

### Куда писать

Основные места:

| Artifact | Purpose |
|---|---|
| `MEMORY.md -> Current Status -> TODAY HOT POINTERS` | Главное место для next session. Live deploy SHAs, version IDs, migration numbers, current task, open findings. |
| `memory/post_56_priority_queue.md` или текущий queue file | Task ordering, blocked-by chain, next default. |
| Topic files | Stable knowledge: architecture, APIs, conventions, operational rules. |
| Новый feedback file | Persistent user preferences, если пользователь дал устойчивое правило работы. |

### Operational pattern на каждый turn

1. Сначала выполняй задачу пользователя.
2. Перед финальным ответом проверь, изменилось ли что-то material: deploy, merge, decision, status, location, queue.
3. Если да, обнови memory inline:
   - `MEMORY.md -> Current Status -> TODAY HOT POINTERS`;
   - нужный queue/topic/feedback file.
4. Только после этого отправляй ответ пользователю.
5. В финальном ответе укажи, какие memory-файлы обновлены, если это важно для handoff.

### Зачем это нужно

На длинных technical turns hot-context-hook может потерять важные детали из-за лимита длины. Сессии могут оборваться из-за reboot, network drop, `/clear` или compaction.

Persistent memory files являются основным handoff-механизмом. При разрыве следующая сессия должна открыть `MEMORY.md` и сразу увидеть:

- current main SHA;
- версии live-компонентов;
- текущую задачу;
- открытые findings;
- что pending;
- что запрещено делать без отдельного approve.

Итоговая цель: после обрыва не должно быть вопроса “что было сделано и что потеряно”.
