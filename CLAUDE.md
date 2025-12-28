# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Development Setup
```bash
pip install nox
```

### Testing
```bash
# Run default test suite (lint + tests)
nox

# Run tests for all Python versions
nox -s tests

# Run tests for specific Python version
nox -s tests-3.13

# Run a single test file
nox -s tests -- tests/unit/lib/test_data_model.py

# Run a specific test
nox -s tests -- tests/unit/lib/test_data_model.py::test_name -v

# Run tests in parallel with pytest-xdist
pytest --numprocesses=logical --dist=loadgroup

# Run e2e tests
nox -s e2e

# Run benchmarks
nox -s bench

# Run example tests
nox -s examples
```

### Linting and Formatting
```bash
# Run all linters and formatters
nox -s lint

# Run pre-commit hooks manually
pre-commit run --all-files
```

### Type Checking
```bash
# mypy runs as part of pre-commit
mypy src tests
```

### Documentation
```bash
# Build docs
nox -s docs

# Serve docs locally with hot reload
mkdocs serve
```

### Build
```bash
nox -s build
```

## Architecture

DataChain is a Python-based AI-data warehouse for processing unstructured data (images, audio, videos, text, PDFs). It uses a layered architecture combining SQL warehouse efficiency with Python flexibility.

### Core Layers

**User API Layer** (`src/datachain/lib/dc/`)
- Entry point for all user interactions
- `DataChain` class provides fluent API: `map()`, `filter()`, `gen()`, `agg()`, `save()`
- Readers: `read_storage()`, `read_json()`, `read_csv()`, `read_parquet()`, etc.
- All operations are immutable and return new DataChain instances

**Query Engine Layer** (`src/datachain/query/`)
- `DatasetQuery` builds lazy SQL query plans
- Query steps: Filter → Map → Join → Aggregate → OrderBy → Limit
- Execution triggered by terminal operations: `.save()`, `.collect()`, `.results()`
- `Session` manages context, temp datasets, and cleanup

**Storage Layer** (`src/datachain/data_storage/`)
- **Metastore**: Metadata DB storing datasets, versions, jobs, dependencies
- **Warehouse**: Actual data tables with dataset rows (`ds_<dataset_name>`)
- Default: SQLite (pluggable via `DATACHAIN_METASTORE`/`DATACHAIN_WAREHOUSE` env vars)
- PostgreSQL support via `datachain[postgres]`

**Client Layer** (`src/datachain/client/`)
- Protocol adapters for S3, GCS, Azure, local, HuggingFace, HTTP
- Built on `fsspec` with specialized implementations
- Handles listing, reading, and caching of remote files

**Catalog Layer** (`src/datachain/catalog/`)
- Orchestrates metastore and warehouse
- Dataset lifecycle: create, version, delete, clone
- Dependency tracking and lineage
- Push/pull to DataChain Studio

### Data Processing Flow

1. **Reading**: `read_storage(uri)` → Client lists files → Creates File objects → Saves to `lst__<uri>` dataset
2. **Transformation**: `.map()/.filter()` → Wraps UDF → Adds step to query plan → Returns new DataChain
3. **Execution**: `.save()` → Builds SQL query → Dispatches UDFs to workers → Joins results → Materializes dataset
4. **Output**: `.to_storage()`, `.to_pandas()`, `.collect()`, `.results()`

### UDF (User-Defined Functions)

**Types:**
- `Mapper`: 1:1 transformation
- `Generator`: 1:N transformation (explode)
- `Aggregator`: N:1 aggregation
- `BatchMapper`: Batch processing

**Execution:**
- Single-process by default
- Multi-process with `.settings(parallel=N)`
- Workers managed by `UDFDispatcher` (`src/datachain/query/dispatch.py`)
- Serialization via `cloudpickle`
- Caching by UDF hash + input hash

### Signal Schema System

- All columns are "signals" with Python types mapped to SQL types
- Nested objects flattened: `person.address.city` → `person__address__city`
- `DataModel` extends Pydantic with auto-registration
- Type conversion: Python ↔ SQL ↔ JSON

### Key Patterns

**Lazy Evaluation**: Queries built lazily, executed only when needed (`.save()`, `.collect()`, `.count()`)

**Immutability**: Every operation returns a new DataChain instance

