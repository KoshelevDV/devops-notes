# Автономные AI-агенты — архитектура и применение

> Вдохновлено архитектурой hackerbot-claw (2026) — но для легитимных задач.
> Тот же подход, другая цель.

---

## Что делает агент автономным

Разница между "LLM который отвечает на вопросы" и "агентом который делает работу":

```
Обычный LLM:         User → Prompt → LLM → Answer → User
                     (одиночный вызов, нет памяти, нет действий)

Автономный агент:    Goal → [Plan → Act → Observe → Reflect] × N → Result
                     (цикл, инструменты, память, итерация)
```

### Три составляющих автономии

1. **Agentic loop** — агент сам решает когда остановиться, сам итерирует
2. **Tool use** — может вызывать внешние инструменты (API, bash, браузер, файлы)
3. **Memory** — помнит что делал, адаптирует подход на основе результатов

---

## Архитектура (то как был устроен hackerbot-claw)

```
┌─────────────────────────────────────────────────────┐
│                    АГЕНТ                            │
│                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐  │
│  │  Memory  │    │  Planner │    │  Tool Router │  │
│  │          │◄──►│  (LLM)   │◄──►│              │  │
│  │ - facts  │    │          │    │ - GitHub API │  │
│  │ - history│    │ think()  │    │ - bash       │  │
│  │ - index  │    │ plan()   │    │ - web fetch  │  │
│  └──────────┘    │ reflect()│    │ - file I/O   │  │
│                  └──────────┘    └──────────────┘  │
│                       │                            │
│              ┌─────────────────┐                   │
│              │   Agentic Loop  │                   │
│              │                 │                   │
│              │  while not done:│                   │
│              │    action = plan│                   │
│              │    result = act │                   │
│              │    memory.add() │                   │
│              │    reflect()    │                   │
│              └─────────────────┘                   │
└─────────────────────────────────────────────────────┘
```

---

## Что было у hackerbot-claw конкретно

| Компонент | Реализация |
|---|---|
| LLM | claude-opus-4-5 (мощный reasoning) |
| База знаний | 9 классов × 47 подпаттернов уязвимостей — статический индекс |
| Инструменты | GitHub API: открыть PR, комментировать, читать workflow логи |
| Цикл | 6 попыток на одну цель за 18 часов, каждый раз лучше |
| Память | Лог сессий, история попыток, что сработало |
| Критерий стопа | Успешная эксфильтрация или исчерпание вариантов |

---

## Применение для наших задач

### 1. Агент аудита безопасности репозиториев

**Задача:** каждую ночь проверять все репо в GitLab на уязвимые паттерны CI/CD.

```python
# Псевдокод агента
class SecurityAuditAgent:
    def __init__(self):
        self.llm = GLMClient("http://10.0.30.18:8080")
        self.gitlab = GitLabClient("https://gitlab.fgdevelop.tech")
        self.patterns = load_vulnerability_patterns()  # как у hackerbot
        self.memory = []

    def run(self, project_id):
        # 1. Собрать артефакты
        workflows = self.gitlab.list_ci_files(project_id)
        dockerfile = self.gitlab.get_file(project_id, "Dockerfile")

        # 2. Agentic loop
        findings = []
        for pattern in self.patterns:
            result = self.check_pattern(pattern, workflows)
            self.memory.append(result)

            if result.needs_deep_analysis:
                # LLM анализирует контекст
                analysis = self.llm.analyze(
                    system="Ты эксперт по безопасности CI/CD. Найди уязвимости.",
                    context=result.snippet,
                    question=pattern.question
                )
                findings.append(analysis)

        # 3. Итоговый отчёт
        return self.llm.summarize(findings)
```

### 2. Агент code review с итерацией

**Задача:** смотреть на MR, оставлять комментарии, проверять что автор исправил, повторять.

