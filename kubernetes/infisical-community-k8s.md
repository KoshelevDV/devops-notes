# Infisical Community — внедрение секретов в Kubernetes

> Self-hosted Infisical (Community/OSS) для инжекта секретов в поды как env-переменные и файлы.
> Источники: исходный код github.com/Infisical/infisical, github.com/Infisical/kubernetes-operator,
> github.com/external-secrets/external-secrets. Версия: Infisical v0.158.0, Operator v0.10.29.

---

## Варианты деплоя Infisical в k8s

### Helm chart: infisical-standalone-postgres (рекомендуется для self-hosted)

```bash
helm repo add infisical-helm-charts 'https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/'
helm repo update
helm install infisical infisical-helm-charts/infisical-standalone-postgres \
  -n infisical --create-namespace \
  -f values.yaml
```

Включает в чарте:
- Infisical App (Node.js backend + React frontend)
- PostgreSQL (bitnami, через mirror.gcr.io)
- Redis (bitnami, через mirror.gcr.io)

### Обязательные env-переменные (минимальный набор)

Передаются через k8s Secret, ссылка в `infisical.kubeSecretRef`:

```yaml
ENCRYPTION_KEY: <openssl rand -hex 16>     # 16-байт hex
AUTH_SECRET: <openssl rand -base64 32>     # 32-байт base64
DB_CONNECTION_URI: postgresql://user:pass@host:5432/db
REDIS_URL: redis://:password@redis:6379
SITE_URL: https://infisical.example.com
```

### Ключевые параметры values.yaml

```yaml
infisical:
  enabled: true
  replicaCount: 2

  image:
    repository: infisical/infisical
    tag: "v0.158.0"
    pullPolicy: IfNotPresent

  kubeSecretRef: "infisical-secrets"   # k8s Secret с env-переменными

  extraEnv:
    - name: TELEMETRY_ENABLED
      value: "false"                   # отключение телеметрии
    - name: NODE_EXTRA_CA_CERTS
      value: /etc/ssl/certs/ca.crt    # для корп. CA

  resources:
    limits:
      memory: 1000Mi
    requests:
      cpu: 350m

  service:
    type: ClusterIP

ingress:
  enabled: true
  hostName: "infisical.example.com"
  ingressClassName: nginx
  tls:
    - secretName: infisical-tls
      hosts:
        - infisical.example.com

postgresql:
  enabled: true
  fullnameOverride: "postgresql"
  image:
    registry: mirror.gcr.io            # уже использует зеркало
    repository: bitnamilegacy/postgresql
  auth:
    username: infisical
    password: changeme
    database: infisicalDB

redis:
  enabled: true
  fullnameOverride: "redis"
  image:
    registry: mirror.gcr.io
    repository: bitnamilegacy/redis
  usePassword: true
  auth:
    password: "changeme"
  architecture: standalone

  databaseSchemaMigrationJob:
    image:
      repository: ghcr.io/groundnuty/k8s-wait-for
      tag: no-root-v2.0
```

---

## Ограничения Community (бесплатной) версии

Из исходников `backend/src/ee/services/license/license-fns.ts` — функция `getDefaultOnPremFeatures()`:

| Фича | Community | Enterprise |
|---|---|---|
| Версионирование секретов | ✅ | ✅ |
| Базовые секреты, проекты, окружения | ✅ | ✅ |
| Machine Identities (Universal/K8s/AWS/Azure/GCP auth) | ✅ | ✅ |
| Kubernetes Operator + Agent | ✅ | ✅ |
| **Rate limits** | 60 read/200 write/40 secrets per min | Настраиваемые |
| **RBAC** (ролевые права) | ❌ | ✅ |
| **Audit Logs** | ❌ | ✅ |
| **SAML SSO** | ❌ | ✅ |
| **OIDC SSO** | ❌ | ✅ |
| **LDAP** | ❌ | ✅ |
| **SCIM** | ❌ | ✅ |
| **Dynamic Secrets** | ❌ | ✅ |
| **Secret Rotation** | ❌ | ✅ |
| **Secret Approval Workflows** | ❌ | ✅ |
| **IP Allowlisting** | ❌ | ✅ |
| **PIT Recovery** (point-in-time) | ❌ | ✅ |
| **HSM** | ❌ | ✅ |
| **External KMS** | ❌ | ✅ |
| **FIPS** | ❌ | ✅ |
| **Secret Scanning** | ❌ | ✅ |
| **Enterprise Syncs/Connections** | ❌ | ✅ |
| Лимит identities | нет лимита | нет лимита |
| Лимит проектов/членов | нет лимита | нет лимита |

**Вывод:** Community подходит для базового хранения и инжекта секретов в k8s. Для аудит-логов, SSO и Secret Rotation нужна Enterprise.

