# Supply Chain атака на Trivy GitHub Action (март 2026)

> Источник: https://socket.dev/blog/trivy-under-attack-again-github-actions-compromise

## Что произошло

Подмена 75 из 76 тегов версий `aquasecurity/trivy-action` на GitHub.
Все теги от `0.18.0` и выше (кроме `0.35.0`) содержат вредоносный коммит.

## Что крадёт вредоносный код

- AWS / GCP / Azure credentials
- SSH-ключи
- git-credentials
- Kubernetes service account tokens и секреты
- GitHub Secrets, переданные в GitHub Runner
- Крипто-кошельки

Пострадали **>10 000 GitHub workflow-файлов**.

## Кто под угрозой

Любой pipeline с:
```yaml
uses: aquasecurity/trivy-action@<любая версия кроме 0.35.0>
```

## Безопасные варианты

```yaml
# Вариант 1 — единственный безопасный тег
uses: aquasecurity/trivy-action@0.35.0

# Вариант 2 — пин по SHA (рекомендуется)
uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1
```

## Признак компрометации

Проверить в GitHub Actions логах запросы к репозиторию **`tpcp-docs`** — он использовался атакующими как exfiltration endpoint.

## Наши пайплайны (проверено 2026-03-20)

**Не затронуты** — `aquasecurity/trivy-action` не используется:
- Все GitLab CI пайплайны используют Docker образ `aquasec/trivy` напрямую
- Trivy Operator в k8s — отдельный деплой, не связан с trivy-action

## Что делать если использовали

1. Считать все секреты из GitHub Actions **скомпрометированными**
2. Ротировать: AWS/GCP/Azure credentials, SSH-ключи, GitHub Secrets, K8s tokens
3. Проверить логи на запросы к `tpcp-docs`
4. Заменить версию на пин по SHA

## Урок

**Никогда не использовать плавающие теги в CI** (`@v1`, `@latest`, `@<semver>`).
Всегда пинить по SHA:
```yaml
# Плохо
uses: some-action@v2

# Хорошо
uses: some-action@abc123def456...  # v2.0.1
```
