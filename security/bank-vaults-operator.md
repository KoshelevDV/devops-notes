# Bank-Vaults — Kubernetes Operator для HashiCorp Vault

> **Дата:** 2026-02-28  
> **Теги:** `vault` `kubernetes` `operator` `secrets` `bank-vaults` `webhook` `unseal`  
> **Источник:** https://bank-vaults.dev

---

## 🎯 Что такое bank-vaults

**Bank-Vaults** (изначально Banzai Cloud, теперь CNCF sandbox project) — набор инструментов поверх HashiCorp Vault, которые автоматизируют операционную нагрузку:

- **Vault Operator** — Kubernetes-оператор, управляет lifecycle Vault-кластера (init, unseal, конфиг)
- **Vault Secrets Webhook** — mutating webhook: инжектирует секреты напрямую в поды как env vars без K8s Secrets
- **bank-vaults CLI** — CLI для init/unseal/конфига вне оператора
- **Vault Go SDK** — обёртка над официальным клиентом с автообновлением токена

**Ключевое отличие от официального `hashicorp/vault` Helm chart:**  
Официальный чарт просто деплоит Vault + Vault Agent Injector.  
Bank-Vaults добавляет **оператор**, который декларативно управляет конфигурацией Vault через CRD (secret engines, auth methods, policies) и полностью автоматизирует unseal.

---

## 📦 Компоненты

| Компонент | Назначение | Репозиторий |
|-----------|-----------|-------------|
| `vault-operator` | Управляет Vault CRD, init/unseal, реконфигурация | `bank-vaults/vault-operator` |
| `vault-secrets-webhook` | Mutating webhook — inject secrets в Pod env vars | `bank-vaults/vault-secrets-webhook` |
| `bank-vaults` CLI | Init/unseal/configure без оператора | `bank-vaults/bank-vaults` |
| `vault-env` | Sidecar/init, подменяет env vars из Vault | `bank-vaults/bank-vaults` |
| Go SDK | Vault client с автообновлением токена | `bank-vaults/bank-vaults` (pkg) |

### Совместимость версий

| Operator | bank-vaults CLI | Vault |
|----------|----------------|-------|
| 1.21.x | >= 1.20.3 | 1.11.x – 1.14.x |
| 1.20.x | >= 1.19.0 | 1.10.x – 1.13.x |

---

## 🚀 Установка

### Вариант 1 — Только Vault Operator (минимум)

Разворачивает Vault как CustomResource. Operator сам делает init, unseal (через K8s Secrets для dev или KMS для prod), и применяет конфиг.

```bash
# 1. Установить оператор
helm upgrade --install --wait vault-operator \
  oci://ghcr.io/bank-vaults/helm-charts/vault-operator \
  --namespace vault-system \
  --create-namespace

# 2. RBAC для Vault
kubectl kustomize https://github.com/bank-vaults/vault-operator/deploy/rbac \
  | kubectl apply -f -

# 3. Создать Vault instance (single node, dev-режим unseal)
kubectl apply -f https://raw.githubusercontent.com/bank-vaults/vault-operator/v1.21.0/deploy/examples/cr-raft.yaml

# 4. Проверить
kubectl get pods
# vault-0                           3/3     Running   0
# vault-configurer-xxxx-xxxx        1/1     Running   0
# vault-operator-xxxx-xxxx          1/1     Running   0

# 5. Получить root token (только для dev/test!)
kubectl get secrets vault-unseal-keys -o jsonpath={.data.vault-root} | base64 --decode
```

### Вариант 2 — Operator + Secrets Webhook (full stack)

Добавляет инжекцию секретов напрямую в поды без K8s Secrets.

```bash
# Webhook в отдельном namespace
kubectl create namespace vault-infra
kubectl label namespace vault-infra name=vault-infra

helm upgrade --install --wait vault-secrets-webhook \
  oci://ghcr.io/bank-vaults/helm-charts/vault-secrets-webhook \
  --namespace vault-infra
```

### Вариант 3 — bank-vaults CLI без оператора

Если нужно управлять существующим Vault (не через оператор):

```bash
# Бинарь со страницы releases:
# https://github.com/bank-vaults/bank-vaults/releases

# Или через go:
go get github.com/bank-vaults/bank-vaults/cmd/bank-vaults

# Init + unseal + конфиг из файла:
bank-vaults unseal --mode kubernetes \
  --k8s-secret-namespace default \
  --k8s-secret-name vault-unseal-keys
```

