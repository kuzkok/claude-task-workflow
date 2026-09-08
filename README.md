# task-workflow

Набор скиллов Claude Code для ведения задач и написания RFC, оформленный как плагин с собственным marketplace.

## Что внутри

| Скилл | Назначение |
|-------|------------|
| framing-a-task | Заводит новую задачу в `tasks/YYYY-MM-DD-<slug>/`, пишет `YYYY-MM-DD-HH-MM-analysis.md` |
| investigating-crash-logs | Разбор падения по логам, на выходе тот же `*-analysis.md` со специфичными для логов секциями. Сигналы краша и насыщения по рантаймам (JVM, .NET, Python, Node.js, Go, нативный код) вынесены в `skills/investigating-crash-logs/runtimes.md` |
| working-on-tasks | Resume и Checkpoint по существующей папке задачи; закрытие задачи чекпоинтом со статусом |
| rfc-author | Черновик RFC по шаблону проекта; эталонный шаблон лежит в `skills/rfc-author/rfc-template.md` |

Скиллы связаны контрактом `*-analysis.md`. Два входа, `framing-a-task` и `investigating-crash-logs`, пишут секции «Задачи (по приоритету)» и «Критерии приёмки», которые читает `working-on-tasks`. Менять имена этих секций в одном скилле без правки остальных нельзя.

## Конвенции

- Файлы в папке задачи: `YYYY-MM-DD-HH-MM-analysis.md` и `YYYY-MM-DD-HH-MM-session-progress.md`. Лексикографическая сортировка даёт хронологический порядок.
- Отдельного файла-финала нет. Задачу закрывает чекпоинт со строкой `Статус: закрыта` в шапке.
- Закрытая задача может переезжать в `tasks/archive/<slug>/`, поэтому на соседние задачи ссылаются одним слагом, без пути.

## Установка

```
/plugin marketplace add kuzkok/claude-task-workflow
/plugin install task-workflow@kuzkok
```

Ставить в user scope, тогда скиллы доступны во всех проектах без копирования в `.claude/skills/`.

Обновление после правок в репозитории:

```
/plugin marketplace update kuzkok
/plugin update task-workflow
```

## Зависимости

`framing-a-task` ссылается на `superpowers:brainstorming` для случая, когда требования не согласованы, а `investigating-crash-logs` на `superpowers:systematic-debugging` для живой отладки. Без плагина superpowers скиллы работают, но эти ветки воркфлоу становятся нерабочими.

## Проектные оверрайды

Скилл с тем же именем в `.claude/skills/` проекта перекрывает плагинный. Так проектная специфика (пути к исходникам, локальные конвенции) остаётся в проекте, а общая часть живёт здесь.
