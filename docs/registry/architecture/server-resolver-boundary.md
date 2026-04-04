# Server-Resolver Boundary

This document defines the hard ownership split between Aptitude Server and aptitude.

## Server Owns Facts

The server owns:

- registry storage
- immutable metadata
- immutable artifacts
- indexed candidate retrieval
- published checksum facts

The server may inform ranking and governance by returning facts, but it does not make the final decision for the resolver.

## Resolver Owns Decisions

The resolver owns:

- intent interpretation
- candidate reranking
- version selection
- final root selection
- dependency solving
- governance
- lock generation
- execution planning

## Practical Consequences

- discovery may consume server facts, but final root choice remains local
- lock replay must not call discovery or dependency solving again
- transport artifacts in `docs/reference/openapi/` inform the registry boundary, but they do not override resolver architecture docs
- if code begins depending on server ordering as final truth, the boundary has been violated
