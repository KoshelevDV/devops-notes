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
