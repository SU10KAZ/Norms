# norms_search

Две функции поверх локального Obsidian-vault'а нормативной документации
(`../vault/`): **список действующих норм** и **поиск пункта по номеру**.

Изолированный MVP, не зависит от `2-Multi-Agent-Manager`.

## Setup после clone

В репо не входят тяжёлые артефакты (`.npz`, `venv/`, `paragraphs.jsonl`).
Чтобы поднять всё локально:

```bash
cd tools
python3 -m venv venv
source venv/bin/activate
pip install numpy sentence-transformers pyyaml

python3 list_active.py --quiet          # active_norms.json (индекс норм)
python3 extract_refs.py                 # refs_graph.json + секции «Связанные нормы»
python3 embed_norms.py                  # embeddings.npz + semantic_neighbors.json
python3 build_paragraph_index.py        # paragraphs.jsonl (парсинг всех пунктов)
python3 embed_paragraphs.py             # paragraphs_embeddings.npz (для search.py)
```

После этого работают `find_paragraph.py`, `search.py`, `mcp_server.py`.

## Файлы

| Файл | Назначение |
|---|---|
| `parse_filename.py` | Хелпер: парсит имя MD → `{type, code, year, title}`. |
| `list_active.py` | Собирает `active_norms.json`, печатает таблицу. |
| `find_paragraph.py` | CLI-поиск пункта в конкретном документе по номеру. |
| `extract_refs.py` | Строит граф цитирований (`refs_graph.json`), инъектирует frontmatter и секцию `## Связанные нормы` в каждый `.md`. |
| `embed_norms.py` | Эмбеддинги норм (по scope-секции) → `semantic_neighbors.json` + `## Похожие по смыслу` в `.md`. Модель e5-large. |
| `build_paragraph_index.py` | Парсит все пункты (41k+) из норм → `paragraphs.jsonl`. |
| `embed_paragraphs.py` | Эмбеддинги каждого пункта → `paragraphs_embeddings.npz`. Модель e5-base. |
| `search.py` | Семантический поиск по пунктам: `search.py "огнестойкость перекрытий"`. |
| `status_overrides.yaml` | Список отменённых/заменённых норм (правишь руками). |
| `active_norms.json` | Результат `list_active.py`. |
| `venv/` | Python-окружение для эмбеддингов (`source venv/bin/activate`). |

## Быстрый старт

### 1. Собрать индекс действующих норм

```bash
python3 list_active.py           # таблица первых 50 + active_norms.json
python3 list_active.py --all     # вся таблица
python3 list_active.py --quiet   # только файл, без таблицы
```

Результат: `active_norms.json` вида
```json
{
  "meta": { "indexed_at": "...", "total": 337, "parse_failures": [...] },
  "norms": [
    { "code": "СП 256.1325800.2016", "type": "СП", "year": 2016,
      "title": "...", "file": "СП 256_1325800_2016_...md", "status": "active" }
  ]
}
```

### 2. Найти пункт в норме

```bash
python3 find_paragraph.py "СП 256.1325800.2016" "15.30"
python3 find_paragraph.py "ГОСТ 10180-2012" "1"
python3 find_paragraph.py "СП 256.1325800.2016" "3.1.6"
```

Вывод: текст пункта в `stdout`, метаданные (файл, номер строки) в `stderr`.
Exit code `0` — найдено, `1` — не найдено.

Флаг `--max-lines N` ограничивает длину вывода (для многостраничных пунктов).

### 3. Пометить норму как отменённую

Откройте `status_overrides.yaml`, добавьте в `overrides`:

```yaml
overrides:
  СП 31-110-2003: cancelled
  СНиП 2.04.01-85:
    replaced_by: СП 30.13330.2020
```

Код пишется в **каноничной форме** — как в `active_norms.json` (точки вместо подчёркиваний для СП и ГОСТ Р). Затем перезапустите `list_active.py`.

## Как парсятся коды

| Вход пользователя | Каноничный код | Имя файла |
|---|---|---|
| `СП 256.1325800.2016` | `СП 256.1325800.2016` | `СП 256_1325800_2016_...md` |
| `ГОСТ 10180-2012` | `ГОСТ 10180-2012` | `ГОСТ 10180-2012_...md` |
| `ГОСТ Р 10.0.02-2019` | `ГОСТ Р 10.0.02-2019` | `ГОСТ Р 10_0_02-2019_...md` |

При поиске файла скрипты нормализуют точки в подчёркивания, так что пишите код в любом формате — будет найдено.

## Алгоритм поиска пункта

1. Нормализуем код (точки → подчёркивания).
2. Glob по vault: `<code>*_document.md`.
3. Внутри файла ищем строку, начинающуюся с номера пункта.
4. **Приоритет markdown-заголовкам** — если `##### 15.30` существует, берём его (защита от ложных срабатываний вроде «15 июля»).
5. Собираем текст от найденной строки до следующего пункта / `## СТРАНИЦА` / `### BLOCK` / любого `#`-заголовка.

## Что не поддерживается

- Семантический поиск («где требования к огнестойкости»). Только точный номер пункта.
- Пункты приказов/постановлений (тип `other`) — у них нестандартная структура, требует отдельной работы.
- Автоматическое определение статуса (`cancelled`) — только ручное через `status_overrides.yaml`.

## Известные ограничения

- `parse_confidence: low` для 37 документов (приказы, постановления, законы). Они попадают в `active_norms.json` с `type: other`, но парсить их `find_paragraph.py` пока не умеет.
- Пункт, встречающийся в тексте (не в заголовке) без уникального префикса, может найтись не в том месте — обычно исправляется добавлением контекста (например, искать `15.30` вместо `30`).

## Будущее

Когда инструмент докажет полезность:
- Импорт `active_norms.json` в `2-Multi-Agent-Manager` как замену `norms_db.json`.
- Вызов `find_paragraph.py` из `Audits/synthesize.py` через subprocess для заполнения `norm_quote`.
- Или оформление как MCP-сервер (tools: `list_active`, `find_paragraph`).
