# Module Guide

This guide maps responsibilities to the current `src/aptitude_resolver/` tree.

## Core Packages

- `application/`: use-case orchestration, DTO boundaries, workflow sequencing
- `cache/`: advisory caches around registry-backed reads
- `discovery/`: intent parsing, query building, non-final candidate reranking
- `domain/`: models, policy types, tracing, resolver-owned errors
- `execution/`: execution planning and materialization from lock data
- `governance/`: legality checks before lock generation
- `lockfile/`: durable resolved representation, serializer, parser, and replay helpers
- `registry/`: Aptitude Server transport, auth, and transport-to-domain mapping
- `resolution/`: deterministic version selection, root selection, dependency expansion, conflict checks, validation

## Interface Packages

- `interfaces/cli/`: Typer command surface, wizard metadata, and install-first guided flow
- `interfaces/shared/`: shared workflow helpers used by multiple interface surfaces

## Support Packages

- `shared/config/`: environment and file-backed configuration
- `shared/logging/`: logging setup
- `telemetry/`: additive instrumentation

## Reserved Or Deferred Areas

- `plugins/`: planned, not implemented
- `interfaces/sdk/`: planned, not implemented
- `interfaces/mcp/`: planned, not implemented
