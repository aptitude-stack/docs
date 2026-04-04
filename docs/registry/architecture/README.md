# Architecture Docs

These files define the current normative technical truth for Aptitude.

## Reading Order

1. [system-overview.md](system-overview.md)
2. [cli-interface.md](cli-interface.md) when the change touches the human CLI surface, wizard routing, prompts, help text, or rendering
3. [server-resolver-boundary.md](server-resolver-boundary.md)
4. [decision-rules.md](decision-rules.md)
5. [selection-and-governance.md](selection-and-governance.md) when the change touches discovery, ranking, policy, governance, lockfiles, or checksums

## Ownership

- `system-overview.md` explains the shape of the system and package boundaries.
- `cli-interface.md` defines the current CLI command, wizard, and rendering contract.
- `server-resolver-boundary.md` defines the hard ownership split between server facts and resolver decisions.
- `decision-rules.md` defines hard implementation constraints.
- `selection-and-governance.md` is the detailed contract for fresh-planning behavior.

If code changes architecture, boundaries, or normative behavior, update these docs in the same change.
