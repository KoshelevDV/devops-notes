# Векторная память (RAG) для OpenClaw с bge-m3 и llama-server

> Дата настройки: 2026-03-11  
> Статус: ✅ работает в проде

## Что это и зачем

OpenClaw поддерживает встроенный hybrid-поиск по файлам памяти (markdown-файлы в workspace). По умолчанию работает только BM25 (полнотекстовый поиск). Добавив векторные эмбеддинги, получаем **RAG** (Retrieval-Augmented Generation) — семантический поиск по смыслу, а не по ключевым словам.

Итог: агент находит релевантные факты из памяти, книг и статей даже если запрос сформулирован иначе, чем исходный текст.

**Архитектура:**
```
OpenClaw → memory_search() → SQLite (BM25 + vectors)
                                      ↑
                              bge-m3 embeddings
                                      ↑
                         llama-server (llama.cpp)
                         --batch-size 1024
                         --ubatch-size 1024
```

---

## Стек

| Компонент | Версия/детали |
|-----------|---------------|
| OpenClaw | 2026.2.21-2+ |
| llama.cpp / llama-server | build 7718+ |
| Модель эмбеддингов | bge-m3-Q8_0.gguf (1024 dims) |
| GPU | AMD Radeon (Vulkan/RADV) |
| Среда запуска | Toolbox-контейнер `llama-vulkan-radv` |
| База данных | SQLite (встроенная в OpenClaw) |
| Поиск | Hybrid: BM25 (0.4) + vectors (0.6) |

---

## 1. Подготовка модели

### Скачать bge-m3

```bash
# Создать папку для модели
mkdir -p ~/ai/models/bge-m3

# Скачать квантованную версию Q8_0 (~1.1 GB)
# Источник: https://huggingface.co/Qdrant/bge-m3-gguf
wget -O ~/ai/models/bge-m3/bge-m3-Q8_0.gguf \
  https://huggingface.co/Qdrant/bge-m3-gguf/resolve/main/bge-m3-Q8_0.gguf
```

Почему Q8_0: максимальное качество при разумном размере. Для слабых машин можно взять Q4_K_M (~600 MB).

---

## 2. Запуск llama-server

### Важные параметры

- `--embedding` — режим эмбеддингов (без него сервер работает как чат)
- `--batch-size 1024 --ubatch-size 1024` — **критично**: оба параметра должны совпадать, иначе llama-server для embeddings автоматически снижает оба до 512, что не позволяет обрабатывать чанки размером 400 токенов
- `--ctx-size 8192` — контекстное окно (bge-m3 поддерживает до 8192)
- `-ngl 999` — offload всё на GPU

### Запуск (AMD GPU через Vulkan в Toolbox)

```bash
toolbox run -c llama-vulkan-radv llama-server \
  -m ~/ai/models/bge-m3/bge-m3-Q8_0.gguf \
  --embedding \
  --host 0.0.0.0 \
  --port 8082 \
  --ctx-size 8192 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  -ngl 999 \
  > /tmp/bge-m3.log 2>&1 &
```

### Запуск (без GPU / CUDA)

```bash
llama-server \
  -m ~/ai/models/bge-m3/bge-m3-Q8_0.gguf \
  --embedding \
  --host 0.0.0.0 \
  --port 8082 \
  --ctx-size 8192 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  > /tmp/bge-m3.log 2>&1 &
```

### Проверка

```bash
# Health check
curl http://127.0.0.1:8082/health
# {"status":"ok"}

# Тест эмбеддинга
curl -s http://127.0.0.1:8082/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"bge-m3-q8_0","input":"test"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('dims:', len(d['data'][0]['embedding']))"
# dims: 1024

# Проверить batch size в логах
grep "n_batch" /tmp/bge-m3.log
# llama_context: n_batch = 1024
# llama_context: n_ubatch = 1024
```

### Автозапуск через systemd (опционально)

```ini
# ~/.config/systemd/user/bge-m3.service
[Unit]
Description=bge-m3 embedding server (llama-server)
After=network.target

[Service]
ExecStart=toolbox run -c llama-vulkan-radv llama-server \
  -m /home/user/ai/models/bge-m3/bge-m3-Q8_0.gguf \
  --embedding --host 0.0.0.0 --port 8082 \
  --ctx-size 8192 --batch-size 1024 --ubatch-size 1024 \
  -ngl 999
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now bge-m3.service
```

