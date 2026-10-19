# Отчёт: запуск test-skills demo-kit на проекте <target_system>

**Дата:** 2026-10-15
**Задача:** <TASK-ID>
**Проект:** `<project_root>/<target_system>`
**Комплект:** `.gigacode/skills/test-skills-demo-kit-20261015/` (пакет `test-skills-bbee84b`, `0.5.0-pilot`)
**Вход (аналитика):** `docs/to_do/<RESEARCH-TASK-ID>_AI_2_Test-case/Мажор 11.md` → нормализован в `Мажор 11.requirements.md`
**Модель:** `DeepSeek-V4.1-Flash`
**Профиль:** `cases-only-v1` (пробный прогон, шаг 5a)

---

## 1. Что запускали и как

Комплект развёрнут в домашней директории по раскладке, которую ожидает `RUN.md`:

```
~/ts/
├── preflight.py          # шаг 1
├── operate.py            # оркестратор шагов 0,1,2,3,4,5a,5,7,8
├── RUN.md
├── wrap/                 # wrap.py, wrap.json, selftest.py, fake_cli.py, examples/
├── bench/                # bench.py, fixtures, stub-wrap.json
├── test-skills/          # сам пакет: tools/, schemas/, skills/, .venv/
└── runner/               # каталог процессов раннера (вне проекта — требование драйвера)
```

Оркестратор запускается **интерпретатором из venv пакета**, а не системным `python3`:

```bash
~/ts/test-skills/.venv/bin/python ~/ts/kit/operate.py \
  --home ~/ts --project <проект> \
  --docs "docs/to_do/<RESEARCH-TASK-ID>_AI_2_Test-case/Мажор 11.requirements.md" \
  --subject "<business_subject>" \
  --models DeepSeek-V4.1-Flash \
  --cli-command 'gigacode --bare --model {model} --approval-mode plan --max-wall-time 1500' \
  --prompt-via stdin --answer-from stdout --cli-version 'gigacode 26.8.53' \
  --module root--6d61696e --runner-dir ~/ts/runner \
  --review-input-bytes 22000 --max-tasks 2 --max-parallel 1 \
  --id-pattern '^\d{1,2}\.\d{1,2}(?:\.\d{1,2})?\s' \
  --no-mutation --from-step 1 --only 1,2,5a --yes
```

---

## 2. Найденные дефекты и их причины

### 2.1. `preflight.py` не находился (исправлено раскладкой)

`operate.py` вызывает `<home>/preflight.py`. В исходной раскладке всё лежало под `~/ts/kit/`, поэтому шаг 1 падал с
`can't open file '~/ts/preflight.py'`. Лечится раскладкой из п. 1.

### 2.2. `jsonschema` отсутствует у системного `python3` (исправлено интерпретатором)

`operate.py` берёт `PY = sys.executable`. При запуске системным `python3` (3.13) шаг 1 падал на
`ModuleNotFoundError: No module named 'jsonschema'`, а `doctor` при этом показывал вводящее в заблуждение
`NOT_RUNNABLE, отсутствуют: ['invalid:pipeline_contract', 'invalid:release/manifest.json']` — это был симптом той же
проблемы, а не порча пакета. Лечится запуском через `~/ts/test-skills/.venv/bin/python`.

### 2.3. Драйвер требует явный `--module` (исправлено патчем `operate.py`)

`tools/skillsrc_manifest.py: select_module` при нескольких модулях в `.skillsrc` требует явный ID:

```python
if module_id is None:
    raise SkillsrcError("module_required", "module selection is required")
```

У проекта 9 модулей, а `operate.py` флаг не передавал → шаг 5a останавливался с
`INVENTORY_INVALID` / `module_required`. Также `step_5a` падал трейсбеком `KeyError: 'run_id'`, потому что прогон,
умерший на скане, `run_id` не создаёт. Обе правки внесены в `kit/operate.py` (`--module` в `driver_argv` + guard на
отсутствие `run_id`).

### 2.4. Запуск скрипта по shebang в этом окружении запрещён — ключевой дефект

Симптом в `result.json` раннера:

```json
{"error": "PermissionError: [Errno 1] Operation not permitted: '<home>/ts/wrap/wrap.py'"}
```

Причина не в правах на файл и не в `noexec`: в этом окружении **`execve` по shebang запрещён для любого скрипта**.
Проверено на трёх однотипных файлах:

| Shebang | Результат |
|---|---|
| `#!/usr/bin/env python3` | `env: неверный интерпретатор: Операция не позволена`, код 126 |
| `#!/usr/bin/python3.13` | `неверный интерпретатор: Операция не позволена`, код 126 |
| `#!/bin/sh` | `неверный интерпретатор: Операция не позволена`, код 126 |

При этом прямой вызов интерпретатора работает: `/usr/bin/env python3 -c 'print("env ok")'` → `env ok`.

`model_runner.shim` запускает команду так:

```python
process = subprocess.Popen(spec["argv"], **kwargs)
```

то есть первым элементом argv идёт сам `wrap.py` — и получает EPERM. Драйвер это видит как транспортный отказ и делает
3 попытки (`TRIES`), все три падают одинаково, после чего задача считается проваленной.

**Решение:** передать драйверу шаблон команды, где обёртка вызывается интерпретатором явным аргументом. В
`kit/operate.py` добавлен метод `runner_command()` и флаг `--review-runner-command` в `driver_argv`:

```python
def runner_command(self) -> str:
    if WINDOWS:
        return json.dumps([str(self.runner_cli()), "exec", ...])
    return json.dumps([PY, str(self.runner_cli()), "exec", "--skip-git-repo-check",
                       "--sandbox", "read-only", "--ephemeral", "--json",
                       "--model", "{model}", "--output-last-message", "{last_message}", "-"], ensure_ascii=False)
```

Драйвер подставляет `{cli}` из preset'а `codex` первым элементом, поэтому `--model` / `--output-last-message` / `-`
остаются на своих местах. Проверено: обёртка запускается, модель отвечает, `last-message.txt` пишется:

```
exit: 0
{"type": "item.completed", "item": {"type": "agent_message", "text": "ок\n"}}
{"type": "turn.completed", "usage": {"input_tokens": 13, "output_tokens": 1, "estimated": true}}
```

**Побочный эффект правки:** после неё прогон впервые дошёл до реального вызова модели — и упёрся в лимит контекста
(см. 2.5). До этого wall-стена маскировалась транспортной ошибкой.

### 2.5. Контекст задачи не влезает в окно модели (не исправлено — ограничение профиля)

После починки 2.4 драйвер вернул содержательный отказ:

```
wrap: the CLI exited 1: [API Error: Context is too large to send safely after automatic compression.
Estimated prompt tokens: 317636; hard limit: 239144; compression status: NOOP.
Start a new session or reduce the resumed history before continuing.]
```

Сходится с измерением: `1 352 513 Б` stdin × `0.2391 токенов/Б` ≈ `323 400` токенов. Лимит — `239 144`.

---

## 3. Арифметика контекста

Замеры по `stdin.txt` задачи `context-marker:baseline` и по инвентарю прогона:

| Величина | Значение |
|---|---|
| `input_bytes` задачи | 1 352 513 Б |
| Оценка в токенах (0.2391 токенов/Б) | ≈ 323 400 |
| Лимит модели | 239 144 токенов |
| Бюджет в байтах | ≈ 1 000 110 Б |
| Превышение | 1.32× |

Состав входа:

| Часть | Байт | Доля |
|---|---|---|
| код и ресурсы проекта | 1 262 533 | 95.6 % |
| черновик требований | 57 725 | 4.4 % |

**Вывод: резать аналитику бессмысленно.** Требования занимают 4.4 % входа; даже обнулив их, задача остаётся за
лимитом. Проблема — объём кодовой базы модуля `main`.

### Почему узкий `paths` в `.skillsrc` не помог

Ограничение `paths.source: [src/main/java]` и `tests: []` / `resources: []` **влияет только на батчи** — видно по
receipt'ам: батч 1 с `byte_budget=262144` содержит 126 файлов ровно из `src/main`, а два крупных мока вынесены в
`gaps` с `FILE_EXCEEDS_BYTE_BUDGET`. Но задача `context-marker:baseline` собирается **без** ограничения:

```python
inputs = [draft_path, *_context_inputs(project, run_root, attempt)]        # без only → все receipt'ы слиты
inputs = [brief_path, marker_path, *_context_inputs(project, run_root, attempt, only=context)]  # only=context
```

