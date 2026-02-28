# HashiCorp Vault — полное руководство

> **Дата:** 2026-02-28  
> **Теги:** `vault` `secrets` `pki` `kubernetes` `ha` `auto-unseal` `security`

---

## 🎯 Что такое Vault

HashiCorp Vault — централизованное хранилище секретов с:
- **Dynamic secrets** — генерация одноразовых credentials (DB, AWS, SSH)
- **Encryption as a Service** — шифрование данных без хранения ключей в приложении
- **PKI** — выдача TLS-сертификатов с коротким TTL
- **Lease & Renewal** — у каждого секрета есть TTL, автоматическое отзывание
- **Audit log** — полная запись всех запросов и ответов

---

## ✅ Развёртывание

---

### Dev-режим (только для тестов)

```bash
vault server -dev -dev-root-token-id="root"

export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root
vault status
```

> ⚠️ Dev-режим: данные в памяти, unseal автоматически, не использовать в продакшене.

---

### Docker Compose (single node)

```yaml
# docker-compose.yml
services:
  vault:
    image: hashicorp/vault:1.17
    restart: unless-stopped
    ports:
      - "8200:8200"
    environment:
      VAULT_ADDR: http://0.0.0.0:8200
    cap_add:
      - IPC_LOCK           # mlock — запрет свопа памяти с секретами
    volumes:
      - ./config/vault.hcl:/vault/config/vault.hcl
      - vault-data:/vault/data
    command: vault server -config=/vault/config/vault.hcl

volumes:
  vault-data:
```

```hcl
# config/vault.hcl
ui            = true
disable_mlock = false

storage "file" {
  path = "/vault/data"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_disable   = true   # в продакшене — TLS обязателен
}

api_addr     = "http://vault:8200"
cluster_addr = "http://vault:8201"
```

---

### Kubernetes — Helm (рекомендуется)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Минимальная установка (single node + Raft storage)
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.dataStorage.size=10Gi

# Посмотреть статус
kubectl exec -n vault vault-0 -- vault status
```

**Полный values.yaml для продакшена:**

```yaml
# values-prod.yaml
global:
  tlsDisable: false

server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true
        listener "tcp" {
          address       = "[::]:8200"
          tls_cert_file = "/vault/userconfig/tls/tls.crt"
          tls_key_file  = "/vault/userconfig/tls/tls.key"
        }
        storage "raft" {
          path    = "/vault/data"
          retry_join {
            leader_api_addr = "https://vault-0.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "https://vault-1.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "https://vault-2.vault-internal:8200"
          }
        }
        seal "awskms" {
          region     = "us-east-1"
          kms_key_id = "alias/vault-unseal"
        }
        service_registration "kubernetes" {}

  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

  dataStorage:
    enabled: true
    size: 20Gi
    storageClass: gp3

  auditStorage:
    enabled: true
    size: 10Gi

ui:
  enabled: true
  serviceType: ClusterIP

injector:
  enabled: true   # Vault Agent Injector
```

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --values values-prod.yaml
```

---

## 🔐 Инициализация и Unseal

### Первый запуск

```bash
# Инициализация — только один раз!
vault operator init \
  -key-shares=5 \         # 5 частей ключа
  -key-threshold=3        # нужно 3 из 5 для unseal

# Вывод:
# Unseal Key 1: abc...
# Unseal Key 2: def...
# Unseal Key 3: ghi...
# Unseal Key 4: jkl...
# Unseal Key 5: mno...
# Initial Root Token: hvs.XXXXXXX
```

> ⚠️ Ключи unseal и root token сохранить в разных местах. Root token использовать только при настройке, потом отозвать.

### Ручной unseal (3 из 5 ключей)

```bash
vault operator unseal <Unseal_Key_1>
vault operator unseal <Unseal_Key_2>
vault operator unseal <Unseal_Key_3>

vault status   # Sealed: false
```

---

## 🔓 Auto-Unseal

Vault автоматически unseals при перезапуске, используя внешний KMS.

### AWS KMS

```hcl
# vault.hcl
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/vault-unseal-key"
  # Или явный ARN:
  # kms_key_id = "arn:aws:kms:us-east-1:123456789:key/abcd-1234"
}
```

```bash
# IAM Policy для Vault (минимально необходимые права):
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "kms:Encrypt",
      "kms:Decrypt",
      "kms:DescribeKey"
    ],
    "Resource": "arn:aws:kms:us-east-1:123456789:key/abcd-1234"
  }]
}
```

### Azure Key Vault

```hcl
seal "azurekeyvault" {
  tenant_id      = "tenant-id"
  client_id      = "client-id"
  client_secret  = "client-secret"
  vault_name     = "my-key-vault"
  key_name       = "vault-unseal"
}
```

### GCP Cloud KMS

```hcl
seal "gcpckms" {
  project     = "my-gcp-project"
  region      = "global"
  key_ring    = "vault-keyring"
  crypto_key  = "vault-unseal"
}
```

### Transit Auto-Unseal (Vault-в-Vault)

Один Vault unseals другой через Transit secrets engine. Используется для prod/dev изоляции.

```hcl
# На child Vault:
seal "transit" {
  address         = "https://core-vault:8200"
  token           = "s.xxxxxxx"
  disable_renewal = "false"
  key_name        = "unseal-key"
  mount_path      = "transit/"
}
```