**Dataset Versioning**: Semantic versioning (`1.2.3`), immutable once created, stored as `namespace.project.dataset@v1.2.3`

**Delta Processing**: `read_storage(uri, delta=True, delta_on="file.path")` processes only new/changed files, with `delta_retry` for error handling

**Storage Listing Cache**: Bucket listings cached as `lst__<uri>` datasets (4-hour TTL)

**Session Management**: Global session tracks temp datasets (`session_<name>_<uuid>`), cleanup on exit

**Checkpoints**: Resumable computation via checkpoint tracking in metastore

## Important Code Locations

### Core DataChain API
- `src/datachain/lib/dc/datachain.py` - Main DataChain class with fluent API
- `src/datachain/lib/dc/storage.py` - `read_storage()` implementation
- `src/datachain/lib/dc/datasets.py` - Dataset management

### Data Models
- `src/datachain/lib/file.py` - File, ImageFile, VideoFile, AudioFile, TextFile models
- `src/datachain/lib/data_model.py` - Base DataModel class
- `src/datachain/model/` - Specialized models (BBox, Segment, Pose)

### Query Engine
- `src/datachain/query/dataset.py` - DatasetQuery (SQL query builder)
- `src/datachain/query/session.py` - Session context
- `src/datachain/query/dispatch.py` - UDF parallel execution dispatcher
- `src/datachain/query/batch.py` - Batching strategies

### UDFs
- `src/datachain/lib/udf.py` - UDF base classes (Mapper, Generator, Aggregator)
- `src/datachain/lib/signal_schema.py` - Schema and type system

### Storage & Catalog
- `src/datachain/catalog/catalog.py` - Catalog orchestrator
- `src/datachain/data_storage/metastore.py` - Metadata storage
- `src/datachain/data_storage/warehouse.py` - Data table storage
- `src/datachain/data_storage/sqlite.py` - SQLite implementation

### Type System
- `src/datachain/lib/convert/python_to_sql.py` - Python to SQL type conversion
- `src/datachain/lib/convert/sql_to_python.py` - SQL to Python type conversion
- `src/datachain/lib/convert/flatten.py` - Nested object flattening
- `src/datachain/lib/convert/unflatten.py` - Nested object reconstruction

### Delta Processing
- `src/datachain/delta.py` - Delta processing logic
- `src/datachain/diff/` - Diff utilities

## Testing Notes

- Tests in `tests/` directory mirror `src/datachain/` structure
- Unit tests: `tests/unit/`
- Functional tests: `tests/func/`
- E2E tests: `tests/e2e/` (marked with `@pytest.mark.e2e`)
- Example tests: `tests/examples/` (marked with `@pytest.mark.examples`)
- Project maintains 100% code coverage
- Use pytest markers for filtering: `pytest -m "not e2e"`, `pytest -m examples`

## Database Schema

When working with database migrations or schema changes:
- Metastore tables: `namespace`, `project`, `dataset`, `dataset_version`, `job`, `job_dependency`, `checkpoint`
- Warehouse tables: `ds_<dataset_name>` for each saved dataset
- Temp tables: `session_<name>_<uuid>` for intermediate results
- Listing cache: `lst__<uri_hash>` for bucket listings

## Signal/Column Naming

DataChain uses "signals" terminology for columns:
- Nested fields use double underscore: `file__path`, `person__address__city`
- Reserved signal names: `sys__id`, `sys__rand`
- User signals added via `.map()`, `.gen()` append to schema
- Signal schema defined in `SignalSchema` class

## Common Gotchas

1. **File caching**: Files downloaded to `.datachain/cache/` by default
2. **Temp dataset cleanup**: Happens on session exit or explicit `.cleanup()`
3. **Parallel execution**: Requires UDF and catalog to be serializable (cloudpickle)
4. **Type annotations**: Required for UDF inputs/outputs for schema inference
5. **Lazy evaluation**: Query not executed until terminal operation
6. **Dataset versions**: Once saved, versions are immutable

## Environment Variables

- `DATACHAIN_METASTORE`: Custom metastore class path
- `DATACHAIN_WAREHOUSE`: Custom warehouse class path
- `DATACHAIN_CACHE_DIR`: Cache directory (default: `.datachain/cache`)
- `DATACHAIN_LOG_LEVEL`: Logging level
