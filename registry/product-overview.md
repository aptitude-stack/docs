# Aptitude Registry - Product Overview

The Aptitude Registry is the hosted backend for immutable skill publication,
discovery, exact fetch, catalog data, governance, and audit. It is the only
Aptitude component that writes to PostgreSQL.

The registry stores facts. It does not decide which skill a local user should
install.

## Users

| User/system | Goal |
| --- | --- |
| Publisher / CI | Publish a new immutable skill version. |
| Resolver / MCP / CLI | Discover candidate slugs, fetch exact metadata, read direct dependency selectors, and download artifact bytes. |
| Website | Read catalog listings, search results, graph data, and star state server-side. |
| Operators | Run migrations, check readiness, rotate service tokens, index embeddings, inspect observability. |
| Reviewers/admins | Manage namespaces, organizations, policy packs, ownership, trust evidence, lifecycle, review state, and promotion channels. |

## What It Owns

- Immutable `slug@version` records.
- Digest-addressed `.tar.zst` artifact bytes.
- Metadata, tags, schemas, token/content estimates, and provenance snapshots.
- Direct authored dependency selectors.
- Lexical search projections and semantic embedding rows.
- Co-usage and catalog ranking signals.
- Lifecycle status, trust tier, namespace, review state, promotion channel, and
  policy pack attachment.
- Service-token authentication and route-scope enforcement.
- Audit records for publishes, governance changes, and discovery/search events.
- Health, readiness, structured logging, and OpenTelemetry metrics export.

## What It Does Not Own

| Concern | Owner |
| --- | --- |
| Prompt interpretation | Resolver / client |
| Client-side candidate reranking | Resolver / client |
| Final skill selection | Resolver / client |
| Transitive dependency solving | Resolver |
| Lock generation | Resolver |
| Local materialization and execution | Resolver / agent host |

## Route Families

| Category | Routes |
| --- | --- |
| Public status | `GET /`, `GET /favicon.svg`, `GET /healthz`, `GET /readyz` |
| Publish | `POST /skills/{slug}` |
| Discovery | `POST /discovery` |
| Exact reads | `GET /skills/{slug}`, `GET /skills/{slug}/{version}`, `GET /resolution/{slug}/{version}`, `GET /skills/{slug}/{version}/content` |
| Catalog | `GET /catalog/skills`, `GET /catalog/top-skills`, `GET /catalog/skill-graph`, `POST /catalog/search` |
| Stars/telemetry | `POST /catalog/star-events`, `GET /catalog/user-stars` |
| Governance | lifecycle, organization, namespace, ownership, trust evidence, policy-pack, review-state, and promotion-channel endpoints |

## Stack

| Layer | Technology |
| --- | --- |
| Runtime | Python 3.12+, FastAPI, Pydantic |
| Persistence | PostgreSQL, SQLAlchemy, Alembic |
| Search | PostgreSQL full-text search, pgvector HNSW, OpenAI embeddings |
| Artifacts | `.tar.zst`, `application/zstd`, SHA-256 checksums |
| Operations | Docker, Render, Neon, OpenTelemetry, Grafana Cloud, Loki |
| Tooling | uv, Ruff, pytest |

## Local Commands

```bash
uv sync --extra dev
make run-dev      # dev stack with demo seed
make run-prod     # production-like stack without demo seed
make quality
make test
make rotate       # service-token generator
```

## Reading Order

1. [Architecture](architecture.md)
2. [Discovery Mechanism](discovery-mechanism.md)
3. [Policies](policies.md)
4. [Tables](tables.md)