### Unseal после настройки auto-unseal

```bash
# При наличии auto-unseal — используется migrate:
vault operator init -recovery-shares=5 -recovery-threshold=3

# Recovery keys нужны только если KMS недоступен
# vault operator generate-root ...
```

---

## 🗄️ Хранилища (Storage Backends)

| Backend | HA | Рекомендация |
|---------|----|----|
| **Raft (Integrated Storage)** | ✅ | ⭐ Рекомендуется, встроен в Vault |
| Consul | ✅ | Только если уже есть Consul |
| DynamoDB | ✅ | AWS-специфично |
| PostgreSQL | ❌ | Single node |
| File | ❌ | Dev/small |
| In-Memory | ❌ | Dev только |

**Raft — настройка кластера:**

```bash
# На vault-0 (лидер):
vault operator init
vault operator unseal ...

# Присоединить vault-1 к кластеру:
vault operator raft join https://vault-0.vault-internal:8200

# Проверить кластер:
vault operator raft list-peers
```

---

## 🗝️ Secrets Engines

Включить engine: `vault secrets enable -path=<путь> <тип>`

---

### KV v2 — ключ-значение с версионированием

```bash
vault secrets enable -path=secret kv-v2

# Записать
vault kv put secret/myapp/prod \
  db_password=supersecret \
  api_key=abc123

# Читать
vault kv get secret/myapp/prod
vault kv get -field=db_password secret/myapp/prod

# Версии
vault kv get -version=2 secret/myapp/prod
vault kv history secret/myapp/prod

# Мягкое удаление (можно восстановить)
vault kv delete secret/myapp/prod
# Безвозвратное удаление
vault kv destroy -versions=1,2 secret/myapp/prod
```

---

### Database — динамические credentials

Vault сам создаёт/удаляет пользователей в БД с TTL.

```bash
vault secrets enable database

# Настройка подключения (PostgreSQL):
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/mydb" \
  allowed_roles="readonly,readwrite" \
  username="vault" \
  password="vaultpassword"

# Роль с шаблоном SQL:
vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl=1h \
  max_ttl=24h

# Получить credentials (создаётся временный пользователь в БД):
vault read database/creds/readonly
# username: v-root-readonly-AbCdEfGh
# password: A1b2C3d4...
# lease_duration: 1h
```

Поддерживаются: PostgreSQL, MySQL, MongoDB, Redis, Cassandra, Elasticsearch, Oracle, MSSQL.

---

### PKI — Certificate Authority

```bash
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=87600h pki

# Создать Root CA:
vault write pki/root/generate/internal \
  common_name="My Root CA" \
  ttl=87600h

# Или импортировать существующий CA:
vault write pki/config/ca pem_bundle=@ca-bundle.pem

# Настроить URLs:
vault write pki/config/urls \
  issuing_certificates="https://vault:8200/v1/pki/ca" \
  crl_distribution_points="https://vault:8200/v1/pki/crl"

# Создать Intermediate CA:
vault secrets enable -path=pki_int pki
vault write pki_int/intermediate/generate/internal \
  common_name="My Intermediate CA"
# Подписать Intermediate у Root:
vault write pki/root/sign-intermediate csr=@int.csr \
  format=pem_bundle ttl=43800h

# Роль для выдачи сертификатов:
vault write pki_int/roles/myapp \
  allowed_domains="myapp.example.com" \
  allow_subdomains=true \
  max_ttl=720h

# Выдать сертификат:
vault write pki_int/issue/myapp \
  common_name="api.myapp.example.com" \
  ttl=24h
```

---

### SSH — подпись SSH-ключей

```bash
vault secrets enable ssh

# CA для подписи:
vault write ssh/config/ca generate_signing_key=true

# Роль:
vault write ssh/roles/admin \
  key_type=ca \
  allowed_users="ubuntu,ec2-user" \
  default_extensions='{"permit-pty":""}' \
  ttl=1h

# Подписать публичный ключ:
vault write ssh/sign/admin \
  public_key=@~/.ssh/id_ed25519.pub \
  valid_principals=ubuntu

# На сервере — доверять Vault CA:
# /etc/ssh/sshd_config:
# TrustedUserCAKeys /etc/ssh/vault_ca.pub
```

---

### AWS — динамические IAM credentials

```bash
vault secrets enable aws

vault write aws/config/root \
  access_key=$AWS_ACCESS_KEY_ID \
  secret_key=$AWS_SECRET_ACCESS_KEY \
  region=us-east-1

vault write aws/roles/readonly \
  credential_type=iam_user \
  policy_document='{
    "Version": "2012-10-17",
    "Statement": [{"Effect":"Allow","Action":"s3:GetObject","Resource":"*"}]
  }'

# Получить временные credentials:
vault read aws/creds/readonly
# access_key:     AKIA...
# secret_key:     ...
# lease_duration: 768h
```

---

### Transit — Encryption as a Service

Vault шифрует данные, ключи никогда не покидают Vault.

