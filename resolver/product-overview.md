# Aptitude Resolver - Product Overview

The Aptitude Resolver is the local decision engine for Aptitude skills. It is a
Python package and command-line tool that also exposes a local stdio MCP server
for agent hosts.

The resolver consumes registry facts and makes runtime decisions locally:
intent parsing, candidate selection, dependency solving, client policy,
lockfile generation, and materialization.

## Users

| User | Goal |
| --- | --- |
| Developer | Install a skill from a query or replay a lockfile. |
| Agent host | Search, inspect, resolve, install, or sync through MCP. |
| CI/automation | Run deterministic install/sync commands with JSON-friendly output. |
| Operator/security reviewer | Inspect effective local policy and selection behavior. |

## What It Owns

- CLI and install-first wizard.
- Local stdio MCP server.
- Intent parsing and registry discovery requests.
- Candidate metadata enrichment, policy filtering, reranking, and final root
  selection.
- Version selection and recursive dependency graph solving.
- Two-phase client governance.
- Lockfile generation and replay.
- Execution plan creation.
- Artifact fetch, compressed-byte checksum verification, safe extraction, and
  target promotion.
- Layered local configuration through `aptitude.toml` and environment
  overrides.

## What It Does Not Own

| Concern | Owner |
| --- | --- |
| Publish packaging | Publisher |
| Authoritative skill metadata | Registry |
| Search index storage | Registry |
| Exact artifact bytes | Registry |
| Registry lifecycle/review policy | Registry |
| Stable public Python SDK | Deferred |

## Commands

```bash
# One-off published usage
uvx aptitude-resolver --help
uvx aptitude-resolver install "review FastAPI pull requests"
uvx aptitude-resolver sync --lock aptitude.lock.json
uvx aptitude-resolver policy show
uvx aptitude-resolver mcp

# Development
uv sync --extra dev
uv run aptitude-resolver --help
make test
make lint
make typecheck
make build
```

Installed console scripts include `aptitude-resolver`, `aptitude`, and
`aptitude-mcp`. The Python import module is `aptitude_resolver`.

## Configuration

| Setting | Purpose |
| --- | --- |
| `APTITUDE_READ_TOKEN` | Registry read token. |
| `APTITUDE_SERVER_BASE_URL` | Registry API base URL override. |
| `APTITUDE_SERVER_TIMEOUT_SECONDS` | Registry request timeout. |
| `APTITUDE_PREFER` | Selection profile override. |
| `APTITUDE_INTERACTION_MODE` | Interaction behavior override. |
| `APTITUDE_CONCURRENT_DOWNLOADS` | Materialization download concurrency. |
| `APTITUDE_CONCURRENT_INSTALLS` | Materialization install concurrency. |

Policy and selection config are loaded from layered `aptitude.toml` files:
system, user, nearest workspace, then request/CLI overrides where supported.

## Main Flows

Fresh planning:

```text
query -> discovery -> candidate metadata -> policy filter -> rerank -> select
      -> dependency graph -> graph governance -> lockfile -> materialize
```

Lock replay:

```text
aptitude.lock.json -> execution plan -> exact artifact fetch -> checksum verify
                   -> safe extract -> target promotion
```

## Reading Order

1. [Architecture](architecture.md)
2. [Policies](policies.md)
3. [Ranking](ranking.md)
