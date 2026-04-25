# Aptitude Decision Rules

These rules are hard constraints, not style suggestions.

## Before Non-Trivial Changes

You must:

1. identify the owning package
2. identify whether the work belongs to fresh planning or lock replay
3. check whether the architecture docs already cover the intended behavior
4. update the relevant architecture docs in the same change when behavior or boundaries change
5. decide how determinism and traceability will be preserved

## Layering Rules

Do not:

- put business logic in CLI or wizard handlers
- put solver logic in `application/`
- let `discovery/` perform final root selection
- let `execution/` re-resolve dependencies
- hide feature logic in `shared/`

## Execution Rules

Execution must operate from lock data only.

Do not:

- rebuild execution order from an unlocked graph during materialization
- treat `ResolutionGraph` as the execution source of truth
- require discovery or resolver behavior during `sync --lock`

## Selection And Governance Rules

- candidate-policy filtering must happen before final ranking and root selection
- graph governance must still run after dependency resolution and before lock generation
- ranking compares only legal candidates
- interactive ambiguity handling stays root-only
- interfaces may render comparison details, but must not recompute ranking decisions

## Determinism Rules

For the same logical inputs, the resolver should produce the same:

- selected candidate
- version choice
- dependency graph
- install order
- lockfile
- execution plan

Whenever order matters, define the tiebreak in code and tests.

## Observability And Cache Rules

- telemetry must stay additive
- cache entries are advisory only
- retries belong at the registry transport boundary
- cache, retry, telemetry, and explainability must not change correctness

## Documentation Rules

When behavior or boundaries change, update the canonical docs in the same change:

- [system-overview.md](system-overview.md)
- [decision-rules.md](decision-rules.md)
- [selection-and-governance.md](selection-and-governance.md) when applicable

Also update supporting docs when needed:

- [../README.md](../README.md)
- [../contributors/documentation-guidelines.md](../contributors/documentation-guidelines.md)