```bash
vault secrets enable transit

vault write -f transit/keys/myapp   # Создать ключ

# Шифровать:
vault write transit/encrypt/myapp \
  plaintext=$(echo "secret data" | base64)
# ciphertext: vault:v1:abc123...

# Расшифровать:
vault write transit/decrypt/myapp \
  ciphertext="vault:v1:abc123..."
# plaintext: base64-encoded

# Ротация ключа (старые версии остаются для расшифровки):
vault write -f transit/keys/myapp/rotate

# Rewrap (перешифровать старые данные новым ключом):
vault write transit/rewrap/myapp \
  ciphertext="vault:v1:abc123..."
```

---

## 🔑 Auth Methods

---

### Token (встроенный)

```bash
# Создать токен с политиками:
vault token create \
  -policy=myapp-readonly \
  -ttl=24h \
  -renewable=true

# Создать orphan-токен (без родителя):
vault token create -orphan -policy=myapp

# Обновить TTL:
vault token renew <token>

# Отозвать:
vault token revoke <token>
vault token revoke -self   # отозвать текущий
```

---

### AppRole — для приложений и CI/CD

```bash
vault auth enable approle

vault write auth/approle/role/myapp \
  token_policies=myapp-policy \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=10m \      # SecretID живёт 10 минут
  secret_id_num_uses=1     # Однократный SecretID

# Получить RoleID (публичный, можно хранить в конфиге):
vault read auth/approle/role/myapp/role-id

# Получить SecretID (доставлять через CI, not в коде):
vault write -f auth/approle/role/myapp/secret-id

# Логин:
vault write auth/approle/login \
  role_id=$ROLE_ID \
  secret_id=$SECRET_ID
```

---

### Kubernetes Auth

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=production \
  token_policies=myapp-policy \
  token_ttl=1h
```

```python
# Логин из пода (Python):
import hvac, os

client = hvac.Client(url="https://vault:8200")
with open("/var/run/secrets/kubernetes.io/serviceaccount/token") as f:
    jwt = f.read()
client.auth.kubernetes.login(role="myapp", jwt=jwt)
```

---

### OIDC/JWT — SSO, GitHub Actions, GitLab CI

```bash
vault auth enable jwt

# GitHub Actions OIDC:
vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"

vault write auth/jwt/role/github-actions \
  role_type=jwt \
  bound_audiences="https://github.com/myorg" \
  user_claim=actor \
  bound_claims='{"repository":"myorg/myrepo"}' \
  token_policies=deploy-policy \
  token_ttl=15m
```

---

### UserPass — для операторов

```bash
vault auth enable userpass

vault write auth/userpass/users/admin \
  password=securepassword \
  policies=admin-policy

vault login -method=userpass \
  username=admin \
  password=securepassword
```

---

## 📋 Policies

```hcl
# myapp-policy.hcl
# Читать секреты приложения
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# Получать динамические DB credentials
path "database/creds/readonly" {
  capabilities = ["read"]
}

# Шифровать/расшифровать через Transit
path "transit/encrypt/myapp" {
  capabilities = ["update"]
}
path "transit/decrypt/myapp" {
  capabilities = ["update"]
}

# Обновлять собственный токен
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
```

```bash
# Применить политику:
vault policy write myapp-policy myapp-policy.hcl

# Capabilities: create, read, update, delete, list, sudo, deny
# "deny" — явный запрет, перебивает всё остальное
```

**Шаблоны в политиках (identity templating):**

```hcl
# Каждый сервис видит только свой namespace
path "secret/data/{{identity.entity.aliases.auth_kubernetes_xxx.metadata.service_account_namespace}}/*" {
  capabilities = ["read"]
}
```

---

## ⏱ Leases — TTL и ротация

```bash
# Список активных leases:
vault list sys/leases/lookup/database/creds/readonly/

# Обновить lease:
vault lease renew <lease_id>

# Отозвать lease (удалить credentials):
vault lease revoke <lease_id>

# Отозвать всё по префиксу:
vault lease revoke -prefix database/creds/readonly/
```

В приложении всегда реализовывать **lease renewal loop**:
```python
# Псевдокод
while True:
    creds = vault.read("database/creds/readonly")
    lease_id = creds["lease_id"]
    ttl = creds["lease_duration"]
    
    # Обновлять за 30% до истечения
    sleep(ttl * 0.7)
    vault.write(f"sys/leases/renew", lease_id=lease_id)
```

---

## 📊 HA — High Availability

### Raft Integrated Storage (рекомендуется)

3 или 5 нод (нечётное количество для кворума).

```
vault-0 (leader) ←→ vault-1 (standby) ←→ vault-2 (standby)
       ↑ Raft consensus ↑
```

- **Leader** — обрабатывает все write-операции
- **Standby** — перенаправляет запросы к leader (или forwarding)
- **Performance Standby** (Enterprise) — читает локально

```bash
# Посмотреть лидера:
vault operator raft list-peers

# Принудительная смена лидера:
vault operator step-down
```

### Snapshot и Restore

```bash
# Снять snapshot (backup состояния Raft):
vault operator raft snapshot save backup.snap