`only=context` применяется только в `_generator_task`. То есть на шаге маркировки контекста модель получает полный
инвентарь проекта (377 файлов / 2 999 609 Б), а не урезанный набор. Отсюда и рост `input_bytes` вместо снижения:
1 320 258 → 1 352 513 Б (разница — правка `.skillsrc`, которая сама попала во вход).

### Почему разбиение аналитики не помогает

Проверено трижды: вход задачи определяется проектом, а не размером документа.

| Документ | Размер | Токены задачи |
|---|---|---|
| `Мажор 11.md` целиком | 312 КБ | 359 142 |
| `part1` + `part2` | 126 КБ | 327 538 |
| `Мажор 11.requirements.md` | 54 КБ | 315 697 |

Документ сокращён в 5.8 раза, а вход упал на 12 %.

---

## 4. Про `requirements.id_pattern`

`.skillsrc` получил секцию, помечающую строку с номером пункта как границу секции:

```yaml
requirements:
  id_pattern: '^\d{1,2}\.\d{1,2}(?:\.\d{1,2})?\s'
```

Эффект на разбиение `Мажор 11.requirements.md`:

| Шаблон | Секций | С явным ID |
|---|---|---|
| встроенный `^Требование \d+` | 4 | 0 |
| заданный в проекте | 19 | 15 |

То есть шаблон работает (`3.1`, `3.2`, `5.1`, `5.2`, `6.1`, `6.2`, `4.2.3`, `6.3`, `6.3.1`–`6.3.4`, `6.4`, `6.4.1`,
`6.4.2`), но на объём входа не влияет — по причине из п. 3.

---

## 5. Что мешает полному прогону

1. **Лимит контекста (блокирующее).** `context-marker` не проходит на модуле `main`: 1.32× превышение. Нужно либо
   сузить инвентарь до реально нужного подмножества кода, либо разбить маркировку контекста на части, либо взять
   проект/модуль меньшего размера.
2. **Мутации выключены.** В `~/.m2` нет `maven-dependency-plugin 3.8.1`, `pitest-command-line 1.30.0`,
   `pitest-junit5-plugin 1.2.3` — при `--mutation` будет `NOT_RUNNABLE`. Прогон шёл с `--no-mutation`.
2а. **Обёртка Maven указана неверно.** Preflight предупредил: `mvnw нет, есть mvn в PATH`. В `.skillsrc` у модуля
   `main` сейчас стоит `adapter_id: maven-wrapper:selected-symbols-v1` и `wrapper: mvnw`. На пробный прогон это не
   влияет (профиль `cases-only-v1` до Maven не доходит), но для полного `local-pilot-v1` нужно переключить на
   `maven:selected-symbols-v1` (системный Maven).
3. **Предупреждение о ПДн.** Preflight нашёл секреты в 11 файлах, email в 37, телефоны в 17, СНИЛС/ИНН в 17. Пакет
   передаёт выбранные файлы модели — нужно проверить, что значения тестовые, иначе исключать через `limits` /
   `exclusions` в `.skillsrc`.

---

## 6. Итог

**Механика запуска разобрана полностью.** Пробный прогон `cases-only-v1` доходит до вызова модели — то есть все
транспортные и конфигурационные дефекты устранены. Дальше упирается в окно контекста модели, и это ограничение не
обходится подгонкой аналитики: вход на 95.6 % состоит из кода проекта.

**Что сделано:**

- раскладка `~/ts/` приведена к ожидаемой `RUN.md`, `doctor` → PASS;
- определён правильный интерпретатор (`~/ts/test-skills/.venv/bin/python`) — иначе нет `jsonschema`;
- в `kit/operate.py` добавлена передача `--module` и guard на отсутствие `run_id`;
- в `kit/operate.py` добавлен `runner_command()` и `--review-runner-command` — обход запрета `execve` по shebang;
- в `.skillsrc` добавлен `requirements.id_pattern`, сужены `paths` модуля `main`;
- пробный прогон 5a выполнен, получен содержательный отказ по лимиту контекста.

**Не сделано:** полный прогон `local-pilot-v1` (шаг 5) — блокирован лимитом контекста; прогон на остальных моделях —
по той же причине.

**Файлы правок:** `kit/operate.py` (3 правки), `.skillsrc` (2 правки), бэкап `.skillsrc.bak-20261015`.
