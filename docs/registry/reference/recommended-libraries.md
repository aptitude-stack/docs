# Aptitude Recommended Libraries

## Purpose

This document records which libraries are currently in use, which are endorsed by responsibility, and which are still deferred.

## Current Runtime Dependencies

These are already part of the resolver:

- `pydantic`: DTOs and explicit serialization boundaries
- `pydantic-settings`: typed configuration
- `typer`: CLI surface
- `prompt-toolkit`: richer interactive CLI prompts and menus
- `rich`: panels, progress indicators, and styled CLI output
- `httpx`: registry transport
- `packaging`: deterministic version comparison
- `diskcache`: advisory caching
- `tenacity`: transient retry handling
- `structlog`: structured logging
- `tomli`: compatibility parsing on older Python runtimes

## Current Dev Dependencies

- `pytest`: tests
- `ruff`: linting and formatting
- `mypy`: static checking

## Deferred Libraries

Add these only if the corresponding capability becomes real:

- `pluggy`: plugin surface under a future `plugins/` package
- `resolvelib`: only if the current deterministic resolver outgrows the explicit implementation
- `networkx`: only if graph analysis or visualization clearly justifies it
- `fastapi` / `uvicorn`: only if Aptitude grows a local HTTP service surface

## Guidance

- prefer the existing stack unless a new dependency solves a real ownership problem
- do not add libraries just because they are popular
- keep runtime dependencies aligned with package responsibilities described in the architecture docs
