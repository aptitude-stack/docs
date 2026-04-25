# Client Policy Configuration

This document captures the v1 design for client-owned policy configuration in Aptitude.

## Purpose

Client policy gives the Aptitude client a local governance model for skill resolution.

It answers two different questions:

- **Policy:** what is allowed at all
- **Selection:** how to choose among already-legal candidates

The feature stays fully client-owned and file-based. It does not add a server-side policy service.

## V1 Scope

The first version includes:

- one persistent config format: `aptitude.toml`
- `[policy]` for hard governance rules
- `[selection]` for ranking and prompting preferences
- system, user, and workspace config discovery
- a read-only inspection command: `aptitude policy show`

The first version explicitly does **not** include:

- `aptitude config set`
- `aptitude config edit`
- `aptitude config delete`
- `--config-file`
- `APTITUDE_CONFIG_FILE`
- remote or centrally managed policy services

## Primary Use Cases

- **Developer:** personal defaults for ranking and prompting
- **Team / repo:** shared repo-level policy committed to Git
- **DevOps / IT:** machine-wide baseline on managed devices and runners

## Config Scopes

### Workspace

- Owner: repo maintainers
- File: nearest `aptitude.toml` found by walking upward from the current working directory
- Typical source: committed to Git

### User

- Owner: individual developer
- Windows: `%APPDATA%\\aptitude\\aptitude.toml`
- macOS / Linux: `$XDG_CONFIG_HOME/aptitude/aptitude.toml` or `~/.config/aptitude/aptitude.toml`

### System

- Owner: DevOps / IT / platform team
- Windows: `%PROGRAMDATA%\\aptitude\\aptitude.toml`
- macOS / Linux: `/etc/aptitude/aptitude.toml`

## Merge Model

### Selection

Selection uses normal override precedence:

`default < system < user < workspace < env < CLI`

The more local or explicit non-null value wins.

### Policy

Policy uses restrictive-only merge:

`default -> system -> user -> workspace -> CLI`

Rules:

- allowed lists intersect
- numeric ceilings take the minimum
- lower layers may tighten higher-level policy
- lower layers may not weaken higher-level policy

## CLI Surface

Human-facing inspection is:

```bash
aptitude policy show
aptitude policy show --json
```

The command reports:

- effective selection preferences
- effective policy
- contributing config layers
- merge semantics

## Runtime Boundary

- `install` and hidden `resolve` use the effective live policy during fresh planning
- `sync --lock` does **not** reapply current live policy
- lock replay stays lock-driven so previously planned results remain reproducible

## Default Behavior

If no config file exists, the client still works.

It falls back to built-in defaults from `PolicyContext` and `SelectionPreferences`.

Current defaults are intentionally permissive enough for first-run use:

- selection profile: `balanced`
- interaction mode: `auto`
- lifecycle allowed: `published`, `deprecated`, `archived`
- trust allowed: `verified`, `internal`, `untrusted`
- no token or content-size ceilings unless configured
