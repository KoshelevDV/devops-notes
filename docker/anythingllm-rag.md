# AnythingLLM — self-hosted база знаний (RAG)

> **Дата:** 2026-03-11  
> **Теги:** `rag` `llm` `knowledge-base` `anythingllm` `docker`

---

## 🎯 Задача

Self-hosted система вопрос-ответ по внутренней документации (Confluence, Wiki.js, Yandex Wiki, файлы).
Архитектура: **RAG** (Retrieval-Augmented Generation).

```
Вопрос → embed → поиск по индексу → топ-N чанков → LLM → ответ
```

---

## 🧩 Компоненты

| Компонент | Роль | Что используем |
|-----------|------|----------------|
| **AnythingLLM** | UI + оркестрация | self-hosted Docker |
| **Embedding модель** | векторизация документов и запросов | bge-m3-Q8_0 (порт 8082) |
| **LLM** | формирование ответа | GLM-4.7-Flash / Qwen3 (llama-server) |

---

## ✅ Запуск AnythingLLM

```bash
docker run -d \
  --name anythingllm \
  -p 3001:3001 \
  -v ~/.anythingllm:/app/server/storage \
  --restart unless-stopped \
  mintplexlabs/anythingllm
```

Или через docker-compose:

```yaml
services:
  anythingllm:
    image: mintplexlabs/anythingllm
    container_name: anythingllm
    ports:
      - "3001:3001"
    volumes:
      - ~/.anythingllm:/app/server/storage
    restart: unless-stopped
```

---

## ⚙️ Настройка LLM (OpenAI-compatible)

В настройках AnythingLLM → LLM Provider → **Generic OpenAI**:

```
Base URL:  http://10.0.30.18:8080/v1
API Key:   dummy
Model:     (имя модели из llama-server)
```

---

## ⚙️ Настройка Embedding

В настройках → Embedding Provider → **Generic OpenAI**:

```
Base URL:  http://10.0.30.18:8082/v1
API Key:   dummy
Model:     bge-m3-q8_0
```

---

## 💡 Подводные камни

- AnythingLLM из Docker-контейнера не видит `127.0.0.1` хоста — использовать `10.0.30.18` (внутренний IP)
- bge-m3 должен быть запущен до старта AnythingLLM (`systemctl --user start bge-m3-embeddings`)
- Для Confluence/Wiki.js нужны коннекторы — доступны в платном плане или через API-импорт файлов

## 📎 Документация

- [AnythingLLM docs](https://docs.anythingllm.com/)
- [Docker Hub](https://hub.docker.com/r/mintplexlabs/anythingllm)
