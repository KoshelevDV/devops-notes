# Секреты в CI/CD, контейнерах и подах — подходы и best practices

> **Дата:** 2026-02-28  
> **Теги:** `secrets` `cicd` `kubernetes` `docker` `vault` `security`

---

## 🎯 Проблема

Секреты (токены, пароли, ключи) нужно как-то доставить в pipeline, контейнер или pod — так, чтобы они:
- не попали в git-историю
- не светились в образе (docker layers)
- не были доступны всем подряд в кластере
- ротировались без пересборки образа

---

## ✅ Подходы по уровням

---

### 1. CI/CD — GitHub Actions

**Базовый подход — GitHub Secrets:**

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    steps:
      - name: Deploy
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          API_KEY: ${{ secrets.API_KEY }}
        run: ./deploy.sh
```

Добавить секрет: Settings → Secrets and variables → Actions → New repository secret.

**Для разных окружений — Environments:**

```yaml
jobs:
  deploy:
    environment: production   # привязан к набору secrets
    steps:
      - run: echo ${{ secrets.PROD_DB_URL }}
```

**OIDC — без долгоживущих токенов (рекомендуется для cloud):**

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/github-actions
      aws-region: us-east-1
  # Теперь AWS credentials получены через временный OIDC-токен
```

> Принцип: GitHub Actions получает OIDC JWT → AWS/GCP/Azure обменивает его на временные credentials. Долгосрочных ключей нет вообще.

---

### 2. CI/CD — GitLab CI

```yaml
# .gitlab-ci.yml
deploy:
  script:
    - echo $DB_PASSWORD   # переменная из Settings → CI/CD → Variables
  environment: production
```

**Masked + Protected переменные:**
- **Masked** — значение скрывается в логах
- **Protected** — доступна только в protected branches/tags

```yaml
variables:
  DB_URL: $DB_URL   # Определена в GitLab UI, не в коде
```

**GitLab CI + Vault (HashiCorp):**

```yaml
variables:
  VAULT_ADDR: https://vault.example.com
  VAULT_AUTH_ROLE: gitlab-ci

script:
  - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login
      role=$VAULT_AUTH_ROLE jwt=$CI_JOB_JWT)
  - export SECRET=$(vault kv get -field=password secret/myapp/db)
```

---

### 3. Docker / Docker Compose

❌ **Никогда не делать:**

```dockerfile
# ПЛОХО — секрет навсегда останется в слое образа
ENV DB_PASSWORD=supersecret
RUN curl -H "Authorization: Bearer hardcoded_token" ...
```

```bash
# ПЛОХО — пароль видно в docker inspect и ps aux
docker run -e DB_PASSWORD=secret myapp
```

✅ **Правильно — env из файла:**

```bash
# .env (в .gitignore!)
DB_PASSWORD=supersecret

docker run --env-file .env myapp
```

✅ **Docker Secrets (только Docker Swarm):**

```bash
echo "mysecret" | docker secret create db_password -

# В docker-compose.yml (swarm mode):
services:
  app:
    secrets:
      - db_password
secrets:
  db_password:
    external: true
```

Секрет монтируется как файл `/run/secrets/db_password` — не как env var.

✅ **BuildKit secrets для сборки (не попадают в слои):**

```dockerfile
# Dockerfile
RUN --mount=type=secret,id=npmrc cat /run/secrets/npmrc > ~/.npmrc \
    && npm install \
    && rm ~/.npmrc
```

```bash
docker build --secret id=npmrc,src=.npmrc .
```

---

### 4. Kubernetes Secrets

**Базовый Secret (base64, не encrypted at rest по умолчанию):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
stringData:         # stringData — plaintext, автоматически base64
  db-password: supersecret
  api-key: abc123
```

```yaml
# Использование в Pod
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: myapp-secret
        key: db-password

# Или монтировать как файл (предпочтительно для SSL/certs):
volumes:
  - name: secret-vol
    secret:
      secretName: myapp-secret
volumeMounts:
  - mountPath: /etc/secrets
    name: secret-vol
    readOnly: true
```

**Encryption at rest (обязательно включить!):**

```yaml
# kube-apiserver --encryption-provider-config=/etc/k8s/encryption.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

**RBAC — ограничить доступ к secrets:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["myapp-secret"]   # только конкретный secret!
    verbs: ["get"]
```

---

### 5. External Secrets Operator (ESO) — рекомендуется для продакшена

ESO синхронизирует секреты из внешних хранилищ в Kubernetes Secrets.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
```

```yaml
# SecretStore — откуда брать секреты
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: https://vault.example.com
      path: secret
      auth:
        kubernetes:
          mountPath: kubernetes
          role: myapp

---
# ExternalSecret — что именно синхронизировать
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-secret       # результирующий k8s Secret
  data:
    - secretKey: db-password
      remoteRef:
        key: myapp/production
        property: db_password
```

Поддерживаемые бэкенды: HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, 1Password, Doppler, и др.

---

### 6. HashiCorp Vault

**Kubernetes Auth:**

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
    kubernetes_host=https://k8s-api:6443 \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault write auth/kubernetes/role/myapp \
    bound_service_account_names=myapp-sa \
    bound_service_account_namespaces=production \
    policies=myapp-policy \
    ttl=1h
```

**Vault Agent Sidecar (inject secrets в pod):**

```yaml
# Аннотации на Pod/Deployment
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp"
  vault.hashicorp.com/agent-inject-secret-db: "myapp/production"
  vault.hashicorp.com/agent-inject-template-db: |
    {{- with secret "myapp/production" -}}
    DB_PASSWORD={{ .Data.data.db_password }}
    {{- end }}
```

Секрет появляется как файл `/vault/secrets/db` внутри контейнера.

---

## 💡 Подводные камни

- **ENV vars видны через `docker inspect`, `kubectl describe pod`, `/proc/1/environ`** — для чувствительных данных лучше файлы в `tmpfs`/`/run/secrets`
- **Base64 в k8s Secret — не шифрование** — без encryption at rest любой с доступом к etcd читает секреты
- **`kubectl get secret -o yaml` — выводит base64** — ограничивать RBAC обязательно, смотреть на `audit logs`
- **Секреты в переменных окружения наследуются дочерними процессами** — и могут утечь через crash dumps, логи
- **Docker build args (`--build-arg SECRET=...`) попадают в историю образа** — использовать только BuildKit `--secret`
- **git-история не прощает** — если секрет попал в коммит, считать его скомпрометированным и ротировать, `git filter-repo` не поможет если уже запушили
- **Short-lived tokens** > долгоживущих ключей — OIDC, Vault dynamic secrets, Workload Identity

---

## 🏆 Best Practices — коротко

| Что | Как |
|-----|-----|
| CI/CD токены | OIDC вместо долгосрочных ключей |
| Docker build | BuildKit `--secret`, не ARG |
| Docker run | `--env-file` из файла вне репо |
| k8s базовый | Secret + encryption at rest + RBAC |
| k8s продакшен | External Secrets Operator + Vault/AWS SM |
| Ротация | ESO refreshInterval или Vault lease TTL |
| Аудит | audit log на `get secrets`, alerts на массовый доступ |
| .gitignore | `.env`, `*.pem`, `*_secret*`, `kubeconfig` |

---

## 📎 Документация

- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes Encryption at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [External Secrets Operator](https://external-secrets.io/latest/)
- [HashiCorp Vault — K8s Auth](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
- [GitHub OIDC](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Docker BuildKit Secrets](https://docs.docker.com/build/building/secrets/)

---

## 🔗 Источники

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Kubernetes Hardening Guide (NSA/CISA)](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
