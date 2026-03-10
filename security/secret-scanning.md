# Secret Scanning — предотвращение утечек токенов

## Проблема

Секрет в публичном репо = публичный секрет **навсегда**.
Git history не забывает. Даже после удаления файла токен индексируется:
- GitHub Secret Scanning (автоматически для публичных репо)
- truffleHog, gitleaks боты
- Архивы и зеркала

## Инструменты

### 1. gitleaks — сканирование репо и истории

```bash
# Установка
brew install gitleaks
# или
docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest detect --source /repo

# Сканировать текущий репо (включая всю историю)
gitleaks detect --source . --verbose

# Только staged изменения (pre-commit)
gitleaks protect --staged
```

### 2. pre-commit хук — блокировать до коммита

```bash
# Установка pre-commit
pip install pre-commit

# .pre-commit-config.yaml в корне репо
```

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
```

```bash
# Установить хуки
pre-commit install

# Проверить всё вручную
pre-commit run --all-files
```

### 3. git-secrets (AWS-ориентированный)

```bash
git secrets --install
git secrets --register-aws
git secrets --scan-history
```

### 4. GitHub Secret Scanning (встроенный)

Для публичных репо включён автоматически.
Для приватных — Settings → Security → Secret scanning → Enable.

При обнаружении токена GitHub **автоматически уведомляет провайдера** (AWS, GCP, Stripe и др.)
и они могут отозвать токен.

---

## Что делать если секрет уже попал в историю

```bash
# 1. Немедленно отозвать/ротировать токен в сервисе
# 2. Удалить из истории через git-filter-repo (замена BFG)
pip install git-filter-repo

git filter-repo --path-glob '*.env' --invert-paths
git filter-repo --replace-text <(echo 'OLD_TOKEN==>REDACTED')

# 3. Force push (предупредить команду заранее)
git push --force --all

# 4. Попросить GitHub очистить кеши: Settings → Danger Zone → Delete cached views
```

> ⚠️ BFG Repo Cleaner тоже работает, но git-filter-repo быстрее и активно поддерживается.

---

## .gitignore — минимальный набор

```gitignore
.env
.env.*
!.env.example
*.pem
*.key
*_rsa
*_dsa
*_ed25519
secrets/
credentials/
```

---

## CI/CD: GitLab

```yaml
# В .gitlab-ci.yml — запуск gitleaks на каждый MR
secret-scan:
  image: zricethezav/gitleaks:latest
  stage: test
  script:
    - gitleaks detect --source . --verbose --exit-code 1
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

---

## Источники

- https://github.com/gitleaks/gitleaks
- https://docs.github.com/en/code-security/secret-scanning
- https://git-filter-repo.readthedocs.io