---

## ⚙️ Vault Custom Resource — конфигурация

Главная суперсила bank-vaults: вся конфигурация Vault (auth methods, secret engines, policies) описывается в одном YAML как CR `Vault`. Оператор применяет её и **реконфигурирует при изменениях**.

### Минимальный CR (dev/test, unseal через K8s Secrets)

```yaml
apiVersion: vault.banzaicloud.com/v1alpha1
kind: Vault
metadata:
  name: vault
spec:
  size: 1
  image: hashicorp/vault:1.17.0

  # Конфиг Vault (hcl-стиль в YAML)
  config:
    storage:
      file:
        path: /vault/file
    listener:
      tcp:
        address: "0.0.0.0:8200"
        tls_disable: true
    ui: true
    api_addr: http://vault.default:8200

  # Unseal: хранить ключи в K8s Secret (только dev/test!)
  unsealConfig:
    kubernetes:
      secretNamespace: default
      secretName: vault-unseal-keys

  # Декларативная конфигурация Vault
  externalConfig:
    policies:
      - name: allow_secrets
        rules: |
          path "secret/*" {
            capabilities = ["create", "read", "update", "delete", "list"]
          }

    auth:
      - type: kubernetes
        roles:
          - name: default
            bound_service_account_names: ["default"]
            bound_service_account_namespaces: ["default"]
            policies: allow_secrets
            ttl: 1h

    secrets:
      - path: secret
        type: kv
        description: KV v2
        options:
          version: "2"
```

### Production CR (HA Raft + AWS KMS auto-unseal)

```yaml
apiVersion: vault.banzaicloud.com/v1alpha1
kind: Vault
metadata:
  name: vault
  namespace: vault
spec:
  size: 3   # 3 ноды (Raft кворум)
  image: hashicorp/vault:1.17.0

  resources:
    vault:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 512Mi

  config:
    storage:
      raft:
        path: /vault/data
        retry_join:
          - leader_api_addr: https://vault-0.vault-internal:8200
          - leader_api_addr: https://vault-1.vault-internal:8200
          - leader_api_addr: https://vault-2.vault-internal:8200
    listener:
      tcp:
        address: "0.0.0.0:8200"
        tls_cert_file: /vault/tls/server.crt
        tls_key_file: /vault/tls/server.key
    api_addr: https://vault.vault:8200
    cluster_addr: https://${.Env.POD_NAME}.vault-internal:8201
    ui: true

  # Auto-unseal через AWS KMS
  unsealConfig:
    aws:
      kmsKeyId: alias/vault-unseal
      kmsRegion: eu-central-1

  # Anti-affinity — разные ноды кластера
  podAntiAffinity: failure-domain.beta.kubernetes.io/zone

  # TLS — сертификат из cert-manager или существующий Secret
  existingTlsSecretName: vault-tls

  # Полная конфигурация Vault через оператор
  externalConfig:
    policies:
      - name: myapp
        rules: |
          path "secret/data/myapp/*" { capabilities = ["read"] }
          path "database/creds/myapp-role" { capabilities = ["read"] }

    auth:
      - type: kubernetes
        roles:
          - name: myapp
            bound_service_account_names: [myapp-sa]
            bound_service_account_namespaces: [production]
            policies: myapp
            ttl: 1h
      - type: approle
        roles:
          - name: ci-deploy
            policies: [deploy-policy]
            secret_id_ttl: 10m
            secret_id_num_uses: 1

    secrets:
      - path: secret
        type: kv
        options:
          version: "2"
      - path: database
        type: database
        configuration:
          config:
            - name: mypostgres
              plugin_name: postgresql-database-plugin
              connection_url: "postgresql://{{username}}:{{password}}@db:5432/mydb"
              allowed_roles: [myapp-role]
              username: vault
              password: vaultpassword
          roles:
            - name: myapp-role
              db_name: mypostgres
              creation_statements: "CREATE ROLE ..."
              default_ttl: 1h
              max_ttl: 24h
```

---

## 🔓 Опции Auto-Unseal (unsealConfig)