# Восстановить:
vault operator raft snapshot restore -force backup.snap
```

**Автоматические snapshot в Kubernetes:**

```yaml
# CronJob для backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-snapshot
spec:
  schedule: "0 */6 * * *"   # каждые 6 часов
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: snapshot
              image: hashicorp/vault:1.17
              command:
                - /bin/sh
                - -c
                - |
                  vault login $VAULT_TOKEN
                  vault operator raft snapshot save /snapshots/vault-$(date +%Y%m%d-%H%M).snap
              env:
                - name: VAULT_ADDR
                  value: https://vault.vault:8200
                - name: VAULT_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: vault-snapshot-token
                      key: token
              volumeMounts:
                - mountPath: /snapshots
                  name: snapshots
```

---

## 🔍 Audit Devices

Логировать все запросы обязательно в продакшене.

```bash
# File audit (рекомендуется + дополнительный):
vault audit enable file file_path=/vault/logs/audit.log

# Syslog:
vault audit enable syslog tag=vault facility=AUTH

# Socket (для SIEM):
vault audit enable socket address=logstash:5000 socket_type=tcp

# Список включённых:
vault audit list
```

> ⚠️ Vault **не принимает запросы**, если все audit devices недоступны (запись в лог невозможна). Всегда настраивать 2+ audit device.

Формат лога — JSON:
```json
{
  "time": "2026-02-28T12:00:00Z",
  "type": "request",
  "auth": {"token_type": "service", "policies": ["myapp-policy"]},
  "request": {"path": "secret/data/myapp/prod", "operation": "read"},
  "response": {"data": {"keys": ["db_password"]}}
}
```

---

## 🔧 Операционные команды

```bash
# Статус кластера:
vault status
vault operator raft list-peers

# Профилирование и отладка:
vault debug                    # собрать диагностику в архив
vault monitor -log-level=debug

# Seal (экстренная блокировка):
vault operator seal

# Rotate encryption key (внутренний ключ шифрования БД Vault):
vault operator rotate

# Rotate Root CA credentials:
vault write -f sys/rotate

# Garbage collect leases:
vault write sys/leases/tidy

# Информация о токене:
vault token lookup
vault token lookup <other-token>

# Проверить capabilities:
vault token capabilities secret/data/myapp/prod
```

---

## 💡 Подводные камни

- **Unseal keys нигде не хранить вместе** — Shamir split: ключи 1-2-3 у разных людей, 4-5 в оффлайн-хранилище
- **Root token отзывать после первичной настройки** — использовать только через `vault operator generate-root` при необходимости
- **Audit log — одна точка отказа** — всегда 2+ audit device (file + syslog)
- **HA не = backup** — Raft-snapshot не заменяет отдельный backup; если все ноды умрут вместе с данными, snapshot — единственное спасение
- **Lease renewal забывают** — если приложение не обновляет lease, credentials истекают, приложение падает в продакшене
- **mlock** — Vault требует `IPC_LOCK` capability в контейнере или `disable_mlock=true` (небезопасно)
- **TLS обязателен** — в продакшене никогда `tls_disable=true`
- **Namespaces** (Enterprise) — изолировать команды/продукты, не давать доступ к sys/
- **Secrets sprawl** — без политик именования (`team/app/env/key`) Vault превращается в неуправляемую помойку

---

## 🏆 Best Practices

| Тема | Рекомендация |
|------|-------------|
| **Unseal** | Auto-unseal через KMS (AWS/GCP/Azure), Recovery keys у 3+ людей |
| **Storage** | Integrated Raft с 3 нодами, snapshot каждые 6 часов в S3 |
| **Root token** | Отозвать после настройки, использовать `operator generate-root` |
| **Auth** | AppRole для CI, Kubernetes Auth для подов, OIDC для людей |
| **Policies** | Least privilege, именование `team-app-env`, deny по умолчанию |
| **Secrets** | KV v2 для статики, Dynamic secrets для DB/Cloud |
| **PKI** | Короткие TTL (24h для leaf certs), автоматическое обновление |
| **Audit** | 2+ audit devices, интеграция с SIEM |
| **Мониторинг** | Prometheus metrics (`vault.core.unsealed`, `vault.expire.num_leases`) |
| **Именование** | `secret/team/app/env/component` — предсказуемая иерархия |
| **Ротация** | Dynamic secrets где возможно, статические ротировать через lease |

---

## 📊 Мониторинг (Prometheus)

```hcl
# vault.hcl
telemetry {
  prometheus_retention_time = "30s"
  disable_hostname          = true
}
```

```bash
# Метрики:
curl https://vault:8200/v1/sys/metrics?format=prometheus
```

Ключевые метрики:
- `vault_core_unsealed` — 1 = unsealed
- `vault_expire_num_leases` — количество активных leases (рост = утечка)
- `vault_raft_leader` — есть ли лидер
- `vault_runtime_gc_pause_ns` — GC давление

---

## 📎 Документация

- [Vault Docs](https://developer.hashicorp.com/vault/docs)
- [Vault Architecture](https://developer.hashicorp.com/vault/docs/internals/architecture)
- [Integrated Storage (Raft)](https://developer.hashicorp.com/vault/docs/configuration/storage/raft)
- [Auto-unseal](https://developer.hashicorp.com/vault/docs/concepts/seal#auto-unseal)
- [Helm Chart](https://developer.hashicorp.com/vault/docs/platform/k8s/helm)
- [External Secrets Operator + Vault](https://external-secrets.io/latest/provider/hashicorp-vault/)

---

## 🔗 Источники

- [Vault Production Hardening Guide](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening)
- [HA with Integrated Storage Tutorial](https://developer.hashicorp.com/vault/tutorials/raft/raft-storage)
- [Vault on Kubernetes — Best Practices](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide)

---

## 🔖 Реальные кейсы использования

---

### KV v2 — Конфигурация микросервисов

**Кейс: Замена ConfigMap + env vars в Kubernetes**

Проблема: в команде 15 микросервисов, каждый хранит `DB_URL`, `REDIS_URL`, `API_KEY` в ConfigMap и Secrets. При ротации ключа нужно вручную обновить 15 манифестов и рестартовать поды.

Решение: Все конфиги в Vault KV v2, Vault Agent Injector монтирует их как файлы.

```
Структура ключей:
secret/
  payments-service/
    prod/
      db_url
      stripe_key
      redis_url
    staging/
      db_url
      stripe_key
  orders-service/
    prod/
      db_url
      kafka_bootstrap
