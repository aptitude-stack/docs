# Testing And Verification

## Expectations

Every non-trivial change should leave behind:

- focused unit coverage for changed behavior
- deterministic behavior checks when ordering or selection matters
- updated docs when boundaries or behavior changed

## Fast Verification

For quick feedback, prefer focused pytest slices around the packages you touched.

Common high-signal slices:

- `tests/unit/interfaces/cli/`
- `tests/unit/application/`
- `tests/unit/resolution/`
- `tests/unit/lockfile/`
- `tests/unit/shared/test_imports.py`

## Full Verification

```bash
make test
make lint
make typecheck
```

## Integration Tests

The integration suite hits the live Aptitude Server boundary and is marked with `integration`.

Run it intentionally, not by default:

```bash
UV_CACHE_DIR=.uv-cache uv run --extra dev pytest -m integration -q
```

## What To Verify Before Calling Work Complete

- the intended tests pass with fresh output
- no import-path drift remains after package moves
- CLI help and user-facing error rendering still make sense
- canonical docs and agent docs reflect the current resolver identity