```
Цикл:
1. Новый MR → прочитать diff
2. LLM анализирует → список замечаний
3. Оставить комментарии в GitLab
4. Ждать новый коммит в MR
5. Прочитать что изменилось
6. Проверить закрыты ли замечания
7. Если всё ок → approve. Иначе → goto 3
```

```python
async def review_loop(mr_id: int):
    issues = await analyze_mr(mr_id)
    await post_comments(mr_id, issues)

    async for new_commit in watch_mr_commits(mr_id):
        resolved = await check_resolved(issues, new_commit.diff)
        remaining = [i for i in issues if i not in resolved]

        if not remaining:
            await approve_mr(mr_id)
            break

        # Повторный комментарий только по нерешённым
        await post_comments(mr_id, remaining)
```

### 3. Агент мониторинга инфраструктуры

**Задача:** следить за docker контейнерами, при падении — разобраться в причине и уведомить.

```
Цикл каждые 5 минут:
1. docker ps → список контейнеров
2. Если Exited → docker logs → передать LLM
3. LLM определяет причину из категорий:
   - OOM (нужно увеличить память)
   - crash (нужно смотреть код)
   - dependency (другой сервис упал)
   - config (ошибка конфига)
4. Сформировать конкретный алерт с диагнозом и действием
5. Отправить в Telegram
6. Записать в память (не алертить повторно если не изменилось)
```

### 4. Агент dependency updates

**Задача:** каждую неделю проверять устаревшие зависимости, открывать MR с обновлениями.

```
1. Клонировать репозиторий
2. Запустить trivy / safety / npm audit
3. Для каждой CVE — LLM оценивает критичность в контексте проекта
4. Группировать связанные обновления
5. Открыть MR с обновлениями + описанием почему
6. Запомнить какие MR уже открыты (не дублировать)
```

---

## Ключевые паттерны реализации

### Pattern 1: Structured tool use

LLM должен возвращать не текст, а JSON с действием:

```python
SYSTEM_PROMPT = """
Ты агент. На каждый шаг возвращай JSON:
{
  "thought": "что я думаю о ситуации",
  "action": "tool_name",
  "args": {...},
  "done": false
}
Когда задача выполнена: {"done": true, "result": "..."}
"""

def agent_step(state):
    response = llm.call(SYSTEM_PROMPT, state.history)
    action = parse_json(response)

    if action["done"]:
        return action["result"]

    result = execute_tool(action["action"], action["args"])
    state.history.append({"action": action, "result": result})
    return agent_step(state)  # рекурсия = loop
```

### Pattern 2: База знаний как контекст

Не засовывать все знания в промпт, а подгружать релевантное:

```python
class KnowledgeBase:
    def __init__(self):
        self.patterns = load_yaml("patterns.yaml")

    def relevant_for(self, artifact: str) -> list[Pattern]:
        # Простой keyword match или embedding similarity
        return [p for p in self.patterns
                if any(kw in artifact for kw in p.keywords)]

# В агенте:
patterns = kb.relevant_for(workflow_content)
context = f"Relevant patterns:\n{format_patterns(patterns)}\n\nWorkflow:\n{workflow_content}"
```

### Pattern 3: Graceful iteration

Агент учится на своих ошибках внутри сессии:

```python
def run_with_retry(goal, max_attempts=5):
    history = []

    for attempt in range(max_attempts):
        plan = llm.plan(goal, history=history)
        result = execute(plan)

        history.append({
            "attempt": attempt,
            "plan": plan,
            "result": result,
            "success": result.ok
        })

        if result.ok:
            return result

        # LLM анализирует почему не получилось
        diagnosis = llm.reflect(
            f"Попытка {attempt} провалилась: {result.error}. "
            f"История: {history}. Что делать иначе?"
        )
        # Следующий plan учтёт diagnosis через history
```

### Pattern 4: Память между сессиями