```

```yaml
# Deployment — только аннотации, никаких env vars с секретами
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "payments-service"
  vault.hashicorp.com/agent-inject-secret-config: "secret/data/payments-service/prod"
  vault.hashicorp.com/agent-inject-template-config: |
    {{- with secret "secret/data/payments-service/prod" -}}
    DB_URL={{ .Data.data.db_url }}
    STRIPE_KEY={{ .Data.data.stripe_key }}
    {{- end }}
```

```python
# В приложении — читаем из файла, не из env:
import os

def load_config():
    config = {}
    with open("/vault/secrets/config") as f:
        for line in f:
            key, _, value = line.strip().partition("=")
            config[key] = value
    return config
```

При смене `stripe_key`: `vault kv put secret/payments-service/prod stripe_key=новый` → Vault Agent подхватывает изменение без рестарта пода.

---

**Кейс: Версионирование конфигов для rollback**

Деплой пошёл не так — в новой версии config оказался неправильный DSN.

```bash
# Посмотреть историю:
vault kv history secret/payments-service/prod

# Откатиться к версии 5:
vault kv get -version=5 secret/payments-service/prod
vault kv get -version=5 -format=json secret/payments-service/prod \
  | jq '.data.data' \
  | vault kv put secret/payments-service/prod -
```

Откат занимает 30 секунд вместо передеплоя.

---

### Database — Динамические credentials

**Кейс: Zero-trust доступ к PostgreSQL**

Проблема: один статический пользователь `app_user` с паролем в конфиге. Пароль живёт годами, ротация ломает продакшен, при взломе сервера злоумышленник получает постоянный доступ к БД.

```
До:  app → DB (статичный app_user/password навсегда)
После: app → Vault → DB (временный v-app-abcd1234 / password на 1 час)
```

```bash
# Vault создаёт пользователя при каждом запросе:
vault read database/creds/app-readonly

# Key              Value
# lease_id         database/creds/app-readonly/AbCdEfGhIjKlMnOp
# lease_duration   1h
# username         v-approle-app-abcd-1234567890
# password         A-xYzAbCd1234...
```

```python
# Spring Boot — автоматическое обновление credentials через Vault Spring:
# application.yml
spring:
  cloud:
    vault:
      database:
        role: app-readonly
        backend: database
      config:
        lifecycle:
          enabled: true
          min-renewal: 10s   # обновлять за 10s до истечения
```

**Кейс: Incident response — скомпрометированный сервис**

Сервис `payments-service` был взломан. Нужно немедленно отозвать его доступ к БД.

```bash
# Отозвать ВСЕ credentials выданные этой роли — за 1 команду:
vault lease revoke -prefix database/creds/payments-readonly/

# Все временные пользователи v-payments-* немедленно удалены из PostgreSQL.
# Злоумышленник теряет доступ через секунды, без изменения основного пользователя.
```

Статические credentials: пришлось бы менять пароль + деплоить всё заново. Vault: одна команда.

---

**Кейс: Аналитики и разработчики — временный read-only доступ**

```bash
# DevOps выдаёт аналитику временный доступ на 4 часа:
vault write database/roles/analyst-temp \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON analytics_schema.* TO \"{{name}}\";" \
  default_ttl=4h \
  max_ttl=4h

vault read database/creds/analyst-temp
# Пользователь автоматически удаляется через 4 часа — даже если аналитик забудет выйти.
```

---

### PKI — Сертификаты

**Кейс: mTLS между микросервисами (service mesh без Istio)**

Проблема: 20 микросервисов должны проверять друг друга через TLS. Управлять 20 парами сертификатов вручную невозможно.

```
payments-service (cert valid 24h) → Vault PKI → orders-service (cert valid 24h)
                                   ↕ авторенью через 20 часов
```

```bash
# Каждый сервис получает свой сертификат при старте:
vault write pki_int/issue/microservices \
  common_name="payments-service.production.svc" \
  alt_names="payments-service,payments-service.production" \
  ttl=24h

# Сертификат действует 24 часа — если сервис взломан,
# максимальный blast radius 24 часа без каких-либо действий оператора.
```

```python
# Python — автоматический renew через библиотеку hvac:
import hvac, ssl, threading, time

