# Aptitude System Overview

## Identity

Aptitude is the local decision-making engine for AI skill planning and materialization.

The system is intentionally split:

- Aptitude Server owns registry data, immutable metadata, immutable artifacts, and indexed candidate retrieval.
- Aptitude owns intent interpretation, candidate selection, dependency solving, governance, locking, and execution planning.

The server returns facts.
The resolver makes decisions.

## Canonical Flows

### Fresh Planning

Use this when the input is a user query.

```text
query
-> interface
-> application
-> discovery
-> resolver
-> governance
-> lockfile
-> execution planning
-> materialization or result rendering
```

This is the flow behind:

- `aptitude install`
- hidden `aptitude resolve`

### Lock Replay

Use this when the input is an existing lockfile.

```text
lockfile
-> interface
-> application
-> lock parse + replay
-> execution planning
-> materialization
```

This is the flow behind:

- `aptitude sync --lock`

Lock replay is intentionally shorter. Once a valid lock exists, discovery and dependency solving must not run again.

## Package Boundaries

The current package tree is rooted at `src/aptitude_resolver/`.

- `application/`: orchestration and DTO boundaries
- `cache/`: advisory caching helpers
- `discovery/`: intent parsing, query building, non-final candidate shaping and reranking
- `domain/`: models, policy types, tracing models, resolver-owned errors
- `execution/`: lock-driven execution planning and materialization
- `governance/`: legality checks before lock generation
- `interfaces/`: CLI, wizard-oriented interface helpers, and shared interface support
- `lockfile/`: lock schema, serializer, parser, and replay helpers
- `registry/`: Aptitude Server transport boundary and transport-to-domain mapping
- `resolution/`: deterministic version choice, root selection, dependency expansion, and validation
- `shared/`: config, logging, and small shared utilities
- `telemetry/`: additive metrics and instrumentation

## Core Invariants

- Resolver logic must be deterministic for the same logical inputs.
- Discovery may shape and rerank candidates, but it must not make final root decisions.
- Execution must consume lock data only.
- The server is a fact source, not the final decision-maker.
- Explainability, telemetry, cache, and retry remain additive; they must not change correctness.

## Current Technical Direction

The active hardening areas are:

- test hardening
- strict policy overrides for fresh planning
- cache and retry
- observability
- advanced governance

The currently deferred areas are:

- organization-managed policy sources
- broader governance and explanation refinement
- plugins
- SDK surface
- MCP surface
