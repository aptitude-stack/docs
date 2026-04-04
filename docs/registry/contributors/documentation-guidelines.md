# Documentation Guidelines

## Canonical Vs Derivative Docs

Canonical human-facing docs live under `docs/`.

Derivative agent-facing docs live under `.agents/` and should point back to canonical docs instead of restating them in full.

## Current-State Vs Forward-Looking Docs

- `docs/architecture/` and `docs/reference/` describe current truth.
- `docs/roadmap/` describes planned or deferred direction.

Do not mix current-state guarantees with future intent in the same paragraph unless the difference is explicit.

## Update Rules

Update docs in the same change when you alter:

- product or package identity
- architecture boundaries
- command behavior
- lock or governance behavior
- package ownership

Minimum files to consider on non-trivial changes:

- `README.md`
- `docs/README.md`
- relevant `docs/architecture/*`
- relevant contributor or reference docs
- `.agents/agent.md`
- `.agents/memory/meta.md`

## Historical Plans

`.agents/plans/` is implementation history.

Do not treat plan files as current architecture or contributor guidance. If a plan disagrees with canonical docs, the canonical docs win.