def renew_cert(vault_client, role, domain):
    while True:
        cert_data = vault_client.secrets.pki.generate_certificate(
            name=role,
            common_name=domain,
            extra_params={"ttl": "24h"}
        )
        # Обновить SSL context в рантайме
        update_ssl_context(cert_data)
        # Обновить за 4 часа до истечения (20h из 24h)
        time.sleep(20 * 3600)

threading.Thread(target=renew_cert, args=(client, "microservices", "payments-service.production.svc"), daemon=True).start()
```

---

**Кейс: cert-manager + Vault PKI для Kubernetes Ingress**

```yaml
# ClusterIssuer — cert-manager выдаёт сертификаты через Vault
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer
spec:
  vault:
    path: pki_int/sign/kubernetes-ingress
    server: https://vault.vault:8200
    auth:
      kubernetes:
        role: cert-manager
        mountPath: /v1/auth/kubernetes
        serviceAccountRef:
          name: cert-manager

---
# Ingress — сертификат выдаётся и обновляется автоматически
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: vault-issuer
spec:
  tls:
    - hosts: [api.myapp.com]
      secretName: api-tls
  rules:
    - host: api.myapp.com
```

Сертификат обновляется за 30 дней до истечения автоматически — человек не участвует.

---

**Кейс: Замена wildcard-сертификата**

Проблема: единый wildcard `*.myapp.com` → если скомпрометирован, падает всё. Ротация раз в год руками.

После: каждый сервис получает отдельный сертификат с TTL 7 дней. Автообновление. Компрометация одного → blast radius один сервис.

---

### SSH — Временный доступ

**Кейс: Bastionless SSH — никаких authorized_keys**

Проблема: у 50 серверов в `~/.ssh/authorized_keys` лежат 30 ключей. Уволившийся сотрудник — нужно чистить все 50 серверов. Кто-то забыл → ключ висит годами.

```bash
# Оператор получает подписанный SSH-сертификат на 1 час:
vault write ssh/sign/sre \
  public_key=@~/.ssh/id_ed25519.pub \
  valid_principals=ubuntu \
  ttl=1h

# Подключение с подписанным сертификатом:
ssh -i ~/.ssh/id_ed25519-cert.pub ubuntu@server-ip
```

```bash
# На каждом сервере — один раз при provisioning:
# /etc/ssh/sshd_config
TrustedUserCAKeys /etc/ssh/vault_ca.pub

# Никаких authorized_keys — доступ по политике Vault.
# Уволился сотрудник → удалить из Vault → доступ пропадает мгновенно.
```

---

**Кейс: Временный доступ подрядчику**

```bash
# Создать роль с ограниченным временем и конкретным пользователем:
vault write auth/userpass/users/contractor \
  password=temppassword \
  policies=ssh-readonly-policy

# policy:
path "ssh/sign/readonly-role" {
  capabilities = ["update"]
}

# Подрядчик логинится и получает SSH cert на 8 часов.
# После истечения — доступа нет без повторного запроса.
```

---

### AWS — Динамические IAM credentials

**Кейс: CI/CD pipeline без долгосрочных AWS ключей**

Проблема: `AWS_ACCESS_KEY_ID` и `AWS_SECRET_ACCESS_KEY` лежат в GitLab CI Variables, живут годами, имеют широкие права.

```
До:  GitLab Runner → статичный IAM User (ключ 2 года, права Admin)
После: GitLab Runner → Vault JWT Auth → Vault AWS Engine → временный IAM (15 мин, только S3 push)
```

```yaml
# .gitlab-ci.yml
deploy:
  script:
    # Получить временный токен Vault через GitLab OIDC
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login
        role=gitlab-deploy jwt=$CI_JOB_JWT_V2)
    # Получить временные AWS credentials
    - export $(vault read -format=json aws/creds/deploy-role
        | jq -r '.data | to_entries[] | "\(.key | ascii_upcase)=\(.value)"')
    # Деплой
    - aws s3 sync dist/ s3://my-bucket/
  after_script:
    - vault lease revoke $LEASE_ID   # сразу отзываем после job
```

**Кейс: Lambda без embedded credentials**

```python
# Lambda function — получает credentials из Vault при старте
import boto3
import hvac

def lambda_handler(event, context):
    # Lambda использует IAM Role для авторизации в Vault
    vault = hvac.Client(url="https://vault.internal:8200")
    vault.auth.aws.iam_login(role="lambda-s3-reader")
    
    s3_creds = vault.read("aws/creds/s3-reader")["data"]
    
    s3 = boto3.client("s3",
        aws_access_key_id=s3_creds["access_key"],
        aws_secret_access_key=s3_creds["secret_key"]
    )
    # s3_creds истекут через 15 минут — раньше чем Lambda timeout
```

---

### Transit — Encryption as a Service

**Кейс: Шифрование PII в базе данных (GDPR/PCI)**

Проблема: таблица `users` хранит `email`, `phone`, `card_number` в открытом виде. Утечка БД = компрометация данных всех пользователей.

```python
import hvac, base64
from sqlalchemy import Column, String

vault = hvac.Client(url="https://vault:8200")
vault.auth.kubernetes.login(role="payments-app", jwt=open("/var/run/secrets/...token").read())