---

## 3. Конфигурация OpenClaw

Редактируем `~/.openclaw/openclaw.json` (или через `openclaw config patch`).

### Полный блок `memorySearch`

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "openai",
        "remote": {
          "baseUrl": "http://127.0.0.1:8082/v1",
          "apiKey": "dummy"
        },
        "fallback": "none",
        "model": "bge-m3-q8_0",
        "chunking": {
          "tokens": 400,
          "overlap": 50
        },
        "sync": {
          "onSessionStart": true,
          "onSearch": true,
          "watch": true,
          "watchDebounceMs": 2000
        },
        "query": {
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.6,
            "textWeight": 0.4
          }
        },
        "extraPaths": [
          "/home/user/.openclaw/workspace/books",
          "/home/user/.openclaw/workspace/articles"
        ]
      }
    }
  }
}
```

### Применить через CLI

```bash
# Патч (merge с существующим конфигом)
openclaw config patch '{"agents":{"defaults":{"memorySearch":{...}}}}'

# После изменения extraPaths — нужен полный рестарт (не SIGUSR1!)
systemctl --user restart openclaw-gateway.service
```

---

## 4. Параметры конфигурации — объяснение

| Параметр | Значение | Описание |
|----------|----------|----------|
| `provider` | `openai` | Тип API. llama-server совместим с OpenAI `/v1/embeddings` |
| `remote.baseUrl` | `http://127.0.0.1:8082/v1` | URL llama-server |
| `remote.apiKey` | `dummy` | Любая строка — llama-server не проверяет |
| `model` | `bge-m3-q8_0` | Название модели (передаётся в запросе) |
| `chunking.tokens` | `400` | Размер чанка в токенах. Влияет на точность поиска и контекст результата |
| `chunking.overlap` | `50` | Перекрытие между чанками — чтобы не терять контекст на границах |
| `sync.onSearch` | `true` | Синкать при каждом поиске (ловит новые/изменённые файлы) |
| `sync.watch` | `true` | Файловый watcher — индексировать при изменении файлов |
| `query.hybrid.vectorWeight` | `0.6` | 60% веса — векторный поиск |
| `query.hybrid.textWeight` | `0.4` | 40% веса — BM25 (ключевые слова) |
| `extraPaths` | `[...]` | Дополнительные папки для индексации (помимо `memory/`) |

---

## 5. Размер чанка: как выбрать

**Проблема:** llama-server имеет физический batch size. Чанк + служебные токены должны влезать в него.

```
чанк 400 токенов + overhead (~160) = ~560 токенов → нужен batch ≥ 1024
чанк 256 токенов + overhead (~100) = ~356 токенов → влезает в batch 512
```

**Рекомендации:**
- `256` — если `--batch-size 512` (по умолчанию): точные результаты, меньше контекста
- `400` — если `--batch-size 1024 --ubatch-size 1024`: хороший баланс ✅
- `512+` — для длинных документов, нужен batch 2048+

**Важно:** при изменении `chunking.tokens` нужно пересоздать SQLite-индекс:
```bash
rm ~/.openclaw/memory/main.sqlite
systemctl --user restart openclaw-gateway.service
# Затем триггернуть поиск для инициализации
```

---

## 6. Добавление книг и статей

Индексируются только текстовые файлы (`.md`, `.txt`). PDF и EPUB нужно конвертировать.

### PDF → Markdown (через pymupdf)

```python
import fitz, re

doc = fitz.open("book.pdf")
pages = [page.get_text() for page in doc if page.get_text().strip()]
text = "\n\n".join(pages)
text = re.sub(r'\n{4,}', '\n\n\n', text)  # схлопнуть пустые строки

with open("/home/user/.openclaw/workspace/books/book-name.md", "w") as f:
    f.write(f"# Название книги\n\n{text}")
```

```bash
# Или через pdftotext (проще, но хуже структура)
pdftotext -layout book.pdf - > books/book-name.txt
```

### EPUB → Markdown

