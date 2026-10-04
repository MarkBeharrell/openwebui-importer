# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Python ETL pipeline that converts AI chat exports (ChatGPT, Claude, Grok, AI Studio) into SQL INSERT statements compatible with [open-webui](https://github.com/open-webui/open-webui)'s SQLite database.

## Commands

**Run tests:**
```bash
pytest tests/
```

**Run a single test:**
```bash
pytest tests/test_conversion.py::test_chatgpt_conversion
```

**Run a converter (Stage 1):**
```bash
python convert_claude.py --input examples/claude_example.json --user-id <uuid>
```

**Generate SQL (Stage 2):**
```bash
python create_sql.py --input output/ --output chats.sql
```

**Batch process (both stages):**
```bash
python scripts/run_batch.py --platform claude --input <file_or_dir> --user-id <uuid>
```

**Docker:**
```bash
docker run --rm -v $(pwd)/data:/data ghcr.io/yetanotherchris/openwebui-importer python convert_claude.py --input /data/export.json --user-id <uuid>
```

## Architecture

Two-stage pipeline:

**Stage 1 — Format conversion** (`convert_*.py`): Each converter transforms a platform-specific JSON export into open-webui's internal JSON format. Output goes to `output/<platform>/`.

**Stage 2 — SQL generation** (`create_sql.py`): Reads converted JSON files and produces SQL with DELETE + INSERT for chats, plus UPSERT statements for import tags (e.g., `imported-claude`).

**Orchestration** (`scripts/run_batch.py`): Wraps both stages for batch processing.

### Converter structure

All `convert_*.py` files share the same pattern:
- Constants: `MODEL`, `MODEL_NAME`, `SUBDIR`
- `sanitize_text()` — strips private-use Unicode (U+E000–U+F8FF)
- `parse_<platform>()` — platform-specific JSON parsing
- `build_webui()` — constructs the open-webui JSON structure
- `main()` — argparse CLI entry point with `--input`, `--user-id`, `--output-dir`

Platform-specific features:
- **convert_claude.py**: Extended thinking blocks rendered as `<details>` HTML tags
- **convert_chatgpt.py**: Branching conversation tree traversal, multi-part content
- **convert_aistudio.py**: Image attachments (Google Drive IDs), chunked prompt structure
- **convert_grok.py**: Responses array or mapping structure

### Adding a new platform

1. Create `convert_<platform>.py` following the existing converter pattern
2. Add a JSON schema to `schemas/`
3. Add example input to `examples/`
4. Add expected output to `tests/expected/`
5. Add a test case to `tests/test_conversion.py` (patch `uuid.uuid4` and `time.time` for determinism)

## Testing Approach

Tests in `tests/test_conversion.py` use `unittest.mock.patch` to freeze UUIDs and timestamps, then compare converter output against fixture files in `tests/expected/`. Always update the expected output fixture when intentionally changing conversion behavior.

## Dependencies

Only `jsonschema` is required externally (`pip install jsonschema`). Everything else uses the Python 3.12 standard library.
