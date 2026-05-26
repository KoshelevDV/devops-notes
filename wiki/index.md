# Knowledge Base Index

## ai

AI-агенты, RAG, автоматизация

| Article | Summary | Updated |
|---------|---------|---------|
| [Автономные AI-агенты — архитектура и применение](ai/autonomous-agents.md) | — | 2026-03-20 |
| [Автоматизация ревью документации через AI](ai/doc-review-automation.md) | — | 2026-03-20 |
| [Краулер-индексатор репозиториев GitHub/GitLab](ai/repo-crawler-indexer.md) | — | 2026-03-20 |

## api

API-справочники

| Article | Summary | Updated |
|---------|---------|---------|
| [Яндекс Трекер и Вики — API справочник](api/yandex-tracker-wiki-api.md) | — | 2026-03-20 |

## apps

Инструменты и приложения

| Article | Summary | Updated |
|---------|---------|---------|
| [ClawFeed — AI-powered новостной дайджест](apps/clawfeed.md) | — | 2026-04-06 |
| [Coroot — eBPF Observability Platform](apps/coroot.md) | — | 2026-03-28 |
| [ELMA365 On-Premises — документация и эксплуатация](apps/elma365.md) | — | 2026-03-20 |

## ci-cd

CI/CD пайплайны, деплой

| Article | Summary | Updated |
|---------|---------|---------|
| [[ARCHIVED] Автоматический цикл разработки: GitLab + OpenClaw](ci-cd/openclaw-auto-dev-cycle.md) | — | 2026-04-06 |

## databases

PostgreSQL, бэкапы, миграции, HA

| Article | Summary | Updated |
|---------|---------|---------|
| [PostgreSQL: Бэкап, Восстановление, Миграция, Синхронизация](databases/postgres-backup-restore-migration.md) | — | 2026-03-28 |
| [HA PostgreSQL: etcd + Patroni + HAProxy + keepalived](databases/postgresql-patroni-ha.md) | — | 2026-03-26 |

## docker

Образы, compose, registry

| Article | Summary | Updated |
|---------|---------|---------|
| [AnythingLLM — self-hosted база знаний (RAG)](docker/anythingllm-rag.md) | — | 2026-03-11 |

## linux

Команды, systemd, сеть, производительность

| Article | Summary | Updated |
|---------|---------|---------|
| [Cron — шпаргалка](linux/cron-cheatsheet.md) | — | 2026-03-28 |
| [Linux: очистка места — логи, кэши, Docker, без поломок](linux/disk-cleanup.md) | — | 2026-02-26 |
| [envsubst — замена переменных окружения в файлах](linux/envsubst.md) | — | 2026-03-28 |
| [[ARCHIVED] Векторная память (RAG) для OpenClaw с bge-m3 и llama-server](linux/openclaw-vector-memory-rag.md) | — | 2026-04-06 |
| [Векторная память (RAG) для AI-агента с BGE-M3 и OpenViking](linux/vector-memory-rag.md) | — | 2026-04-06 |

## monitoring

Prometheus, Grafana, Pushgateway, алерты

| Article | Summary | Updated |
|---------|---------|---------|
| [Prometheus Pushgateway](monitoring/prometheus-pushgateway.md) | — | 2026-05-25 |
| [VictoriaMetrics Cluster — архитектура](monitoring/victoria-metrics-cluster-architecture.md) | RBT: внешний VM-кластер, компоненты, потоки данных | 2026-05-26 |
| [VictoriaMetrics — литьё метрик из автотестов](monitoring/victoria-metrics-autotest-push.md) | Как пушить web-vitals через `/api/v1/import/prometheus` | 2026-05-26 |
| [Мониторинг серверов — железо и виртуалки](monitoring/server-monitoring-baremetal-vm.md) | — | 2026-04-07 |

## networking

Сети, DNS, firewall, VPN

| Article | Summary | Updated |
|---------|---------|---------|
| [Разделение типов доступа к B2E: магазин vs удалёнка](networking/b2e-store-vs-remote-access.md) | — | 2026-02-24 |
| [BGP Monitoring через BMP (BGP Monitoring Protocol)](networking/bgp-monitoring-bmp.md) | — | 2026-03-26 |
| [Headscale + Tailscale — Self-hosted Mesh VPN](networking/headscale-tailscale.md) | — | 2026-03-29 |
| [Meshtastic — децентрализованная mesh-сеть на LoRa](networking/meshtastic.md) | — | 2026-04-02 |
| [MTProto Proxy с Fake TLS](networking/mtproto-proxy.md) | — | 2026-03-17 |

