# Development Setup

## Runtime

- Python `>=3.9`
- `uv` for environment and task execution

## Install Dependencies

```bash
uv sync --extra dev
```

## Common Commands

Use the provided `Makefile` for everyday work:

```bash
make run
make debug
make test
make lint
make format
make typecheck
```

Direct `uv` equivalents:

```bash
UV_CACHE_DIR=.uv-cache uv run --extra dev pytest -q
UV_CACHE_DIR=.uv-cache uv run --extra dev ruff check src tests
UV_CACHE_DIR=.uv-cache uv run --extra dev ruff format src tests
UV_CACHE_DIR=.uv-cache uv run --extra dev python -m mypy src tests
```

## Entry Points

- verified repo-local entrypoint: `PYTHONPATH=src .venv/bin/python -m aptitude_resolver`
- logical console command name: `aptitude`

Running the module entrypoint with no arguments currently launches the install-first wizard and falls back to CLI subcommands when arguments are present. Use `aptitude manifest` to inspect the full command and flag surface without entering the wizard.

## Naming Conventions

- prefer `kebab-case` for non-imported filenames such as Markdown docs and similar repo artifacts
- keep Python packages and importable module files in `snake_case`

Python import paths must be valid identifiers. Using `kebab-case` for importable Python modules creates awkward or invalid imports, so this repo should not apply the same filename rule everywhere.