---

## Телеметрия — отключение

Из `backend/src/lib/config/env.ts`:

```bash
# PostHog аналитика (пользовательская телеметрия)
TELEMETRY_ENABLED=false           # по умолчанию "true" — ОБЯЗАТЕЛЬНО отключить

# OpenTelemetry (операционная метрики, по умолчанию уже false)
OTEL_TELEMETRY_COLLECTION_ENABLED=false   # default: false

# Можно также переопределить (но при TELEMETRY_ENABLED=false не нужно):
POSTHOG_HOST=https://app.posthog.com
POSTHOG_PROJECT_API_KEY=...
```

В values.yaml через extraEnv:

```yaml
infisical:
  extraEnv:
    - name: TELEMETRY_ENABLED
      value: "false"
```

Или в k8s Secret `infisical-secrets`:

```yaml
stringData:
  TELEMETRY_ENABLED: "false"
  OTEL_TELEMETRY_COLLECTION_ENABLED: "false"
```

---

## Инжект секретов в поды

Есть три подхода:

### Подход 1: Infisical Kubernetes Operator (рекомендуется)

Оператор следит за `InfisicalSecret` CRD и создаёт нативные k8s Secrets. Потом секрет монтируется в под как обычный k8s Secret.

**Установка оператора:**

```bash
helm repo add infisical-helm-charts 'https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/'
helm install infisical-operator infisical-helm-charts/secrets-operator \
  -n infisical-operator --create-namespace
```

Образ: `infisical/kubernetes-operator:v0.10.29` (docker.io)

**Создание Machine Identity credentials:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: universal-auth-credentials
  namespace: default
type: Opaque
stringData:
  clientId: "машина-client-id"
  clientSecret: "машина-client-secret"
```

**InfisicalSecret CRD:**

```yaml
apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret
metadata:
  name: myapp-infisical-secret
  namespace: default
spec:
  hostAPI: https://infisical.example.com/api   # для self-hosted

  authentication:
    universalAuth:
      credentialsRef:
        secretName: universal-auth-credentials
        secretNamespace: default
      secretsScope:
        projectSlug: my-web-app    # Project Settings → Copy Project Slug
        envSlug: prod
        secretsPath: /
        recursive: false

  managedKubeSecretReferences:
    - secretName: myapp-k8s-secrets    # имя создаваемого k8s Secret
      secretNamespace: default
      secretType: Opaque
      creationPolicy: Owner            # "Owner" = удалится с CRD, "Orphan" = остаётся

  resyncInterval: 5m
```

**Kubernetes Auth (без credentials secret — через SA JWT):**

```yaml
authentication:
  kubernetesAuth:
    identityId: "machine-identity-id"
    serviceAccountRef:
      name: default
      namespace: default
    secretsScope:
      projectSlug: my-web-app
      envSlug: prod
      secretsPath: /
```

**Использование секрета в поде — как env:**

```yaml
containers:
  - name: myapp
    image: myapp:latest
    envFrom:
      - secretRef:
          name: myapp-k8s-secrets    # созданный оператором Secret
```

**Использование секрета в поде — как файл:**

```yaml
volumes:
  - name: secrets-vol
    secret:
      secretName: myapp-k8s-secrets

containers:
  - name: myapp
    volumeMounts:
      - name: secrets-vol
        mountPath: /etc/secrets
        readOnly: true
```

**Отдельные переменные из Secret:**

```yaml
containers:
  - name: myapp
    env:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: myapp-k8s-secrets
            key: DATABASE_URL
```

---

### Подход 2: Infisical Agent (sidecar/initContainer)

Agent читает YAML-конфиг, получает секреты из Infisical и рендерит Go-шаблоны в файлы на общий `emptyDir` том. Подходит когда нужна **горячая перезагрузка** без рестарта пода.

**Конфиг агента (ConfigMap):**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: infisical-agent-config
data:
  config.yaml: |
    infisical:
      address: "https://infisical.example.com"

    auth:
      type: "universal-auth"
      config:
        client-id: ""        # или env: INFISICAL_UNIVERSAL_AUTH_CLIENT_ID
        client-secret: ""    # или env: INFISICAL_UNIVERSAL_AUTH_CLIENT_SECRET

    sinks:
      - type: file
        config:
          path: /tmp/infisical-token   # токен доступа

    templates:
      - source-path: /agent-config/app.env.ctmpl
        destination-path: /etc/secrets/app.env
        config:
          polling-interval: 5m
          execute:
            command: "kill -HUP 1"    # сигнал основному процессу при обновлении
            timeout: 30

  app.env.ctmpl: |
    {{- with secret "my-project" "prod" "/" -}}
    DATABASE_URL={{ .Secrets.DATABASE_URL }}
    API_KEY={{ .Secrets.API_KEY }}
    {{- end }}
```

