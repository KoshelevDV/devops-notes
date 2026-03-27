# FluxCD — развёртывание infra layer в Kubernetes

> GitOps-подход для управления инфраструктурным слоем k8s: Helm charts, манифесты, зависимости, секреты, мониторинг.
> Версия документации: Flux v2 (latest stable). Источник: https://fluxcd.io/flux/

---

## Компоненты Flux

| Контроллер | Назначение |
|---|---|
| **source-controller** | Следит за Git репо, HelmRepository, OCI Registry |
| **kustomize-controller** | Применяет Kustomize overlays и plain YAML-манифесты |
| **helm-controller** | Управляет HelmRelease (install/upgrade/rollback/drift detection) |
| **notification-controller** | Алерты, webhook receivers |
| **image-automation-controller** | Автообновление image tags (опционально) |

---

## Установка и bootstrap

### Установка CLI

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
# Completions
. <(flux completion bash)
```

### Bootstrap с GitLab

```bash
export GITLAB_TOKEN=<gl-pat>

# Self-hosted GitLab
flux bootstrap gitlab \
  --token-auth \
  --hostname=gitlab.example.com \
  --owner=my-group \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/prod

# GitLab с deploy token (рекомендуется для CI/CD)
flux bootstrap gitlab \
  --deploy-token-auth \
  --owner=my-group/my-subgroup \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/prod
```

**PAT требует:** full read/write access to GitLab API.
Bootstrap идемпотентен — можно запускать повторно для upgrade Flux.

### Bootstrap с GitHub

```bash
export GITHUB_TOKEN=<gh-token>

flux bootstrap github \
  --owner=my-org \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/prod \
  --personal
```

---

## Структура репозитория

```
fleet-infra/
├── clusters/
│   └── prod/
│       ├── flux-system/          # Flux controllers (auto-generated)
│       └── infra.yaml            # Kustomization entry point
├── infrastructure/
│   ├── namespaces/               # Namespaces
│   ├── sources/                  # HelmRepository, GitRepository
│   ├── controllers/              # metallb, nginx-ingress, cert-manager
│   └── apps/                     # kafka, redis, postgresql
└── apps/
    └── prod/                     # Application Helm releases
```

---

## Sources — откуда брать чарты

### HTTP/S Helm репозиторий (публичный)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 1h
  url: https://kubernetes.github.io/ingress-nginx
```

### HTTP/S Helm репозиторий с авторизацией (приватный)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: private-charts
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.example.com
  secretRef:
    name: private-charts-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: private-charts-auth
  namespace: flux-system
stringData:
  username: my-user
  password: my-password
```

### OCI Registry (рекомендуется для современных чартов)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m
  url: oci://ghcr.io/stefanprodan/charts/podinfo
  layerSelector:
    mediaType: "application/vnd.cncf.helm.chart.content.v1.tar+gzip"
    operation: copy
  ref:
    semver: "^6.9.0"
```

### OCI Registry с авторизацией (GitLab Registry)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  url: oci://registry.gitlab.example.com/my-group/my-charts/my-app
  ref:
    semver: ">=1.0.0"
  secretRef:
    name: gitlab-registry-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-registry-auth
  namespace: flux-system
stringData:
  username: deploy-token-user
  password: deploy-token-value
```

> Секрет для OCI/Docker Registry должен быть типа `kubernetes.io/dockerconfigjson`.
> Создать: `kubectl create secret docker-registry gitlab-registry-auth --docker-server=... --docker-username=... --docker-password=...`

### Локальный Helm чарт из Git репозитория

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet-infra
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.example.com/my-group/fleet-infra
  ref:
    branch: main
  secretRef:
    name: fleet-infra-auth   # для приватного репо
---
# HelmRelease ссылается на путь в GitRepository
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: apps
spec:
  interval: 10m
  chart:
    spec:
      chart: ./helm/my-app       # путь внутри репо
      sourceRef:
        kind: GitRepository
        name: fleet-infra
        namespace: flux-system
```

---

## HelmRelease — установка чартов

### Чарт из HelmRepository (публичный)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 1h
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x.x"          # semver range
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  driftDetection:
    mode: enabled
  values:
    controller:
      replicaCount: 2
      service:
        type: LoadBalancer
