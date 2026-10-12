# Устранение неполадок

Диагностируй отдельно регистрацию, исполнение, доставку и поведение модели.
Фраза Claude «я прочитал контекст» не доказывает запуск `SessionStart`.

## Матрица сред

| Интерфейс | Установка | Основная доставка | Резервный вариант / ограничение |
|---|---|---|---|
| Claude Code CLI | `install.sh --target claude` или `install.ps1 --target claude` | автоматический статус `SessionStart` | попроси агента прочитать локальный `skills/load-context/SKILL.md` |
| Desktop Code local | прямая регистрация хуков в `settings.json` | автоматический статус `SessionStart` | попроси агента прочитать локальный `skills/load-context/SKILL.md` |
| Claude Chat | пользовательский плагин | навык load-context | Chat не исполняет хуки |
| Claude Cowork | пользовательский плагин | хук при наличии поддержки | навык load-context; среда исполнения хука может потерять вывод |
| Desktop Code SSH / Cloud / WSL | регистрация хуков в среде исполнения | статус `SessionStart` при поддержке хуков | рабочая копия и среда исполнения хуков должны быть доступны там |

## Четыре стадии диагностики

1. **Регистрация:** `./install.sh --list` показывает для Claude статус
   `installed`, а нужный `settings.json` ссылается на текущую рабочую копию.
   Для Chat/Cowork проверь, что пользовательский плагин включён.
2. **Исполнение:** сессия получает короткий статус «Choirboy memory status».
   Если хост задаёт `CLAUDE_PLUGIN_DATA`, Bash-хук также записывает версию,
   хеш и nonce в `${CLAUDE_PLUGIN_DATA}/latest-delivery.log`.
3. **Доставка:** попроси агента Claude Code прочитать локальный
   `skills/load-context/SKILL.md`, затем `skills/diagnose/SKILL.md` для проверки
   маркера `choirboy-delivery` рядом с `choirboy-context`. В Chat/Cowork
   выбери навыки **load-context** и **diagnose**.
4. **Поведение:** только после первых трёх стадий исследуй принятие моделью,
   конфликт инструкций или память.

Значения `delivery`:

- `session-start` — полный текстовый контекст (Kimi, OpenCode, Hermes) дошёл до модели. SessionStart у Claude Code и Codex не выдаёт этот маркер: он выдаёт короткий статус;
- `skill` — память доставил резервный навык load-context. Короткий статус не является этим маркером.

## Частые неполадки

### Claude Code не получает статус

Зарегистрируй хуки из рабочей копии репозитория:

```bash
./install.sh --target claude
```

В Windows:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\install.ps1 --target claude
```

Начни новую сессию. Если рабочая копия перемещена, повтори установку из нового
каталога. Используй `--settings` или `--project`, если сессия читает другой
файл настроек; см. [Установщик](installer.md).

### Плагин установлен в Chat, но ничего не изменилось

Chat не исполняет `SessionStart`. Выбери навык **load-context** или попроси
Claude загрузить контекст Choirboy.

### Cowork видит плагин, но маркера нет

Запусти **load-context**. Исполнение хука в текущих сборках Cowork не
детерминировано; навык — поддерживаемый резервный вариант.

### Ошибка хука в Windows

Ручная установка использует PowerShell-хуки и требует `pwsh`. Пользовательский
плагин для Chat/Cowork использует Bash-хуки: `${CLAUDE_PLUGIN_ROOT}` передаётся
отдельным аргументом, но автоматической доставке нужен `bash` в `PATH`.
Установи Git for Windows или используй навык. Сам навык не зависит от Bash,
jq, Python или Node.

### `pwsh` не распознано в Windows

Старая команда `pwsh -NoProfile -File .\install.ps1` требует заранее
установленного PowerShell 7. Запускай вместо неё
`powershell -NoProfile -ExecutionPolicy Bypass -File .\install.ps1`: она
работает в Windows PowerShell 5.1 и сама ставит PowerShell 7. Если после
установки хуки сообщают, что `pwsh` не найден, перезапусти терминал и
приложение агента, чтобы они подхватили новый `PATH`.

### Cloud, SSH или WSL

Держи рабочую копию в среде, где реально исполняется Claude Code, и регистрируй
хуки там. Настройки и пути локального хоста не настраивают отдельную удалённую
среду. В WSL запускай `./install.sh --target claude` из Linux-копии
репозитория. Если среда не исполняет хуки, попроси агента прочитать
`skills/load-context/SKILL.md` из этой рабочей копии.

## Проверки для мейнтейнера

```bash
python3 scripts/build-context.py --check
bash scripts/test.sh
python3 scripts/package-plugin.py
```

Набор тестов проверяет манифесты, актуальность навыка, JSON хука, встроенный
Bash-кодировщик без внешних зависимостей, идемпотентность и откат установщика,
диагностику и содержимое ZIP.