**Под с agent как initContainer:**

```yaml
spec:
  volumes:
    - name: secrets-vol
      emptyDir: {}
    - name: agent-config
      configMap:
        name: infisical-agent-config

  initContainers:
    - name: infisical-agent
      image: infisical/cli:latest
      command: ["infisical", "agent", "--config", "/agent-config/config.yaml"]
      env:
        - name: INFISICAL_UNIVERSAL_AUTH_CLIENT_ID
          valueFrom:
            secretKeyRef:
              name: universal-auth-credentials
              key: clientId
        - name: INFISICAL_UNIVERSAL_AUTH_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: universal-auth-credentials
              key: clientSecret
      volumeMounts:
        - name: secrets-vol
          mountPath: /etc/secrets
        - name: agent-config
          mountPath: /agent-config

  containers:
    - name: myapp
      image: myapp:latest
      volumeMounts:
        - name: secrets-vol
          mountPath: /etc/secrets
          readOnly: true
```

---

### Подход 3: External Secrets Operator (ESO) + Infisical provider

ESO поддерживает Infisical как провайдер (начиная с ESO v0.9+). Преимущество: стандартизированный интерфейс если уже используется ESO.

**Установка ESO:**

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

**Credentials Secret:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: universal-auth-credentials
  namespace: default
type: Opaque
stringData:
  clientId: "machine-identity-client-id"
  clientSecret: "machine-identity-client-secret"
```

**SecretStore:**

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: infisical
  namespace: default
spec:
  provider:
    infisical:
      hostAPI: https://infisical.example.com   # для self-hosted
      auth:
        universalAuthCredentials:
          clientId:
            name: universal-auth-credentials
            key: clientId
          clientSecret:
            name: universal-auth-credentials
            key: clientSecret
      secretsScope:
        projectSlug: my-project
        environmentSlug: prod
        secretsPath: /
        recursive: false
        expandSecretReferences: true

      # Для self-hosted с кастомным CA:
      caProvider:
        type: Secret
        name: infisical-ca-cert
        key: ca.crt
```

**ExternalSecret — все секреты из пути:**

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: default
spec:
  refreshInterval: 5m
  secretStoreRef:
    kind: SecretStore
    name: infisical
  target:
    name: myapp-k8s-secrets   # создаёт k8s Secret
    creationPolicy: Owner
  dataFrom:
    - find:
        name:
          regexp: .*           # все секреты
```

**ExternalSecret — фильтр по имени или пути:**

```yaml
dataFrom:
  - find:
      path: DB_              # только ключи начинающиеся с DB_
```

**ExternalSecret — конкретный секрет:**

```yaml
data:
  - secretKey: DATABASE_URL    # ключ в k8s Secret
    remoteRef:
      key: DATABASE_URL        # имя секрета в Infisical
```

---

## Сравнение подходов

| | Operator | Agent (sidecar) | ESO |
|---|---|---|---|
| **Как работает** | CRD → k8s Secret | sidecar → файл | CRD → k8s Secret |
| **Инжект как env** | ✅ (через envFrom) | ❌ (только файл) | ✅ (через envFrom) |
| **Инжект как файл** | ✅ (volume mount) | ✅ (нативно) | ✅ (volume mount) |
| **Горячее обновление** | нет (нужен rollout) | ✅ (execute hook) | нет (нужен rollout) |
| **Доп. зависимости** | только оператор | CLI image в поде | ESO + CRDs |
| **Нативный Infisical** | ✅ | ✅ | через ESO |
| **Стандартизация** | Infisical-специфично | Infisical-специфично | ✅ любой провайдер |

**Рекомендация:** Оператор — если только Infisical. ESO — если используются несколько провайдеров секретов.

---

## Деплой в закрытом контуре (air-gapped)

### Образы для зеркалирования в Harbor

| Компонент | Source | Образ |
|---|---|---|
| Infisical App | docker.io | `infisical/infisical:v0.158.0` |
| K8s Operator | docker.io | `infisical/kubernetes-operator:v0.10.29` |
| PostgreSQL | mirror.gcr.io | `bitnamilegacy/postgresql` (уже зеркало) |
| Redis | mirror.gcr.io | `bitnamilegacy/redis` (уже зеркало) |
| Migration job | ghcr.io | `groundnuty/k8s-wait-for:no-root-v2.0` |

> PostgreSQL и Redis в чарте уже используют `mirror.gcr.io` — не docker.io. Но в закрытом контуре нужно и этот Registry проксировать.

**Зеркалирование образов в Harbor:**

```bash
# Авторизация
docker login harbor.internal.example.com
docker login registry-1.docker.io  # на интернет-хосте

