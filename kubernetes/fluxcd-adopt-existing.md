# FluxCD — перенос уже развёрнутых приложений под управление

> Как взять под управление Flux приложения, которые уже работают в кластере.
> Источник: официальная документация + исходники helm-controller (state.go, atomic_release.go).
> Flux v2 (helm-controller v2).

---

## Механика: что делает Flux при обнаружении существующего релиза

Helm-controller читает Helm storage (секреты `sh.helm.release.v1.*`) и сравнивает с состоянием HelmRelease.

Состояния релиза:

| Статус | Что означает |
|---|---|
| **Absent** | Релиза нет в Helm storage → Flux делает `helm install` |
| **Unmanaged** | Релиз есть в storage, но не управляется этим HelmRelease → Flux делает `helm upgrade` |
| **InSync** | Релиз совпадает с желаемым состоянием → ничего не делает |
| **OutOfSync** | Релиз есть, но values/chart отличаются → Flux делает `helm upgrade` |

**Ключевое:** при `ReleaseStatusUnmanaged` Flux выполняет `helm upgrade` (не uninstall+install), то есть **без даунтайма**.

---

## Случай 1: Приложение задеплоено через Helm

### Алгоритм

**1. Снять текущее состояние:**

```bash
# Точное имя релиза и версия чарта
helm list -n <namespace>

# Текущие values
helm get values <release-name> -n <namespace> -o yaml > current-values.yaml
```

**2. Создать HelmRepository + HelmRelease:**

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
  name: redis                    # имя объекта в k8s (любое)
  namespace: flux-system
spec:
  releaseName: redis             # КРИТИЧНО: должно совпадать с `helm list NAME`
  targetNamespace: redis         # namespace где работает приложение
  storageNamespace: redis        # namespace где хранятся helm secrets (обычно = targetNamespace)
  interval: 1h
  chart:
    spec:
      chart: redis
      version: ">=19.0.0"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  install:
    disableTakeOwnership: false  # по умолчанию false — Flux берёт ownership ресурсов
  upgrade:
    disableTakeOwnership: false
  driftDetection:
    mode: enabled
  values:
    # Вставить содержимое current-values.yaml
    auth:
      enabled: true
      existingSecret: redis-auth
```

**3. Применить и проверить:**

```bash
kubectl apply -f redis-helmrelease.yaml

# Следить за статусом
flux get helmreleases -A
kubectl describe helmrelease redis -n flux-system
```

### Главная ловушка — несовпадение releaseName

Имя по умолчанию в Flux: `[<targetNamespace>-]<name>` (имя HelmRelease объекта).

Если `spec.releaseName` не совпадёт с именем существующего релиза:
- Flux не найдёт релиз → статус `Absent`
- Flux попытается сделать `helm install`
- Упадёт с ошибкой: `cannot re-use a name that is still in use`

**Проверить имя существующего релиза:**

```bash
helm list -n redis
# NAME   NAMESPACE   REVISION   ...
# redis  redis       5          ...
```

### Проверить где хранится Helm storage:

```bash
kubectl get secrets -n redis | grep helm.release
# sh.helm.release.v1.redis.v5   helm.sh/release.v1   ...
```

Namespace этих секретов = `storageNamespace`.

---

## Случай 2: Приложение задеплоено через kubectl apply (plain manifests)

Kustomize-controller использует **Server-Side Apply (SSA)**. При применении ресурсов из Git становится их field manager.

### Нормальный сценарий

Если ресурс уже существует — SSA корректно мёрджит поля. Flux берёт ownership над полями из манифеста. Ошибок нет.

### Проблема: конфликт field managers

Если другой менеджер (kubectl apply, helm, оператор) уже владеет теми же полями — SSA вернёт конфликт.

**Решение 1 — временно форсировать на уровне Kustomization:**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  force: true    # временно! убрать после того как Flux взял ownership
  # ...
```

**Решение 2 — точечно на конкретный ресурс (аннотация):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    kustomize.toolkit.fluxcd.io/force: enabled  # убрать после применения
```

---

## Случай 3: Kafka через Strimzi (оператор + CRDs)

Strimzi — оператор + кастомные ресурсы (Kafka, KafkaTopic, KafkaUser, etc.).

### Шаг 1: Взять под управление Strimzi Operator

Используется HelmRelease как в Случае 1:

```yaml
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
  namespace: flux-system
spec:
  releaseName: strimzi-kafka-operator   # совпадает с helm list NAME
  targetNamespace: kafka
  storageNamespace: kafka
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
```

### Шаг 2: Взять под управление Kafka CRs

Экспортировать существующие ресурсы:

```bash
kubectl get kafka -n kafka -o yaml > kafka-cluster.yaml
kubectl get kafkatopic -n kafka -o yaml > kafka-topics.yaml
# и т.д. для KafkaUser, KafkaMirrorMaker2, etc.
```

Сохранить в Git репозиторий и создать Kustomization:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: kafka-cluster
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: "./infrastructure/apps/kafka"
  prune: true
  dependsOn:
    - name: strimzi-operator   # ждём оператора
```

Kafka ресурсы **не пересоздаются** — оператор продолжает управлять существующим кластером. Flux просто берёт ownership CR-ресурсов через SSA.

---

## Поле disableTakeOwnership

Есть у `spec.install` и `spec.upgrade`.

| Значение | Поведение |
|---|---|
| `false` (default) | Flux берёт ownership существующих k8s ресурсов, которыми управляет этот Helm chart |
| `true` | Flux НЕ берёт ownership — если ресурс уже существует и управляется другим менеджером, конфликта не будет, но и Flux не будет им полностью управлять |

Полезно `true` если: ресурс управляется одновременно оператором и Helm-чартом.

---

## Итого — чеклист переноса Helm-приложения

- [ ] `helm list -n <ns>` → узнать точное имя релиза
- [ ] `helm get values <name> -n <ns> -o yaml` → экспортировать values
- [ ] `kubectl get secrets -n <ns> | grep helm.release` → подтвердить storageNamespace
- [ ] Создать HelmRelease с явным `spec.releaseName` (совпадает с helm list)
- [ ] Вставить values из экспорта в `spec.values`
- [ ] Применить → Flux обнаружит `Unmanaged`, сделает upgrade, даунтайма нет
- [ ] `flux get helmreleases -A` → проверить статус Ready