```

### Чарт из OCIRepository

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 10m
  chartRef:                     # ссылка на существующий OCIRepository
    kind: OCIRepository
    name: podinfo
    namespace: flux-system
  values:
    replicaCount: 2
```

### values из ConfigMap или Secret

```yaml
spec:
  valuesFrom:
    - kind: ConfigMap
      name: my-app-values
      valuesKey: values.yaml    # ключ в ConfigMap (по умолчанию values.yaml)
    - kind: Secret
      name: my-app-secrets
      valuesKey: values.yaml
      optional: true
```

---

## Kustomization — plain манифесты

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: namespaces
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: "./infrastructure/namespaces"
  prune: true                   # удалять ресурсы, пропавшие из Git
  targetNamespace: ""           # применять в namespace, указанном в манифестах
  timeout: 2m
```

---

## Зависимости — dependsOn

`dependsOn` есть у **Kustomization** и **HelmRelease**. Flux ждёт Ready статуса у зависимости перед применением.

### Пример: metallb → nginx-ingress → apps

```yaml
# 1. metallb — без зависимостей
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: metallb
  namespace: flux-system
spec:
  interval: 1h
  path: ./infrastructure/controllers/metallb
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  prune: true

---
# 2. nginx-ingress — ждёт metallb
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  dependsOn:
    - name: metallb
      namespace: flux-system
  # ...

---
# 3. Apps — ждут ingress-nginx
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: ingress-nginx
      namespace: flux-system
  # ...
```

### Как избежать каскада зависимостей

**Проблема:** A → B → C → D → ... — при падении A весь стек заморожен.

**Решения:**

1. **Группировать уровни, а не отдельные компоненты:**
   ```
   namespaces (нет зависимостей)
       ↓
   infrastructure-controllers (metallb, cert-manager — параллельно)
       ↓
   infrastructure-configs (ingress, storage-class — один уровень)
       ↓
   apps (параллельно друг другу)
   ```

2. **healthChecks** — Flux ждёт реальной готовности ресурса, не просто apply:
   ```yaml
   spec:
     healthChecks:
       - apiVersion: apps/v1
         kind: Deployment
         name: metallb-controller
         namespace: metallb-system
     healthCheckTimeout: 3m
   ```

3. **Минимум цепочек** — dependsOn только на компоненты, которые реально нужны для старта (не "на всякий случай").

4. **Suspend/Resume** — при проблемах изолируй компонент: `flux suspend kustomization metallb`

---

## Примеры infra layer

### Namespaces

```yaml
# infrastructure/namespaces/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - metallb-system.yaml
  - ingress-nginx.yaml
  - kafka.yaml
  - redis.yaml
  - postgresql.yaml
```

```yaml
# infrastructure/namespaces/metallb-system.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
```

### MetalLB (HelmRelease)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: metallb
  namespace: flux-system
spec:
  interval: 1h
  url: https://metallb.github.io/metallb
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: metallb
  namespace: metallb-system
spec:
  interval: 1h
  dependsOn:
    - name: namespaces
      namespace: flux-system
  chart:
    spec:
      chart: metallb
      version: ">=0.14.0"
      sourceRef:
        kind: HelmRepository
        name: metallb
        namespace: flux-system
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace
  values:
    {}
```

### nginx-ingress (HelmRelease)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 1h
  dependsOn:
    - name: metallb
      namespace: flux-system
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x.x"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  values:
    controller:
      replicaCount: 2
```

### Kafka — Strimzi (operator + cluster)

```yaml
# Шаг 1: установить Strimzi operator
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: strimzi
  namespace: flux-system
spec:
  interval: 1h
  url: https://strimzi.io/charts/
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: strimzi-operator
  namespace: kafka
spec:
  interval: 1h
  chart:
    spec:
      chart: strimzi-kafka-operator
      version: ">=0.42.0"
      sourceRef:
        kind: HelmRepository
        name: strimzi
        namespace: flux-system
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace

---
# Шаг 2: Kafka cluster (Kustomization с plain manifest)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: kafka-cluster
  namespace: flux-system
spec:
  dependsOn:
    - name: strimzi-operator
      namespace: flux-system   # ждём готовности operator
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./infrastructure/apps/kafka
  prune: true
```

### Redis (Bitnami)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: redis
  namespace: redis
spec:
  interval: 1h
  chart:
    spec:
      chart: redis
      version: ">=20.0.0"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    architecture: standalone
    auth:
      enabled: true
      existingSecret: redis-auth
      existingSecretPasswordKey: password
```

### CloudNativePG (оператор PostgreSQL)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: cnpg
  namespace: flux-system
spec:
  interval: 1h
  url: https://cloudnative-pg.github.io/charts
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cnpg
  namespace: cnpg-system
spec:
  interval: 1h
  chart:
    spec:
      chart: cloudnative-pg
      version: ">=0.23.0"
      sourceRef:
        kind: HelmRepository
        name: cnpg
        namespace: flux-system
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace
```

---

## Секреты

### SOPS + age (рекомендуется)

```bash
# 1. Генерируем age key
age-keygen -o age.agekey
# Public key: age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg

# 2. Создаём Secret в кластере
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin

# 3. Шифруем секрет
sops --age=age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg \
  --encrypt --encrypted-regex '^(data|stringData)$' --in-place my-secret.yaml

# 4. Коммитим зашифрованный файл — в Git хранить безопасно
```

```yaml
# .sops.yaml — конфиг для автоматического шифрования
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg
```

```yaml
# Kustomization с расшифровкой
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-secrets
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./infrastructure/secrets
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### Распространение секрета во все namespace

Kubernetes не копирует секреты между namespace нативно. Варианты:

**1. External Secrets Operator (ESO) + ClusterSecretStore** — рекомендуется:
```yaml
# ClusterSecretStore читает из Vault/AWS SM/GitLab CI Variable
apiVersion: external-secrets.io/v1beta1
kind: ClusterExternalSecret
metadata:
  name: registry-auth
spec:
  namespaceSelector:
    matchLabels:
      inject-registry-secret: "true"   # метка на namespace
  externalSecretSpec:
    refreshInterval: 1h
    secretStoreRef:
      name: vault-backend
      kind: ClusterSecretStore
    target:
      name: registry-auth
    data:
      - secretKey: .dockerconfigjson
        remoteRef:
          key: secret/registry
          property: dockerconfig
```

**2. Reflector** (emberstack/kubernetes-reflector) — отражает секрет во все namespace:
```yaml
# На исходном секрете — аннотации
apiVersion: v1
kind: Secret
metadata:
  name: registry-auth
  namespace: flux-system
  annotations:
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
    reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: ""  # все
```

**3. SOPS Kustomization на каждый namespace** — декларативно, но требует дублирования.

---

## Интеграция с Vault (через External Secrets Operator)

FluxCD не имеет нативной интеграции с Vault. Стандартный подход — ESO.

```yaml
# Установка ESO через Flux
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: external-secrets
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.external-secrets.io
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-secrets
  namespace: external-secrets
spec:
  interval: 1h
  chart:
    spec:
      chart: external-secrets
      version: ">=0.10.0"
      sourceRef:
        kind: HelmRepository
        name: external-secrets
        namespace: flux-system
  install:
    crds: CreateReplace
```

```yaml
# ClusterSecretStore → Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "flux-reader"
```

---

## Bootstrap с GitLab CI — автоматический деплой

```yaml
# .gitlab-ci.yml — bootstrap Flux в CD pipeline
deploy:
  stage: deploy
  image: ghcr.io/fluxcd/flux-cli:latest
  script:
    - flux bootstrap gitlab
        --token-auth
        --hostname=$GITLAB_HOST
        --owner=$CI_PROJECT_NAMESPACE
        --repository=fleet-infra
        --branch=main
        --path=clusters/prod
  environment:
    name: production
```

---

## Мониторинг

### Prometheus метрики (порт 8080, /metrics)

Ключевые метрики контроллеров:

```
# Время reconciliation по типу ресурса
gotk_reconcile_duration_seconds_bucket{kind, name, namespace, le}
gotk_reconcile_duration_seconds_sum{kind, name, namespace}

# Состояние ресурсов (через kube-state-metrics)
gotk_resource_info{customresource_kind, exported_namespace, name, ready, suspended}

# Для HelmRelease — дополнительные labels:
# revision, chart_name, chart_app_version, chart_source_name

# CPU/Memory контроллеров
process_cpu_seconds_total{namespace, pod}
container_memory_working_set_bytes{namespace, pod}
```

### Настройка мониторинга