Одно из ключевых преимуществ bank-vaults — богатый выбор бэкендов для хранения unseal-ключей:

| Бэкенд | Поле | Когда использовать |
|--------|------|--------------------|
| **Kubernetes Secret** | `kubernetes:` | Dev/test. Ключи в K8s Secret |
| **AWS KMS** | `aws:` | Prod на AWS |
| **GCP KMS** | `google:` | Prod на GCP |
| **Azure Key Vault** | `azure:` | Prod на Azure |
| **Alibaba KMS** | `alibaba:` | Prod на Alibaba Cloud |
| **Vault Transit** | `vault:` | Vault-in-Vault (один Vault unseals другой) |
| **HSM** | `hsm:` | Hardware Security Module (Enterprise grade) |

```yaml
# Kubernetes Secret (dev)
unsealConfig:
  kubernetes:
    secretNamespace: vault
    secretName: vault-unseal-keys

# AWS KMS (prod)
unsealConfig:
  aws:
    kmsKeyId: alias/vault-unseal
    kmsRegion: eu-central-1
    # Если нет IAM Role — можно указать credentials через EnvVars

# GCP KMS (prod)
unsealConfig:
  google:
    kmsKeyRing: vault-keyring
    kmsCryptoKey: vault-unseal
    kmsLocation: global
    kmsProject: my-gcp-project

# Azure Key Vault (prod)
unsealConfig:
  azure:
    keyVaultName: my-key-vault
    keyName: vault-unseal

# Vault Transit (prod в изолированной среде)
unsealConfig:
  vault:
    address: https://core-vault:8200
    token: s.xxxx
    keyPath: transit/keys/child-vault-unseal
```

---

## 💉 Vault Secrets Webhook — инжекция без K8s Secrets

Webhook мутирует Pod при создании: вставляет `vault-env` как init container (или заменяет entrypoint), который перехватывает переменные формата `vault:path#key` и подставляет реальные значения **до старта приложения**.

Секреты никогда не попадают в etcd и не сохраняются на диск.

### Использование в Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        vault.security.banzaicloud.io/vault-addr: "https://vault.vault:8200"
        vault.security.banzaicloud.io/vault-role: "myapp"
        vault.security.banzaicloud.io/vault-path: "kubernetes"
        vault.security.banzaicloud.io/vault-tls-secret: "vault-tls"
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: app
          image: myapp:latest
          env:
            # Webhook подставит реальное значение из Vault
            - name: DB_PASSWORD
              value: "vault:secret/data/myapp/prod#db_password"
            - name: API_KEY
              value: "vault:secret/data/myapp/prod#api_key"
            - name: DB_USER
              value: "vault:database/creds/myapp-role#username"
```

### Через K8s Secret (прозрачно для legacy-приложений)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
data:
  # base64("vault:secret/data/myapp/prod#db_password")
  DB_PASSWORD: dmF1bHQ6c2VjcmV0L2RhdGEvbXlhcHAvcHJvZCNkYl9wYXNzd29yZA==
type: Opaque
```

Webhook перехватит Secret и подставит реальный секрет из Vault.

---

## 🔄 Bank-Vaults vs HashiCorp Vault Agent Injector

| Фича | Bank-Vaults Webhook | HashiCorp Agent Injector |
|------|--------------------|-----------------------|
| **Метод** | Заменяет entrypoint (vault-env) | Sidecar container |
| **Overhead** | Нет постоянного sidecar | Постоянный sidecar (memory, CPU) |
| **Секреты в etcd** | Никогда | Никогда |
| **Secret rotation** | Нет (при старте) | Да (agent следит за TTL) |
| **Daemon mode** | Опционально | Всегда |
| **Конфиг** | Env vars / аннотации | Аннотации + templates |
| **Декларативный конфиг Vault** | ✅ (externalConfig в CR) | ❌ (нужен отдельный инструмент) |
| **Auto-unseal** | ✅ встроен в operator | ❌ нужен отдельный механизм |
| **Multi-cloud unseal** | ✅ AWS/GCP/Azure/K8s | ❌ нужен отдельный механизм |

**Вывод:** Bank-Vaults лучше когда нужен декларативный IaC-подход + zero-overhead инжекция.  
HashiCorp Agent нужен если важен live secret rotation без рестарта пода.

---