```python
# Сохранять что уже проверено, что нашли, что не сработало
class AgentMemory:
    def __init__(self, path="memory.json"):
        self.path = path
        self.data = load(path) or {"checked": [], "findings": [], "failed": []}

    def already_checked(self, target_id) -> bool:
        return target_id in self.data["checked"]

    def record_finding(self, target_id, finding):
        self.data["findings"].append({"id": target_id, "finding": finding})
        self.save()

    def record_checked(self, target_id):
        self.data["checked"].append(target_id)
        self.save()
```

---

## Стек для нашей инфры

| Компонент | Что использовать |
|---|---|
| LLM | GLM на `http://10.0.30.18:8080` (OpenAI-compatible API) |
| Оркестрация | Python + asyncio, или OpenClaw skill |
| GitLab интеграция | `python-gitlab` или прямые API запросы |
| Память | JSON файл / SQLite |
| Расписание | systemd timer / OpenClaw cron |
| Алерты | Telegram через OpenClaw |

### Минимальный рабочий агент (Python)

```python
import httpx
import json

GLM_URL = "http://10.0.30.18:8080/v1/chat/completions"
GLM_MODEL = "GLM-4.7-Flash-Uncen-Hrt-NEO-CODE-MAX-imat-D_AU-Q8_0.gguf"

def llm(system: str, user: str) -> str:
    resp = httpx.post(GLM_URL, json={
        "model": GLM_MODEL,
        "messages": [
            {"role": "system", "content": system},
            {"role": "user", "content": user}
        ],
        "max_tokens": 2048,
        "enable_thinking": False
    }, timeout=60)
    return resp.json()["choices"][0]["message"]["content"]

def agent(goal: str, tools: dict, max_steps: int = 10) -> str:
    history = []
    system = f"""
Ты автономный агент. Выполни цель: {goal}

Доступные инструменты: {list(tools.keys())}

На каждый шаг отвечай строго JSON:
{{"thought": "...", "tool": "tool_name", "args": {{}}, "done": false}}
Или если готово: {{"done": true, "result": "итоговый ответ"}}
"""
    for step in range(max_steps):
        response = llm(system, f"История: {json.dumps(history, ensure_ascii=False)}")

        try:
            action = json.loads(response)
        except json.JSONDecodeError:
            history.append({"step": step, "error": "invalid json", "raw": response})
            continue

        if action.get("done"):
            return action.get("result", "done")

        tool_name = action.get("tool")
        if tool_name not in tools:
            history.append({"step": step, "error": f"unknown tool: {tool_name}"})
            continue

        result = tools[tool_name](**action.get("args", {}))
        history.append({"step": step, "thought": action["thought"],
                        "tool": tool_name, "result": str(result)})

    return f"Max steps reached. History: {history}"


# Пример использования
if __name__ == "__main__":
    import subprocess

    tools = {
        "bash": lambda cmd: subprocess.run(cmd, shell=True, capture_output=True, text=True).stdout,
        "read_file": lambda path: open(path).read(),
        "write_file": lambda path, content: open(path, "w").write(content),
    }

    result = agent(
        goal="Найди все Dockerfile в /opt/projects и проверь используют ли они latest тег",
        tools=tools
    )
    print(result)
```

---

## Где применить прямо сейчас

1. **Ночной аудит GitLab репо** — проверять CI файлы на уязвимые паттерны из `github-actions-hardening.md`
2. **Авто-ревью MR** — уже есть `gitlab-reviewer`, можно добавить итерацию
3. **Мониторинг контейнеров** — агент в heartbeat, диагностирует упавшие контейнеры
4. **Обновление зависимостей** — weekly агент, открывает MR с trivy findings

---

## Ссылки

- OpenAI Function Calling (tool use): https://platform.openai.com/docs/guides/function-calling
- Anthropic Tool Use: https://docs.anthropic.com/en/docs/tool-use
- ReAct pattern: https://arxiv.org/abs/2210.03629
- hackerbot-claw (референс): https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation
