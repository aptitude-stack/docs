# Client Policy Quick Summary

## Purpose

Client policy is a **local, client-owned rule system** for Aptitude skill resolution.

It controls:
- what skills are allowed
- how strict resolution should be
- how the client chooses between multiple legal candidates

It does **not** add a server-side policy service in v1.

## Main Use Cases

- **Developer**: personal local defaults
- **Team / repo**: shared rules committed with the project
- **DevOps / IT**: machine-wide baseline on managed devices or runners

## Where Config Lives

### Workspace config
- File: nearest `aptitude.toml` in the repo tree
- Owner: team / repo maintainers
- How it gets there: committed to Git

### User config
- Windows: `%APPDATA%\\aptitude\\aptitude.toml`
- macOS / Linux: `~/.config/aptitude/aptitude.toml`
- Owner: individual developer
- How it gets there: created manually or by onboarding/bootstrap scripts

### System config
- Windows: `%PROGRAMDATA%\\aptitude\\aptitude.toml`
- macOS / Linux: `/etc/aptitude/aptitude.toml`
- Owner: DevOps / IT / platform team
- How it gets there: machine provisioning, MDM, bootstrap, or image baking

## What Happens If No Config File Exists

The client still works. It falls back to the built-in default `PolicyContext`.

Current default policy:
- allowed lifecycle statuses: `published`, `deprecated`, `archived`
- allowed trust tiers: `verified`, `internal`, `untrusted`
- no token ceiling
- no content-size ceiling
- no total-graph token ceiling
- no total-graph content-size ceiling

Current default source:
- `client_default`

## Policy vs Selection

These are **both implemented**, because they solve different problems.

### Policy
Policy decides **what is allowed**.

Examples:
- allowed trust tiers
- allowed lifecycle statuses
- token ceilings
- content-size ceilings

### Selection
Selection decides **which allowed candidate should win**.

Examples:
- `balanced`
- `low-cost`
- `high-trust`
- interaction mode such as `auto`, `always`, `never`

## Merge Rules

### Selection merge
Selection uses normal override precedence:

`default < system < user < workspace < env < CLI`

Meaning: the more local / more explicit value wins.

### Policy merge
Policy uses **restrictive-only merge**:
- allowed lists are intersected
- numeric ceilings use the minimum

Meaning: lower layers may tighten policy, but may not weaken stricter higher-level policy.

## Selection Profiles

### `balanced`
- general-purpose default
- does not aggressively optimize for only trust or only cost

### `low-cost`
- prefers lighter / cheaper candidates
- tends to favor lower token estimate and smaller content size

### `high-trust`
- prefers safer / more trusted candidates
- tends to favor stronger trust tier and better lifecycle status

## Trust Tiers

### `verified`
- highest-confidence / better-vetted

### `internal`
- organization-trusted / internal source

### `untrusted`
- lowest trust tier

## Runtime Behavior

### `install` / `resolve`
- use the effective live policy during planning

### `sync --lock`
- replays the lockfile
- should **not** reinterpret current live policy
- goal: preserve reproducibility

## Current CLI Surface

- `aptitude install "<query>"`
- `aptitude resolve "<query>"` (hidden/internal)
- `aptitude sync --lock aptitude.lock.json`
- `aptitude policy show`

## Current Registry Endpoints Used By Install

- `POST /discovery`
- `GET /skills/{slug}/versions`
- `GET /skills/{slug}/versions/{version}`
- `GET /resolution/{slug}/{version}`
- `GET /skills/{slug}/versions/{version}/content`

## Influence From pip / npm / uv

We borrow the **layering idea**, not the full feature set.

- **pip**: global / user / site config, plus `PIP_CONFIG_FILE`
- **npm**: project / user / global config files
- **uv**: project / user / system layering and upward discovery

Aptitude should copy:
- layered config
- clear file locations
- inspectability

Aptitude should **not** copy:
- config mutation commands like `set`, `edit`, `delete`
- a generic config-management UX

## V1 Decisions

- keep policy fully client-owned and file-based
- reuse `aptitude.toml`
- keep policy under `[policy]`
- keep preferences under `[selection]`
- add system-level config discovery
- add `aptitude policy show`
- do **not** add `aptitude config set/edit/delete`
- do **not** add a server-side policy service
- defer `--config-file` / `APTITUDE_CONFIG_FILE` to phase 2