```python
import zipfile, re
from html.parser import HTMLParser

class HTMLStripper(HTMLParser):
    def __init__(self):
        super().__init__()
        self.text = []
    def handle_data(self, data):
        self.text.append(data)
    def get_text(self):
        return ''.join(self.text)

with zipfile.ZipFile("book.epub") as z:
    texts = []
    for name in sorted(z.namelist()):
        if name.endswith(('.html', '.xhtml', '.htm')):
            html = z.read(name).decode('utf-8', errors='ignore')
            stripper = HTMLStripper()
            stripper.feed(html)
            texts.append(stripper.get_text())

with open("books/book-name.md", "w") as f:
    f.write("\n\n".join(texts))
```

После добавления файла в `books/` или `articles/` — индексер подхватит автоматически через 2 секунды (watch mode).

---

## 7. Проверка состояния индекса

```bash
python3 - << 'EOF'
import sqlite3, json

conn = sqlite3.connect('/home/user/.openclaw/memory/main.sqlite')
c = conn.cursor()

c.execute("SELECT * FROM meta")
row = c.fetchone()
if row:
    d = json.loads(row[1])
    print(f"model:     {d.get('model')}")
    print(f"provider:  {d.get('provider')}")
    print(f"chunk:     {d.get('chunkTokens')} tokens")
    print(f"dims:      {d.get('vectorDims')}")

c.execute("SELECT COUNT(*) FROM files");    print(f"files:     {c.fetchone()[0]}")
c.execute("SELECT COUNT(*) FROM chunks");   print(f"chunks:    {c.fetchone()[0]}")
c.execute("SELECT COUNT(*) FROM embedding_cache"); print(f"embeddings:{c.fetchone()[0]}")

print("\nИндексированные файлы:")
c.execute("SELECT path FROM files ORDER BY path")
for (path,) in c.fetchall():
    print(f"  {path}")

conn.close()
EOF
```

---

## 8. Частые проблемы

### ❌ `input (560 tokens) is too large (batch size: 512)`

**Причина:** chunk слишком большой для физического batch llama-server.  
**Решение:** перезапустить llama-server с `--batch-size 1024 --ubatch-size 1024` (оба обязательны).

### ❌ После изменения конфига индекс не обновляется

**Причина:** SIGUSR1 (openclaw gateway restart) не пересоздаёт memory indexer.  
**Решение:** полный рестарт `systemctl --user restart openclaw-gateway.service` + удалить `main.sqlite` если менялся `chunking.tokens`.

### ❌ Новые `extraPaths` не индексируются

**Причина:** file watcher инициализируется один раз при старте.  
**Решение:** полный рестарт сервиса после изменения `extraPaths`.

### ❌ `model: fts-only, provider: none` в meta

**Причина:** индекс был создан до настройки провайдера и закешировал старое состояние.  
**Решение:** `rm ~/.openclaw/memory/main.sqlite` + рестарт + триггернуть `memory_search`.

### ❌ `main.sqlite` остаётся 0 байт после рестарта

**Причина:** индексер инициализируется лениво — только при первом поиске.  
**Решение:** вызвать любой `memory_search` запрос.

---

## 9. Итоговая архитектура (как работает)

```
1. Файлы в workspace/memory/, books/, articles/
           ↓  (watch / onSearch / onSessionStart)
2. OpenClaw разбивает на чанки (400 токенов, overlap 50)
           ↓
3. Каждый чанк → POST /v1/embeddings → llama-server → bge-m3
           ↓
4. Вектор (1024 float) сохраняется в SQLite embedding_cache
           ↓
5. memory_search("запрос"):
   - запрос → вектор (через bge-m3)
   - BM25 поиск по chunks_fts
   - cosine similarity по векторам
   - hybrid RRF merge (0.6 vector + 0.4 BM25)
           ↓
6. Топ N чанков → в контекст агента (с path + line numbers)
```

---

## Источники

- [OpenClaw docs: Memory Search](https://docs.openclaw.ai)
- [llama.cpp llama-server](https://github.com/ggerganov/llama.cpp)
- [bge-m3 GGUF модели](https://huggingface.co/Qdrant/bge-m3-gguf)
- [BGE-M3 paper](https://arxiv.org/abs/2402.03216) — Multi-Functionality, Multi-Linguality, Multi-Granularity
