# AGENTS.md — devops-notes

## What is this

Личная DevOps-вики Абобы. Заметки на русском языке — инструкции, гайды, решения
задач из реальной практики. Ведёт AI-ассистент Afflictus по триггеру.

**Триггер:** `"Нужна заметка: <вопрос>"` → создать/обновить заметку, закоммитить, сообщить.

## Stack

Markdown + Git. Никакого генератора сайтов, никаких зависимостей.

## Structure

```
kubernetes/    — k8s, etcd, ingress, helm, операторы
docker/        — образы, compose, registry, troubleshooting
linux/         — команды, systemd, сеть, производительность
security/      — CVE, hardening, секреты, аудит, AppSec
networking/    — сети, DNS, firewall, VPN, балансировщики
helm/          — чарты, values, репозитории
git/           — workflow, хуки, troubleshooting
ci-cd/         — пайплайны, GitLab CI, деплой
_template.md   — шаблон новой заметки
README.md      — оглавление
AGENTS.md      — этот файл
```

## Development Rules

### Обязательный формат каждой заметки

```markdown
# <Заголовок>

## 🎯 Проблема
Что за ситуация или вопрос. Конкретно и кратко.

## ✅ Решение
Шаги, команды, примеры. Копипаст-ready.

## 📎 Официальная документация
- [Название](URL)

## 🔗 Источники
- Reddit, форумы, статьи

## ⚠️ Подводные камни
Что не очевидно. Где можно ошибиться.
```

### Правила написания
- Язык: **только русский** (код/команды — английский, это нормально)
- Команды — в code blocks, copy-paste готовые
- Конкретика > теория: реальные примеры, реальные команды
- Подводные камни обязательны — это самая ценная часть
- Официальные доки ссылкой, не пересказывать
- Reddit/форумы — если есть, добавить; люди там часто находят решения которых нет в доках

### Git
- Коммит сразу после создания/обновления заметки
- Сообщение: `<раздел>: <краткое описание>`
  - Пример: `security: AppSec Platform stack (Trivy + D-Track + DefectDojo + Semgrep)`
  - Пример: `networking: B2E store vs remote access control`
- Push: `git push https://KoshelevDV:$(gh auth token)@github.com/KoshelevDV/devops-notes.git main`

### Когда создавать новую заметку vs обновлять существующую
- Новая тема → новый файл в нужном разделе
- Дополнение к существующей теме → обновить существующий файл
- Новый раздел → создать директорию, добавить в README.md оглавление

## Status (2026-02-24)

### Существующие заметки
- `networking/b2e-store-vs-remote-access.md` — B2E store-private/store-public/remote access (Mikrotik, HAProxy, split-DNS, X-Access-Type header)
- `security/appsec-platform-stack.md` — AppSec платформа: Trivy + Dependency-Track + DefectDojo + Semgrep

### TODO (темы для будущих заметок)
- [ ] Trivy Operator установка и настройка в k8s
- [ ] Dependency-Track + DefectDojo интеграция пошагово
- [ ] GitLab CI reusable templates (include:)
- [ ] Helm: rollback стратегии
- [ ] k8s: debug failing pods (события, логи, exec)
- [ ] Docker: multi-stage builds best practices
- [ ] Renovate self-hosted setup

## How to run locally

```bash
git clone https://KoshelevDV:$(gh auth token)@github.com/KoshelevDV/devops-notes.git
cd devops-notes
# Просто открыть в редакторе или браузере GitHub
```

## Pitfalls

- Не создавать заметки без раздела "Подводные камни" — это самая ценная часть
- Не забывать обновлять `README.md` при добавлении нового раздела (папки)
- Триггер — именно `"Нужна заметка: <вопрос>"`, не `"запиши"` или `"сохрани"`
