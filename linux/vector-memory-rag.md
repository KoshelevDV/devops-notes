# Векторная память (RAG) для AI-агента с BGE-M3 и OpenViking

> Обновлено: 2026-04-06
> Статус: ✅ работает в проде (Hermes + OpenViking 0.3.3)

## Что это и зачем

AI-агент использует гибридный поиск по базе знаний (документация, книги, заметки).
По умолчанию — только BM25 (полнотекстовый поиск). Добавив векторные эмбеддинги,
получаем **RAG** (Retrieval-Augmented Generation) — семантический поиск по смыслу.

Итог: агент находит релевантные факты даже если запрос сформулирован иначе, чем исходный текст.

**Архитектура:**
```
Hermes → viking_search() → OpenViking (hybrid search: BM25 + vector)
                                    ↑
                            BGE-M3 embeddings (1024-dim)
                                    ↑
                       llama-server (llama.cpp, Vulkan/RADV)
                       --embedding --ctx-size 8192
```

---

## Стек

| Компонент | Версия/детали |
|-----------|---------------|
| OpenViking | 0.3.3 |
| BGE-M3 | Q8_0, 1024-dim |
| llama-server | llama.cpp, Vulkan/RADV |
| VLM (семантика) | Qwen2.5-Coder-7B Q4_K_M |
| Hermes | текущий агент |

---

## Сервисы (systemd, автозапуск)

| Сервис | Порт | Описание |
|--------|------|----------|
| `bge-m3-embeddings.service` | 8082 | BGE-M3 Q8_0, embedding |
| `qwen-vlm.service` | 8083 | Qwen2.5-Coder-7B, семантический анализ |
| `openviking.service` | 1933 | OpenViking, vector store + RAG |

```bash
systemctl --user status bge-m3-embeddings openviking qwen-vlm
curl -s http://127.0.0.1:8082/health   # embeddings
curl -s http://127.0.0.1:1933/health   # openviking
```

---

## Конфиг OpenViking

`/home/user/.openviking/ov.conf`:
```json
{
  "embedding": {
    "dense": {
      "provider": "openai",
      "model": "bge-m3-Q8_0.gguf",
      "api_base": "http://localhost:8082/v1",
      "dimension": 1024
    }
  },
  "vlm": {
    "provider": "openai",
    "model": "qwen2.5-coder-7b-instruct-q4_k_m.gguf",
    "api_base": "http://localhost:8083/v1"
  },
  "storage": { "workspace": "/home/user/.openviking/data" },
  "server": { "host": "127.0.0.1", "port": 1933 }
}
```

---

## Добавление документов в базу знаний

Файлы нельзя добавить напрямую по локальному пути — нужен temp_upload:

```bash
# 1. Загрузить файл во временное хранилище
TEMP_ID=$(curl -s -X POST http://127.0.0.1:1933/api/v1/resources/temp_upload \
  -F "file=@/path/to/doc.md" \
  -F "filename=doc.md" | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['temp_file_id'])")

# 2. Добавить ресурс
curl -s -X POST http://127.0.0.1:1933/api/v1/resources \
  -H 'Content-Type: application/json' \
  -d "{\"temp_file_id\": \"$TEMP_ID\", \"reason\": \"описание документа\", \"wait\": false}"
```

Для GitHub/GitLab репо и URL — напрямую через `path`:
```bash
curl -s -X POST http://127.0.0.1:1933/api/v1/resources \
  -H 'Content-Type: application/json' \
  -d '{"path": "https://github.com/org/repo", "reason": "...", "wait": false}'
```

---

## Мониторинг очереди индексирования

```bash
curl -s http://127.0.0.1:1933/api/v1/observer/queue | python3 -c \
  "import json,sys; print(json.load(sys.stdin)['result']['status'])"
```

Типичный вывод:
```
| Embedding    |   0  |  10  | 1920  | 0 | 1944 |
| Semantic     |   0  |   0  |   17  | 0 |   17 |
```

GPU нагрузка во время индексирования: Semantic-фаза (Qwen) — ~90%, 95W. После неё Embedding-фаза (BGE-M3) — ~15-20W.

---

## Загруженные источники

| Источник | Описание |
|----------|----------|
| Ansible Up and Running (RU) | Полный справочник |
| Nginx Cookbook (RU) | Конфигурация и паттерны |
| Podman in Action (RU) | Rootless контейнеры |
| Linux From Scratch 12.2 (RU) | Сборка Linux с нуля |
| PostgreSQL + Patroni HA | High availability кластер |
| System Design Fundamentals | Проектирование систем |
| yandex-cloud/docs (ru/) | ~23k md-файлов, синк еженедельно |
| github.com/hoophq/hoop | Access Gateway платформа |

---

## ⚠️ Подводные камни

- Локальные пути (`/home/user/...`) API не принимает — только temp_upload или URL
- `wait: false` — асинхронно; документ индексируется в фоне, сразу искать не получится
- Semantic-очередь (Qwen) нагружает GPU сильнее, чем Embedding (BGE-M3)
- Первый запуск llama-server занимает ~15-20 сек (загрузка модели)
