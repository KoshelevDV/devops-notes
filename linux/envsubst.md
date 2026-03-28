# envsubst — замена переменных окружения в файлах

Утилита из пакета `gettext`. Заменяет `${VAR}` и `$VAR` на значения из окружения.

## Установка

```bash
sudo dnf install gettext        # Fedora/RHEL
sudo apt install gettext-base   # Debian/Ubuntu
```

## Базовое использование

```bash
# template.yml содержит ${DB_HOST}, ${DB_PORT}
export DB_HOST=10.0.0.1
export DB_PORT=5432

envsubst < template.yml > config.yml

# Вывод на stdout (для pipe)
envsubst < nginx.conf.template | sudo tee /etc/nginx/nginx.conf
```

## Заменить только конкретные переменные

По умолчанию заменяет ВСЕ `$VAR` — опасно в nginx/bash конфигах где есть `$1`, `$uri` и т.д.

```bash
# Только DB_HOST и DB_PORT, остальное не трогать
envsubst '${DB_HOST} ${DB_PORT}' < template.yml > config.yml
```

## Пример с nginx

```bash
# nginx.conf.template
# server_name ${DOMAIN};
# root /var/www/${APP_NAME};

export DOMAIN=example.com
export APP_NAME=myapp

# Безопасно — только нужные переменные
envsubst '${DOMAIN} ${APP_NAME}' < nginx.conf.template > /etc/nginx/conf.d/app.conf
```

## В CI/CD (GitLab CI)

```yaml
deploy:
  script:
    - envsubst '${DB_HOST} ${DB_PORT} ${APP_VERSION}' < k8s/deployment.yml.template > k8s/deployment.yml
    - kubectl apply -f k8s/deployment.yml
```

## В docker-compose (альтернатива)

Docker Compose сам поддерживает `.env` файл и `${VAR}` синтаксис — envsubst не нужен.

## Ограничения

- Нет условий, циклов, функций
- Нет значений по умолчанию (`${VAR:-default}` — не работает)
- Если переменная не задана — заменяется на пустую строку (тихо!)

## Если нужно больше — gomplate

```bash
# Установка
curl -Lo gomplate https://github.com/hairyhenderson/gomplate/releases/latest/download/gomplate_linux-amd64
chmod +x gomplate && sudo mv gomplate /usr/local/bin/

# Значение по умолчанию
# {{ getenv "DB_HOST" "localhost" }}

# Условие
# {{ if getenv "DEBUG" }}debug: true{{ end }}

gomplate -f template.yml -o config.yml
```

## Проверить какие переменные нужны в шаблоне

```bash
grep -oP '\$\{[A-Z_]+\}' template.yml | sort -u
```
