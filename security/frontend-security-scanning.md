# Сканирование безопасности фронтенда (Next.js / SPA)

> **Дата:** 2026-02-25
> **Теги:** `security` `frontend` `next.js` `dast` `sast` `sca` `owasp-zap` `nuclei` `semgrep` `trivy`

---

## 🎯 Проблема

Фронтенд — это не только статика. Уязвимости бывают в:
- зависимостях npm (CVE в react, axios, next.js и т.д.)
- самом коде (XSS, небезопасный eval, секреты в исходниках)
- поведении приложения в runtime (path traversal, open redirect, CORS misconfiguration)
- Docker-образе (supply chain, права root, RW-файловая система)

Подход — четыре слоя сканирования, каждый своим инструментом.

---

## 🗺 Архитектура

```
Слой           Инструмент          Что ищет
─────────────  ──────────────────  ─────────────────────────────────────
SCA            npm audit           CVE в npm-зависимостях
               Trivy (fs)          CVE + секреты в репо / node_modules
SAST           Semgrep             Уязвимые паттерны в JS/TS коде
               Trivy secret        Секреты в исходниках (ключи, токены)
DAST           OWASP ZAP           Полное сканирование живого приложения
               Nuclei              CVE-шаблоны, path traversal, misconfig
Image scan     Trivy (image)       CVE в базовом образе, слоях
                                   (см. заметку appsec-platform-stack.md)

Результаты → DefectDojo (единое место)
```

---

## ✅ Инструменты

### 1. SCA — зависимости npm

**npm audit** — встроен в npm, ничего не нужно ставить:
```bash
npm audit --json > npm-audit.json

# Только критические
npm audit --audit-level=critical

# Автофикс (осторожно в продакшне, может сломать)
npm audit fix
```

**Trivy** — глубже чем npm audit, покрывает также секреты:
```bash
# Сканирование репозитория (зависимости + секреты)
trivy fs . --format json -o trivy-fs.json

# Только уязвимости, без секретов
trivy fs . --scanners vuln --format json -o trivy-fs.json

# Только секреты
trivy fs . --scanners secret --format json -o trivy-secrets.json
```

---

### 2. SAST — статический анализ кода

**Semgrep** — находит уязвимые паттерны: XSS, prototype pollution, небезопасный dangerouslySetInnerHTML, eval и т.д.:
```bash
# Установка
pip install semgrep
# или
docker pull semgrep/semgrep

# Сканирование (auto = умный выбор правил по языку)
semgrep --config auto --json -o semgrep.json src/

# Только security-правила для TypeScript/React
semgrep --config "p/typescript" --config "p/react" --json -o semgrep.json src/

# В Docker
docker run --rm -v $(pwd):/src semgrep/semgrep \
  semgrep --config auto --json -o /src/semgrep.json /src/src
```

Полезные наборы правил:
- `p/typescript` — TypeScript/JavaScript
- `p/react` — React-специфичные паттерны
- `p/secrets` — секреты в коде
- `p/owasp-top-ten` — OWASP Top 10
- `p/xss` — XSS-паттерны

---

### 3. DAST — динамическое сканирование живого приложения

**OWASP ZAP** — комплексный DAST, самый мощный в open source:
```bash
# Baseline scan (пассивный, не атакует)
docker run --rm ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t https://marketplace.fgdevelop.tech \
  -J zap-baseline.json \
  -r zap-report.html

# Full scan (активный, ищет XSS, SQLi, path traversal и т.д.)
docker run --rm ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py \
  -t https://marketplace.fgdevelop.tech \
  -J zap-full.json \
  -r zap-full-report.html

# API scan (для REST API — передаёшь OpenAPI спецификацию)
docker run --rm ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
  -t https://api.fgdevelop.tech/openapi.json \
  -f openapi \
  -J zap-api.json
```

⚠️ Full scan активно атакует приложение — запускай только на dev/staging, не на продакшне.

---

**Nuclei** — быстрый сканер по шаблонам (CVE, path traversal, misconfiguration):
```bash
# Установка
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
# или
brew install nuclei

# Обновить шаблоны
nuclei -update-templates

# Полное сканирование
nuclei -u https://marketplace.fgdevelop.tech -j -o nuclei.json

# Только CVE-шаблоны
nuclei -u https://marketplace.fgdevelop.tech -tags cve -j -o nuclei-cve.json

# Path traversal + SSRF + misconfig
nuclei -u https://marketplace.fgdevelop.tech \
  -tags traversal,ssrf,misconfig \
  -j -o nuclei-web.json

# Конкретный CVE (например Next.js)
nuclei -u https://marketplace.fgdevelop.tech \
  -tags nextjs \
  -j -o nuclei-nextjs.json
```

---

### 4. Быстрая ручная проверка path traversal

```bash
TARGET="https://marketplace.fgdevelop.tech"

# Прямой обход
curl -si "$TARGET/../../etc/passwd" | head -5

# URL-encoded
curl -si "$TARGET/%2e%2e%2f%2e%2e%2fetc%2fpasswd" | head -5

# Double encoding
curl -si "$TARGET/%252e%252e%252f%252e%252e%252fetc%252fpasswd" | head -5

# Через _next static path
curl -si "$TARGET/_next/../../etc/passwd" | head -5
curl -si "$TARGET/_next/static/../../../../etc/passwd" | head -5

# Null byte
curl -si "$TARGET/../../../../etc/passwd%00.jpg" | head -5
```

Ожидаемый результат — 404 или 400 на всё. Если 200 с содержимым — проблема.

---

## 📤 Хранение результатов → DefectDojo

Все сканеры поддерживают импорт в DefectDojo из коробки:

```bash
DOJO_URL="https://defectdojo.fgdevelop.tech"
DOJO_TOKEN="your-token"
ENGAGEMENT_ID="123"

# npm audit
curl -X POST "$DOJO_URL/api/v2/import-scan/" \
  -H "Authorization: Token $DOJO_TOKEN" \
  -F "scan_type=NPM Audit Scan" \
  -F "file=@npm-audit.json" \
  -F "engagement=$ENGAGEMENT_ID"

# Trivy fs
curl -X POST "$DOJO_URL/api/v2/import-scan/" \
  -H "Authorization: Token $DOJO_TOKEN" \
  -F "scan_type=Trivy Scan" \
  -F "file=@trivy-fs.json" \
  -F "engagement=$ENGAGEMENT_ID"

# Semgrep
curl -X POST "$DOJO_URL/api/v2/import-scan/" \
  -H "Authorization: Token $DOJO_TOKEN" \
  -F "scan_type=Semgrep JSON Report" \
  -F "file=@semgrep.json" \
  -F "engagement=$ENGAGEMENT_ID"

# OWASP ZAP
curl -X POST "$DOJO_URL/api/v2/import-scan/" \
  -H "Authorization: Token $DOJO_TOKEN" \
  -F "scan_type=ZAP Scan" \
  -F "file=@zap-full.json" \
  -F "engagement=$ENGAGEMENT_ID"

# Nuclei
curl -X POST "$DOJO_URL/api/v2/import-scan/" \
  -H "Authorization: Token $DOJO_TOKEN" \
  -F "scan_type=Nuclei Scan" \
  -F "file=@nuclei.json" \
  -F "engagement=$ENGAGEMENT_ID"
```

---

## ⚙️ GitLab CI — полный пайплайн

```yaml
stages:
  - test
  - scan
  - dast
  - upload

# SCA: npm зависимости
sca-npm:
  stage: scan
  image: node:23-alpine
  script:
    - npm audit --json > npm-audit.json || true
  artifacts:
    paths: [npm-audit.json]
  allow_failure: true

# SCA + секреты: Trivy
sca-trivy:
  stage: scan
  image: aquasec/trivy
  script:
    - trivy fs . --format json -o trivy-fs.json || true
  artifacts:
    paths: [trivy-fs.json]

# SAST: Semgrep
sast-semgrep:
  stage: scan
  image: semgrep/semgrep
  script:
    - semgrep --config auto --config "p/react" --config "p/typescript"
        --json -o semgrep.json src/ || true
  artifacts:
    paths: [semgrep.json]

# DAST: Nuclei (быстрый, запускать на staging)
dast-nuclei:
  stage: dast
  image: projectdiscovery/nuclei
  script:
    - nuclei -u $STAGING_URL -tags cve,traversal,misconfig -j -o nuclei.json || true
  artifacts:
    paths: [nuclei.json]
  only:
    - schedules  # запускать по расписанию, не на каждый пуш

# DAST: ZAP baseline (пассивный, безопасно на любом окружении)
dast-zap:
  stage: dast
  image: ghcr.io/zaproxy/zaproxy:stable
  script:
    - zap-baseline.py -t $STAGING_URL -J zap.json -r zap.html || true
  artifacts:
    paths: [zap.json, zap.html]
  only:
    - schedules

# Отправка в DefectDojo
upload-findings:
  stage: upload
  image: alpine/curl
  script:
    - |
      for scan in "NPM Audit Scan:npm-audit.json" \
                  "Trivy Scan:trivy-fs.json" \
                  "Semgrep JSON Report:semgrep.json" \
                  "Nuclei Scan:nuclei.json"; do
        type="${scan%%:*}"
        file="${scan##*:}"
        [ -f "$file" ] && curl -X POST "$DOJO_URL/api/v2/import-scan/" \
          -H "Authorization: Token $DOJO_TOKEN" \
          -F "scan_type=$type" \
          -F "file=@$file" \
          -F "engagement=$DOJO_ENGAGEMENT_ID" || true
      done
  dependencies: [sca-npm, sca-trivy, sast-semgrep, dast-nuclei]
```

---

## 💡 Подводные камни

- **ZAP full scan на проде** — не делать. Активно атакует, может создать мусорные данные в БД или лечь сервис. Только baseline на проде, full — на staging.
- **npm audit false positives** — много шума про уязвимости в devDependencies, которых нет в проде. Фильтровать через `--omit=dev` или смотреть только `production` зависимости.
- **Semgrep `auto` режим** — медленный на больших репо. Ограничь путями: `semgrep --config auto src/`.
- **Nuclei rate limiting** — по умолчанию агрессивный. На чужих хостах добавь `-rate-limit 10`. На своих — ок.
- **OWASP ZAP + SPA** — ZAP плохо работает с SPA (React/Next.js) в базовом режиме. Для полного покрытия нужен Ajax Spider: добавь `-z "-config ajaxSpider.enabled=true"` к zap-full-scan.
- **Nuclei шаблоны** — обновлять регулярно: `nuclei -update-templates`. Без обновления пропустишь свежие CVE.

---

## 📎 Документация

- [OWASP ZAP](https://www.zaproxy.org/docs/)
- [Nuclei](https://docs.projectdiscovery.io/tools/nuclei/overview)
- [Semgrep rules](https://semgrep.dev/explore)
- [Trivy](https://trivy.dev/latest/docs/target/filesystem/)
- [DefectDojo import API](https://defectdojo.github.io/django-DefectDojo/integrations/api-v2-docs/)

---

## 🔗 Источники

- [OWASP Testing Guide — Client Side](https://owasp.org/www-project-web-security-testing-guide/)
- [Next.js CVE-2025-66478](https://nextjs.org/blog/CVE-2025-66478)
- [Nuclei templates — github.com/projectdiscovery/nuclei-templates](https://github.com/projectdiscovery/nuclei-templates)