# Infisical App
docker pull infisical/infisical:v0.158.0
docker tag infisical/infisical:v0.158.0 harbor.internal.example.com/infisical/infisical:v0.158.0
docker push harbor.internal.example.com/infisical/infisical:v0.158.0

# K8s Operator
docker pull infisical/kubernetes-operator:v0.10.29
docker tag infisical/kubernetes-operator:v0.10.29 harbor.internal.example.com/infisical/kubernetes-operator:v0.10.29
docker push harbor.internal.example.com/infisical/kubernetes-operator:v0.10.29

# Migration job
docker pull ghcr.io/groundnuty/k8s-wait-for:no-root-v2.0
docker tag ghcr.io/groundnuty/k8s-wait-for:no-root-v2.0 harbor.internal.example.com/infisical/k8s-wait-for:no-root-v2.0
docker push harbor.internal.example.com/infisical/k8s-wait-for:no-root-v2.0

# PostgreSQL и Redis (через mirror.gcr.io)
docker pull mirror.gcr.io/bitnamilegacy/postgresql:16.x
docker tag mirror.gcr.io/bitnamilegacy/postgresql:16.x harbor.internal.example.com/infisical/postgresql:16.x
docker push harbor.internal.example.com/infisical/postgresql:16.x
```

**Harbor как Pull-through proxy (альтернатива зеркалированию вручную):**

В Harbor создать Proxy Cache проекты:
- `docker-hub-proxy` → endpoint: `https://registry-1.docker.io`
- `gcr-proxy` → endpoint: `https://mirror.gcr.io`
- `ghcr-proxy` → endpoint: `https://ghcr.io`

Тогда образы подтягиваются автоматически при первом pull через Harbor.

### Nexus как прокси Helm-репозитория

Официальный Helm repo Infisical: `https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/`

**Nexus OSS** поддерживает Helm через raw-репозиторий (не нативный Helm тип в OSS):

1. Создать Raw (Proxy) репозиторий в Nexus:
   - Type: `raw (proxy)`
   - Remote URL: `https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/`
   - Name: `infisical-helm-proxy`

2. Добавить в Helm:
```bash
helm repo add infisical http://nexus.internal.example.com/repository/infisical-helm-proxy/
helm repo update
```

**Альтернатива: Harbor OCI для Helm charts:**

```bash
# На интернет-хосте: скачать чарт
helm pull infisical-helm-charts/infisical-standalone-postgres --version 1.x.x

# Запушить в Harbor как OCI artifact
helm push infisical-standalone-postgres-*.tgz oci://harbor.internal.example.com/helm-charts/

# Установить из Harbor
helm install infisical oci://harbor.internal.example.com/helm-charts/infisical-standalone-postgres \
  --version 1.x.x \
  -n infisical --create-namespace \
  -f values-airgap.yaml
```

### values-airgap.yaml — переопределение образов

```yaml
infisical:
  image:
    repository: harbor.internal.example.com/infisical/infisical
    tag: "v0.158.0"
    pullPolicy: IfNotPresent
    imagePullSecrets:
      - name: harbor-pull-secret

  extraEnv:
    - name: TELEMETRY_ENABLED
      value: "false"
    - name: NODE_EXTRA_CA_CERTS
      value: /etc/ssl/certs/ca.crt

  databaseSchemaMigrationJob:
    image:
      repository: harbor.internal.example.com/infisical/k8s-wait-for
      tag: no-root-v2.0

postgresql:
  enabled: true
  fullnameOverride: "postgresql"
  image:
    registry: harbor.internal.example.com
    repository: infisical/postgresql
    tag: "16.x"
  auth:
    username: infisical
    password: changeme
    database: infisicalDB

redis:
  enabled: true
  fullnameOverride: "redis"
  image:
    registry: harbor.internal.example.com
    repository: infisical/redis
    tag: "7.x"
  usePassword: true
  auth:
    password: "changeme"
```

**imagePullSecret для Harbor:**

```bash
kubectl create secret docker-registry harbor-pull-secret \
  --docker-server=harbor.internal.example.com \
  --docker-username=robot\$infisical \
  --docker-password=<token> \
  -n infisical
```

---

## Ссылки

- Helm chart: https://github.com/Infisical/infisical/tree/main/helm-charts/infisical-standalone-postgres
- K8s Operator: https://github.com/Infisical/kubernetes-operator
- ESO provider: https://github.com/external-secrets/external-secrets/blob/main/docs/provider/infisical.md
- Env vars docs: https://infisical.com/docs/self-hosting/configuration/envars
- K8s deploy docs: https://infisical.com/docs/self-hosting/deployment-options/kubernetes-helm