```bash
# Готовый пример от Flux team (kube-prometheus-stack + Loki + Grafana дашборды)
git clone https://github.com/fluxcd/flux2-monitoring-example
```

Включает:
- kube-prometheus-stack
- kube-state-metrics с CustomResourceState для Flux CRDs
- Loki для логов контроллеров
- Готовые Grafana дашборды

### CLI мониторинг

```bash
# Статус всех Flux ресурсов
flux get all -A

# Статус конкретных типов
flux get kustomizations -A
flux get helmreleases -A
flux get sources all -A

# Логи контроллера
flux logs --follow --level=error
flux logs -n flux-system --kind=HelmRelease --name=ingress-nginx

# Принудительная reconciliation
flux reconcile kustomization apps --with-source
flux reconcile helmrelease ingress-nginx -n ingress-nginx

# Приостановить/возобновить
flux suspend helmrelease ingress-nginx -n ingress-nginx
flux resume helmrelease ingress-nginx -n ingress-nginx
```

### Алерты через notification-controller

```yaml
# Отправка алертов в Slack
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack-bot
  namespace: flux-system
spec:
  type: slack
  channel: flux-alerts
  secretRef:
    name: slack-url
---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-call-webapp
  namespace: flux-system
spec:
  summary: "Flux reconciliation alert"
  providerRef:
    name: slack-bot
  eventSeverity: error
  eventSources:
    - kind: GitRepository
      name: "*"
    - kind: Kustomization
      name: "*"
    - kind: HelmRelease
      name: "*"
```

---

## Drift Detection и исправление

HelmRelease умеет детектировать и исправлять drift (кто-то вручную изменил ресурс в кластере):

```yaml
spec:
  driftDetection:
    mode: enabled          # enabled | warn | disabled
    ignore:
      - paths: ["/spec/replicas"]    # игнорировать HPA-управляемые поля
        target:
          kind: Deployment
```

---

## Полезные команды

```bash
# Проверить преконфигурацию перед push
flux diff kustomization apps --path ./apps/prod

# Export всех ресурсов
flux export kustomization --all > backup.yaml

# Tree зависимостей
flux tree kustomization flux-system

# Проверить health cluster
flux check

# Статус bootstrap
flux get all -n flux-system
```

---

---

## Air-gap / закрытый контур (Nexus / Harbor)

> Источник: https://fluxcd.io/flux/installation/configuration/air-gapped/

Flux полностью поддерживает установку в изолированном контуре — нужны только:
- приватный Git (GitLab self-hosted, Gitea и т.д.)
- приватный container registry (Harbor, Nexus Docker hosted)
- приватный Helm registry (Nexus Helm hosted/proxy, Harbor)

### Шаг 1 — скопировать образы Flux в Harbor / Nexus

На машине с доступом в интернет (jump host):

```bash
# Посмотреть какие образы нужны
flux install --export | grep ghcr.io

# Скопировать все образы в приватный registry
FLUX_CONTROLLERS=(
  "source-controller"
  "kustomize-controller"
  "helm-controller"
  "notification-controller"
  "image-reflector-controller"
  "image-automation-controller"
)

for controller in "${FLUX_CONTROLLERS[@]}"; do
  crane copy --all-tags \
    ghcr.io/fluxcd/$controller \
    harbor.internal.example.com/fluxcd/$controller
done
```

`crane` — утилита от Google: https://github.com/google/go-containerregistry

Альтернатива без crane:
```bash
for controller in "${FLUX_CONTROLLERS[@]}"; do
  docker pull ghcr.io/fluxcd/$controller:latest
  docker tag ghcr.io/fluxcd/$controller:latest harbor.internal.example.com/fluxcd/$controller:latest
  docker push harbor.internal.example.com/fluxcd/$controller:latest
done
```

### Шаг 2 — bootstrap с приватным registry

**Harbor без авторизации (публичный проект):**
```bash
flux bootstrap git \
  --registry=harbor.internal.example.com/fluxcd \
  --url=ssh://git@gitlab.internal.example.com/platform/fleet-infra \
  --branch=main \
  --private-key-file=~/.ssh/flux_ed25519 \
  --path=clusters/prod
```

**Harbor с авторизацией (приватный проект):**
```bash
flux bootstrap git \
  --registry=harbor.internal.example.com/fluxcd \
  --registry-creds=robot$flux:harbor-robot-token \
  --image-pull-secret=harbor-regcred \
  --url=ssh://git@gitlab.internal.example.com/platform/fleet-infra \
  --branch=main \
  --private-key-file=~/.ssh/flux_ed25519 \
  --path=clusters/prod
```

Flux создаёт Secret `harbor-regcred` в `flux-system` namespace автоматически и прописывает его в podах контроллеров.

Обновить credentials позже:
```bash
flux -n flux-system create secret oci harbor-regcred \
  --username=robot\$flux \
  --password=new-token
```

### Шаг 3 — HelmRepository через Nexus / Harbor прокси

**Nexus как Helm proxy** (проксирует внешние репозитории):
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 1h
  url: https://nexus.internal.example.com/repository/helm-proxy/
  secretRef:
    name: nexus-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: nexus-auth
  namespace: flux-system
stringData:
  username: flux-reader
  password: nexus-password
```

**Nexus Helm hosted** (локальные чарты, загруженные вручную):
```yaml
spec:
  url: https://nexus.internal.example.com/repository/helm-hosted/
```

**Harbor как Helm registry (OCI):**
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-charts
  namespace: flux-system
spec:
  interval: 1h
  type: oci
  url: oci://harbor.internal.example.com/helm-charts
  secretRef:
    name: harbor-auth
---
apiVersion: v1
kind: Secret  # тип dockerconfigjson для OCI
metadata:
  name: harbor-auth
  namespace: flux-system
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "harbor.internal.example.com": {
          "username": "robot$flux",
          "password": "harbor-robot-token"
        }
      }
    }
```

Или создать через kubectl:
```bash
kubectl create secret docker-registry harbor-auth \
  --namespace=flux-system \
  --docker-server=harbor.internal.example.com \
  --docker-username=robot\$flux \
  --docker-password=harbor-robot-token
```

### Загрузка Helm чартов в Nexus / Harbor

**Nexus hosted:**
```bash
# Скачать чарт снаружи
helm pull ingress-nginx/ingress-nginx --version 4.10.0

# Загрузить в Nexus
curl -u user:pass \
  --upload-file ingress-nginx-4.10.0.tgz \
  https://nexus.internal.example.com/repository/helm-hosted/
```

**Harbor OCI:**
```bash
helm pull ingress-nginx/ingress-nginx --version 4.10.0
helm push ingress-nginx-4.10.0.tgz oci://harbor.internal.example.com/helm-charts
```

### Nexus как Docker proxy для образов приложений

В закрытом контуре приложения тоже тянут образы из Harbor/Nexus:
- Nexus: создать **Docker proxy** репозиторий (проксирует docker.io, ghcr.io, quay.io)
- Harbor: создать **proxy cache** проект для каждого внешнего registry

Настроить containerd на нодах:
```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://harbor.internal.example.com/v2/dockerhub-proxy"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."ghcr.io"]
      endpoint = ["https://harbor.internal.example.com/v2/ghcr-proxy"]
```

### Bootstrap с self-signed сертификатом

Если Nexus/Harbor используют внутренний CA:
```bash
# Добавить CA cert в flux-system
kubectl create secret generic ca-cert \
  --namespace=flux-system \
  --from-file=ca.crt=./internal-ca.crt

# В HelmRepository указать caSecretRef
spec:
  caSecretRef:
    name: ca-cert
```

Или через `insecure: true` (только для dev, не для prod):
```yaml
spec:
  insecure: true
```

### Итоговая схема для закрытого контура

```
[Интернет] → (периодически) → [Jump host / CI] → Harbor/Nexus
                                                        │
                                               ┌────────┴────────┐
                                               │                 │
                                          Docker images     Helm charts
                                               │                 │
                                    [Закрытый контур]            │
                                               │                 │
                                          k8s nodes          Flux HelmRepository
                                               │
                                    Flux controllers (из Harbor)
                                               │
                                    GitLab self-hosted (fleet-infra repo)
```

---

## Multi-cluster: dev / stage / preprod / prod

> Источник: https://fluxcd.io/flux/guides/repository-structure/

### Подходы к организации репозитория

#### 1. Monorepo (рекомендуется для большинства случаев)