## 📋 externalConfig — декларативный конфиг Vault

Оператор применяет и **поддерживает в актуальном состоянии** следующие ресурсы Vault:

```yaml
externalConfig:
  # Policies
  policies:
    - name: my-policy
      rules: |
        path "secret/data/myapp/*" {
          capabilities = ["read", "list"]
        }

  # Auth Methods
  auth:
    - type: kubernetes
      roles:
        - name: myapp
          bound_service_account_names: [myapp-sa]
          bound_service_account_namespaces: [production]
          policies: my-policy
          ttl: 1h
    - type: approle
      roles:
        - name: ci
          policies: [ci-policy]
    - type: jwt
      config:
        oidc_discovery_url: https://token.actions.githubusercontent.com

  # Secrets Engines
  secrets:
    - path: secret
      type: kv
      options:
        version: "2"
    - path: database
      type: database
    - path: pki
      type: pki
      configuration:
        config:
          - name: urls
            issuing_certificates: "https://vault.vault:8200/v1/pki/ca"
    - path: transit
      type: transit
    - path: aws
      type: aws
    - path: ssh
      type: ssh

  # Groups и Group Aliases (для identity engine)
  groups:
    - name: sre-team
      policies: [sre-policy]
      type: internal
```

Если изменить CR — оператор **автоматически** обновит конфиг в Vault (reconcile loop каждые 30s).

---

## 🔍 Мониторинг и диагностика

```bash
# Статус Vault CR
kubectl describe vault vault -n vault

# Логи оператора
kubectl logs deployment/vault-operator -n vault-system

# Логи vault-configurer (применяет externalConfig)
kubectl logs deployment/vault-configurer -n vault

# Unseal-ключи (только dev/test!)
kubectl get secret vault-unseal-keys -o json | jq -r '.data | map_values(@base64d)'
```

---

## ⚠️ Подводные камни

- **Backup не включён** — оператор не делает snapshot Raft автоматически. Использовать [Velero](https://bank-vaults.dev/docs/operator/backup/) или CronJob с `vault operator raft snapshot save`.
- **Kubernetes Secret для unseal — только dev** — в продакшене unseal-ключи в K8s Secret — это плохо: получив доступ к etcd, злоумышленник получает ключи от Vault.
- **Webhook daemon mode** — по умолчанию webhook не следит за обновлением секретов (inject при старте). Если нужен live rotation — включать `vault.security.banzaicloud.io/vault-agent: "true"` (ставит sidecar).
- **Версии** — compatibility matrix строгая, не ставить operator 1.21 с vault < 1.11.
- **TLS обязателен в prod** — в CR явно указывать `existingTlsSecretName` или использовать cert-manager.
- **externalConfig reconcile** — если вручную изменить что-то в Vault (через CLI), оператор **перезапишет** изменение. Всё через CR.
- **Удаление CR-ресурсов** — если удалить вручную ресурс созданный оператором (Ingress, Service), он пересоздастся через ~30s.

---

## 🆚 Когда использовать bank-vaults vs официальный Helm chart

**Используй bank-vaults если:**
- Нужен декларативный GitOps-подход к конфигурации Vault (secrets engines, policies через YAML)
- Хочешь минимальный overhead от инжекции секретов (без постоянного sidecar)
- Нужна унификация unseal-бэкендов из одного места
- Команда небольшая, хочется меньше ручных операций с Vault

**Используй официальный HashiCorp чарт если:**
- Нужен live rotation секретов без рестарта подов (Agent template + renew)
- Используешь HCP Vault (HashiCorp Cloud) — есть интеграции
- Уже настроен Agent Injector, менять не хочется
- Нужен Vault Proxy mode

---

## 📎 Ссылки

- [Bank-Vaults docs](https://bank-vaults.dev/docs/)
- [vault-operator Helm chart values](https://github.com/bank-vaults/vault-operator/tree/main/deploy/charts/vault-operator#values)
- [Vault CR examples](https://github.com/bank-vaults/vault-operator/tree/main/deploy/examples)
- [Webhook configuration](https://bank-vaults.dev/docs/mutating-webhook/configuration/)
- [Cloud permissions required](https://bank-vaults.dev/docs/concepts/cloud-permissions/)
- [Backup with Velero](https://bank-vaults.dev/docs/operator/backup/)