def encrypt_field(plaintext: str) -> str:
    """Шифрует перед записью в БД"""
    resp = vault.secrets.transit.encrypt_data(
        name="user-pii",
        plaintext=base64.b64encode(plaintext.encode()).decode()
    )
    return resp["data"]["ciphertext"]  # vault:v1:AbCdEf...

def decrypt_field(ciphertext: str) -> str:
    """Расшифровывает при чтении"""
    resp = vault.secrets.transit.decrypt_data(
        name="user-pii",
        ciphertext=ciphertext
    )
    return base64.b64decode(resp["data"]["plaintext"]).decode()

# В БД хранится: vault:v1:AbCdEfGhIjKlMnOpQrStUvWxYz...
# Утечка БД = зашифрованный мусор без доступа к Vault
```

**Ротация ключа шифрования без расшифровки всех записей:**

```bash
# Ротировать ключ — старые данные остаются читаемы:
vault write -f transit/keys/user-pii/rotate

# Перешифровать все записи новым ключом (в фоне):
vault write transit/rewrap/user-pii ciphertext="vault:v1:старый_шифртекст"
# → vault:v2:новый_шифртекст

# Запретить расшифровку старых версий ключа:
vault write transit/keys/user-pii/config min_decryption_version=2
```

---

**Кейс: Подпись токенов / данных (замена self-signed JWT)**

```python
# Подписывать критичные операции (переводы, смена пароля):
def sign_transfer(transfer_data: dict) -> str:
    payload = json.dumps(transfer_data).encode()
    resp = vault.secrets.transit.sign_data(
        name="transfer-signing-key",
        hash_input=base64.b64encode(payload).decode(),
        hash_algorithm="sha2-256",
        signature_algorithm="pkcs1v15"
    )
    return resp["data"]["signature"]

def verify_transfer(transfer_data: dict, signature: str) -> bool:
    payload = json.dumps(transfer_data).encode()
    resp = vault.secrets.transit.verify_signed_data(
        name="transfer-signing-key",
        hash_input=base64.b64encode(payload).decode(),
        signature=signature
    )
    return resp["data"]["valid"]
```

Ключ никогда не покидает Vault → компрометация приложения не даёт доступ к ключу подписи.

---

### AppRole — Интеграция CI/CD и сервисов

**Кейс: Terraform Cloud получает credentials для деплоя**

```bash
# Terraform Cloud runner логинится через AppRole:
# RoleID — публичный, в конфиге Terraform
# SecretID — выдаётся оператором разово через CI/CD orchestrator

# Terraform provider:
provider "vault" {
  address = "https://vault.example.com"
  auth_login_approle {
    role_id   = var.vault_role_id
    secret_id = var.vault_secret_id  # из env TF_VAR_vault_secret_id
  }
}

# Получить credentials для AWS деплоя:
data "vault_aws_access_credentials" "deploy" {
  backend = "aws"
  role    = "terraform-deploy"
}

provider "aws" {
  access_key = data.vault_aws_access_credentials.deploy.access_key
  secret_key = data.vault_aws_access_credentials.deploy.secret_key
}
```

---

**Кейс: Response Wrapping — безопасная доставка SecretID**

Проблема: как доставить SecretID в приложение, не засветив его?

```bash
# Orchestrator (Nomad, K8s init container) получает wrapped SecretID:
# Wrapped token живёт 60 секунд и может быть использован только 1 раз.
WRAPPED=$(vault write -wrap-ttl=60s -f auth/approle/role/myapp/secret-id \
  -format=json | jq -r '.wrap_info.token')

# Передать WRAPPED в приложение (через env, init container)
# Приложение разворачивает:
SECRET_ID=$(vault unwrap -field=secret_id $WRAPPED)
vault write auth/approle/login role_id=$ROLE_ID secret_id=$SECRET_ID
# WRAPPED больше нельзя использовать — перехват бесполезен
```

---

### Kubernetes Auth — Pod получает секреты без env vars

**Кейс: Spring Boot приложение без secret в манифесте**

```yaml
# Манифест полностью без секретов — только Service Account:
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "payments-service"
        vault.hashicorp.com/agent-inject-secret-application: "secret/data/payments-service/prod"
        vault.hashicorp.com/agent-inject-template-application: |
          {{- with secret "secret/data/payments-service/prod" -}}
          spring.datasource.url={{ .Data.data.db_url }}
          spring.datasource.password={{ .Data.data.db_password }}
          stripe.api.key={{ .Data.data.stripe_key }}
          {{- end }}
        vault.hashicorp.com/agent-inject-file-application: "application.properties"
    spec:
      serviceAccountName: payments-service-sa
      containers:
        - name: app
          image: payments-service:latest
          # Никаких env с секретами — приложение читает /vault/secrets/application.properties
```

```
git blame → ни в одном коммите нет пароля от БД.
kubectl get secret → нет K8s Secrets с credentials.
kubectl describe pod → нет env vars с секретами.
```

---

**Кейс: Init container подготавливает secrets для legacy-приложения**

Legacy-приложение читает secrets из файлов и не умеет в Vault SDK.

```yaml
initContainers:
  - name: vault-init
    image: hashicorp/vault:1.17
    command: ["/bin/sh", "-c"]
    args:
      - |
        vault login -method=kubernetes role=legacy-app \
          jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
        vault kv get -field=db_password secret/legacy/prod > /secrets/db_password
        vault kv get -field=api_key secret/legacy/prod > /secrets/api_key
    volumeMounts:
      - mountPath: /secrets
        name: secrets-vol
