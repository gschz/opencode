---
name: sepomex-psql-db
description: "Trigger: sepomex, postal code, códigos postales, SEPOMEX data pipeline, PostgreSQL postal DB. Work on the Python→SQL data pipeline or PostgreSQL database for Mexican postal codes."
license: Apache-2.0
metadata:
  author: "gsanz"
  version: "1.0"
---

## Activation Contract

Use this skill when:
- Modifying the Python pipeline (`src/`) that reads SEPOMEX TXT and generates INSERT SQL.
- Changing the PostgreSQL schema, indexes, views, or PL/pgSQL functions (`database/`).
- Debugging encoding issues, data validation, or SQL generation.
- Adding new endpoints or queries to the PL/pgSQL layer.

## Project Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Language | Python 3.13 | Entry: `python -m src.main` |
| Data | pandas 2.2.3 | Only dependency |
| DB | PostgreSQL | ~150k rows in `codigos_postales` |
| Formatting | black + isort | `line-length = 88`, profile `black` |
| Input format | TXT, `\|`-delimited, `windows-1252` | File: `data/input/sepomex_data.txt` |

## File Roles

| Path | Purpose |
|------|---------|
| `src/main.py` | Orchestrator: read → validate → generate SQL (respects FK order) |
| `src/config.py` | All constants: paths, regex patterns, column mappings, zone map |
| `src/data_reader.py` | Reads TXT into DataFrame (handles `windows-1252` encoding) |
| `src/data_validator.py` | Optional row-level validation (currently bypassed in `main.py`) |
| `src/sql_generator.py` | Generates INSERT SQL files per entity into `data/generated_sql_v2/` |
| `src/utils.py` | `clean_text()` (CRITICAL — encoding sanitization), `format_codigo()`, `normalize_zona()` |
| `src/models.py` | Data classes / type definitions |
| `database/schema.sql` | Table DDL: `estados`, `municipios`, `ciudades`, `tipos_asentamiento`, `zonas`, `codigos_postales` |
| `database/indexes.sql` | Partial indexes, lowercase search index, VM indexes |
| `database/views.sql` | Materialized view `vm_codigos_postales` (precomputed joins) |
| `database/functions.sql` | 14 PL/pgSQL functions mapping 1:1 to `/api/v2/` endpoints |
| `data/generated_sql_v2/` | Output: numbered INSERT SQL files (apply in numerical order) |

## Conventions

- **DB naming**: `pk_` for primary keys, `fk_` for foreign keys. All table names plural Spanish.
- **Normalization**: 3NF. No audit columns (`hora`, `fecha`, `estado` removed — static dataset).
- **Data types**: `VARCHAR(50)` for names, `SMALLINT` for zone IDs (~3-5 rows), `SERIAL` for most PKs.
- **DB apply order**: `schema.sql` → `indexes.sql` → `views.sql` → `functions.sql`.
- **Generated SQL apply order**: numerical filename order in `data/generated_sql_v2/`.
- **"Duplicidad funcional"**: Rows that look identical but have distinct PKs. Preserved at DB level. Deduplication responsibility is in the API layer, NOT in the pipeline.
- **Zone normalization**: Always maps to `Urbano` / `Rural` / `Semiurbano`. Default is `Semiurbano`.

## Gotchas — READ BEFORE CHANGES

1. **`clean_text()` in `src/utils.py` is critical.** Input encoding produces invalid UTF-8 sequences (C1 control chars `0x80-0x9F`). The function strips them and escapes SQL quotes. ANY text field change must route through `clean_text()`.

2. **No tests.** There is no test suite, no linter, no type checker. Verify manually. Run `python -m src.main` and inspect generated SQL.

3. **`validate_dataframe()` is bypassed.** `main.py` line 52: raw data goes straight to generators. Validation happens per-row in `sql_generator.py`. Do not assume validation has run.

4. **FK generation order matters.** `main.py` generates SQL in dependency order: estados → municipios → tipos_asentamiento → zonas → ciudades → codigos_postales. Do not reorder.

5. **Encoding on input.** The source file is `windows-1252`, NOT UTF-8. `data_reader.py` reads with `encoding='windows-1252'`. Changing this will break the pipeline.

6. **Materialized view must be refreshed** after regenerating `codigos_postales` INSERTs. It's not auto-refreshed.

7. **`codigo_postal` is `CHAR(5)` in DB but VARCHAR during generation.** Check constraints enforce `^[0-9]{5}$`. Trailing whitespace matters.

8. **Functions return `RETURNS TABLE (...)`** with exact column types matching the view. Changing the VM column types requires updating all functions that query it.

## Common Commands

```bash
# Run the pipeline
python -m src.main

# Format code (from project root)
black src/
isort src/

# Apply DB schema (order matters)
psql -d <db> -f database/schema.sql
psql -d <db> -f database/indexes.sql
psql -d <db> -f database/views.sql
psql -d <db> -f database/functions.sql

# Apply generated inserts (numerical order)
for f in data/generated_sql_v2/*.sql; do psql -d <db> -f "$f"; done

# Refresh materialized view after inserts
psql -d <db> -c "REFRESH MATERIALIZED VIEW vm_codigos_postales;"
```

## Endpoints ↔ Functions Map

Each PL/pgSQL function maps 1:1 to an API endpoint. See `docs/SEPOMEX_V2.md` §"Análisis de la Colección Postman v2" for full parameter/return specs. Key ones:

| Function | Endpoint | Query |
|----------|----------|-------|
| `search_settlements_by_name` | `/api/v2/postal/search` | `vm_codigos_postales` |
| `search_by_postal_code` | `/api/v2/postal/codigo/{code}` | `vm_codigos_postales` |
| `get_postal_codes_by_state` | `/api/v2/postal/estado/{id}` | `vm_codigos_postales` |
| `get_all_states` | `/api/v2/estado` | `estados` |
| `get_all_cities` | `/api/v2/cities` | `ciudades` |

Full list: 14 functions in `database/functions.sql`.