Все кластеры в одном репозитории, один branch (`main`). Разница между окружениями — через Kustomize overlays.

```
fleet-infra/
├── apps/
│   ├── base/             # общие HelmRelease, общие values
│   ├── dev/              # patch: version >=1.0.0-alpha, replicas: 1
│   ├── staging/          # patch: version >=1.0.0-alpha, replicas: 2
│   ├── preprod/          # patch: version >=1.0.0, replicas: 2
│   └── production/       # patch: version >=1.0.0, replicas: 3, HPA
├── infrastructure/
│   ├── base/             # общие контроллеры
│   ├── controllers/      # cert-manager, ingress-nginx, metallb
│   └── configs/          # cluster-issuers, NetworkPolicy, StorageClass
└── clusters/
    ├── dev/              # flux bootstrap path → кластер dev
    ├── staging/
    ├── preprod/
    └── production/
```

**clusters/production/flux-system/** — автогенерируется bootstrap.
**clusters/production/infrastructure.yaml** и **clusters/production/apps.yaml** — создаёшь сам:

```yaml
# clusters/production/infrastructure.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/controllers
  prune: true
  wait: true                  # ждать готовности всех ресурсов

---
# clusters/production/apps.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production     # <-- env-специфичный overlay
  prune: true
```

**Bootstrap для каждого кластера** (разные kubeconfig):
```bash
# dev
flux bootstrap gitlab \
  --kubeconfig=~/.kube/dev.yaml \
  --hostname=gitlab.example.com \
  --owner=platform \
  --repository=fleet-infra \
  --path=clusters/dev \
  --branch=main

# production
flux bootstrap gitlab \
  --kubeconfig=~/.kube/prod.yaml \
  --hostname=gitlab.example.com \
  --owner=platform \
  --repository=fleet-infra \
  --path=clusters/production \
  --branch=main
```

Каждый кластер смотрит только в свой `path=clusters/<env>`, но черпает конфиги из общего `apps/` и `infrastructure/`.

---

#### 2. Repo per environment

Отдельный Git репозиторий на каждое окружение.

```
fleet-infra-dev/        → только dev
fleet-infra-staging/    → только staging
fleet-infra-prod/       → только production (ограниченный доступ)
```

**Когда выбирать:**
- Нужны разные права доступа (разработчики видят dev/staging, но не prod)
- Production — очень чувствительный, хочется изолированный review процесс
- Команды разные по каждому окружению

**Минусы:**
- Продвижение изменений между окружениями нужно делать вручную (или через CI)
- Дублирование infrastructure кода

---

#### 3. Repo per team (platform + tenant модель)

Platform team управляет инфраструктурой, dev-команды — своими приложениями.

```
# Platform repo
fleet-infra/
├── infrastructure/
│   ├── base/
│   ├── production/
│   └── staging/
├── clusters/
│   ├── production/
│   └── staging/
└── teams/
    ├── team-a/           # Kustomization → app-team-a repo
    └── team-b/

# App team repo (team-a/apps)
apps/
├── base/
├── production/
└── staging/
```

Platform team регистрирует репозиторий команды в Flux:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: team-a-apps
  namespace: team-a
spec:
  interval: 1m
  url: https://gitlab.example.com/team-a/apps
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-a-apps
  namespace: team-a
spec:
  serviceAccountName: team-a         # RBAC ограничение — только свой namespace
  sourceRef:
    kind: GitRepository
    name: team-a-apps
  path: ./production
  prune: true
```

---

### Kustomize overlays — как работает патчинг

**base** содержит общий HelmRelease, **overlay** переопределяет только нужные поля:

```yaml
# apps/base/myapp/release.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: myapp
spec:
  chart:
    spec:
      chart: myapp
      sourceRef:
        kind: HelmRepository
        name: my-charts
        namespace: flux-system
  values:
    replicaCount: 1
    image:
      tag: latest
```

```yaml
# apps/production/myapp-patch.yaml — только diff!
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: myapp
spec:
  chart:
    spec:
      version: ">=1.0.0"       # только stable
  values:
    replicaCount: 3
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
    ingress:
      hosts:
        - host: myapp.prod.example.com
```

```yaml
# apps/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base/myapp
patches:
  - path: myapp-patch.yaml
```

---

### Promotion: продвижение между окружениями

**Вариант 1 — ручной через PR (рекомендуется для prod):**
1. Merge в `main` → автодеплой в dev/staging
2. После тестирования — PR в ветку `release/prod`, review, merge → деплой в prod

**Вариант 2 — Image Automation (автоматический для dev/staging):**
```yaml
# Flux следит за новыми image tags в registry
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: registry.example.com/myapp
  interval: 1m
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: ">=1.0.0-0"      # для staging: alpha/beta тоже
---
# В HelmRelease values — маркер для автообновления
# image: registry.example.com/myapp:1.2.3 # {"$imagepolicy": "flux-system:myapp"}
```

**Вариант 3 — GitLab CI pipeline продвигает через commit:**
```yaml
# .gitlab-ci.yml
promote-to-staging:
  stage: promote
  script:
    - git clone $FLEET_INFRA_REPO
    - sed -i "s/tag: .*/tag: $CI_COMMIT_TAG/" fleet-infra/apps/staging/values-patch.yaml
    - git commit -am "promote myapp $CI_COMMIT_TAG to staging"
    - git push
  only:
    - tags
