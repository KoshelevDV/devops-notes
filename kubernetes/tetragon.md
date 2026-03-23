# Tetragon — eBPF Security Observability & Runtime Enforcement

> Cilium Tetragon — инструмент наблюдения за безопасностью в реальном времени на базе eBPF с поддержкой принудительного исполнения политик. Работает на уровне ядра Linux, понимает Kubernetes-идентити.

GitHub: https://github.com/cilium/tetragon

---

## Что такое Tetragon

- eBPF-агент, работающий на каждой ноде Kubernetes
- Видит syscalls, сетевые соединения, файловые операции **до** того, как userspace успел что-то сделать
- Понимает k8s-контекст: знает не просто "pid 1234", а "pod `payment-service`, namespace `prod`"
- Умеет **наблюдать** (observe) и **блокировать** (enforce) через `TracingPolicy`

---

## Ключевые возможности

| Возможность | Описание |
|---|---|
| Process Execution | Отслеживание запуска процессов с аргументами |
| File Access | Мониторинг чтения/записи файлов |
| Network | Отслеживание TCP/UDP соединений |
| Syscall tracing | Перехват системных вызовов |
| Runtime enforcement | Блокировка через `TracingPolicy` (SIGKILL и др.) |

---

## Интеграция с Kubernetes

- Понимает Pod, Namespace, Labels — привязывает события к k8s-объектам
- Нативно интегрируется с **Cilium CNI** (если уже стоит — добавляется почти бесплатно)
- Экспортирует события в JSON → можно слать в SIEM, Loki, Elasticsearch

---

## Установка (Helm)

```bash
helm repo add cilium https://helm.cilium.io
helm repo update

helm install tetragon cilium/tetragon \
  --namespace kube-system
```

Проверка:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=tetragon
```

---

## TracingPolicy — пример блокировки

Запретить запись в `/etc/passwd`:

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-etc-passwd-write
spec:
  kprobes:
  - call: "sys_openat"
    syscall: true
    args:
    - index: 1
      type: "string"
    selectors:
    - matchArgs:
      - index: 1
        operator: "Prefix"
        values:
        - "/etc/passwd"
      matchActions:
      - action: Sigkill
```

---

## Наблюдение событий

```bash
# CLI утилита tetra
kubectl exec -n kube-system ds/tetragon -- tetra getevents -o compact

# Фильтр по namespace
kubectl exec -n kube-system ds/tetragon -- tetra getevents \
  --namespace production -o compact
```

---

## Экспорт в SIEM

Tetragon пишет JSON-события в `/var/run/cilium/tetragon/tetragon.log` на каждой ноде.
Можно подключить Fluentd/Fluent Bit → Loki/Elasticsearch.

---

## Когда использовать

- Runtime security в production k8s кластерах
- Детектирование lateral movement, privilege escalation
- Compliance: аудит файлового доступа и сетевых соединений
- В связке с Cilium CNI для полного security observability стека

---

## Ссылки

- GitHub: https://github.com/cilium/tetragon
- Docs: https://tetragon.io/docs/
- Helm chart: https://artifacthub.io/packages/helm/cilium/tetragon
