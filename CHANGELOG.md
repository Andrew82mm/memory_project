# Changelog

Все значимые изменения в проекте. Формат: `[Phase] — дата — описание`.

---

## [Phase 1.1] — 2026-04-19 — Структура пакета `llm_memory/`

**Что сделано:**
Flat layout (7 файлов в корне) переведён в полноценный Python-пакет `llm_memory/` с подмодулями.

**Перемещены и обновлены импорты:**
| Было | Стало |
|---|---|
| `config.py` | `llm_memory/config.py` |
| `colors.py` | `llm_memory/colors.py` |
| `core_engine.py` | `llm_memory/engine.py` |
| `storage.py` | `llm_memory/storage/filesystem.py` |
| `knowledge_graph.py` | `llm_memory/graph/extractor.py` |
| `rag_engine.py` | `llm_memory/rag/retriever.py` |
| `llm_client.py` | `llm_memory/llm/client.py` |

**Созданы стабы для будущих фаз:**
- `llm_memory/models.py` — Pydantic-схемы (Phase 1.2)
- `llm_memory/buffer.py` — ConversationBuffer (Phase 1.4)
- `llm_memory/summarizer.py` — SummarizationPipeline (Phase 1.3)
- `llm_memory/prompt_builder.py` — ContextAssembler (Phase 2.8)
- `llm_memory/storage/base.py` — StorageBackend Protocol
- `llm_memory/graph/index.py` — SQLite relations (Phase 1.9)
- `llm_memory/graph/resolver.py` — Entity resolution (Phase 2.3)
- `llm_memory/graph/renderer.py` — Fact → Markdown (Phase 1.9)
- `llm_memory/rag/chunker.py` — Markdown chunking (Phase 1.8)
- `llm_memory/rag/vector.py` — Chroma wrapper (Phase 1.7)
- `llm_memory/llm/tokens.py` — Token counting (Phase 1.5)
- `llm_memory/cli/commands.py` — typer commands (Phase 2.13)
- `scripts/` — директория для миграционных скриптов
- `tests/unit/`, `tests/integration/` — директории для будущих тестов

**Корневой `main.py`** стал однострочным shim:
```python
from llm_memory.cli.main import run_cli
run_cli()
```

**Результат:** все 15 smoke-тестов проходят, `python main.py` работает.

---

## [Phase 0] — 2026-04-19 — Фундамент: инфраструктура и тесты

### 0.3 — Lazy config (убран `raise` на импорте)

**Проблема:** `config.py` делал `raise EnvironmentError` при импорте без API-ключа — любой тест падал при старте.

**Решение:**
- Убран `raise` из тела модуля
- Добавлена функция `get_openrouter_key()` — делает raise только при первом вызове
- В `llm_client.py` ключ теперь берётся лениво в `_call()`, а не в `__init__`

**Результат:** `OPENROUTER_API_KEY="" python -c "import llm_memory.config"` — не падает. Тесты без LLM не требуют ключа.

### 0.1 — Инфраструктура проекта

Создан базовый инструментарий:

- **`pyproject.toml`** — метаданные пакета, dev-зависимости (`pytest`, `pytest-cov`, `mypy`, `ruff`, `pydantic>=2`), конфиг pytest (`pythonpath = ["."]`, `testpaths = ["tests"]`) и ruff
- **`Makefile`** — цели `test`, `lint`, `typecheck`, `run`
- **`.pre-commit-config.yaml`** — `ruff` + `ruff-format` на каждый коммит
- **`.gitignore`** — добавлены `.pytest_cache/`, `.mypy_cache/`, `.coverage`, `htmlcov/`

### 0.2 — Smoke-тесты (15 штук)

Написаны smoke-тесты в `tests/smoke/` фиксирующие поведение до рефакторинга:

| Файл | Что тестирует |
|---|---|
| `test_storage_read_write.py` | `write_file` + `read_file` round-trip с frontmatter и без; дедупликация `append_to_file` |
| `test_storage_snapshot.py` | Создание снэпшота, восстановление состояния, наличие метаданных |
| `test_kg_extract.py` | Извлечение сущностей (мок LLM), обработка ошибки LLM, граф связей |
| `test_rag_index.py` | Индексация файлов в Chroma, семантический поиск, пустой индекс |
| `test_engine_chat.py` | Рост буфера после chat(), триггер суммаризации по порогу |

**Фикстуры в `tests/conftest.py`:**
- `tmp_memory_dir` — изолированный `StorageManager` в `tmp_path`
- `mock_llm` — monkeypatch `llm_client.generate` со списком заготовленных ответов

**Результат:** 15/15 зелёных, coverage 65% (план требовал ≥40%).

---

## [MVP] — до начала плана — Первая версия

Рабочий прототип с flat layout. Основные возможности:
- 4-слойная архитектура памяти (buffer → summary → KG → RAG)
- ChromaDB + `all-MiniLM-L6-v2` для векторного поиска
- Граф знаний на основе WikiLinks в Markdown
- Снэпшоты и rollback
- CLI с командами `pin`, `edit`, `project`, `rollback`, `reindex`
- OpenRouter как LLM-провайдер (две модели: smart + fast)

**Известные проблемы на момент начала работы по плану** (зафиксированы в IMPROVEMENT_PLAN.md):
- Нет тестов
- `config.py` падает при импорте без ключа
- Буфер очищается даже при фейле суммаризации (потеря данных)
- RAG не фильтрует нерелевантные результаты (нет score threshold)
- Chroma не включается в снэпшоты → rollback некорректен
- Английская эмбеддинг-модель для потенциально русского контента
- Циклический импорт `rag_engine` → `knowledge_graph` внутри метода