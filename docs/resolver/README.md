# Aptitude Docs

Use this directory as the canonical documentation entrypoint for contributors, reviewers, and operators.

## Taxonomy

- `architecture/`: normative current-state system structure, boundaries, and invariants
- `contributors/`: practical repo workflow for engineers and reviewers
- `reference/`: stable technical facts, contracts, schema, storage, and operations material
- `roadmap/`: forward-looking technical direction and drafts, explicitly non-normative unless promoted
- `changelog/`: protected implementation history

Related entrypoints outside this directory:

- [../README.md](../README.md): cold-start repo overview
- [../TODO.md](../TODO.md): intentionally small near-term backlog
- [../.agents/README.md](../.agents/README.md): agent-only derivative context
- [../.agents/plans/roadmap.md](../.agents/plans/roadmap.md): canonical implementation sequence

## Read By Goal

### Understand the system

- [architecture/README.md](architecture/README.md): architecture reading order and normative docs
- [architecture/system-overview.md](architecture/system-overview.md): high-level system shape
- [architecture/cli-interface.md](architecture/cli-interface.md): current CLI command, wizard, and rendering contract
- [architecture/server-resolver-boundary.md](architecture/server-resolver-boundary.md): hard ownership split with the registry and server-side facts
- [architecture/decision-rules.md](architecture/decision-rules.md): implementation invariants and doc-sync rules

### Make or review changes

- [contributors/README.md](contributors/README.md): contributor entrypoint
- [contributors/development-setup.md](contributors/development-setup.md): environment setup and common commands
- [contributors/testing-and-verification.md](contributors/testing-and-verification.md): verification expectations
- [contributors/module-guide.md](contributors/module-guide.md): practical package ownership map
- [contributors/documentation-guidelines.md](contributors/documentation-guidelines.md): canonical vs derivative docs rules

### Check live technical facts

- [reference/README.md](reference/README.md): reference index
- [reference/cli-command-reference.md](reference/cli-command-reference.md): full user-facing CLI command reference
- [reference/api-contract.md](reference/api-contract.md): canonical HTTP contract
- [reference/schema.md](reference/schema.md): canonical local schema surfaces
- [reference/storage-strategy.md](reference/storage-strategy.md): current storage strategy
- [reference/operations/README.md](reference/operations/README.md): operational runbook index

### Review roadmap and history

- [roadmap/README.md](roadmap/README.md): forward-looking documentation index
- [roadmap/client-policy-summary.md](roadmap/client-policy-summary.md): short explainer for client policy scopes and terminology
- [roadmap/client-policy-configuration.md](roadmap/client-policy-configuration.md): v1 client policy configuration design
- [roadmap/requirements-and-phases.md](roadmap/requirements-and-phases.md): product requirements and phased rollout framing
- [roadmap/near-term-evolution.md](roadmap/near-term-evolution.md): next hardening and capability themes
- [changelog/README.md](changelog/README.md): protected implementation history entrypoint

## Ownership Rules

- `docs/architecture/*`: normative current-state behavior and invariants
- `docs/contributors/*`: how to work in the repo
- `docs/reference/*`: stable supporting facts and operations material
- `docs/roadmap/*`: forward-looking guidance, explicitly non-normative unless promoted
- `docs/changelog/*`: implementation history, not canonical current-state truth
- `.agents/*`: derivative agent operating context only

Historical milestone docs may mention old paths or the former product name. Treat them as implementation history, not as the current source of truth. Human canonical docs live under `docs/`, while implementation sequencing intentionally remains anchored in [../.agents/plans/roadmap.md](../.agents/plans/roadmap.md).