```

---

### Secrets per environment

Каждый кластер имеет **свой SOPS ключ** — секреты не перекрёстно расшифруемы:

```
.sops.yaml
creation_rules:
  - path_regex: clusters/dev/.*
    age: age1dev...
  - path_regex: clusters/staging/.*
    age: age1staging...
  - path_regex: clusters/production/.*
    age: age1prod...
```

Или через Vault с разными role per cluster:
```yaml
# ClusterSecretStore для dev
spec:
  provider:
    vault:
      auth:
        kubernetes:
          role: "flux-dev"          # права только на dev secrets path

# ClusterSecretStore для prod
spec:
  provider:
    vault:
      auth:
        kubernetes:
          role: "flux-prod"         # права только на prod secrets path
```

---

### Multi-tenancy lockdown (опционально, для строгой изоляции)

При bootstrap можно запретить cross-namespace refs и remote bases — каждый tenant видит только свои ресурсы:

```yaml
# clusters/production/flux-system/kustomization.yaml
patches:
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --no-cross-namespace-refs=true
    target:
      kind: Deployment
      name: "(kustomize-controller|helm-controller|notification-controller)"
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --no-remote-bases=true
    target:
      kind: Deployment
      name: "kustomize-controller"
```

---

### Типичная схема для 4 окружений

```
                    Git push → main
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
        dev             staging         preprod
    (autoдеплой)      (автодеплой)    (автодеплой)
                                          │
                                    PR review + approve
                                          │
                                          ▼
                                       prod
                                   (только после
                                    ручного merge)
```

**Что меняется между окружениями через overlays:**
- `version` semver range в HelmRelease (alpha/beta для staging, только stable для prod)
- `replicaCount` — меньше на dev, больше на prod
- `resources` — limits/requests
- `ingress.hosts` — домены
- `values` специфичные (feature flags, connection strings через secrets)

**Что одинаково для всех (base):**
- Сами чарты (HelmRepository sources)
- Структура HelmRelease (chart name, sourceRef)
- Базовые labels и аннотации

---

### Полезные команды для multi-cluster

```bash
# Статус конкретного кластера
flux get all -A --kubeconfig=~/.kube/prod.yaml

# Сравнить что изменится (dry-run)
flux diff kustomization apps --path ./apps/production --kubeconfig=~/.kube/prod.yaml

# Принудительная reconciliation в prod
flux reconcile kustomization apps \
  --with-source \
  --kubeconfig=~/.kube/prod.yaml

# Проверить health перед promotion
flux check --kubeconfig=~/.kube/staging.yaml
```

---

## Ссылки

- Docs: https://fluxcd.io/flux/
- Bootstrap GitLab: https://fluxcd.io/flux/installation/bootstrap/gitlab/
- HelmRelease API: https://fluxcd.io/flux/components/helm/helmreleases/
- Kustomization API: https://fluxcd.io/flux/components/kustomize/kustomizations/
- SOPS guide: https://fluxcd.io/flux/guides/mozilla-sops/
- Monitoring: https://fluxcd.io/flux/monitoring/metrics/
- Monitoring example: https://github.com/fluxcd/flux2-monitoring-example
- Helm releases guide: https://fluxcd.io/flux/guides/helmreleases/