containers:
  - name: legacy-app
    volumeMounts:
      - mountPath: /app/secrets
        name: secrets-vol
        readOnly: true
volumes:
  - name: secrets-vol
    emptyDir:
      medium: Memory   # tmpfs — секреты не пишутся на диск
```

---

### Auto-Unseal — Операционные сценарии

**Кейс: Vault в Kubernetes рестартует без участия оператора**

```
Без auto-unseal:                    С auto-unseal (AWS KMS):
vault pod killed                    vault pod killed
  ↓                                   ↓
vault restarts → SEALED             vault restarts
  ↓                                   ↓ (автоматически)
PagerDuty → дежурный               vault unseal via KMS
  ↓ 2 AM                             ↓ ~5 секунд
оператор вводит 3 ключа             vault READY
  ↓ 15 минут даунтайм               нет даунтайма, никто не просыпался
vault READY
```

```bash
# Проверить статус unseal:
vault status | grep -E "Sealed|Unseal"
# Sealed: false
# Unseal Progress: 0/0  ← auto-unseal, ключи не нужны
```

**Recovery keys** (вместо unseal keys при auto-unseal) нужны только для:
- Regeneration root token при его потере
- Disaster recovery если KMS недоступен
- Rekey самих recovery ключей

---

### HA — Сценарий отказа

**Кейс: Потеря leader-ноды в продакшене**

```
Нормальная работа:
vault-0 (leader) ←→ vault-1 (standby) ←→ vault-2 (standby)

vault-0 падает:
vault-0 (DOWN)       vault-1 (standby)    vault-2 (standby)
                          ↓ Raft election (~10s)
                     vault-1 (NEW LEADER)  vault-2 (standby)

Приложения: ~10 секунд ошибок при election, потом автоматически работают через vault-1.
```

```bash
# Мониторинг смены лидера:
vault operator raft list-peers
# Node     Address                      State       Voter
# vault-0  vault-0.vault-internal:8201  follower    true  ← был лидером
# vault-1  vault-1.vault-internal:8201  leader      true  ← новый лидер
# vault-2  vault-2.vault-internal:8201  follower    true

# Health check endpoint (для load balancer):
# Возвращает 200 только для active (leader) ноды:
curl https://vault-0.vault-internal:8200/v1/sys/health
# {"initialized":true,"sealed":false,"standby":false,...}
# standby: false → это лидер
```

**Load balancer настройка:**

```yaml
# k8s Service — только активная нода принимает write:
apiVersion: v1
kind: Service
metadata:
  name: vault-active
spec:
  selector:
    app: vault
    vault-active: "true"   # label ставится Vault автоматически
  ports:
    - port: 8200
```

---

### Audit — Расследование инцидента

**Кейс: Кто и когда читал секрет**

Пришёл запрос от compliance: "Кто имел доступ к `secret/payments/stripe_key` за последние 30 дней?"

```bash
# Audit log — каждый запрос записан:
grep '"path":"secret/data/payments/stripe_key"' /vault/logs/audit.log \
  | jq -r '[.time, .auth.display_name, .auth.policies[], .request.operation] | @csv' \
  | sort

# Вывод:
# "2026-02-01T09:15:32Z","payments-service","payments-policy","read"
# "2026-02-15T14:22:11Z","john.doe@company","admin-policy","read"
# "2026-02-20T02:33:44Z","unknown-token","root","read"  ← подозрительно!
```

```bash
# Расследование подозрительного доступа:
# В 2 ночи, root token — кто это?
grep '"token":"hvs.CAESIAbCd"' /vault/logs/audit.log | jq .auth
# Найти откуда token, когда создан, отозвать:
vault token lookup hvs.CAESIAbCd
vault token revoke hvs.CAESIAbCd
```

---

## 📐 Архитектура — Типичный production setup

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌──────────────────────────────────────────┐              │
│  │            Vault HA (3 replicas)         │              │
│  │  vault-0 (leader) ←Raft→ vault-1,2      │              │
│  │  Auto-unseal: AWS KMS                   │              │
│  │  Storage: Integrated Raft 20Gi          │              │
│  │  Audit: file + syslog → SIEM            │              │
│  └──────────────┬───────────────────────────┘              │
│                 │                                           │
│        Vault Agent Injector                                │
│                 │                                           │
│  ┌──────────────┴────────────────────────┐                │
│  │    Microservices (via K8s Auth)       │                │
│  │                                       │                │
│  │  payments-svc → DB creds (1h TTL)    │                │
│  │  orders-svc → S3 creds (15m TTL)     │                │
│  │  api-gateway → TLS cert (24h TTL)    │                │
│  │  notifications → Stripe key (KV v2)  │                │
│  └───────────────────────────────────────┘                │
│                                                             │
│  ┌─────────────────────────────────────┐                  │
│  │    CI/CD (GitLab via OIDC/JWT)      │                  │
│  │  runner → AWS creds (15m) → deploy  │                  │
│  └─────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘

External:
  AWS KMS ← auto-unseal
  PostgreSQL ← dynamic users
  AWS IAM ← dynamic access keys
  Internal CA ← PKI root
```