## nginx

Конфиги nginx, proxy, hardening

| Article | Summary | Updated |
|---------|---------|---------|
| [Nginx: визуализация и разбор спагетти конфигов](nginx/config-visualization.md) | — | 2026-02-26 |
| [Nginx: наследование proxy_set_header и приоритеты директив](nginx/proxy-set-header-inheritance.md) | — | 2026-02-26 |
| [Nginx: безопасность конфигов и приложений за ним](nginx/security-hardening.md) | — | 2026-02-28 |

## kubernetes

K8s, etcd, ingress, helm, операторы

| Article | Summary | Updated |
|---------|---------|---------|
| [Botkube + Policy Reporter — алерты о деплоях и Kyverno-политиках](kubernetes/botkube-policy-reporter.md) | — | 2026-02-25 |
| [Distributed Tracing: от 100% error rate до первопричины за 60 секунд](kubernetes/distributed-tracing-uptrace.md) | — | 2026-03-22 |
| [FluxCD — перенос уже развёрнутых приложений под управление](kubernetes/fluxcd-adopt-existing.md) | — | 2026-04-07 |
| [FluxCD — развёртывание infra layer в Kubernetes](kubernetes/fluxcd-infra-layer.md) | — | 2026-03-27 |
| [Infisical Community — внедрение секретов в Kubernetes](kubernetes/infisical-community-k8s.md) | — | 2026-04-07 |
| [Infrastructure Observability Stack — мониторинг, алертинг, аудит, обновление образов](kubernetes/infra-observability-stack.md) | — | 2026-03-02 |
| [Kubernetes Audit Logging — централизованный аудит событий](kubernetes/kubernetes-audit-logging.md) | — | 2026-02-25 |
| [Kyverno — Policy Engine для Kubernetes](kubernetes/kyverno-policy-engine.md) | — | 2026-02-25 |
| [Логирование в Kubernetes: VictoriaLogs стек](kubernetes/logging-victorialogs-stack.md) | — | 2026-03-25 |
| [Tetragon — eBPF Security Observability & Runtime Enforcement](kubernetes/tetragon.md) | — | 2026-03-23 |

## security

CVE, hardening, секреты, аудит, AppSec

| Article | Summary | Updated |
|---------|---------|---------|
| [Управление правами доступа — PostgreSQL, MongoDB, ClickHouse, Kubernetes, Linux VM, AD/LDAP](security/access-control-rbac.md) | — | 2026-04-07 |
| [AppSec Platform Stack](security/appsec-platform-stack.md) | — | 2026-03-26 |
| [CVE-сканирование нативно установленных приложений](security/cve-scanning-native-apps.md) | — | 2026-02-25 |
| [CVE-сканирование развёрнутого ПО — Docker Compose, ВМ, Kubernetes](security/cve-scanning.md) | — | 2026-04-06 |
| [Сканирование безопасности фронтенда (Next.js / SPA)](security/frontend-security-scanning.md) | — | 2026-02-25 |
| [ФСТЭК Приказ №117 — Защита контейнерных сред (ЗКС)](security/fstec-117-container-security.md) | — | 2026-04-02 |
| [GitHub Actions — Hardening против injection и supply chain атак](security/github-actions-hardening.md) | — | 2026-03-20 |
| [Hoop — Access Gateway / PAM](security/hoop-access-gateway.md) | — | 2026-04-06 |
| [Хардинг Linux-сервера](security/linux-server-hardening.md) | — | 2026-03-02 |
| [Secret Scanning — предотвращение утечек токенов](security/secret-scanning.md) | — | 2026-03-10 |
| [Секреты в CI/CD, контейнерах и подах — подходы и best practices](security/secrets-management-cicd-containers.md) | — | 2026-02-28 |
| [Внутренний CA с Smallstep (step-ca)](security/smallstep-internal-ca.md) | — | 2026-03-02 |
| [HashiCorp Vault — полное руководство](security/vault-hashicorp.md) | — | 2026-02-28 |
| [Velociraptor — DFIR и threat hunting платформа](security/velociraptor.md) | — | 2026-03-20 |
| [WebmonitorX WAF (WMX)](security/webmonitorx-waf.md) | — | 2026-04-07 |
| [Supply Chain атака на Trivy GitHub Action (март 2026)](security/incidents/trivy-supply-chain-2026-03.md) | — | 2026-03-20 |
